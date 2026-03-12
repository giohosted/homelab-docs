# nas-prod-01 — Storage Architecture

**Host:** nas-prod-01  
**OS:** Unraid 7.2.4  
**Chassis:** Rosewill RSV-L4412U (4U, 12-bay)  
**Last Updated:** 2026-03-12

---

## Network Interfaces

| Interface | Type | IP | VLAN | Purpose |
|---|---|---|---|---|
| eth0 (onboard 2.5GbE) | br0 bridge | 192.168.10.10/24 | 10 — Management | Unraid web UI only |
| eth1 (X710 Port 2, 10GbE) | Direct | 192.168.30.16/24 | 30 — Services | NFS exports, Plex traffic |
| eth2 (X710 Port 1, 10GbE) | Unconfigured | 10.0.0.x/30 (Phase 3) | None | Point-to-point DAC to pve-prod-01 |

---

## Array — Parity Array (XFS)

| Role | Drive | Size |
|---|---|---|
| Parity 1 | WD Red Pro (WD121KFBX) | 12TB |
| Parity 2 | WD Red Pro (WD121KFBX) | 12TB |
| Disk 1 | Seagate IronWolf (ST6000VN0033) | 6TB |
| Disk 2 | Seagate IronWolf (ST6000VN0033) | 6TB |

- **Usable:** ~12TB
- **Filesystem:** XFS per disk (Unraid default)
- **Parity sync completed:** 2026-03-11, 19h 28m, 0 errors
- **Planned expansion:** 3rd WD Red Pro 12TB data drive to be added after TrueNAS migration completes

---

## Pool — ZFS Mirror (zfs-mirror)

| Role | Drive | Size |
|---|---|---|
| Mirror disk 1 | WD Red Plus (WD40EFRX) | 4TB |
| Mirror disk 2 | WD Red Plus (WD40EFPX) | 4TB |

- **Usable:** ~3.68TB
- **Filesystem:** ZFS mirror
- **Snapshots:** ZnapZend — daily kept 7 days, weekly kept 30 days on photos and backups datasets

---

## Shares

| Share | Pool | Path | Purpose |
|---|---|---|---|
| media | Parity Array | /mnt/user/media | All media — movies, TV, anime, books |
| downloads | Parity Array | /mnt/user/downloads | Active download staging |
| appdata | Parity Array | /mnt/user/appdata | Docker/app config (NAS perspective) |
| isos | Parity Array | /mnt/user/isos | Proxmox ISO images |
| photos | ZFS Mirror | /mnt/user/photos | Immich library |
| backups | ZFS Mirror | /mnt/user/backups | PBS backups, Docker appdata, Plex DB |

- **NFS:** Enabled on all shares, exported Public
- **Permissions:** UID/GID 2000:2000, chmod 775 on all shares
- **SMB:** Disabled on all shares (NFS only)

---

## Docker Containers

| Container | IP | Purpose |
|---|---|---|
| plex | 192.168.30.2 | Plex Media Server — bound to eth1 (VLAN 30) |

- iGPU passthrough: /dev/dri → Intel Raptor Lake-S UHD Graphics (QuickSync)
- Hardware transcoding: enabled in Plex transcoder settings
- Media path: /data → /mnt/user (full share access)
- Appdata: /mnt/user/appdata/plex

---

## UPS & Graceful Shutdown

* **UPS:** Tripp-Lite SMART1500LCDXL connected via USB-B cable to nas-prod-01
* **NUT plugin:** dmacias/Network UPS Tools, NUT 2.8.4
* **NUT mode:** Netserver — nas-prod-01 is the NUT server, Proxmox nodes connect as clients (Phase 3)
* **Driver:** usbhid-ups (auto-detected), port: auto
* **UPS confirmed:*** On Line, 100% battery, 34 min runtime at 15% load (225 VA), 1500VA nominal
* **Shutdown timer:** 6 minutes on battery before initiating shutdown sequence
* **Shutdown order:** Proxmox nodes receive NUT signal and shut down VMs/LXCs then hypervisor → NAS shuts down last
* **NUT clients:** pve-prod-01, pve-prod-02 — to be configured in Phase 3


---

## Plugins Installed

| Plugin | Purpose |
|---|---|
| NUT | UPS graceful shutdown |
| ZFS Master | ZFS pool management UI |
| ZnapZend | ZFS snapshot scheduling |
| Intel-GPU-TOP | iGPU monitoring |
| Dynamix System Temp | CPU/board temp monitoring |
| Unassigned Devices | External drive management |
| Fix Common Problems | Configuration health checks |