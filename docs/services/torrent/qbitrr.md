# qBitrr

**Role:** Torrent and ARR instance manager — health monitoring, automated search, seeding control  
**Host:** docker-prod-01 (192.168.30.11)  
**Image:** feramance/qbitrr:latest  
**Version:** 5.10.1  
**Compose:** `/opt/stacks/torrent/compose.yaml`  
**Appdata:** `/opt/appdata/qbitrr/`  
**URL:** `https://qbitrr.giohosted.com/ui`  
**Last Updated:** 2026-03-19

---

## Overview

qBitrr is the glue between qBittorrent and the ARR stack. It runs as a separate container on the `proxy` network (not inside Gluetun's namespace) and connects to qBittorrent via the `gluetun` hostname.

qBitrr is configured to:
- Monitor torrent health across all 4 ARR instances
- Trigger automated searches for missing media
- Re-search failed or stalled torrents
- Tag torrents for seeding tracking
- Enforce MAM-specific seeding rules for ebooks and audiobooks (14-day minimum, 1.0 ratio)

---

## Managed Instances

| Instance | Type | URI | Category |
|----------|------|-----|----------|
| Sonarr-TV | Sonarr | `http://sonarr-tv:8989` | `sonarr-tv` |
| Sonarr-Anime | Sonarr | `http://sonarr-anime:8989` | `sonarr-anime` |
| Radarr-1080 | Radarr | `http://radarr-1080p:7878` | `radarr-1080` |
| Radarr-4K | Radarr | `http://radarr-4k:7878` | `radarr-4k` |

---

## Configuration

Config file: `/opt/appdata/qbitrr/config.toml`

qBitrr generates a default `config.toml` on first run. The file is edited directly — changes take effect after a container restart.

> **Do not restart qBitrr while it is actively processing torrents** — it will re-tag and re-evaluate all torrents on startup which can cause brief disruption to in-progress downloads.

### Key Global Settings

| Setting | Value | Notes |
|---------|-------|-------|
| `CompletedDownloadFolder` | `/data/downloads` | Root of all download categories |
| `FreeSpace` | `-1` | Disabled — no automatic pause on low disk |
| `AutoPauseResume` | `true` | Required for FreeSpace logic if enabled later |
| `LoopSleepTimer` | `5` | Seconds between torrent reprocessing loops |
| `FailedCategory` | `failed` | Torrents tagged with this are treated as failed |
| `RecheckCategory` | `recheck` | Torrents tagged with this are force-rechecked |
| `BehindHttpsProxy` | `true` | Required — qBitrr is behind Traefik |

### qBittorrent Connection

| Setting | Value |
|---------|-------|
| `Host` | `gluetun` |
| `Port` | `8080` |
| `UserName` | `admin` |
| `Password` | set in config.toml (not in .env — qBitrr reads directly from config) |

### Per-Instance Search Settings

| Setting | Value | Notes |
|---------|-------|-------|
| `SearchMissing` | `true` | Actively searches for missing media |
| `SearchLimit` | `5` | Max concurrent searches per instance |
| `SearchByYear` | `true` | Searches newest to oldest by release year |
| `SearchBySeries` | `smart` | Sonarr only — uses series or episode search intelligently |
| `PrioritizeTodaysReleases` | `true` | Sonarr only — today's episodes searched first |
| `DoUpgradeSearch` | `false` | ARR apps handle upgrades natively — no double-dipping |
| `RssSyncTimer` | `1` | RSS sync every 1 minute |
| `RefreshDownloadsTimer` | `1` | Queue refresh every 1 minute |
| `ReSearch` | `true` | Re-search failed torrents |

### Per-Instance Torrent Settings

| Setting | Value | Notes |
|---------|-------|-------|
| `StalledDelay` | `15` | Minutes before a stalled torrent is acted on |
| `ReSearchStalled` | `true` | Stalled torrents are automatically re-searched for an alternative source |
| `DoNotRemoveSlow` | `true` | Slow torrents are not removed — only stalled ones |
| `MaximumDeletablePercentage` | `0.99` | Never delete torrents above 99% complete |
| `IgnoreTorrentsYoungerThan` | `180` | Seconds — new torrents are left alone |
| `RemoveTorrent` | `-1` | Never remove ARR torrents — seeding is indefinite |

---

## Seeding Rules

### Books categories — global defaults

The `[qBit.CategorySeeding]` block applies to the `audiobooks` and `ebooks` categories. Global defaults allow indefinite seeding with no ratio or time cap — the MAM tracker override below handles enforcement.

| Setting | Value |
|---------|-------|
| `MaxUploadRatio` | `-1` (unlimited) |
| `MaxSeedingTime` | `-1` (unlimited) |
| `MinSeedRatio` | `1.0` |
| `MinSeedingTimeDays` | `0` |
| `RemoveTorrent` | `-1` (never — MAM override handles removal) |

### MAM tracker override

MAM (MyAnonamouse) torrents in the `audiobooks` and `ebooks` categories are subject to tracker-scoped seeding rules. Once both conditions are met (14 days AND 1.0 ratio), qBitrr removes the torrent from qBittorrent automatically.
```toml
[qBit.CategorySeeding.Trackers.myanonamouse]
URL = "myanonamouse.net"
MinSeedingTimeDays = 14
MinSeedRatio = 1.0
MaxUploadRatio = -1
MaxSeedingTime = -1
RemoveTorrent = true
HitAndRunMode = "disabled"
```

**Why `HitAndRunMode = "disabled"`:** MAM audiobooks and ebooks are often slow seeds — HitAndRun detection could incorrectly flag legitimate slow-seeding torrents. The 14-day + 1.0 ratio requirements already satisfy MAM's seeding policy without it.

Torrents from AudiobookBay or any other tracker are unaffected by this rule — they fall back to the global defaults and seed indefinitely.

---

## WebUI

Access at `https://qbitrr.giohosted.com/ui`.

The WebUI provides:
- **Processes view** — live status of all worker processes per ARR instance
- **Logs view** — real-time log output
- **Arr views** — queue and library status per instance
- **Config editor** — edit `config.toml` directly from the browser (restart still required after save)

### Authentication

qBitrr WebUI uses its own auth system with OIDC via Authentik. A bearer token is also auto-generated in `config.toml` under `[WebUI].Token` for API access.

---

## Monitoring qBitrr

### Check logs
```bash
docker logs qbitrr --tail 50
```

### Watch live
```bash
docker logs -f qbitrr
```

### Confirm all instances are connecting
On a healthy startup, 12 workers start:
```
search(sonarr-tv), torrent(sonarr-tv)
search(sonarr-anime), torrent(sonarr-anime)
search(radarr-1080), torrent(radarr-1080)
search(radarr-4k), torrent(radarr-4k)
torrent(recheck), torrent(failed)
torrent(ebooks), torrent(audiobooks)
```

If you see `CRITICAL: Failed to connect to Arr instance` — check the URI and API key in `config.toml` for that instance.

---

## Directory Structure
```
/opt/stacks/torrent/
  compose.yaml                        ← in git (shared with Gluetun and qBittorrent)
  .env                                ← gitignored

/opt/appdata/qbitrr/
  config.toml                         ← main config — edit directly, restart to apply
  config.backup.<timestamp>.toml      ← auto-generated backup before schema migrations
  logs/                               ← application logs
  qBitManager/
    qbitrr.db                         ← SQLite database — torrent tracking state
    ffprobe.exe                       ← auto-downloaded by qBitrr for media validation
```

---

## Troubleshooting

### Instance showing CHANGE_ME errors on startup
Config file was not saved correctly before container started. Stop the container, verify `config.toml` has correct values, then start:
```bash
docker stop qbitrr
cat /opt/appdata/qbitrr/config.toml | grep -E "Host|URI|APIKey|CompletedDownload"
docker start qbitrr
```

### Can't connect to qBittorrent
qBitrr connects to qBittorrent via `gluetun:8080`. Verify Gluetun is running and healthy:
```bash
docker ps | grep gluetun
```

### Sonarr-Anime showing "No series returned"
Expected if Sonarr-Anime has an empty library. Not an error — qBitrr has nothing to search for.

### Search loop crashing repeatedly
Check the URI for that instance in `config.toml` — must include `http://` scheme. Missing scheme causes an immediate crash with `MissingSchema` error.

### Config changes not taking effect
qBitrr reads `config.toml` on startup only. Always restart after editing:
```bash
docker restart qbitrr
```

---

## Notes

- qBitrr is on the `proxy` network — unlike qBittorrent it is NOT inside Gluetun's namespace
- qBitrr connects to ARR instances by container name (e.g. `sonarr-tv`) — works because all containers share the `proxy` Docker network
- qBitrr connects to qBittorrent via `gluetun` hostname — not `qbittorrent`, because qBittorrent has no independent network presence
- The `tty: true` setting in the compose is required — qBitrr uses terminal control codes in its output and will behave oddly without it
- Auto-updates are disabled (`AutoUpdateEnabled = false`) — updates are handled manually by pulling the latest image via Dockman