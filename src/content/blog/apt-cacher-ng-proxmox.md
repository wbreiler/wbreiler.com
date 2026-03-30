---
title: "Setting Up an apt-caching Proxy for a Proxmox Cluster"
description: "How I set up apt-cacher-ng to serve all my Proxmox nodes and LXCs from a single cache, cutting down on redundant downloads and making updates faster across the board."
pubDate: 2026-03-30
tags:
  - proxmox
  - debian
  - homelab
  - linux
  - networking
---

Every node in a Proxmox cluster runs Debian under the hood, and every LXC you spin up is probably also Debian or Ubuntu. Without a caching proxy, every `apt upgrade` on every node hits the upstream mirrors independently. In a cluster with a handful of nodes and a dozen LXCs, you're pulling the same packages over and over. An apt-caching proxy fixes that: the first node to request a package fetches it from the mirror, and every subsequent request is served from the local cache.

The benefits are straightforward — less bandwidth consumed on your WAN connection, faster updates on subsequent nodes since they're pulling from LAN speeds, and consistent package versions across nodes if you run updates in sequence. That last one matters more than people expect: mid-rollout mirror updates are rare but not impossible, and a cache eliminates the variable.

I run mine in a Debian LXC at `apt.homelab.lan` on port 3142. It serves all my Proxmox nodes and LXCs.

## Setting Up apt-cacher-ng

Spin up a Debian LXC — nothing special needed, 1 CPU and 512MB RAM is plenty. Give it a static IP and make sure it's reachable from your other nodes.

Install apt-cacher-ng:

```bash
apt update && apt install apt-cacher-ng
```

The default config at `/etc/apt-cacher-ng/acng.conf` works without modification, but there are two settings worth tweaking:

```ini
# How much disk space to allow for the cache (in MB). Default is 0 (unlimited).
# Set a limit if you want to cap it.
CacheDir: /var/cache/apt-cacher-ng

# Defaults to 3142 — leave it unless you have a conflict.
Port: 3142
```

If you want to set a cache size cap, find and uncomment the `ExTreshold` line — it controls how aggressively old packages are evicted. The default (unlimited) is fine for a homelab.

Enable and start the service:

```bash
systemctl enable --now apt-cacher-ng
```

Verify it's listening:

```bash
ss -tlnp | grep 3142
```

apt-cacher-ng also ships a basic web UI at `http://apt.homelab.lan:3142/acng-report.html` where you can see cache stats and trigger cache scans.

## A Note on HTTPS Repositories

apt-cacher-ng can cache HTTP repositories transparently. HTTPS is trickier — by default, it tunnels HTTPS traffic rather than caching it (it can't inspect the content of an encrypted stream without additional configuration).

For most Proxmox setups this isn't a problem. The no-subscription repository at `download.proxmox.com` is served over HTTP and caches cleanly. If you're using the enterprise repository or any HTTPS-only source, those requests will pass through the proxy uncached. That's acceptable — they still work, they just don't benefit from caching.

If you want full HTTPS caching, apt-cacher-ng supports SSL bumping via its `SslCertDir` configuration, but that requires deploying a custom CA to all your clients. Not worth it for a homelab unless you have a specific reason.

## Pointing Proxmox Nodes at the Cache

On each Proxmox node, create a proxy config file for apt:

```bash
echo 'Acquire::http::Proxy "http://apt.homelab.lan:3142";' > /etc/apt/apt.conf.d/00proxy
```

That's it. Any subsequent `apt update` or `apt upgrade` will route HTTP repository traffic through the cache. Verify it's working by running `apt update` on two different nodes and watching the cache hit count increase in the web UI.

If you want HTTPS traffic to bypass the proxy rather than tunnel through it:

```bash
cat > /etc/apt/apt.conf.d/00proxy <<EOF
Acquire::http::Proxy "http://apt.homelab.lan:3142";
Acquire::https::Proxy "DIRECT";
EOF
```

## Pointing LXCs at the Cache

Same one-liner, run inside each LXC:

```bash
echo 'Acquire::http::Proxy "http://apt.homelab.lan:3142";' > /etc/apt/apt.conf.d/00proxy
```

If you want this applied automatically to new LXCs rather than configuring each one manually, you can add it to your LXC template before cloning, or use a post-create hook script to drop the file in place. For a small cluster, doing it manually as you provision containers is fine.

## Verifying It Works

After configuring a client, run `apt update` and check the apt-cacher-ng log on the cache host:

```bash
tail -f /var/log/apt-cacher-ng/apt-cacher-ng.log
```

You'll see requests coming in. First requests show as cache misses (fetched from upstream), repeated requests from other nodes show as hits. Once you've updated all your nodes, the hit rate will climb quickly — most packages are shared across every Debian-based host in the cluster.
