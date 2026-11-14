# Hardware Inventory

**Last Updated:** 2026-11-13 (Proxmox compute and cluster inventory refreshed; NAS drive verification remains dated 2026-10-31 and other sections retain their own prior verification dates)

---

## Hosts

### pve-prod-01 — Primary Compute

**Role:** Primary Proxmox node. Runs primary AdGuard, Docker, Authentik, and the AI Observer VM; game-server VM 106 is currently stopped.

| Component                | Detail                                                                     |
| ------------------------ | -------------------------------------------------------------------------- |
| **Model**                | Minisforum MS-A2                                                           |
| **CPU**                  | AMD Ryzen 9 7945HX (16C/32T, 5.4GHz boost)                                 |
| **iGPU**                 | AMD Radeon 680M (available for future ML/transcoding — not used at launch) |
| **RAM**                  | 32GB DDR5 SO-DIMM (1x 32GB. Will add 2nd stick in the future)              |
| **RAM Max**              | 96GB                                                                       |
| **Boot Drive 1**         | Samsung 980 NVMe 1TB S/N: S64ANS0W120169T                                  |
| **Boot Drive 2**         | Sabrent Rocket NVMe 1TB                                                    |
| **Boot Config**          | ZFS RAID-1 mirror — configured in Proxmox installer                        |
| **Networking (LAN)**     | 2x 2.5GbE RJ45 built-in — uplink to USW-Pro-Max-24 for VMs                 |
| **Networking (Storage)** | 2x 10GbE SFP+ built-in — Port 1: DAC to NAS on own storage subnet          |
| **OS**                   | Proxmox VE 9.1.6                                                           |
| **IP**                   | 192.168.10.11 (VLAN 10 — Management)                                       |
| **Purchased**            | 2026-02-21                                                                 |
| **Source**               | Amazon                                                                     |
| **Price Paid**           | $575                                                                       |

**Guests running on pve-prod-01:**

| ID | Name | Type | IP |
|----|------|------|----|
| 100 | dns-prod-01 | LXC (Debian 12) | 192.168.30.10 |
| 101 | docker-prod-01 | VM (Ubuntu 24.04) | 192.168.30.11 |
| 104 | auth-prod-01 | VM (Ubuntu 24.04) | 192.168.30.13 |
| 107 | observer-prod-01 | VM (AI Observer) | See `services/infra/ai-observer.md` |

**Also present but stopped:** VM 106 (`dragonwilds-server`, VLAN 50).

**Notes:**

- Radeon 680M iGPU not configured at launch — Plex transcoding runs on the NAS via i5-13400 QuickSync. Available later for ML inference or additional transcoding if needed.
- ZFS RAID-1 mirror on boot drives fixes the main fragility from v2 (single NVMe boot). Single drive failure does not kill the hypervisor.
- Immich was originally planned for pve-prod-01, later ran on the old pve-prod-02, temporarily on pve-prod-03, and now runs on the replacement pve-prod-02. See `architecture/decisions-log.md`.

---

### pve-prod-02 — Secondary Compute

**Role:** Dell OptiPlex 5000 SFF running Immich VM 105; replaced the original OptiPlex 3070 Micro on November 13, 2026.

| Component | Detail |
|-----------|--------|
| **Model** | Dell OptiPlex 5000 SFF |
| **CPU** | Intel Core i7-12700 (12C / 20T) |
| **RAM** | 48 GB DDR4 (2×16 + 2×8 GB) |
| **Boot Storage** | 2×512 GB NVMe (SK hynix BC711 + Kioxia KBG50ZNS512G), ZFS RAID-1 mirror |
| **Networking** | Built-in 1 GbE RJ45 → USW-Pro-Max-24; trunk-all profile, VLAN-aware `vmbr0` |
| **OS** | Proxmox VE 9.2.26, kernel `7.0.14-28-pve` |
| **BIOS** | Dell OptiPlex 5000 BIOS 1.43.0 (operator-confirmed November 13) |
| **IP** | `192.168.10.12/24` (VLAN 10 — Management) |
| **Source** | Repurposed hardware; replacement of work-surplus OptiPlex 3070 Micro |

**Current guest:**

| ID | Name | Type | IP | Status |
|----|------|------|----|--------|
| 105 | immich-prod-01 | VM (Ubuntu 24.04) | `192.168.30.14` | Running; VLAN 30 |

