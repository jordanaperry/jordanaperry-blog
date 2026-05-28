---
title: "pfSense: Internal DNS with Unbound"
date: 2026-05-26
draft: false
tags: ["homelab", "networking", "pfsense", "dns"]
series: ["Homelab Journey"]
description: "Setting up Unbound for internal hostname resolution, DNSSEC, and the rebind protection trap everyone hits."
cover:
  image: "/images/banners/banner-post-04.svg"
  alt: "pfSense: Internal DNS with Unbound"
  relative: false
---
Once the VLANs and firewall rules are in place, the next quality-of-life improvement is DNS. Without it, reaching any internal service means remembering IP addresses. With it, `ssh leviathan` just works, `https://krakenfw.home:8443` opens the pfSense GUI, and the NAS lives at `wreck.abyss` on every device on the network.

pfSense includes Unbound as its built-in DNS resolver and it handles this beautifully.

---

## Enabling Unbound

**Services → DNS Resolver → Enable**

A few settings worth changing from defaults:

- **DNSSEC**: enable it. Unbound validates DNS responses cryptographically, preventing DNS spoofing attacks where a malicious server points your queries somewhere they shouldn't go.
- **Network interfaces**: All
- **Outgoing network interfaces**: All
- **System Domain Local Zone Type**: Static — this tells Unbound that `.home` is a local zone it handles authoritatively, not something to forward upstream.

![Services → DNS Resolver — general settings page showing DNSSEC enabled and interface config](/images/Enabling-Unbound.png)
*Services → DNS Resolver — general settings page showing DNSSEC enabled and interface config*

---

## Host overrides

Host overrides are how you map hostnames to IPs. Every device in the lab gets one. **Services → DNS Resolver → Host Overrides → Add**.

| Hostname | Domain | IP | Notes |
|---|---|---|---|
| krakenfw | home | 192.168.1.1 | pfSense (CNAME alias for regulator) |
| manifold | home | 192.168.1.2 | Unifi switch |
| buoy | home | 192.168.1.3 | Unifi AP |
| leviathan | home | 192.168.40.10 | Proxmox |
| helm | home | 192.168.40.20 | Unifi controller |
| bastion | home | 192.168.40.50 | Bastion host |
| wreck | abyss | 192.168.50.10 | NAS |
| nautilus | drift | 192.168.20.11 | Fridge |
| zapper | drift | 192.168.20.12 | Microwave |
| hydra | drift | 192.168.20.13 | Stove |

![Services → DNS Resolver → Host Overrides — full list of all entries](/images/Host-Overrides.png)
*Services → DNS Resolver → Host Overrides — full list of all entries*

`krakenfw` is added as a CNAME alias under the `regulator` host override, not as a separate entry. The system hostname is `regulator` — `krakenfw` is the friendly name. Unbound handles CNAME chaining natively, so both resolve to the same IP.

![One host override entry — showing the CNAME setup for krakenfw/regulator](/images/CNAME-Setup.png)
*One host override entry — showing the CNAME setup for krakenfw/regulator*

`wreck` uses the domain `abyss` instead of `home`, giving it the address `wreck.abyss`. Same for the IoT devices — `nautilus.drift`, `zapper.drift`, `hydra.drift`. The subdomain signals the VLAN in the hostname itself. When you're reading logs and you see `nautilus.drift` trying to make a connection somewhere, you know immediately what you're looking at: an IoT device going somewhere it shouldn't. The naming carries the threat model.

---

## Verifying resolution

After hitting Apply Changes, verify from SSH:

```bash
# Restart Unbound if needed
pfSsh.php playback execcmd "services_unbound_configure();"

# Test resolution
host krakenfw.home 192.168.1.1
host leviathan.home 192.168.1.1
host wreck.abyss 192.168.1.1
```

Each should return the correct IP. If any fail, double-check the host override entries — a typo in the domain field is the most common culprit.

---

## The rebind protection trap

DNS rebind protection is a feature built into Unbound itself — not just a pfSense GUI thing. Here's what it actually does: whenever Unbound resolves a hostname to a private RFC1918 address, it checks whether that hostname is on an allowlist. If it isn't, Unbound drops the response entirely and never returns the IP to the client.

This means the failure isn't limited to browser access. `ssh leviathan.home` fails. NAS mounts fail. Anything that relies on DNS resolution of an internal hostname fails — silently, with no obvious error pointing at DNS rebind protection as the cause. The browser case is the most visible because pfSense actually returns an explicit error message:

> *Potential DNS Rebind attack detected*

![The rebind error in a browser — if you can reproduce it before adding the allowlist](/images/FD1zqtP.png)
*The rebind error in a browser — if you can reproduce it before adding the allowlist*

Everything else just times out, which makes it much harder to diagnose.

The protection itself is legitimate. DNS rebind attacks are real — a malicious website serves a domain that initially resolves to its own IP, then switches the DNS response to an internal RFC1918 address. Your browser, still thinking it's talking to the original site, then sends requests to your internal network on the attacker's behalf. Blocking unrecognised RFC1918 resolutions cuts this off.

The fix is telling Unbound which internal hostnames are intentional. **System → Advanced → Admin Access → Alternate Hostnames**:

```
krakenfw.home regulator.home leviathan.home helm.home wreck.abyss
```

![System → Advanced → Alternate Hostnames — showing the allowlist populated](/images/Alternate-Hostnames.png)
*System → Advanced → Alternate Hostnames — showing the allowlist populated*

Space separated, all on one line. This is why all the internal hostnames need to be here — not just `krakenfw.home` for the GUI, but every device you want reachable by name. Any hostname you add to host overrides should also go here. Add them both at the same time and you'll never hit this again.

---

## Optional: DNS over TLS

If you want DNS queries leaving the network to be encrypted — so your ISP can't see what you're resolving — you can configure Unbound to forward upstream queries over TLS to Cloudflare (1.1.1.1) or Quad9 (9.9.9.9).

**Services → DNS Resolver → Display Custom Options**, add:

```
server:
    forward-zone:
        name: "."
        forward-ssl-upstream: yes
        forward-addr: 1.1.1.1@853
        forward-addr: 9.9.9.9@853
```

I haven't enabled this yet — it adds a dependency on external resolvers and I'd rather Unbound resolve everything itself for now. Worth revisiting.

---

## What's next

DNS is done — every device in the lab has a hostname that resolves correctly from any VLAN that's allowed to reach it. Next post: DHCP, static leases, and why locking down IPs by MAC address is worth the extra setup time.