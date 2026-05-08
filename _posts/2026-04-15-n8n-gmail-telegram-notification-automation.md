---
title: Building a Gmail-to-Telegram Alert System with Self-Hosted n8n
date: 2026-04-15 09:00:00 -0400
categories: [Deep Dives]
tags: [homelab, n8n, telegram, gmail]
image:
  path: /assets/img/posts/n8n-gmail-telegram-hero.svg
  alt: Gmail-to-Telegram Alert System Illustration
---

# Building a Gmail-to-Telegram Alert System with Self-Hosted n8n

I needed to know the moment a specific email arrived. Not within the hour, not when I happened to check — immediately. The email was coming from a known sender, and I wanted a Telegram ping the second it hit my inbox.

This post walks through how I built that system, including the parts where things broke.

---

## Choosing the Approach

Gmail doesn't natively integrate with Telegram. It offers push notifications to your phone and server-side filters, but neither connects to a Telegram bot. So I evaluated three paths:

**Google Apps Script** could poll Gmail and call the Telegram Bot API. Minimum polling interval is about one minute. Simple, but tightly coupled to Google's scripting environment.

**Gmail API with Pub/Sub** is Google's official mechanism for third-party push notifications. Gmail pushes a notification to a Google Cloud Pub/Sub topic when something changes in the mailbox. A listener picks it up and acts on it. This is event-driven and near-instant, but requires a GCP project, OAuth, Pub/Sub configuration, and a persistent listener.

**n8n**, self-hosted, sits in the middle. It's a visual workflow automation platform with built-in Gmail and Telegram nodes. I'd been meaning to try it anyway, and it runs as a Docker container — easy to deploy on infrastructure I already manage.

I went with n8n, and decided to build two variants of the same workflow: one using polling and one using push. Partly for redundancy, partly to see the latency difference firsthand.

![Diagram of the Gmail-to-Telegram Alert System](/assets/img/posts/n8n-gmail-telegram-hero-latest.svg)


### A Note on n8n's Gmail Integration

Early in the research, I assumed n8n's Gmail Trigger node used webhooks or Pub/Sub for real-time delivery. It doesn't. The node polls Gmail at a configurable interval — every minute in production. That's an important distinction: the polling flow is simple but introduces up to 60 seconds of delay. For truly instant notifications, you need the Pub/Sub path, which n8n supports through its generic Webhook node but doesn't abstract for you.

### Why Not Jenkins?

I considered Jenkins since I already run it on the same VPS. But Jenkins has no native way to subscribe to Google Cloud Pub/Sub. I'd need a bridge — a Cloud Function or a custom script — to translate Pub/Sub pushes into Jenkins webhook triggers. Jenkins also spins up a full build executor just to send a Telegram message, which felt heavy. n8n is purpose-built for this kind of lightweight event-driven workflow.

### n8n vs Node-RED

Both are visual, node-based, and self-hostable. The difference is focus: Node-RED was built for IoT and hardware — sensors, MQTT, serial ports. n8n was built for SaaS integrations and API workflows. For connecting Gmail to Telegram, n8n's pre-built nodes made it the natural fit. Node-RED would have required more manual HTTP wiring.

---

## Deploying n8n

### Infrastructure

n8n runs as a Docker container on a VPS I already use for other services (Jenkins, Grafana, Keycloak). The deployment is managed by Ansible with the following setup:

- **n8n container** bound to `localhost:5678`
- **Dedicated Postgres 16 (Alpine)** container for n8n's data — not sharing the existing Keycloak database, for isolation
- **Caddy** (bare-metal, not containerized) as the reverse proxy, using a wildcard TLS certificate via Route53 DNS-01
- **Bind mounts** for data directories rather than Docker named volumes, for backup compatibility
- **Secrets** managed through Ansible Vault, rendered into a `.env` file with restrictive permissions — not baked into `docker-compose.yml`

The Caddy configuration drops into `/etc/caddy/conf.d/` as a handle block, matching the pattern I use for other services. The critical addition for n8n is `flush_interval -1` in the reverse proxy directive — n8n uses Server-Sent Events for its UI, and without unbuffered streaming, the editor loses its real-time connection to the backend.

### First Boot

The first deployment got most of the way there. The n8n frontend loaded, but the browser showed "Error connecting to n8n." The HTML was served, but the SSE connection couldn't establish.

The root cause was straightforward: the Ansible playbook had failed at the Caddy configuration step (a template validation issue), so the Caddy reload handler never fired. The n8n container was running and healthy — `curl localhost:5678/healthz` returned 200 — but Caddy wasn't proxying to it correctly.

A Caddy reload alone didn't fix it. I had to bring the n8n container down and back up to reset its connection state. After that, the editor loaded cleanly.

### Telemetry and Licensing

n8n collects anonymous telemetry from self-hosted instances by default — workflow execution counts, enabled integrations, and an instance pulse sent every six hours. This is separate from any license activation. I disabled it with environment variables:

