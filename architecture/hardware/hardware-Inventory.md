# Hardware Inventory

**Last Updated:** 2026-02-28 _Update this file whenever hardware is added, swapped, or retired._

---

## Hosts

### pve-prod-01 — Primary Compute

**Role:** Primary Proxmox node. Runs all primary VMs and LXCs (docker-prod-01, auth-prod-01, immich-prod-01, dns-prod-01).

|Component|Detail|
|---|---|
|**Model**|Minisforum MS-A2|
|**CPU**|AMD Ryzen 9 7945HX (16C/32T, 5.4GHz boost)|
|**iGPU**|AMD Radeon 680M (available for future ML/transcoding — not used at launch)|
|**RAM**|64GB DDR5 SO-DIMM (2x 32GB)|
|**RAM Max**|96GB|
|**Boot Drive 1**|Samsung NVMe 1TB _(fill in exact model + serial when installed)_|
|**Boot Drive 2**|Sabrent NVMe 1TB _(fill in exact model + serial when installed)_|
|**Boot Config**|ZFS RAID-1 mirror — configured in Proxmox installer|
|**Networking (LAN)**|2x 2.5GbE RJ45 built-in — uplink to USW-Pro-Max-24|
|**Networking (Storage)**|2x 10GbE SFP+ built-in — Port 1: DAC to NAS \| Port 2: DAC to switch|
|**OS**|Proxmox VE _(fill in version when installed)_|
|**IP**|192.168.10.11 (VLAN 10 — Management)|
|**Purchased**|2026-02-21|
|**Source**|Amazon|
|**Price Paid**|$559 (before tax)|

**Notes:**

- Radeon 680M iGPU not configured at launch — Plex transcoding runs on the NAS via i5-13400 QuickSync. Available later for ML inference or additional transcoding if needed.
- ZFS RAID-1 mirror on boot drives fixes the main fragility from v2 (single NVMe boot). Single drive failure does not kill the hypervisor.

---

### nas-prod-01 — NAS

**Role:** Unraid NAS. Dual-parity array for bulk media and downloads. ZFS mirror pool for precious data (photos, backups). Plex runs here as a Docker container using QuickSync iGPU.

|Component|Detail|
|---|---|
|**Chassis**|Rosewill RSV-L4412U (4U, 12-bay)|
|**CPU**|Intel Core i5-13400 (10C/16T — 6P + 4E cores)|
|**iGPU**|Intel UHD 730 (QuickSync — used for Plex hardware transcoding)|
|**Motherboard**|ASUS TUF Gaming Z690-Plus WiFi D4 (ATX, DDR4)|
|**RAM**|32GB DDR4|
|**HBA**|LSI SAS 9120-8i (migrated from previous host)|
|**10GbE NIC**|Dell Intel X710-DA2 (dual-port SFP+, PCIe) — Port 1: DAC to MS-A2 \| Port 2: unused|
|**Onboard LAN**|Intel 2.5GbE (built-in on TUF Z690) — management uplink to USW-Pro-Max-24|
|**Boot**|USB flash drive (Unraid OS)|
|**OS**|Unraid 7.2.3|
|**IP**|192.168.10.10 (VLAN 10 — Management)|
|**Chassis Source**|Amazon|
|**Chassis Price**|$345|
|**Motherboard Source**|Reddit r/hardwareswap|
|**Motherboard Price**|~$100|
|**NIC Source**|eBay|
|**NIC Price**|~$25|

**Notes:**

- Original mobo (Gigabyte B760M DS3H DDR4) was swapped out — it only had 1x usable PCIe slot (x16 physical, no secondary slot). Could not simultaneously fit the LSI HBA (x8 card) and 10GbE NIC (needs x4 minimum). ASUS TUF Z690 has a second x16 slot wired at x4 from the chipset, which solves the conflict cleanly. See `architecture/decisions-log.md`.
- i5-13400 kept over i5-13600 — Unraid is IO-bound not compute-bound. i5-13600 sold to partially offset MS-A2 cost.
- 32GB DDR4 kept; extra 2x 16GB sticks sold. No heavy VM workloads on Unraid — 32GB is generous for a pure NAS.
- X710-DA2 chosen over X520 — newer chipset, better long-term driver support, dual port for future flexibility.
- No NVMe cache pool at launch. See `architecture/decisions-log.md` for full rationale.

---

### pve-prod-02 — Secondary Compute

**Role:** Secondary Proxmox node. Runs PBS VM (pbs-prod-01) and secondary AdGuard LXC (dns-prod-02).

|Component|Detail|
|---|---|
|**Model**|Dell Optiplex 3070 Micro|
|**CPU**|Intel Core i5-9500T (6C/6T, 9th gen)|
|**RAM**|16GB DDR4 SO-DIMM (2x 8GB Micron)|
|**Boot Storage**|_(fill in — NVMe model + capacity)_|
|**Networking**|1x GbE RJ45 built-in — uplink to USW-Pro-Max-24|
|**OS**|Proxmox VE _(fill in version when installed)_|
|**IP**|192.168.10.12 (VLAN 10 — Management)|
|**Source**|Work surplus (free)|

**Notes:**

