# Drive Inventory & Assignments

**Last Updated:** 2026-03-11

---

## Installed Drives — nas-prod-01

| Drive | Model | Size | Type | Role | Location |
|---|---|---|---|---|---|
| WD Red Pro | WD121KFBX-68EF5N0 | 12TB | CMR NAS | Parity 1 | nas-prod-01 array |
| WD Red Pro | WD121KFBX-68EF5N0 | 12TB | CMR NAS | Parity 2 | nas-prod-01 array |
| Seagate IronWolf | ST6000VN0033 | 6TB | CMR NAS | Disk 1 (data) | nas-prod-01 array |
| Seagate IronWolf | ST6000VN0033 | 6TB | CMR NAS | Disk 2 (data) | nas-prod-01 array |
| WD Red Plus | WD40EFRX | 4TB | CMR NAS | ZFS mirror disk 1 | nas-prod-01 zfs-mirror pool |
| WD Red Plus | WD40EFPX | 4TB | CMR NAS | ZFS mirror disk 2 | nas-prod-01 zfs-mirror pool |

---

## Uninstalled / Spare Drives

| Drive | Model | Size | Type | Status | Notes |
|---|---|---|---|---|---|
| WD Red Pro | WD121KFBX | 12TB | CMR NAS | In TrueNAS (v2) | Add to nas-prod-01 array as Disk 3 after TrueNAS migration completes |
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

| Drive | Known Issues | Last Checked |
|---|---|---|
| IronWolf ST6000VN0033 (Disk 2) | UDMA CRC Error Count = 2 (historical, Old Age attribute) | 2026-03-11 |

> CRC errors are cumulative and do not reset. Raw value of 2 is not concerning — likely caused by a cable/connection event in prior use. All Pre-fail attributes clean. Monitor on future SMART tests.

---

## Planned Expansion

- Add 3rd WD Red Pro 12TB as Disk 3 (data) after TrueNAS v2 migration completes in Phase 4
- Add 2x IronWolf 6TB as Disk 4 and Disk 5 if storage needs grow