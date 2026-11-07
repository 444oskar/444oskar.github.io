---
layout: post
title: "Building My Home Server with TrueNAS"
date: 2026-11-07
categories: [homelab, self-hosting]
tags: [truenas, zfs, tailscale, adguard-home, immich, jellyfin, networking]
---

Most of my cyber security learning happens on college equipment: big racks of switches, lab VMs, firewalls that are pre-configured in a remote lab. I wanted something sitting at home that was completely mine to build, break and fix again.
So, I built a home server running **TrueNAS Community Edition** (formerly TrueNAS SCALE), and it's been running in my living room 24/7 since March.

This post covers the hardware, how the storage is laid out, the apps that do most of the work, and the problems that I (inevitably) ran into along the way and learned from!

## The hardware

The server is built mostly from parts I already had from years of PC upgrades, plus a few upgrades:

- CPU: Intel Core i7-6700K, for general compute, old but good enough for now
- RAM: 16GB DDR4-3200, this handles my ZFS cache and containers
- Boot drive: Patriot Burst Elite 120GB SSD, this runs the TrueNAS OS only, so it doesn't need to be anything too fancy
- Fast storage: Kingston 1TB NVMe, this holds apps, metadata, anything that needs fast access
- Bulk storage: Seagate IronWolf 4TB, this stores media (movies, shows etc.) and backups
- GPU: Nvidia GTX 1660, this handles video transcoding and machine learning on Immich
- Network: Intel I226-V 2.5GbE PCIe NIC, I bought this to enable faster transfers to and from the server over my home network

That GTX 1660 is really worth its weight in gold here, it handles hardware video transcoding for Jellyfin and runs Immich's face and object recognition, so the CPU isn't stuck for ages doing either.

## Why TrueNAS?

- **ZFS file system**: Every block is checksummed, snapshots are cheap, and the tooling for replacing and expanding disks is mature and well documented. I ended up leaning on that tooling heavily (more below).
- **Apps run as Docker containers**, so what I learn here carries over to industry where virtualisation and containerisation are super common.
- **It's free!** and the web UI handles routine jobs, plus there's a full shell when the UI isn't enough to do what I want.

## Storage layout

I run two pools, split by speed:

- **NVMePool** (Kingston 1TB NVMe that I took out of my laptop): app data, databases and metadata, e.g. Immich's database and thumbnails live here, so browsing the library feels snappy since I'm not waiting for a mechanical drive's read head to find where each file is.
- **BulkPool** (Seagate IronWolf 4TB): the big data. That's mostly films and shows for Jellyfin, and the original photos and videos backed up from my phone.

Keeping the two separate means the spinning drive mostly handles big sequential reads and writes, while the NVMe handles the small random I/O that hard drives are just naturally bad at.

### No redundancy (yet!)

BulkPool is just a single drive for now. ZFS checksums will *detect* any corruption, but with only one copy of the data it can't *repair* it, and if the drive dies the pool goes with it. This isn't a super big problem for the media dataset because it's just movies I can replace anytime, but my phone backups are a different story: the Immich database lives on the NVMe, so if that goes my photo backup's albums/face tags are cooked too. The plan is to buy a second 4TB IronWolf and 1TB NVMe and turn the pools into mirrors (RAID 1), but I'm a student and drives unfortunately cost money.

Until then, my photos and videos are also backed up to Google Photos. I really feel the irony here, since replacing Google Photos was half the reason I built this server. Snapshots and a proper backup routine are next on the list.

## Upgrade 1: moving to 2.5GbE

The server started on the motherboard's gigabit port. I weighed 10GbE against 2.5GbE and went with 2.5GbE: the hardware is far cheaper, it runs cooler, and it draws less power on a machine that's on 24/7 (and a single HDD maxes out at under 2.5Gbps anyway, which is where most of my data lives). I added an **Intel I226-V** PCIe card (chosen for driver stability) and a **UGREEN 2.5G switch**.

Two things tripped me up:

**The "Add Interface" trap.** My first instinct in TrueNAS was *Network → Add Interface*, which asks you to choose Bridge, Link Aggregation or VLAN. Turns out, all three create *virtual* interfaces, and I didn't need any of them. The new physical NIC had already been detected and was sitting on the main Network screen. All I needed to do was to edit that interface, give it the server's static IP, remove the IP from the old motherboard port, then move the cable to the new switch.

**The dumb idea I talked myself out of.** My router, a Netgear RAXE300, only has one 2.5G port. To get internet speeds above 1Gbps, I briefly considered plugging the fibre ONT directly into the unmanaged switch alongside the router, my PC and the NAS. That would have put my WAN and LAN on the same flat network. The NAS would be sitting on the internet side with no firewall in front of it, local routing would break, and since my ISP only hands out one public IP, devices would end up fighting over it. WAN stays on the router (oopsie, glad I thought that through first).

## Upgrade 2: migrating 2TB to 4TB without losing anything

BulkPool started on an old 2TB Seagate Barracuda I had lying around. When I bought a second-hand 4TB IronWolf, I needed to move everything across.

**Stripe vs replace.** The obvious button is *Add VDEV*. That would have striped the two drives into one roughly 6TB pool with zero fault tolerance, so losing *either* drive would lose *everything* (hell nah). Instead I did an in-place **Replace**. ZFS copies the pool block by block onto the new disk, and the old drive is freed afterwards.

