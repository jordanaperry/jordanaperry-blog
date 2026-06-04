---
title: "SSL Certificates for Your Homelab with Let's Encrypt and pfSense ACME"
date: 2026-06-04
draft: false
tags: ["homelab", "ssl", "letsencrypt", "pfsense", "acme", "dns"]
series: ["Homelab Journey"]
description: "Getting a real wildcard SSL certificate for every internal service using pfSense's ACME package and GoDaddy DNS-01 challenge — no port forwards required."
cover:
  image: "/images/banners/banner-post-13.svg"
  alt: "SSL Certificates for Your Homelab with Let's Encrypt and pfSense ACME"
  relative: false
---

Every internal service in the lab was showing browser security warnings. The pfSense GUI, Proxmox, Unifi, Gitea — all of them either had self-signed certs or no SSL at all. Every time you open a self-signed cert page you either click through a warning or add a permanent exception, and neither of those is a good habit.

The fix is a real certificate from Let's Encrypt. They're free, trusted by every browser, and auto-renew. For a homelab, the cleanest approach is a wildcard cert — one cert that covers every subdomain — obtained via DNS-01 challenge so Let's Encrypt never needs to reach any of your internal services directly.

---

## How DNS-01 works

The standard Let's Encrypt challenge (HTTP-01) requires Let's Encrypt to reach your server over port 80 to verify you own the domain. That means exposing a port to the internet, which defeats the whole point of a segmented homelab.

DNS-01 works differently. Let's Encrypt gives you a token to place as a DNS TXT record. It then queries your public DNS to verify the record is there. You prove domain ownership through DNS, not through a reachable server. No port forwards needed.

pfSense's ACME package can automate this entirely using your domain registrar's API — in my case GoDaddy. It creates the TXT record, waits for verification, removes it, and stores the cert. Renewal is automatic.

---

## The cert — what was actually implemented

The original plan was `*.lab.jordanaperry.com`. After working through it, I went with `*.jordanaperry.com` directly — one cert that covers everything including the blog and any future public-facing services.

| Field | Value |
|---|---|
| Cert name | jordanaperry-wildcard |
| Domains covered | `*.jordanaperry.com` + `jordanaperry.com` |
| Key type | 384-bit ECDSA |
| Challenge method | DNS-GoDaddy |
| Auto-renewal | Every 90 days via ACME cron |

One note on key type: I used ECDSA (384-bit) rather than RSA. It's more modern and produces smaller certs. However, the Synology NAS DSM doesn't support ECC certificates, so the NAS was left on its default self-signed cert for now.

---

## Step 1 — GoDaddy API key

**GoDaddy account → Settings → Developer → API Keys → Create New Key**

You get a **Key** and a **Secret**. Store both in Bitwarden immediately — the secret is only shown once. These credentials give full DNS control over your domain, so treat them like a password.

---

## Step 2 — install the ACME package

**System → Package Manager → Available Packages** → search `acme` → install.

After install, ACME appears under **Services → ACME Certificates**.

![System → Package Manager showing ACME package installed](/images/TODO.png)
*Package Manager — ACME installed*

---

## Step 3 — create an ACME account

**Services → ACME Certificates → Account Keys → Add**

| Field | Value |
|---|---|
| Name | letsencrypt-prod |
| ACME Server | Let's Encrypt Production ACME v2 |
| Email | your@email.com |

Click **Create new account key** then **Register ACME account key**. This registers your email with Let's Encrypt's CA — they'll email you when certs are expiring.

![ACME Account Keys page showing letsencrypt-prod registered](/images/TODO.png)
*Services → ACME Certificates → Account Keys — letsencrypt-prod registered*

---

## Step 4 — create the wildcard certificate

**Services → ACME Certificates → Certificates → Add**

| Field | Value |
|---|---|
| Name | jordanaperry-wildcard |
| Account Key | letsencrypt-prod |
| Key type | ECDSA 384 |

