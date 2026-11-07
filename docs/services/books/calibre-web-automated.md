# Calibre-Web-Automated (CWA)

**Host:** docker-prod-01 (192.168.30.11)  
**Status:** Active  
**Last Updated:** 2026-11-06

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
| **Management** | Komodo — `books` stack |

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
| **Backend protocol** | HTTP |
| **Backend port** | 8083 |

Traefik terminates HTTPS and forwards requests to CWA over HTTP on port 8083.

CWA redirects unauthenticated requests to its login page at `https://cwa.giohosted.com/login`.

Traefik uses Docker container health status when determining whether a backend is available. If CWA becomes unhealthy, Traefik may remove it from routing even while the container remains running.

---

## Authentication

- **Method:** Authentik OIDC — carried forward from v2 backup, working on restore
- **Local admin:** Retained as break-glass — do not delete

---

## Docker Health Check

### Configuration

CWA uses a custom Docker Compose health-check override.

The override was implemented on 2026-11-06 to correct an issue with the container image's built-in health check.

**Location:** `/opt/stacks/books/compose.yaml`  
**Service:** `calibre-web-automated`

```yaml
    healthcheck:
      test:
        - CMD-SHELL
        - curl -fsS -o /dev/null --connect-timeout 2 --max-time 5 http://127.0.0.1:8083/
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
```

### Health Check Behavior

The health check:

1. Sends an HTTP request directly to CWA on `127.0.0.1:8083`.
2. Accepts successful HTTP responses, including the expected `302 Found` redirect.
3. Does not follow redirects outside the container.
4. Does not depend on DNS, Traefik, or the public application hostname.
5. Runs every 30 seconds, with three consecutive failures required to mark the container unhealthy.
6. Allows a 60-second startup grace period.

**Important:** Do not add `-L` to the curl command. Following redirects caused the original health-check failure.

The application normally responds to unauthenticated requests to `/` with HTTP `302 Found`, redirecting the user to its login page. This is expected behavior and should be treated as a successful health-check response.

### Verification

Check container health:

```bash
docker inspect calibre-web-automated \
  --format 'Status={{.State.Status}} Health={{.State.Health.Status}}'
```

Expected:

```text
Status=running Health=healthy
```

Test the application directly:

```bash
docker exec calibre-web-automated sh -c '
curl -fsS --connect-timeout 2 --max-time 5 \
  -o /dev/null \
  -w "HTTP Status: %{http_code}\n" \
  http://127.0.0.1:8083/

echo "Exit Code: $?"
'
```

Expected:

```text
HTTP Status: 302
Exit Code: 0
```

Test Traefik routing:

```bash
curl -sS -o /dev/null \
  -w 'HTTP Status: %{http_code}\n' \
  https://cwa.giohosted.com/
```

Expected:

```text
HTTP Status: 302
```

A successful end-to-end verification should also include opening the application in a browser and confirming that the library is accessible.

---

## Ebook Ingest Workflow

1. User requests ebook in Shelfmark.
2. Shelfmark sends torrent to qBittorrent with category `ebooks`.
3. qBittorrent downloads to `/data/downloads/books/ebooks/downloads` — seeds permanently from here.
4. Shelfmark hardlinks completed file to `/data/downloads/books/ebooks/ingest` (= `/books` inside Shelfmark container).
5. CWA detects file in `/cwa-book-ingest`, imports it to `/calibre-library` (= `/data/media/books/ebooks`), deletes the hardlink from ingest.
6. Original file in `/data/downloads/books/ebooks/downloads` is untouched — qBittorrent continues seeding.

> **Why this works:** CWA deletes the hardlink in ingest, not the original file in downloads. The two files share the same inode — deleting one leaves the other intact. qBit's copy survives.

> **NETWORK_SHARE_MODE=true is required.** Without it, CWA's file watcher does not reliably detect new files on NFS-mounted paths.

---

## Troubleshooting

### Incident: HTTP 404 and Unhealthy Container — 2026-11-06

**Status:** Resolved and verified

#### Symptoms

