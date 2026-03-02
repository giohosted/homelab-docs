# Phase 0 — Procurement & Physical Build

**Status:** Procurement complete — physical build in progress **Started:** 2026-02-21 **Completed:** _(fill in when rack is built and all hardware POST-verified)_

---

## Objective

Acquire all hardware, build the rack, and validate the physical layer before any software is installed. UPS must be protecting the full stack before any drives are powered on.

---

## Procurement Log

### What Was Bought

|Item|Model|Source|Price|Date|
|---|---|---|---|---|
|Primary Compute|Minisforum MS-A2 (Ryzen 9 7945HX, 64GB DDR5)|Amazon|$559 + tax|2026-02-21|
|NAS Chassis|Rosewill RSV-L4412U (4U, 12-bay)|Amazon|$345|_(fill in)_|
|NAS Motherboard|ASUS TUF Gaming Z690-Plus WiFi D4|Reddit r/hardwareswap|~$100|_(fill in)_|
|10GbE NIC|Dell Intel X710-DA2 (dual-port SFP+)|eBay|~$25|_(fill in)_|
|UPS|Tripp-Lite SMART1500LCDXL 1500VA|_(fill in)_|~$145|_(fill in)_|
|Rack|StarTech 4POSTRACK18U (18U open-frame)|_(fill in — bundled with switch)_|Bundled|_(fill in)_|
|Switch|UniFi USW-Pro-Max-24|_(fill in)_|$560 (bundled with rack)|_(fill in)_|
|DAC Cable 0.5M|10Gtek SFP+ DAC|Amazon|~$10|_(fill in)_|
|DAC Cable 1M|Cable Matters SFP+ DAC|Amazon|~$15|_(fill in)_|
|DAC Cable 2M|Cable Matters SFP+ DAC|Amazon|~$17|_(fill in)_|
|Patch Panel|UniFi UP-PATCH-24 (keystone)|_(fill in)_|_(fill in)_|_(fill in)_|
|Keystone Couplers|Iwillink Cat6 shielded 10-pack|Amazon|~$15|_(fill in)_|
|Patch Cables|Monoprice SlimRun Cat6 0.5ft 10-pack|Amazon|_(fill in)_|_(fill in)_|
|Blank Keystone Inserts|_(fill in model)_|_(fill in)_|_(fill in)_|_(fill in)_|
|MS-A2 Shelf|2U custom bracket|Etsy|_(fill in)_|_(fill in)_|
|Optiplex Shelf|1U shelf mount|Etsy|_(fill in)_|_(fill in)_|
|Brush Panel|_(fill in model)_|_(fill in)_|_(fill in)_|_(fill in)_|

### What Was Already Owned (No Purchase)

|Item|Model|Notes|
|---|---|---|
|NAS CPU|Intel Core i5-13400|Kept from existing tower — i5-13600 sold|
|NAS RAM|32GB DDR4|Kept from existing tower — extra 2x 16GB sticks sold|
|NAS HBA|LSI SAS 9120-8i|Migrated from previous host|
|All HDD drives|WD Red Pro 12TB (x5), WD Red Plus 4TB (x5), IronWolf 6TB (x2), SkyHawk 6TB (x4)|Carried forward from v2|
|Secondary Compute|Dell Optiplex 3070 Micro|Free from work|
|Optiplex RAM|16GB DDR4 (2x 8GB Micron)|Free from work|
|Monitoring / QDevice|Raspberry Pi 4B (4GB)|Existing|
|Router / Firewall|UniFi UDM-SE|Carried forward from v2|

### What Was Sold

|Item|Reason|Estimated Return|
|---|---|---|
|Intel Core i5-13600|Unraid is IO-bound — i5-13400 is sufficient. Sold to offset MS-A2 cost.|~$150|
|2x 16GB DDR4 SO-DIMM|NAS only needs 32GB. No heavy VM workloads on Unraid.|~$50|
|MSI PRO B660M-A DDR4|Found in IT closet — spare board. Known Realtek NIC driver issues in Linux.|~$65|

---

## Procurement Decisions & Issues Encountered

### PCIe Slot Problem — Gigabyte B760M DS3H DDR4

The original plan was to reuse the Gigabyte B760M DS3H DDR4 motherboard already in the tower. During planning it became clear the board only has 1x usable PCIe slot — one x16 physical slot with two x1 slots that are too small for either the LSI HBA or the 10GbE NIC. The NAS needs both cards simultaneously, making this board incompatible. This was caught during planning before any hardware was purchased, which was fortunate.

A spare MSI PRO B660M-A DDR4 was also considered but ruled out due to known Realtek NIC driver issues in Linux environments. The ASUS TUF Gaming Z690-Plus WiFi D4 was sourced used from Reddit r/hardwareswap — it has a second x16 slot wired at x4 from the chipset which fits both the LSI HBA and the X710 NIC cleanly.

