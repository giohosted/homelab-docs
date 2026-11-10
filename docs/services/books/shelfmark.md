# Shelfmark

**Role:** Ebook and audiobook discovery, downloading, and library import  
**Host:** docker-prod-01 (192.168.30.11)  
**Status:** Active  
**Compose:** `/opt/stacks/books/compose.yaml`  
**Appdata:** `/opt/appdata/shelfmark/`  
**URL:** `https://shelf.giohosted.com`  
**Last Updated:** 2026-11-09

---

## Overview

Shelfmark is the ebook and audiobook request frontend for Homelab v3.

Users search for books, select available sources, and initiate downloads. Shelfmark integrates with qBittorrent for torrent downloads and handles routing completed files to the appropriate library destinations.

The workflow differs by media type:

- **Ebooks:** Shelfmark places completed ebooks into the Calibre-Web-Automated (CWA) ingest directory.
- **Audiobooks:** Shelfmark places completed audiobooks into the Audiobookshelf (ABS) library.
- **Torrent management:** qBittorrent performs downloading and seeding; qBitrr manages seeding policies and automatic torrent cleanup.
- **Direct downloads:** Shelfmark can also retrieve ebooks directly without involving qBittorrent or qBitrr.

Torrent-based imports use hardlinks where configured and supported.

**Important:** Shelfmark does not enforce private-tracker seeding requirements itself. Those policies are configured in qBitrr.

In November 2026, the integration was tested successfully with a direct EPUB download and a public audiobook torrent. The audiobook remained playable after qBitrr removed its torrent.

---

## Deployment

| Field | Value |
|-------|-------|
| Image | `ghcr.io/calibrain/shelfmark:latest` |
| Compose | `/opt/stacks/books/compose.yaml` |
| Appdata | `/opt/appdata/shelfmark/` |
| URL | `https://shelf.giohosted.com` |
| Internal port | `8084` |
| PUID/PGID | `2000:2000` |
| Timezone | `America/Chicago` |
| Docker network | `proxy` |

The image, runtime user, and network configuration are carried forward from the existing deployment documentation. The November 2026 investigation independently verified the relevant container bind mounts and download/import behavior.

---

## Volumes

The following bind mounts were verified using `docker inspect shelfmark` during the November 2026 troubleshooting session.

| Host Path | Container Path | Purpose |
|-----------|---------------|---------|
| `/opt/appdata/shelfmark/config` | `/config` | Shelfmark configuration and database |
| `/data/downloads/books/ebooks/ingest` | `/books` | Ebook destination monitored by CWA |
| `/data/media/books/audiobooks` | `/audiobooks` | Audiobook library destination |
| `/data/downloads/books/audiobooks/downloads` | `/data/downloads/books/audiobooks/downloads` | Audiobook torrent download source |
| `/data/downloads/books/ebooks/downloads` | `/data/downloads/books/ebooks/downloads` | Ebook torrent download source |

### Important Audiobook Mount Correction

The original documentation incorrectly listed this container mount:

```text id="s21a1"
/data/downloads/books/audiobooks/downloads
    → /books/audiobooks/downloads
```

The actual deployed mount is:

```text id="s21a2"
/data/downloads/books/audiobooks/downloads
    → /data/downloads/books/audiobooks/downloads
```

This was confirmed from the running container on November 9, 2026.

The documentation has been updated to match the live configuration. No mount change was required.

### Hardlink Requirements

Torrent download directories must be accessible to Shelfmark and compatible with the paths reported by qBittorrent.

For the current deployment:

- Ebook torrent source: `/data/downloads/books/ebooks/downloads`
- Ebook import destination: `/books`
- Audiobook torrent source: `/data/downloads/books/audiobooks/downloads`
- Audiobook import destination: `/audiobooks`

The source and destination must also reside on the same underlying filesystem for hardlink creation to succeed.

On `docker-prod-01`, the host's `/data` directory is backed by the single Unraid NFS share:

