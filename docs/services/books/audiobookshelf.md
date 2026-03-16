# Audiobookshelf

**Host:** docker-prod-01 (192.168.30.11)  
**Status:** Active  
**Last Updated:** 2026-03-16

---

## Overview

Audiobookshelf is the audiobook server. Users request audiobooks via Shelfmark, which sends them to qBittorrent. Once downloaded, Shelfmark hardlinks the completed file directly into the ABS library folder. ABS picks it up automatically.

---

## Deployment

| Field | Value |
|-------|-------|
| **Image** | `ghcr.io/advplyr/audiobookshelf:latest` |
| **Compose** | `/opt/stacks/books/compose.yaml` |
| **Appdata** | `/opt/appdata/audiobookshelf/` |
| **URL** | `https://audiobooks.giohosted.com` |
| **Internal port** | 80 |
| **PUID/PGID** | 2000:2000 |
| **TZ** | America/Chicago |

---

## Volumes

| Host Path | Container Path | Purpose |
|-----------|---------------|---------|
| `/data/media/books/audiobooks` | `/audiobooks` | Audiobook library |
| `/opt/appdata/audiobookshelf/config` | `/config` | ABS config and database |
| `/opt/appdata/audiobookshelf/metadata` | `/metadata` | Cover art, metadata cache |

---

## Network & Access

| Field | Value |
|-------|-------|
| **Docker network** | `proxy` (external, shared with Traefik) |
| **Traefik router** | `audiobookshelf` |
| **Middleware** | None — externally exposed |
| **External access** | Yes — Cloudflare Tunnel at `audiobooks.giohosted.com` |
| **Cloudflare Access** | Yes — login required before reaching the app |

---

## Authentication

- **Method:** OIDC via Authentik
- **Authentik application:** `Audiobookshelf`
- **Authentik provider:** Carried forward from v2 backup — all users and sessions intact on restore
- **Admin group:** `admins`
- **Auto-provision users:** Enabled
- **Local admin:** Retained as break-glass — do not delete

---

## Library Configuration

| Library | Path |
|---------|------|
| Audiobooks | `/audiobooks` |

> **Note:** After restoring from v2 backup, the library path was still pointing to the v2 path (`/books/audiobooks`). Had to manually repoint to `/audiobooks` in ABS → Settings → Libraries.

---

## Audiobook Download Workflow

1. User requests audiobook in Shelfmark
2. Shelfmark sends torrent to qBittorrent with category `audiobooks`
3. qBittorrent downloads to `/data/downloads/books/audiobooks/downloads` — seeds permanently from here
4. Shelfmark hardlinks completed file to `/audiobooks` (= `/data/media/books/audiobooks`)
5. ABS detects new file in library folder — book is available to stream
6. Original torrent in downloads folder is untouched — seeding continues

---

## Backup & Restore

- **Appdata backed up to:** `/mnt/user/backups/` (Phase 5 rsync script)
- **v2 backup location:** `/mnt/user/backups/docker-v2/appdata/books-stack/audiobookshelf/`
- **Restore procedure:** Copy `config/` and `metadata/` to `/opt/appdata/audiobookshelf/`, set ownership to `2000:2000`, start container, repoint library path if needed

---

## Notes

- No `local-only@docker` middleware — ABS is externally accessible by design, protected by Cloudflare Access + Authentik OIDC
- Podcasts not used — audiobooks only