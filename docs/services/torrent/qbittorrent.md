# qBittorrent

**Role:** Torrent client — runs inside Gluetun VPN network namespace  
**Host:** docker-prod-01 (192.168.30.11)  
**Image:** `lscr.io/linuxserver/qbittorrent:latest`  
**Version:** 5.3.0 (verified November 2026)  
**Compose:** `/opt/stacks/torrent/compose.yaml`  
**Appdata:** `/opt/appdata/qbittorrent/`  
**URL:** `https://qbittorrent.giohosted.com`  
**Last Updated:** 2026-11-09

---

## Overview

qBittorrent is the torrent client for media downloads across the ARR stack and Shelfmark.

It runs inside Gluetun's network namespace, meaning it has no independent Docker network identity. Torrent traffic routes through the ProtonVPN WireGuard tunnel, and Gluetun provides the VPN connectivity and network-level protection.

If Gluetun's VPN connectivity is lost, its firewall rules are intended to prevent qBittorrent from leaking torrent traffic through the normal network connection.

qBittorrent is responsible for:

- Downloading torrents submitted by Sonarr, Radarr, and Shelfmark.
- Organizing downloads using categories and save paths.
- Maintaining torrent data for active downloads and seeding.
- Using the VPN-forwarded listening port supplied by Gluetun.
- Providing the WebUI and API used by the ARR applications and qBitrr.

**Seeding-policy management is delegated to qBitrr.** qBittorrent's former native global 60-minute seeding rule was disabled in November 2026 to prevent interference with tracker-specific seeding requirements.

See `services/torrent/gluetun.md` for VPN and network architecture details.

See `services/torrent/qbitrr.md` for automated torrent health monitoring, search, recovery, and seeding policies.

---

## Network

| Setting | Value |
|---------|-------|
| Network mode | `service:gluetun` — shares Gluetun's network namespace |
| WebUI port | `8080` |
| WebUI URL | `https://qbittorrent.giohosted.com` |
| Reverse proxy | Traefik routes through Gluetun |
| Network interface | `tun0` — VPN tunnel interface |
| VPN provider | ProtonVPN |
| VPN protocol | WireGuard |
| Torrent listening port | Dynamic — updated using Gluetun's port-forwarding script |
| UPnP / NAT-PMP | Disabled in qBittorrent |

### Network Architecture

qBittorrent does not have its own network presence on the Docker `proxy` network.

Instead:

1. Gluetun establishes the ProtonVPN WireGuard tunnel.
2. qBittorrent shares Gluetun's network namespace.
3. Gluetun's firewall protects torrent traffic from bypassing the VPN.
4. A Gluetun-side script obtains and configures the forwarded listening port.
5. Traefik exposes the qBittorrent WebUI through Gluetun.
6. Sonarr, Radarr, Shelfmark, and qBitrr communicate with qBittorrent using `http://gluetun:8080`.

**Important:** Do not change qBittorrent's network interface binding from `tun0` without evaluating the VPN security implications.

### Port Forwarding

qBittorrent does not independently request port forwarding from the local router.

Instead, Gluetun manages VPN port forwarding through ProtonVPN.

The port-forwarding script is located at:

```text
/opt/appdata/gluetun/scripts/qbit-set-port.sh
```

The script updates qBittorrent's listening port through its WebUI API.

**UPnP/NAT-PMP being disabled in qBittorrent does not mean VPN port forwarding is disabled.** These are separate mechanisms.

---

## Download Categories

Categories must match the values configured in Sonarr, Radarr, Shelfmark, and qBitrr.

| Category | Save Path | Managed By |
|----------|-----------|------------|
| `sonarr-tv` | `/data/downloads/tv` | Sonarr-TV / qBitrr |
| `sonarr-anime` | `/data/downloads/anime` | Sonarr-Anime / qBitrr |
| `radarr-1080` | `/data/downloads/movies/1080p` | Radarr-1080 / qBitrr |
| `radarr-4k` | `/data/downloads/movies/4k` | Radarr-4K / qBitrr |
| `ebooks` | `/data/downloads/books/ebooks/downloads` | Shelfmark / qBitrr |
| `audiobooks` | `/data/downloads/books/audiobooks/downloads` | Shelfmark / qBitrr |

