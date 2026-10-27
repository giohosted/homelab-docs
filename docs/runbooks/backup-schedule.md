# Backup Schedule

**Last Updated:** 2026-10-26

Reference document for all backup jobs running across the homelab. Lists what runs, when, where it goes, and how to verify it worked.

---

## Schedule Overview

|Time|Job|Source|Destination|Host|
|---|---|---|---|---|
|02:00 daily|PBS — VM & CT backup|All VMs and CTs|nas-prod-01 `/mnt/user/backups/pbs`|pbs-prod-01|
|03:00 daily|rsync — stacks + appdata|`/opt/stacks` + `/opt/appdata`|nas-prod-01 `/mnt/user/backups/`|docker-prod-01|
|03:00 daily|rsync — stacks + appdata|`/opt/stacks` + `/opt/appdata`|nas-prod-01 `/mnt/user/backups/`|auth-prod-01|
|03:00 daily|rsync — stacks + appdata|`/opt/stacks` + `/opt/appdata`|nas-prod-01 `/mnt/user/backups/`|immich-prod-01|
|Every 3 days, ~04:03|Plex built-in DB backup|Plex library and blobs databases|`/mnt/user/appdata/plex`|nas-prod-01|
|04:00 Saturday|PBS — Garbage Collection|nas-backups datastore|—|pbs-prod-01|
|05:00 Saturday|PBS — Verify|nas-backups datastore|—|pbs-prod-01|
|TBD — Wave 8|Dedicated Plex full-appdata backup|`/mnt/user/appdata/plex`|NAS `backups` share — final path TBD|nas-prod-01|
|TBD — Wave 8|Synology ABB|nas-prod-01 `backups` share|Synology NAS|Synology|

The Plex built-in database backup schedule was verified from backup file timestamps on 2026-10-26. The dedicated Plex full-appdata backup and Synology ABB are planned jobs, not confirmed operational backups.

---

## Ordering Rationale

Jobs are sequenced intentionally:

1. **02:00 — PBS** runs first while the homelab is at its quietest
    
2. **03:00 — rsync** runs after PBS completes, no overlap
    
3. **04:00–06:00 — Plex maintenance** includes the built-in database backup every three days
    
4. **04:00 Saturday — GC** runs after the week's backups and prunes are done
    
5. **05:00 Saturday — Verify** runs after GC completes
    
6. **Dedicated Plex backup (Wave 8)** — schedule TBD; must account for Plex's maintenance window, service interruption, and other NAS backup jobs
    
7. **Synology ABB (Wave 8)** — will be scheduled after source backups complete so it captures consistent, completed backups
    

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
    

**Behavior:** Rolling mirror with `--delete`. Destination always matches source. No versioning — planned to be covered by Synology ABB (Wave 8).

**Healthchecks.io:**

|Host|Check URL|
|---|---|
|docker-prod-01|`https://hc-ping.com/132bdc56-2f0e-45b8-85a8-c07dc1c049ab`|
|auth-prod-01|`https://hc-ping.com/7c625181-5347-4ce5-ab5a-385b72201d91`|
|immich-prod-01|`https://hc-ping.com/1329d9f0-9416-4d81-935c-23ce2969c1a6`|

---

## Plex — Built-in Database Backup ✅

**Host:** nas-prod-01  
**Managed by:** Plex Media Server → Settings → Scheduled Tasks  
**Schedule:** Every three days during the 04:00–06:00 maintenance window  
**Status:** Verified operational 2026-10-26

**Source and backup location:**

`/mnt/user/appdata/plex/Library/Application Support/Plex Media Server/Plug-in Support/Databases/`

**Backed up:**

- Plex library database
    
- Plex library blobs database
    

**Verified backup dates:**

- 2026-10-16
    
- 2026-10-19
    
- 2026-10-22
    
- 2026-10-25
    

Both databases had dated backup copies for all four dates, created around 04:03–04:04.

**Verification command:**

```
find "/mnt/user/appdata/plex/Library/Application Support/Plex Media Server/Plug-in Support/Databases" \
  -maxdepth 1 -type f \
  -printf '%TY-%Tm-%Td %TH:%TM | %s bytes | %f\n' \
  | sort -r | head -30
```

Look for recent date-suffixed copies of:

- `com.plexapp.plugins.library.db`
    
- `com.plexapp.plugins.library.blobs.db`
    

**Limitations:**

- Backups are stored within the same Plex appdata directory as the live databases.
    
- They do not provide an independent copy of all Plex application data.
    
- Backup file creation was verified; integrity testing and restore testing remain unverified.
    
- Retention settings were not independently verified.
    

---

## Plex — Dedicated Full-Appdata Backup ⬜ Pending (Wave 8)

A dedicated custom backup script for Plex is planned under Phase 5 Wave 8.

**Planned source:** `/mnt/user/appdata/plex`  
**Planned destination:** NAS `backups` share; final subdirectory TBD  
**Backup mechanism:** Custom script on Unraid  
**Schedule:** TBD  
**Retention:** TBD  
**Status:** Not implemented as of 2026-10-26

The original Plex service documentation described a stop–backup–restart script writing to `/mnt/user/backups/plex/db/`.

Investigation on 2026-10-26 found that:

- The documented destination directory did not exist.
    
- No Plex-specific root cron entry was found.
    
- The Unraid User Scripts directory was absent.
    
- No dedicated Plex backup script was located in the common configuration and executable directories searched.
    

The existing rsync jobs cover Docker hosts other than nas-prod-01 and should not be assumed to protect Plex appdata.

**Implementation requirements to finalize in Wave 8:**

- Safe handling of the running Plex container and its SQLite databases
    
- Backup destination and archive format
    
- Automated scheduling
    
- Retention and historical recovery points
    
- Logging, error handling, and notification or monitoring
    
- Backup integrity and restore testing
    
- Coordination with ZFS snapshots and planned Synology ABB
    

See `services/media/plex.md` and `phases/phase-5-hardening.md`.

---

## NAS Backup Share Structure

Documented structure before the dedicated Plex backup is implemented:

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

> **Warning:** Do not place files owned by root directly in the `backups` share root. PBS GC scans the entire datastore path and will abort with a permission error if it encounters files it cannot read. The PBS datastore must remain scoped to its dedicated `pbs/` subdirectory.

---

## Pending

- PBS Healthchecks.io heartbeat — not yet configured
    
- Dedicated Plex full-appdata backup — Phase 5 Wave 8; custom script selected, implementation pending
    
- Plex backup retention, integrity checks, and restore procedure — to be finalized and verified during Wave 8
    
- Synology ABB — Wave 8, versioned backup of entire `backups` share