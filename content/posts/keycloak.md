---
title: "Running Keycloak at Home: SSO for Your Homelab"
date: 2026-06-10
draft: false
tags: ["homelab", "keycloak", "iam", "sso", "oidc", "identity", "proxmox"]
series: ["Homelab Journey"]
description: "Installing Keycloak 26 on Proxmox, connecting it to FreeIPA via LDAP, and wiring up Grafana and Proxmox SSO — plus the FreeIPA OTP incident that took down everything."
cover:
  image: "/images/banners/banner-post-20.svg"
  alt: "Running Keycloak at Home: SSO for Your Homelab"
  relative: false
---

Every internal service in the lab had its own login. Grafana had one password, Proxmox had another, Gitea had a third. No consistency, no central control, no audit trail. This is exactly the problem IAM exists to solve — and Keycloak is how I solved it at home.

Keycloak is an open-source identity provider that implements the same protocols used in enterprise IAM: OpenID Connect, SAML 2.0, OAuth2, and LDAP federation. Getting it running means learning the actual technology behind Okta, Azure AD, and Google Workspace — not a simplified version, the real thing.

This post covers the installation, the FreeIPA LDAP federation setup, wiring up Grafana and Proxmox as OIDC clients, and the incident where enabling FreeIPA OTP took down every SSO integration simultaneously.

---

## What Keycloak gives you

- **Single sign-on** — one login for every integrated service
- **OIDC and SAML** — the protocols behind modern enterprise SSO
- **TOTP MFA** — time-based one-time passwords, same as Google Authenticator
- **LDAP federation** — sync users from FreeIPA (or Active Directory) automatically
- **RBAC** — role-based access control with realm roles and client-specific roles
- **Multi-tenancy** — realms isolate different identity domains (useful for learning)

---

## The container

CT 104 on `leviathan`, Debian 12, VLAN 40 (midnight):

```
Hostname: keycloak
IP: 192.168.40.30/24
Gateway: 192.168.40.1
Storage: wreck-lvm (iSCSI), 20GB
RAM: 2GB minimum — Keycloak's JVM is memory-hungry
```

Provisioned via Terraform. The `nesting = true` feature is required in the container config.

---

## Installation

### Java

Keycloak 26 requires Java 17+. Java 21 isn't in Debian 12's repos — use Java 17 from backports:

```bash
echo "deb http://deb.debian.org/debian bookworm-backports main" >> /etc/apt/sources.list
apt update
apt install -y openjdk-17-jdk-headless
```

### Keycloak binary

```bash
cd /opt
wget https://github.com/keycloak/keycloak/releases/download/26.2.4/keycloak-26.2.4.tar.gz
tar -xzf keycloak-26.2.4.tar.gz
mv keycloak-26.2.4 keycloak
rm keycloak-26.2.4.tar.gz
useradd -r -s /sbin/nologin keycloak
chown -R keycloak:keycloak /opt/keycloak
```

---

## SSL certificate

Copy the wildcard cert from pfSense:

```bash
# On Mac
scp admin@192.168.1.1:/var/etc/cert.crt /tmp/keycloak.crt
scp admin@192.168.1.1:/var/etc/cert.key /tmp/keycloak.key
scp /tmp/keycloak.crt root@192.168.40.30:/etc/ssl/keycloak.crt
scp /tmp/keycloak.key root@192.168.40.30:/etc/ssl/keycloak.key

# On keycloak container
chown root:keycloak /etc/ssl/keycloak.crt /etc/ssl/keycloak.key
chmod 640 /etc/ssl/keycloak.crt /etc/ssl/keycloak.key
```

---

## Configuration

`/opt/keycloak/conf/keycloak.conf`:

```
https-certificate-file=/etc/ssl/keycloak.crt
https-certificate-key-file=/etc/ssl/keycloak.key
https-port=8443
http-enabled=true
hostname=keycloak.jordanaperry.com
```

---

## Systemd service

`/etc/systemd/system/keycloak.service`:

