# Phase 2 — NAS Build & Storage Commissioning

**Status:** Completed
**Started:** 2026-03-10  
**Completed:** 2026-03-12  
**Hardware:** nas-prod-01 (Rosewill RSV-L4412U, i5-13400, 32GB DDR4, ASUS TUF Z690-Plus WiFi D4)

---

## Completed

- Installed Unraid 7.2.4 (USB flash boot, SanDisk Ultra Fit)
- Configured BIOS: disabled Secure Boot, enabled iGPU, enabled Above 4G Decoding, disabled CSM
- Configured network interfaces:
  - eth0 (onboard 2.5GbE) → br0 → 192.168.10.10/24 (VLAN 10, management)
  - eth1 (X710 Port 2, 10GbE) → 192.168.30.16/24 (VLAN 30, data/NFS)
  - eth2 (X710 Port 1, 10GbE) → unconfigured, reserved for DAC link to pve-prod-01 (Phase 3)
- Built parity array: 2x WD Red Pro 12TB parity + 2x Seagate IronWolf 6TB data. Parity sync completed 2026-03-11, 19h 28m, 0 errors.
- Built ZFS mirror pool (zfs-mirror): 2x WD Red Plus 4TB. ~3.68TB usable.
- Created shares:
  - media → parity array
  - downloads → parity array
  - appdata → parity array
  - isos → parity array
  - photos → zfs-mirror
  - backups → zfs-mirror
- Enabled NFS on all shares, exported public (access controlled at firewall layer)
- Set UID/GID 2000:2000 ownership and 775 permissions on all shares
- Configured ZFS snapshot schedules via ZnapZend: daily kept 7 days, weekly kept 30 days on zfs-mirror/photos and zfs-mirror/backups
- Enabled Docker, configured Plex container on eth1 (192.168.30.2), iGPU /dev/dri passthrough confirmed, Intel Raptor Lake-S UHD Graphics (QuickSync) active in Plex transcoder settings
- Configured Discord webhook notifications
- Installed plugins: ZFS Master, ZnapZend, NUT, Intel-GPU-TOP, Dynamix System Temp, Unassigned Devices, Fix Common Problems
- Configured NUT (Network UPS Tools) — Netserver mode, usbhid-ups driver, USB-B cable connected to Tripp-Lite SMART1500LCDXL. UPS confirmed: On Line, 100% battery, 34 min runtime at 15% load (225 VA). Shutdown timer: 6 minutes on battery before initiating shutdown sequence. NAS shuts down last by design — Proxmox nodes are NUT clients and shut down first when they receive the signal.

---

## Decisions Made

- Started with 2x 12TB parity + 2x 6TB IronWolf data instead of full 5x 12TB array — one 12TB locked in TrueNAS pending migration, one kept as cold spare, one to sell
- No cache pool at launch — confirmed per roadmap, no workload justifies it
- ZnapZend chosen over cron jobs for ZFS snapshots — cleaner, purpose-built tool
- Plex bound to eth1 (192.168.30.16 network, container IP 192.168.30.2) not eth0 — services belong on VLAN 30, not management VLAN 10
- linxserver Plex template used (ich777 template not available)
- PUID/PGID set to 2000:2000 on Plex container, consistent with all service containers
- NUT mode set to Netserver (not Standalone) — Proxmox nodes will connect as NUT clients in Phase 3. Netserver required for multi-host shutdown coordination.
- Shutdown timer set to 6 minutes on battery — provides blip protection and enough time for Proxmox nodes to gracefully shut down VMs before NAS powers off.