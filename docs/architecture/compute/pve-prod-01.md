# pve-prod-01 — Primary Proxmox Node

**Role:** Primary Proxmox node — runs all primary VMs and LXCs  
**Status:** Active  
**Last Updated:** 2026-03-13

---

## Hardware

| Component | Detail |
|-----------|--------|
| **Model** | Minisforum MS-A2 |
| **CPU** | AMD Ryzen 9 7945HX (16C/32T, 5.4GHz boost) |
| **iGPU** | AMD Radeon 680M (available for future ML/transcoding — not used at launch) |
| **RAM** | 32GB DDR5 SO-DIMM (1x 32GB — 2nd slot empty, max 96GB) |
| **Boot Drive 1** | Samsung 980 NVMe 1TB — S/N: S64ANS0W120169T |
| **Boot Drive 2** | Sabrent Rocket NVMe 1TB |
| **Boot Config** | ZFS RAID-1 mirror (pool: rpool) — configured in Proxmox installer |
| **LAN NIC** | 2x 2.5GbE RJ45 built-in (igc + r8169) |
| **Storage NIC** | 2x 10GbE SFP+ built-in — Port 1: DAC to nas-prod-01 |

---

## Software

| Field | Value |
|-------|-------|
| **OS** | Proxmox VE 9.1.6 |
| **Kernel** | 6.17.13-1-pve |
| **Debian base** | Trixie (Debian 13) |

---

## Network

| Field | Value |
|-------|-------|
| **IP** | 192.168.10.11 |
| **VLAN** | 10 — Management |
| **Gateway** | 192.168.10.1 |
| **DNS** | 9.9.9.9 (public — VLAN 10 does not use AdGuard) |
| **Switch port** | USW-Pro-Max-24 port in 2.5GbE range (ports 17–24) |
| **Link speed** | 2500Mb/s confirmed via ethtool |

### Network Interfaces

| Interface | Type | Role |
|-----------|------|------|
| nic0 (igc) | 2.5GbE RJ45 | Management + VM/LXC traffic via vmbr0 |
| nic1 (r8169) | 2.5GbE RJ45 | Unused — no cable connected |
| vmbr0 | Linux bridge on nic0 | VLAN-aware — management untagged, VMs/LXCs tagged per guest |
| SFP+ Port 1 | 10GbE | DAC to nas-prod-01 X710 Port 1 — dedicated storage link |
| SFP+ Port 2 | 10GbE | Unused at launch |

> **vmbr0 is VLAN-aware.** All VMs and LXCs are assigned VLAN tag 30 on vmbr0 and land on the Services network (192.168.30.0/24). The management interface (192.168.10.11) runs untagged on the same bridge.

---

## Proxmox Configuration

### Apt Repositories
- Enterprise repos removed: `pve-enterprise.sources`, `ceph.sources`
- No-subscription repo added: `pve-no-subscription` (trixie)
- Subscription nag popup removed from proxmoxlib.js

### Cluster
- Member of cluster `homelab-v3`
- Node ID: 0x00000001
- QDevice configured — pi-prod-01 (192.168.10.20) acts as tiebreaker

### Storage
| Storage | Type | Path | Notes |
|---------|------|------|-------|
| local | Directory | /var/lib/vz | ISO images, CT templates |
| local-zfs | ZFS | rpool/data | VM disks, LXC rootfs |

> NAS NFS shares will be added as Proxmox storage in a later step — isos, backups.

---

## Guests

| ID | Name | Type | VLAN | IP | Status |
|----|------|------|------|----|--------|
| 100 | dns-prod-01 | LXC (Debian 12) | 30 | 192.168.30.10 | Running |
| — | docker-prod-01 | VM (Ubuntu 24.04) | 30 | 192.168.30.11 | Pending |
| — | auth-prod-01 | VM (Debian) | 30 | 192.168.30.13 | Pending |
| — | immich-prod-01 | VM (Ubuntu 24.04) | 30 | 192.168.30.14 | Pending |

---

## Installation Notes

### Boot Drive Configuration
ZFS RAID-1 mirror selected during Proxmox installer. Both NVMe drives appear in rpool — single drive failure does not kill the hypervisor. This was the primary fragility in v2 (single NVMe boot) and is fixed in v3.

### Issues Encountered
- PVE 9 uses `.sources` apt format instead of `.list` — enterprise repo cleanup required targeting `pve-enterprise.sources` and `ceph.sources` instead of the `.list` files documented for PVE 8
- MS-A2 initially connected to a 1GbE switch port (ports 1–16) — moved to 2.5GbE port (ports 17–24), confirmed 2500Mb/s

---

## NIC Selection Notes
The MS-A2 has two 2.5GbE RJ45 ports:
- **igc** (Intel I226-V) — primary port, selected for management interface. Better Linux driver support, more reliable at 2.5GbE.
- **r8169** (Realtek) — secondary port, no cable connected.

During Proxmox install, igc was selected as the management NIC. Only one cable runs from the MS-A2 to the switch — single VLAN-aware bridge (vmbr0) handles all traffic.