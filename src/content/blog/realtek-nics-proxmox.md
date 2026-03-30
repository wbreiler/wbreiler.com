---
title: "Realtek NICs in a Proxmox Cluster: A Cautionary Tale"
description: "Network instability, corosync meltdowns, and driver headaches — why I eventually pulled every Realtek NIC out of my homelab cluster and what I'd tell someone about to make the same mistake."
pubDate: 2026-03-30
tags:
  - proxmox
  - networking
  - homelab
  - hardware
  - linux
---

If you've spent any time in homelab forums, you've seen the Realtek NIC warnings. I read them. I built a couple of nodes with Realtek NICs anyway because the boards were cheap and I figured the warnings were overblown. They were not overblown. This is what actually happened, and what I'd tell past me before clicking "add to cart."

## The Symptoms

It started subtle. Occasional corosync warnings in the Proxmox logs — nothing that caused a real problem at first, just noise. Then a node would go briefly unreachable, come back on its own, and corosync would settle. I chalked it up to misconfiguration and kept tuning.

The clearer signal came when I migrated my cluster network to a new subnet. Reconfiguring the network stack on a node with a Realtek NIC was an experience. The NIC would lose its head partway through — link would drop, come back, drop again. Corosync, which has a very short tolerance for packet loss and latency spikes on the cluster network, started marking the node as unreachable. What should have been a ten-minute network reconfiguration turned into a multi-hour recovery session involving forced node removal and re-adding it to the cluster.

Under sustained load the instability got worse. Heavy VM migrations, bulk storage operations, anything that pushed real traffic through the NIC would occasionally cause it to just... stop. Not a clean drop — the link light stayed on, the interface showed as up, but traffic stopped flowing. Reloading the driver brought it back:

```bash
modprobe -r r8169 && modprobe r8169
```

That command became muscle memory. That's not a place you want to be with cluster networking.

## The r8168 Workaround

The standard advice for Realtek cards in Linux is to ditch the in-kernel `r8169` driver and install Realtek's own `r8168` driver via DKMS. Proxmox even has a package for it:

```bash
apt install pve-headers
apt install r8168-dkms
```

After a reboot, `r8168` takes over and `r8169` is suppressed. I did this on the affected nodes and it helped — the random stalls became less frequent, and the driver-reload ritual got less common.

But "less frequent" is not "fixed." I still got corosync quorum warnings under heavy traffic. The NIC still occasionally needed a kick. And any time I touched the network configuration on those nodes, I was holding my breath. The r8168 driver is a band-aid, not a solution. It brings the cards to a functional level for light workloads, but it's still Realtek hardware with all the underlying reliability characteristics that implies.

## What Finally Pushed Me Out

The breaking point was a failed live migration. A VM mid-migration, Realtek NIC on the source node decides it's done cooperating, corosync loses quorum, Proxmox locks up the migration to protect data integrity. The VM ended up in a suspended state that took a while to sort out. No data loss, but real downtime and a real headache.

At that point I did the math. The cost difference between a board with a Realtek NIC and one with an Intel NIC is not large — maybe $20-40 depending on the platform. I had already spent more than that in time dealing with driver issues. I swapped the affected nodes to boards with Intel i225 NICs and the cluster network has been unremarkable ever since. Unremarkable is exactly what you want cluster networking to be.

## What I'd Tell Someone Building a Homelab Node

Don't buy a board with a Realtek NIC for a Proxmox node. I know the N100 mini PCs are tempting — they're cheap, efficient, and half of them ship with Realtek NICs. The savings evaporate the first time you're debugging a corosync split-brain at midnight because your NIC decided to take a nap.

Look for Intel NICs — the i210, i211, i225, and i226 are all solid. The i225 and i226 show up in a lot of the current-gen mini PCs and small form factor boards. Some of the N100 boards specifically ship with Intel NICs; it's worth the extra few minutes of research to find them.

If you already have a node with a Realtek NIC and it's been fine, maybe you'll get lucky. But if you're building a cluster where corosync reliability actually matters — where you're going to do live migrations, where you want HA to work when it needs to — don't gamble on it. Intel NICs are boring. In this context, boring is the right answer.
