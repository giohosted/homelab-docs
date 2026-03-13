# pve-prod-02 — Secondary Proxmox Node

**Role:** Secondary Proxmox node — runs PBS VM and secondary AdGuard LXC  
**Status:** Active  
**Last Updated:** 2026-03-13

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
| eno1 (or similar) | 1GbE RJ45 | Management + VM/LXC traffic via vmbr0 |
| vmbr0 | Linux bridge | VLAN-aware — management untagged, VMs/LXCs tagged per guest |

---

## Proxmox Configuration

### Apt Repositories
- Enterprise repos removed
- No-subscription repo added (trixie)
- Subscription nag popup removed

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

---

## Guests

| ID | Name | Type | VLAN | IP | Status |
|----|------|------|------|----|--------|
| — | pbs-prod-01 | VM (Debian) | 30 | 192.168.30.12 | Pending |
| — | dns-prod-02 | LXC (Debian 12) | 30 | 192.168.30.15 | Pending |

---

## Design Decisions

### Single NVMe Boot — No Mirror
Single NVMe boot is acceptable for pve-prod-02 given its secondary non-critical role. If this node goes down, services running on it (PBS, secondary AdGuard) are temporarily unavailable but the primary workload on pve-prod-01 is unaffected.

### HA Not Enabled
HA (High Availability) is deliberately not enabled on this cluster. The Optiplex cannot absorb pve-prod-01's workload — enabling HA without matched hardware provides false security. Clustering is for unified management UI only.

### RAM Not Upgraded Yet
RAM left at 16GB for now — current workload is light (PBS VM + one AdGuard LXC). Upgrade to 32GB later if needed.

---

## Installation Notes

### Issues Encountered
- Proxmox installer showed "no supported hard disk found" on first boot — resolved by changing SATA controller mode from RAID/RST to AHCI in Dell BIOS (F2 → System Configuration → SATA Operation → AHCI)
- Windows was previously installed on the NVMe drive — AHCI mode change resolved installer detection issue