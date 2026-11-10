# qBitrr

**Role:** Torrent and ARR instance manager — health monitoring, automated search, seeding control  
**Host:** docker-prod-01 (192.168.30.11)  
**Image:** `feramance/qbitrr:latest`  
**Version:** 5.14.9-1 (verified November 2026)  
**Compose:** `/opt/stacks/torrent/compose.yaml`  
**Appdata:** `/opt/appdata/qbitrr/`  
**URL:** `https://qbitrr.giohosted.com/ui`  
**Last Updated:** 2026-11-09

---

## Overview

qBitrr manages torrent health, automated searches, failed-download recovery, and seeding policies across the ARR stack and Shelfmark's book-download categories.

It runs as a separate Docker container on the `proxy` network, outside Gluetun's VPN network namespace, and connects to qBittorrent through `gluetun:8080`.

qBitrr is configured to:

- Monitor torrent health across four ARR instances.
- Search for missing, monitored media.
- Re-search failed and stalled downloads.
- Filter unwanted files using extension allowlists and exclusion patterns.
- Manage public torrents using 60-minute seeding limits.
- Apply tracker-specific MAM seeding requirements to ebooks and audiobooks.
- Maintain torrent-tracking state in a SQLite database.

**Verification status:** Public audiobook downloading, importing, automatic torrent removal, and continued playback have been tested successfully. MAM tracker-specific protection is configured but has not yet been validated with a real MAM torrent.

---

## Managed Instances

| Instance | Type | URI | Category |
|----------|------|-----|----------|
| Sonarr-TV | Sonarr | `http://sonarr-tv:8989` | `sonarr-tv` |
| Sonarr-Anime | Sonarr | `http://sonarr-anime:8989` | `sonarr-anime` |
| Radarr-1080 | Radarr | `http://radarr-1080p:7878` | `radarr-1080` |
| Radarr-4K | Radarr | `http://radarr-4k:7878` | `radarr-4k` |

The additional `audiobooks` and `ebooks` categories are managed through qBitrr's qBittorrent category manager rather than dedicated ARR instances.

---

## Configuration

**Config file:** `/opt/appdata/qbitrr/config.toml`

qBitrr generates a configuration file on first run. Configuration changes are made directly in the file and loaded after restarting the container.

**Operational precaution:** Avoid restarting qBitrr during important active downloads unless necessary. On startup, it reconnects to qBittorrent, initializes its workers, and re-evaluates managed torrents.

### Key Global Settings

| Setting | Value | Notes |
|---------|-------|-------|
| `CompletedDownloadFolder` | `/data/downloads` | Root of download categories |
| `FreeSpace` | `-1` | Free-space automation disabled |
| `AutoPauseResume` | `true` | Available for free-space management |
| `LoopSleepTimer` | `5` | Main-loop interval, seconds |
| `SearchLoopDelay` | `-1` | Uses the application's 30-second search delay |
| `FailedCategory` | `failed` | Failed-torrent handling |
| `RecheckCategory` | `recheck` | Torrent recheck handling |
| `FFprobeAutoUpdate` | `true` | FFprobe binary management enabled |
| `BehindHttpsProxy` | `true` | Used behind Traefik |

The November 2026 performance review found no reason to adjust the loop and polling intervals.

### qBittorrent Connection

| Setting | Value |
|---------|-------|
| `Host` | `gluetun` |
| `Port` | `8080` |
| `UserName` | `admin` |
| `Password` | Stored in `config.toml`; do not expose in logs or documentation |

qBittorrent shares Gluetun's network namespace. Use the `gluetun` hostname rather than `qbittorrent`.

### Per-Instance Search Settings

The following values were verified across all four ARR instances in November 2026.

| Setting | Value | Notes |
|---------|-------|-------|
| `SearchMissing` | `true` | Search for missing monitored content |
| `SearchLimit` | `5` | Limit on queued search commands |
| `SearchRequestsEvery` | `300` | Request-system polling interval, seconds |
| `SearchAgainOnSearchCompletion` | `true` | Allow later re-search passes |
| `SearchByYear` | `true` | Year-based ordering |
| `SearchInReverse` | `false` | Reverse ordering disabled |
| `Unmonitored` | `false` | Do not intentionally search unmonitored content |
| `DoUpgradeSearch` | `false` | Upgrade searches disabled |
| `QualityUnmetSearch` | `false` | Quality-unmet searches disabled |
| `CustomFormatUnmetSearch` | `false` | Custom-format upgrade searches disabled |
| `RefreshDownloadsTimer` | `1` | Download refresh interval, minutes |
| `ReSearch` | `true` | Re-search after failure handling |

