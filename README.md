# chisomonuegbu.dev

Personal blog. Built with Jekyll + the Chirpy theme, hosted on GitHub Pages,
served from `chisomonuegbu.dev`.

This README is the operational guide: how to set this up the first time, how
to write a post, and how to run things locally. Read it once end-to-end before
you start clicking around — it'll save you debugging later.

---

## What's in this repo

```
.
├── _config.yml                            # site configuration
├── _data/
│   └── contact.yml                        # social/contact links in the sidebar
├── _tabs/
│   └── about.md                           # About page (placeholder — customize me)
├── assets/
│   └── css/
│       └── jekyll-theme-chirpy.scss       # Fraunces + Inter + deep navy palette
├── _posts/                                # blog posts go here
├── Gemfile                                # ruby dependencies (from chirpy-starter)
└── .github/workflows/pages-deploy.yml     # GitHub Pages CD (from chirpy-starter)
```

The files this repo *adds* on top of the chirpy-starter template:
`_config.yml`, `_data/contact.yml`, `_tabs/about.md`, and the SCSS override.
Everything else (Gemfile, workflow, default tabs, etc.) comes from the
template at first-run.

---

## First-time setup

### 1. Create the repo from chirpy-starter

While signed in as `chisomonuegbu` on GitHub:

1. Go to <https://github.com/cotes2020/chirpy-starter>
2. Click **Use this template → Create a new repository**
3. Name the repo `chisomonuegbu.github.io` (this exact name is required for
   user-level GitHub Pages)
4. Set it to **Public** (required on the GitHub Free tier for Pages)
5. Click **Create repository from template**

### 2. Copy the customization files into the new repo

Clone the new repo locally and copy in (or commit directly via the GitHub web
UI) these four files from this scaffold:

- `_config.yml`           → overwrites the starter's
- `_data/contact.yml`     → new file
- `_tabs/about.md`        → overwrites the starter's
- `assets/css/jekyll-theme-chirpy.scss` → new file

Push to `main`. The shipped GitHub Actions workflow (`.github/workflows/pages-deploy.yml`)
will build the site and publish it to GitHub Pages within ~2–3 minutes.

### 3. Enable GitHub Pages

In the repo's **Settings → Pages**:

- **Source**: GitHub Actions
- (Custom domain field is left blank for now — we'll fill it in step 5.)

You should now have a working site at `https://chisomonuegbu.github.io`.

### 4. Buy the domain

Buy `chisomonuegbu.dev` on **Cloudflare Registrar** (`dash.cloudflare.com`).
Auto-renew on. WHOIS privacy is free and on by default.

### 5. Point the domain at GitHub Pages

#### a. Add the custom domain on GitHub *first*

In repo **Settings → Pages → Custom domain**: enter `chisomonuegbu.dev`,
click **Save**. (Adding here before configuring DNS prevents subdomain
hijacking.) GitHub will create a `CNAME` file in the repo root.

#### b. Configure DNS in Cloudflare

In the Cloudflare dashboard for `chisomonuegbu.dev`, **DNS → Records**:

```
Type   Name    Content              Proxy
A      @       185.199.108.153      DNS only (gray cloud)
A      @       185.199.109.153      DNS only
A      @       185.199.110.153      DNS only
A      @       185.199.111.153      DNS only
CNAME  www     chisomonuegbu.github.io   DNS only
```

**Important**: keep the proxy *off* (gray cloud) for the first hour or so
while GitHub provisions the Let's Encrypt certificate. You can turn proxy on
later if you want Cloudflare's CDN/analytics in front of GitHub Pages.

#### c. Wait for HTTPS

DNS propagates in 5–60 minutes. Then in GitHub **Settings → Pages**, the
**Enforce HTTPS** checkbox becomes available — tick it. GitHub provisions
a Let's Encrypt cert automatically.

After this: <https://chisomonuegbu.dev> should resolve to the site.
`https://www.chisomonuegbu.dev` automatically redirects to the apex.

### 6. SSH key for the new account

You're already running multi-account SSH. Generate a key for this account
and add a host block to `~/.ssh/config`:

```bash
ssh-keygen -t ed25519 -C "chisom@chisomonuegbu.dev" -f ~/.ssh/id_ed25519_chisom
```

Add the public key (`~/.ssh/id_ed25519_chisom.pub`) to GitHub: avatar →
**Settings → SSH and GPG keys → New SSH key**.

In `~/.ssh/config`, append:

```
Host github.com-chisom
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_chisom
  IdentitiesOnly yes
```

Clone the blog repo with the per-account host:

```bash
git clone git@github.com-chisom:chisomonuegbu/chisomonuegbu.github.io.git
```

In the local clone, set the per-repo identity so commits attribute to the
right account:

```bash
cd chisomonuegbu.github.io
git config user.name  "Chisom Onuegbu"
git config user.email "chisom@chisomonuegbu.dev"   # or the GitHub no-reply address
```

---

## Local development

Requires Ruby 3.1+, Bundler, and Node.js (for the asset pipeline).

```bash
# install dependencies
bundle install

# serve locally with live reload at http://127.0.0.1:4000
bundle exec jekyll serve --livereload

# build for production
JEKYLL_ENV=production bundle exec jekyll build
```

Posts go in `_posts/` named `YYYY-MM-DD-slug.md` with this front matter:

```yaml
---
title: Migrating Vaultwarden behind WireGuard
date: 2026-05-12 09:00:00 -0400
categories: [Deep Dives]
tags: [homelab, security, wireguard, vaultwarden]
image:
  path: /assets/img/posts/vaultwarden-wireguard.png
  alt: Network diagram of the WireGuard-fronted Vaultwarden stack
---

Post body in Markdown here.
```

The three categories used on this site are: **Deep Dives**, **Notes**,
**Postmortems**. Stick to these — additional categories dilute the site's
shape.

Tags are open-ended. Keep them lowercase, kebab-case (e.g. `month-end-close`,
`github-actions`, not `Month End Close`). Reuse existing tags before
inventing new ones.

---

## Adding self-hosted Umami analytics later

When you're ready to wire up analytics (recommended after the site has been
live a few weeks), self-host Umami on Contabo behind Caddy:

