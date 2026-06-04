---
title: "Tailscale: Remote Access Without Opening Ports"
date: 2026-06-03
draft: false
tags: ["homelab", "networking", "tailscale", "vpn", "remote-access"]
series: ["Homelab Journey"]
description: "Installing Tailscale on pfSense as a subnet router so every VLAN is reachable from anywhere — no port forwards, no dynamic DNS, no VPN server to maintain."
cover:
  image: "/images/banners/banner-post-12.svg"
  alt: "Tailscale: Remote Access Without Opening Ports"
  relative: false
---

At some point you need to reach your lab from outside — whether that's checking something from your phone, SSHing in while travelling, or just opening the pfSense GUI without being home. The traditional approach involves port forwards, dynamic DNS, and a VPN server to maintain. Tailscale sidesteps all of that.

Tailscale is a mesh VPN built on WireGuard. Every device on your tailnet gets a `100.x.x.x` address and can reach every other device directly, regardless of NAT or firewalls. The clever part for a home lab is **subnet routing** — you install Tailscale on one device (pfSense in this case) and configure it to advertise your entire `192.168.0.0/16` block. Every device on every VLAN becomes reachable from anywhere, through one node, without installing Tailscale on each individual server.

---

## How it works

```
MacBook (anywhere)               pfSense krakenfw
──────────────────               ────────────────
100.104.71.109      ←────────→   100.74.161.40
                                 Advertises: 192.168.0.0/16
```

When your Mac connects through Tailscale and routes are approved, traffic destined for any `192.168.x.x` address is automatically tunnelled through pfSense. You can SSH to `leviathan`, open the Proxmox UI, reach the Unifi controller — all without any port forwards or exposed services on the WAN.

No open ports. No dynamic DNS. Works through any NAT, firewall, or mobile carrier. If both sides can reach the Tailscale coordination server, they can reach each other.

---

## Installation

Tailscale is available as a pfSense package. **System → Package Manager → Available Packages** — search `tailscale` and install.

![System → Package Manager showing Tailscale package installed](/images/TODO.png)
*Package Manager — Tailscale installed and available under VPN → Tailscale*

After install, Tailscale appears under **VPN → Tailscale**.

---

## Authentication — use the CLI

The GUI auth flow has a known issue with cookie handling that causes it to fail silently. Skip it and authenticate via CLI instead.

SSH into pfSense:

```bash
# Start the daemon manually for the first time
service tailscaled onestart

# Authenticate and advertise your subnet routes
tailscale up --advertise-routes=192.168.0.0/16 --authkey=YOUR-AUTH-KEY

# Enable the daemon on boot
echo 'tailscaled_enable="YES"' >> /etc/rc.conf

# Verify it's connected
tailscale status
```

Get your auth key from the Tailscale admin console: **Settings → Keys → Generate auth key**. Use an ephemeral key for testing, a reusable key for a permanent install.

The `--advertise-routes=192.168.0.0/16` flag tells Tailscale that pfSense is willing to route traffic for the entire `192.168.0.0/16` block — which covers all six VLANs in one shot.

---

## Approve the subnet routes

Advertising routes isn't enough — you also have to approve them in the admin console. This is a deliberate security step.

**login.tailscale.com/admin/machines** → find `krakenfw` → click the `...` menu → **Edit route settings** → enable `192.168.0.0/16`.

![Tailscale admin console — krakenfw with subnet route 192.168.0.0/16 approved](/images/TODO.png)
*Tailscale admin → krakenfw — subnet route approved*

While you're in the machine settings: also click `...` → **Disable key expiry**. By default Tailscale keys expire every 180 days and require re-authentication. A pfSense router losing its Tailscale connection unexpectedly because a key expired is not a good experience.

---

## Configure the Mac client

Install Tailscale on your Mac from tailscale.com and sign in with the same account. One extra step required: in the Tailscale menu bar icon → **Use Tailscale subnets**. Without this, your Mac connects to the tailnet but doesn't actually route `192.168.x.x` traffic through pfSense.

![Tailscale menu bar on Mac showing Use Tailscale subnets enabled](/images/TODO.png)
*Tailscale menu bar — Use Tailscale subnets must be enabled to route VLAN traffic*

---

## pfSense firewall rule

Tailscale traffic arrives from the `100.64.0.0/10` CGNAT range (the Tailscale subnet). Add a pass rule on the WAN interface to let this traffic reach internal VLANs:

```
Action:      Pass
Interface:   WAN
Protocol:    any
Source:      100.64.0.0/10
Destination: 192.168.0.0/16
Description: Allow Tailscale to reach internal VLANs
```

Without this rule, pfSense drops the routed traffic before it reaches any VLAN.

---

## MagicDNS — optional but useful

Tailscale's MagicDNS feature can make your internal `.home` hostnames resolve over Tailscale, so `ssh leviathan` works the same whether you're home or not. Configure it in the admin console: **DNS → Add nameserver → Custom** → `192.168.1.1` (pfSense). Enable MagicDNS.

This tells Tailscale to use your pfSense Unbound resolver for DNS, which already knows all the internal hostnames from the host overrides set up in the DNS post.

---

## Known gotcha — General Setup resets Tailscale

This one is worth documenting clearly because it caused a complete remote access outage. Saving **System → General Setup** in pfSense — even without making any changes — can silently reset Tailscale package settings, clearing the advertised routes and unchecking the exit node flag.

**Symptoms:** Tailscale shows connected in the admin console, but nothing internal is reachable. The Mac is on the tailnet but can't reach any `192.168.x.x` address.

**Fix:**
1. Connect to pfSense locally (you'll need to be on the network)
2. **VPN → Tailscale → Settings**
3. Re-add advertised route: `192.168.0.0/16`
4. Save

**Prevention:** If you ever need to save General Setup, check Tailscale settings immediately afterwards.

---

## Devices on the tailnet

| Device | Tailnet IP | Role |
|---|---|---|
| krakenfw (pfSense) | 100.74.161.40 | Subnet router |
| manta (MacBook) | 100.104.71.109 | Client |

Any device you want remote access to doesn't need Tailscale installed — as long as krakenfw is the subnet router and the routes are approved, every device on every VLAN is reachable through it.

---

## What's next

Tailscale is running — the lab is reachable from anywhere without opening a single port. Next up: SSL certificates, so every internal service has a real trusted cert and browser security warnings are gone for good.