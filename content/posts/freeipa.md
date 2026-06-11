---
title: "FreeIPA: Building an Enterprise-Style Directory Service at Home"
date: 2026-06-09
draft: false
tags: ["homelab", "freeipa", "ldap", "kerberos", "iam", "identity", "proxmox"]
series: ["Homelab Journey"]
description: "Installing FreeIPA on Rocky Linux in Proxmox, setting up users and groups, HBAC rules, sudo policies, and why FreeIPA is closer to Active Directory than plain OpenLDAP."
cover:
  image: "/images/banners/banner-post-19.svg"
  alt: "FreeIPA: Building an Enterprise-Style Directory Service at Home"
  relative: false
---

Every enterprise IAM stack has a directory service at its foundation. In Microsoft environments that's Active Directory. In cloud-native stacks it's often Azure AD or Okta Universal Directory. In this lab it's FreeIPA — an open source identity management platform that bundles LDAP, Kerberos, DNS, a certificate authority, and sudo/HBAC policy management into one cohesive system.

FreeIPA is the closest open source analog to Active Directory. Running it at home gives genuine hands-on experience with the concepts that underpin enterprise identity: directory structure, Kerberos authentication, group policy, and user lifecycle management. It's also the identity source that feeds Keycloak SSO and Vault LDAP auth — the single source of truth for who exists in the lab.

---

## Why FreeIPA over plain OpenLDAP

OpenLDAP is just the directory layer — it stores users and groups and speaks the LDAP protocol. That's useful, but it's only one piece of what Active Directory does.

FreeIPA adds Kerberos (the authentication protocol Active Directory uses), a built-in DNS server, an internal certificate authority, host-based access control (HBAC) rules, sudo policy management, and a web UI. It's a full identity management platform, not just a user store. The experience of running it maps much more directly to what an IAM engineer would encounter in an enterprise environment.

---

## The container

CT 106 on `leviathan`, **Rocky Linux 9**, VLAN 40 (midnight):

```
Hostname: ipa.jordanaperry.com
IP: 192.168.40.31/24
Storage: wreck-lvm (iSCSI), 40GB
RAM: 4GB — FreeIPA needs it
```

Rocky Linux rather than Debian — FreeIPA is a Red Hat project and the packages are maintained for RHEL-family distributions. Rocky Linux 9 is the closest free RHEL equivalent.

One immediate difference from Debian-based containers: **Rocky Linux 9 has no SSH server in the default LXC template.** Install it first or you'll be stuck using `pct enter` to get in:

```bash
dnf install -y openssh-server
systemctl enable --now sshd
```

---

## Pre-installation setup

FreeIPA is strict about its hostname. The FQDN must resolve correctly before the installer runs — if it doesn't, the install fails with a confusing error.

```bash
# Set FQDN
hostnamectl set-hostname ipa.jordanaperry.com

# Fix /etc/hosts — Proxmox sets the short name only
# Change: 192.168.40.31  ipa
# To:     192.168.40.31  ipa.jordanaperry.com  ipa
nano /etc/hosts

# Verify — must return the full FQDN
hostname -f
# ipa.jordanaperry.com ✅
```

---

## Installation

```bash
dnf update -y
dnf install -y freeipa-server freeipa-server-dns

ipa-server-install \
  --domain=jordanaperry.com \
  --realm=JORDANAPERRY.COM \
  --hostname=ipa.jordanaperry.com \
  --ip-address=192.168.40.31 \
  --no-ntp \
  --no-forwarders \
  --setup-dns \
  --auto-reverse \
  --unattended \
  --ds-password=<dir-manager-password> \
  --admin-password=<admin-password>
```

Takes 5–10 minutes. A successful install ends with `The ipa-server-install command was successful`.

Two passwords to save in Bitwarden:
- **Directory Manager** — low-level LDAP admin, used for break-glass access
- **admin** — FreeIPA's admin account for day-to-day operations

```bash
# Verify with a Kerberos ticket
kinit admin
klist
# Credentials cache: ...
# Default principal: admin@JORDANAPERRY.COM ✅
```

The web UI is at `https://ipa.jordanaperry.com` — FreeIPA uses its own internal CA so expect a browser certificate warning. This is expected and fine for a homelab.

![FreeIPA web UI showing the admin dashboard](/images/freeipa.png)
*FreeIPA web UI — Users, Groups, Hosts, Policies all in one place*

---

## Creating users

Always `kinit admin` before running `ipa` commands — FreeIPA uses Kerberos for CLI auth and commands fail with permission errors if you don't have a valid ticket.

