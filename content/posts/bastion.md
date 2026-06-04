---
title: "Bastion Host: SSH Access the Right Way"
date: 2026-06-05
draft: false
tags: ["homelab", "security", "ssh", "bastion", "jumpbox", "proxmox"]
series: ["Homelab Journey"]
description: "Building a hardened SSH jumpbox on Proxmox — separate keys per server, ProxyJump config, fail2ban, and iptables rules that enforce the bastion as the only SSH entry point."
cover:
  image: "/images/banners/banner-post-14.svg"
  alt: "Bastion Host: SSH Access the Right Way"
  relative: false
---

Every server in the lab accepts SSH connections. Without any additional controls, each one is an independent attack surface — a separate set of credentials to manage, a separate set of logs to watch, a separate point of failure. The enterprise pattern for this is a bastion host: one hardened server that's the only entry point for SSH, with every other server only reachable through it.

`bastion` is CT 102 on `leviathan` — a minimal Debian 12 container on the midnight VLAN at 192.168.40.50. All SSH flows through it. If you want to SSH to Proxmox, you go Mac → bastion → leviathan. To the NAS: Mac → bastion → wreck. The bastion has no private keys on it — it's purely a network hop — but it's the only server that accepts SSH from outside, which means one place to monitor, one place to lock down, one place to cut access if needed.

---

## The architecture

```
MacBook (anywhere via Tailscale)
         ↓
   bastion (192.168.40.50)
    ↙          ↘
leviathan      wreck
192.168.40.10  192.168.50.10
```

The bastion is the only server reachable directly over SSH from the Mac. Every other server has iptables rules that drop SSH connections from anything other than `192.168.40.50`. Combined with Tailscale for the outer layer of access control, this means SSH access requires: Tailscale authentication, then bastion authentication, then target server authentication. Three separate credential checks.

---

## SSH key strategy

One key per server. Not one key for everything, not one key for "internal servers."

```
~/.ssh/id_ed25519          → pfSense
~/.ssh/id_ed25519_jump     → bastion
~/.ssh/id_ed25519_proxmox  → leviathan
~/.ssh/id_ed25519_nas      → wreck
~/.ssh/id_ed25519_keycloak → keycloak
~/.ssh/id_ed25519_ipa      → FreeIPA
~/.ssh/id_ed25519_vault    → Vault
```

If a key is ever compromised, only one server is affected. All keys are backed up in Bitwarden under `SSH Keys — Homelab`. Private keys never leave the Mac — the bastion holds no private keys, it's just a ProxyJump hop.

---

## Mac SSH config

The `~/.ssh/config` file handles all the routing transparently. Once configured, `ssh leviathan` works exactly like a direct connection — the ProxyJump through bastion is invisible.

```
Host pfsense
    HostName 192.168.1.1
    User admin
    IdentityFile ~/.ssh/id_ed25519

Host bastion
    HostName 192.168.40.50
    User jordan
    IdentityFile ~/.ssh/id_ed25519_jump
    ForwardAgent yes

Host leviathan
    HostName 192.168.40.10
    User root
    IdentityFile ~/.ssh/id_ed25519_proxmox
    ProxyJump bastion

Host wreck
    HostName 192.168.50.10
    User jordanp
    IdentityFile ~/.ssh/id_ed25519_nas
    ProxyJump bastion

Host keycloak
    HostName 192.168.40.30
    User root
    IdentityFile ~/.ssh/id_ed25519_keycloak
    ProxyJump bastion

Host vault
    HostName 192.168.40.32
    User root
    IdentityFile ~/.ssh/id_ed25519_vault
    ProxyJump bastion
```

`ProxyJump bastion` tells SSH to connect to bastion first, then forward the connection to the target. From the Mac's perspective you're running `ssh leviathan` — behind the scenes it's authenticating to bastion with `id_ed25519_jump`, then authenticating to leviathan with `id_ed25519_proxmox`. Two key challenges, completely transparent.

`ForwardAgent yes` on the bastion config passes the Mac's SSH agent through to bastion, so onward connections can use the Mac's keys without storing them on bastion.

---

## Building the bastion container

CT 102 on leviathan — Debian 12, unprivileged, VLAN tag 40:

```
IP: 192.168.40.50/24
Gateway: 192.168.40.1
DNS: 192.168.40.1
RAM: 512MB
Disk: 8GB
```

![Proxmox CT 102 bastion summary page](/images/TODO.png)
*Proxmox — CT 102 bastion container summary*

