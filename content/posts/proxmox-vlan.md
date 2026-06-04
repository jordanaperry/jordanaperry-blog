---
title: "Proxmox: VLAN Setup and Container Networking"
date: 2026-06-01
draft: false
tags: ["homelab", "vlan", "networking", "lxc", "proxmox"]
series: ["Homelab Journey"]
description: "Renaming the node, enabling VLAN-aware bridging, configuring the network interfaces, and getting containers onto the right VLANs."
cover:
  image: "/images/banners/banner-post-10.svg"
  alt: "Proxmox: VLAN Setup and Container Networking"
  relative: false
---

`leviathan` is a 2013 Mac Pro — the cylindrical trash can design — running Proxmox 8. It has 12 cores, 32GB RAM, and two NICs. Fans are nearly silent even under load, which makes it a surprisingly pleasant hypervisor to have running in a home lab. My previous setup ran on a Dell R210 II — a proper rack server that sounded like a jet engine and made one room noticeably warmer year-round. The Mac Pro is a completely different experience. Power draw is still higher than a mini PC — that's the tradeoff for the hardware you get — but the silence alone was worth the switch.

CPU load with all eight containers running is essentially nothing — the 12 cores barely register. RAM is the real constraint. 32GB sounds like plenty until Keycloak, FreeIPA, and Vault are all running simultaneously — upgrading to 64GB is on the roadmap. But for the price of a dusty closet appliance, the hardware punches well above its weight.

This post covers the network setup: getting the two NICs configured correctly, enabling VLAN-aware bridging so containers can live on any VLAN, and setting up the management interface so Proxmox is reachable from the midnight VLAN.

---

## First thing — rename the node

The node came in with the hostname `decomp-pve1` from a previous Proxmox install I had done before this rebuild. The first thing I did was rename it to `leviathan` to fit the ocean theme and match the DNS entry.

```bash
# Update hostname
echo "leviathan" > /etc/hostname
hostnamectl set-hostname leviathan

# Update /etc/hosts — replace the old name
sed -i 's/decomp-pve1/leviathan/g' /etc/hosts

# Rename the node in Proxmox cluster config
mv /etc/pve/nodes/decomp-pve1 /etc/pve/nodes/leviathan

reboot
```

After the reboot, the Proxmox UI should show `leviathan` as the node name and the management interface is reachable at `https://leviathan.home:8006`.

![Proxmox dashboard showing leviathan as the node name](/images/TODO.png)
*Proxmox node overview — leviathan with all containers listed*

---

## The two NICs

`leviathan` has two physical NICs:

- **nic0** — the main NIC, connected to switch port 2 (trunk, all VLANs tagged)
- **nic1** — the second NIC, connected to switch port 3 (access port, untagged abyss VLAN 50)

This separation matters. VM and container traffic flows through nic0 tagged with the appropriate VLAN. iSCSI storage traffic flows through nic1 on a dedicated untagged abyss link — keeping storage off the main VM network entirely and giving it a clean 2.5G path to the NAS.

---

## Network interface configuration

The full `/etc/network/interfaces` config:

```
auto lo
iface lo inet loopback

iface nic0 inet manual

iface nic1 inet manual

auto vmbr0
iface vmbr0 inet manual
        bridge-ports nic0
        bridge-stp off
        bridge-fd 0
        bridge-vlan-aware yes
        bridge-vids 2-4094

auto vmbr0.40
iface vmbr0.40 inet static
        address 192.168.40.10/24
        gateway 192.168.40.1
        vlan-raw-device vmbr0

auto vmbr1
iface vmbr1 inet manual
        bridge-ports nic1
        bridge-stp off
        bridge-fd 0
        #Storage - VLAN 50

source /etc/network/interfaces.d/*
```

A few things worth explaining here.

**`vmbr0` with `bridge-vlan-aware yes`** — this is the key setting. VLAN-aware mode lets a single bridge carry multiple VLANs, with each container or VM assigned its own VLAN tag. Without this, you'd need a separate bridge per VLAN. `bridge-vids 2-4094` tells the bridge to accept any VLAN tag in that range.

