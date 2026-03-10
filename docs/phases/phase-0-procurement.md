# Phase 0 — Procurement & Physical Build

**Status:** Completed **Started:** 2026-02-21 **Completed:** 2026-03-10

---

## Objective

Acquire all hardware, build the rack, and validate the physical layer before any software is installed. UPS must be protecting the full stack before any drives are powered on.

---

## Procurement Log

### What Was Bought

|Item|Model|Source|Price|Date|
|---|---|---|---|---|
|Primary Compute|Minisforum MS-A2 (Ryzen 9 7945HX)|Amazon|$575|2026-02-21|
|RAM for MS-A2|SK Hynix 32GB DDR5 SO-DIMM (1x 32GB)|FB Marketplace|$140|2026-02-22|
|USB Drive for Unraid|SanDisk 16GB|Amazon|$15|2026-02-22|
|Rack + Switch (bundle)|StarTech 4POSTRACK18U + UniFi USW-Pro-Max-24|Reddit r/homelabsales|$560|2026-02-25|
|NAS Chassis|Rosewill RSV-L4412U (4U, 12-bay)|eBay|$345|2026-02-26|
|NAS Motherboard|ASUS TUF Gaming Z690-Plus WiFi D4|Reddit r/hardwareswap|$100|2026-02-27|
|10GbE NIC|Dell Intel X710-DA2 (dual-port SFP+)|eBay|$28|2026-02-27|
|Keystone Couplers|Iwillink Cat6 shielded 10-pack|Amazon|$16|2026-02-27|
|Patch Cables|Monoprice SlimRun Cat6 0.5ft 10-pack|Amazon|$17|2026-02-27|
|Blank Keystone Inserts|VCE UL|Amazon|$9|2026-02-27|
|Patch Panel|UniFi UP-PATCH-24 (keystone)|Ubiquiti|$48|2026-02-28|
|MS-A2 Rack Mount|2U custom bracket|Etsy|$69|2026-02-28|
|Rack Rails|iStarUSA TC-RAIL-26|eBay|$74|2026-02-28|
|UPS + new batteries|Tripp-Lite SMART1500LCDXL 1500VA|FB Marketplace + Amazon|$149|2026-03-01|
|DAC Cable 0.5M|10Gtek SFP+ DAC|Amazon|$11|2026-03-01|
|DAC Cable 1M|Cable Matters SFP+ DAC|Amazon|$16|2026-03-01|
|DAC Cable 2M|Cable Matters SFP+ DAC|Amazon|$19|2026-03-01|
|Optiplex Shelf|1U shelf mount|Amazon|$22|2026-03-01|
|Brush Panel|Suprwin|Amazon|$22|2026-03-01|

### What Was Already Owned (No Purchase)

|Item|Model|Notes|
|---|---|---|
|NAS CPU|Intel Core i5-13400|Kept from existing tower — i5-13600 sold|
|NAS RAM|32GB DDR4|Kept from existing tower — extra 2x 16GB sold|
|NAS HBA|LSI SAS 9120-8i|Migrated from previous host|
|All HDD drives|WD Red Pro 12TB (x5), WD Red Plus 4TB (x5), IronWolf 6TB (x2), SkyHawk 6TB (x4), Barracuda 4TB (x1)|Carried forward from v2|
|Secondary Compute|Dell Optiplex 3070 Micro|Free from work|
|Optiplex RAM|16GB DDR4 (2x 8GB Micron)|Free from work|
|Monitoring / QDevice|Raspberry Pi 4B (4GB)|Existing|
|Router / Firewall|UniFi UDM-SE|Carried forward from v2|

### What Was Sold

|Item|Reason|Amount Received|
|---|---|---|
|Intel Core i5-13600 + MSI PRO B660M-A DDR4|Bundled sale. i5-13400 sufficient for Unraid (IO-bound). MSI board had Realtek NIC driver issues.|$180|
|Gigabyte B760M DS3H DDR4|Original NAS mobo — only 1x usable PCIe slot. Couldn't fit LSI HBA + X710 simultaneously.|$90|
|2x 16GB DDR4 SO-DIMM|NAS only needs 32GB — no heavy VM workloads on Unraid.|~$50|

---

## Procurement Decisions & Issues Encountered

### PCIe Slot Problem — Gigabyte B760M DS3H DDR4

The original plan was to reuse the Gigabyte B760M DS3H DDR4 motherboard already in the tower. During planning it became clear the board only has 1x usable PCIe slot — one x16 physical slot with two x1 slots too small for either the LSI HBA or the 10GbE NIC. The NAS needs both cards simultaneously, making this board incompatible. A spare MSI PRO B660M-A DDR4 was also considered but ruled out due to known Realtek NIC driver issues in Linux environments. The ASUS TUF Gaming Z690-Plus WiFi D4 was sourced used from Reddit r/hardwareswap — it has a second x16 slot wired at x4 from the chipset which fits both the LSI HBA and the X710 NIC cleanly.

See `architecture/decisions-log.md` for full rationale.

### DAC Cable Route Change — 1M Cable Repurposed

The 1M DAC cable was originally purchased for MS-A2 SFP+ → switch SFP+. During architecture finalization, the NAS X710 Port 2 was assigned to the switch SFP+ port for the VLAN 30 data interface (NFS + Plex traffic), giving 10GbE on the busiest path in the lab. The MS-A2 was switched to its onboard 2.5GbE RJ45 for VM/LXC LAN traffic instead — no performance impact since that workload never approaches 2.5GbE saturation. The 1M DAC was repurposed for NAS X710 Port 2 → switch, eliminating the need to buy an additional cable.

