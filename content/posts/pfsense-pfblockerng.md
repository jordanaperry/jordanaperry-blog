---
title: "pfSense: DNS-Level Blocking with pfBlockerNG"
date: 2026-05-29
draft: false
tags: ["homelab", "networking", "pfsense", "security", "pfblockerng"]
series: ["Homelab Journey"]
description: "Installing pfBlockerNG, getting DNSBL running, and the PHP memory fix that isn't documented anywhere obvious."
cover:
  image: "/images/banners/banner-post-07.svg"
  alt: "pfSense: DNS-Level Blocking with pfBlockerNG"
  relative: false
---
pfBlockerNG is a pfSense package that plugs into Unbound and blocks ads, tracking domains, malware, and known-bad IP addresses at the DNS level — before any request ever leaves the network. No browser extension required. No client configuration. Every device on every VLAN gets the benefit automatically, including IoT devices that you can't install anything on.

This is the last post in the pfSense series. By the end of it, `krakenfw` is blocking nearly 845,000 domains and 17,000 IPs from threat intelligence feeds that update daily.

---

## Why DNS-level blocking

The standard approach to ad blocking is a browser extension. It works, but it only works in that browser, on that device, for traffic you can see. IoT devices, smart TVs, game consoles, and anything else on the network get no protection at all.

DNS-level blocking works differently. When any device makes a DNS query for a blocked domain — say, a Samsung fridge phoning home to a telemetry server — Unbound intercepts the query and returns nothing instead of the real IP. The connection never happens. The device has no idea, and you didn't have to install anything on it.

For the IoT VLAN this is especially useful. `drift` is already isolated by firewall rules, but DNS blocking adds a second layer — even if a device somehow had a path to reach a malicious domain, the name wouldn't resolve.

---

## Installation

**System → Package Manager → Available Packages**, search for `pfBlockerNG-devel` and install. The `-devel` version is the one with active development; the plain `pfBlockerNG` package is older. Reboot after install.

---

## Prerequisites before the wizard

pfBlockerNG's DNSBL feature requires a Virtual IP on the loopback interface. The wizard won't explain this clearly — just do it first.

**Firewall → Virtual IPs → Add**:
- Type: IP Alias
- Interface: Localhost (lo0)
- Address: `10.10.10.1/32`

Save it. Now run the wizard.

---

## Setup wizard

**Firewall → pfBlockerNG → General → Run Wizard**

Most defaults are fine. Two things to change:

**IPv4 DNSBL Virtual IP**: set to `10.10.10.1` — the Virtual IP you just created.

**SSL Port**: change from 8443 to 8444. If you've already moved the pfSense GUI to port 8443 (covered in the hardening post), there will be a conflict. 8444 sidesteps it cleanly.

---

## DNSBL configuration

**Firewall → pfBlockerNG → DNSBL → Enable**

Settings worth confirming:
- Resolver Live Sync: enabled — keeps Unbound in sync as blocklists update
- Permit Firewall Rules: enabled, all VLANs selected — creates the rules needed for the blocking VIP to work
- Logging/Blocking Mode: DNSBL WebServer/VIP

For feeds, I'm running:

| Feed | Blocked | Purpose |
|---|---|---|
| Prigent_Malware | 655,042 | Malware domains |
| StevenBlack_ADs | 82,200 | Ads and tracking |
| EasyList | 54,396 | Ad domains |
| EasyPrivacy | 41,073 | Tracking domains |
| DandelionSprouts | 11,851 | Various threats |
| NoCoin | 238 | Cryptomining |
| Abuse_urlhaus | 85 | Malware URLs |
| MoneroMiner | 2 | Cryptomining |

Total: **844,887 domains**.

For IP blocking I'm running CINS Army (15,000 known attacker IPs), Emerging Threats, Spamhaus Drop, and a handful of smaller feeds. Total: **17,086 IPs**.

All feeds update daily via a cron job pfBlockerNG sets up automatically.

---

## The PHP memory fix

This one isn't documented anywhere obvious. After enabling pfBlockerNG alongside Kea DHCP and Unbound, pfSense started throwing this error in the logs:

```
PHP Fatal error: Allowed memory size of 536870912 bytes exhausted
```

The `kea2unbound` process that syncs Kea DHCP leases into Unbound's host database runs out of memory when pfBlockerNG is also loading large blocklists. The fix is to increase PHP's memory limit:

```bash
echo "memory_limit = 1024M" >> /usr/local/etc/php.ini
pfSsh.php playback svc restart php-fpm
```

512MB wasn't enough. 1024MB is. If you're running Kea + Unbound + pfBlockerNG and see memory errors, this is the fix.

---

## Monitoring

**Firewall → pfBlockerNG → Reports → Statistics** — total block counts by feed, updated in real time.

**Firewall → pfBlockerNG → Reports → Alerts** — every blocked DNS query, with the device IP, the domain it tried to resolve, and which feed caught it. The IoT VLAN is the most interesting one to watch in theory — in practice, if your firewall rules are doing their job, `nautilus.drift` is already so locked down it barely gets a chance to try anything. Most of the action ends up coming from reef devices hitting ad and tracking domains on normal browsing traffic.

If something legitimate gets blocked — it happens — find it in Alerts and click the whitelist icon next to it. The domain is added to the allowlist and pfBlockerNG rebuilds the blocklist on the next update cycle.

## Wrapping up pfSense

That's the complete pfSense build across six posts:

1. Hardware and VLAN design
2. Firewall rules and segmentation
3. DNS with Unbound
4. DHCP and static leases
5. Security hardening
6. pfBlockerNG ← you are here

`krakenfw` is now a hardened, segmented firewall with internal DNS resolution, locked-down DHCP, SSH key authentication, automated config backups, and nearly a million blocked domains. Everything runs on a fanless mini PC drawing 10 watts.

Next up: the Unifi side — configuring the switch, setting up the SSIDs, and getting the controller running as a container on Proxmox.