See `architecture/decisions-log.md` for full rationale.

### Scope Creep — Second Full Proxmox Node

During planning there was a temptation to build a second full-spec Proxmox node using spare parts (i5-13600, spare mobo, extra 32GB RAM) instead of using the Optiplex as pve-prod-02. This was rejected. The Optiplex's workload is intentionally light (PBS + AdGuard LXC), HA is not enabled in this design, and a second powerful node would mean idle resources and higher power draw for no benefit. The spare parts were sold instead.

### DAC Cable Lengths

Cable lengths were calculated based on the finalized rack layout:

- **0.5M** — UDM-SE to switch. Both front-mounted, 2U apart, short run front-to-front.
- **1M** — Switch to MS-A2. Front of switch to rear of MS-A2 shelf, routed through brush panel.
- **2M** — NAS to MS-A2. Rear-to-rear run accounting for RSV-L4412U sliding rail extension slack (~1.5M actual path + margin). Storage traffic kept off the LAN switch — dedicated point-to-point link.

---

## Physical Build Checklist

_Complete this section as hardware arrives and the rack is built._

### Hardware Arrival

- [ ] MS-A2 arrived and inspected
- [ ] Rosewill RSV-L4412U chassis arrived
- [ ] ASUS TUF Z690 motherboard arrived
- [ ] Dell X710-DA2 NIC arrived
- [ ] UPS arrived
- [ ] StarTech rack arrived
- [ ] USW-Pro-Max-24 arrived
- [ ] DAC cables arrived (0.5M, 1M, 2M)
- [ ] Patch panel arrived
- [ ] MS-A2 shelf mount arrived (Etsy)
- [ ] Optiplex shelf mount arrived (Etsy)
- [ ] Brush panel arrived

### NAS Assembly

- [ ] i5-13400 installed in TUF Z690
- [ ] 32GB DDR4 installed
- [ ] LSI HBA seated in PCIe x16 slot
- [ ] X710-DA2 NIC seated in PCIe x4 slot (second x16 physical slot)
- [ ] All drives installed in RSV-L4412U bays
- [ ] USB flash drive prepared with Unraid 7.2.3
- [ ] Board + components installed in RSV-L4412U chassis
- [ ] POST verified — no errors

### MS-A2

- [ ] Both NVMe drives installed (Samsung + Sabrent — record exact models and serials)
- [ ] POST verified

### Optiplex

- [ ] RAM confirmed (16GB, 2x 8GB Micron)
- [ ] POST verified

### Rack Build

- [ ] UPS positioned at U18, powered on, battery verified
- [ ] UDM-SE mounted at U1
- [ ] Patch panel mounted at U2
- [ ] USW-Pro-Max-24 mounted at U3
- [ ] Brush panel mounted at U4
- [ ] MS-A2 shelf mounted at U5-U6, MS-A2 seated
- [ ] Optiplex shelf mounted at U7, Optiplex + Pi seated
- [ ] RSV-L4412U rails installed at U8-U11, chassis mounted
- [ ] All devices connected to UPS battery-backed outlets
- [ ] UPS USB cable connected to NAS (for NUT)

### Cabling

- [ ] 0.5M DAC: UDM-SE SFP+ → Switch SFP+
- [ ] 1M DAC: Switch SFP+ → MS-A2 SFP+ (Port 1)
- [ ] 2M DAC: NAS X710 SFP+ (Port 1) → MS-A2 SFP+ (Port 2)
- [ ] RJ45: MS-A2 2.5GbE → Switch
- [ ] RJ45: NAS 2.5GbE (onboard TUF Z690) → Switch
- [ ] RJ45: Optiplex 1GbE → Switch
- [ ] RJ45: Pi 1GbE → Switch
- [ ] RJ45: UDM-SE LAN → Switch
- [ ] Patch panel wall runs terminated and labeled (7 runs)
- [ ] Patch cables (0.5ft) from patch panel → switch
- [ ] All cables labeled

### Validation

- [ ] All devices power on and POST cleanly
- [ ] UPS protecting full stack — verify in UPS display
- [ ] All DAC links showing 10GbE (verify in UniFi and on switch port LEDs)
- [ ] All RJ45 links active on switch

---

## Exit Criteria

- [ ] All hardware powered on and POST-verified
- [ ] UPS protecting full stack
- [ ] All network links active
- [ ] Cables labeled
- [ ] This document updated with any build notes or issues encountered

**→ Phase 1 (Network Foundation) begins after all exit criteria are met.**

---

## Build Notes

_(Fill in as you go — issues encountered, deviations from plan, anything worth remembering)_

---

_Phase 0 — Procurement & Physical Build | Homelab v3.0_