**Notes:**

- Mirrored `rpool` and `rpool/data` for VM storage; two EFI boot partitions managed by `proxmox-boot-tool`. Each-drive boot failover not physically tested.
- Both NVMe drives passed SMART checks on November 13 (SK hynix 0%, Kioxia 16% endurance used; zero media/data errors).
- `local-zfs` assigned across all three Proxmox nodes; each node has its **own local storage**, not shared disks.
- Shares the Unraid ISO library via `nas-isos-p2` (`192.168.30.16:/mnt/user/isos`), also assigned to pve-prod-03.
- Immich arrived via offline migration from pve-prod-03 on November 13; web UI, NFS photos mount, containers, and a manual PBS backup were verified.
- Old OptiPlex 3070 Micro (i5-9500T/16 GB/single 256 GB NVMe) was removed from the cluster after evacuating all guests. Retired PBS VM 103 was destroyed, not retained on the new node.
- Proxmox HA remains disabled.

See `architecture/compute/pve-prod-02.md`.

---

### pve-prod-03 — Third Compute

**Role:** Lenovo ThinkCentre M90q Tiny; third Proxmox voting node hosting secondary AdGuard CT 102.

| Component | Detail |
|-----------|--------|
| **Model** | Lenovo ThinkCentre M90q Tiny |
| **CPU** | Intel Core i7-10700 (8C / 16T) |
| **RAM** | 24 GB DDR4 SO-DIMM (16 + 8 GB) |
| **Boot Storage** | 2×512 GB NVMe (Toshiba KXG60ZNV512G + SK hynix HFS512GDE9X081N), ZFS RAID-1 |
| **Networking** | 1 GbE RJ45 → USW-Pro-Max-24, port 16 in last verified layout |
| **OS** | Proxmox VE 9.2.26, kernel `7.0.14-28-pve` |
| **IP** | `192.168.10.13/24` (VLAN 10 — Management) |
| **Bridge** | `vmbr0` VLAN-aware, `2-4094` |

**Current guest:**

| ID | Name | Type | IP | Status |
|----|------|------|----|--------|
| 102 | dns-prod-02 | LXC (Debian 12) | `192.168.30.15` | Running; VLAN 30 |

**Notes:**

- Added as the third voting cluster node after removal of the former QDevice from pi-prod-01.
- `local-zfs`, `nas-isos-p2` (same NAS ISO share as node 02), and `pbs-prod-01` are available.
- On November 13, the ZFS mirror was ONLINE; Toshiba SSD reported 28% endurance used, SK hynix 0%, no known data errors.
- Temporarily hosted Immich VM 105 while old pve-prod-02 was retired; VM 105 moved to the new Dell on November 13.
- Independent boot from each SSD remains untested; BIOS update to newer Lenovo firmware identified but not confirmed performed.

See `architecture/compute/pve-prod-03.md`.

---

### nas-prod-01 — NAS

**Role:** Unraid NAS. Dual-parity array for bulk media and downloads. ZFS mirror pool for precious data (photos, backups). Plex runs here as a Docker container using QuickSync iGPU.

| Component              | Detail                                                                                                                         |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| **Chassis**            | Rosewill RSV-L4412U (4U, 12-bay)                                                                                               |
| **CPU**                | Intel Core i5-13400 (10C/16T — 6P + 4E cores)                                                                                  |
| **iGPU**               | Intel UHD 730 (QuickSync — used for Plex hardware transcoding)                                                                 |
| **Motherboard**        | ASUS TUF Gaming Z690-Plus WiFi D4 (ATX, DDR4)                                                                                  |
| **RAM**                | 32GB DDR4                                                                                                                      |
| **HBA**                | LSI SAS 9120-8i (migrated from previous host)                                                                                  |
| **Onboard LAN**        | Intel 2.5GbE (built-in on TUF Z690) → USW-Pro-Max-24 — VLAN 10 management interface (192.168.10.10). Unraid web UI only.       |
| **10GbE NIC**          | Dell Intel X710-DA2 (dual-port SFP+, PCIe)                                                                                     |
| **X710 Port 1**        | DAC → pve-prod-01 SFP+ Port 1 — dedicated point-to-point storage link (10GbE, no VLAN, off LAN switch)                         |
| **X710 Port 2**        | DAC → USW-Pro-Max-24 SFP+ Port 2 — VLAN 30 data interface (192.168.30.16). NFS exports and Plex container traffic served here. |
| **Boot**               | USB flash drive (Unraid OS)                                                                                                    |
| **OS**                 | Unraid 7.2.4                                                                                                                   |
| **IP (Management)**    | 192.168.10.10 — onboard 2.5GbE — VLAN 10 — Unraid UI only                                                                      |
| **IP (Data/NFS)**      | 192.168.30.16 — X710 Port 2 — VLAN 30 — NFS exports, Plex container traffic                                                    |
| **Chassis Source**     | eBay                                                                                                                           |
| **Chassis Price**      | $345                                                                                                                           |
| **Motherboard Source** | Reddit r/hardwareswap                                                                                                          |
| **Motherboard Price**  | $100                                                                                                                           |
| **NIC Source**         | eBay                                                                                                                           |
| **NIC Price**          | $28                                                                                                                            |

