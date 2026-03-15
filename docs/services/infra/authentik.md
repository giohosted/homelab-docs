# Authentik

**Role:** Identity Provider (IdP) — SSO and OIDC for all internal services  
**Host:** auth-prod-01 (192.168.30.13)  
**Version:** 2025.12.3  
**Compose:** `/opt/stacks/authentik/compose.yaml`  
**Appdata:** `/opt/appdata/authentik/`  
**URL:** https://auth.giohosted.com  
**Last Updated:** 2026-03-15

---

## Overview

Authentik is the single Identity Provider for homelab-v3. It handles SSO via OIDC for all services that support it. Carried forward from v2 with full config restored from backup — all providers, applications, users, and policies intact.

Authentik runs on a dedicated VM (`auth-prod-01`) rather than as containers on docker-prod-01. This is intentional — LXC support was officially removed from Proxmox community scripts in May 2025, and isolating Authentik on its own VM prevents resource contention with the media stack and other containers.

---

## VM — auth-prod-01

| Component | Detail |
|-----------|--------|
| **VM ID** | 105 |
| **Host** | pve-prod-01 |
| **OS** | Ubuntu Server 24.04 LTS |
| **CPU** | 2 cores (x86-64-v3) |
| **RAM** | 4GB |
| **Disk** | 32GB (local-zfs, no cache) |
| **System** | q35, OVMF UEFI, VirtIO SCSI |
| **IP** | 192.168.30.13/24 |
| **Gateway** | 192.168.30.1 |
| **DNS** | 192.168.30.10 (dns-prod-01) |
| **VLAN** | 30 — Services |
| **Docker** | Installed via official script |
| **QEMU Agent** | Installed and running |

### SSH Access
```bash
ssh gio@192.168.30.13
```

### Git
- homelab-infra repo cloned at `/opt/stacks`
- Credentials cached via `git config --global credential.helper store`
- Always run `git pull` before making changes — see `runbooks/git-workflow.md`

---

## Containers

| Container | Image | Role |
|-----------|-------|------|
| authentik-server | ghcr.io/goauthentik/server:2025.12.3 | Main application server |
| authentik-worker | ghcr.io/goauthentik/server:2025.12.3 | Background task worker |
| authentik-postgres | postgres:16-alpine | Database |
| authentik-redis | redis:alpine | Cache / session store |

All containers are on an internal Docker network (`authentik-internal`) — not exposed to the proxy network. Only the server exposes port 9000 to the host.

---

## Network & Routing

Authentik is on a dedicated VM — Traefik on docker-prod-01 cannot discover it via the Docker socket. Routing is handled via Traefik's file provider:

- **Traefik static route:** `auth.giohosted.com` → `http://192.168.30.13:9000`
- **Config file:** `/opt/appdata/traefik/config/authentik.yml` on docker-prod-01
- **AdGuard DNS rewrite:** `auth.giohosted.com` → `192.168.30.11` (Traefik IP)

Traffic path:
```
Browser → AdGuard (192.168.30.10) → 192.168.30.11 (Traefik) → 192.168.30.13:9000 (authentik-server)
```

---

## Configuration

### Environment Variables (.env)

| Variable | Notes |
|----------|-------|
| `PG_PASS` | Postgres password — carried forward from v2, must match database |
| `PG_USER` | `authentik` |
| `PG_DB` | `authentik` |
| `AUTHENTIK_SECRET_KEY` | Carried forward from v2 — changing this invalidates all sessions |
| `AUTHENTIK_TAG` | `2025.12.3` |
| `AUTHENTIK_BOOTSTRAP_EMAIL` | Used on first boot only |
| `AUTHENTIK_BOOTSTRAP_PASSWORD` | Used on first boot only |

> **Critical:** `PG_PASS` and `AUTHENTIK_SECRET_KEY` must match the values used when the database was originally created. Changing either will break the existing database.

### Volumes

| Host Path | Container Path | Notes |
|-----------|---------------|-------|
| `/opt/appdata/authentik/postgres` | `/var/lib/postgresql/data` | Database files |
| `/opt/appdata/authentik/redis` | `/data` | Redis persistence |
| `/opt/appdata/authentik/data` | `/data` | Authentik application data |
| `/opt/appdata/authentik/media` | `/media` | Uploaded media |
| `/opt/appdata/authentik/certs` | `/certs` | Certificates |
| `/opt/appdata/authentik/custom-templates` | `/templates` | Custom UI templates |

---

## v2 → v3 Migration

Authentik was restored from a v2 backup rather than set up fresh — all OIDC providers, applications, users, groups, and policies carried forward intact.

### Restore Process
1. Appdata backed up from v2 host to Synology via Active Backup for Business
2. `.env` file recovered from Synology (not in NAS rsync backup — rsync excluded secrets)
3. Appdata copied to auth-prod-01: `/opt/appdata/authentik/`
4. Compose started with original `PG_PASS` and `AUTHENTIK_SECRET_KEY` values
5. Authentik came up with full v2 config intact — no reconfiguration needed

---

## SSO-Enabled Services

| Service | Method | Notes |
|---------|--------|-------|
| Proxmox | OIDC | Both nodes |
| Audiobookshelf | OIDC | Sub unlink fix documented |
| Calibre-Web-Automated | OAuth2 | Manual link per user |
| Beszel | OIDC | Requires email_verified: true custom scope |
| Synology DSM | OIDC | Local admin retained as break-glass |
| Homarr | OIDC | v3 new |
| Immich | OIDC | v3 new |
| ARR stack, Uptime Kuma | None | LAN-only, low risk — no SSO overhead |

---

## Design Decisions

| Decision | Rationale |
|----------|-----------|
| Dedicated VM, not shared with docker-prod-01 | LXC officially removed from Proxmox community scripts May 2025. VM is the only stable option. Isolates IdP from media stack resource contention. |
| No disk cache | ZFS (local-zfs) has its own ARC caching. QEMU cache layer is unnecessary and can cause data integrity issues on ZFS storage. |
| Secrets carried forward from v2 | PG_PASS and AUTHENTIK_SECRET_KEY must match the existing database — changing them breaks the restore. |

---

## Notes

- Root SSH login: disabled (standard Ubuntu default)
- QEMU guest agent installed — Proxmox can cleanly shut down this VM on UPS event
- NUT client not yet configured — to be done in Phase 5
- Authentik worker mounts the Docker socket (`/var/run/docker.sock`) — required for embedded outpost functionality
- "No providers assigned to this outpost" warning in logs is normal — embedded outpost present but not configured for any providers
- Bootstrap email/password only apply on very first boot with empty database — ignored on subsequent starts