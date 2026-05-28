---
cover:
  image: "/images/banners/banner-post-02.svg"
  alt: "pfSense: The Hardware and VLAN Design"
  relative: false
date: 2026-05-24
draft: false
tags: ["homelab", "networking", "pfsense", "vlan"]
series: ["Homelab Journey"]
description: "Starting from a factory reset — the hardware, the VLAN design, and why everything is named after ocean zones."
---
The first thing I did when I decided to rebuild this lab was wipe pfSense back to factory defaults.

Not because the old config was broken. Because it was a mess — years of accumulated rules, interfaces I didn't remember adding, DHCP leases for devices that no longer existed. Starting clean felt right. And since I was switching to a Ubiquiti switch and access point for the first time, a clean slate made the design decisions easier to think through from scratch.

This post covers the foundation: the hardware running pfSense, and how I designed the VLAN structure. Firewall rules, DNS, DHCP, hardening, and pfBlockerNG each get their own post — there's too much to cover well in one go.

---

## The Hardware

pfSense runs on a Qotom Q330G4 — a fanless mini PC with four Intel NICs. It draws about 10 watts at idle, makes no noise, and has more network ports than a home lab needs, which is exactly what you want in a firewall appliance.

![Photo of the Qotom Q330G4](/images/Qotom-mini-q300g4.jpg)
*Photo of the Qotom Q330G4*

The hostname is `krakenfw`. The kraken watches over everything that flows in and out. It seemed right.

The factory reset gave me a clean slate. Confirmed interfaces on first boot: `igb0` as WAN, `igb1` as LAN, `igb2` and `igb3` spare. No VLANs, no rules carried over. Perfect.

---

## Why VLANs

A flat home network puts every device on the same /24 — your MacBook, your phone, your smart fridge, and your NAS all on equal footing, all able to reach each other freely. That's convenient and terrible for security.

The refrigerator in my kitchen runs firmware I've never audited from a manufacturer I don't fully trust. It needs internet access to do whatever it's doing. It does not need to be able to reach my NAS, my Proxmox server, or my work laptop. On a flat network there's nothing stopping it from trying.

VLANs (Virtual Local Area Networks) let you carve a single physical network into logically separate segments. Traffic between segments has to pass through a router — in my case, pfSense — which means you can write explicit firewall rules controlling exactly what's allowed to talk to what. The refrigerator gets internet. It does not get anything else.

This is the same concept enterprise networks use to isolate workloads. Doing it at home with a $150 mini PC and a managed switch is genuinely one of the most satisfying parts of running a home lab.

---

## The VLAN Design

I ended up with six VLANs, each named after an ocean zone:

| VLAN | Name | Subnet | Purpose |
|---|---|---|---|
| 1 | `surface` | 192.168.1.0/24 | Management — Unifi devices only |
| 10 | `reef` | 192.168.10.0/24 | Trusted — MacBook, phone, gaming PC |
| 20 | `drift` | 192.168.20.0/24 | IoT — smart appliances, fully isolated |
| 30 | `twilight` | 192.168.30.0/24 | DMZ — public-facing servers |
| 40 | `midnight` | 192.168.40.0/24 | Internal servers — private VMs |
| 50 | `abyss` | 192.168.50.0/24 | NAS — tightest access control |

![VLAN segmentation diagram](/images/vlan-segmentation-diagram.svg)

The naming is intentional. `Surface` is management — controlled, right at the top. `Reef` is trusted — shallow, warm, where things live. `Drift` is IoT — untethered, don't let it reach anything important. `Twilight` is the DMZ — light fading, public-facing. `Midnight` is internal servers — dark, nothing unauthorized goes down here. `Abyss` is the NAS — deepest, most isolated, where things go to be stored and not touched.

When I'm reading firewall logs and I see `nautilus.drift` trying to reach `wreck.abyss`, the threat model is immediately obvious from the names alone. More on the naming scheme in the first post.

**Surface** is management only. The Unifi switch and AP sit here and reach the Unifi controller on `midnight`. No internet. No access to other VLANs. Just enough to do their job.

**Reef** is my trusted network — devices I actually use. It can reach `midnight` and the NAS on `abyss`. It can reach the internet. It cannot reach `drift` or `twilight`.

**Drift** is IoT jail. Devices here get internet and nothing else. My refrigerator, microwave, and stove live here and have no idea the rest of the network exists.

**Twilight** is the DMZ — Proxmox VMs that need to be reachable from the internet. Internet yes, internal network no. Blast radius contained.

**Midnight** is internal servers — the Unifi controller, Proxmox management, a bastion host. Can reach `abyss` for storage. Cannot reach `reef` or `drift`.

**Abyss** is the NAS. Only accepts connections from `reef` and `midnight`. Everything else blocked, including outbound. A storage volume that can initiate connections to the internet is a storage volume that can exfiltrate your data.

---

## Setting It Up in pfSense

All five VLAN interfaces run off `igb1` as 802.1Q tagged subinterfaces. pfSense sees each one as a separate interface.

In the GUI: **Interfaces → VLANs → Add** for each VLAN tag (10, 20, 30, 40, 50), all with `igb1` as the parent. 

![Interfaces → VLANs — the full list showing all five tagged subinterfaces on igb1](/images/Interfaces-VLANs)
*Interfaces → VLANs — the full list showing all five tagged subinterfaces on igb1*

Then **Interfaces → Assignments** to add each VLAN as an interface (OPT1 through OPT5), enable each one, set IPv4 to static, and assign the gateway IP for that subnet:

| Interface | IP | Description |
|---|---|---|
| OPT1 | 192.168.10.1 | reef |
| OPT2 | 192.168.20.1 | drift |
| OPT3 | 192.168.30.1 | twilight |
| OPT4 | 192.168.40.1 | midnight |
| OPT5 | 192.168.50.1 | abyss |

![Interfaces → Assignments — showing OPT1–OPT5 mapped to each VLAN](/images/Interface-Assignments.png)
*Interfaces → Assignments — showing OPT1–OPT5 mapped to each VLAN*

![One VLAN interface config page — showing static IP, description, and enabled state (reef is a good example)](/images/VLAN-Interface-Config.png)
*One VLAN interface config page — showing static IP, description, and enabled state (reef is a good example)*

Quick SSH verify once everything is saved:

```bash
ifconfig | grep -E "igb1\.|inet " | grep -v inet6
```

You should see `igb1.10`, `igb1.20`, `igb1.30`, `igb1.40`, and `igb1.50` all up.

![Home lab network topology](/images/network-topology-diagram.svg)

---

## What's Next

That's the foundation — hardware confirmed, VLANs designed, interfaces up. The network is segmented but nothing is enforced yet. That's what firewall rules are for.

Next post: writing the firewall rules that actually give this segmentation its teeth, including the default deny philosophy, alias usage, and the DNS gotcha that will cost you an hour if you don't know about it.