**Sonarr-specific options:**

| Setting | Value |
|---------|-------|
| `SearchBySeries` | `smart` |
| `PrioritizeTodaysReleases` | `true` |

Sonarr and Radarr continue to manage their own release quality profiles, monitoring decisions, and normal RSS activity.

**Search-loop assessment:** Continuous searching was retained intentionally. During the November audit, no excessive repeated indexer searches were confirmed. Search-loop delay messages alone do not establish that indexer queries were submitted.

**Monitoring precaution:** In Sonarr-Anime, series-level monitoring alone is not sufficient to assume that Qbitrr will never select individual monitored episodes. Verify both series and episode monitoring when intentionally excluding content.

### Per-Instance Torrent Settings

These settings were verified across Sonarr-TV, Sonarr-Anime, Radarr-1080, and Radarr-4K.

| Setting | Value | Notes |
|---------|-------|-------|
| `StalledDelay` | `15` | Delay before stalled-torrent handling |
| `ReSearchStalled` | `true` | Search for replacement content when stalled |
| `DoNotRemoveSlow` | `true` | Protect slow but active downloads |
| `MaximumETA` | `-1` | ETA-based removal disabled |
| `MaximumDeletablePercentage` | `0.99` | Protect near-complete downloads in applicable failure cleanup |
| `IgnoreTorrentsYoungerThan` | `180` | Initial grace period, seconds |
| `AutoDelete` | `false` | Do not automatically delete unwanted downloaded files |

The old seeding configuration used `RemoveTorrent = -1`, which prevented Qbitrr from removing stalled torrents even when it correctly detected a stall.

That restriction was corrected in November 2026.

**Confirmed historical issue:** The stalled torrent `Dracula Untold (2014)` repeatedly triggered removal decisions while Qbitrr logged that `RemoveTorrent=-1` blocked the action. The corrected configuration removes that specific restriction. A new real-world stalled-download recovery event has not yet been observed.

### File Filtering

**Extension allowlist**, configured for all four ARR instances:

```toml
FileExtensionAllowlist = [".mp4", ".mkv", ".sub", ".ass", ".srt", ".!qB", ".parts"]
```

Executables such as `.exe` are not included.

`AutoDelete = false` is retained as a conservative safeguard. Qbitrr may exclude unwanted files from downloading, but it is not configured to automatically delete disallowed files that already exist.

Folder and filename exclusion patterns cover samples, extras, featurettes, and selected ancillary media.

**November 2026 regex correction:**

Twelve incorrectly escaped `\d` expressions were corrected across the four ARR instances.

For example:

```toml
FileNameExclusionRegex = ["\\bncop\\d+?\\b", "\\bnced\\d+?\\b", "\\bsample\\b", "brarbg.com\\b", "\\btrailer\\b", "music video", "comandotorrents.com"]
```

All 24 follow-up regex tests passed, including `NCOP12`, `NCED05`, and sample filenames.

The complete exclusion lists remain in `config.toml`, which is the authoritative configuration reference.

---

## Seeding Rules

### Overview

qBitrr is the primary seeding-policy manager.

**Important:** qBittorrent's former global 60-minute seeding stop rule was disabled in November 2026 to avoid interfering with tracker-specific policies.

| Content | Intended policy | Status |
|---------|-----------------|--------|
| Public Sonarr/Radarr torrents | 60-minute seeding limit | Configured |
| Public ebook torrents | 60-minute seeding limit | Configured |
| Public audiobook torrents | 60-minute seeding limit | End-to-end verified |
| MyAnonamouse ebooks/audiobooks | At least 14 days AND ratio 1.0 | Configured; live tracker behavior pending verification |

Automatic cleanup applies to torrents managed by these qBitrr categories and instances. Unmanaged categories are not guaranteed to receive the same cleanup.

### ARR Seeding Settings

Each ARR instance uses the following configuration:

```toml
[Sonarr-TV.Torrent.SeedingMode]
MaxUploadRatio = -1
MaxSeedingTime = 3600
RemoveTorrent = 2
```

Equivalent values are configured under:

- `[Sonarr-Anime.Torrent.SeedingMode]`
- `[Radarr-1080.Torrent.SeedingMode]`
- `[Radarr-4K.Torrent.SeedingMode]`

`RemoveTorrent = 2` selects time-based removal. `MaxSeedingTime = 3600` specifies one hour in seconds.

