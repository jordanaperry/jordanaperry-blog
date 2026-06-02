---
title: "Unifi: Switch Configuration and VLAN Tagging"
cover:
  image: "/images/banners/banner-post-08.svg"
  alt: "Unifi: Switch Configuration and VLAN Tagging"
  relative: false
date: 2026-05-30
draft: false
tags: ["homelab", "networking", "unifi", "vlan", "switch"]
series: ["Homelab Journey"]
description: "Configuring the USW Flex 2.5G, port assignments, DHCP guarding, and two SSIDs with very different security postures."
---
The pfSense side of the network is fully configured — VLANs defined, firewall rules written, DNS resolving, DHCP handing out static leases. But none of that segmentation actually works until the switch enforces it at the physical layer. This post covers the Unifi switch setup: creating the networks, assigning ports, and getting the two WiFi SSIDs running with the right security settings for their respective VLANs.

---

## The hardware

The switch is a Ubiquiti USW Flex 2.5G 8 PoE running firmware 2.1.8, managed through the Unifi Network Application running on `helm` (192.168.40.20) — a Proxmox container on `leviathan`. The next post covers getting that controller running. For now, everything here is configured through it.

![Photo of the USW Flex 2.5G](/images/TODO.png)
*Photo of the USW Flex 2.5G*

---

## Creating the networks

Before touching any ports, the VLANs need to exist as networks in the Unifi controller. **Settings → Networks → Create New Network** for each one.

The key setting for every network here: **VLAN Only**. pfSense handles all routing and DHCP — the Unifi controller just needs to know the VLAN IDs so it can tag traffic correctly. Don't let Unifi try to manage DHCP on any of these.

| Network | VLAN ID | DHCP Guard IP |
|---|---|---|
| reef | 10 | 192.168.10.1 |
| drift | 20 | 192.168.20.1 |
| twilight | 30 | 192.168.30.1 |
| midnight | 40 | 192.168.40.1 |
| abyss | 50 | 192.168.50.1 |

![Settings → Networks — the full network list showing all five VLANs](/images/TODO.png)
*Settings → Networks — the full network list showing all five VLANs*

**DHCP Guarding** deserves a mention. Enable it on every VLAN and set the trusted DHCP server IP to the pfSense gateway for that subnet. This blocks any rogue device from spinning up its own DHCP server and handing out addresses on your network — a surprisingly common IoT device behavior. On `drift` especially, this is worth having.

![One network config page — showing VLAN only mode and DHCP guarding enabled (drift is a good example)](/images/TODO.png)
*One network config page — showing VLAN only mode and DHCP guarding enabled (drift is a good example)*

---

## Port assignments

This is where the physical segmentation gets enforced. Every port gets a profile that controls what traffic it carries.

| Port | Device | Native VLAN | Tagged VLANs |
|---|---|---|---|
| 1 | Gaming PC (orca) | reef (10) | Block All |
| 2 | Leviathan NIC 1 | None | midnight (40) |
| 3 | Leviathan NIC 2 — storage | abyss (50) | Block All |
| 4 | NAS (wreck) | abyss (50) | Block All |
| 5 | MacBook (manta) | reef (10) | Block All |
| 8 | AP uplink (buoy) | Default (1) | reef (10), drift (20) |
| 9 | pfSense uplink | Default (1) | reef, drift, twilight, midnight, abyss |

![Devices → Switch → Ports — the port assignment overview showing all ports with their profiles](/images/TODO.png)
*Devices → Switch → Ports — the port assignment overview showing all ports with their profiles*

A few things worth explaining here.

**Port 2 — leviathan NIC 1** has Native VLAN set to None. This is intentional and important. Proxmox manages its own VLAN tagging via subinterfaces (`vmbr0.40`, `vmbr0.30`, etc.) — if the switch also applied a native VLAN tag, traffic would get double-tagged and nothing would work. Native VLAN: None tells the switch to pass all tagged traffic through untouched and let Proxmox sort it out.

![Port 2 config — showing Native VLAN: None for the Proxmox trunk](/images/TODO.png)
*Port 2 config — showing Native VLAN: None for the Proxmox trunk*

**Port 3 — leviathan NIC 2** carries only untagged `abyss` traffic. This is the dedicated iSCSI storage link — a clean, single-purpose connection between Proxmox and the NAS that doesn't share bandwidth with VM traffic or get mixed up with other VLANs.

![Port 3 config — showing abyss native VLAN for the storage link](/images/TODO.png)
*Port 3 config — showing abyss native VLAN for the storage link*

**Port 9 — pfSense uplink** carries all VLANs tagged. This is the trunk port — everything flows through here to pfSense, where the firewall rules decide what's actually allowed.

**Port 8 — AP uplink** only carries `reef` and `drift` tagged, plus the management VLAN. The AP doesn't need twilight, midnight, or abyss — those are wired-only VLANs. Only the two SSIDs' VLANs need to be tagged on the AP uplink.

---

## WiFi SSIDs

Two SSIDs, two very different security postures.

**TheReef** — for trusted devices.

- VLAN: reef (10)
- Security: WPA3
- Band Steering: enabled — pushes capable devices to 5GHz
- BSS Transition (802.11v): enabled
- Fast Roaming (802.11r): enabled
- Client Isolation: disabled — reef devices can talk to each other

![WiFi → TheReef settings — showing WPA3, band steering, fast roaming](/images/TODO.png)
*WiFi → TheReef settings — showing WPA3, band steering, fast roaming*

WPA3 is worth enabling here even though some older devices don't support it. Any device that can't do WPA3 probably shouldn't be on the trusted VLAN anyway — put it on TheDrift.

**TheDrift** — for IoT devices.

- VLAN: drift (20)
- Security: WPA2 — IoT devices frequently don't support WPA3
- Application: IoT mode
- PMF (Protected Management Frames): disabled — breaks many IoT devices
- Client Isolation: enabled — IoT devices can't talk to each other
- Multicast and Broadcast Blocker: enabled — reduces IoT chatter
- Band Steering: disabled — IoT devices are often 2.4GHz only

![WiFi → TheDrift settings — showing IoT mode, client isolation, multicast blocker](/images/TODO.png)
*WiFi → TheDrift settings — showing IoT mode, client isolation, multicast blocker*

Client Isolation on TheDrift means even devices on the same VLAN can't reach each other. Combined with the firewall rules blocking all RFC1918 traffic, an IoT device is thoroughly isolated — it can reach the internet and nothing else, and it can't even see the other appliances on the same SSID.

---

## Migrating devices

The previous setup had everything on a single SSID called `Manta Ray-Fi`. Switching to the new SSIDs was straightforward — connect each device to the right network, verify it got the correct IP range, then remove the old SSID once everything was off it.

The IoT devices (fridge, microwave, stove) went on TheDrift and picked up addresses in 192.168.20.x. Quick verify from pfSense: **Status → DHCP Leases** — confirm the MACs match and the IPs are in the right subnet. Then check **Firewall → Logs** to confirm drift traffic is hitting the RFC1918 block rule as expected and not reaching anything internal.

---

## What's next

Switch configured, SSIDs live, devices on their correct VLANs. Next post covers the other half of the Unifi setup — getting the controller running as an LXC container on Proxmox, migrating from the old controller, and the firewall rules needed to keep the switch and AP online after locking down the surface VLAN.