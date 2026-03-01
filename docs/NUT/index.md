# NUT UPS Setup — Overview

This section covers the full setup of Network UPS Tools (NUT) for a MinuteMan UPS connected to a Raspberry Pi 3B, with clients on all machines in the rack.

## Hardware

| Component | Details |
|-----------|---------|
| UPS | MinuteMan Pro 1000 (1000VA / 700W) |
| NUT Server | Raspberry Pi 3B — Raspberry Pi OS |
| Connection | USB (reports as Delta Electronics HID device) |

## Devices on UPS

| Device | NUT Client | Notes |
|--------|------------|-------|
| Main server (Proxmox) | Yes — Stage 3 | Hosts TrueNAS VM, LXCs, VMs |
| Backup server (TrueNAS SCALE) | Yes — Stage 1 | Also runs Proxmox Backup Server container |
| Raspberry Pi | Yes — Stage 3 (master) | NUT server itself |
| Router (OPNsense) | No | Left running to keep network alive |
| Brocade switch | No | Left running to keep network alive |

## Shutdown Strategy

NUT clients use **runtime remaining** as the shutdown trigger — battery percentage is unreliable on this UPS model. Treat 50% battery as effectively dead.

| Stage | Machines | Runtime Trigger | Reasoning |
|-------|----------|-----------------|-----------|
| 1 | NFS/SMB client LXCs, qbit2 (Windows VM), Backup TrueNAS | 15 min (900s) | Unmount shares before TrueNAS goes down |
| 2 | Main TrueNAS VM | 10 min (600s) | Shut down after all dependents are clear |
| 3 | Proxmox + Raspberry Pi | 5 min (300s) | Last machines off before network gear loses power |

## Pages in This Section

- [NUT Server (Pi)](server.md) — Install and configure the NUT server on the Raspberry Pi
- [NUT Clients](clients.md) — Configure NUT clients on Debian LXCs, TrueNAS, Proxmox, and Windows
- [Home Assistant](homeassistant.md) — Integrate NUT with Home Assistant and build the dashboard
- [Proxmox Shutdown Order](proxmox.md) — Configure VM/LXC startup and shutdown ordering