Final DAC assignments:

- **0.5M** — UDM-SE SFP+ → Switch SFP+ (WAN uplink)
- **1M** — NAS X710 Port 2 SFP+ → Switch SFP+ (VLAN 30 data/NFS interface)
- **2M** — NAS X710 Port 1 SFP+ → MS-A2 SFP+ Port 1 (dedicated point-to-point storage link)

### Scope Creep — Second Full Proxmox Node

During planning there was a temptation to build a second full-spec Proxmox node using spare parts (i5-13600, spare mobo, extra 32GB RAM) instead of using the Optiplex as pve-prod-02. This was rejected. The Optiplex's workload is intentionally light (PBS + AdGuard LXC), HA is not enabled in this design, and a second powerful node would mean idle resources and higher power draw for no benefit. The spare parts were sold instead.

### UPS — Tripp-Lite SMART1500LCDXL Kept

The existing Tripp-Lite SMART1500LCDXL was evaluated against replacing it with an APC 1500VA. Decision: keep the Tripp-Lite with fresh batteries. 900W capacity covers the full stack comfortably (~270–300W typical load). Simulated sine wave output is fine for modern PSUs with active PFC. No SNMP card — NUT via USB is the correct approach for a 3-node homelab. Rack ears confirmed compatible.

---

## Physical Build Checklist

### Hardware Arrival

- [x] MS-A2 arrived and inspected
- [x] Rosewill RSV-L4412U chassis arrived
- [x] ASUS TUF Z690 motherboard arrived
- [x] Dell X710-DA2 NIC arrived
- [x] UPS arrived
- [x] StarTech rack arrived
- [x] USW-Pro-Max-24 arrived
- [x] DAC cables arrived (0.5M, 1M, 2M)
- [x] Patch panel arrived
- [x] MS-A2 shelf mount arrived (Etsy)
- [x] Optiplex shelf mount arrived
- [x] Brush panel arrived

### NAS Assembly

- [x] i5-13400 installed in TUF Z690
- [x] 32GB DDR4 installed
- [x] LSI HBA seated in PCIe x16 slot
- [x] X710-DA2 NIC seated in PCIe x4 slot (second x16 physical slot)
- [x] All drives installed in RSV-L4412U bays
- [x] USB flash drive prepared with Unraid 7.2.3
- [x] Board + components installed in RSV-L4412U chassis
- [x] POST verified — no errors

### MS-A2

- [x] Both NVMe drives installed (Samsung 980 1TB S/N: S64ANS0W120169T + Sabrent Rocket 1TB)
- [x] 32GB DDR5 SO-DIMM installed (1x 32GB SK Hynix — second slot reserved for future upgrade)
- [x] POST verified

### Optiplex

- [x] RAM confirmed (16GB, 2x 8GB Micron)
- [x] POST verified

### Rack Build

- [x] UPS positioned at U18, powered on, battery verified
- [x] UDM-SE mounted at U1
- [x] Patch panel mounted at U2
- [x] USW-Pro-Max-24 mounted at U3
- [x] Brush panel mounted at U4
- [x] MS-A2 shelf mounted at U5-U6, MS-A2 seated
- [x] Optiplex shelf mounted at U7, Optiplex + Pi seated
- [x] RSV-L4412U rails installed at U8-U11, chassis mounted
- [x] All devices connected to UPS battery-backed outlets
- [x] UPS USB cable connected to NAS (for NUT)

### Cabling

- [x] 0.5M DAC: UDM-SE SFP+ → Switch SFP+ (WAN uplink)
- [x] 1M DAC: NAS X710 Port 2 SFP+ → Switch SFP+ (VLAN 30 data interface)
- [x] 2M DAC: NAS X710 Port 1 SFP+ → MS-A2 SFP+ Port 1 (dedicated storage link)
- [x] RJ45: MS-A2 2.5GbE → Switch (VLAN 10 management + VM/LXC LAN)
- [x] RJ45: NAS 2.5GbE (onboard TUF Z690) → Switch (VLAN 10 management)
- [x] RJ45: Optiplex 1GbE → Switch
- [x] RJ45: Pi 1GbE → Switch
- [x] RJ45: UDM-SE LAN → Switch
- [x] Patch panel wall runs terminated and labeled (7 runs)
- [x] Patch cables (0.5ft) from patch panel → switch
- [x] All cables labeled

### Validation

- [x] All devices power on and POST cleanly
- [x] UPS protecting full stack — verify in UPS display
- [x] All DAC links showing at link speed (verify in UniFi and on switch port LEDs)
- [x] All RJ45 links active on switch

---

## Exit Criteria

- [x] All hardware powered on and POST-verified
- [x] UPS protecting full stack
- [x] All network links active
- [x] Cables labeled
- [x] This document updated with any build notes or issues encountered

**→ Phase 1 (Network Foundation) begins after all exit criteria are met.**

---

## Build Notes

_(Fill in as you go — issues encountered, deviations from plan, anything worth remembering)_

- Molex power for RSV-L4412U backplanes: original PSU cable (3x molex) + SATA-to-molex adapter 3-pack = 6 molex total covering all 3 backplane boards
- Stock 80mm rear fans replaced with 2x Arctic P8 Max PWM fans connected to 4-pin headers on TUF Z690. 3x stock 120mm chassis fans kept.
- LSI HBA powered via PCIe slot only — no molex required.

---

_Phase 0 — Procurement & Physical Build | Homelab v3.0_