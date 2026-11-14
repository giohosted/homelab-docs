# Hardware Inventory

**Last Updated:** 2026-11-14 (Proxmox compute, guest placement, and post-maintenance inventory refreshed; NAS drive verification remains dated 2026-10-31 and other sections retain their own prior verification dates)

---

## Hosts

### pve-prod-01 — Primary Compute

**Role:** Primary Proxmox node. Runs primary AdGuard, the primary Docker host, and the AI Observer VM; game-server VM 106 is currently stopped.

| Component | Detail |
|---|---|
| **Model** | Minisforum MS-A2 |
| **CPU** | AMD Ryzen 9 7945HX (16C/32T, 5.4GHz boost) |
| **iGPU** | AMD Radeon 680M (available for future ML/transcoding — not used at launch) |
| **RAM** | 32GB DDR5 SO-DIMM (1x 32GB. Will add 2nd stick in the future) |
| **RAM Max** | 96GB |
| **Boot Drive 1** | Samsung 980 NVMe 1TB S/N: S64ANS0W120169T |
| **Boot Drive 2** | Sabrent Rocket NVMe 1TB |
| **Boot Config** | ZFS RAID-1 mirror — configured in Proxmox installer |
| **Networking (LAN)** | 2x 2.5GbE RJ45 built-in — uplink to USW-Pro-Max-24 for VMs |
| **Networking (Storage)** | 2x 10GbE SFP+ built-in — Port 1: DAC to NAS on own storage subnet |
| **OS** | Proxmox VE 9.1.6 |
| **IP** | 192.168.10.11 (VLAN 10 — Management) |
| **Purchased** | 2026-02-21 |
| **Source** | Amazon |
| **Price Paid** | $575 |

**Guests running on pve-prod-01 (verified November 14, 2026):**

| ID | Name | Type | IP |
|---|---|---|---|
| 100 | dns-prod-01 | LXC (Debian 12) | 192.168.30.10 |
| 101 | docker-prod-01 | VM (Ubuntu 24.04) | 192.168.30.11 |
| 107 | observer-prod-01 | VM (AI Observer) | 192.168.30.17 |

**Also present but stopped:** VM 106 (`dragonwilds-server`, VLAN 50).

**Notes:**

- Radeon 680M iGPU not configured at launch — Plex transcoding runs on the NAS via i5-13400 QuickSync. Available later for ML inference or additional transcoding if needed.
- ZFS RAID-1 mirror on boot drives fixes the main fragility from v2 (single NVMe boot). Single drive failure does not kill the hypervisor.
- Immich was originally planned for pve-prod-01, later ran on the old pve-prod-02, temporarily on pve-prod-03, and now runs on the replacement pve-prod-02. See `architecture/decisions-log.md`.
- Authentik VM 104 originally ran on pve-prod-01. It was migrated offline to pve-prod-03 on November 14, 2026, retaining its 2 vCPU, 4 GiB RAM allocation, VLAN 30, and service IP `192.168.30.13`. Post-migration application, networking, SSO, and PBS backup checks passed. VM 104 is no longer registered on pve-prod-01.
- The Authentik migration reduced critical service dependencies on pve-prod-01 ahead of its planned Proxmox VE 9.2.x update.
- Proxmox VE 9.1.6 and kernel `6.17.13-1-pve` remained installed and running during the November 14 verification. The planned 9.2.x update had not yet been performed.
- NUT network-client monitoring is installed and operational; the existing configuration served as the reference for the two newly deployed Proxmox nodes.

See `architecture/compute/pve-prod-01.md`.

---

### pve-prod-02 — Secondary Compute

**Role:** Dell OptiPlex 5000 SFF running Immich VM 105; replaced the original OptiPlex 3070 Micro on November 13, 2026.

