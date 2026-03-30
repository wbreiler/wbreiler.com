---
title: "Running Plex in a Proxmox LXC with Stable GPU Passthrough"
description: "Moving Plex from Docker to an LXC is straightforward — until you need hardware transcoding across multiple cluster nodes. Here's the by-path fix that made it work reliably."
pubDate: 2026-03-30
tags:
  - proxmox
  - lxc
  - plex
  - gpu
  - homelab
  - linux
---

Moving Plex from TrueNAS SCALE to a Proxmox LXC is mostly painless. SCALE runs Plex as a Docker container through its app catalog, so the config and metadata are all there — you just need to extract them and drop them somewhere the LXC can reach. Point it at the same media path, restore your config, done. The part that took real time was getting hardware transcoding working _stably_ — specifically, keeping it working after live-migrating the LXC between cluster nodes.

## The Basic Setup

Stand up a Debian LXC, install Plex via the official `.deb`, bind-mount your media directory. For the config, SCALE stores Plex's app data in its ix-applications dataset — copy that directory out to your new LXC bind-mount path and Plex will pick it up as-is: library, metadata, watch history, everything. Start the service and it finds the library immediately.

For GPU access, Plex needs the DRI device nodes — typically `/dev/dri/card0` and `/dev/dri/renderD128`. The standard LXC config for this:

```config
lxc.cgroup2.devices.allow: c 226:0 rwm
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.mount.entry: /dev/dri/card0 dev/dri/card0 none bind,optional,create=file
lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file
```

Enable hardware transcoding in Plex settings, verify it works. On a single node, this is the end of the story.

## The Multi-Node Problem

On a cluster, `/dev/dri/card0` is a dynamic assignment. All three of my nodes have iGPUs, but the enumeration isn't consistent — on node A the iGPU lands on `card0`, while on nodes B and C it ends up as `card1`. After migrating the Plex LXC to node B, transcoding broke immediately: the LXC config was still pointing at `card0`, which on those nodes is the wrong device.

You could update the config after every migration, but that's manual, error-prone, and defeats the point of live migration.

## The Fix: By-Path Symlinks

The stable alternative is the symlinks under `/dev/dri/by-path/`. These are created by udev and tied to the physical PCIe address of the device, so they don't shift when other devices are present.

On each node:

```bash
ls -la /dev/dri/by-path/
```

Output looks like:

```bash
pci-0000:00:02.0-card -> ../card0
pci-0000:00:02.0-render -> ../renderD128
```

On identical hardware — same motherboard, same slot — the PCIe address is the same across nodes. Update the LXC mount entries to use the by-path symlink as the source:

```config
lxc.mount.entry: /dev/dri/by-path/pci-0000:00:02.0-card dev/dri/card0 none bind,optional,create=file
lxc.mount.entry: /dev/dri/by-path/pci-0000:00:02.0-render dev/dri/renderD128 none bind,optional,create=file
```

The `cgroup2` device allowlist entries reference major:minor numbers, not paths, so those stay the same as long as the nodes have identical GPU hardware (same major:minor on both).

After this change, Plex migrations between nodes are transparent. Hardware transcoding comes up automatically on whichever node the LXC lands on.

## If the PCIe Path Differs Between Nodes

Even with identical GPU hardware across nodes, the PCIe address can vary depending on motherboard layout. If `ls -la /dev/dri/by-path/` shows a different path on node B than node A, the by-path symlink alone won't be enough — the LXC config will still be wrong on some nodes.

In that case, Proxmox supports node-specific LXC config overrides via snippet files — you can define the correct by-path symlink per-node without duplicating the entire LXC config. The by-path approach still applies on each node; you're just pointing each node's snippet at its own correct symlink.

## One Other Gotcha

Make sure the `video` group (or whatever group owns `/dev/dri/*` on your host) is mapped into the LXC and that the Plex user is a member. Without this, the device nodes are accessible in the LXC but Plex can't open them — transcoding silently falls back to software. Checking `ls -la /dev/dri/` inside the LXC and verifying the group ownership matches what Plex runs as will save you a debugging session.
