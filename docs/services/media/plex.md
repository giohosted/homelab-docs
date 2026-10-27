# Plex Media Server

**Host:** nas-prod-01  
**Container name:** plex  
**Status:** Running  
**Last Updated:** 2026-10-26

---

## Deployment

- **Image:** `lscr.io/linuxserver/plex` (linuxserver.io)
    
- **Managed via:** Unraid Docker UI — not part of any compose stack
    
- **Network:** custom `eth1` (X710 Port 2, VLAN 30 data interface)
    
- **Container IP:** 192.168.30.2
    
- **Web UI:** `http://192.168.30.2:32400/web`
    

Plex runs directly on Unraid, not on docker-prod-01. This keeps media access local — no NFS hop for transcoding. QuickSync iGPU is on the same host as the media files.

---

## Container Variables

|Variable|Value|
|---|---|
|`PUID`|2000|
|`PGID`|2000|
|`VERSION`|docker|

---

## Paths

|Container Path|Host Path|Purpose|
|---|---|---|
|`/data`|`/mnt/user/data`|Full data share access — media and downloads|
|`/config`|`/mnt/user/appdata/plex`|Plex database and config|

---

## Hardware Transcoding

- **iGPU:** Intel UHD 730 (i5-13400, Intel Raptor Lake-S)
    
- **Method:** QuickSync
    
- **Device passthrough:** `/dev/dri` mapped into container
    
- **Plex transcoder settings:** Hardware acceleration enabled, hardware-accelerated encoding enabled, device set to Intel Raptor Lake-S UHD Graphics
    
- **HEVC encoding:** HEVC sources only
    

---

## Libraries

|Library|Path|
|---|---|
|Movies (1080p)|`/data/media/movies/1080p`|
|Movies (4K)|`/data/media/movies/4k`|
|TV|`/data/media/tv`|
|Anime|`/data/media/anime`|
|Audiobooks|`/data/media/books/audiobooks`|

Libraries were configured in Phase 4 after media migration from TrueNAS completed. DB restored from `/mnt/user/backups/plex/db/plex-db-2026-03-10.tar.gz`.

---

## Scheduled Tasks

Configured in Settings → Scheduled Tasks.

|Setting|Value|Notes|
|---|---|---|
|Tasks start time|4:00 AM|After rsync (03:00) and PBS (02:00) windows|
|Tasks stop time|6:00 AM||
|Backup database every three days|✅ Enabled|Verified operational 2026-10-26|
|Optimize database every week|✅ Enabled||
|Remove old bundles every week|✅ Enabled||
|Remove old cache files every week|✅ Enabled||
|Upgrade media analysis during maintenance|✅ Enabled||
|Perform extensive media analysis during maintenance|✅ Enabled||
|Update all libraries during maintenance|❌ Disabled|ARR stack triggers Plex scans via API — redundant|
|Refresh local metadata every three days|❌ Disabled|Unnecessary churn|
|Refresh library metadata periodically|❌ Disabled|Unnecessary churn|

Plex's built-in database backup runs every three days during the maintenance window. Backups are stored in the default database directory under `/mnt/user/appdata/plex`.

This mechanism is separate from the planned dedicated full-appdata backup in Phase 5 Wave 8.

---

## Transcoder Settings

|Setting|Value|Notes|
|---|---|---|
|Transcoder quality|Automatic||
|Transcoder temp directory|Default|RAM transcoding not configured — direct play is the norm, transcoding is rare|
|Hardware-accelerated transcoding|Enabled|QuickSync via `/dev/dri`|

---

## Thumbnail Settings

|Setting|Value|Notes|
|---|---|---|
|Generate video preview thumbnails|Never|Storage and CPU intensive — not worth it for a personal library|
|Generate chapter image thumbnails|As scheduled task + when media added|Small and useful for movies|

---

## External Access

- Port 32400 forwarded on UDM-SE → `192.168.30.2:32400`
    
- Plex handles its own TLS and relay
    
- Not behind Traefik — direct port forward is intentional
    
- Not exposed via Cloudflare Tunnel — Cloudflare ToS prohibits video streaming through their tunnel
    

---

## Backup

### Built-in Plex Database Backups — Operational ✅

Plex's built-in database backup is enabled under Settings → Scheduled Tasks.