These are seeding-cleanup settings and do not guarantee that every failed or stalled torrent will be handled identically to a successfully completed torrent.

### Books Categories — Default Policy

The managed qBittorrent categories are:

```toml
ManagedCategories = ["audiobooks", "ebooks"]
```

The verified category seeding settings are:

```toml
[qBit.CategorySeeding]
MaxUploadRatio = -1
MaxSeedingTime = 3600
RemoveTorrent = 2
```

This establishes the default one-hour seeding policy for the managed book categories.

### MAM Tracker Override

The old `[qBit.CategorySeeding.Trackers.myanonamouse]` definition was replaced with the supported `[[qBit.Trackers]]` structure.

The verified current configuration is:

```toml
[[qBit.Trackers]]
Name = "MyAnonamouse"
URI = "myanonamouse.net"
Priority = 10
MaxUploadRatio = 1.0
MaxSeedingTime = 1209600
HitAndRunMode = "and"
MinSeedRatio = 1.0
MinSeedingTimeDays = 14
TrackerUpdateBuffer = 3600
AddTrackerIfMissing = false
RemoveIfExists = false
```

**Policy intent:**

- Identify torrents whose tracker matches `myanonamouse.net`.
- Require a minimum upload ratio of 1.0.
- Require at least 14 days of seeding.
- Use `HitAndRunMode = "and"` so both minimum requirements must be met before clearance.
- Apply a 3,600-second buffer to accommodate tracker-statistics reporting lag.
- Avoid automatically adding or removing tracker URLs.

The configured maximum seeding time is 1,209,600 seconds, equivalent to 14 days.

**Important verification boundary:** TOML parsing, worker startup, and the presence of this tracker definition have been verified. A real MAM download has not yet been observed remaining protected through the public torrent's 60-minute threshold. Do not describe tracker-specific compliance as end-to-end verified until that test succeeds.

**Tracker caution:** Keep a MAM torrent seeding until its tracker recognizes the required obligations as satisfied. Do not manually remove it merely because the local client reports a sufficient ratio and elapsed time.

### Verified Public Audiobook Workflow

November 9, 2026: `Threshing Day`, downloaded from AudiobookBay through Shelfmark, was used for an end-to-end test.

1. Shelfmark submitted the audiobook torrent to qBittorrent.
2. qBittorrent completed the download under category `audiobooks`.
3. Shelfmark placed the audiobook into the Audiobookshelf library.
4. Audiobookshelf successfully played the audiobook while the torrent was seeding.
5. After approximately 60 minutes of accumulated seeding, the torrent disappeared from qBittorrent.
6. The imported audiobook remained present and playable in Audiobookshelf.

**Result:** Public audiobook import, Qbitrr cleanup, and retained library playback verified.

This does not constitute verification of MAM's tracker-specific requirements.

---

## WebUI

Access at `https://qbitrr.giohosted.com/ui`.

The WebUI provides:

- **Processes view** — worker and instance status.
- **Logs view** — application activity.
- **ARR views** — queue and media status.
- **Config editor** — configuration editing.

### Authentication

qBitrr uses its own authentication system with OIDC through Authentik.

A WebUI bearer token is stored in `config.toml` under `[WebUI].Token`.

Never copy authentication tokens or API keys into documentation, troubleshooting messages, or publicly shared logs.

---

## Monitoring qBitrr

### Check Container Status

```bash
docker inspect qbitrr \
  --format 'Status: {{.State.Status}} | Restart count: {{.RestartCount}}'
```

### Check Logs

```bash
docker logs qbitrr --tail 50
```

### Follow Live Logs

```bash
docker logs -f qbitrr
```

### Confirm Worker Initialization

The November 2026 deployment starts **13 worker processes**, plus a separately initialized qBit category manager.

Expected worker categories:

```text
search(sonarr-tv), torrent(sonarr-tv)
search(sonarr-anime), torrent(sonarr-anime)
search(radarr-1080), torrent(radarr-1080)
search(radarr-4k), torrent(radarr-4k)
torrent(TorrentPolicyManager)
torrent(failed), torrent(recheck)
torrent(ebooks), torrent(audiobooks)
```

The category manager handles the `audiobooks` and `ebooks` categories.

At the last verified restart, all 13 workers started successfully.

A warning about an already-started category manager was also observed, with that process reported alive; it was not accompanied by a confirmed startup failure.

### Container Resource Usage

```bash
docker stats --no-stream qbitrr
```

November 2026 snapshot:

| Metric | Observed |
|--------|----------|
| CPU usage | 3.45% |
| Memory usage | 368.5 MiB |
| Process/thread count | 36 |
| Restart policy | `unless-stopped` |
| OOM killed | `false` |
| Docker healthcheck | Not configured |

These are point-in-time observations rather than continuous performance metrics.

### Log Management

Docker currently uses the `json-file` logging driver without explicit per-container rotation settings.

The log file measured approximately 7.3 MB during the November audit.

No immediate action was needed.

**Optional future improvement:** Configure bounded Docker log rotation, for example `max-size=10m` and `max-file=3`, during a planned container configuration update. No such change has been implemented.

---

## Database Maintenance

**Database:** `/opt/appdata/qbitrr/qBitManager/qbitrr.db`

qBitrr uses SQLite for application tracking state.

The current deployment runs periodic WAL checkpoint maintenance approximately every five minutes.

### Check Database Files

```bash
sudo find /opt/appdata/qbitrr/qBitManager \
  -maxdepth 1 -type f \
  \( -name '*.db' -o -name '*.db-wal' -o -name '*.db-shm' \) \
  -printf '%p — %s bytes\n'
```

### Read-Only SQLite Integrity Check

```bash
python3 - <<'PY'
import sqlite3

path = "file:/opt/appdata/qbitrr/qBitManager/qbitrr.db?mode=ro"

with sqlite3.connect(path, uri=True, timeout=5) as conn:
    result = conn.execute("PRAGMA quick_check").fetchone()[0]
    print(f"SQLite integrity: {result}")
PY
```

**November 2026 result:** `SQLite integrity: ok`

Repeated WAL checkpoints completed successfully without reported maintenance errors.

Do not manually delete active SQLite WAL or SHM files.

### November 2026 Orphaned-Record Cleanup

During the audit, 384 stale Sonarr-Anime database entries were identified. These referenced missing or no-longer-valid episode records.

After a database backup and targeted cleanup:

- Exactly 384 orphaned entries were removed.
- SQLite integrity was verified.
- Sonarr-Anime's stale records were no longer present.
- Sonarr-TV's tracking records were preserved.
- Sixteen unrelated leftover Bleach media files, totaling approximately 17 GB, were removed after explicit approval.

Pre-cleanup backup:

```text
/opt/appdata/qbitrr/backups/qbitrr-pre-cleanup-20261108T233539Z.db
```

The cleanup was verified, but a suspected search-state/reset issue was not conclusively established as the underlying cause. Monitor for recurrence rather than assuming the issue has been permanently eliminated.

---

## Directory Structure

```text
/opt/stacks/torrent/
  compose.yaml                      ← shared with Gluetun and qBittorrent
  .env                              ← credentials; not for documentation

/opt/appdata/qbitrr/
  config.toml                       ← active configuration
  config.toml.pre-seeding-optimization
  config.toml.pre-regex-fix-*       ← November 2026 backup
  backups/
    qbitrr-pre-cleanup-*.db         ← pre-cleanup database snapshot
  logs/                             ← application logs
  qBitManager/
    qbitrr.db                       ← SQLite application database
    qbitrr.db-wal                   ← SQLite write-ahead log when present
    qbitrr.db-shm                   ← SQLite shared memory when present
```

Additional generated support files may be present depending on the running version.

---

## Troubleshooting

### Instance Showing Configuration Errors at Startup

Inspect relevant configuration fields without printing passwords or API keys.

```bash
python3 - <<'PY'
import tomllib

with open("/opt/appdata/qbitrr/config.toml", "rb") as f:
    cfg = tomllib.load(f)

for name in ("Sonarr-TV", "Sonarr-Anime", "Radarr-1080", "Radarr-4K"):
    print(name, "configured:", name in cfg)
PY
```

Confirm the affected instance URI includes the correct `http://` scheme.

Restart qBitrr only after validating the configuration and considering any active downloads.

### Cannot Connect to qBittorrent

qBitrr connects through `gluetun:8080`.

Check Gluetun and qBittorrent:

```bash
docker ps --filter name=gluetun
docker ps --filter name=qbittorrent
docker logs gluetun --tail 20
```

Consult `qbittorrent.md` and `gluetun.md` for the network and VPN architecture.

### Stalled Torrent Repeatedly Marked for Deletion

Check for messages resembling:

```text
Removal blocked ... RemoveTorrent=-1 (never remove)
```

In November 2026, that policy prevented the removal of a stalled Radarr torrent.

The intended current setting for the four ARR instances is:

```toml
RemoveTorrent = 2
```

Review the current seeding policy, torrent state, and hit-and-run clearance before making changes.