| Component | Detail |
|---|---|
| **Model** | Dell OptiPlex 5000 SFF |
| **CPU** | Intel Core i7-12700 (12C / 20T) |
| **RAM** | 48 GB DDR4 (2×16 + 2×8 GB) |
| **Boot Storage** | 2×512 GB NVMe (SK hynix BC711 + Kioxia KBG50ZNS512G), ZFS RAID-1 mirror |
| **Networking** | Built-in 1 GbE RJ45 → USW-Pro-Max-24; trunk-all profile, VLAN-aware `vmbr0` |
| **OS** | Proxmox VE 9.2.26, kernel `7.0.14-28-pve` |
| **BIOS** | Dell OptiPlex 5000 BIOS 1.43.0 (operator-confirmed November 13) |
| **IP** | `192.168.10.12/24` (VLAN 10 — Management) |
| **Source** | Repurposed hardware; replacement of work-surplus OptiPlex 3070 Micro |

**Current guest (verified November 14, 2026):**

| ID | Name | Type | IP | Status |
|---|---|---|---|---|
| 105 | immich-prod-01 | VM (Ubuntu 24.04) | `192.168.30.14` | Running; VLAN 30 |

**Notes:**

- Mirrored `rpool` and `rpool/data` for VM storage; two EFI boot partitions managed by `proxmox-boot-tool`. Each-drive boot failover not physically tested.
- Both NVMe drives passed SMART checks on November 13 (SK hynix 0%, Kioxia 16% endurance used; zero media/data errors).
- `local-zfs` assigned across all three Proxmox nodes; each node has its **own local storage**, not shared disks.
- Shares the Unraid ISO library via `nas-isos-p2` (`192.168.30.16:/mnt/user/isos`), also assigned to pve-prod-03.
- Immich arrived via offline migration from pve-prod-03 on November 13; web UI, NFS photos mount, containers, and a manual PBS backup were verified.
- Old OptiPlex 3070 Micro (i5-9500T/16 GB/single 256 GB NVMe) was removed from the cluster after evacuating all guests. Retired PBS VM 103 was destroyed, not retained on the new node.
- Proxmox HA remains disabled.
- NUT client 2.8.1-5 was installed and configured November 13. Monitoring is enabled and active against the NAS UPS server; the Debian tmpfiles pre-start issue was corrected with a systemd drop-in. Real UPS-triggered shutdown remains untested.
- On November 14, the node was gracefully shut down, physically cleaned, reassembled, and placed on a rack shelf. It returned online and resumed hosting Immich VM 105.
- The live Proxmox check following maintenance confirmed the node online, `local-zfs` and PBS storage available, and cluster quorum healthy.
- The exact rack-unit location and switch-port connection following physical relocation have not been reverified.

See `architecture/compute/pve-prod-02.md`.

---

### pve-prod-03 — Third Compute

**Role:** Lenovo ThinkCentre M90q Tiny; third Proxmox voting node hosting secondary AdGuard CT 102 and Authentik VM 104.

| Component | Detail |
|---|---|
| **Model** | Lenovo ThinkCentre M90q Tiny |
| **CPU** | Intel Core i7-10700 (8C / 16T) |
| **RAM** | 24 GB DDR4 SO-DIMM (16 + 8 GB) |
| **Boot Storage** | 2×512 GB NVMe (Toshiba KXG60ZNV512G + SK hynix HFS512GDE9X081N), ZFS RAID-1 |
| **Networking** | 1 GbE RJ45 → USW-Pro-Max-24, port 16 in last verified layout |
| **OS** | Proxmox VE 9.2.26, kernel `7.0.14-28-pve` |
| **BIOS** | Lenovo M2WKT61A last documented; newer M2WKT65A identified for review, update completion not verified |
| **IP** | `192.168.10.13/24` (VLAN 10 — Management) |
| **Bridge** | `vmbr0` VLAN-aware, `2-4094` |

**Current guests (verified November 14, 2026):**

| ID | Name | Type | IP | Status |
|---|---|---|---|---|
| 102 | dns-prod-02 | LXC (Debian 12) | `192.168.30.15` | Running; VLAN 30; 1 vCPU, 1 GiB RAM |
| 104 | auth-prod-01 | VM (Ubuntu 24.04) | `192.168.30.13` | Running; VLAN 30; 2 vCPU, 4 GiB RAM |

**Notes:**

