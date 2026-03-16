# Shelfmark

**Host:** docker-prod-01 (192.168.30.11)  
**Status:** Active  
**Last Updated:** 2026-03-16

---

## Overview

Shelfmark is the ebook and audiobook request frontend. Users search for and request books, Shelfmark sends the torrent to qBittorrent, and once downloaded it handles the file routing — hardlinking audiobooks directly into the ABS library, and hardlinking ebooks into the CWA ingest folder.

---

## Deployment

| Field | Value |
|-------|-------|
| **Image** | `ghcr.io/calibrain/shelfmark:latest` |
| **Compose** | `/opt/stacks/books/compose.yaml` |
| **Appdata** | `/opt/appdata/shelfmark/` |
| **URL** | `https://shelf.giohosted.com` |
| **Internal port** | 8084 |
| **PUID/PGID** | 2000:2000 |
| **TZ** | America/Chicago |

---

## Volumes

| Host Path | Container Path | Purpose |
|-----------|---------------|---------|
| `/opt/appdata/shelfmark/config` | `/config` | Shelfmark config and database |
| `/data/downloads/books/ebooks/ingest` | `/books` | Ebook destination — Shelfmark hardlinks completed ebooks here |
| `/data/media/books/audiobooks` | `/audiobooks` | Audiobook destination — Shelfmark hardlinks completed audiobooks here |
| `/data/downloads/books/audiobooks/downloads` | `/books/audiobooks/downloads` | Audiobook qBit download folder |
| `/data/downloads/books/ebooks/downloads` | `/data/downloads/books/ebooks/downloads` | Ebook qBit download folder — must match client path exactly for hardlinks |

> **Critical:** The ebook downloads volume must be mounted at the exact same path as qBittorrent's save path (`/data/downloads/books/ebooks/downloads`). Per Shelfmark docs, source and destination must be visible at their exact client-side paths for hardlinks to work.

---

## Network & Access

| Field | Value |
|-------|-------|
| **Docker network** | `proxy` (external, shared with Traefik) |
| **Traefik router** | `shelfmark` |
| **Middleware** | None — externally exposed |
| **External access** | Yes — Cloudflare Tunnel at `shelf.giohosted.com` |
| **Cloudflare Access** | Yes — login required before reaching the app |

---

## Authentication

- **Method:** OIDC via Authentik
- **Authentik application:** `Shelfmark`
- **Authentik provider:** `shelfmark` — created fresh in v3 (not in v2 backup)
- **Redirect URI:** `https://shelf.giohosted.com/api/auth/oidc/callback`
- **Discovery URL:** `https://auth.giohosted.com/application/o/shelfmark/.well-known/openid-configuration`
- **Admin group:** `admins`
- **Auto-provision users:** Enabled
- **Local admin:** Retained as break-glass (`admin` user) — do not delete

---

## Download Client Configuration

| Field | Value |
|-------|-------|
| **Torrent client** | qBittorrent |
| **qBittorrent URL** | `http://gluetun:8080/` |
| **Book category** | `ebooks` |
| **Audiobook category** | `audiobooks` |
| **Download directory** | `/downloads` |

> **Note:** qBittorrent URL must point to `gluetun`, not `qbittorrent` — qBit runs inside Gluetun's network namespace and the port is exposed through Gluetun.

---

## Download Settings

| Setting | Ebooks | Audiobooks |
|---------|--------|------------|
| **Destination** | `/books` (= ingest folder) | `/audiobooks` (= ABS library) |
| **File organization** | Rename only | Rename and organize |
| **Naming template** | `{Author} - {Title} ({Year})` | `{Author}/{Title}` |
| **Hardlink torrents** | ✅ Enabled | ✅ Enabled |

---

## Ebook Hardlink Workflow

1. User requests ebook in Shelfmark
2. Shelfmark sends torrent to qBittorrent with category `ebooks`
3. qBittorrent downloads to `/data/downloads/books/ebooks/downloads` — seeds permanently from here
4. Shelfmark hardlinks completed file to `/books` (= `/data/downloads/books/ebooks/ingest`)
5. CWA detects file in ingest, imports to `/data/media/books/ebooks`, deletes the hardlink
6. Original in `/data/downloads/books/ebooks/downloads` is untouched — seeding continues

> **Why hardlinks are safe here despite Shelfmark's warning:** Shelfmark warns against hardlinking when the destination is a library ingest folder because CWA will delete the file after import. Here that is intentional — CWA deletes the hardlink, not the original. The original in downloads is untouched. This is the correct pattern.

---

## Audiobook Hardlink Workflow

1. User requests audiobook in Shelfmark
2. Shelfmark sends torrent to qBittorrent with category `audiobooks`
3. qBittorrent downloads to `/data/downloads/books/audiobooks/downloads` — seeds permanently from here
4. Shelfmark hardlinks completed file to `/audiobooks` (= `/data/media/books/audiobooks`)
5. ABS detects new file in library folder — book is available to stream
6. Original in downloads is untouched — seeding continues

---

## Connected Services

| Service | URL | Purpose |
|---------|-----|---------|
| qBittorrent | `http://gluetun:8080/` | Torrent download client |
| Prowlarr | `http://prowlarr:9696/` | Indexer for book searches |
| Audiobookshelf | `https://audiobooks.giohosted.com` | Audiobook library — configured in Shelfmark settings |
| Calibre-Web-Automated | `https://cwa.giohosted.com/` | Ebook library — configured in Shelfmark settings |

---

## Backup & Restore

- **Appdata backed up to:** `/mnt/user/backups/` (Phase 5 rsync script)
- **v2 backup location:** `/mnt/user/backups/docker-v2/appdata/books-stack/shelfmark/`
- **Restore procedure:** Copy `config/` to `/opt/appdata/shelfmark/`, set ownership to `2000:2000`, start container, update qBit URL and Prowlarr URL in config files directly (do not rely on UI if old URLs cause timeout)

### Post-Restore Config Fixes Required

After restoring from v2 backup, these files must be updated manually before starting the container — the UI will time out trying to reach stale v2 URLs:

| File | Field | Old Value | New Value |
|------|-------|-----------|-----------|
| `config/plugins/prowlarr_clients.json` | `QBITTORRENT_URL` | `http://192.168.20.10:8080/` | `http://gluetun:8080/` |
| `config/plugins/prowlarr_config.json` | `PROWLARR_URL` | `http://192.168.20.10:9696/` | `http://prowlarr:9696/` |

---

## Notes

- Shelfmark settings page can be slow to load — known app behavior, not a configuration issue
- MAM-specific seeding rules (14-day minimum, tracker-scoped) are managed by qBitrr in Phase 5 — not configured in Shelfmark
- Shelfmark does not enforce seeding rules itself — it delegates all torrent management to qBittorrent and qBitrr