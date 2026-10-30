# Dockman

**Role:** Docker Compose management UI — start, stop, restart, and update containers  
**Host:** docker-prod-01 (192.168.30.11)  
**Image:** `ghcr.io/ra341/dockman:latest`  
**Compose:** `/opt/stacks/dockman/compose.yaml`  
**Appdata:** `/opt/appdata/dockman/`  
**URL:** `https://dockman.giohosted.com`  
**Status:** Documented active deployment — current container health not independently reverified  
**Last Updated:** 2026-10-29

---

## Overview

Dockman provides a web interface for managing Docker Compose stacks on `docker-prod-01`.

Its primary advantage is the ability to manage individual containers without unnecessarily restarting every service in a Compose stack.

This is particularly useful for the ARR stack, where restarting one application should not disrupt unrelated services.

Dockman is an internal management application and is not exposed through the homelab's Cloudflare Tunnel.

### Management Scope

Dockman manages the local Docker engine on `docker-prod-01`.

|Host|Managed by Dockman|
|---|---|
|docker-prod-01|Yes|
|auth-prod-01|No|
|immich-prod-01|No|
|observer-prod-01|No|

Although these hosts use the same `homelab-infra` repository, their Docker engines are separate.

Dockman does not automatically manage remote hosts simply because their Compose configurations exist in the same Git repository.

**AI Observer:** `observer-prod-01` uses Docker Compose through SSH and CLI. Its OpenAI tunnel client is managed separately through systemd.

See `services/infra/ai-observer.md`.

---

## Configuration

### Environment Variables (.env)

|Variable|Notes|
|---|---|
|`DOCKMAN_AUTH_USERNAME`|Login username — stored in `.env`, gitignored|
|`DOCKMAN_AUTH_PASSWORD`|Login password — stored in `.env`, gitignored|

Authentication credentials must not be committed to the `homelab-infra` repository.

### Key Settings

|Setting|Value|Notes|
|---|---|---|
|`DOCKMAN_COMPOSE_ROOT`|`/opt/stacks`|Root directory for Compose management|
|`DOCKMAN_AUTH_ENABLE`|`true`|Basic authentication enabled|
|`DOCKMAN_LOG_AUTH_WARNING`|`false`|Suppresses authentication warning|

### Volumes

|Host Path|Container Path|Notes|
|---|---|---|
|`/opt/stacks`|`/opt/stacks`|Compose configuration files|
|`/opt/appdata/dockman/config`|`/config`|Dockman configuration and state|
|`/var/run/docker.sock`|`/var/run/docker.sock`|Local Docker socket — required for container management|

### Docker Socket Security

Dockman requires access to the Docker socket to manage containers.

Access to this socket is highly privileged and can effectively provide control over the host's Docker environment.

Consequently:

- Dockman must remain restricted to authorized internal users.
    
- Authentication must remain enabled.
    
- The Docker socket must not be exposed over an unauthenticated network endpoint.
    
- Dockman should not be exposed publicly without a separate security review.
    
- Its access should not be extended to other Docker hosts without an explicit architecture and security decision.
    

The socket mount grants management access only to the Docker engine available at that mounted socket.

It does not independently provide remote access to Docker on observer-prod-01 or other hosts.

---

## Access

**URL:** `https://dockman.giohosted.com`

**Documented access controls:**

- Internal VLAN access only.
    
- Routed through Traefik.
    
- Uses `local-only@docker` middleware.
    
- Not exposed through Cloudflare Tunnel.
    
- Dockman authentication enabled.
    

Port `8866` is not directly exposed for normal client access; application access is routed through Traefik.

The current Traefik configuration and UniFi enforcement were not independently re-audited during this documentation update.

---

## File Permission Requirement

Dockman edits Compose configuration files directly through its interface.

For autosave to function, the relevant files must be writable by the account or group used by Dockman.

The established permissions on docker-prod-01 are:

```
# compose.yaml — group writable, world readable
sudo chown gio:service /opt/stacks/<stack-name>/compose.yaml
sudo chmod 664 /opt/stacks/<stack-name>/compose.yaml

# .env — group writable, not world readable
sudo chown gio:service /opt/stacks/<stack-name>/.env
sudo chmod 660 /opt/stacks/<stack-name>/.env
```

Files created through `sudo nano` may be owned by `root:root`, preventing Dockman from saving changes.

A previously observed symptom was:

`Error: 500 - Error saving file`

Correcting the relevant ownership and permissions resolved the issue.

### Permission Reference

|Path|Owner|Group|Mode|Purpose|
|---|---|---|---|---|
|`compose.yaml`|gio|service|664|Allows Dockman to edit Compose files|
|`.env`|gio|service|660|Allows authorized access without world-readable secrets|
|`/opt/appdata/<app>/`|2000:2000|—|755|Established service ownership for applicable containers|

**Scope:** These settings apply to the documented Docker deployment on docker-prod-01.

Do not automatically apply them to other hosts.

For example, `observer-prod-01` uses different ownership and credential restrictions. Its MCP `.env` is configured as `gio:gio` with mode `0600`.

See `runbooks/new-stack-checklist.md`.

---

## Git Integration

Dockman operates on the local `/opt/stacks` repository checkout.

This directory is part of the private `homelab-infra` GitHub repository.

Changes made through Dockman's interface are local until committed and pushed through Git.

### After Editing Compose Configuration

From docker-prod-01:

```
cd /opt/stacks

# Inspect the working tree
git status

# Review intended modifications
git diff

# Stage the appropriate file
git add -- <stack-name>/compose.yaml

# Review staged content
git diff --cached

# Commit and push
git commit -m "update <stack-name> configuration"
git push
```

If Git reports that the remote branch has advanced, inspect and reconcile the differences before pushing.

