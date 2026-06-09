---
title: "Proxmox: iSCSI Storage with Synology"
date: 2026-06-02
draft: false
tags: ["homelab", "storage", "iscsi", "synology", "proxmox", "nas"]
series: ["Homelab Journey"]
description: "Setting up an iSCSI target on the Synology NAS, connecting it to Proxmox over a dedicated storage link, and adding LVM storage for containers."
cover:
  image: "/images/banners/banner-post-11.svg"
  alt: "Proxmox: iSCSI Storage with Synology"
  relative: false
---

The Mac Pro has an M.2 drive internally, which works fine for a few containers. But putting everything on local storage defeats the point of a NAS — and if `leviathan` ever needs to be rebuilt, every container goes with it.

The solution is iSCSI — a protocol that lets Proxmox mount a block device from the NAS over the network and treat it exactly like a local disk. Containers and VMs running on iSCSI storage are backed by the NAS, survive a Proxmox reinstall, and benefit from Synology's snapshot and backup capabilities.

This post covers the full setup: creating the iSCSI target on the Synology, connecting Proxmox over the dedicated nic1 storage link, and adding the LVM storage pool that containers actually use.

---

## Why a dedicated storage NIC

iSCSI is sensitive to latency and competes for bandwidth with VM traffic if they share a NIC. Leviathan's second NIC (nic1) is connected to switch port 3, which is an untagged access port on the abyss VLAN (50) — a completely separate path to the NAS at 192.168.50.10.

This means iSCSI storage traffic never touches the same wire as VM networking. The NAS sees a clean dedicated 2.5G connection from Proxmox and nothing else is on that link.

---

## Setting up iSCSI on the Synology

iSCSI on Synology is managed through **iSCSI Manager** in DSM.

**Step 1 — Create a target**

**iSCSI Manager → Target → Create**

- Target name: `proxmox-storage`
- No authentication (CHAP) for now — the abyss VLAN firewall rules already restrict who can reach the NAS

![Synology iSCSI Manager — Target tab showing proxmox-storage target](/images/proxmox-target.png)
*Synology iSCSI Manager — the proxmox-storage target*

**Step 2 — Create a LUN**

**iSCSI Manager → LUN → Create**

- LUN name: `proxmox-lun`
- Location: Volume 1
- Size: 200GB
- Provisioning: Thick (pre-allocates space — more predictable performance than thin)
- Map to: `proxmox-storage` target

![Synology iSCSI Manager — LUN tab showing proxmox-lun mapped to proxmox-storage](/images/proxmox-lun.png)
*Synology iSCSI Manager — 200GB LUN mapped to the proxmox-storage target*

---

## Adding iSCSI storage in Proxmox

**Step 1 — Add the iSCSI storage**

In Proxmox: **Datacenter → Storage → Add → iSCSI**

| Field | Value |
|---|---|
| ID | wreck-iscsi |
| Portal | 192.168.50.10 |
| Target | iqn.2000-01.com.synology:wreck.proxmox-storage |
| Content | none (raw block device — LVM goes on top) |

![Proxmox Datacenter → Storage showing wreck-iscsi added](/images/wreck-iscsi.png)
*Datacenter → Storage — wreck-iscsi pointing to the NAS portal*

Uncheck "Use LUNs directly" — you want raw block access so LVM can sit on top. If Proxmox can't reach the target, verify the nic1/vmbr1 interface is up and the abyss firewall rules allow iSCSI (port 3260) from midnight to abyss.

**Step 2 — Add LVM storage on top**

**Datacenter → Storage → Add → LVM**

| Field | Value |
|---|---|
| ID | wreck-lvm |
| Base storage | wreck-iscsi |
| Volume Group | wreck-vg |
| Content | Disk image, Container |

![Proxmox Datacenter → Storage showing wreck-lvm on top of wreck-iscsi](/images/wreck-lvm.png)
*wreck-lvm sitting on top of wreck-iscsi — this is what containers actually use*

After adding, `wreck-lvm` appears in the storage list and is available when creating new containers. Any container created on `wreck-lvm` is backed by the NAS LUN.

---

## Verifying the connection

Check that the iSCSI session is established:

```bash
iscsiadm -m session
```

You should see a session to `192.168.50.10:3260`. If not, check that `open-iscsi` is running:

```bash
systemctl status open-iscsi
iscsiadm -m discovery -t sendtargets -p 192.168.50.10
```

Check the LVM volume group is visible:

```bash
vgs
pvs
```

You should see `wreck-vg` with the 200GB LUN as its physical volume.

---

## Moving containers to iSCSI storage

Older containers (100–103) were created before iSCSI was set up and live on `local-lvm`. They work fine there and don't need to be moved — `local-lvm` on the M.2 is actually faster for read-heavy workloads.

New containers from CT 104 onwards were created directly on `wreck-lvm`. The naming convention makes it obvious which storage a container is on at a glance.

For any container you do want to migrate, Proxmox has a built-in move disk function: right-click the container → **Move Volume** → select `wreck-lvm` as the target. The container needs to be stopped during the move.

![Proxmox storage view showing wreck-lvm with used and available space](/images/wreck-storageview.png)
*wreck-lvm storage view — space used by containers on the NAS LUN*

---

## A note on Synology quirks

The NAS SSH setup has its own set of gotchas documented in the pfSense hardening post. For iSCSI specifically, a few things worth knowing:

**Thick vs thin provisioning** — I chose thick (eager zeroed). Thin provisioning saves space but adds latency on first write as blocks are allocated. For a home lab hypervisor, the predictability of thick is worth it.

**Port 3260** — iSCSI uses TCP port 3260. If you have tight firewall rules on abyss (which you should), add an explicit pass rule allowing TCP port 3260 from the midnight subnet to the NAS IP before trying to connect.

**Snapshots** — Synology supports iSCSI LUN snapshots via Snapshot Manager. Worth setting up a daily snapshot schedule on `proxmox-lun` once everything is stable.

---

## What's next

Storage sorted — containers running on wreck-lvm are backed by the NAS and survive a Proxmox rebuild. Leviathan is now a proper hypervisor with networked storage.

Next up: Tailscale — setting up remote access to the whole lab without opening any ports to the internet.