Add two domain entries — you need both for the cert to cover the apex domain and all subdomains:

| Domain | Method |
|---|---|
| `jordanaperry.com` | DNS-GoDaddy |
| `*.jordanaperry.com` | DNS-GoDaddy |

For each domain entry, enter your GoDaddy API Key and Secret in the credentials fields.

Set the **Actions list** (post-renewal actions) to restart the pfSense web GUI:
```
/etc/rc.restart_webgui
```

This ensures pfSense picks up the new cert automatically after each renewal.

Click **Issue/Renew**. pfSense will call the GoDaddy API, create a TXT record, wait for Let's Encrypt to verify it, then remove the record and store the cert. The whole process takes about 60 seconds.

![ACME Certificates page showing jordanaperry-wildcard issued successfully](/images/TODO.png)
*Services → ACME Certificates — wildcard cert issued*

---

## Step 5 — apply the cert to pfSense

**System → Advanced → Admin Access → SSL/TLS Certificate** → select `jordanaperry-wildcard` → Save.

pfSense restarts its web interface. Access it at `https://krakenfw.jordanaperry.com:8443` — green padlock, no warnings.

![pfSense GUI showing green padlock with jordanaperry.com certificate](/images/TODO.png)
*pfSense login page — trusted SSL cert, no warnings*

---

## Step 6 — DNS host overrides for the new hostnames

Add these in **Services → DNS Resolver → Host Overrides** so the new hostnames resolve internally:

| Host | Domain | IP |
|---|---|---|
| krakenfw | jordanaperry.com | 192.168.1.1 |
| leviathan | jordanaperry.com | 192.168.40.10 |
| unifi | jordanaperry.com | 192.168.40.20 |
| blog | jordanaperry.com | 192.168.30.10 |

Also add to **System → Advanced → Admin Access → Alternate Hostnames**:
```
krakenfw.jordanaperry.com leviathan.jordanaperry.com unifi.jordanaperry.com
```

Without the alternate hostnames entry, pfSense's DNS rebind protection will block access via these hostnames — covered in detail in the DNS post.

---

## Deploying the cert to other services

The wildcard cert lives on pfSense. To use it on other services you copy the cert and key files out and configure each service to use them.

The cert files are at:
```
/var/etc/acme/jordanaperry-wildcard/
  - fullchain.pem
  - privkey.pem
```

For each service — Proxmox, Unifi, Gitea, Grafana, Keycloak, Vault — the procedure is:
1. SCP the cert files to the container
2. Configure the service to use them
3. Restart the service

The procedure varies per service and deserves its own post. The short version: it works, everything is green, and there's a manual re-copy step needed when the cert renews every 90 days until an automation script is written.

---

## Current SSL status

| Service | URL | Status |
|---|---|---|
| pfSense | https://krakenfw.jordanaperry.com:8443 | ✅ |
| Proxmox | https://leviathan.jordanaperry.com:8006 | ✅ |
| Unifi | https://unifi.jordanaperry.com:8443 | ✅ |
| Blog | https://blog.jordanaperry.com | ✅ |
| Gitea | https://gitea.jordanaperry.com | ✅ |
| Grafana | https://monitor.jordanaperry.com | ✅ |
| Keycloak | https://keycloak.jordanaperry.com:8443 | ✅ |
| Vault | https://vault.jordanaperry.com:8200 | ✅ |
| NAS | — | ⏭️ DSM doesn't support ECC |

---

## ⚠️ Renewal deadline: 2026-08-19

The cert auto-renews on pfSense via the ACME cron job. But the copies deployed to other services won't update automatically — they were pushed manually. Before August 19, those services need fresh copies of the renewed cert. A renewal script is on the backlog.

---

## What's next

Every service has a trusted SSL cert. Next up: the bastion host — setting up an SSH jumpbox so all internal SSH access flows through one hardened server with proper audit logging.