Do not assume a log entry saying **Marking for deletion** means the torrent was actually removed. Look for confirmed removal messages or a deletion summary with `confirmed_removed` greater than zero.

### MAM Torrent Removed Too Early or Stopped Seeding

Verify:

1. qBittorrent's global seeding limits remain disabled.
2. The torrent's tracker URL matches the configured MAM tracker definition.
3. The expected `[[qBit.Trackers]]` entry is loaded.
4. `HitAndRunMode = "and"` is configured.
5. The torrent remains managed by the intended category.
6. The actual torrent state and Qbitrr logs support the removal decision.

**Do not assume MAM protection is verified based solely on a successful configuration restart.**

### Unwanted or Fake Releases

Check:

- Sonarr/Radarr release history and blocklist.
- Indexer configuration and release restrictions.
- Downloaded file extensions.
- Qbitrr file-exclusion decisions.
- Whether the episode or movie is actually available.

During the November audit, suspicious unreleased episodes for `Dexter: Resurrection` and `Abbott Elementary` were removed and blocklisted. Their exact file contents were not established.

Review Prowlarr indexer restrictions if unrelated content sources continue supplying unsuitable releases.

### Regex Patterns Not Matching

TOML double-quoted strings require escaped backslashes.

Correct example:

```toml
"\\bncop\\d+?\\b"
```

Incorrect example:

```toml
"\\bncop\\\\d+?\\b"
```

The November 2026 correction was validated with 24 passing regex tests.

### Shelfmark Imports Failing

Shelfmark can encounter stale NFS file handles on bind-mounted download or library paths.

During the November audit:

- Host `/data` directory listings were successful.
- Other media containers could list their directories.
- Shelfmark returned `Stale file handle` when listing mounted directories.
- Restarting only Shelfmark restored access.
- Direct EPUB import into CWA and torrent-based audiobook import into Audiobookshelf subsequently succeeded.

This was treated as a resolved incident, not proof that the underlying NFS issue can never recur.

See `../books/shelfmark.md` for the complete workflow and incident notes.

### Configuration Changes Not Taking Effect

Validate `config.toml`, then restart qBitrr:

```bash
docker restart qbitrr
```

Inspect startup logs for worker initialization.

---

## Operational Notes and Outstanding Verification

### Verified as of November 9, 2026

- Qbitrr v5.14.9-1 deployed.
- Four ARR instances configured.
- Two qBit-managed book categories configured.
- All 13 worker processes started successfully.
- Public audiobook cleanup and retained library playback verified.
- Historical stalled-removal restriction corrected.
- File-exclusion regex escaping corrected across all four ARR instances.
- SQLite integrity check passed.
- Database maintenance checkpoints operating normally.
- CPU and memory usage reasonable.
- Restart policy set to `unless-stopped`.

### Configured but Not Yet Verified End-to-End

- MAM tracker matching and continued seeding beyond the public 60-minute limit.
- MAM protection until both the minimum time and upload ratio are satisfied.
- New stalled-torrent detection followed by successful removal and replacement under the corrected configuration.
- Torrent-based ebook import and subsequent cleanup, distinct from the successful direct-download EPUB test.

### Optional Improvements

- Add bounded Docker JSON log rotation during a planned maintenance change.
- Investigate recurring excessive search traffic only if monitoring reveals actual repeated searches or indexer strain.
- Avoid introducing additional Shelfmark/Qbitrr integration complexity unless failed imports recur.

---

## References

- [qBitrr Configuration](https://feramance.github.io/qBitrr/configuration/)
- [qBitrr Seeding and Trackers](https://feramance.github.io/qBitrr/configuration/seeding/)
- [qBitrr Torrent Settings](https://feramance.github.io/qBitrr/configuration/torrents/)
- [qBitrr Automated Search](https://feramance.github.io/qBitrr/features/automated-search/)
- [qBitrr Health Monitoring](https://feramance.github.io/qBitrr/features/health-monitoring/)

---

## Notes

- qBitrr runs on the `proxy` network, outside Gluetun's VPN namespace.
- qBitrr connects to qBittorrent using `gluetun`, not `qbittorrent`.
- ARR applications connect through their Docker service names.
- The container's `tty: true` setting was part of the documented deployment and should be retained unless the Compose configuration is deliberately reviewed.
- Auto-updates were documented as disabled; container updates are managed manually. Reconfirm that setting before relying on it operationally.
- Sensitive credentials, WebUI tokens, and API keys belong in their existing protected configuration locations, not in this documentation.