Do not force-push to bypass conflicts.

**Important:** Do not run `git pull` over unreviewed local changes made through Dockman.

Review and preserve the changes first.

`.env` files remain excluded from Git.

See `runbooks/git-workflow.md`.

---

## Operations and Maintenance

### Routine Container Management

Use the Dockman interface for supported operations such as:

- Viewing local Compose stacks.
    
- Checking container status.
    
- Starting or stopping containers.
    
- Restarting individual containers.
    
- Updating images and containers.
    
- Reviewing and editing Compose configuration.
    

Before updating a critical application, review its release notes, backups, and service-specific upgrade requirements.

Some services require coordinated updates or database migrations and should not be updated using a generic one-click operation.

See `runbooks/updating-containers.md`.

### Read-Only CLI Health Checks

When Dockman is unavailable or its state is unclear, SSH into docker-prod-01:

```
ssh gio@192.168.30.11
```

Navigate to the Compose directory:

```
cd /opt/stacks/dockman
```

Inspect the deployment:

```
docker compose ps
```

Review recent logs:

```
docker compose logs --tail 100 dockman
```

Validate the Compose configuration without displaying resolved environment secrets:

```
docker compose config --quiet
```

These commands do not modify the running deployment.

---

## Updating Dockman Itself

**Use Docker Compose through SSH rather than relying on Dockman's own web interface for self-updates.**

During a previous Dockman self-update, the existing container stopped and the replacement was created but did not start.

The service was subsequently recovered through Docker Compose.

### Recommended Procedure

1. Review the upstream release and current deployment.
    
2. Confirm required configuration and backup readiness.
    
3. SSH into docker-prod-01.
    
4. Inspect the current container.
    
5. Pull the approved image.
    
6. Recreate only the Dockman service.
    
7. Verify the web interface.
    

Commands:

```
ssh gio@192.168.30.11
```

```
cd /opt/stacks/dockman

# Inspect the existing deployment
docker compose ps
docker compose images
```

After confirming the update is appropriate:

```
docker compose pull dockman

docker compose up -d --no-deps dockman
```

Verify:

```
docker compose ps
docker compose logs --tail 100 dockman
```

Finally, verify that `https://dockman.giohosted.com` loads and displays the expected stacks.

Do not mark the update successful based solely on a running container.

### Recovery — Container Created but Not Started

The previously successful recovery command was:

```
cd /opt/stacks/dockman

sudo docker compose up -d --no-deps dockman
```

After recovery:

```
sudo docker compose ps
sudo docker compose logs --tail 100 dockman
```

If Dockman remains unavailable, investigate the container logs, Traefik routing, and application state before attempting additional changes.

Do not repeatedly initiate self-updates through a nonfunctional UI.

---

## AI Observer Management Boundary

AI Observer was deliberately deployed independently from the production Docker host.

**Observer VM:** `observer-prod-01`  
**IP:** `192.168.30.17`  
**Proxmox VMID:** 107

Its components include:

|Component|Management|
|---|---|
|Proxmox MCP container|Local Docker Compose CLI|
|OpenAI Secure MCP Tunnel|systemd|
|Proxmox API credentials|Protected local configuration|
|OpenAI tunnel credentials|Protected `/etc/tunnel-client` files|

Dockman on docker-prod-01 does not manage these components.

The observer's management boundary is intentional because AI integrations interact with infrastructure monitoring APIs and require separately controlled credentials and permissions.

**Decision:** Retain CLI-only observer management unless a future architecture decision establishes a secure and justified reason to introduce centralized Docker management.

No such management change has been implemented.

See `services/infra/ai-observer.md`.

---

## Backup and Recovery

Dockman's relevant persistent data includes:

- Version-controlled Compose configuration in `/opt/stacks/dockman/`.
    
- Git-excluded authentication configuration in `.env`.
    
- Application state in `/opt/appdata/dockman/`.
    

The documented nightly rsync job on docker-prod-01 includes `/opt/stacks` and `/opt/appdata`.

The actual latest rsync execution and restoration capability were not reverified during this documentation update.

Docker host recovery also depends on the broader Proxmox and NAS backup infrastructure.

**Known issue:** Scheduled PBS backup jobs on both Proxmox nodes were confirmed failing during the October 29, 2026 investigation.

That issue remains open and must not be interpreted as resolved by the existence of the scheduled backup configuration.

See `runbooks/backup-schedule.md`.

---

## Notes

- Dockman manages only docker-prod-01.
    
- The local Compose root is `/opt/stacks`, matching the `homelab-infra` checkout.
    
- Compose changes made through Dockman must be reviewed and committed to Git separately.
    
- Dockman has privileged Docker socket access and must remain appropriately restricted.
    
- The documented `service` group on docker-prod-01 uses GID 2000.
    
- File permissions are host-specific; do not reuse them blindly on AI Observer.
    
- Port 8866 is not the documented direct client access path; access is through Traefik.
    
- Dockman is not exposed through the existing Cloudflare Tunnel.
    
- The original v3 installation did not restore Dockman appdata from v2 because its configuration was minimal.
    
- Dockman self-updates should use the CLI.
    
- The observer's tunnel client is a native systemd service and is outside Dockman's scope.
    

---

## Related Documentation

- `runbooks/new-stack-checklist.md` — Docker stack creation and required permissions on docker-prod-01.
    
- `runbooks/updating-containers.md` — General and service-specific Docker update procedures.
    
- `runbooks/git-workflow.md` — Repository synchronization and version control.
    
- `runbooks/backup-schedule.md` — Current backup schedules and verification status.
    
- `services/infra/ai-observer.md` — Independent AI Observer infrastructure and security boundaries.
    
- `runbooks/ai-observer-operations.md` — AI Observer operations and recovery