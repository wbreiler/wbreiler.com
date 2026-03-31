---
title: "PXE Booting the Homelab with netboot.xyz in a Proxmox LXC"
description: "How I set up netboot.xyz inside a Proxmox LXC (pxe-nash) to handle PXE booting across the homelab — DHCP config, proxy DHCP, and what I actually use it for."
pubDate: 2026-03-31
tags:
  - proxmox
  - lxc
  - homelab
  - networking
  - linux
---

PXE booting in a homelab usually means one of two things: a dedicated Raspberry Pi running dnsmasq, or a lot of hours lost to writing iPXE scripts and managing a TFTP tree. Neither is great. netboot.xyz sidesteps most of that pain, and running it in a Proxmox LXC means it lives alongside everything else in the cluster, backed up with everything else, no extra hardware required.

My LXC for this is called `pxe-nash`. Here's how it's set up and why.

## Why netboot.xyz

A traditional PXE setup has you hosting a TFTP server, a bootloader binary (usually `pxelinux.0` or `ipxe.efi`), and then a menu config that points at kernel images and initrds you've downloaded and are hosting yourself. That's a lot of moving parts to maintain. When Ubuntu releases a new version, you update the files. When you add a new architecture, you add more files. It compounds.

netboot.xyz takes a different approach: it's a self-contained iPXE bootloader that pulls its menu and all OS images from the internet at boot time. The TFTP server hosts one file — the netboot.xyz bootloader. Everything else is fetched on demand. You get a live-updated menu of Linux distros, rescue environments, and utilities without managing any of it yourself.

What you actually host locally:

- A TFTP server serving the netboot.xyz bootloader binary
- An HTTP server serving a local assets cache (optional, but useful if you want faster boots or air-gapped support)
- Optionally, a custom menu layer for your own local boot targets

The official netboot.xyz project ships a Docker image that bundles all of this. Running it in an LXC instead of Docker just means a slightly more manual setup — but you get native Proxmox integration for backups, snapshots, and resource limits.

## Setting Up the LXC

Start from a Debian 12 template. I use unprivileged containers for everything, and this one is no exception — netboot.xyz doesn't need any elevated host access.

Resource allocation is minimal. PXE booting is not CPU- or memory-intensive; it's basically file serving:

- **1 vCPU** — more than enough
- **512MB RAM** — comfortable with headroom
- **4GB disk** — enough for the OS and a local assets cache

Network: one NIC on your main homelab VLAN. This matters more than the resource allocation — covered in the next section.

Once the container is up:

```bash
apt update && apt install -y dnsmasq tftpd-hpa wget curl
```

Grab the netboot.xyz bootloader files. You want both the BIOS and UEFI versions:

```bash
mkdir -p /var/lib/tftpboot
cd /var/lib/tftpboot

# Legacy BIOS
wget https://boot.netboot.xyz/ipxe/netboot.xyz.kpxe

# UEFI
wget https://boot.netboot.xyz/ipxe/netboot.xyz.efi
```

Configure tftpd-hpa to serve from that directory. Edit `/etc/default/tftpd-hpa`:

```ini
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/var/lib/tftpboot"
TFTP_ADDRESS=":69"
TFTP_OPTIONS="--secure"
```

Restart and enable it:

```bash
systemctl restart tftpd-hpa
systemctl enable tftpd-hpa
```

At this point `pxe-nash` is serving the netboot.xyz bootloader over TFTP. The remaining question is how clients find it.

## Network Config: Proxy DHCP vs. Full DHCP

This is where most homelab PXE setups go wrong or get overcomplicated. You have two real options:

**Option 1 — Full DHCP server on pxe-nash.** You run dnsmasq as a full DHCP server, handing out IPs *and* telling clients where to find the TFTP server. This works but almost certainly conflicts with your existing router or DHCP server. Unless you're carving out a dedicated PXE VLAN where pxe-nash is the only DHCP server, don't do this.

**Option 2 — Proxy DHCP.** dnsmasq runs in proxy mode: it listens for DHCP traffic but doesn't hand out IPs. When it sees a PXEboot request, it responds only with the boot filename and TFTP server address. Your existing DHCP server handles IPs as normal. This is the right answer for most homelabs.