1. **Deploy Umami via Docker Compose** (one of your existing patterns)
   alongside Postgres on the VPS. The official compose file is at
   <https://umami.is/docs/install>.
2. **Caddy reverse-proxy** at `analytics.chisomonuegbu.dev`. Add an A record
   in Cloudflare for `analytics` → your VPS IP, then a Caddy site block.
   Wildcard TLS via your existing Route53/DNS-01 setup keeps the subdomain
   out of CT logs.
3. **Add the site** in Umami's admin UI. It generates a website ID.
4. **Configure Chirpy** — fill in `_config.yml`:
   ```yaml
   analytics:
     umami:
       id: "<website-id-from-umami>"
       domain: "analytics.chisomonuegbu.dev"
   ```
5. **Push.** Chirpy ships with Umami support out of the box; the snippet is
   injected into `<head>` on every page when `analytics.umami.id` is set.

Add the new endpoint to your Uptime Kuma instance and point Prometheus/Alloy
at the Umami container's metrics if it exposes them.

---

## Adding comments later

Deferred for v1. When there's an actual audience and comments would add
value, the recommended path is:

- **Giscus** (uses GitHub Discussions on this repo). Lowest ops burden.
  Enable Discussions, install the Giscus app, fill in the `comments.giscus`
  block in `_config.yml`, set `comments.provider: giscus`, and commit.
- **Self-hosted alternative**: Remark42 in Docker on Contabo, behind Caddy
  at `comments.chisomonuegbu.dev`. More control, more work, more attack
  surface. Only worth it if there's a specific reason to avoid GitHub
  Discussions.

---

## Useful links

- Chirpy theme repo: <https://github.com/cotes2020/jekyll-theme-chirpy>
- Chirpy demo / docs: <https://chirpy.cotes.page/>
- chirpy-starter template: <https://github.com/cotes2020/chirpy-starter>
- GitHub Pages docs: <https://docs.github.com/en/pages>
- Cloudflare Registrar: <https://dash.cloudflare.com>
- Umami docs: <https://umami.is/docs>

---

## Roadmap

This scaffold is **Phase 1**. The site looks intentional and ships fast.

- **Phase 2** — float the right-side TOC; further refine the left sidebar.
- **Phase 3** — deliberate dark mode (Maggie Appleton / Linear changelog
  flavored, not a flat inversion of light mode).
- **Phase 4** — hand-drawn Excalidraw assets: hero images on Deep Dives,
  custom favicon and avatar, possibly a custom 404 page.

Each phase ships a working site. Don't move to phase N+1 without phase N
being good.
