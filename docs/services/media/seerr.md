# Seerr

**Host:** docker-prod-01 (192.168.30.11)  
**Container name:** seerr  
**Status:** Running  
**Last Updated:** 2026-11-14

---

## Deployment

- **Image:** `ghcr.io/seerr-team/seerr:latest`
- **Compose:** `/opt/stacks/arr/compose.yaml`
- **Appdata:** `/opt/appdata/seerr/`
- **URL:** `https://request.giohosted.com`
- **Network:** `proxy`
- **Config source:** Restored from v2 backup (formerly Overseerr)

## Container Variables

| Variable | Value |
|---|---|
| TZ | America/Chicago |

> Seerr v3 runs internally as `node:node` (UID 1000:1000) — PUID/PGID are not used. Appdata must be owned by `1000:1000`.

The following command was used to correct ownership during the original migration:

```bash
sudo chown -R 1000:1000 /opt/appdata/seerr/
```

This is a historical recovery command, not a routine maintenance step. Verify ownership before changing it.

---

## Overview

Seerr is the media request frontend. Users submit requests for movies and TV shows through Seerr, which passes them to the appropriate Radarr or Sonarr instance. Requests are tracked and users are notified when their media becomes available.

Seerr is intentionally exposed externally via Cloudflare Tunnel at `https://request.giohosted.com`.

**Authentication is handled through Plex. Authentik OIDC is not configured for Seerr.**

The operator grants users access to the Plex server. Authorized Plex users can then authenticate with Seerr using their Plex accounts.

---

## Paths

| Container Path | Host Path | Purpose |
|---|---|---|
| `/app/config` | `/opt/appdata/seerr` | Config and database |

---

## Connected Instances

| Service | URL | Purpose |
|---|---|---|
| Plex | `http://192.168.30.2:32400` | User authentication and library sync |
| Radarr 1080p | `http://radarr-1080p:7878` | Movie requests |
| Radarr 4K | `http://radarr-4k:7878` | 4K movie requests |
| Sonarr TV | `http://sonarr-tv:8989` | TV requests |
| Sonarr Anime | `http://sonarr-anime:8989` | Anime requests |

---

## External Access

Seerr is intentionally available through Cloudflare Tunnel.

| Setting | Configuration |
|---|---|
| Public hostname | `request.giohosted.com` |
| Tunnel | `homelab-v3` |
| Origin | `https://192.168.30.11` |
| Reverse proxy | Traefik |
| Traefik middleware | No `local-only@docker` |
| Cloudflare Access application | None configured |
| Application authentication | Plex |

The published application route was verified in the Cloudflare dashboard on November 14, 2026.

Seerr does not use Cloudflare Access because external access is intended for approved Plex users, who authenticate directly with the application.

The absence of `local-only@docker` is intentional and should not be changed without reviewing the external-access requirements.

### Authentication Boundary

External requests follow this path:

```text
Internet
  ↓
Cloudflare Tunnel
  ↓
Traefik
  ↓
Seerr login
  ↓
Plex authentication
```

Cloudflare Tunnel provides network connectivity and routing but is not itself an application authorization mechanism.

Seerr must continue enforcing authentication and user permissions.

---

## Authentication and User Provisioning

**Verified configuration — 2026-11-14**

Seerr uses Plex as its intended user authentication provider.

| Setting | Configuration |
|---|---|
| Enable Plex Sign-In | Enabled |
| Enable New Plex Sign-In | Enabled |
| Enable Local Sign-In | Disabled |
| Authentik OIDC | Not configured |
| User provisioning | Plex-based |
| Administrative access | Restricted through Seerr permissions |

The operator confirmed that users are granted access to the Plex server before being allowed to use Seerr.

New Plex Sign-In allows eligible users to authenticate and receive their configured default permissions without the administrator manually creating a local-password account.

**Security requirement:** Preserve the connection between Seerr user eligibility and the intended Plex server access list. Review Plex sharing permissions when adding or removing users.

### Local Sign-In Hardening

During the November 14 security review, Local Sign-In was found enabled even though the operator did not use local-password accounts.