- Added as the third voting cluster node after removal of the former QDevice from pi-prod-01.
- `local-zfs`, `nas-isos-p2` (same NAS ISO share as node 02), and `pbs-prod-01` are available.
- On November 13, the ZFS mirror was ONLINE; Toshiba SSD reported 28% endurance used, SK hynix 0%, no known data errors.
- The ZFS mirror was checked again November 14 and remained `ONLINE` with zero reported read, write, and checksum errors.
- Temporarily hosted Immich VM 105 while old pve-prod-02 was retired; VM 105 moved to the new Dell on November 13.
- Authentik VM 104 was migrated offline from pve-prod-01 on November 14 using local ZFS disk migration. Its original CPU configuration, 4 GiB RAM, virtual disks, VLAN 30, MAC address, and service IP were retained.
- Authentik passed guest networking, Docker container, local health endpoint, Traefik, DNS, and operator-confirmed SSO checks after migration.
- A fresh PBS snapshot backup of VM 104 completed successfully from pve-prod-03 on November 14 at 14:09:45 CST.
- Independent boot from each SSD remains untested.
- BIOS update to newer Lenovo firmware was reviewed, but successful installation has not been verified with a post-maintenance firmware version readback.
- NUT client 2.8.1-5 was installed and configured November 13. A systemd drop-in corrected the Debian tmpfiles pre-start issue.
- Following the November 14 maintenance reboot, `nut-monitor.service` was verified enabled and active; `upsc ups@192.168.10.10 ups.status` returned `OL`.
- Real UPS-triggered shutdown remains untested.
- On November 14, the Lenovo was physically cleaned and placed on a rack shelf. The exact rack-unit placement and current physical switch port remain to be verified.

**Workload placement consideration:**

- Primary AdGuard CT 100 remains on pve-prod-01.
- Secondary AdGuard CT 102 and Authentik VM 104 now share pve-prod-03.
- Planned maintenance on pve-prod-01 no longer directly stops Authentik.
- A failure of pve-prod-03 would affect both secondary DNS and Authentik.
- No Proxmox HA or automatic guest failover is configured.

See `architecture/compute/pve-prod-03.md` and `services/infra/authentik.md`.

---

### Proxmox Cluster — Current Architecture

**Cluster:** `homelab-v3`  
**Topology:** Three voting Proxmox nodes  
**Quorum:** Two of three votes required  
**External QDevice:** Removed; not configured  
**Proxmox HA:** Disabled  
**Last live verification:** November 14, 2026

| Node | Hardware | Management IP | Proxmox VE | Current guests |
|---|---|---|---|---|
| pve-prod-01 | Minisforum MS-A2 | 192.168.10.11 | 9.1.6 | CT 100, VM 101, VM 107; VM 106 stopped |
| pve-prod-02 | Dell OptiPlex 5000 SFF | 192.168.10.12 | 9.2.26 | VM 105 |
| pve-prod-03 | Lenovo ThinkCentre M90q Tiny | 192.168.10.13 | 9.2.26 | CT 102, VM 104 |

All three nodes were online and the cluster was quorate during the November 14 live verification.

The former Raspberry Pi QDevice was removed before the Lenovo joined the cluster. The third Proxmox node now provides the additional quorum vote directly.

**Architecture boundaries:**

- Each Proxmox host uses local ZFS-backed guest storage.
- The shared `local-zfs` storage ID does not imply shared physical disks.
- No Proxmox HA groups or automatic guest failover policies are enabled.
- Centralized management and planned migrations are supported.
- Recovery after an unexpected host failure requires an appropriate manual recovery or restore process.
- PBS storage is available to all three nodes through the NAS-hosted PBS Docker service.

See `architecture/compute/` and `architecture/decisions-log.md` for detailed node and design documentation.

---

### nas-prod-01 — NAS

**Role:** Unraid NAS. Dual-parity array for bulk media and downloads. ZFS mirror pool for precious data (photos, backups). Plex runs here as a Docker container using QuickSync iGPU. Proxmox Backup Server also runs as a Docker container, using a direct ZFS datastore path.