- RAM left at 16GB for now — workload is light (PBS VM + one AdGuard LXC). Upgrade to 32GB later if needed.
- HA (High Availability) is NOT enabled in the Proxmox cluster — Optiplex cannot absorb MS-A2's workload. Clustering is for unified management UI only. See `architecture/decisions-log.md`.
- Single NVMe boot is acceptable given this node's non-critical secondary role.

---

### pi-prod-01 — Monitoring / QDevice

**Role:** Lightweight always-on device. Runs Proxmox QDevice (cluster tiebreaker), Uptime Kuma, and Beszel monitoring server.

|Component|Detail|
|---|---|
|**Model**|Raspberry Pi 4B|
|**RAM**|4GB|
|**Boot Storage**|SD card|
|**Networking**|1x GbE RJ45 built-in|
|**OS**|_(fill in — Raspberry Pi OS / Ubuntu)_|
|**IP**|192.168.10.20 (VLAN 10 — Management)|
|**Source**|_(fill in — existing? purchased when?)_|

**Notes:**

- Secondary AdGuard moved off the Pi compared to v2 — now runs as an LXC on pve-prod-02 (dns-prod-02). Pi is dedicated to QDevice + monitoring only.
- QDevice acts as tiebreaker for the 2-node Proxmox cluster to prevent split-brain. Lightweight setup, meaningful reliability value.
- SD card boot is acceptable here given the lightweight role — consider USB SSD if SD card causes issues.

---

## Network

### UDM-SE — Router / Firewall

**Role:** Primary router, firewall, UniFi controller, DHCP server, WireGuard VPN endpoint.

|Component|Detail|
|---|---|
|**Model**|UniFi Dream Machine Special Edition (UDM-SE)|
|**IP**|192.168.10.1 (VLAN 10 — Management, gateway)|
|**Rack Position**|U1|
|**Source**|Existing — carried forward from v2|

---

### Core Switch

**Role:** Managed 2.5GbE switch. VLAN trunking to all hosts.

|Component|Detail|
|---|---|
|**Model**|UniFi USW-Pro-Max-24|
|**Ports**|24x 2.5GbE RJ45 + 2x 10GbE SFP+|
|**IP**|192.168.10.2 (VLAN 10 — Management)|
|**Rack Position**|U3|
|**Source**|_(fill in — Amazon / UI store?)_|
|**Price Paid**|$560 bundled with StarTech rack|

---

## Power

### UPS

**Role:** Protects full stack from power loss. USB-connected to nas-prod-01 for NUT graceful shutdown.

|Component|Detail|
|---|---|
|**Model**|Tripp-Lite SMART1500LCDXL|
|**Capacity**|1500VA / 900W|
|**Battery-Backed Outlets**|8|
|**USB Interface**|Yes — NUT server runs on nas-prod-01|
|**Rack Position**|U18 (bottom)|
|**Price Paid**|~$145|

**Notes:**

- NUT server on nas-prod-01, NUT clients on pve-prod-01 and pve-prod-02. On low-battery signal: Proxmox VMs and LXCs shut down first → hypervisors shut down → Unraid shuts down last.
- First hardware purchased — non-negotiable before spinning up any HDDs.

---

## Rack

|Component|Detail|
|---|---|
|**Model**|StarTech 4POSTRACK18U|
|**Size**|18U open-frame|
|**Price Paid**|Bundled with USW-Pro-Max-24 ($560 total)|

---

## Interconnects & Cabling

### DAC Cables

|Length|Route|Model|Price|
|---|---|---|---|
|0.5M|UDM-SE SFP+ → Switch SFP+ (WAN uplink)|10Gtek|~$10|
|1M|Switch SFP+ → MS-A2 SFP+ (VM/LXC LAN)|Cable Matters|~$15|
|2M|NAS SFP+ (X710 Port 1) → MS-A2 SFP+ (storage traffic)|Cable Matters|~$17|

**Notes:**

- 2M cable for NAS ↔ MS-A2 link accounts for rail extension slack on RSV-L4412U sliding rails (~1.5M path + margin).
- Storage traffic (NAS ↔ MS-A2) is intentionally kept off the LAN switch — dedicated DAC link keeps 10GbE storage bandwidth isolated.

### Patch Panel & Structured Cabling

|Component|Detail|
|---|---|
|**Patch Panel**|UniFi UP-PATCH-24 (keystone style)|
|**Rack Position**|U2|
|**Keystone Couplers**|Iwillink Cat6 shielded (10-pack) — 7 used for wall runs|
|**Blank Inserts**|Fill remaining 17 ports|
|**Patch Cables**|Monoprice SlimRun Cat6 0.5ft (patch panel U2 → switch U3)|

---

## Drives

_See `architecture/storage.md` for full pool layout, assignment, and rationale. Summary below._

|Drive|Count|Type|Role|
|---|---|---|---|
|WD Red Pro 12TB|5|CMR NAS|2x parity + 3x data (parity array)|
|WD Red Plus 4TB|5|CMR NAS|2x ZFS mirror pool + 2x future array expansion + 1x cold spare|
|Seagate IronWolf 6TB|2|CMR NAS|Hot spares / future array expansion|
|Seagate SkyHawk 6TB|4|Surveillance CMR|Cold backup storage in Synology only — not in Unraid array|
|Seagate Barracuda 4TB|1|Desktop|Retired — not suitable for always-on NAS duty|