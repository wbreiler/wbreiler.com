---
title: "Running 4 Minecraft Servers on a Single Proxmox Node with DiscoPanel"
description: "How I set up a small Minecraft server infrastructure for friends using DiscoPanel in an LXC container on a Proxmox homelab node."
pubDate: 2026-04-28
tags:
  - homelab
  - proxmox
  - lxc
  - gaming
---

At some point "I have a homelab" inevitably turns into "I should just host Minecraft for us." I've been running four servers — three modded, one vanilla — for a small group of friends, all out of a single LXC container on one of my Proxmox nodes. Here's how it's set up and why I made the choices I did.

## The Panel: DiscoPanel

The thing managing all four servers is [DiscoPanel](https://github.com/nickheyer/discopanel), a Docker-based game server management panel. It gives you a web UI to start, stop, and configure servers, view console output, manage files, and generally not have to SSH in every time someone wants a server restarted.

I looked at a few options before landing on DiscoPanel. Pterodactyl is the standard answer but it's genuinely overengineered for "four servers for friends" — it wants a separate Wings daemon node, a separate panel, a database, the works. DiscoPanel is leaner and runs as a single Docker Compose stack, which is all I needed.

## Why LXC Instead of a VM

I put DiscoPanel in an LXC container rather than a VM, and I'd do it again. For a workload like this, the overhead of a full VM doesn't buy you anything. LXC containers share the host kernel, which means lower memory overhead and less CPU tax per container — and when you're running four Minecraft servers, you want as much of your resources going to actual game simulation as possible, not hypervisor overhead.

The tradeoff is that LXC doesn't give you the same isolation as a VM. For a game server running modpacks from the internet, some people would prefer the extra boundary. For me, running this for a closed group of friends on a homelab, it's not a concern I lose sleep over.

The container is CT 300, with 128GB of disk — more than enough headroom for four worlds, mod files, and backups.

## The Hardware: Why Prometheus Specifically

I have a few nodes in the cluster, and I deliberately put this workload on Prometheus — an HPE DL380 Gen9 with dual Xeon E5-2697A v4 processors and 384GB of RAM.

The reason is single-thread performance. Minecraft is notoriously single-threaded for game tick processing. The E5-2697A v4 turbos up to 3.6GHz, which is the highest single-core frequency in my cluster. More cores and more RAM doesn't help much if the main game loop is bottlenecked on one thread — you want that one thread running as fast as possible.

The 384GB of RAM means I can be generous with heap allocations for each server without worrying, especially for the modded instances that tend to have enormous memory appetites.

## Networking

The LXC container lives on VLAN 40 (`10.10.40.0/24`), which is the Proxmox guests network. Minecraft's default port is 25565, and each server gets its own port so they can all be reached from the same IP. The UCG handles the port forwarding and firewall rules to expose the servers externally.

Prometheus itself is on VLAN 30 (`10.10.30.0/28`), the Proxmox management network — but the guests it runs, including this container, get addresses from VLAN 40. The networks are properly separated, which means the Minecraft container can't accidentally reach the hypervisor management interfaces.

## World Pregeneration

One thing that makes a huge difference for server performance, especially on modded instances, is pregenerating the world before anyone plays on it. When chunks are generated on-demand as players explore, the server spikes hard — chunk generation is expensive, and doing it mid-session causes noticeable lag.

I use the [Chunk Pregenerator](https://www.curseforge.com/minecraft/mc-mods/chunkpregenerator) plugin and kick it off before opening the server to players:

```
/pregen start gen radius MyPregen SQUARE 0 0 250 minecraft:overworld
```

That pregenerates a 500×500 chunk square (8km×8km) around spawn in the overworld. It runs in the background, usually takes an hour or two depending on the modpack, and the result is that normal play is dramatically smoother because the server isn't generating terrain under your feet in real time.

For modded servers with heavy worldgen (Biomes O' Plenty, TerraBlender, etc.), I sometimes bump the radius to 300 or pregenerate additional dimensions. The Nether and End can also be pregenned with the same command by swapping `minecraft:overworld` for the appropriate dimension key.

## Is It Worth Running Yourself?

If you have a homelab with a spare node and a group of friends who want to play together, absolutely. The initial setup is maybe an afternoon — get DiscoPanel running, configure the servers, sort out port forwarding, and do the pregeneration pass. After that it basically runs itself.

The main ongoing thing is keeping modpacks updated when major versions drop, which is a normal Minecraft server operator problem regardless of how you're hosting. DiscoPanel makes the actual server management (restarts, console access, file edits) easy enough that I don't think about it much.

Four servers, one container, friends playing whenever they want. Good use of a Tuesday evening's worth of setup work.