| Component | Detail |
|---|---|
| **Chassis** | Rosewill RSV-L4412U (4U, 12-bay) |
| **CPU** | Intel Core i5-13400 (10C/16T — 6P + 4E cores) |
| **iGPU** | Intel UHD 730 (QuickSync — used for Plex hardware transcoding) |
| **Motherboard** | ASUS TUF Gaming Z690-Plus WiFi D4 (ATX, DDR4) |
| **RAM** | 32GB DDR4 |
| **HBA** | LSI SAS 9120-8i (migrated from previous host) |
| **Onboard LAN** | Intel 2.5GbE (built-in on TUF Z690) → USW-Pro-Max-24 — VLAN 10 management interface (192.168.10.10). Unraid web UI intended for this interface. |
| **10GbE NIC** | Dell Intel X710-DA2 (dual-port SFP+, PCIe) |
| **X710 Port 1** | DAC → pve-prod-01 SFP+ Port 1 — dedicated point-to-point storage link (10GbE, no VLAN, off LAN switch) |
| **X710 Port 2** | DAC → USW-Pro-Max-24 SFP+ Port 2 — VLAN 30 data interface (192.168.30.16). NFS exports and Plex container traffic served here. |
| **Boot** | USB flash drive (Unraid OS) |
| **OS** | Unraid 7.2.4 |
| **IP (Management)** | 192.168.10.10 — onboard 2.5GbE — VLAN 10 |
| **IP (Data/NFS)** | 192.168.30.16 — X710 Port 2 — VLAN 30 |
| **Chassis Source** | eBay |
| **Chassis Price** | $345 |
| **Motherboard Source** | Reddit r/hardwareswap |
| **Motherboard Price** | $100 |
| **NIC Source** | eBay |
| **NIC Price** | $28 |

**Notes:**

- Original mobo (Gigabyte B760M DS3H DDR4) was swapped out — it only had 1x usable PCIe slot (x16 physical, no secondary slot). Could not simultaneously fit the LSI HBA (x8 card) and 10GbE NIC (needs x4 minimum). ASUS TUF Z690 has a second x16 slot wired at x4 from the chipset, which solves the conflict cleanly. See `architecture/decisions-log.md`.
- Two-interface design intentionally separates management plane from data plane. Services on VLAN 30 reach NFS/Plex via 192.168.30.16 without needing access to the Unraid management interface on VLAN 10. General Services-to-Management traffic is restricted by firewall policy.
- Docker PBS uses Host networking, and the possibility of TCP 8007 being reachable on the management interface has not been independently ruled out. The intended management/data separation should not be interpreted as proof that every host-networked service is interface-restricted.
- i5-13400 kept over i5-13600 — Unraid is IO-bound not compute-bound. i5-13600 sold to partially offset MS-A2 cost.
- 32GB DDR4 kept; extra 2x 16GB sticks sold. No heavy VM workloads on Unraid — 32GB was considered generous for the original NAS role. Current memory requirements should also account for the PBS Docker deployment.
- X710-DA2 chosen over X520 — newer chipset, better long-term driver support, dual port for future flexibility.
- No NVMe cache pool at launch. See `architecture/decisions-log.md` for full rationale.
- PBS was migrated from the former Proxmox VM 103 to Docker on Unraid in November 2026. The retired VM was subsequently destroyed.
- The active PBS datastore uses the direct ZFS location `/mnt/zfs-mirror/backups/pbs`, mounted into the PBS container at `/backups`. This avoids the previous PBS VM's NFSv3/FUSE datastore path.
- The cluster-wide `pbs-prod-01` storage definition is shared across all three Proxmox nodes. The November 14 Authentik backup from node 03 completed successfully.

---

### pi-prod-01 — Monitoring (former QDevice host)

**Role:** Lightweight always-on monitoring device running Uptime Kuma. The former Proxmox QDevice was removed before the cluster grew to three voting nodes; older Beszel documentation may describe historical monitoring, but current Beszel service state was not reverified here.

