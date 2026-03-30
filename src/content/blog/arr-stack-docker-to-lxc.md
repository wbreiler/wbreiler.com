---
title: "Migrating the *arr Stack from Docker to Proxmox LXCs"
description: "Why I moved Sonarr, Radarr, and Prowlarr out of Docker and into LXC containers on Proxmox, and what the migration actually looked like."
pubDate: 2026-03-30
tags:
  - proxmox
  - lxc
  - docker
  - homelab
  - linux
---

I ran Sonarr, Radarr, and Prowlarr on TrueNAS SCALE for a few years. SCALE runs its app catalog on Docker under the hood, which worked fine — until I moved to a Proxmox cluster and started questioning why I was running Docker on a separate NAS box when Proxmox has its own container runtime. Turns out LXCs aren't just Docker with extra steps. They're a genuinely different tradeoff, and for a media stack on Proxmox, they come out ahead.

## Why Bother

The immediate trigger was consolidation. TrueNAS SCALE is great storage, but running the \*arrs on it always felt like an awkward fit — SCALE's app system abstracts away the Docker layer, so debugging or customizing anything means digging through an interface that's not really designed for it. When I moved to Proxmox, keeping the \*arrs on the NAS meant maintaining a dependency I didn't need. LXCs on Proxmox give you direct filesystem access and network without any container runtime overhead or abstraction layer getting in the way.

The other driver was management. With SCALE apps, config lived in the NAS's ix-applications dataset, backups were separate from everything else, and there was no easy way to snapshot or migrate individual services. Moving to LXCs meant snapshots, backups, and migrations all from the same Proxmox UI I use for everything else in the cluster.

## LXCs vs. Docker on Proxmox

The conceptual shift that tripped me up initially: LXCs are closer to lightweight VMs than to Docker containers. Each LXC has its own init system, its own `/etc/`, its own package manager. You're not pulling an image and declaring environment variables — you're configuring an actual (minimal) Linux system.

What this means in practice:

- **Installation**: you install Sonarr, Radarr, etc. as system packages via their native install scripts, not as container images.
- **Persistent data**: no volume mounts to reason about. Directories from the Proxmox host (or NFS) are bind-mounted directly into the LXC via `/etc/pve/lxc/<id>.conf`. It's explicit and survives reboots.
- **Networking**: LXCs get their own network interface and IP. No port-mapping mental overhead — Sonarr is just at `192.168.x.x:8989`, full stop.
- **Updates**: you're updating packages inside the LXC like any Linux box. More manual than `docker pull`, but more predictable.

## Permissions Across Containers

Unprivileged LXCs are the default and the right choice. The catch: unprivileged containers map host UIDs to a shifted range internally, which matters when multiple LXCs share the same media directories.

My fix was to create a common group on the Proxmox host (e.g., `media`, GID 10000) and add that GID to the media directories. Then in each LXC config, I mapped that GID consistently so all containers — Sonarr, Radarr, Prowlarr — could read and write the same paths without fighting over permissions:

```ini
# In /etc/pve/lxc/<id>.conf
lxc.idmap: u 0 100000 65536
lxc.idmap: g 0 100000 65536
```

With the media directories owned by the mapped GID and the services running under a user in that group, cross-container access just works.

## The Migration

The actual move for each service:

1. Stand up a Debian LXC from the Proxmox template library.
2. Install the service via its official install script.
3. Stop the old Docker container and copy its config/data directory to the new LXC bind-mount path.
4. Add the media bind-mount to the LXC config: `mp0: /mnt/media,mp=/media`
5. Start the service — it picks up the existing database and config automatically.

Prowlarr reconnected to Sonarr and Radarr automatically once I updated the app URLs in its settings. Sonarr and Radarr had their full history and queue state intact.

The whole stack was running in LXCs in an afternoon. No drama, no data loss.

## Worth It?

For a Proxmox-native setup, yes. The cluster feels cleaner, everything is visible from one UI, and I'm not paying VM overhead just to run a container runtime. The setup is more hands-on than pulling a Docker image, but that cost is paid once — and the result is easier to reason about long-term.
