---
title: "pfSense: Security Hardening"
date: 2026-05-28
draft: false
tags: ["homelab", "networking", "pfsense", "security", "hardening"]
series: ["Homelab Journey"]
description: "SSH keys, GUI hardening, killing unnecessary services, and automated config backups to the NAS."
---
A freshly installed pfSense is reasonably secure out of the box. But reasonable isn't the same as hardened. This post covers everything I did to lock down `krakenfw` — from SSH key authentication to automated config backups — and a few specific gotchas with the Synology NAS that took longer than they should have.

---

## SSH hardening

The first thing to lock down is SSH. By default pfSense allows password-based SSH authentication, which means a brute-force attack against port 22 is theoretically viable if it's ever exposed.

Generate an ed25519 key pair on your Mac:

```bash
ssh-keygen -t ed25519 -C "homelab-pfsense"
```

Accept the default path (`~/.ssh/id_ed25519`) and set a strong passphrase. Then copy the public key:

```bash
cat ~/.ssh/id_ed25519.pub
```

In pfSense: **System → User Manager → admin → Authorized SSH Keys** — paste the output. Then test it before disabling password auth:

```bash
ssh -i ~/.ssh/id_ed25519 admin@192.168.1.1
```

Once confirmed working: **System → Advanced → Admin Access → SSHd Key Only → Public Key Only**. Password authentication is now disabled. If you lose the key, you need physical console access to get back in — make sure the key is backed up somewhere safe. Mine lives in a Bitwarden secure note.

Add a `~/.ssh/config` entry on your Mac to make connecting cleaner:

```
Host pfsense
    HostName 192.168.1.1
    User admin
    IdentityFile ~/.ssh/id_ed25519
    Port 22
```

Now `ssh pfsense` connects directly. Add the key to the Mac keychain so you're not prompted for the passphrase every time:

```bash
ssh-add --apple-use-keychain ~/.ssh/id_ed25519
```

Finally, block SSH from VLANs that have no business reaching pfSense:

```
Block TCP · drift net → 192.168.1.1:22
Block TCP · twilight net → 192.168.1.1:22
```

SSH is not exposed on WAN. There is no port forward, no NAT rule, nothing. Remote access is handled by Tailscale instead — more on that in a future post.

---

## GUI hardening

The web interface runs on port 443 by default. Moving it to a non-standard port doesn't stop a determined attacker but it eliminates a lot of automated scanning noise.

**System → Advanced → Admin Access → TCP port → 8443**

While you're in that screen:
- Enable "Disable webConfigurator redirect rule" — stops HTTP automatically redirecting to HTTPS, which removes one exposure vector
- Enable login protection — the defaults (30 failures, 120 second block) are fine

Block GUI access from VLANs that shouldn't have it:

```
Block TCP · drift net → 192.168.1.1:8443
Block TCP · twilight net → 192.168.1.1:8443
```

Verify the GUI is not reachable from WAN — there should be no port 8443 forward on the WAN interface, and there isn't. The only way to reach the pfSense GUI remotely is through Tailscale, which requires an authenticated device on the tailnet.

---

## General hardening checklist

A few smaller items worth doing:

**Disable IPv6** — I don't run IPv6 in this lab and every enabled protocol is a potential attack surface. **System → Advanced → Networking → uncheck Allow IPv6**.

**Verify UPnP is off** — UPnP lets devices punch holes in your firewall automatically. It's off by default in pfSense and should stay that way. **Services → UPnP & NAT-PMP** — confirm it's not enabled.

**Verify SNMP is off** — same reasoning. Off by default, should stay off unless you specifically need it.

**Block bogon and private networks on WAN** — pfSense enables these by default but worth confirming. **Interfaces → WAN → Block private networks** and **Block bogon networks** should both be checked. Bogon networks are IP ranges that should never appear on the public internet; blocking them at the WAN interface is free mitigation.

**Delete the default allow-all LAN rules** — covered in the firewall rules post but worth repeating. These rules exist for convenience on a fresh install. **Firewall → Rules → LAN → delete "Default allow LAN to any rule"** and the IPv6 equivalent. If you've already set up your VLAN rules they've been replaced, but clean them up anyway.

---

## Automated config backups

Manual backups are the kind of thing you do once, feel good about, and never do again until the day you actually need one. I automated it.

The Synology NAS runs a daily Task Scheduler job at 3am that SSH's into pfSense and pulls a copy of `config.xml`:

```bash
#!/bin/sh
DATE=`/bin/date +%F`
scp -i /volume1/homes/jordanp/.ssh/id_ed25519_pfsense \
  admin@192.168.1.1:/cf/conf/config.xml \
  /volume1/homes/jordanp/backups/pfsense/config-$DATE.xml
```

Getting SSH key auth working between Synology DSM and pfSense took more fiddling than expected. A few things the Synology docs don't make obvious.

> **Note:** I'm running DSM 6.2.4-25556 Update 8. Synology made significant changes to SSH handling in DSM 7 — if you're on a newer version some of these gotchas may not apply, or may present differently. Worth checking the DSM 7 release notes if things aren't behaving as described here.

**Home directory ACL permissions** — Synology's OpenSSH is strict. The home directory at `/volume1/homes/jordanp` cannot have any ACL entries (the `+` flag in `ls -la`). If it does, SSH key auth silently fails. Remove them via File Station → Properties → Permission → Delete ACL.

**Absolute path in sshd_config** — `AuthorizedKeysFile` needs an absolute path: `/volume1/homes/%u/.ssh/authorized_keys`. A relative path won't work.

**PubkeyAuthentication** — must be explicitly uncommented as `yes` in `/etc/ssh/sshd_config`. It's present but commented out by default.

**Use `/bin/date` not `date` in scripts** — Task Scheduler runs with a limited PATH. `date` isn't found; `/bin/date` is. This one burned 20 minutes.

The backup key for the NAS is separate from the daily-use Mac key and lives in its own Bitwarden secure note. The pfSense firewall rule on abyss allows the NAS to SSH outbound to pfSense on port 22 — without that rule the backup script fails silently.

## What's next

pfSense is now properly hardened. Last post in the pfSense series covers pfBlockerNG — adding DNS-level ad, tracking, and malware blocking that works for every device on the network, including IoT appliances that ignore system-level proxy settings.