### Category Management

The four ARR categories correspond to separate qBitrr application instances.

The two book categories are configured under qBitrr's `qBit.ManagedCategories` setting:

```toml
ManagedCategories = ["audiobooks", "ebooks"]
```

These allow qBitrr to manage audiobook and ebook torrents without requiring dedicated Sonarr or Radarr instances.

Category names and paths should remain consistent across qBittorrent and the services that submit downloads.

---

## ARR Download Client Configuration

All four ARR instances connect to qBittorrent using:

| Field | Value |
|-------|-------|
| Host | `gluetun` |
| Port | `8080` |
| Username | `admin` |
| Password | Stored in the existing protected configuration |
| Client type | qBittorrent |

**Host must be `gluetun`, not `qbittorrent`.**

qBittorrent shares Gluetun's network namespace, and port 8080 is accessible through Gluetun.

| ARR Instance | Category |
|--------------|----------|
| Sonarr-TV | `sonarr-tv` |
| Sonarr-Anime | `sonarr-anime` |
| Radarr-1080 | `radarr-1080` |
| Radarr-4K | `radarr-4k` |

### Shelfmark Download Client

Shelfmark connects to the same qBittorrent instance using:

```text
http://gluetun:8080/
```

Shelfmark assigns:

- `ebooks` to ebook torrents.
- `audiobooks` to audiobook torrents.

Shelfmark manages file import and hardlink creation. qBitrr manages the configured torrent seeding and cleanup policies.

See `services/books/shelfmark.md` for complete ebook and audiobook workflows.

---

## Key Settings

The following settings reflect the existing deployment documentation and confirmed November 2026 changes. Settings not individually retested during the audit should be reconfirmed in the WebUI before consequential troubleshooting.

| Setting | Value | Location |
|---------|-------|----------|
| Default save path | `/data/downloads` | Settings → Downloads |
| Network interface | `tun0` | Settings → Advanced |
| UPnP / NAT-PMP | Disabled | Settings → Connection |
| Torrent listening port | Dynamically set by Gluetun script | Settings → Connection |
| Incomplete torrent folder | Disabled | Settings → Downloads |
| DHT | Enabled | Settings → BitTorrent |
| Peer Exchange | Enabled | Settings → BitTorrent |
| Global upload speed limit | 350,000 KiB/s (historically documented) | Settings → Speed |
| Maximum upload slots per torrent | 5 (historically documented) | Settings → Connection / BitTorrent |
| Native global seeding time limit | Disabled | Settings → BitTorrent |
| Seeding cleanup management | qBitrr | `/opt/appdata/qbitrr/config.toml` |

### November 2026 Seeding Change

Previously, qBittorrent had a native global 60-minute seeding stop rule.

**That rule was disabled during the November 2026 qBitrr optimization.**

The old global limit could interfere with MAM torrents that must remain seeding beyond one hour.

The intended architecture is:

- qBittorrent performs downloading and seeding.
- qBitrr evaluates managed torrent categories and applicable tracker policies.
- qBitrr controls the configured automatic torrent cleanup.

Do not re-enable qBittorrent's global 60-minute rule without reviewing its impact on private-tracker requirements.

---

## Seeding Policies

### Overview

qBitrr manages the seeding policies for the four ARR instances and the two book categories.

| Torrent Type | Intended Policy | Verification |
|--------------|-----------------|--------------|
| Sonarr-TV | One-hour seeding limit | Configuration verified |
| Sonarr-Anime | One-hour seeding limit | Configuration verified |
| Radarr-1080 | One-hour seeding limit | Configuration verified |
| Radarr-4K | One-hour seeding limit | Configuration verified |
| Public ebooks | One-hour seeding limit | Configuration verified |
| Public audiobooks | One-hour seeding limit | End-to-end verified |
| MyAnonamouse torrents | Minimum 14 days AND 1.0 upload ratio | Configured; live verification pending |

### ARR Torrent Cleanup

All four ARR instances use the following qBitrr seeding configuration:

```toml
MaxUploadRatio = -1
MaxSeedingTime = 3600
RemoveTorrent = 2
```