```
N8N_DIAGNOSTICS_ENABLED=false
N8N_VERSION_NOTIFICATIONS_ENABLED=false
N8N_TEMPLATES_ENABLED=false
N8N_DIAGNOSTICS_CONFIG_FRONTEND=
N8N_DIAGNOSTICS_CONFIG_BACKEND=
```

Some of those are set to empty strings intentionally. They override default values baked into n8n that point to telemetry endpoints. Setting them empty nullifies those defaults.

On licensing: n8n markets itself as open source, but it's more accurately described as source-available under the Sustainable Use License. The community edition is free for internal and personal use. Enterprise features (SSO, RBAC, log streaming) are both legally gated by the license and technically gated by a license key check in the running application. The free activation key unlocks a small subset — folders, advanced debugging, execution search — which is genuinely useful and has no expiration.

---

## Google Cloud Platform Setup

Both flows need Gmail API OAuth credentials. The push flow additionally requires Pub/Sub.

### OAuth Consent Screen

Google has redesigned this UI. The scope configuration now lives under a separate "Data Access" page in the Google Auth Platform section, not inline with the consent screen setup. I initially couldn't find where to add scopes until I navigated to the Data Access page in the left sidebar.

### The Scope Saga

This was where the most iterative debugging happened.

I started by adding `gmail.readonly` as the only scope. The Gmail Trigger node authenticated successfully and fetched emails. But when I tried to use the generic Google OAuth2 credential for HTTP Request nodes (needed for the push flow), I hit `403 — insufficient authentication scopes`.

The problem cascaded across several steps:

1. The Gmail Trigger node silently requested the `gmail.labels` scope in addition to `gmail.readonly`. I hadn't added it to the GCP project's Data Access configuration.

2. After adding the labels scope in GCP, I assumed the existing OAuth token would pick it up. It didn't. OAuth tokens are minted with the scopes available at authorization time. I needed to re-authorize to get a new token that included both scopes.

3. Re-clicking "Sign in with Google" on the existing n8n credential sometimes reused the old grant without prompting for new scopes. The reliable fix was to delete the credential entirely, recreate it, and go through the full OAuth flow again.

4. n8n has two different credential types for Gmail: **Gmail OAuth2 API** (for built-in Gmail nodes) and **Google OAuth2 API** (for generic HTTP Request nodes). They use the same underlying Client ID and Secret but are different credential types in n8n's system. The Gmail Trigger node only shows credentials of the matching type, so creating the wrong one means it doesn't appear in the dropdown.

Each of these individually would have been a quick fix. Stacked together, they made for a frustrating hour of "why isn't this working."

---

## Flow A: Polling

The polling workflow is three nodes:

**Gmail Trigger** → **IF (filter by sender)** → **Telegram (send message)**

The Gmail Trigger polls every minute, filtered by a Gmail label (`n8n-antoine-alert`) and read status (unread). The unread filter serves as a deduplication mechanism — once I read the email in Gmail, it won't trigger again on the next poll cycle.

A Gmail server-side filter applies the label automatically to emails matching the target sender addresses. This means Gmail handles the initial sorting, and n8n only processes what's relevant.

The IF node is a safety net — it verifies the sender address against my target list using `contains` conditions with OR logic. If the Gmail filter ever catches something unexpected, this node drops it before it reaches Telegram.

### Telegram Parse Mode

I initially used MarkdownV2 for the notification message. This broke because the `From` field contains angle brackets around the email address (`Sender Name <email@example.com>`), which MarkdownV2 interprets as formatting. I switched to HTML parse mode, which also broke — the angle brackets were now parsed as HTML tags. The fix was to use no parse mode at all (plain text). For a notification alert, formatting isn't worth the fragility.

The polling flow worked on the first end-to-end test. Notifications arrived within about 60 seconds of sending a test email.

---

## Flow B: Push

The push workflow is substantially more complex:

**Webhook** → **Code (decode Pub/Sub)** → **HTTP Request (fetch messages)** → **IF (results exist)** → **HTTP Request (get message content)** → **Code (extract headers)** → **IF (filter by sender)** → **Telegram (send message)**

### Google Cloud Pub/Sub Setup

The push flow requires several GCP components:

1. A Pub/Sub topic
2. A push subscription pointing to the n8n webhook URL
3. The `gmail-api-push@system.gserviceaccount.com` service account granted Publisher role on the topic
4. A Gmail watch activated via the `users.watch` API endpoint

The watch tells Gmail to push notifications to the Pub/Sub topic whenever something changes in the mailbox. The Pub/Sub subscription forwards those notifications to n8n's webhook endpoint.

### The Watch Activation Problem

After wiring everything up, nothing happened. No webhook triggers, no Pub/Sub messages. The issue was simple: I hadn't activated the Gmail watch. The Pub/Sub infrastructure was in place, but Gmail wasn't told to push to it.

When I tried to activate the watch via curl using a token from Google's OAuth Playground, it failed with "Invalid topicName does not match projects/google.com:oauth-2-playground/topics/*". Despite using my own Client ID and Secret in the Playground, the token was issued under the Playground's project context, not my GCP project.