| Component | Detail |
|---|---|
| **Model** | Raspberry Pi 4B |
| **RAM** | 4GB |
| **Boot Storage** | SD card |
| **Networking** | 1x GbE RJ45 built-in |
| **OS** | Raspberry Pi OS |
| **IP** | 192.168.10.20 (VLAN 10 — Management) |
| **Source** | Microcenter ~ 2023 |

**Notes:**

- Secondary AdGuard moved off the Pi compared to v2; it now runs as CT 102 on **pve-prod-03**, while the primary remains on pve-prod-01.
- The Raspberry Pi formerly hosted QDevice for the two-node cluster. **QDevice has been removed**; the three PVE nodes provide quorum directly.
- The Pi is no longer required for Proxmox cluster quorum.
- Uptime Kuma monitoring of both pve-prod-02 and pve-prod-03 has been confirmed by the operator.
- SD card boot is acceptable here given the lightweight role — consider USB SSD if SD card causes issues.
- The status of any leftover Pi-side `corosync-qnetd` packages or services has not been independently verified. Their presence would not establish an active QDevice in the Proxmox cluster.

---

## Network

### UDM-SE — Router / Firewall

**Role:** Primary router, firewall, UniFi controller, DHCP server, WireGuard VPN endpoint.

| Component | Detail |
|---|---|
| **Model** | UniFi Dream Machine Special Edition (UDM-SE) |
| **IP** | 192.168.10.1 (VLAN 10 — Management, gateway) |
| **Rack Position** | U1 |
| **Source** | Existing — carried forward from v2 |

---

### Core Switch

**Role:** Managed switch providing 1GbE, 2.5GbE, and 10GbE connectivity. VLAN trunking to Proxmox and other applicable hosts.

| Component | Detail |
|---|---|
| **Model** | UniFi USW-Pro-Max-24 |
| **Ports** | 16x 1GbE RJ45 + 8x 2.5GbE RJ45 + 2x 10GbE SFP+ |
| **IP** | 192.168.10.2 (VLAN 10 — Management) |
| **Rack Position** | U3 |
| **Source** | Reddit r/homelabsales |
| **Price Paid** | $560 bundled with StarTech rack |

**November 14 note:** pve-prod-02 and pve-prod-03 were physically cleaned and placed on rack shelves. Their exact rack-unit locations and physical switch-port assignments require re-verification before updating the authoritative rack-layout diagram.

---

## Power

### UPS

**Role:** Protects full stack from power loss. USB-connected to nas-prod-01 for NUT graceful shutdown.

| Component | Detail |
|---|---|
| **Model** | Tripp-Lite SMART1500LCDXL |
| **Capacity** | 1500VA / 900W |
| **Battery-Backed Outlets** | 8 |
| **USB Interface** | Yes — NUT server runs on nas-prod-01 |
| **Rack Position** | U18 (bottom) |
| **Price Paid** | ~$145 |

**Notes:**

- NUT server configured on `nas-prod-01` in Netserver mode, using the `usbhid-ups` driver. Previously documented UPS readings included On Line, 100% battery, 1500VA nominal, and approximately 15% load.
- NUT clients are now installed and operational on **all three Proxmox nodes**: `pve-prod-01` (`192.168.10.11`), `pve-prod-02` (`192.168.10.12`), and `pve-prod-03` (`192.168.10.13`).
- The two newly configured nodes use `nut-client` 2.8.1-5, `MODE=netclient`, UPS target `ups@192.168.10.10`, and secondary-monitor role. Their `nut-monitor.service` units were confirmed enabled and active on November 13, 2026.
- Both new nodes have a systemd drop-in correcting the NUT tmpfiles pre-start reference; the corrected pre-start operation returned `0/SUCCESS`.
- On November 13, remote UPS queries returned `OL` from both new nodes. Startup logs identified each as a secondary UPS monitor, with no authentication-denied or communication-loss errors observed in the reviewed journals.
- After the November 14 physical maintenance, NUT monitoring on `pve-prod-03` was reverified: `nut-monitor.service` remained enabled and active, and the remote UPS returned `OL`.
- **Documented shutdown policy (not revalidated during this deployment):** After six minutes on battery, Unraid's NUT server initiates the intended shutdown sequence. Proxmox nodes are expected to stop their guests and shut down before the NAS.
- **Outstanding validation:** Real power-loss response, actual forced-shutdown signaling, guest/host shutdown order, and power-restoration behavior have not been tested. Service health and `OL` communication checks are not substitutes for an end-to-end shutdown test.
- First hardware purchased — non-negotiable before spinning up any HDDs.

