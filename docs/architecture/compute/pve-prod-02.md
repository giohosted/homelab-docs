# pve-prod-02 — Secondary Proxmox Node

**Role:** Secondary Proxmox node — runs PBS VM, secondary AdGuard LXC, and immich-prod-01  
**Status:** Active  
**Last Updated:** 2026-03-16

---

## Hardware

| Component | Detail |
|-----------|--------|
| **Model** | Dell Optiplex 3070 Micro |
| **CPU** | Intel Core i5-9500T (6C/6T, 9th gen) |
| **RAM** | 16GB DDR4 SO-DIMM (2x 8GB Micron) |
| **Boot Storage** | 256GB NVMe — single drive, ext4 |
| **Networking** | 1x GbE RJ45 built-in |

---

## Software

| Field | Value |
|-------|-------|
| **OS** | Proxmox VE 9.1.6 |
| **Debian base** | Trixie (Debian 13) |

---

## Network

| Field | Value |
|-------|-------|
| **IP** | 192.168.10.12 |
| **VLAN** | 10 — Management |
| **Gateway** | 192.168.10.1 |
| **DNS** | 9.9.9.9 (public — VLAN 10 does not use AdGuard) |
| **Switch port** | USW-Pro-Max-24 (any port — 1GbE max NIC) |
| **Link speed** | 1000Mb/s (NIC limitation) |

### Network Interfaces

| Interface | Type | Role |
|-----------|------|------|
| eno1 | 1GbE RJ45 | Management + VM/LXC traffic via vmbr0 |
| vmbr0 | Linux bridge | VLAN-aware — management untagged, VMs/LXCs tagged per guest |

---

## Proxmox Configuration

### Apt Repositories
- Enterprise repos removed: `pve-enterprise.sources`
- No-subscription repo added (trixie)
- Subscription nag popup removed from proxmoxlib.js

### Cluster
- Member of cluster `homelab-v3`
- Node ID: 0x00000002
- Joined via join token from pve-prod-01
- QDevice configured — pi-prod-01 (192.168.10.20) acts as tiebreaker

### Storage

| Storage | Type | Notes |
|---------|------|-------|
| local | Directory | ISO images, CT templates |
| local-lvm | LVM-thin | VM disks, LXC rootfs |
| nas-isos-p2 | NFS | ISO images — `192.168.30.16:/mnt/user/isos`. Scoped to pve-prod-02 only. Added Phase 4 Wave 5. |
| pbs-prod-01 | PBS | nas-backups datastore at 192.168.30.12 |

> **nas-isos note:** The cluster-wide `nas-isos` storage entry uses `10.0.0.2` (DAC link) which is only reachable from pve-prod-01. A separate `nas-isos-p2` entry using `192.168.30.16` was added and scoped to pve-prod-02 only. The original `nas-isos` entry is scoped to pve-prod-01 only to prevent mount failures on pve-prod-02.

> local-zfs ghost entry was present after install and removed via `pvesm remove local-zfs` — pve-prod-02 uses local-lvm for VM disks.  
> local-lvm entry was also lost and re-added via: `pvesm add lvmthin local-lvm --vgname pve --thinpool data --content rootdir,images`

---

## Guests

| ID  | Name | Type | VLAN | IP | Status |
|-----|------|------|------|----|--------|
| 102 | dns-prod-02 | LXC (Debian 12) | 30 | 192.168.30.15 | Running |
| 103 | pbs-prod-01 | VM (PBS 4.1.0) | 30 | 192.168.30.12 | Running |
| 106 | immich-prod-01 | VM (Ubuntu 24.04) | 30 | 192.168.30.14 | Running — 2 vCPU, 6GB RAM, 32GB disk |

> immich-prod-01 was moved here from pve-prod-01 due to RAM headroom — pve-prod-01 was already running docker-prod-01 (8GB) and auth-prod-01 (4GB). pve-prod-02 had ample capacity. See `architecture/decisions-log.md`.

---

## Design Decisions

### Single NVMe Boot — No Mirror
Single NVMe boot is acceptable for pve-prod-02 given its secondary non-critical role. If this node goes down, services running on it (PBS, secondary AdGuard, Immich) are temporarily unavailable but the primary workload on pve-prod-01 is unaffected.

### HA Not Enabled
HA (High Availability) is deliberately not enabled on this cluster. The Optiplex cannot absorb pve-prod-01's workload — enabling HA without matched hardware provides false security. Clustering is for unified management UI only.

### RAM Not Upgraded Yet
RAM left at 16GB for now — current workload (PBS VM + AdGuard LXC + Immich VM) fits comfortably. Upgrade to 32GB later if needed.

---

## Installation Notes

### Issues Encountered
- Proxmox installer showed "no supported hard disk found" on first boot — resolved by changing SATA controller mode from RAID/RST to AHCI in Dell BIOS (F2 → System Configuration → SATA Operation → AHCI)
- Windows was previously installed on the NVMe drive — AHCI mode change resolved installer detection issue
- local-zfs ghost entry present after install — removed with `pvesm remove local-zfs`
- local-lvm entry missing after local-zfs removal — re-added with `pvesm add lvmthin`
- nas-isos NFS mount broken on pve-prod-02 — cluster-wide entry used DAC link IP (10.0.0.2) unreachable from this node. Fixed by adding separate `nas-isos-p2` storage entry scoped to pve-prod-02 using 192.168.30.16.