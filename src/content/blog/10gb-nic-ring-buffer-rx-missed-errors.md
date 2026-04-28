---
title: "Diagnosing 10Gb Throughput Issues: rx_missed_errors and NIC Ring Buffer Sizing"
description: "A routine iperf3 test on a 10Gb link revealed underwhelming throughput — rx_missed_errors, ring buffer defaults, and a one-liner fix."
pubDate: 2026-04-28
tags:
  - homelab
  - networking
  - proxmox
  - linux
  - hardware
---

I've got two servers that talk to each other a lot: Prometheus (an HPE DL380 Gen9 with dual Xeon E5-2697A v4s and 384GB of RAM) and Mnemosyne (a Dell R330 running Proxmox Backup Server). They're on different VLANs — Prometheus lives on VLAN 30 (my Proxmox hosts network, `10.10.30.0/28`) and Mnemosyne is on VLAN 20 (storage, `10.10.20.0/27`) — with an Arista 7050SX handling inter-VLAN routing. Both have 10Gb uplinks to the Arista.

I ran a quick `iperf3` test between them just to baseline the link and got numbers that were nowhere near what I expected from a 10Gb path. Not "clearly broken" bad, but noticeably off. I decided to actually dig into it rather than shrug.

## Starting with ethtool

My first stop was `ethtool -S` on the NIC on Prometheus. The `-S` flag dumps driver-level statistics, and piping through `grep missed` is a quick way to see whether the NIC is dropping frames at the hardware level before the kernel gets a chance to process them:

```bash
ethtool -S <iface> | grep missed
```

While running another `iperf3` test, I watched `rx_missed_errors` climb steadily. That's the smoking gun — it means the NIC's receive ring buffer is filling up faster than the kernel can drain it, so frames are getting dropped in hardware. The kernel never sees them, TCP has to retransmit, and throughput tanks.

## The Ring Buffer Default Problem

NIC ring buffers are fixed-size queues that sit between the hardware and the kernel. To check your current sizes and the hardware maximum:

```bash
ethtool -g <iface>
```

On Prometheus, both RX and TX were at the default of 512. The NIC's maximum was 8160. At 10Gb line rate with sustained traffic, a 512-entry ring buffer doesn't leave much room for the kernel to fall behind even momentarily — and with inter-VLAN routing in the path, there's a bit more going on than a back-to-back link.

## The Fix

Bump the ring buffers to max:

```bash
ethtool -G <iface> rx 8160 tx 8160
```

This takes effect immediately without dropping the link. I kicked off another `iperf3` run, watched the counter — `rx_missed_errors` stopped climbing, and throughput jumped to where it should be for a 10Gb link.

`ethtool -G` doesn't survive a reboot, so to make it persistent I added a `post-up` line to the interface block in `/etc/network/interfaces` on Prometheus:

```
post-up ethtool -G <iface> rx 8160 tx 8160
```

Now it runs automatically whenever the interface comes up.

## Why This Bites You at 10Gb

At 1Gb, the default ring buffer sizes are usually fine — the kernel can keep up without a lot of headroom. At 10Gb you're pushing ten times the data through the same interface, and the defaults start to show cracks under sustained load. A bulk transfer like an `iperf3` test or a large backup job is exactly the kind of workload that exposes it, because it actually saturates the link for an extended period. Casual traffic never would.

The fix is a one-liner. The hard part is knowing where to look. If you've got 10Gb links and throughput feels soft without an obvious reason, `ethtool -S <iface> | grep missed` is worth checking before you start blaming the switch or the routing.
