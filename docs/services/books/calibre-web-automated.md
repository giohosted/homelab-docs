# Calibre-Web-Automated (CWA)

**Host:** docker-prod-01 (192.168.30.11)  
**Status:** Active  
**Last Updated:** 2026-03-16

---

## Overview

Calibre-Web-Automated is the ebook library manager. It watches an ingest folder for new ebook files, automatically imports them into the Calibre library, and deletes the ingest copy after import. The original download file in qBittorrent's download folder is untouched — seeding continues uninterrupted.

---

## Deployment

| Field | Value |
|-------|-------|
| **Image** | `crocodilestick/calibre-web-automated:latest` |
| **Compose** | `/opt/stacks/books/compose.yaml` |
| **Appdata** | `/opt/appdata/calibre-web-automated/` |
| **URL** | `https://cwa.giohosted.com` |
| **Internal port** | 8083 |
| **PUID/PGID** | 2000:2000 |
| **TZ** | America/Chicago |

---

## Volumes

| Host Path | Container Path | Purpose |
|-----------|---------------|---------|
| `/opt/appdata/calibre-web-automated/config` | `/config` | CWA config and Calibre database |
| `/opt/appdata/calibre-web-automated/plugins` | `/config/.config/calibre/plugins` | Calibre plugins |
| `/data/downloads/books/ebooks/ingest` | `/cwa-book-ingest` | Ingest watch folder — CWA imports from here |
| `/data/media/books/ebooks` | `/calibre-library` | Calibre library — imported books land here |

---

## Environment Variables

| Variable | Value | Notes |
|----------|-------|-------|
| `NETWORK_SHARE_MODE` | `true` | Required for NFS-mounted library path |
| `CWA_PORT_OVERRIDE` | `8083` | Overrides default port |
| `HARDCOVER_TOKEN` | In `.env` | Hardcover API token — gitignored |

---

## Network & Access

| Field | Value |
|-------|-------|
| **Docker network** | `proxy` (external, shared with Traefik) |
| **Traefik router** | `cwa` |
| **Middleware** | `local-only@docker` — internal access only |
| **External access** | No |

---

## Authentication

- **Method:** Authentik OIDC — carried forward from v2 backup, working on restore
- **Local admin:** Retained as break-glass — do not delete

---

## Ebook Ingest Workflow

1. User requests ebook in Shelfmark
2. Shelfmark sends torrent to qBittorrent with category `ebooks`
3. qBittorrent downloads to `/data/downloads/books/ebooks/downloads` — seeds permanently from here
4. Shelfmark hardlinks completed file to `/data/downloads/books/ebooks/ingest` (= `/books` inside Shelfmark container)
5. CWA detects file in `/cwa-book-ingest`, imports it to `/calibre-library` (= `/data/media/books/ebooks`), deletes the hardlink from ingest
6. Original file in `/data/downloads/books/ebooks/downloads` is untouched — qBittorrent continues seeding

> **Why this works:** CWA deletes the hardlink in ingest, not the original file in downloads. The two files share the same inode — deleting one leaves the other intact. qBit's copy survives.

> **NETWORK_SHARE_MODE=true is required.** Without it, CWA's file watcher does not reliably detect new files on NFS-mounted paths.

---

## Backup & Restore

- **Appdata backed up to:** `/mnt/user/backups/` (Phase 5 rsync script)
- **v2 backup location:** `/mnt/user/backups/docker-v2/appdata/books-stack/cwa/`
- **Restore procedure:** Copy `config/` and `plugins/` to `/opt/appdata/calibre-web-automated/`, set ownership to `2000:2000`, start container

---

## Notes

- CWA is internal only — no reason to expose the library manager externally
- `HARDCOVER_TOKEN` must never be committed to git — lives in `/opt/stacks/books/.env` which is gitignored
- The Calibre library at `/data/media/books/ebooks` is on the NFS-mounted `/data` share — same filesystem as downloads, hardlinks work correctly