**`vmbr0.40`** — this is the management interface. It's a VLAN subinterface on top of `vmbr0`, tagged as VLAN 40 (midnight). The `vlan-raw-device vmbr0` line is critical — it tells the system this is a VLAN interface sitting on top of `vmbr0` and ensures the correct startup order. Without it, the interface comes up before the bridge is ready and the management IP is unreachable.

**`vmbr1`** — the storage bridge. It sits on nic1 and carries untagged abyss traffic. No IP assigned here — it's a passthrough for iSCSI traffic to the NAS.

![Proxmox Network interfaces page showing vmbr0, vmbr0.40, and vmbr1](/images/TODO.png)
*System → Network — showing both bridges and the management VLAN subinterface*

---

## Switch port configuration

On the Unifi switch, port 2 (connected to nic0) is set to:
- Native VLAN: **None**
- Tagged VLANs: midnight (40), plus any other VLANs containers need

Native VLAN must be None — Proxmox tags its own VM traffic via the VLAN-aware bridge. If the switch also applied a native VLAN, traffic would get double-tagged and nothing would work. The switch passes tagged frames through and lets Proxmox sort them out.

Port 3 (connected to nic1) is set to native VLAN abyss (50), untagged. Simple access port for storage.

---

## DNS

By default Proxmox uses whatever DNS was set during installation. Update it to point to the pfSense gateway on midnight:

```bash
echo "nameserver 192.168.40.1" > /etc/resolv.conf
```

Then add a firewall rule in pfSense on the midnight interface allowing TCP/UDP port 53 from the midnight subnet to 192.168.40.1 — covered in the firewall rules post, but worth double-checking if `apt update` hangs or DNS resolution fails on the host.

---

## Creating containers on the right VLAN

When creating any new LXC container in Proxmox, two settings in the **Network** tab matter:

- **Bridge**: `vmbr0`
- **VLAN Tag**: the VLAN number for that container (40 for midnight, 30 for twilight, etc.)

That's it. The container gets an IP from pfSense DHCP on that VLAN, or you assign a static one. The VLAN-aware bridge handles the tagging transparently.

![Proxmox create CT network tab — showing vmbr0 and VLAN tag field](/images/TODO.png)
*Create CT → Network — bridge set to vmbr0, VLAN tag set to 40 for a midnight container*

To verify the VLAN tag is actually applied after container creation:

```bash
bridge vlan show
```

You should see the container's `veth` interface listed with the correct VLAN tag.

---

## Current container inventory

Here's what's running on leviathan now:

| CT | Hostname | IP | VLAN | Storage | Purpose |
|---|---|---|---|---|---|
| 100 | helm | 192.168.40.20 | midnight | local-lvm | Unifi controller |
| 101 | blog | 192.168.30.10 | twilight | local-lvm | Hugo blog server |
| 102 | bastion | 192.168.40.50 | midnight | local-lvm | SSH jumpbox |
| 103 | gitea | 192.168.40.33 | midnight | local-lvm | Gitea Git server |
| 104 | keycloak | 192.168.40.30 | midnight | wreck-lvm | Keycloak IAM |
| 105 | monitor | 192.168.40.40 | midnight | wreck-lvm | Prometheus + Grafana |
| 106 | ipa | 192.168.40.31 | midnight | wreck-lvm | FreeIPA |
| 107 | vault | 192.168.40.32 | midnight | wreck-lvm | HashiCorp Vault |

The containers on `wreck-lvm` are using iSCSI storage from the NAS — covered in the next post. The earlier containers (100–103) are still on `local-lvm` (the M.2 drive) and work fine there.

![Proxmox container list showing all 8 CTs with their IPs and statuses](/images/TODO.png)
*Proxmox node view — all containers with status, VLAN, and storage*

---

## What's next

VLAN-aware bridging is running, containers are on their correct VLANs, and the management interface is reachable from midnight. Next post: iSCSI storage — connecting the NAS to Proxmox over the dedicated storage link and moving container workloads onto it.