**References:** `architecture/storage/nas-prod-01.md`, `architecture/compute/pve-prod-02.md`, and `architecture/compute/pve-prod-03.md`.

---

## Rack

| Component | Detail |
|---|---|
| **Model** | StarTech 4POSTRACK18U |
| **Size** | 18U open-frame |
| **Price Paid** | Bundled with USW-Pro-Max-24 ($560 total) |

**November 14 maintenance:** The Dell OptiPlex 5000 SFF (`pve-prod-02`) and Lenovo ThinkCentre M90q Tiny (`pve-prod-03`) were cleaned, returned to service, and placed on rack shelves.

The exact rack-unit positions and updated physical cable map have not been verified. The older `architecture/hardware/rack-layout.md` still describes the retired OptiPlex 3070 Micro and should not be treated as an accurate current physical layout until reconciled.

---

## Interconnects & Cabling

### DAC Cables

| Length | Route | Model | Price |
|---|---|---|---|
| 0.5M | UDM-SE SFP+ → Switch SFP+ Port 1 (uplink) | 10Gtek | ~$10 |
| 1M | nas-prod-01 X710 Port 2 → Switch SFP+ Port 2 (VLAN 30 data/NFS) | Cable Matters | ~$15 |
| 2M | nas-prod-01 X710 Port 1 → pve-prod-01 SFP+ Port 1 (storage, off-switch) | Cable Matters | ~$17 |

**Notes:**

- 2M cable for NAS ↔ MS-A2 link accounts for rail extension slack on RSV-L4412U sliding rails (~1.5M path + margin).
- Storage traffic (NAS ↔ MS-A2) is intentionally kept off the LAN switch — dedicated DAC link keeps 10GbE storage bandwidth isolated.
- The November 14 relocation of nodes 02 and 03 may have changed physical patching. Their current switch-port mapping has not been independently reverified.

### Patch Panel & Structured Cabling

| Component | Detail |
|---|---|
| **Patch Panel** | UniFi UP-PATCH-24 (keystone style) |
| **Rack Position** | U2 |
| **Keystone Couplers** | Iwillink Cat6 shielded (10-pack) — 7 used for wall runs |
| **Blank Inserts** | Fill remaining 17 ports |
| **Patch Cables** | Monoprice SlimRun Cat6 0.5ft (patch panel U2 → switch U3) |

---

## Drives

See `architecture/storage/nas-prod-01.md` and `architecture/storage/drives.md` for complete disk identities, current pool layout, and the resolved October 31, 2026 cable incident. The table below distinguishes live-confirmed installed disks from historical spare-inventory entries.

| Drive | Installed count | Type | Current NAS role |
|---|---|---|---|
| WD Red Pro 12TB | 3 | CMR NAS | Parity 1 (`AAHKXP9H`), Parity 2 (`2AH2JSWY`), Disk 3 (`5PK7HDXF`) |
| Seagate IronWolf 6TB | 2 | CMR NAS | Data Disk 1 (`ZAD97T6M`) and Data Disk 2 (`ZAD97RT0`) |
| WD Red Plus 4TB | 2 | CMR NAS | Two-member `zfs-mirror` pool |

**Other-drive inventory (historical, not freshly verified):** The March 2026 `drives.md` recorded additional WD Red Pro 12 TB, WD Red Plus 4 TB, and IronWolf 6 TB drives as spares, in TrueNAS v2, or for sale. Their present quantities and locations were not audited during the October 31 recovery. Four Seagate SkyHawk 6 TB drives were documented in Synology for cold backups, and a Seagate Barracuda 4 TB was retired; those assignments were also not rechecked. The 14 TB Seagate formerly assigned to Parity 1 is no longer in the NAS array; its physical storage location was not verified.