These values are configured in each ARR instance's `[Torrent.SeedingMode]` section.

The intended behavior is time-based cleanup after approximately one hour of seeding, subject to Qbitrr's applicable processing and removal conditions.

### Ebook and Audiobook Cleanup

The managed book categories use:

```toml
[qBit.CategorySeeding]
MaxUploadRatio = -1
MaxSeedingTime = 3600
RemoveTorrent = 2
```

For public torrents, this provides the default one-hour seeding policy.

### MyAnonamouse (MAM) Exception

MAM torrents have additional private-tracker requirements.

The tracker-specific configuration in qBitrr is:

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

- Require at least 14 days of seeding.
- Require a minimum upload ratio of 1.0.
- Use AND logic so both requirements are satisfied.
- Protect matching MAM torrents from the normal public-torrent cleanup policy.
- Allow for delayed tracker-statistics updates.

**Verification limitation:** The configuration parses correctly and qBitrr workers initialize successfully, but a live MAM torrent has not yet been used to validate the tracker-specific exception.

Until verified, do not manually remove MAM torrents based solely on the local client displaying 14 days of seeding or a ratio of 1.0. Verify the private tracker's actual requirements and recorded status.

Full tracker-policy details are documented in `services/torrent/qbitrr.md`.

### Important File-Retention Behavior

Deleting or removing a completed torrent is not necessarily the same as deleting its imported library media.

The system relies on hardlink-based imports where applicable.

Once the library has a separate hardlink to a completed file, removing the original download directory entry does not remove the data referenced by the library hardlink.

However, this depends on the import having completed correctly.

**Do not assume every completed torrent has already been imported.** Check Sonarr, Radarr, Shelfmark, or the destination library when troubleshooting missing files.

---

## Config File

The qBittorrent configuration file is located at:

```text
/opt/appdata/qbittorrent/qBittorrent/qBittorrent.conf
```

qBittorrent may write configuration changes during normal operation and shutdown.

**Important:** Do not manually edit the configuration file while the container is running. Changes may be overwritten.

For direct file edits, use a planned maintenance window:

```bash
docker stop qbittorrent
sudo nano /opt/appdata/qbittorrent/qBittorrent/qBittorrent.conf
docker start qbittorrent
```

For ordinary configuration changes, prefer the qBittorrent WebUI when available.

The configuration files are owned by the service account used by the container, historically UID/GID `2000:2000`.

Avoid modifying file ownership or permissions unless the existing state has been inspected and the change is necessary.

---

## Directory Structure

```text
/opt/stacks/torrent/
  compose.yaml                         ← Gluetun, qBittorrent, qBitrr
  .env                                 ← protected environment values

/opt/appdata/qbittorrent/
  qBittorrent/
    qBittorrent.conf                   ← main configuration
    categories.json                    ← download categories
    BT_backup/                         ← torrent resume data
    GeoDB/                             ← peer geolocation data
    logs/                              ← application logs
```

### Download Data

```text
/data/
  downloads/
    tv/
    anime/
    movies/
      1080p/
      4k/
    books/
      ebooks/
        downloads/
        ingest/
      audiobooks/
        downloads/
  media/
    tv/
    movies/
    books/
      ebooks/
      audiobooks/
```

The directories above describe the logical download and library layout, not a separate filesystem for each category.

---

## Permissions

qBittorrent runs with the documented PUID/PGID:

```text
2000:2000
```

The `/data/downloads` NFS mount must provide appropriate write permissions for that user and group.

Historically, the NAS-side ownership was corrected during the Homelab v3 migration:

```bash
# Historical deployment action on nas-prod-01
chown -R 2000:2000 /mnt/user/data/
```

**Do not rerun the recursive ownership command automatically.** Inspect the current permissions, affected paths, and other services before applying ownership changes.

### Permission Troubleshooting

If torrents show errors or cannot write files:

1. Confirm `/data` is mounted and accessible.
2. Inspect ownership and permissions for the affected download directory.
3. Confirm qBittorrent's actual runtime UID/GID.
4. Check qBittorrent logs for the specific filesystem error.
5. Verify the NFS export and NAS permissions if necessary.

A read-only inspection on `docker-prod-01`:

```bash
stat -c '%n | %U:%G | %a' \
  /data \
  /data/downloads \
  /data/downloads/tv
```

Only change permissions after identifying a real mismatch.

---

## Hardlinks

qBittorrent, Sonarr, Radarr, and the media applications use a shared `/data` filesystem layout.

On `docker-prod-01`, `/data` is mounted from the Unraid NAS:

```text
192.168.30.16:/mnt/user/data
```

The deployment uses NFSv3 and a single parent mount to preserve hardlink compatibility.

### Why the Single NFS Mount Matters

Hardlinks require the source and destination to exist on the same filesystem.

During the Homelab v3 deployment, separate `media` and `downloads` NFS exports prevented hardlinks from working across those paths.

The solution was to restructure the NAS storage so that:

```text
/mnt/user/data/
  downloads/
  media/
```

is exposed through one NFS share.

**Do not split `/data/media` and `/data/downloads` into independent mounts without redesigning the hardlink workflow.**

### ARR Hardlink Workflow

1. Sonarr or Radarr sends a torrent to qBittorrent.
2. qBittorrent downloads the content into its category directory.
3. The ARR application imports the completed media using a hardlink where supported.
4. One directory entry remains under downloads for seeding.
5. A second directory entry exists in the media library.
6. When cleanup removes the download-side entry, the library-side hardlink can preserve the file data.

### Verify Hardlinks

Compare inode numbers and hardlink counts for a known imported file:

```bash
stat /data/downloads/tv/<torrent-file>.mkv
stat /data/media/tv/<show>/<imported-file>.mkv
```

The files should have matching inode numbers and a hardlink count reflecting the additional directory entry.

Use actual file paths when running this check.

### Books and Audiobooks

Shelfmark uses the same underlying filesystem approach.

**Ebooks:**

- qBittorrent downloads to `/data/downloads/books/ebooks/downloads`.
- Shelfmark hardlinks completed files into the CWA ingest directory.
- CWA imports the ebook into its managed library.

**Audiobooks:**

- qBittorrent downloads to `/data/downloads/books/audiobooks/downloads`.
- Shelfmark hardlinks the completed audiobook into `/data/media/books/audiobooks`.
- Audiobookshelf detects and serves the imported media.

During the November 2026 audit, a public audiobook remained playable in Audiobookshelf after qBitrr removed its torrent.

### NFS Reliability Considerations

The single shared NFS mount is an intentional architecture decision and should be preserved.

A November 2026 incident affected Shelfmark's existing NFS directory handles:

- Shelfmark returned `Stale file handle` errors.
- The host's `/data` mount was accessible.
- Other containers continued to access their directories.
- Restarting Shelfmark restored directory access.
- Subsequent ebook and audiobook imports succeeded.

The incident did not establish that qBittorrent's own filesystem access was failing.

If similar errors recur, investigate the affected container, host NFS mount, and NAS export before changing the storage architecture.

See `services/books/shelfmark.md`.

---

## Troubleshooting

### All Torrents Errored or Download Speed Is Zero

First distinguish between filesystem errors, VPN connectivity, and tracker/peer availability.

Check container status:

```bash
docker ps --filter name=gluetun
docker ps --filter name=qbittorrent
```

Review logs:

```bash
docker logs gluetun --tail 30
docker logs qbittorrent --tail 30
```

Check relevant download directory permissions:

```bash
stat /data/downloads
```

If the torrent reports a permission error, investigate the specific save path rather than recursively modifying ownership across the entire NAS.

### WebUI Not Reachable / Bad Gateway

qBittorrent's WebUI is exposed through Gluetun.

Check Gluetun first:

```bash
docker ps --filter name=gluetun
docker logs gluetun --tail 20
```

Also verify that Traefik can reach Gluetun on port 8080.

A qBittorrent WebUI failure does not necessarily mean the VPN tunnel itself is down.

### Listening Port Not Matching VPN-Forwarded Port

Gluetun obtains a dynamic forwarded port and uses its script to update qBittorrent.

Inspect the Gluetun logs:

```bash
docker logs gluetun --tail 100
```

Review the port-forwarding script and confirm the reported forwarded port matches qBittorrent's active listening port.