**Create the user** — root SSH login is disabled, so create a non-root user:

```bash
adduser jordan
usermod -aG sudo jordan
```

**Add the SSH public key:**

```bash
mkdir -p /home/jordan/.ssh
echo "your-id_ed25519_jump.pub-content" >> /home/jordan/.ssh/authorized_keys
chmod 700 /home/jordan/.ssh
chmod 600 /home/jordan/.ssh/authorized_keys
chown -R jordan:jordan /home/jordan/.ssh
```

**Harden sshd** — edit `/etc/ssh/sshd_config`:

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
LogLevel VERBOSE
```

Restart: `systemctl restart sshd`

**Install fail2ban:**

```bash
apt install fail2ban -y
```

Create `/etc/fail2ban/jail.local`:

```ini
[sshd]
enabled = true
port = 22
maxretry = 5
findtime = 600
bantime = 3600
logpath = /var/log/auth.log
```

If `/var/log/auth.log` doesn't exist on a fresh container:
```bash
touch /var/log/auth.log
systemctl restart fail2ban
```

![fail2ban status showing sshd jail active with ban count](/images/TODO.png)
*fail2ban-client status sshd — jail active, ready to ban brute force attempts*

---

## Distribute the public keys

Each server needs the right public key in its `authorized_keys`:

```bash
# On leviathan
echo "id_ed25519_proxmox.pub content" >> /root/.ssh/authorized_keys

# On wreck (Synology)
echo "id_ed25519_nas.pub content" >> /volume1/homes/jordanp/.ssh/authorized_keys
```

For the NAS, the same Synology SSH key auth gotchas from the pfSense hardening post apply — ACL permissions on the home directory must be clean.

---

## Lock down leviathan with iptables

This is the step that actually enforces the bastion pattern. Without it, someone could bypass the bastion entirely by SSH'ing directly to leviathan's IP.

On `leviathan`:

```bash
# Allow SSH only from bastion
iptables -A INPUT -p tcp --dport 22 -s 192.168.40.50 -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j DROP

# Save the rules
iptables-save > /etc/iptables.rules
```

Make persistent across reboots — add to `/etc/network/interfaces` under the `vmbr0` interface:

```
pre-up iptables-restore < /etc/iptables.rules
```

A few notes on why this approach rather than UFW: UFW isn't available in Proxmox's repos, and `iptables-persistent` also wasn't available on this install. The manual `iptables-save` + `pre-up` restore works reliably.

One gotcha: Tailscale NATting means traffic from Tailscale arrives on leviathan appearing to come from `192.168.40.1` (the pfSense gateway), not from the Tailscale CGNAT range. pfSense firewall rules alone won't catch a direct SSH attempt from Tailscale — the iptables rule on the host itself is what enforces it.

---

## pfSense firewall rules

Allow the bastion to reach other servers over SSH:

```
Pass TCP · 192.168.40.50 → MIDNIGHT · port 22 · Allow bastion SSH to midnight servers
Pass TCP · 192.168.40.50 → 192.168.50.10 · port 22 · Allow bastion SSH to NAS
```

---

## Verifying it works

```bash
# Direct connection to bastion — should work
ssh bastion

# Hop through bastion to leviathan — should work seamlessly
ssh leviathan

# Check last login on leviathan — should show from 192.168.40.50 (bastion)
# Last login: ... from 192.168.40.50

# Direct SSH to leviathan bypassing bastion — should fail
ssh -o ProxyJump=none leviathan
# Connection refused — iptables blocking direct access ✅
```

---

## A note on the "too many authentication failures" error

With multiple SSH keys configured, SSH's default behaviour is to try every key it knows about. If you have 7+ keys and the server has a low `MaxAuthTries` setting, you'll hit this error before the right key is offered.

Fix by specifying the key explicitly:

```bash
ssh -o IdentitiesOnly=yes -i ~/.ssh/id_ed25519_proxmox leviathan
```

Or add `IdentitiesOnly yes` to the relevant `Host` block in `~/.ssh/config` — which the config above already does via the `IdentityFile` directive on each host.

---

## What's next

Bastion is running, SSH access is funnelled through one hardened entry point, and iptables enforces it at the host level. The lab's security posture is now enterprise-grade for remote access.

Next up: the blog server — how `blog` (CT 101) is set up in the twilight DMZ, the Hugo auto-deploy pipeline via Gitea webhooks, and how pushing to Git gets a post live in under a minute.