---
title: "pfSense: Firewall Rules and Network Segmentation"
date: 2026-05-25
draft: false
tags: ["homelab", "networking", "pfsense", "firewall", "security"]
series: ["Homelab Journey"]
description: "Default deny, aliases, and the DNS gotcha that will cost you an hour if you don't know about it"
cover:
  image: "/images/banners/banner-post-03.svg"
  alt: "pfSense: Firewall Rules and Network Segmentation"
  relative: false
---
The VLAN design from the last post is just labels without firewall rules. Traffic doesn't respect VLAN boundaries automatically — pfSense has to be explicitly told what's allowed to cross them. This post covers how I wrote those rules and the philosophy behind them.

---

## Default deny

Most home routers work on a default allow model — everything outbound is permitted, unsolicited inbound is blocked. It's simple and it works fine if all your devices are equally trusted.

pfSense works the opposite way by default on VLANs: if there's no rule permitting traffic, it's dropped. This is called default deny, and it's the only approach that actually enforces segmentation. If you forget to write a rule, traffic fails. That's the right failure mode — silent permission is far more dangerous than silent denial.

The first thing I did after setting up the VLANs was delete pfSense's default "allow LAN to any" rules. Those exist for convenience on a fresh install and they defeat everything else you're trying to do. Gone immediately.

---

## Aliases

Before writing a single rule I created aliases — named references for IPs and networks that rules can point to instead of hardcoded values.

| Alias | Type | Value |
|---|---|---|
| RFC1918 | Network | 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 |
| NAS | Host | 192.168.50.10 |
| MIDNIGHT | Network | 192.168.40.0/24 |
| TWILIGHT | Network | 192.168.30.0/24 |
| REEF | Network | 192.168.10.0/24 |

![Firewall → Aliases — the full alias list showing all five entries](/images/Firewall-Aliases.png)
*Firewall → Aliases — the full alias list showing all five entries*

`RFC1918` covers all private address space — the entire range of IPs that can never appear on the public internet. Any rule that says "block RFC1918" is blocking access to every private network in one shot. That's the workhorse alias.

Using aliases means if an IP changes, you update the alias once and every rule that references it updates automatically. It also makes rules readable at a glance — "block RFC1918" tells you exactly what's happening without having to decode a CIDR range.

---

## The DNS gotcha

Before covering the rules themselves, there's one thing that will cost you an hour if you don't know about it: **pfSense blocks DNS queries from VLANs to the pfSense gateway by default.**

Every VLAN's gateway is also its DNS server. When a device on `reef` (192.168.10.0/24) makes a DNS query, it goes to 192.168.10.1 — which is pfSense. But once you add an RFC1918 block rule, that DNS query matches the block rule and gets dropped. No DNS, no internet, very confusing.

The fix is to add an explicit pass rule for TCP/UDP port 53 to the gateway IP on **every VLAN**, and it must sit **above** the RFC1918 block rule in the list. pfSense evaluates rules top to bottom and stops at the first match.

| VLAN | Rule |
|---|---|
| reef | Pass TCP/UDP → 192.168.10.1:53 |
| drift | Pass TCP/UDP → 192.168.20.1:53 |
| twilight | Pass TCP/UDP → 192.168.30.1:53 |
| midnight | Pass TCP/UDP → 192.168.40.1:53 |
| abyss | Pass TCP/UDP → 192.168.50.1:53 |

Add these first. Then write everything else.

---

## Rules by VLAN

![pfSense firewall rules flow](/images/firewall-rules-diagram.svg)

Rules are evaluated top to bottom. First match wins. Order matters.

**reef (trusted devices)**

| Order | Action | Source | Destination |
|---|---|---|---|
| 1 | Pass | reef net | gateway:53 (DNS) |
| 2 | Pass | reef net | NAS |
| 3 | Pass | reef net | MIDNIGHT |
| 4 | Block | reef net | RFC1918 |
| 5 | Pass | reef net | any |

