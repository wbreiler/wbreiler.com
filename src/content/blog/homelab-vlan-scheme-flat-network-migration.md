---
title: "My VLAN Scheme: How I Structured a Flat Home Network Into Something Manageable"
description: "A walkthrough of migrating a everything-on-one-subnet homelab to a properly segmented VLAN scheme, and the reasoning behind each segment."
pubDate: 2026-04-28
tags:
  - homelab
  - networking
  - proxmox
  - infrastructure
---

For a long time, everything in my homelab lived on `192.168.1.0/24`. Servers, virtual machines, IPMI interfaces, storage traffic, guests — all of it, shoulder to shoulder on the same flat subnet. It worked, technically. But "technically works" is doing a lot of heavy lifting when your storage replication traffic is sharing a broadcast domain with your IoT devices and your IPMI is reachable from anything on the network.

At some point I had enough nodes and enough things that could go wrong that I decided to actually fix it. Here's what I ended up with and why.

## The Hardware

Core switching is an Arista 7050SX, with a UniFi Console Gateway (UCG) sitting upstream as the router. The Arista handles inter-VLAN routing for all the homelab segments — it has the horsepower to do it in hardware and it keeps that traffic off the UCG. The UCG's job is internet access and anything that needs to talk to the broader home network.

## The VLAN Scheme

I ended up with five VLANs, each with a specific job.

### VLAN 10 — IPMI (`10.10.10.0/27`)

Out-of-band management interfaces for servers — iDRAC, iLO, BMCs, whatever your hardware calls it.

The problem IPMI solves is letting you reach a server even when the OS is hung, won't boot, or you've done something dumb to the network config. The problem with IPMI on a flat network is that those interfaces are notoriously under-patched, use default credentials more often than they should, and have a long history of serious vulnerabilities. Putting them on their own VLAN means they're only reachable if you're deliberately on that segment — they're not accessible from a guest VM, a laptop on the main LAN, or anything else that ends up on the network.

A `/27` gives me 30 usable addresses, which is plenty for the number of physical servers I'll ever have.

### VLAN 20 — Storage (`10.10.20.0/27`)

Dedicated storage traffic. Proxmox Backup Server lives here at `10.10.20.2`.

Storage traffic is different from everything else. It tends to be high-bandwidth and bursty, and you don't want it competing with management traffic or guest workloads on the same wire. More importantly, you probably don't want your guest VMs to have a path to the storage network at all — if something gets compromised, you'd rather it couldn't reach the NAS directly.

By isolating storage to its own VLAN, I can apply firewall rules at the VLAN boundary rather than trying to control access host by host. Only the Proxmox hosts and PBS need to be on this segment; nothing else gets in.

### VLAN 30 — Proxmox Hosts (`10.10.30.0/28`)

Hypervisor management interfaces — the IPs you SSH to, the ones Proxmox cluster traffic runs over, and the addresses the web UI lives on.

Keeping hypervisor management separate from guest traffic matters because the management plane is where the important stuff lives. If a guest VM can reach the Proxmox management interface directly, a compromised guest has a much shorter path to doing real damage. A `/28` is 14 usable addresses, which is about right — I'm not planning to run more than a handful of physical Proxmox nodes.

### VLAN 40 — Proxmox Guests (`10.10.40.0/24`)

VMs and LXC containers. This is the biggest segment because it's where most things actually run.

The main point of this VLAN is containment. Guests can reach the internet (through the UCG's firewall rules) and can reach services they're supposed to reach, but they can't wander into the IPMI network or the storage network unless explicitly permitted. A `/24` gives me 254 addresses, which is enough headroom that I'll never be assigning IPs by hand in a hurry.

### VLAN 50 — Corosync Heartbeat (`10.10.50.0/29`)

Dedicated cluster heartbeat ring. Prometheus is `.1`, Atlas is `.2`, Nyx is `.3`.

This one's a bit more specialized. Proxmox clusters use Corosync for quorum and cluster state, and Corosync is sensitive to latency and packet loss. Running the heartbeat over a shared network means it's competing with everything else, and if something causes a spike in latency or drops some packets, Corosync can start reporting false failures and triggering fencing.

A dedicated VLAN with nothing else on it means the heartbeat traffic gets a clean path with no competition. A `/29` gives me 6 usable addresses — more than enough for a cluster that'll top out at maybe five or six nodes.

## How Routing Works

The Arista does inter-VLAN routing for all five segments. Each VLAN has a corresponding SVI (switched virtual interface) on the Arista, and that SVI is the default gateway for that subnet. Traffic between VLANs goes through the Arista's routing table, which means I can write ACLs at the Arista to control what's allowed to talk to what.

The UCG doesn't see intra-homelab traffic at all — it only handles north-south (in and out to the internet and the home LAN). This keeps the homelab routing fast and keeps the UCG from becoming a bottleneck for internal traffic.

## Was It Worth It?

Yes, pretty clearly. The migration was tedious — reassigning IPs, updating configs everywhere, fixing things I'd broken — but the result is a network that actually reflects what I care about.

The big wins:
- IPMI is no longer reachable from the guest network
- Storage traffic has its own path and doesn't compete with anything else
- The Corosync heartbeat is rock-solid since it stopped sharing a wire with everything else
- Firewall rules are much easier to reason about when they're at VLAN boundaries instead of per-host

If you're running a flat homelab network and starting to feel the pain of everything being on one subnet, the migration is annoying but not hard. Start with figuring out your traffic types and which ones you'd most want isolated — for most homelabs, IPMI and storage are the obvious first candidates. The rest follows naturally.