- **Schedule:** Every three days during the 04:00–06:00 maintenance window
    
- **Storage:** `/mnt/user/appdata/plex/Library/Application Support/Plex Media Server/Plug-in Support/Databases/`
    
- **Contents:** Plex library database and associated blobs database
    
- **Mechanism:** Plex's native database backup functionality
    
- **Status:** Verified operational on 2026-10-26
    

Verified backup sets:

|Date|Library DB|Blobs DB|
|---|---|---|
|2026-10-16|✅|✅|
|2026-10-19|✅|✅|
|2026-10-22|✅|✅|
|2026-10-25|✅|✅|

Backup filenames follow these patterns:

- `com.plexapp.plugins.library.db-YYYY-MM-DD`
    
- `com.plexapp.plugins.library.blobs.db-YYYY-MM-DD`
    

The observed backups were created at approximately 04:03–04:04.

**Limitation:** These backups are stored alongside the live Plex application data. They provide database recovery points but do not constitute independent full-appdata backups or protection against loss of the underlying appdata storage.

Backup file creation has been verified. Database integrity testing and an actual restore test have not yet been performed. The configured backup retention policy has not been independently verified.

### Dedicated Full-Appdata Backup — Pending (Phase 5 Wave 8) ⬜

A dedicated backup of Plex's complete persistent appdata is planned but has not been implemented.

Original design intent:

- Use a custom backup script on `nas-prod-01`.
    
- Stop the Plex container before backing up persistent appdata to avoid copying an actively changing database.
    
- Back up `/mnt/user/appdata/plex` to the NAS `backups` share.
    
- Restart Plex after the backup, including when a backup operation fails.
    
- Use an exit trap or equivalent cleanup mechanism to protect against leaving Plex stopped.
    
- Add scheduled execution, retention, logging, backup verification, and a documented restore procedure.
    

The originally documented destination was `/mnt/user/backups/plex/db/`. The final destination layout, backup format, schedule, and retention policy will be determined during implementation.

**Investigation completed 2026-10-26:**

- The documented `/mnt/user/backups/plex/db/` directory does not exist.
    
- No Plex-specific entries were found in the root crontab.
    
- The Unraid User Scripts directory `/boot/config/plugins/user.scripts/scripts` was absent.
    
- A filename search in `/boot/config`, `/usr/local/bin`, and `/usr/local/sbin` did not locate a dedicated Plex backup script.
    
- Before the October 26 container update, Plex had been running continuously since 2026-09-05. This is inconsistent with regular execution of the originally documented stop–backup–restart workflow.
    

These findings, together with the unfinished Phase 5 Wave 8 plan, indicate that the dedicated backup was not implemented as previously described. Other unexamined scheduling locations cannot be categorically ruled out.

**Capacity check (2026-10-26):**

|Item|Result|
|---|---|
|Plex appdata size|6.8G|
|ZFS mirror total capacity|3.62T|
|ZFS mirror available space|3.41T|
|ZFS mirror health|ONLINE|

Available ZFS capacity is sufficient to consider a dedicated backup solution, subject to the retention policy selected during implementation.

**Decision:** Prefer a custom backup script instead of the existing Unraid Appdata Backup plugin. Implementation remains deferred to Phase 5 Wave 8.

See `runbooks/backup-schedule.md`, `runbooks/backup-restore.md`, and `phases/phase-5-hardening.md`.

---

## Issues Encountered

|Issue|Resolution|
|---|---|
|DB restore permissions error|Extracted files owned by root — Plex runs as 2000:2000. Fixed with `chown -R 2000:2000` on restored files|
|Plex container media path after NFS restructure|Path updated from `/mnt/user` to `/mnt/user/data` after single `data` share was created in Wave 2|
|Dedicated Plex backup incorrectly documented as implemented|Investigation on 2026-10-26 found no configured script or expected destination. Corrected documentation; implementation remains pending Phase 5 Wave 8|

---

## Notes

- Plex is not managed by Dockman — it lives on Unraid and is managed via the Unraid Docker UI
    
- The 4K library migration (existing 4K movies into Radarr 4K instance) is handled via Radarr's Movie Editor — see `services/media/radarr-4k.md`
    
- Future: 4K library expansion via Radarr 4K automated downloads