![Firewall → Rules → reef — showing all rules in order with the DNS pass rule at the top](/images/Firewall-Rules-Reef.png)
*Firewall → Rules → reef — showing all rules in order with the DNS pass rule at the top*

Reef can reach the NAS and internal servers. It cannot reach IoT devices on drift or public-facing VMs on twilight — those are both covered by the RFC1918 block. It can reach the internet. A compromised device on reef cannot pivot to anything it shouldn't touch.

**drift (IoT)**

| Order | Action | Source | Destination |
|---|---|---|---|
| 1 | Pass | drift net | gateway:53 (DNS) |
| 2 | Block | drift net | RFC1918 |
| 3 | Pass | drift net | any |

![Firewall → Rules → drift — showing the clean two-rule IoT setup](/images/Firewall-Rules-Drift.png)
*Firewall → Rules → drift — showing the clean two-rule IoT setup*
Two substantive rules. IoT devices get DNS and internet. They get nothing else. The fridge cannot reach the NAS, the MacBook, or anything on any other VLAN.

**twilight (DMZ)**

| Order | Action | Source | Destination |
|---|---|---|---|
| 1 | Block | twilight net | RFC1918 |
| 2 | Pass | twilight net | any |

Public-facing VMs can reach the internet. They cannot reach anything internal. If a server in the DMZ gets compromised, the attacker is stuck in twilight with no path to anything that matters.

**midnight (internal servers)**

| Order | Action | Source | Destination |
|---|---|---|---|
| 1 | Pass | midnight net | gateway:53 (DNS) |
| 2 | Pass | midnight net | NAS |
| 3 | Block | midnight net | RFC1918 |
| 4 | Pass | midnight net | any |

Internal servers can reach the NAS for storage and the internet for updates. They cannot reach reef or drift.

**abyss (NAS)**

| Order | Action | Source | Destination |
|---|---|---|---|
| 1 | Pass | REEF | abyss net |
| 2 | Pass | MIDNIGHT | abyss net |
| 3 | Block | any | any |

![Firewall → Rules → abyss — showing the inbound-only config with outbound block](/images/Firewall-Rules-Abyss.png)
*Firewall → Rules → abyss — showing the inbound-only config with outbound block*

The NAS only accepts connections from trusted devices and internal servers. Everything else is blocked — including outbound traffic initiated by the NAS itself. A storage volume that can make outbound connections is a liability. This one blocks it entirely.

**surface (management)**

| Order | Action | Source | Destination |
|---|---|---|---|
| 1 | Pass | surface net | 192.168.1.1:8443 (pfSense GUI) |
| 2 | Pass | surface net | 192.168.40.20:8080,8443 (Unifi controller) |
| 3 | Block | surface net | any |

![Firewall → Rules → surface — showing the locked-down management rules](/images/Firewall-Rules-Surface.png)
*Firewall → Rules → surface — showing the locked-down management rules*

Management devices can reach pfSense and the Unifi controller. Nothing else. Note that port 8080 is the Unifi device inform port — without it, the switch and AP show as offline in the controller even though they're physically connected. Learned that one the hard way.

One important sequencing note: I couldn't enable the surface block rule until the Unifi switch was configured and all devices were on their correct VLANs. Enabling it too early would have locked everything out. Configure pfSense → configure the switch → move devices → then tighten surface.

---

## Verifying the rules

Once everything was in place I verified from SSH:

```bash
pfctl -sr | tail -60
```

This prints the active ruleset. You can see every pass and block rule and confirm they're in the right order. For ongoing monitoring, enable logging on the critical deny rules — especially drift's RFC1918 block — so you can see IoT devices attempting to reach internal networks:

```bash
clog /var/log/filter.log | tail -f
```

---

## What's next

Firewall rules done. Next up: DNS — setting up Unbound to resolve internal hostnames so you can reach every device by name instead of IP address.