The setting was changed:

**Settings → Users → Enable Local Sign-In → Disabled**

The operator then successfully signed in through Plex.

**Result:** Plex authentication continued functioning after the change.

Local-password authentication is not part of the intended configuration and should remain disabled unless an operational requirement is identified.

Disabling Local Sign-In also removes the local-password recovery path. Administrators should retain reliable access to the configured Plex identity.

### Default User Permissions

The November 14 settings review showed:

| Permission or behavior | Configuration |
|---|---|
| Administrator permission by default | Disabled |
| Manage Users permission by default | Disabled |
| Request permission | Enabled |
| Default automatic approval | Enabled |
| Global request quotas | Unlimited |

These permissions reflect the current operator preference.

Users can submit requests without receiving default administrative or user-management privileges.

Default automatic approval permits eligible requests to progress without manual administrator approval according to Seerr's configured rules.

**Operational consideration:** Unlimited request quotas are intentional but can increase download and storage consumption. Revisit quotas if abuse, excessive requests, or capacity pressure becomes a problem.

### Verification — November 14, 2026

The security review included:

1. Confirming that no Cloudflare Access application covered `request.giohosted.com` in the reviewed application inventory.
2. Verifying the Cloudflare Tunnel's published route.
3. Opening Seerr in a fresh incognito browser session.
4. Confirming unauthenticated visitors reached the `/login` page rather than the application dashboard.
5. Inspecting the user authentication and default permission settings.
6. Disabling Local Sign-In.
7. Confirming Plex login still worked afterward.

The verification established that the tested browser workflow requires authentication.

It did not constitute a penetration test of all Seerr endpoints or a separate audit of every existing user's privileges.

---

## Issues Encountered

| Issue | Resolution |
|---|---|
| Crash-looping on startup | Seerr v3 runs as `node:node` (UID 1000) — v2 appdata was owned by `service` (UID 2000). Fixed with `chown -R 1000:1000` on appdata |
| Repeatedly falling off proxy network | Caused by crash-loop — resolved once permissions were fixed and container stabilized |
| Unnecessary local-password login enabled | Disabled Local Sign-In on 2026-11-14; Plex login verified working |
| Outdated Authentik reference in documentation | Corrected — Authentik OIDC was not configured for Seerr |

---

## Security and Maintenance

When reviewing Seerr configuration or performing upgrades:

1. Preserve Plex authentication and the intended Plex sharing restrictions.
2. Keep Local Sign-In disabled unless specifically required.
3. Review new Plex user permissions before changing auto-provisioning settings.
4. Keep default administrator and user-management permissions restricted.
5. Verify that unauthenticated browser requests cannot access protected application functionality.
6. Confirm authenticated Plex users can sign in after updates.
7. Review changes to Cloudflare Tunnel routes and Traefik routing.
8. Do not expose additional administrative interfaces unnecessarily.

A Cloudflare Access layer could be introduced in the future, but it is not required solely because Seerr is publicly reachable. Any additional authentication layer should be evaluated against the actual Plex user workflow.

---

## Notes

- The public URL is `request.giohosted.com`, not `seerr.giohosted.com` — the user-facing name is more intuitive.
- Seerr is a fork of Overseerr; the v2 Overseerr configuration and database were restored successfully.
- **Authentik OIDC is not configured for Seerr.**
- Plex authentication is the intended login method.
- Local Sign-In was disabled on 2026-11-14 and Plex login was verified afterward.
- Cloudflare Access is intentionally not configured for Seerr.
- The Traefik router intentionally omits `local-only@docker` because external access is required.
- New Plex users receive the configured Seerr default permissions.
- Default administrative permissions remain restricted.
- Request auto-approval and unlimited quotas should be reviewed if operational requirements change.

---

## Related Documentation

- `services/infra/cloudflared.md` — Cloudflare Tunnel and external application routes.
- `services/infra/traefik.md` — Reverse proxy and HTTP security controls.
- `services/media/plex.md` — Plex server, user access, and authentication.
- `runbooks/updating-containers.md` — Docker application updates and post-deployment verification.