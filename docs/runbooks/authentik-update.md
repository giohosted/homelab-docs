# Authentik Update Procedure

**Applies to:** auth-prod-01 (192.168.30.13)
**Last Updated:** 2026-07-13

---

## Critical: Sequential Upgrade Requirement

Authentik enforces sequential major-version upgrades. **You cannot skip directly from an old version family to the latest** — attempting to do so risks failed database migrations with no clean rollback path other than restoring from backup.

Upgrades must go through every intermediate major version (each `YYYY.MM` family), always landing on the latest patch within a family before moving to the next:

Example: `2025.12.3` → latest `2025.12.x` → latest `2026.2.x` → latest `2026.5.x`

Check the current version gap before starting:
- Current version: Admin UI → Dashboards → System, or `docker compose logs server | grep version` after any restart
- Latest available: https://github.com/goauthentik/authentik/releases
- Determine every intermediate family between current and latest — do not skip any

Authentik does **not** support downgrading. A pre-upgrade database backup is the only way back if a migration goes wrong.

---

## Procedure (per hop)

Repeat for each version family between current and target.

1. **Back up the database first:**
```bash
   cd /opt/stacks/authentik
   docker compose exec postgresql pg_dump -U authentik authentik > ~/backup-pre-upgrade-$(date +%Y%m%d).sql
   ls -lh ~/backup-pre-upgrade-*.sql
```
   Confirm non-trivial file size before proceeding.

2. **Update the version tag** in `.env` (not `compose.yaml` — the tag is templated as `${AUTHENTIK_TAG:-...}` and `.env` overrides the default):
```bash
   nano .env
```
   Update `AUTHENTIK_TAG=` to the target version.

3. **Check `.env` ownership** — should be `gio:gio` on auth-prod-01, not root (easy to break if `sudo nano` was used by habit):
```bash
   ls -la .env
   # if wrong:
   sudo chown gio:gio .env
   sudo chmod 660 .env
```

4. **Pull and recreate:**
```bash
   docker compose pull
   docker compose up -d
```

5. **Verify migrations succeeded** — watch for `OK` on every migration line and a clean `releasing database lock`, with no `ERROR` or `Traceback`:
```bash
   sleep 15
   docker compose ps
   docker compose logs server --tail 60
```

6. **Verify end-to-end**, not just container health:
   - Log into `https://auth.giohosted.com`, confirm Dashboard → System shows the new version
   - Test at least one OIDC-integrated app (e.g. Audiobookshelf) to confirm SSO round-trips correctly

7. Only after verification, move to the next hop.

---

## After All Hops Complete

Clean up old images once fully verified on the latest target version:
```bash
docker images | grep authentik
docker rmi ghcr.io/goauthentik/server:<old-tag>
```

Don't clean up mid-upgrade — keeping old images cached locally means a rollback (if needed) doesn't require re-pulling from GHCR mid-incident.

---

## Notes

- `authentik-server` and `authentik-worker` share the same image/tag — both update together automatically since they reference the same `.env` variable
- Support window is typically the two most recent version families — falling behind means losing security patches, which is itself a reason to check for updates periodically rather than only when a feature is needed
- This is Authentik's SSO for nearly every homelab service — treat every hop with the same caution as the first, even if prior hops went smoothly