To configure dnsmasq as a proxy DHCP server, edit `/etc/dnsmasq.conf` (or drop a file in `/etc/dnsmasq.d/`):

```ini
# Don't function as a DNS server
port=0

# Enable DHCP proxy on the interface facing your homelab network
interface=eth0

# Proxy DHCP — respond to PXE requests but don't hand out IPs
# Replace with your actual subnet
dhcp-range=192.168.1.0,proxy

# Tell PXE clients where to find the TFTP server
# Replace with pxe-nash's IP
dhcp-boot=netboot.xyz.kpxe,pxe-nash,192.168.1.x

# UEFI support — serve the EFI bootloader to UEFI clients
dhcp-match=set:efi-x86_64,option:client-arch,7
dhcp-match=set:efi-x86_64,option:client-arch,9
dhcp-boot=tag:efi-x86_64,netboot.xyz.efi

# Enable the built-in TFTP server
enable-tftp
tftp-root=/var/lib/tftpboot
```

A few things worth calling out here:

- `port=0` disables DNS. dnsmasq by default also runs a DNS resolver. You don't want that competing with your existing DNS server.
- The `dhcp-range` line with `,proxy` at the end is what makes this proxy mode rather than a full DHCP server. dnsmasq will respond to PXE-aware DHCP requests but ignore everything else.
- The `client-arch` matching handles UEFI vs. BIOS automatically. Option 7 is EFI x86-64, option 9 is EFI x86-64 (some vendor variant). Both should get the EFI bootloader; everything else gets the `.kpxe`.

If your homelab network runs on a different subnet, adjust `dhcp-range` accordingly.

Restart dnsmasq:

```bash
systemctl restart dnsmasq
systemctl enable dnsmasq
```

## Testing It

Boot any machine on the network and tell it to PXE boot. For a quick test, a VM in Proxmox with network boot enabled works well — set the boot order to "Network" in the VM options and start it. If dnsmasq is seeing the request and responding, the machine will fetch the netboot.xyz bootloader over TFTP and drop you into the netboot.xyz menu within a few seconds.

If it's not working, check in order:

1. `tcpdump -i eth0 port 67 or port 68` on pxe-nash — are you seeing the DHCP requests?
2. `journalctl -u dnsmasq -f` — is dnsmasq responding to them?
3. `tcpdump -i eth0 port 69` — is TFTP traffic happening after the DHCP exchange?
4. Firewall rules — Proxmox LXCs don't have a firewall by default, but if you've enabled it, make sure UDP 67, 68, and 69 are open.

## What I Actually Use It For

The honest answer is that most days pxe-nash runs quietly and does nothing. PXE booting is one of those infrastructure pieces that earns its keep in the moments you actually need it.

In practice, I reach for it when:

- **Installing new machines.** Boot to the netboot.xyz menu, pick Debian or Ubuntu, get through the installer over the network. No USB drive needed.
- **Running memtest.** The netboot.xyz menu includes memtest86+. When a machine is acting strange, boot it into memtest overnight without touching a USB stick.
- **Rescue environments.** Clonezilla and SystemRescue are both in the menu. Useful when a machine won't boot from its own disk and you need to get in without hunting for bootable media.
- **Re-deploying LXC hosts.** When I rebuild a Proxmox node, it's faster to PXE boot a Debian installer than dig out a USB.

## Custom Menu Entries

netboot.xyz supports a local menu layer — you can add your own entries that appear alongside the standard distro list, pointing at local preseed files, kickstarts, or custom iPXE scripts. I haven't needed it yet. The stock menu covers everything I reach for.

## Why Not Just Use Docker?

If you're already running Proxmox, yes. The setup is slightly more manual than running the official Docker image, but the result integrates cleanly with Proxmox backups and the resource overhead is negligible. `pxe-nash` uses about 40MB of RAM at idle.

The bigger benefit is operational consistency. When I snapshot the cluster or run backups, `pxe-nash` is included automatically. When I want to check what's running and what resources it uses, it's in the same Proxmox UI as everything else. One less Docker daemon to think about.

PXE booting isn't something you actively manage — it's infrastructure you set up once and rely on. An LXC that runs quietly, stays backed up, and doesn't demand attention is exactly the right home for it.