```text id="s21a3"
192.168.30.16:/mnt/user/data
```

This single-share architecture is intentional and supports hardlinks between downloads and media.

Do not split the host's `/data/downloads` and `/data/media` paths into independent NFS mounts without reviewing the hardlink workflow.

See `services/torrent/qbittorrent.md` for the storage architecture.

---

## Network & Access

| Field | Value |
|-------|-------|
| Docker network | `proxy` (external, shared with Traefik) |
| Traefik router | `shelfmark` |
| Traefik middleware | None documented |
| External access | Cloudflare Tunnel |
| Public hostname | `shelf.giohosted.com` |
| Cloudflare Access | Enabled — login required before reaching the app |
| Application authentication | Authentik OIDC |

The access configuration above is retained from the existing deployment documentation; it was not the subject of the November 2026 troubleshooting session.

---

## Authentication

- **Method:** OIDC via Authentik
- **Authentik application:** `Shelfmark`
- **Authentik provider:** `shelfmark`
- **Redirect URI:** `https://shelf.giohosted.com/api/auth/oidc/callback`
- **Discovery URL:** `https://auth.giohosted.com/application/o/shelfmark/.well-known/openid-configuration`
- **Admin group:** `admins`
- **Auto-provision users:** Enabled
- **Local admin:** Retained as break-glass (`admin` user)

The Authentik provider was created during the Homelab v3 deployment rather than restored from the Homelab v2 backup.

**Security:** Keep the local break-glass administrator available, protected by a strong credential, and restricted appropriately. Do not store credentials or tokens in documentation.

---

## Download Client Configuration

| Field | Value |
|-------|-------|
| Torrent client | qBittorrent |
| qBittorrent URL | `http://gluetun:8080/` |
| Ebook category | `ebooks` |
| Audiobook category | `audiobooks` |
| Historically documented download directory | `/downloads` |

**Important:** The qBittorrent URL must use `gluetun`, not `qbittorrent`, because qBittorrent runs inside Gluetun's network namespace.

The `/downloads` setting is retained from the original Shelfmark configuration documentation. The effective torrent save paths are category-specific and must agree with the bind mounts documented above.

### qBitrr Integration

qBitrr manages the following qBittorrent categories:

```toml id="s21a4"
ManagedCategories = ["audiobooks", "ebooks"]
```

The two categories share a default one-hour seeding policy.

MyAnonamouse torrents are intended to receive additional tracker-specific protection.

See `services/torrent/qbitrr.md` for the authoritative Qbitrr configuration.

---

## Download Settings

| Setting | Ebooks | Audiobooks |
|---------|--------|------------|
| Destination | `/books` | `/audiobooks` |
| Host destination | `/data/downloads/books/ebooks/ingest` | `/data/media/books/audiobooks` |
| File organization | Rename only | Rename and organize |
| Naming template | `{Author} - {Title} ({Year})` | `{Author}/{Title}` |
| Hardlink torrents | Enabled | Enabled |
| qBittorrent category | `ebooks` | `audiobooks` |

The organization settings and naming templates are retained from the original deployment documentation.

### Torrent Downloads vs. Direct Downloads

Shelfmark supports more than one download workflow.

**Torrent downloads:**

1. Shelfmark submits a torrent to qBittorrent.
2. qBittorrent downloads it into the corresponding category path.
3. Shelfmark imports the completed media.
4. qBitrr independently manages seeding and torrent cleanup.

**Direct downloads:**

1. Shelfmark downloads the file directly from the selected source.
2. Shelfmark places the file in the configured destination.
3. The destination application processes or indexes the file.

Direct downloads do not create a qBittorrent torrent and therefore do not involve qBitrr's seeding policies.

---

## Ebook Hardlink Workflow

For torrent-based ebook downloads:

