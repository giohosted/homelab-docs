# Drive Inventory & Assignments

**Last Updated:** 2026-10-31

---

## Installed Drives — nas-prod-01

**Verified October 31, 2026** using the Unraid array integration and physical drive SMART serials. The parity-array recovery is complete; all five array drives are `DISK_OK`.

| Drive | Model | Size | Type | Role | Location | Serial |
|---|---|---|---|---|---|---|
| WD Red Pro | WD121KFBX-68EF5N0 | 12TB | CMR NAS | Parity 1 | nas-prod-01 array | `AAHKXP9H` |
| WD Red Pro | WD121KFBX-68EF5N0 | 12TB | CMR NAS | Parity 2 | nas-prod-01 array | `2AH2JSWY` |
| Seagate IronWolf | ST6000VN0033 | 6TB | CMR NAS | Disk 1 (data) | nas-prod-01 array | `ZAD97T6M` |
| Seagate IronWolf | ST6000VN0033 | 6TB | CMR NAS | Disk 2 (data) | nas-prod-01 array | `ZAD97RT0` |
| WD Red Pro | WD121KFBX-68EF5N0 | 12TB | CMR NAS | Disk 3 (data) | nas-prod-01 array | `5PK7HDXF` |
| WD Red Plus | WD40EFRX-68N32N0 | 4TB | CMR NAS | ZFS mirror disk 1 | nas-prod-01 zfs-mirror pool | `WD-WCC7K1XU8CVJ` |
| WD Red Plus | WD40EFPX-68C6CN0 | 4TB | CMR NAS | ZFS mirror disk 2 | nas-prod-01 zfs-mirror pool | `WD-WX72D43CS001` |

The former 14 TB Seagate Parity 1 (`ST14000NT001`) was removed from that array assignment on October 30, 2026; its current storage location and independent condition were not reverified.

---

## Uninstalled / Spare Drives

| Drive | Model | Size | Type | Status | Notes |
|---|---|---|---|---|---|
| WD Red Pro | WD121KFBX | 12TB | CMR NAS | Historical TrueNAS (v2) entry — not reverified | Superseded March 2026 plan to add Disk 3; `5PK7HDXF` is verified installed as Disk 3. Do not count this row as a confirmed available spare without verifying serial/identity. |
| WD Red Pro | WD121KFBX | 12TB | CMR NAS | Cold spare | Reserved in case of drive failure — do not use unless needed |
| WD Red Pro | WD121KFBX | 12TB | CMR NAS | For sale | Selling — not needed |
| WD Red Plus | WD40EFPX | 4TB | CMR NAS | Cold spare | Reserved for ZFS mirror pool replacement |
| Seagate IronWolf | ST6000VN0033 | 6TB | CMR NAS | Cold spare | Future array expansion |
| Seagate IronWolf | ST6000VN0033 | 6TB | CMR NAS | Cold spare | Future array expansion |
| Seagate SkyHawk | ST6000VX0023 | 6TB | Surveillance | In Synology | Cold backup storage only — not suitable for NAS array |
| Seagate SkyHawk | ST6000VX0023 | 6TB | Surveillance | In Synology | Cold backup storage only — not suitable for NAS array |
| Seagate SkyHawk | ST6000VX0023 | 6TB | Surveillance | In Synology | Cold backup storage only — not suitable for NAS array |
| Seagate SkyHawk | ST6000VX0023 | 6TB | Surveillance | In Synology | Cold backup storage only — not suitable for NAS array |
| Seagate Barracuda | ST4000DM004 | 4TB | Desktop | Retired | Not suitable for always-on NAS duty |

---

## SMART Notes

| Drive | Known Issues / Verification | Last Checked |
|---|---|---|
| WD Red Pro `AAHKXP9H` (Parity 1) | UDMA CRC = **454**, unchanged after cable replacement; fresh short self-test passed; 0 reallocated/pending/offline-uncorrectable sectors | 2026-10-31 |
| WD Red Pro `5PK7HDXF` (Disk 3) | UDMA CRC = **29**, unchanged after cable replacement; fresh short self-test passed; 0 reallocated/pending/offline-uncorrectable sectors | 2026-10-31 |
| IronWolf ST6000VN0033 (Disk 2) | UDMA CRC Error Count = 2 (historical, Old Age attribute; not freshly rechecked here) | 2026-03-11 |

> CRC errors are cumulative and do not reset after cabling repairs. Historical interface errors on Parity 1 and Disk 3, which shared the replaced breakout cable, support the **probable** cable/connection root cause. Monitor for any **increase** rather than expecting past counters to clear. No media-sector warnings were observed in the October 31 SMART reports for those two drives. See `nas-prod-01.md` for the resolved incident and verification boundaries.

---

## Planned Expansion

- Disk 3 (WD Red Pro 12TB, `5PK7HDXF`) is already installed and operational; the former Disk 3 addition plan is complete/superseded.
- Add 2x IronWolf 6TB as Disk 4 and Disk 5 if storage needs grow (historical plan; spare-drive availability has not been reverified).