# Network Design

**Date:** 2026-04-08  
**Status:** Complete

## Overview

Two separate Proxmox virtual bridges are used to segment cloud services from the lab network. The lab bridge has no physical bridge port or configured upstream gateway. This reduces direct connectivity between the segments, but it is not physical isolation: the Proxmox host manages both bridges and is part of the trusted computing boundary. Host firewall policy, IP forwarding, and management-plane exposure must also be considered.

## Bridges

| Bridge | Subnet | Internet | Purpose |
|---|---|---|---|
| `vmbr0` | DHCP (LAN) | Yes | Cloud services |
| `vmbr1` | 10.10.10.0/24 | No | Lab only |

## vmbr0 — Cloud network

```
vmbr0
├── Nextcloud LXC     personal cloud, LUKS encrypted storage
└── PiHole LXC        DNS blocking (planned)
```

Connects to the physical LAN. Services here are reachable from the home network and remotely via WireGuard once configured.

## vmbr1 — Isolated lab network

```
vmbr1 (10.10.10.0/24)
├── Kali Linux VM     10.10.10.x   attack machine
└── Target VMs        10.10.10.x   Metasploitable, VulnHub, HTB
```

No upstream gateway configured. No route to vmbr0. No internet access. Attack traffic is contained entirely within this segment.

## vmbr1 configuration

Added to `/etc/network/interfaces` on the Proxmox host:

```
auto vmbr1
iface vmbr1 inet static
        address 10.10.10.1/24
        bridge-ports none
        bridge-stp off
        bridge-fd 0
```

Applied with:

```bash
systemctl restart networking
ifup vmbr1
```

## Segmentation model and limitations

A bridge with no physical uplink or configured upstream gateway reduces the lab VMs' direct path to external networks. It is not, by itself, a complete security boundary. The Proxmox host has interfaces on both networks and can potentially route or otherwise bridge traffic if forwarding, firewall, or host services permit it.

Before treating this design as a security control, validate at least:

- IP forwarding and routing behavior on the Proxmox host
- Proxmox firewall policy for the host, bridges, and guest interfaces
- Management services listening on the lab-facing host address
- Connectivity tests from lab guests to cloud guests, the host, and external networks
- Whether any VM has multiple interfaces connected to both segments

Document test commands, expected results, and observed results. Avoid describing the segments as physically isolated unless the hardware topology actually provides that separation.

## Traffic flow diagram

```
Internet
    |
 Router
    |
  vmbr0 (LAN)
    ├── Nextcloud LXC
    └── PiHole LXC (planned)

  vmbr1 (isolated — no uplink)
    ├── Kali Linux VM
    └── Target VMs

  vmbr0 ←✗→ vmbr1   (no route between them)
```

## Planned additions

- WireGuard on Proxmox host — encrypted remote access into vmbr0
- PiHole as DNS server for all vmbr0 devices
- UFW on host — deny all inbound except ports 443 and 51820
- SSH hardening — key-only auth on port 2222