**Historical recovery action:** During the initial Homelab v3 deployment, a full torrent-stack restart resolved a port-forward synchronization problem.

```bash
cd /opt/stacks/torrent
docker compose down
docker compose up -d
```

**Caution:** These commands interrupt Gluetun, qBittorrent, qBitrr, and other services defined by the Compose project. They should not be used as the first troubleshooting step while important downloads are active.

Prefer read-only investigation before restarting the stack.

### Torrent Stops Seeding After One Hour

Determine whether the torrent belongs to a managed public category or a private tracker with additional obligations.

For public content managed by qBitrr, removal after approximately one hour is expected.

For MAM content, early removal would be unexpected under the intended tracker-specific policy and requires investigation.

Check:

- Torrent category.
- Tracker URL.
- qBitrr tracker-match configuration.
- Whether qBittorrent has any native seeding stop rule enabled.
- Torrent state and relevant Qbitrr logs.

Do not assume that all torrent categories share identical policies.

### MAM Torrent Not Seeding Long Enough

Check both qBittorrent's native limits and qBitrr's tracker rules.

The expected configuration is:

```text
qBittorrent native global seeding limit: disabled

qBitrr MAM tracker policy:
  Minimum seed time: 14 days
  Minimum upload ratio: 1.0
  Hit-and-run mode: and
```

MAM compliance remains pending live verification.

Do not manually remove the torrent until the applicable tracker obligations have been confirmed as satisfied.

### Completed Download Missing From Library

A completed torrent does not guarantee a successful library import.

For ARR media, inspect Sonarr/Radarr activity and import history.

For books, inspect Shelfmark and the destination application.

Confirm both the original download path and the destination library path are accessible.

If filesystem errors appear, investigate the shared NFS mount and container bind mounts.

### Configuration Changes Reverting After Restart

qBittorrent may overwrite manually edited configuration values during shutdown.

Use the WebUI for normal configuration changes.

For direct configuration-file editing, stop the container first and make a backup of the configuration file.

Do not modify live configuration files and then assume the changes will survive a restart.

---

## November 2026 Audit Findings

The following findings were confirmed during the qBitrr optimization and troubleshooting audit.

### Verified

- qBittorrent version 5.3.0 was observed.
- The old native global 60-minute seeding rule was disabled.
- qBitrr manages the four ARR torrent categories.
- qBitrr manages the `ebooks` and `audiobooks` categories.
- Public audiobook downloading and import worked after recovery from a Shelfmark NFS issue.
- A public audiobook remained playable after its torrent was automatically removed.
- qBitrr's seeding settings were confirmed on disk.
- qBitrr's 13 workers started successfully after restart.

### Configured but Awaiting Live Verification

- MAM tracker matching and protection beyond the public one-hour policy.
- MAM torrent cleanup only after both the 14-day and 1.0-ratio requirements are satisfied.
- A newly stalled torrent successfully completing the full automated removal and replacement workflow.
- Torrent-based ebook cleanup with successful retained library import.

### Optional Follow-Up

- Reconfirm the historically documented upload speed and slot limits in the qBittorrent WebUI during a future configuration review.
- Investigate recurring NFS errors if they reappear.
- Avoid introducing additional download-client automation without evidence of an operational need.

---

## Notes

- qBittorrent was restored from the Homelab v2 backup at `/mnt/user/backups/docker-v2/appdata/torrent-stack/qbittorrent/`.
- The v2 backup contained `Session\DefaultSavePath=/data/torrents`; this was corrected to `/data/downloads` during the v3 migration.
- All ARR applications connect using `gluetun:8080`.
- Shelfmark uses the same qBittorrent instance with separate book categories.
- qBittorrent runs within Gluetun's VPN network namespace.
- Gluetun handles VPN port forwarding and protection against traffic escaping the VPN.
- Keep qBittorrent's network interface bound to `tun0`.
- Keep qBittorrent's global seeding stop rule disabled while qBitrr manages private-tracker exceptions.
- Maintain the single shared `/data` NFS mount for hardlink compatibility.
- qBitrr handles torrent monitoring, automated recovery, and seeding policies; see `services/torrent/qbitrr.md`.
- Avoid storing credentials, passwords, or session cookies in documentation.
