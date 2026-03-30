---
title: "How I Name Things in My Homelab (and Why It Matters)"
description: "A consistent naming scheme is boring infrastructure work that pays off every time you SSH somewhere, read a log, or add a new node."
pubDate: 2026-03-30
---

Naming infrastructure is one of those decisions that feels trivial until you have fifteen things running and can't remember which one is which. I spent some time early on thinking through a scheme I could live with, and I've been happy with it ever since. Here's what I landed on and why.

## The Scheme: function-location-number

Every host in my homelab follows the same pattern: `function-location-number`. In practice:

- `pve-nash-1`, `pve-nash-2` — Proxmox VE nodes
- `storage-nash` — TrueNAS storage server
- `pbs-nash` — Proxmox Backup Server
- `pxe-nash` — network boot server
- `grafana-nash` — monitoring

The function segment says what the machine *does*. The location says where it *is*. The number disambiguates when there are multiples of the same thing.

## Why Location

The location segment is the one people push back on. "You only have one site, why bother?" Because I won't always.

Even if I never expand beyond Nashville, location is still useful context. When I'm reading a log, glancing at a dashboard, or looking at a Prometheus target list, seeing `pve-nash-1` tells me immediately what cluster that node belongs to without having to look it up. If I ever spin up a second site — a colocation rack, a cloud region, a friend's basement — I don't have to rename everything or bolt on a suffix that breaks the existing pattern. The scheme already has a slot for it.

Infrastructure naming is one of those things where the cost of adding structure up front is nearly zero, and the cost of retrofitting it later is annoying.

## Why Consistency Pays Off

The practical benefits show up in small ways, constantly:

**SSH and autocomplete.** When every Proxmox node starts with `pve-`, typing `ssh pve-<tab>` gives me exactly the list I want. Same for any other function class. Inconsistent names mean inconsistent autocomplete.

**Logs and monitoring.** Hostnames end up in logs, Prometheus labels, Grafana dashboards, alert payloads. If your naming is consistent, you can write queries and alert rules that work across a class of hosts — `job=~"pve-.*"` — without special-casing anything. If your names are a mix of schemes accumulated over time, you're either writing ugly regex or manually maintaining lists.

**Mental load.** The real cost of bad naming is that you have to remember things. A good scheme means the name itself is information. I can infer what `pbs-nash` does and where it is without checking any documentation, because the name follows the same pattern as everything else.

## What I'd Tell Past Me

Pick a scheme before you name your first machine, not after. It doesn't have to be this one — but it should be one you can apply consistently and extend without breaking. The specific format matters less than the commitment to using it everywhere.

A naming convention you enforce only sometimes is just noise.