**Checking what "Force" does before clicking it.** The 4TB drive still had an NTFS partition from its previous owner, so TrueNAS wanted the *Force* option. Before ticking it, I confirmed it only wipes the *incoming* disk. The source drive is only read from during the process (and that means there was no need to reformat the drive in Windows first either).

**The resilver.** I stopped Immich so nothing was writing to the pool, started the replace, and watched it with:

```bash
watch -n 2 zpool status BulkPool
```

It finished in **3 hours 26 minutes**, with **0 errors** and drive temperatures between 31°C and 41°C. Nice!

**The capacity that wouldn't grow.** Afterwards, TrueNAS showed the 3.64 TiB disk in the pool, but usable space was stuck at **1.76 TiB**. Running `zpool set autoexpand=on` and `zpool online -e` did nothing. The cause was that the replace had recreated the *old 2TB partition boundary* on the new disk. ZFS can only grow into space the partition gives it, so the partition itself had to be resized first:

```bash
parted /dev/sdX resizepart 2 100%
zpool online -e BulkPool <disk>
```

Then clicking *Expand Pool* in the UI unlocked the full 3.64 TiB. *yippeeeeee*

## The apps

### AdGuard Home: network-wide DNS

AdGuard Home is the DNS server for my whole network. Instead of configuring each device, I pointed the router's DNS at the server, so every phone, laptop and smart TV gets ad and tracker blocking automatically. That includes devices where you can't install an ad blocker at all. (almost no ads in mobile games!!)

The query log is also genuinely one of the best learning tools I have. You find out very quickly which devices phone home constantly, and where to.

**Be careful** though, the server *must* have a static IP before you do this. If its address changes, the whole network loses DNS and it looks like the internet is down.

### Immich: my own, self-hosted Google Photos

Immich backs up my phone's photos and videos automatically. The GTX 1660 runs face recognition and object search locally, so searching my own library for "dog" or someone's name works without the photos ever leaving my living room. This is also accessible through Tailscale, so I can just connect to the Tailscale VPN on my phone and it'll back my media up wherever I am in the world :) 

### Jellyfin: films and shows

Jellyfin streams my media to any device at home or, via Tailscale, anywhere else. The GPU transcodes on the fly when a device can't play a file natively (or if network speeds are bad), and subtitles in multiple languages are picked up automatically, for whoever I'm watching stuff with.

**The permissions bug.** After enabling media deletion in Jellyfin, deleting a TV show episode failed with an error saying Jellyfin needs write access to the media folder. The fix was at the filesystem level, not in Jellyfin. First I made sure the container's host path mount wasn't set to read-only. Then I gave TrueNAS's `apps` user (UID 568) ownership of the media dataset:

```bash
chown -R 568:568 /mnt/BulkPool/<media-dataset>
chmod -R 775 /mnt/BulkPool/<media-dataset>
```

It's a good reminder that a container's permissions are only as good as the host's permissions underneath (oops again).

## Security

Since this is also a cyber blog, here are the decisions I made deliberately:

- **No ports forwarded.** The server is only reachable from my local network or over **Tailscale**. Nothing is exposed to the open internet, so getting to it from outside means being on my tailnet.
- **HTTPS on the admin UI.** I enabled HTTPS on port 443 with an automatic HTTP to HTTPS redirect, using a self-signed certificate, and the admin account has a strong password.
- **Closing the app HTTPS gap.** It took me a moment to realise that the TrueNAS SSL setting only covers the admin dashboard. The apps run on their own ports (Immich on 2283, Jellyfin on 8096) and serve plain HTTP by default. Fixing that properly means a reverse proxy. I've mapped out **Nginx Proxy Manager** with Let's Encrypt certificates, including WebSocket support for Immich and `client_max_body_size 0;` plus longer timeouts so large 4K video uploads don't fail halfway through. Tailscale Serve is the other option I'm considering.
- **Isolation.** Each app runs in its own container with only the datasets it needs mounted.

## The upgrade I didn't buy

The i7-6700K from my first PC is getting old, so I researched DDR4-compatible upgrades that would keep my existing RAM and GPU: an Intel i5-12600KF on LGA1700 and a Ryzen 7 5700X on AM4. Along the way I found an £80 second-hand bundle, a Ryzen 7 2700X with an Asus Prime A320M-K board, which looked like a bargain until I looked closer: the 2700X is a 105W chip, and the A320M-K has a basic power delivery setup with no heatsinks on the VRMs. Under sustained load, like Immich indexing a whole photo library, it would probably throttle. It would also idle hotter on a machine that never turns off, and Zen+ memory controllers can struggle to run DDR4-3200 stably. For a server like mine, a cheap upgrade can actually be more expensive.

## What's next

- **A second IronWolf drive & NVMe drive** to turn BulkPool into a mirror
- **Snapshots and a real backup strategy**, so Google Photos can finally go
- **HTTPS for the apps** via a reverse proxy or Tailscale Serve
- **A managed 2.5G switch** so I can put my smart home devices on their own VLAN, and possibly a UniFi access point with its controller running on this server, because my flat has all brick walls so an extra AP in the hallway would mean much better signal.

I'll yap some more about AdGuard Home, Immich and Jellyfin in detail in future posts :)