---
title: "The Blog Server: Hugo in the DMZ with Auto-Deploy"
date: 2026-06-06
draft: false
tags: ["homelab", "hugo", "nginx", "gitea", "webhook", "proxmox", "dmz"]
series: ["Homelab Journey"]
description: "Setting up the blog server in the twilight DMZ, building the Hugo auto-deploy pipeline via Gitea webhooks, and the gotchas that make this harder than it looks."
cover:
  image: "/images/banners/banner-post-15.svg"
  alt: "The Blog Server: Hugo in the DMZ with Auto-Deploy"
  relative: false
---

The blog you're reading is served from a container running in the twilight DMZ on my home lab. Pushing a commit to Gitea triggers a webhook, the blog server pulls the changes, Hugo rebuilds the site in 72ms, and Nginx serves the result. The whole pipeline from `git push` to live takes about 5 seconds.

This post covers how it's built — the container setup, the Nginx and Hugo configuration, the deploy pipeline, and a collection of gotchas that each cost more time than they should have.

---

## Architecture

```
VS Code (Mac)
    ↓ git push
Gitea (gitea.jordanaperry.com · CT 103 · midnight)
    ↓ webhook POST to 192.168.30.10:9000
webhook listener (blog server · port 9000)
    ↓ runs deploy script
git pull + hugo build (72ms)
    ↓
Nginx serves /var/www/html
    ↓
https://blog.jordanaperry.com (public internet)
```

The blog server lives in the twilight DMZ (VLAN 30) — public-facing, internet reachable, isolated from everything internal. It can reach Gitea (on midnight, VLAN 40) to pull code, and Gitea can reach it to fire the webhook. That's the only cross-VLAN traffic allowed.

---

## The container

CT 101 on `leviathan`, Debian 12, VLAN 30 (twilight):

```
IP: 192.168.30.10/24
Gateway: 192.168.30.1
Storage: wreck-lvm (iSCSI), 20GB
```

One important setting: the container's network firewall must be **disabled** in Proxmox. Leaving it enabled causes VLAN tagging issues that are frustrating to debug — traffic appears to be flowing but the container can't reach its gateway.

---

## Hugo installation

Hugo is not in the Debian apt repos at a recent enough version. PaperMod requires Hugo v0.146.0+, and apt gives you something much older. Install from GitHub releases directly:

```bash
wget https://github.com/gohugoio/hugo/releases/download/v0.147.1/hugo_extended_0.147.1_linux-amd64.deb
dpkg -i hugo_extended_0.147.1_linux-amd64.deb
```

The `extended` variant is required for themes that use SCSS/SASS — PaperMod needs it.

A few Hugo config gotchas:

- Use `hugo.yaml` not `config.yaml` — Hugo v0.147+ changed the default config filename
- `paginate:` is deprecated — use `pagination: pagerSize:` instead
- Hugo binary isn't in root's PATH in all contexts — use the full path `/usr/local/bin/hugo` in scripts

---

## Nginx and SSL

Install Nginx:

```bash
apt install nginx -y
```

The blog uses the `*.jordanaperry.com` wildcard cert from pfSense. Copy the cert and key to the container:

```bash
scp pfSense:/var/etc/acme/jordanaperry-wildcard/fullchain.pem /etc/nginx/ssl/jordanaperry.crt
scp pfSense:/var/etc/acme/jordanaperry-wildcard/privkey.pem /etc/nginx/ssl/jordanaperry.key
```

Nginx config at `/etc/nginx/sites-available/default`:

```nginx
server {
    listen 80;
    server_name blog.jordanaperry.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name blog.jordanaperry.com;

    ssl_certificate /etc/nginx/ssl/jordanaperry.crt;
    ssl_certificate_key /etc/nginx/ssl/jordanaperry.key;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

![Nginx serving blog.jordanaperry.com with green padlock](/images/TODO.png)
*blog.jordanaperry.com — live with trusted SSL*

---

## Port forwards and DNS

The blog needs to be reachable from the public internet. Two things required:

**pfSense port forwards** (WAN → blog server):
```
WAN:80  → 192.168.30.10:80
WAN:443 → 192.168.30.10:443
```

**GoDaddy DNS A record:**
```
blog.jordanaperry.com → <WAN IP>
```

For the WAN IP — home connections change occasionally. pfSense's Dynamic DNS service (Services → Dynamic DNS) handles this using the GoDaddy API. Same credentials as the ACME setup, and pfSense updates the DNS record automatically when the WAN IP changes.

---

## The deploy pipeline

The pipeline has three components: a deploy script, a webhook listener, and a Gitea webhook.

**Deploy script** at `/usr/local/bin/deploy-blog.sh`:

```bash
#!/bin/bash
cd /var/www/jordanaperry
git pull origin main
/usr/local/bin/hugo --destination /var/www/html --minify
```

The blog repo lives at `/var/www/jordanaperry`. Git pulls from Gitea via SSH using a deploy key at `/root/.ssh/id_ed25519_gitea`.

**webhook listener** — a small Go binary that listens for HTTP POST requests and runs a command when a matching request arrives. Install from GitHub releases:

```bash
wget https://github.com/adnanh/webhook/releases/download/2.8.0/webhook-linux-amd64.tar.gz
tar -xzf webhook-linux-amd64.tar.gz
mv webhook-linux-amd64/webhook /usr/local/bin/webhook
```

Config at `/etc/webhook/hooks.json`:

```json
[
  {
    "id": "deploy-blog",
    "execute-command": "/usr/local/bin/deploy-blog.sh",
    "command-working-directory": "/var/www/jordanaperry",
    "trigger-rule": {
      "match": {
        "type": "payload-hmac-sha256",
        "secret": "your-webhook-secret",
        "parameter": {
          "source": "header",
          "name": "X-Gitea-Signature"
        }
      }
    }
  }
]
```

Run webhook as a systemd service on port 9000. When Gitea fires a push webhook, the listener verifies the HMAC signature and runs the deploy script.

![webhook service running — systemctl status webhook showing active](/images/TODO.png)
*systemctl status webhook — listener active on port 9000*

**Gitea webhook** — in the blog repo settings: **Settings → Webhooks → Add Webhook → Gitea**:

```
URL:    http://192.168.30.10:9000/hooks/deploy-blog
Secret: (same secret as webhook config)
Trigger: Push events
```

![Gitea webhook config showing URL and recent successful deliveries](/images/TODO.png)
*Gitea webhook — recent deliveries all green*

---

## Firewall rules for the pipeline

The pipeline crosses VLANs — Gitea is on midnight (40), the blog is on twilight (30). Both directions need rules:

```
# Blog pulls from Gitea
Pass TCP · twilight → 192.168.40.33 · port 22  · Blog SSH to Gitea (git pull)
Pass TCP · twilight → 192.168.40.33 · port 443 · Blog HTTPS to Gitea

# Gitea fires webhook to blog
Pass TCP · 192.168.40.33 → twilight · port 9000 · Gitea webhook to blog
```

---

## Gotchas

**Gitea blocks webhooks to private IPs by default.** Add to `/etc/gitea/app.ini` under `[webhook]`:
```
ALLOWED_HOST_LIST = 192.168.30.10
```
Without this, Gitea silently refuses to fire the webhook and logs "webhook url is not allowed."

**Gitea signature header** is `X-Gitea-Signature`, not `X-Gitea-Signature-256`. The webhook listener won't verify the signature correctly if you use the wrong header name and all deliveries will fail with 403.

**Hugo not in PATH for systemd services.** The webhook runs as a systemd service with a minimal environment. Use the full path `/usr/local/bin/hugo` in the deploy script — `hugo` alone will produce "command not found."

**vmbr0.30 doesn't persist across reboots without a workaround.** Add this to the vmbr0 interface in `/etc/network/interfaces` on leviathan:
```
post-up ip link set vmbr0.30 up
```

---

## What's next

The blog pipeline works end-to-end — write in VS Code, push to Gitea, site rebuilds automatically. Next post: Gitea itself — the self-hosted Git server running on midnight, how GitHub mirroring works for the public portfolio, and setting it up when the web installer hangs.