1. A user requests an ebook through Shelfmark.
2. Shelfmark submits the torrent to qBittorrent with category `ebooks`.
3. qBittorrent downloads the content to `/data/downloads/books/ebooks/downloads`.
4. Shelfmark hardlinks the completed ebook into `/books`, mapped to `/data/downloads/books/ebooks/ingest`.
5. Calibre-Web-Automated detects the file in its ingest directory.
6. CWA processes the file and imports it into its managed ebook library.
7. qBitrr manages the torrent's remaining seeding period and eventual cleanup.

### Why the Ingest Hardlink Is Intentional

Shelfmark warns against hardlinking into an ingest folder when a downstream application might remove the file.

In this architecture, the ingest directory is a staging location.

CWA can remove the ingest-side directory entry after processing without necessarily removing a separately linked download-side file.

However, the final ebook library copy is managed by CWA. Its retention should be verified independently of the original torrent download.

**Do not assume a completed torrent guarantees that CWA successfully imported the ebook.**

### Verified Direct Ebook Download

During the November 2026 investigation, a direct EPUB download of *The Night She Disappeared* was completed successfully through Shelfmark after its container was restarted.

The resulting ebook appeared in CWA.

This verified the direct-download-to-CWA workflow.

**Outstanding:** A complete torrent-based ebook import and subsequent qBitrr cleanup have not been independently tested during this audit.

---

## Audiobook Hardlink Workflow

For torrent-based audiobook downloads:

1. A user requests an audiobook through Shelfmark.
2. Shelfmark submits the torrent to qBittorrent with category `audiobooks`.
3. qBittorrent downloads the content to `/data/downloads/books/audiobooks/downloads`.
4. Shelfmark hardlinks the completed audiobook into `/audiobooks`, mapped to `/data/media/books/audiobooks`.
5. Audiobookshelf detects the imported audiobook.
6. The audiobook becomes available for playback.
7. qBitrr manages seeding and removes the torrent when its applicable policy permits.
8. The audiobook remains in the library provided the import completed successfully and the library entry is retained.

### November 2026 End-to-End Verification

The public audiobook *Threshing Day* from AudiobookBay was used to test the workflow.

The following actions were confirmed:

- Shelfmark successfully submitted the audiobook torrent.
- qBittorrent completed the download and began seeding.
- Shelfmark imported the audiobook into the ABS library.
- Audiobookshelf successfully played the audiobook while the torrent was seeding.
- qBitrr automatically removed the torrent after the configured one-hour seeding period.
- The imported audiobook remained present and playable in ABS after torrent removal.

**Result:** Public audiobook downloading, importing, seeding cleanup, and library retention were verified end-to-end.

This is the confirmed reference workflow for public audiobook torrents.

It does not establish that the private-tracker-specific MAM rules have been tested.

---

## Seeding Policies

### Overview

Shelfmark does not independently enforce seeding time or ratio requirements.

qBittorrent handles torrent transfers, while qBitrr manages the configured seeding policies.

The previous documentation described ebooks and audiobooks as seeding indefinitely. That is no longer the intended configuration.

### Default Public Torrent Policy

The verified qBitrr configuration is:

```toml id="s21a5"
[qBit.CategorySeeding]
MaxUploadRatio = -1
MaxSeedingTime = 3600
RemoveTorrent = 2
```

This applies to the managed `ebooks` and `audiobooks` categories.

**Intended behavior:**

- Public torrents seed for approximately one hour.
- Automatic cleanup is time-based rather than upload-ratio-based.
- Successfully imported library media should remain available after torrent cleanup.

Public audiobook cleanup has been verified through a real download.

### MyAnonamouse Private-Tracker Policy

MAM requires additional seeding protection.

The following tracker-specific configuration was verified in qBitrr:

```toml id="s21a6"
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

**Policy intent:** MAM torrents must satisfy both the configured 14-day minimum seeding duration and 1.0 upload ratio before being cleared for cleanup.

The configuration includes a one-hour tracker-update buffer to account for reporting delays.

### Verification Status

| Behavior | Status |
|----------|--------|
| Public audiobook download | Verified |
| Public audiobook library import | Verified |
| Public audiobook automatic torrent cleanup | Verified |
| Audiobook playback after cleanup | Verified |
| Direct EPUB import into CWA | Verified |
| MAM tracker matching | Not yet verified with a live torrent |
| MAM 14-day and 1.0-ratio enforcement | Not yet verified end-to-end |
| Torrent-based ebook cleanup | Not yet verified end-to-end |

**Important:** Do not consider MAM compliance validated merely because the configuration parses and qBitrr starts correctly.

A future real-world MAM torrent should be used to verify that the tracker override protects the torrent beyond the public one-hour limit.

Until then, avoid manually removing MAM torrents before confirming their actual tracker obligations are satisfied.

### qBittorrent Global Limits

qBittorrent's former native global 60-minute seeding stop rule was disabled during the November 2026 audit.

This is intentional.

A global client-side stop rule could interfere with private-tracker seeding requirements even if qBitrr has a separate tracker-specific policy.

Do not re-enable it without reviewing the MAM implications.

---

## Connected Services

| Service | URL | Purpose |
|---------|-----|---------|
| qBittorrent | `http://gluetun:8080/` | Torrent downloading and seeding |
| qBitrr | `https://qbitrr.giohosted.com/ui` | Torrent monitoring, recovery, and seeding cleanup |
| Prowlarr | `http://prowlarr:9696/` | Indexer integration |
| Audiobookshelf | `https://audiobooks.giohosted.com` | Audiobook library and playback |
| Calibre-Web-Automated | `https://cwa.giohosted.com/` | Ebook ingest and library management |
| Authentik | `https://auth.giohosted.com` | OIDC authentication |

### Integration Responsibilities

**Shelfmark:** Discovery, requests, downloading coordination, file routing, and import.

**qBittorrent:** Torrent transfers and seeding.

**qBitrr:** Torrent monitoring, recovery, and seeding policies.

**CWA:** Ebook ingest and library management.

**Audiobookshelf:** Audiobook indexing and playback.

These responsibilities should remain distinct when troubleshooting.

For example, a qBitrr cleanup event does not establish that Shelfmark successfully imported a file, and an import failure does not necessarily indicate a qBittorrent or qBitrr problem.

---

## Backup & Restore

The original deployment documentation records:

- **Appdata backup destination:** `/mnt/user/backups/` through the Phase 5 rsync workflow.
- **Homelab v2 backup location:** `/mnt/user/backups/docker-v2/appdata/books-stack/shelfmark/`
- **Persistent configuration:** `/opt/appdata/shelfmark/config/`

These are documented backup arrangements, not independently verified backup results from the November 2026 audit.

### Restore Procedure

The original migration procedure was:

1. Restore the Shelfmark `config/` directory into `/opt/appdata/shelfmark/`.
2. Restore the expected ownership and permissions for UID/GID `2000:2000`.
3. Confirm the current Docker Compose bind mounts.
4. Correct any stale qBittorrent and Prowlarr URLs from the previous deployment.
5. Start the Shelfmark container.
6. Verify login, search, download, and import functionality.

Do not assume that restored configuration files contain the current container paths or connection URLs.

### Post-Restore Configuration Fixes

During migration from Homelab v2, these files required corrections before the application could reliably contact its dependencies.

| File | Field | Old Value | New Value |
|------|-------|-----------|-----------|
| `config/plugins/prowlarr_clients.json` | `QBITTORRENT_URL` | `http://192.168.20.10:8080/` | `http://gluetun:8080/` |
| `config/plugins/prowlarr_config.json` | `PROWLARR_URL` | `http://192.168.20.10:9696/` | `http://prowlarr:9696/` |

The original deployment notes report that the settings interface could time out while trying to reach the stale v2 endpoints.

These corrections are historical restore requirements, not evidence of an ongoing connection problem.

---

## Troubleshooting

### Downloads Fail or Remain Unavailable

First identify the download type.

**Direct ebook download:**