```ini
[Unit]
Description=Keycloak IAM
After=network.target

[Service]
User=keycloak
Group=keycloak
Environment=KC_BOOTSTRAP_ADMIN_USERNAME=admin
Environment=KC_BOOTSTRAP_ADMIN_PASSWORD=your-password
ExecStart=/opt/keycloak/bin/kc.sh start
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable --now keycloak
```

The first start takes 60–90 seconds — Keycloak rebuilds its Quarkus configuration. Don't panic and restart it early.

Two important distinctions: always use `kc.sh start` (production mode), never `start-dev`. Development mode uses an in-memory H2 database and HTTP only — nothing persists across restarts.

If Keycloak fails with "database is read only" on first start, it means you ran `start-dev` as root during testing and the H2 data files are root-owned. Fix: `chown -R keycloak:keycloak /opt/keycloak`.

![Keycloak admin console — homelab realm dashboard](/images/keycloak.png)
*Keycloak admin console at keycloak.jordanaperry.com:8443 — homelab realm*

---

## Initial setup

After the first successful start, access `https://keycloak.jordanaperry.com:8443/admin` and log in with the bootstrap credentials.

**Create the homelab realm** — never work in the `master` realm. The master realm is for Keycloak administration only. Create a dedicated `homelab` realm for all lab services.

One gotcha: always verify the active realm in the top-left dropdown before creating clients or users. It's easy to accidentally create something in `master` and spend time debugging why it can't be found in `homelab`.

**Enable TOTP MFA** — in the homelab realm: **Authentication → Required Actions → Configure OTP → Default On**. Users will be prompted to enroll on first login.

**Add a client mapper** — Keycloak's OIDC tokens don't include realm roles by default in a way that most integrations can read. Add a mapper to move roles to the top-level `roles` claim: **Clients → [any client] → Client scopes → roles → Mappers → Add mapper → User Realm Role → Token Claim Name: `roles`**.

Without this mapper, role-based access in Grafana won't work — the `role_attribute_path` JMESPath expression can't traverse the nested `realm_access.roles` object.

---

## FreeIPA LDAP federation

Keycloak can sync users directly from FreeIPA using LDAP federation, so you maintain one set of user accounts and everything else inherits from it.

**homelab realm → User Federation → Add provider → LDAP**

| Field | Value |
|---|---|
| Vendor | Red Hat Directory Server |
| Connection URL | `ldap://192.168.40.31` |
| Bind DN | `uid=keycloak-bind,cn=users,cn=accounts,dc=jordanaperry,dc=com` |
| Bind credentials | Bitwarden: `FreeIPA Keycloak Bind` |
| Edit mode | READ_ONLY |
| Users DN | `cn=users,cn=accounts,dc=jordanaperry,dc=com` |
| Username LDAP attribute | `uid` |
| UUID LDAP attribute | `ipaUniqueID` |

After saving, click **Synchronize all users**. FreeIPA users appear in Keycloak's user list.

**Fix the first name mapper** — by default Keycloak maps `cn` to the first name field. In FreeIPA, `cn` is the full common name, not just the first name. Edit the `first name` mapper and change the LDAP attribute from `cn` to `givenName`.

![Keycloak User Federation page showing FreeIPA LDAP connected and synced](/images/keycloak-ldap.png)
*User Federation — FreeIPA LDAP connected, users synchronized*

---

## Grafana OIDC integration

### Keycloak client setup

**homelab realm → Clients → Create client**

| Field | Value |
|---|---|
| Client ID | `grafana` |
| Client authentication | On |
| Standard flow | On |
| Root URL | `https://monitor.jordanaperry.com` |
| Valid redirect URIs | `https://monitor.jordanaperry.com/login/generic_oauth` |

Copy the client secret from the **Credentials** tab and store it in Bitwarden.

### Grafana configuration

In `/etc/grafana/grafana.ini`:

```ini
[server]
domain = monitor.jordanaperry.com
root_url = https://monitor.jordanaperry.com

[auth.generic_oauth]
enabled = true
name = Keycloak
allow_sign_up = true
skip_org_role_sync = false
allow_assign_grafana_admin = true
client_id = grafana
client_secret = your-client-secret
scopes = openid email profile
auth_url = https://keycloak.jordanaperry.com:8443/realms/homelab/protocol/openid-connect/auth
token_url = https://keycloak.jordanaperry.com:8443/realms/homelab/protocol/openid-connect/token
api_url = https://keycloak.jordanaperry.com:8443/realms/homelab/protocol/openid-connect/userinfo
role_attribute_path = contains(roles[*], 'admin') && 'GrafanaAdmin' || 'Viewer'
```

Two things worth calling out:

**`root_url` is required.** Without it, Grafana sends `localhost:3000` as the OAuth redirect URI, which Keycloak rejects. The `domain` field alone isn't enough.

**`GrafanaAdmin` not `Admin`.** In Grafana 13, `Admin` maps to organisation admin only. `GrafanaAdmin` maps to server admin, which also grants org admin. Using `Admin` in the role mapping path silently gives the Viewer role instead — no error, just wrong permissions.

![Grafana login page showing Sign in with Keycloak button](/images/grafana-signon.png)
*Grafana login — Keycloak SSO button alongside local login*

---

## Proxmox OIDC integration

### Keycloak client setup

| Field | Value |
|---|---|
| Client ID | `proxmox` |
| Client authentication | On |
| Root URL | `https://leviathan.jordanaperry.com:8006` |
| Valid redirect URIs | `https://leviathan.jordanaperry.com:8006/*` |

### Proxmox configuration

**Datacenter → Permissions → Realms → Add → OpenID Connect**

| Field | Value |
|---|---|
| Issuer URL | `https://keycloak.jordanaperry.com:8443/realms/homelab` |
| Realm | `keycloak` |
| Client ID | `proxmox` |
| Client Key | from Bitwarden |
| Username Claim | `username` |
| Auto Create Users | on |

Then assign permissions: **Datacenter → Permissions → Add → User Permission**
- Path: `/`
- User: `jordanp@keycloak`
- Role: `Administrator`
- Propagate: on

Log in via Keycloak first to auto-create the user in Proxmox, then assign the permissions as root. The user doesn't exist until the first Keycloak login.

![Proxmox login page showing Keycloak realm option in the dropdown](/images/proxmox-realm.png)
*Proxmox login — realm dropdown showing keycloak as an option*

---

## The FreeIPA OTP incident

This one took down every SSO integration at once and is worth documenting in detail.

**What happened:** I enabled OTP on the FreeIPA user account with `ipa user-mod jordanp --user-auth-type=otp`. The intent was to add MFA at the FreeIPA level. What I didn't know: enabling FreeIPA OTP changes how the LDAP bind process works — external LDAP clients (like Keycloak) can no longer authenticate the user via a simple password bind. Keycloak started returning `ldap_bind: Invalid credentials (49)` for every authentication attempt.

Every SSO login broke simultaneously — Grafana, Proxmox, everything.

FreeIPA also has a password minimum lifetime policy that defaults to 1 hour. While trying to fix this I reset the password multiple times, and each reset was rejected because the previous password hadn't been in use for an hour yet.

**Fix sequence:**

```bash
# Disable FreeIPA OTP — back to password auth only
ipa user-mod jordanp --user-auth-type=password

# Remove password minimum lifetime
ipa pwpolicy-mod --minlife=0

# Reset password
ipa passwd jordanp
```

Then in Keycloak: re-assign the `admin` realm role to `jordanp` — it gets wiped when the user is deleted and re-synced from FreeIPA. In Grafana: delete the `jordanp` user and let it recreate on next login.

**The lesson:** Keep OTP in Keycloak, not FreeIPA, unless you configure Kerberos on the Keycloak container. Keycloak handles MFA for the services it protects — FreeIPA OTP is for FreeIPA's own web UI and native Kerberos flows.

---

## What's next

Keycloak is running, SSO is working across Grafana and Proxmox, and LDAP federation keeps users in sync with FreeIPA. That wraps up the IAM stack posts for now — HashiCorp Vault and Okta integration are on the roadmap as the lab continues to grow.