**Notes:**

- Original mobo (Gigabyte B760M DS3H DDR4) was swapped out — it only had 1x usable PCIe slot (x16 physical, no secondary slot). Could not simultaneously fit the LSI HBA (x8 card) and 10GbE NIC (needs x4 minimum). ASUS TUF Z690 has a second x16 slot wired at x4 from the chipset, which solves the conflict cleanly. See `architecture/decisions-log.md`.
- Two-interface design intentionally separates management plane from data plane. Services on VLAN 30 reach NFS/Plex via 192.168.30.16 without ever crossing into VLAN 10. Firewall hard-blocks VLAN 30 → VLAN 10, so a compromised service container cannot reach the Unraid UI.
- i5-13400 kept over i5-13600 — Unraid is IO-bound not compute-bound. i5-13600 sold to partially offset MS-A2 cost.
- 32GB DDR4 kept; extra 2x 16GB sticks sold. No heavy VM workloads on Unraid — 32GB is generous for a pure NAS.
- X710-DA2 chosen over X520 — newer chipset, better long-term driver support, dual port for future flexibility.
- No NVMe cache pool at launch. See `architecture/decisions-log.md` for full rationale.

---

### pi-prod-01 — Monitoring (former QDevice host)

**Role:** Lightweight always-on monitoring device running Uptime Kuma. The former Proxmox QDevice was removed before the cluster grew to three voting nodes; older Beszel documentation may describe historical monitoring, but current Beszel service state was not reverified here.

| Component        | Detail                               |
| ---------------- | ------------------------------------ |
| **Model**        | Raspberry Pi 4B                      |
| **RAM**          | 4GB                                  |
| **Boot Storage** | SD card                              |
| **Networking**   | 1x GbE RJ45 built-in                 |
| **OS**           | Raspberry Pi OS                      |
| **IP**           | 192.168.10.20 (VLAN 10 — Management) |
| **Source**       | Microcenter ~ 2023                   |

**Notes:**

- Secondary AdGuard moved off the Pi compared to v2; it now runs as CT 102 on **pve-prod-03**, while the primary remains on pve-prod-01.
- The Raspberry Pi formerly hosted QDevice for the two-node cluster. **QDevice has been removed**; the three PVE nodes provide quorum directly.
- Uptime Kuma monitoring of both pve-prod-02 and pve-prod-03 has been confirmed by the operator.
- SD card boot is acceptable here given the lightweight role — consider USB SSD if SD card causes issues.

---

## Network

### UDM-SE — Router / Firewall

**Role:** Primary router, firewall, UniFi controller, DHCP server, WireGuard VPN endpoint.

| Component | Detail |
|-----------|--------|
| **Model** | UniFi Dream Machine Special Edition (UDM-SE) |
| **IP** | 192.168.10.1 (VLAN 10 — Management, gateway) |
| **Rack Position** | U1 |
| **Source** | Existing — carried forward from v2 |

---

### Core Switch

**Role:** Managed 2.5GbE switch. VLAN trunking to all hosts.

| Component         | Detail                                           |
| ----------------- | ------------------------------------------------ |
| **Model**         | UniFi USW-Pro-Max-24                             |
| **Ports**         | 16x 1GbE RJ45 + 8x 2.5GbE RJ45 + 2x 10GbE SFP+ |
| **IP**            | 192.168.10.2 (VLAN 10 — Management)              |
| **Rack Position** | U3                                               |
| **Source**        | Reddit r/homelabsales                            |
| **Price Paid**    | $560 bundled with StarTech rack                  |

