# Backup Schedule

**Last Updated:** 2026-03-18

Reference document for all backup jobs running across the homelab. Lists what runs, when, where it goes, and how to verify it worked.

---

## Schedule Overview

| Time | Job | Source | Destination | Host |
|------|-----|--------|-------------|------|
| 02:00 daily | PBS — VM & CT backup | All VMs and CTs | nas-prod-01 `/mnt/user/backups/pbs` | pbs-prod-01 |
| 03:00 daily | rsync — stacks + appdata | `/opt/stacks` + `/opt/appdata` | nas-prod-01 `/mnt/user/backups/` | docker-prod-01 |
| 03:00 daily | rsync — stacks + appdata | `/opt/stacks` + `/opt/appdata` | nas-prod-01 `/mnt/user/backups/` | auth-prod-01 |
| 03:00 daily | rsync — stacks + appdata | `/opt/stacks` + `/opt/appdata` | nas-prod-01 `/mnt/user/backups/` | immich-prod-01 |
| 04:00 Saturday | PBS — Garbage Collection | nas-backups datastore | — | pbs-prod-01 |
| 05:00 Saturday | PBS — Verify | nas-backups datastore | — | pbs-prod-01 |
| TBD | Synology ABB | nas-prod-01 `backups` share | Synology NAS | Synology |

---

## Ordering Rationale

Jobs are sequenced intentionally:
1. **02:00 — PBS** runs first while the homelab is at its quietest
2. **03:00 — rsync** runs after PBS completes, no overlap
3. **04:00 Saturday — GC** runs after the week's backups and prunes are done
4. **05:00 Saturday — Verify** runs after GC completes
5. **Synology ABB** — will be scheduled after 03:00 once configured, so it pulls an already-complete rsync mirror

---

## PBS Backup

**Managed by:** Proxmox Backup Server 4.1.4 on pbs-prod-01 (192.168.30.12)  
**UI:** `https://192.168.30.12:8007`  
**Datastore:** `nas-backups` → NFS mount at `/mnt/backups/pbs` → NAS path `/mnt/user/backups/pbs`  
**Backup job:** Datacenter → Backup on Proxmox UI (port 8006)

**Retention:**
- Keep Last: 3
- Keep Daily: 7
- Keep Weekly: 4
- Keep Monthly: 3

**Verification:** Healthchecks.io not yet configured for PBS — TBD.

---

## rsync — Stacks & Appdata

**Script location:** `/usr/local/bin/backup-appdata.sh` on each host  
**Log location:** `/var/log/backup-appdata.log` on each host  
**SSH key:** `/root/.ssh/backup_rsa` on each host  
**Scheduled via:** root crontab — edit with `sudo crontab -e`, view with `sudo crontab -l`  
**Run manually:** `sudo /usr/local/bin/backup-appdata.sh`  
**Transport:** SSH rsync to nas-prod-01 at `192.168.30.16`

**What is backed up:**
- `/opt/stacks` — all compose files and `.env` files (`.env` files are gitignored, this is the only backup)
- `/opt/appdata` — all container config and data directories

**Behavior:** Rolling mirror with `--delete`. Destination always matches source. No versioning — covered by Synology ABB (Wave 8).

**Healthchecks.io:**
| Host | Check URL |
|------|-----------|
| docker-prod-01 | `https://hc-ping.com/132bdc56-2f0e-45b8-85a8-c07dc1c049ab` |
| auth-prod-01 | `https://hc-ping.com/7c625181-5347-4ce5-ab5a-385b72201d91` |
| immich-prod-01 | `https://hc-ping.com/1329d9f0-9416-4d81-935c-23ce2969c1a6` |

---

## NAS Backup Share Structure
```
/mnt/user/backups/
├── pbs/                    ← PBS datastore (uid 34 owned — do not touch)
├── appdata/                ← rsync appdata backups
│   ├── docker-prod-01/
│   ├── auth-prod-01/
│   └── immich-prod-01/
└── stacks/                 ← rsync stacks backups
    ├── docker-prod-01/
    ├── auth-prod-01/
    └── immich-prod-01/
```

> **Warning:** Do not place files owned by root directly in the `backups` share root. PBS GC scans the entire datastore path and will abort with a permission error if it encounters files it cannot read.

---

## Pending

- PBS Healthchecks.io heartbeat — not yet configured
- Plex database backup — Plex runs directly on Unraid, needs dedicated solution
- Synology ABB — Wave 8, versioned backup of entire `backups` share