- Check Shelfmark's download status and errors.
- Verify access to the `/books` destination.
- Check whether CWA received the imported file.

**Torrent download:**

- Check whether qBittorrent received the torrent.
- Confirm the correct `ebooks` or `audiobooks` category.
- Verify the download save path.
- Check whether the torrent completed.
- Verify that Shelfmark imported the completed file.
- Check qBitrr cleanup history if the torrent is no longer present.

Avoid treating every failed import as a seeding-policy problem.

### Stale NFS File Handles — November 2026 Incident

**Incident date:** November 9, 2026  
**Affected service:** Shelfmark on `docker-prod-01`  
**Status:** Resolved; recurrence not observed during the verification period

#### Symptoms

Shelfmark reported download/import errors for ebook and audiobook requests.

The affected requests included:

- *The Night She Disappeared*
- *Threshing Day*

Direct ebook retrieval also failed during the incident.

#### Investigation

Shelfmark's bind mounts were inspected with:

```bash id="s21a7"
docker inspect shelfmark
```

The expected mount mappings were present.

Inside the running Shelfmark container, directory metadata checks succeeded, but attempts to list entries returned:

```text id="s21a8"
Stale file handle
```

Affected container paths included:

```text id="s21a9"
/books
/audiobooks
/data/downloads/books/ebooks/downloads
/data/downloads/books/audiobooks/downloads
```

Additional observations:

- The Docker host's `/data` NFS mount was accessible.
- The host could list the affected directories.
- Other containers, including qBittorrent, CWA, and ABS, could access their corresponding directories.
- The problem was therefore isolated to Shelfmark's view of the mounted directories during the investigation.

**Likely explanation:** Shelfmark retained stale NFS directory handles, potentially following a prior mount interruption or remount.

The exact triggering event was not established.

#### Resolution

After confirming the host mount and other containers were operating normally, Shelfmark was restarted:

```bash id="s21b0"
docker restart shelfmark
```

After the restart:

- All four affected directories could be listed successfully from inside Shelfmark.
- Direct EPUB download and CWA import succeeded.
- A new audiobook torrent downloaded successfully.
- ABS imported and played the audiobook.
- qBitrr subsequently removed the public audiobook torrent.
- The imported audiobook remained playable.

**Conclusion:** Restarting Shelfmark resolved the observed incident without changing the NFS architecture, bind mounts, qBittorrent configuration, or seeding policies.

#### If the Problem Recurs

Follow a read-only investigation first.

**1. Check the Shelfmark container and logs:**

```bash id="s21b1"
docker ps --filter name=shelfmark
docker logs shelfmark --tail 100
```

**2. Inspect the current bind mounts:**

```bash id="s21b2"
docker inspect shelfmark \
  --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

**3. Check the host's NFS mount:**

```bash id="s21b3"
findmnt /data
```

**4. Compare host and container directory access:**

On the host:

```bash id="s21b4"
ls -A /data/downloads/books/audiobooks/downloads
```

Inside Shelfmark:

```bash id="s21b5"
docker exec shelfmark \
  ls -A /data/downloads/books/audiobooks/downloads
