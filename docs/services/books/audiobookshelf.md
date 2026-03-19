# Audiobookshelf

**Host:** docker-prod-01 (192.168.30.11)  
**Status:** Active  
**Last Updated:** 2026-03-18

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
| **Cloudflare Access** | Yes — browser users require Authentik login; mobile app uses service token bypass (see below) |

---

## Authentication

- **Method:** OIDC via Authentik
- **Authentik application:** `Audiobookshelf`
- **Authentik provider:** Carried forward from v2 backup — all users and sessions intact on restore
- **Admin group:** `admins`
- **Auto-provision users:** Enabled
- **Local admin:** Retained as break-glass — do not delete

### Authentik Redirect URIs

The following redirect URIs are registered on the Authentik provider:

- `https://audiobooks.giohosted.com/audiobookshelf/auth/openid/callback` (strict)
- `https://audiobooks.giohosted.com/audiobookshelf/auth/openid/mobile-redirect` (strict)
- `audiobooth://oauth` (strict) — required for iOS/Android mobile app OIDC login

### ABS Server Redirect URIs

`audiobooth://oauth` must also be added in ABS itself under **Settings → Authentication → OpenID Connect → Mobile Redirect URIs**. Both Authentik and ABS must have this URI or mobile SSO login will fail with a redirect error.

---

## Mobile App External Access (CF Access Service Token)

The ABS mobile app cannot complete Cloudflare Access's browser-based auth challenge, so a CF Access service token is used to bypass the challenge for mobile app requests.

**CF Access application:** `Audiobookshelf` (`audiobooks.giohosted.com`)  
**Service token name:** `abs-mobile`  
**Token location:** Password manager (Client ID + Client Secret — secret shown once at creation)

### CF Access Policy Order

| Order | Policy | Action |
|-------|--------|--------|
| 1 | `abs-mobile-token` | SERVICE AUTH — service token match |
| 2 | `allow-users-authentik` | ALLOW — Authentik login |
| 3 | `block-all-others` | BLOCK |

Service Auth policies are always evaluated first by CF Access regardless of order in the UI.

### Mobile App Configuration

In the ABS iOS/Android app → server connection → **Custom Headers**, add:

| Header Name | Value |
|-------------|-------|
| `CF-Access-Client-Id` | `<client-id from abs-mobile token>` |
| `CF-Access-Client-Secret` | `<client-secret from abs-mobile token>` |

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