- Accessing `https://cwa.giohosted.com` returned `404 page not found`.
- Komodo showed the `books` stack and CWA container running.
- Docker reported `Status=running Health=unhealthy`.
- CWA logs repeatedly showed invalid HTTP method and invalid HTTP version errors approximately every 30 seconds.
- Application data and ebook library remained intact.

#### Root Cause

The CWA container image included a built-in health check that attempted HTTP first, then HTTPS:

```bash
curl -fsL --connect-timeout 1 --max-time 2 \
  http://localhost:${CWA_PORT_OVERRIDE:-8083}/ ||
curl -fsL -k --connect-timeout 1 --max-time 2 \
  https://localhost:${CWA_PORT_OVERRIDE:-8083}/ ||
exit 1
```

The `-L` option caused curl to follow HTTP redirects.

CWA responded to the initial HTTP request with `302 Found`, redirecting to `cwa.giohosted.com/login`.

The redirect subsequently reached Traefik, which returned HTTP 404 because CWA was already considered unhealthy.

This created a circular dependency:

- CWA's health check depended on successful routing through Traefik.
- Traefik depended on CWA having a healthy Docker status.
- The health check failed, leaving the container unhealthy and unavailable through Traefik.

The health check's fallback also attempted HTTPS directly against CWA's HTTP listener on port 8083.

This generated TLS handshake errors, including:

```text
Invalid HTTP method: '\x16\x03\x01...'
```

The errors indicated an HTTPS/TLS connection being sent to an HTTP-only service.

#### Resolution

1. Confirmed CWA was running and listening on port 8083.
2. Tested the application directly and confirmed HTTP 302 responses.
3. Verified that requests following redirects reached Traefik and returned HTTP 404.
4. Added a custom health-check override to `/opt/stacks/books/compose.yaml` without redirect following.
5. Validated the Compose configuration.
6. Recreated only the `calibre-web-automated` service.
7. Confirmed Docker health changed to `healthy`.
8. Confirmed Traefik returned HTTP 302.
9. Confirmed application access worked in a browser.

#### Validation Commands

Validate Compose:

```bash
cd /opt/stacks/books

docker compose --env-file .env -f compose.yaml config --quiet
echo "Validation exit code: $?"
```

Apply changes to CWA only:

```bash
docker compose --env-file .env -f compose.yaml \
  up -d --no-deps calibre-web-automated
```

Check container health and application accessibility using the commands in the Docker Health Check section.

#### Final Verified State

| Component | Result |
|-----------|--------|
| CWA container | Running |
| Docker health | Healthy |
| Direct application request | HTTP 302 |
| Health-check curl | Exit code 0 |
| Traefik routing | HTTP 302 |
| Browser access | Working |
| Ebook library | Preserved |
| Other books containers | No redeployment required |

**Operational lesson:** Container health checks should verify local application readiness without requiring external DNS, reverse proxies, or application redirects.

---

## Backup & Restore

- **Appdata backed up to:** `/mnt/user/backups/` (Phase 5 rsync script)
- **v2 backup location:** `/mnt/user/backups/docker-v2/appdata/books-stack/cwa/`
- **Restore procedure:** Copy `config/` and `plugins/` to `/opt/appdata/calibre-web-automated/`, set ownership to `2000:2000`, start container.

The custom health-check override is stored in the Docker Compose file, not inside CWA's application configuration.

Ensure the Compose file retains the override when restoring the stack or rebuilding the Docker host.

---

## Notes

- CWA is internal only — no reason to expose the library manager externally.
- `HARDCOVER_TOKEN` must never be committed to git — lives in `/opt/stacks/books/.env` which is gitignored.
- The Calibre library at `/data/media/books/ebooks` is on the NFS-mounted `/data` share — same filesystem as downloads, hardlinks work correctly.
- CWA uses HTTP internally; HTTPS is terminated by Traefik.
- The custom health check must not follow redirects.
- Future image updates should retain the health-check override defined in Compose.
- Changes to the Compose file should be validated before redeployment.
- Routine CWA redeployment does not require restarting Shelfmark or Audiobookshelf.