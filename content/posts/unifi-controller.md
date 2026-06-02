#THIS NOTE NEEDS UPDATING!!!! We increased size of the Container
---
title: "Unifi: Running the Controller on Proxmox"
cover:
  image: "/images/banners/banner-post-09.svg"
  alt: "Unifi: Running the Controller on Proxmox"
  relative: false
date: 2026-05-31
draft: false
tags: ["homelab", "networking", "unifi", "proxmox", "lxc"]
series: ["Homelab Journey"]
description: "Installing the Unifi Network Application in an LXC container on Proxmox, migrating from an old controller, and the firewall rules that kept the switch and AP online."
---
The Unifi Network Application needs to rßun somewhere. For most people that's a cloud account or a dedicated Unifi device. For a home lab, a lightweight LXC container on Proxmox is a better fit — no subscription, no cloud dependency, no extra hardware, and it's on your network where it belongs.

This post covers setting up `helm` — the Proxmox container running the Unifi controller — and migrating the switch and AP from the old controller that was running on my gaming PC.

---

## The container

`helm` is LXC container CT 100 on `leviathan`, sitting on the midnight VLAN (192.168.40.20). It's a minimal Debian 12 container — 2GB RAM, 2 CPU cores, 16GB disk. The 2GB RAM floor matters: running with 1GB causes slow startup and instability. Proxmox lets you do it, but the controller will complain and eventually fall over under load.

| Setting | Value |
|---|---|
| CT ID | 100 |
| Hostname | helm |
| IP | 192.168.40.20/24 |
| Gateway | 192.168.40.1 |
| VLAN | 40 (midnight) |
| RAM | 2GB |
| Disk | 16GB |

![Proxmox → CT 100 (helm) summary — showing RAM, CPU, disk, and IP](/images/ct100.png)
*Proxmox → CT 100 (helm) summary — showing RAM, CPU, disk, and IP*

---

## Installation

The Unifi installer script at get.ui.com now requires a Ubiquiti login to download. Skip it and use the apt repository instead — it's cleaner and makes updates easier anyway.

```bash
# MongoDB 7.0
curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | \
  gpg --dearmor -o /usr/share/keyrings/mongodb-server-7.0.gpg
echo "deb [signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg] \
  https://repo.mongodb.org/apt/debian bookworm/mongodb-org/7.0 main" | \
  tee /etc/apt/sources.list.d/mongodb-org-7.0.list

# Java (Temurin 17)
curl -fsSL https://packages.adoptium.net/artifactory/api/gpg/key/public | \
  gpg --dearmor -o /usr/share/keyrings/adoptium.gpg
echo "deb [signed-by=/usr/share/keyrings/adoptium.gpg] \
  https://packages.adoptium.net/artifactory/deb bookworm main" | \
  tee /etc/apt/sources.list.d/adoptium.list

# Unifi
curl -fsSL https://dl.ui.com/unifi/unifi-repo.gpg | \
  gpg --dearmor -o /usr/share/keyrings/unifi-repo.gpg
echo "deb [signed-by=/usr/share/keyrings/unifi-repo.gpg] \
  https://www.ui.com/downloads/unifi/debian stable ubiquiti" | \
  tee /etc/apt/sources.list.d/unifi.list

# Install
apt update && apt install -y temurin-17-jre mongodb-org unifi

# Enable on boot
systemctl enable unifi mongodb
systemctl start unifi
```

Once the service starts, the controller is accessible at `https://helm.home:8443`. The first time you load it you'll go through a setup wizard — create a local account (not a Ubiquiti cloud account) and complete the initial configuration.

---

## Migrating from the old controller

The previous controller was running on the gaming PC. Moving to `helm` without losing the switch and AP configuration requires a backup and restore — don't try to re-adopt devices from scratch, you'll lose all your port profiles and network config.

**On the old controller:**
1. Settings → System → Backup → Download backup
2. Under Settings → System → Advanced, set **Inform Host Override** to `192.168.40.20` — this tells devices to check in with the new controller IP instead of the old one

**On the new controller:**
1. Settings → System → Restore → upload the backup file
2. Set Inform Host Override to `192.168.40.20` here as well

![Settings → System → Advanced — showing Inform Host Override set correctly](/images/TODO.png)
*Settings → System → Advanced — showing Inform Host Override set correctly*

One thing Ubiquiti changed and doesn't make obvious: the Inform Host Override field accepts an IP address only — not a full URL. The controller constructs the full inform URL itself. If you paste `http://192.168.40.20:8080/inform` it won't work.

After the restore, the switch and AP should re-adopt automatically within a few minutes. Check **Devices** in the controller — both should show as Connected.

![Devices page — showing both manifold and buoy as Connected with uptime](/images/TODO.png)
*Devices page — showing both manifold and buoy as Connected with uptime*

If they show as Disconnected, check the firewall rules below.

---

## The firewall rules

This is the part that catches people. After locking down the surface VLAN in pfSense, the switch and AP lost connectivity to the controller on midnight. They were online on the network but the controller showed them as disconnected.

The Unifi controller uses two ports: **8080** for the device inform protocol (how devices check in with the controller) and **8443** for the web UI. Both need to be reachable from surface.

Add these rules in pfSense on the surface interface:

```
Pass TCP · surface net → 192.168.40.20 · port 8080 · Allow Unifi inform
Pass TCP · surface net → 192.168.40.20 · port 8443 · Allow Unifi controller UI
```

And on the midnight interface, allow inbound from surface:

```
Pass TCP · surface net → 192.168.40.20 · ports 8080, 8443 · Allow controller access
```

![pfSense → Firewall → Rules → surface — showing the port 8080 and 8443 pass rules to helm](/images/TODO.png)
*pfSense → Firewall → Rules → surface — showing the port 8080 and 8443 pass rules to helm*

Port 8080 is the critical one — without it devices show as offline even though they're physically connected and passing traffic correctly. The controller relies on devices actively checking in over 8080 to mark them online.

---

## Controller settings worth configuring

A few things worth setting up once the controller is running and devices are adopted:

**Device SSH Authentication** — under Settings → System → Advanced → Device SSH Settings. Replace password authentication with an SSH key. Same reasoning as everywhere else — passwords are a liability.

![Settings → System → Advanced → Device SSH Settings — showing SSH key auth enabled](/images/TODO.png)
*Settings → System → Advanced → Device SSH Settings — showing SSH key auth enabled*

**Auto-update schedule** — set devices to update at 3am, same as everything else in the lab. Unifi firmware updates occasionally break things, so having a consistent update window makes it easier to correlate issues.

**Inform Host Override** — double check this is set to `192.168.40.20`. If it ever gets cleared, devices will lose contact with the controller after a restart.

---

## What's next

Unifi is done — switch configured, controller running, devices adopted and online. The entire network stack is now in place: pfSense handling routing and security, the Unifi switch enforcing VLANs at the physical layer, and the controller managing it all from a container on Proxmox.

Next up: Proxmox itself — how leviathan is set up, VLAN tagging for VM and container networking, and iSCSI storage from the NAS.