```

If the host and other containers can access the path while Shelfmark alone reports `Stale file handle`, consider restarting only Shelfmark after evaluating active downloads.

If multiple containers or the host show similar errors, investigate the NFS server, export, mount state, and underlying storage before restarting services.

**Do not automatically remount `/data`, restart the entire Docker stack, or change NFS protocol versions for an isolated Shelfmark incident.**

### Audiobook Downloaded but Missing From ABS

Check:

1. Whether qBittorrent completed the torrent.
2. Whether Shelfmark imported the audiobook.
3. Whether `/audiobooks` is accessible inside Shelfmark.
4. Whether ABS has the corresponding library directory configured.
5. Whether the file is present under `/data/media/books/audiobooks`.
6. Whether ABS has scanned or indexed the imported content.

Do not assume qBitrr removed the file merely because the torrent disappeared.

### Ebook Downloaded but Missing From CWA

Check whether the ebook entered the ingest directory:

```text id="s21b6"
/data/downloads/books/ebooks/ingest
```

Then inspect CWA's ingest and import activity.

A successful Shelfmark download does not necessarily mean CWA successfully processed the ebook.

### Torrent Removed Before Import

If a torrent disappears before Shelfmark imports it:

- Check the torrent category and tracker.
- Review qBitrr's removal decision.
- Check whether the intended import path was accessible.
- Inspect Shelfmark's logs around the download completion.
- Verify whether the library file already exists.

A one-hour seeding policy assumes the downstream application has had an opportunity to import completed content. Import failures must be investigated rather than assuming cleanup is harmless.

### MAM Torrent Stops or Disappears Prematurely

Verify:

- qBittorrent's global native seeding limits remain disabled.
- The torrent is in a category managed by qBitrr.
- The tracker URL matches the MAM tracker configuration.
- `HitAndRunMode = "and"` is configured.
- The 14-day and ratio requirements are represented correctly.
- The actual qBitrr logs explain any cleanup decision.

**MAM-specific end-to-end behavior remains unverified as of November 9, 2026.**

Do not manually delete a private-tracker torrent without confirming its tracker obligations.

### Shelfmark Settings Page Loads Slowly

The original deployment documented that the settings page could load slowly.

During the v2-to-v3 migration, stale dependency URLs also caused timeouts.

If the settings page becomes unresponsive, distinguish between ordinary UI slowness and failed connections to qBittorrent or Prowlarr before editing configuration files.

---

## November 2026 Operational Review

### Verified

- Shelfmark's five Docker bind mounts were inspected.
- The actual audiobook download mount path was confirmed.
- The host's shared `/data` mount was accessible during troubleshooting.
- Stale file handles were observed inside Shelfmark.
- Restarting Shelfmark restored directory access.
- Direct EPUB download and CWA import succeeded.
- Public audiobook torrent download and ABS import succeeded.
- Audiobook playback worked before and after torrent cleanup.
- qBitrr automatically removed the tested public audiobook torrent after the configured seeding period.

### Configured but Awaiting Verification

- MAM torrents remaining protected beyond the normal one-hour public-torrent cleanup period.
- MAM cleanup only after both minimum seeding duration and ratio obligations are satisfied.
- Complete torrent-based ebook import and subsequent cleanup.
- Long-term absence of recurring stale NFS file handles.

### Operational Decisions

- Preserve the shared single `/data` NFS architecture.
- Do not introduce additional automated restart, buffering, or import-retry mechanisms based on one resolved incident.
- Investigate recurring failures if they appear.
- Keep seeding-policy management centralized in qBitrr.
- Keep completed library imports separate from torrent cleanup responsibilities.

---

## Notes

- Shelfmark uses Authentik OIDC and retains a local break-glass administrator.
- The Shelfmark qBittorrent endpoint is `http://gluetun:8080/`, not `http://qbittorrent:8080/`.
- The ebook destination is the CWA ingest directory, not the final managed ebook library.
- The audiobook destination is the ABS library directory.
- Ebook and audiobook torrent download paths are mounted inside Shelfmark using their matching `/data/downloads/books/...` paths.
- The old audiobook container path `/books/audiobooks/downloads` is incorrect for the current deployment.
- Public torrent seeding is no longer indefinite; qBitrr manages one-hour cleanup.
- MAM tracker-specific seeding is configured for a 14-day minimum and 1.0 ratio using AND logic, but live verification remains outstanding.
- qBittorrent's global 60-minute seeding stop rule was disabled to avoid conflicts with MAM.
- The November 2026 stale-file-handle incident was resolved by restarting Shelfmark. Its underlying trigger remains unknown.
- See `services/torrent/qbitrr.md` for the authoritative seeding and tracker settings.
- See `services/torrent/qbittorrent.md` for VPN connectivity, torrent categories, storage layout, and hardlink requirements.
