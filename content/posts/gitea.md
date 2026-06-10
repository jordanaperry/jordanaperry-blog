---
title: "Self-Hosted Git with Gitea"
date: 2026-06-07
draft: false
tags: ["homelab", "git", "gitea", "self-hosted", "proxmox", "github"]
series: ["Homelab Journey"]
description: "Building a full observability stack across all homelab containers — node_exporter on every host, Prometheus scraping every 15 seconds, and Grafana dashboards with Keycloak SSO."
cover:
  image: "/images/banners/banner-post-16.svg"
  alt: "Self-Hosted Git with Gitea"
  relative: false
---

Gitea is the Git server everything in the lab runs through. Terraform configs, blog posts, homelab documentation — it all lives in Gitea first. Selected repos then mirror automatically to GitHub for the public portfolio. The separation keeps internal work private while still maintaining a visible presence for job searching.

This post covers the installation, the web installer workaround, the Nginx reverse proxy setup, and how the GitHub mirroring is configured.

---

## Why self-host Git

The practical reasons: no rate limits, no external dependency for CI/CD pipelines, webhooks that can fire to internal private IPs that GitHub could never reach. For a homelab running an auto-deploy blog pipeline, Gitea's ability to webhook to `192.168.30.10:9000` (an internal DMZ address) is specifically why it's here rather than relying on GitHub.

---

## The container

CT 103 on `leviathan`, Debian 12, VLAN 40 (midnight):

```
IP: 192.168.40.33/24
Gateway: 192.168.40.1
Hostname: gitea
Run as: git user
Gitea version: 1.22.6
```

---

## Installation

Install dependencies and create the git user:

```bash
apt update && apt install git nginx -y
adduser --system --shell /bin/bash --gecos 'Git Version Control' \
  --group --disabled-password --home /home/git git
```

Download the Gitea binary:

```bash
wget -O /usr/local/bin/gitea \
  https://dl.gitea.com/gitea/1.22.6/gitea-1.22.6-linux-amd64
chmod +x /usr/local/bin/gitea
```

Create the directory structure:

```bash
mkdir -p /etc/gitea /var/lib/gitea/{custom,data,log}
chown -R git:git /var/lib/gitea /etc/gitea
chmod 750 /etc/gitea
```

---

## Bypassing the web installer

The standard Gitea setup tells you to visit the web UI to complete installation. On a fresh container with a slow M.2 disk, the installer can hang indefinitely — the database initialisation times out and the UI just spins.

The fix is to pre-create `app.ini` with `INSTALL_LOCK = true`, which skips the web installer entirely, then create the admin account via CLI.

Minimal `/etc/gitea/app.ini`:

```ini
[server]
DOMAIN           = gitea.jordanaperry.com
HTTP_PORT        = 3000
ROOT_URL         = https://gitea.jordanaperry.com/
DISABLE_SSH      = false
SSH_PORT         = 22
OFFLINE_MODE     = false

[database]
DB_TYPE = sqlite3
PATH    = /var/lib/gitea/data/gitea.db

[security]
INSTALL_LOCK       = true
SECRET_KEY         = generate-a-random-string-here
INTERNAL_TOKEN     = generate-another-random-string-here

[webhook]
ALLOWED_HOST_LIST = 192.168.30.10
```

The `ALLOWED_HOST_LIST` under `[webhook]` is critical — without it, Gitea refuses to fire webhooks to private IP addresses and you'll spend time wondering why the blog deploy pipeline isn't triggering.

Create the admin user via CLI:

```bash
su -s /bin/bash git -c "/usr/local/bin/gitea admin user create \
  --username jordanp \
  --password yourpassword \
  --email your@email.com \
  --admin \
  --config /etc/gitea/app.ini"
```

Create a systemd service at `/etc/systemd/system/gitea.service`:

```ini
[Unit]
Description=Gitea
After=network.target

[Service]
User=git
Group=git
WorkingDirectory=/var/lib/gitea
ExecStart=/usr/local/bin/gitea web --config /etc/gitea/app.ini
Restart=always
Environment=USER=git HOME=/home/git GITEA_WORK_DIR=/var/lib/gitea

[Install]
WantedBy=multi-user.target
```

```bash
systemctl enable --now gitea
```

![Gitea running at gitea.jordanaperry.com — dashboard view](/images/gitea.png)
*Gitea dashboard — running on midnight, accessible via Nginx reverse proxy*

---

## Nginx reverse proxy

Gitea runs on port 3000 internally. Nginx terminates SSL and proxies to it.

```nginx
server {
    listen 80;
    server_name gitea.jordanaperry.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name gitea.jordanaperry.com;

    ssl_certificate /etc/nginx/ssl/jordanaperry.crt;
    ssl_certificate_key /etc/nginx/ssl/jordanaperry.key;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 300;
        client_max_body_size 100m;
    }
}
```

`proxy_read_timeout 300` — the default 60 seconds is too short for large git pushes. Increase it or git operations will time out mid-push.

`client_max_body_size 100m` — Nginx's default body size limit will reject large pushes. 100MB is comfortable for most repos.

---

## Repository structure

Four repos under the `jordanaperry-homelab` organisation:

| Repo | Purpose | GitHub mirror |
|---|---|---|
| `jordanaperry-blog` | Hugo blog content and config | ✅ public |
| `homelab-terraform` | Terraform infrastructure as code | ✅ public |
| `homelab-configs` | Config files and scripts | ❌ private |
| `homelab-docs` | Documentation | ❌ private |

Public repos mirror to GitHub automatically. Private repos stay internal.

---

## GitHub mirroring

Gitea's built-in push mirror feature keeps public repos in sync with GitHub without any manual effort.

**Setup for each repo:**

1. Create the repo on GitHub (empty, no README)
2. Generate a GitHub personal access token with repo scope
3. In Gitea: **Repository → Settings → Mirror Settings → Push Mirror → Add**
   - Remote URL: `https://github.com/jordanaperry/repo-name.git`
   - Authorization: GitHub username + token
   - Sync interval: every 8 hours (or trigger manually)

![Gitea repository settings showing push mirror to GitHub configured](/images/gitea-push.png)
*Gitea push mirror — homelab-terraform syncing to github.com/jordanaperry*

After the first sync, the GitHub repo shows all commits with the original Gitea timestamps. From a portfolio perspective, everything looks like it came from GitHub — but the source of truth is the self-hosted Gitea instance.

---

## Deploy keys

Services that need to pull from Gitea use deploy keys — SSH keys that have read-only access to a specific repository. The blog server uses one to pull the blog repo:

1. Generate a key on the blog server: `ssh-keygen -t ed25519 -f /root/.ssh/id_ed25519_gitea`
2. In Gitea: **Repository → Settings → Deploy Keys → Add**
3. Paste the public key, read-only access only
4. Add the Gitea host to blog server's SSH known_hosts

The blog server's SSH config at `/root/.ssh/config`:
```
Host gitea.jordanaperry.com
    IdentityFile /root/.ssh/id_ed25519_gitea
    StrictHostKeyChecking accept-new
```

---

## LXC startup hang fix

During initial setup, CT 103 would occasionally hang on `pct start` with a D-state umount process blocking. The fix at the time was:

```bash
rm /run/lock/lxc/pve-config-103.lock
pct start 103
```

The root cause turned out to be the slow M.2 drive in leviathan — the container storage was on `local-lvm` and disk I/O latency was causing the umount process to stall. After migrating CT 103 to `wreck-lvm` (iSCSI on the NAS) the issue stopped entirely.

---

## What's next

Gitea is running, repos are organised, and the GitHub mirror keeps the public portfolio current automatically. Next up: Prometheus and Grafana — building a monitoring dashboard that covers every container and the Proxmox host itself.