```bash
kinit admin

# Create primary user
ipa user-add jordanp \
  --first=Jordan \
  --last=Perry \
  --email=jordan@jordanaperry.com \
  --password

# Create Keycloak service account (bind user for LDAP federation)
ipa user-add keycloak-bind \
  --first=Keycloak \
  --last=Bind \
  --password
```

The `keycloak-bind` account is a read-only service account that Keycloak uses to authenticate to FreeIPA's LDAP directory. It should have minimal permissions — just enough to read users and groups.

---

## Groups, HBAC, and sudo

FreeIPA's HBAC (Host-Based Access Control) rules determine which users can log in to which hosts. By default, `allow_all` is enabled — any FreeIPA user can log in anywhere. For a homelab this is fine initially, but tightening it is a good exercise.

```bash
# Create groups
ipa group-add homelab-admins --desc="Homelab administrators"
ipa group-add homelab-users --desc="Homelab standard users"
ipa group-add-member homelab-admins --users=jordanp

# Create a targeted HBAC rule
ipa hbacrule-add homelab-admin-access \
  --desc="Allow homelab admins to access all hosts"
ipa hbacrule-add-user homelab-admin-access --groups=homelab-admins
ipa hbacrule-add-host homelab-admin-access --hostcat=all
ipa hbacrule-add-service homelab-admin-access --servicecat=all

# Disable the catch-all rule
ipa hbacrule-disable allow_all
```

Sudo policy works the same way — define a rule, assign it to a group, and FreeIPA distributes it to enrolled hosts:

```bash
ipa sudorule-add homelab-admin-sudo \
  --desc="Allow homelab admins full sudo"
ipa sudorule-add-user homelab-admin-sudo --groups=homelab-admins
ipa sudorule-mod homelab-admin-sudo --hostcat=all --cmdcat=all
ipa sudorule-add-option homelab-admin-sudo --sudooption='!authenticate'
```

The `!authenticate` option means sudo doesn't prompt for a password — authentication was already handled by Kerberos at login time.

---

## How it fits in the stack

FreeIPA is the identity foundation everything else builds on:

```
FreeIPA (LDAP + Kerberos)
    └── Keycloak (LDAP federation → OIDC broker)
            ├── Grafana (OIDC) ✅
            ├── Proxmox (OIDC) ✅
            └── other services
    └── Vault (LDAP auth)
            └── SSH CA signing ✅
```

FreeIPA is the single source of truth for users. Keycloak federates from it and exposes users to modern apps via OIDC. Vault authenticates directly against it for secrets access. This mirrors the enterprise pattern of Active Directory as the identity foundation with Okta or Azure AD sitting in front for modern protocol support.

---

## A note on SSL

I tried replacing FreeIPA's default self-signed certificate with the wildcard `*.jordanaperry.com` cert. It worked for the web UI but broke the `ipa` CLI tools — FreeIPA is tightly coupled to its own internal CA and replacing the Apache httpd certificate breaks the CLI's certificate validation.

The self-signed cert warning in the browser is the right tradeoff for a homelab. FreeIPA's internal CA is a legitimate CA — it just isn't trusted by your browser by default. You can import the FreeIPA CA cert into your browser to remove the warning, but it's not worth the maintenance overhead.

---

## Known gotchas

**FQDN in `/etc/hosts` is required** — Proxmox sets the short hostname only. FreeIPA's installer requires the FQDN to resolve before it runs. Fix `/etc/hosts` before installing or the install will fail.

**`kinit admin` before every CLI session** — FreeIPA uses Kerberos for CLI authentication. Running `ipa` commands without a valid ticket produces permission errors, not "please authenticate" messages. Always `kinit admin` first.

**Rocky Linux uses `sshd` not `ssh`** — the service name differs from Debian. `systemctl restart sshd` not `systemctl restart ssh`.

**`dnf` not `apt`** — Rocky Linux uses the RPM package manager. Every `apt install` command from the Debian-based posts becomes `dnf install`.

**Password minimum lifetime** — FreeIPA's default password policy has a 1-hour minimum lifetime. If you reset a password and immediately try to reset it again, it's rejected. For a homelab, set it to zero:

```bash
ipa pwpolicy-mod --minlife=0
```

**Don't enable FreeIPA OTP if using Keycloak LDAP federation** — covered in detail in the Keycloak post, but worth repeating here: enabling `--user-auth-type=otp` on a FreeIPA user breaks Keycloak's ability to authenticate them via LDAP bind. Keep OTP in Keycloak, not FreeIPA.

---

## What's next

FreeIPA is running as the identity foundation for the lab — users, groups, HBAC rules, and sudo policies all defined in one place. The next post covers Keycloak, which sits in front of FreeIPA and exposes its users to modern applications via OIDC and SAML.