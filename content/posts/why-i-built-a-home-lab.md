
I've had some version of a home lab for as long as I've been in IT.

It started the way it does for most people — bringing work home because the 
curiosity doesn't clock out at 5pm. Over the years it grew into a full 8U rack 
stuffed with 19" enterprise gear humming in the corner, running personal services, 
hosting experiments, and serving as my own little proving ground separate from 
production systems at work.

It was great. It was also loud, hot, and taking up way too much space.

## The Minilab Rabbit Hole

A few months ago I fell down the r/minilab rabbit hole on Reddit — an entire 
community obsessed with building fully functional home labs in impossibly small 
form factors, many using custom 3D printed racks. I was hooked immediately.

The timing was right. I wanted to downsize, reclaim space, cut power consumption, 
and reduce the heat output that had been quietly making one room of my home 
noticeably warmer year-round. So I started over from scratch with entirely new 
hardware philosophy — small, fanless, efficient.

The current setup is a Qotom Q330G4 fanless mini PC running pfSense, a Ubiquiti 
Flex 2.5G PoE switch, a Ubiquiti U7 Lite access point, a Synology NAS, and — 
admittedly less conventional — a 2013 Mac Pro "trash can" running Proxmox. 
I know, I know. It's a power hog. But it was collecting dust in a closet, has 
dual NICs which are perfect for a dedicated storage uplink, and honestly it 
looks really cool in a lab setup. I'll upgrade it eventually. The 3D printed 
rack is still in progress — that's a future post.

## A Detour Through the Nanoscale World

In November 2024 I made a deliberate pivot to try something completely different. 
I became a Field Service Engineer maintaining electron microscopes in the Materials 
Science division — specifically the kind used in semiconductor manufacturing. The 
microscopes I work on help engineers inspect silicon wafers at the nanometer scale, 
validating the precision etching and deposition processes that produce the chips 
powering everything from your phone to AI data centers. With AI accelerating chip 
demand globally, the work feels genuinely important and I love it.

It's also given me a different perspective on technology — seeing firsthand how the 
hardware that runs the internet actually gets made. But IT has always been a parallel 
passion, and the homelab has always been running in the background. This rebuild is 
about leveling up those skills intentionally and documenting the journey.

## Why IAM Engineering

Identity and Access Management wasn't always on my radar. Then I started paying 
attention.

Every major breach you read about has an identity component — stolen credentials, 
overprivileged accounts, misconfigured SSO, compromised API keys. In an era where 
the perimeter is gone and everything lives in the cloud, identity *is* the perimeter. 
Who can access what, from where, under what conditions — that's the entire security 
model now.

I've touched IAM my whole career without fully realizing it. Okta administration, 
Active Directory management, on/offboarding workflows, endpoint access policies 
through Tanium — it was always there. I just never had the title.

I also have some personal influences I'll admit to. My brother works in the security 
and AI space. My sister-in-law works at Okta. After enough dinner conversations about 
Zero Trust, SCIM provisioning, and the sprawling complexity of modern identity — I 
was convinced. IAM engineering sits at the intersection of security, automation, and 
user experience in a way that few disciplines do. It's technical, it matters, and 
it's only getting more important.

## The Ocean Theme

Every piece of hardware and every container in this lab has an ocean or scuba diving 
name. The pfSense firewall is `krakenfw`. The Proxmox hypervisor is `leviathan`. 
The NAS is `wreck`. VLANs are named after ocean zones — `surface`, `reef`, `midnight`, 
`abyss`.

Scuba diving is one of my favorite things in the world and the names actually fit 
surprisingly well. The firewall watching over everything like a kraken. The hypervisor 
carrying all the containers like a leviathan. The storage sitting deep and quiet like 
a wreck on the ocean floor. Once I started I couldn't stop.

It makes the documentation more fun to write and honestly helps me remember what 
everything does.

## What This Blog Is

This blog is the documentation of the build — every design decision, every mistake, 
every thing I learned the hard way. It's a portfolio. It's a study journal. And it's 
for anyone else out there who's trying to break into IAM engineering or just wants to 
build something real at home instead of following a tutorial in a sandbox.

Everything here is actually running. Nothing is simulated.

Next up: the network design — why I chose VLAN segmentation, how I structured the 
pfSense firewall, and why I named everything after ocean zones.