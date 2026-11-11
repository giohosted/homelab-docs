# Profilarr

**Host:** docker-prod-01 (192.168.30.11)  
**Container name:** profilarr  
**Status:** Running — Healthy  
**Last Updated:** 2026-11-10

---

## Deployment

- **Image:** `ghcr.io/dictionarry-hub/profilarr:latest`
- **Verified application version:** 2.3.1
- **Compose:** `/opt/stacks/arr/compose.yaml`
- **Komodo stack:** `arr`
- **Appdata:** `/opt/appdata/profilarr-v2/`
- **URL:** `https://profilarr.giohosted.com`
- **Network:** `proxy`
- **Config source:** Existing Profilarr v2 configuration migrated from the temporary standalone deployment

## Container Variables

| Variable | Value |
|----------|-------|
| PUID | 2000 |
| PGID | 2000 |
| TZ | America/Chicago |
| ORIGIN | https://profilarr.giohosted.com |

---

## Overview

Profilarr centrally manages quality profiles, custom formats, release scoring, and supported media management settings across Sonarr and Radarr instances.

Profilarr v2 uses the Dumpstarr configuration database as its sole upstream configuration source. Dumpstarr provides curated quality profiles and custom formats based on TRaSH Guides and related recommendations.

All four Arr instances are connected to Profilarr, with quality profile synchronization configured to run automatically when upstream database changes are pulled.

---

## Paths

| Container Path | Host Path | Purpose |
|----------------|-----------|---------|
| `/config` | `/opt/appdata/profilarr-v2` | Application configuration and database |

The previous Profilarr v1 appdata remains at `/opt/appdata/profilarr` for rollback purposes. It is not used by the active deployment.

---

## Connected Instances

| App | Internal URL | Quality Profiles Trigger |
|-----|--------------|--------------------------|
| Sonarr TV | `http://sonarr-tv:8989` | On Pull |
| Sonarr Anime | `http://sonarr-anime:8989` | On Pull |
| Radarr 1080p | `http://radarr-1080p:7878` | On Pull |
| Radarr 4K | `http://radarr-4k:7878` | On Pull |

Each instance uses its own API key stored in Profilarr.

### Quality Profile Selections

| Instance | Dumpstarr Profiles |
|----------|--------------------|
| Sonarr TV | TV 1080p, TV 2160p |
| Sonarr Anime | Anime 1080p |
| Radarr 1080p | LQ 1080p, Movies 1080p, Movies 1080p HQ |
| Radarr 4K | Selected Dumpstarr 4K profiles — exact names not independently recorded |

---

## Configuration Database

| Setting | Value |
|---------|-------|
| Database | Dumpstarr |
| Repository | https://github.com/Dumpstarr/Database |
| Branch | `stable` |
| Conflict strategy | Align |
| Automatic pull | Enabled |
| Pull interval | Hourly |
| Quality Profiles sync trigger | On Pull |
| Dictionarry database | Unlinked |

### Synchronization Behavior

- Profilarr automatically pulls updates from the Dumpstarr `stable` branch.
- Quality Profiles configured with `On Pull` synchronize when database updates are pulled.
- Selected custom formats and their scores are synchronized as part of quality profile synchronization.
- Existing quality profiles are matched by name and updated in place where possible.
- Profiles or custom formats not selected for synchronization are not automatically removed by ordinary quality profile synchronization.
- Naming, quality definitions, media settings, and delay profiles have separate synchronization controls and should be reviewed independently.
- Upstream scoring changes can affect future grabs and quality upgrades in Sonarr and Radarr.

---

## Reverse Proxy and Access Control

Profilarr is accessed through Traefik using the existing Homelab HTTPS routing configuration.

| Setting | Value |
|---------|-------|
| Hostname | `profilarr.giohosted.com` |
| Entry point | `websecure` |
| TLS | Enabled |
| Container port | 6868 |
| Docker network | `proxy` |
| Middleware | `local-only@docker` |
| Published host ports | None |

The `ORIGIN=https://profilarr.giohosted.com` environment variable is required for the application's reverse-proxy origin validation.

Profilarr is restricted through the existing local-only Traefik middleware rather than exposed for unrestricted public access.

---

## Backup and Recovery

### Verified Migration Backups

| Backup | Location |
|--------|----------|
| Profilarr v1 | `/opt/backups/profilarr/profilarr-v1-20261110-043935.tar.gz` |
| Profilarr v2 pre-cutover | `/opt/backups/profilarr/profilarr-v2-pre-cutover-20261111-025838.tar.gz` |
| Arr appdata pre-migration | `/opt/backups/profilarr/arr-pre-v2-20261110-044214.tar.gz` |

The listed archives were verified during migration. The pre-cutover v2 backup was created with the application stopped.

### Legacy Appdata

- **Legacy v1:** `/opt/appdata/profilarr`
- **Active v2:** `/opt/appdata/profilarr-v2`

Preserve the legacy v1 directory and migration backups until they are no longer needed for rollback.

### Recovery Considerations

- Stop Profilarr before restoring its active database or appdata.
- Back up the current appdata before performing any rollback.
- Restore a compatible application version and its corresponding configuration.
- Verify database loading, connected Arr instances, and synchronization configuration after restoration.
- Do not run two Profilarr instances against the same writable appdata directory.

---

## Migration History

On November 10, 2026, Profilarr was migrated from the original v1 application to Profilarr v2.

The migration included:

1. Backing up Profilarr v1 and all four Arr application configurations.
2. Deploying Profilarr v2 temporarily using a separate Compose project.
3. Linking the Dumpstarr database using its `stable` branch.
4. Synchronizing the selected quality profiles and custom formats for both Sonarr and both Radarr instances.
5. Configuring quality profile synchronization using the `On Pull` trigger.
6. Unlinking the unused Dictionarry configuration database.
7. Replacing the legacy Profilarr service within the existing `arr` Compose stack.
8. Retaining the existing Traefik hostname, internal-only access middleware, and container port.
9. Configuring the `ORIGIN` environment variable for successful reverse-proxy access.
10. Switching the Docker image reference to `ghcr.io/dictionarry-hub/profilarr:latest`.
11. Removing the temporary standalone deployment files after verifying the new container was healthy.

The active deployment is now managed through the existing `arr` stack. The four Sonarr/Radarr containers were not intentionally recreated during the Profilarr service cutover.

---

## Notes

- Quality profiles and custom formats should ordinarily be managed in Profilarr, not edited directly in individual Arr instances, since synchronized changes can overwrite local settings.
- The Docker `latest` tag does not automatically update the running application; a new image must be pulled and the container recreated.
- Future `latest` images may contain major application upgrades. Review release notes and back up appdata before upgrading.
- Preserve `ORIGIN` when modifying the Compose configuration.
- The standalone `/opt/stacks/profilarr-v2/` deployment directory was removed after migration.
- All four Arr integrations and the Profilarr web interface were checked following the migration.