---

## Power

### UPS

**Role:** Protects full stack from power loss. USB-connected to nas-prod-01 for NUT graceful shutdown.

| Component | Detail |
|-----------|--------|
| **Model** | Tripp-Lite SMART1500LCDXL |
| **Capacity** | 1500VA / 900W |
| **Battery-Backed Outlets** | 8 |
| **USB Interface** | Yes — NUT server runs on nas-prod-01 |
| **Rack Position** | U18 (bottom) |
| **Price Paid** | ~$145 |

**Notes:**

- NUT server configured and running on nas-prod-01 (Netserver mode, usbhid-ups driver). UPS confirmed: On Line, 100% battery, 1500VA nominal, 15% load at current draw.
- Shutdown timer: 6 minutes on battery before initiating shutdown sequence. Proxmox nodes (NUT clients) shut down VMs/LXCs then hypervisor → Unraid shuts down last.
- First hardware purchased — non-negotiable before spinning up any HDDs.

---

## Rack

| Component | Detail |
|-----------|--------|
| **Model** | StarTech 4POSTRACK18U |
| **Size** | 18U open-frame |
| **Price Paid** | Bundled with USW-Pro-Max-24 ($560 total) |

---

## Interconnects & Cabling

### DAC Cables

| Length | Route                                                                   | Model         | Price |
| ------ | ----------------------------------------------------------------------- | ------------- | ----- |
| 0.5M   | UDM-SE SFP+ → Switch SFP+ Port 1 (WAN uplink)                           | 10Gtek        | ~$10  |
| 1M     | nas-prod-01 X710 Port 2 → Switch SFP+ Port 2 (VLAN 30 data/NFS)         | Cable Matters | ~$15  |
| 2M     | nas-prod-01 X710 Port 1 → pve-prod-01 SFP+ Port 1 (storage, off-switch) | Cable Matters | ~$17  |

**Notes:**

- 2M cable for NAS ↔ MS-A2 link accounts for rail extension slack on RSV-L4412U sliding rails (~1.5M path + margin).
- Storage traffic (NAS ↔ MS-A2) is intentionally kept off the LAN switch — dedicated DAC link keeps 10GbE storage bandwidth isolated.

### Patch Panel & Structured Cabling

| Component | Detail |
|-----------|--------|
| **Patch Panel** | UniFi UP-PATCH-24 (keystone style) |
| **Rack Position** | U2 |
| **Keystone Couplers** | Iwillink Cat6 shielded (10-pack) — 7 used for wall runs |
| **Blank Inserts** | Fill remaining 17 ports |
| **Patch Cables** | Monoprice SlimRun Cat6 0.5ft (patch panel U2 → switch U3) |

---

## Drives

_See `architecture/storage/nas-prod-01.md` and `architecture/storage/drives.md` for complete disk identities, current pool layout, and the resolved October 31, 2026 cable incident. The table below distinguishes live-confirmed installed disks from historical spare-inventory entries._

| Drive | Installed count | Type | Current NAS role |
|-------|-------:|------|------|
| WD Red Pro 12TB | 3 | CMR NAS | Parity 1 (`AAHKXP9H`), Parity 2 (`2AH2JSWY`), Disk 3 (`5PK7HDXF`) |
| Seagate IronWolf 6TB | 2 | CMR NAS | Data Disk 1 (`ZAD97T6M`) and Data Disk 2 (`ZAD97RT0`) |
| WD Red Plus 4TB | 2 | CMR NAS | Two-member `zfs-mirror` pool |

**Other-drive inventory (historical, not freshly verified):** The March 2026 `drives.md` recorded additional WD Red Pro 12 TB, WD Red Plus 4 TB, and IronWolf 6 TB drives as spares, in TrueNAS v2, or for sale. Their present quantities and locations were not audited during the October 31 recovery. Four Seagate SkyHawk 6 TB drives were documented in Synology for cold backups, and a Seagate Barracuda 4 TB was retired; those assignments were also not rechecked. The 14 TB Seagate formerly assigned to Parity 1 is no longer in the NAS array; its physical storage location was not verified.
