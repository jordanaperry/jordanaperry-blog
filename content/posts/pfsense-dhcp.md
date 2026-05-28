---
title: "pfSense: DHCP and Static Leases"
date: 2026-05-27
draft: false
tags: ["homelab", "networking", "pfsense", "dhcp"]
series: ["Homelab Journey"]
description: "Switching to Kea DHCP, setting up per-VLAN pools, and why every important device gets a static lease."
cover:
  image: "/images/banners/banner-post-05.svg"
  alt: "pfSense: DHCP and Static Leases"
  relative: false
---
DHCP is the part of networking most people never think about — it just works. But in a segmented lab with six VLANs, a few deliberate choices here make everything else easier to manage.

---

## Switching to Kea

pfSense 2.7+ ships with two DHCP backends: the older ISC DHCP server and the newer Kea DHCP. I switched to Kea — it's actively maintained, handles per-VLAN pools cleanly, and is the direction pfSense is heading.

**System → Advanced → Networking → DHCP Backend → Kea**

![System → Advanced → Networking — showing Kea selected as the DHCP backend](/images/Kea-DHCP.png)
*System → Advanced → Networking — showing Kea selected as the DHCP backend*

Reboot after switching. Then configure pools per VLAN under **Services → DHCP Server**, where each VLAN interface now has its own tab.

![Services → DHCP Server — showing the tab bar with all VLANs listed](/images/DHCP-Bar.png)
*Services → DHCP Server — showing the tab bar with all VLANs listed*

---

## DHCP pools

Each VLAN gets a pool sized for its expected device count. I left the lower end of each range free for static leases so they sit in predictable, memorable addresses.

| VLAN | Pool range |
|---|---|
| reef (10) | 192.168.10.100 – 200 |
| drift (20) | 192.168.20.100 – 150 |
| twilight (30) | 192.168.30.100 – 120 |
| midnight (40) | 192.168.40.100 – 150 |
| abyss (50) | No pool — static only |

![One VLAN DHCP pool config — reef showing the range and settings](/images/DHCP-Config.png)
*One VLAN DHCP pool config — reef showing the range and settings*

`abyss` has no DHCP pool at all. The NAS has a static IP and nothing else should ever appear on that VLAN. No pool means no accidental device picking up an address there.

---

## Static leases

Static leases assign a fixed IP to a device by MAC address. Every time that device connects, it gets the same IP — regardless of whether you've hardcoded anything on the device itself. pfSense hands out the right address based on the MAC.

This matters for a few reasons. DNS host overrides point to specific IPs. Firewall rules reference specific IPs. If a device's address changes, both break silently. Static leases prevent that.

Every important device in the lab has one:

**surface (VLAN 1)**
| Device | Hostname | IP |
|---|---|---|
| Unifi switch | manifold | 192.168.1.2 |
| Unifi AP | buoy | 192.168.1.3 |

**reef (VLAN 10)**
| Device | Hostname | IP |
|---|---|---|
| MacBook Pro | manta | 192.168.10.10 |
| Gaming PC | orca | 192.168.10.20 |

**drift (VLAN 20)**
| Device | Hostname | IP |
|---|---|---|
| Fridge | nautilus | 192.168.20.11 |
| Microwave | zapper | 192.168.20.12 |
| Stove | hydra | 192.168.20.13 |

**midnight (VLAN 40)**
| Device | Hostname | IP |
|---|---|---|
| Proxmox | leviathan | 192.168.40.10 |
| Unifi controller | helm | 192.168.40.20 |
| Bastion host | bastion | 192.168.40.50 |

**abyss (VLAN 50)**
| Device | Hostname | IP |
|---|---|---|
| Synology NAS | wreck | 192.168.50.10 |

![Static Mappings list for reef — showing manta and orca with their IPs](/images/DHCP-Static.png)
*Static Mappings list for reef — showing manta and orca with their IPs*

To add a static lease: **Services → DHCP Server → [VLAN tab] → Static Mappings → Add**. You need the MAC address of the device, the IP you want to assign, and optionally a hostname. The hostname field in Kea will also register the device in Unbound if you have that integration enabled.

---

## Finding MAC addresses

For devices already on the network, the easiest way is **Status → DHCP Leases** in pfSense — it shows every active lease with its MAC.

![Status → DHCP Leases — showing active leases across VLANs with hostnames](/images/DHCP-Leases.png)
*Status → DHCP Leases — showing active leases across VLANs with hostnames*

For IoT devices that haven't connected to the right VLAN yet, check the device's settings menu or the label on the back of the hardware.

---

## Verifying Kea is running

```bash
ps aux | grep kea
```

You should see `kea-dhcp4` running. If not, check **Status → Services** in the GUI and start it manually.

---

## What's next

DHCP locked down. Next post: hardening pfSense itself — SSH keys, GUI hardening, disabling what doesn't need to be on, and setting up automated config backups so a failed hardware replacement isn't a crisis.