The fix was to activate the watch from within n8n itself, using an HTTP Request node with the Google OAuth2 credential — which is properly tied to my GCP project. This worked immediately.

### HTTP Request Node Issues

The HTTP Request nodes gave me persistent trouble during development. They would spin indefinitely with no output and no error. I verified the Gmail API was reachable by running curl from inside the n8n container (`docker exec`), and it returned data fine.

The root cause turned out to be unrelated to networking or auth: n8n's execution engine requires a trigger node to kick off the execution chain. Clicking "Execute Step" on a standalone node with no input just hangs. Adding a Manual Trigger node before the HTTP Request and clicking "Test Workflow" from there solved it immediately. This is a quirk of n8n's execution model that isn't immediately obvious.

### Expression Mode

One node failed with "Invalid id value" when trying to fetch a specific email by ID. The URL contained `{{ $json.messages[0].id }}`, but n8n was treating it as literal text rather than evaluating the expression. The URL field needs to be toggled into expression mode (the `fx` icon) for template syntax to be evaluated. Without that toggle, the double braces are just characters.

### Case Sensitivity

The Code node that extracts email headers output lowercase keys (`from`, `subject`). The downstream IF node referenced `$json.From` with a capital F. The filter silently sent everything to the false branch. A one-character case mismatch caused the entire push flow to appear broken.

### The Result

Once all the pieces were in place, the push flow delivered Telegram notifications within 2-5 seconds of an email arriving. Compared to the polling flow's ~60 second delay, the difference was immediately obvious.

---

## Watch Renewal

Gmail watches expire after approximately 7 days. I built a separate workflow to auto-renew:

**Schedule Trigger (every 3 days)** → **HTTP Request (call users.watch)** → **IF (check success)** → **Telegram (confirm renewal / alert failure)**

The success notification includes the watch expiration timestamp, converted from Unix milliseconds to a human-readable date in my timezone. A separate reusable Error Workflow catches any unexpected failures (network errors, auth expiry, timeouts) across all three workflows and sends a Telegram alert.

---

## The Ansible Re-run Incident

Several days after the initial deployment, I ran the Ansible playbook again to apply configuration updates. It failed during `docker compose pull` with a YAML parsing error in the compose file. A commented-out environment variable with malformed syntax caused the YAML parser to choke.

The more serious consequence: after the failed run, Postgres couldn't start. The error was `could not open file "global/pg_filenode.map": Permission denied`.

The Ansible task that creates the Postgres data directory had set ownership to UID 999, which is the conventional Postgres UID in Debian-based images. But the `postgres:16-alpine` image runs as UID 70. On the host, UID 999 mapped to the Caddy user. The initial deployment worked because Postgres initializes its own data files on first run — the directory ownership didn't matter when the directory was empty. On the re-run, the task touched the directory ownership, and the combination of a dirty shutdown from the failed compose and the UID mismatch left Postgres unable to read its own files.

The fix was `chown -R 70:70` on the Postgres data directory, and updating the Ansible task to use the correct UID for Alpine-based images.

This is a common assumption that catches people: "Postgres runs as UID 999" is only true for Debian-based images. Alpine uses UID 70. If you're using `postgres:16-alpine` and setting file ownership in your deployment automation, verify the actual UID inside the container with `docker exec <container> id postgres`.

---

## Future Work

A few items I've deferred:

- **Intermittent UI disconnects:** The n8n editor shows "connection lost" every few seconds. Container logs are clean, memory is well within limits, and the proxy configuration is correct. It doesn't affect workflow execution, but it's annoying during development. Still investigating.
- **Duplicate push notifications:** The push flow re-notifies on the same email because Gmail's watch fires on any mailbox change, not just new emails. The current workaround is manually marking emails as read. The proper fix is to add a node that marks the email as read via the Gmail API after sending the Telegram notification, which requires the `gmail.modify` scope.
- **Infrastructure as Code for workflows:** n8n supports exporting workflows as JSON and managing them via API or CLI. The next step is storing the workflow definitions in Git and deploying them alongside the infrastructure.

---

## What I'd Do Differently

If I were starting over, I'd skip the push flow initially and ship the polling variant first. The 60-second delay is acceptable for most use cases, and the polling flow took about 30 minutes to build compared to several hours for the push variant. The push flow was a worthwhile learning exercise — it forced me to understand Gmail's Pub/Sub integration, OAuth token scoping, and n8n's execution model — but in terms of practical value, the polling flow does the job.

I'd also verify the Postgres container's UID before writing the Ansible task, rather than assuming 999. A one-line `docker exec` check would have saved the debugging session.

---

## Stack

- **n8n 2.18.4** (self-hosted, Docker)
- **Postgres 16 Alpine** (dedicated instance)
- **Caddy** (bare-metal, wildcard TLS via Route53 DNS-01)
- **Ansible** (deployment automation, vault-encrypted secrets)
- **GitHub Actions** (CI/CD with ansible-lint and automated deployment)
- **Google Cloud Platform** (Gmail API, Pub/Sub)
- **Telegram Bot API**
