# Updating a Docker Container

**Applies to:** docker-prod-01, auth-prod-01, immich-prod-01, observer-prod-01  
**Last Updated:** 2026-10-29

---

## Overview

Docker containers in Homelab v3 are managed through Docker Compose stacks under `/opt/stacks`.

Updating a container generally involves:

1. Reviewing the target application's release notes.
    
2. Verifying configuration and backup readiness.
    
3. Pulling the intended Docker image.
    
4. Recreating the affected container.
    
5. Verifying the application and its dependencies.
    

Docker Compose updates can be performed through Dockman where supported or directly through the CLI.

Not all services should follow the same update procedure. Applications with databases, migrations, or special dependencies may require service-specific runbooks.

**Important:** An image update is different from a Compose configuration change. Pulling a newer image does not require a Git commit unless a version-controlled file was modified.

---

## Before Updating

### 1. Review the Release

Before making changes:

- Identify the currently deployed version or image.
    
- Review the target release notes for breaking changes.
    
- Check whether database migrations or configuration changes are required.
    
- Identify dependencies that may be affected.
    
- Review the relevant service documentation or specialized update runbook.
    

Avoid blindly updating critical services simply because a newer image is available.

Containers using the `latest` tag require particular care because the tag can point to a different image without a Compose file change.

### 2. Verify Backups

Confirm that the affected application's required configuration and persistent data are protected.

Depending on the service, this may include:

- Proxmox Backup Server VM backups.
    
- Docker appdata backups.
    
- Database-specific backups.
    
- Git-excluded `.env` files.
    
- Host-level configuration outside `/opt/stacks`.
    

An existing backup schedule is not proof that a usable backup exists.

For database-backed or management-critical services, verify recent successful backups before proceeding with nonurgent updates.

**AI Observer:** VM 107 is included in the configured Proxmox backup-job selection, but no archive was visible during the October 30, 2026 UTC verification. Its recoverable backup status must be checked before a routine update.

See `runbooks/backup-schedule.md`.

### 3. Review the Git Working Tree

On hosts using `homelab-infra`:

```
cd /opt/stacks

git status
```

If the working tree is clean and configuration synchronization is needed:

```
git pull --ff-only
```

On observer-prod-01, authenticated Git operations use the credential-helper-disabled procedure described in `runbooks/git-workflow.md`.

If local changes are present, review and reconcile them before pulling.

Do not reset or discard changes merely to make an update proceed.

### 4. Check the Current Deployment

From the target stack directory:

```
docker compose ps

docker compose images

docker compose config --quiet
```

Use `sudo docker` where Docker access requires elevated privileges.

`docker compose config --quiet` validates the Compose configuration without printing its resolved environment values.

If the current configuration is invalid or the service is already unhealthy, investigate before updating.

---

## Method 1 — Dockman (UI)

**Supported host:** docker-prod-01  
**URL:** `https://dockman.giohosted.com`

Dockman provides a web interface for managing Docker Compose stacks on docker-prod-01.

It does not manage Docker engines on observer-prod-01, auth-prod-01, or immich-prod-01.

### Procedure

1. Open `https://dockman.giohosted.com`.
    
2. Sign in.
    
3. Locate the stack containing the application.
    
4. Review the running containers and their status.
    
5. Review the application's release notes and backup requirements.
    
6. Select the appropriate update or image-pull operation.
    
7. Check which services will be recreated.
    
8. Apply the update.
    
9. Verify the affected containers and application functionality.
    

**Important:** Do not assume that every Dockman update action affects only one container. Confirm the scope of the selected operation before proceeding.

Some operations can affect multiple services in the same Compose stack.

### Updating Individual Services

Where the installed Dockman version supports individual service operations, prefer targeted changes when updating one component should not interrupt unrelated services.

For example, updating one ARR application should not automatically require restarting the entire media stack.

However, targeted updates are appropriate only when the application does not require coordinated changes to its dependencies.

### Dockman Self-Updates

**Do not rely on Dockman to update its own running container.**

During an earlier Dockman self-update, the original container stopped and the replacement container was created but not started.

The service was recovered through Docker Compose on docker-prod-01.

For future Dockman self-updates, prefer the CLI.

See `services/infra/dockman.md`.

---

## Method 2 — CLI

CLI updates are supported on all Docker Compose hosts.

They are the preferred method for:

- observer-prod-01
    
- Services requiring precise dependency control
    
- Dockman itself
    
- Troubleshooting failed updates
    
- Service-specific maintenance procedures
    

### Standard Stack Update

SSH into the host running the container.

```
# Navigate to the stack directory
cd /opt/stacks/<stack-name>

# Review the current deployment
docker compose ps
docker compose images

# Validate configuration
docker compose config --quiet

# Pull the images referenced by the stack
docker compose pull

# Recreate services whose image or configuration changed
docker compose up -d

# Verify container status
docker compose ps

# Inspect recent logs
docker compose logs --tail 50
```

Use `sudo docker` if required by the host's permissions.

### Updating One Compose Service

When a stack contains multiple services and only one needs updating:

```
cd /opt/stacks/<stack-name>

# Pull the selected service image
docker compose pull <service-name>

# Recreate only the selected service
docker compose up -d --no-deps <service-name>

# Verify
docker compose ps
docker compose logs --tail 50 <service-name>
```

`--no-deps` prevents Docker Compose from starting or updating the service's dependencies as part of this command.

**Use this only when dependencies do not require coordinated updates.**

For example, an application version requiring a database migration may need its own documented update sequence.

### Important Compose Behavior

- `docker compose pull` downloads images but does not recreate running containers.
    
- `docker compose up -d` reconciles the stack against its Compose configuration and available images.
    
- Containers with changed images or configuration may be recreated.
    
- Unchanged containers are generally left running.
    
- `docker compose up -d --no-deps <service-name>` restricts the operation to the specified service.
    
- A recreated container may be running before its application is actually ready.
    

Container status alone is not sufficient verification.

---

## Example — Updating the Media Stack on docker-prod-01

**Host:** docker-prod-01  
**IP:** 192.168.30.11

First identify the appropriate Compose stack and service.

For the existing media stack:

```
ssh gio@192.168.30.11

cd /opt/stacks/media

docker compose ps

docker compose images
```

For an approved full-stack update:

```
docker compose config --quiet

docker compose pull

docker compose up -d

docker compose ps

docker compose logs --tail 50
```

Afterward, verify the affected applications through their normal interfaces.

When updating only one service, use the targeted Compose procedure if compatible with that application's dependencies.

Avoid unnecessary restarts of unrelated services.

---

## Example — Updating Authentik on auth-prod-01

**Host:** auth-prod-01  
**IP:** 192.168.30.13

**Do not use the generic Docker update procedure for Authentik.**

Authentik uses database migrations and may require intermediate version upgrades.

The supported operational workflow is documented in:

`runbooks/authentik-update.md`

Before updating:

1. Read the specialized runbook.
    
2. Identify the currently deployed version.
    
3. Determine the required upgrade path.
    
4. Verify the database backup.
    
5. Follow the documented upgrade sequence.
    
6. Verify application health and OIDC authentication after each upgrade step.
    

Do not bypass the required checks by simply pulling `latest` or blindly recreating the stack.

---

## Example — Updating Immich on immich-prod-01

**Host:** immich-prod-01  
**IP:** 192.168.30.14

Immich updates require special attention to release compatibility and database migrations.

Before updating:

- Read the official release notes.
    
- Confirm the current and target versions.
    
- Review changes to the supported Compose configuration.
    
- Verify the required database and application backups.
    
- Confirm that persistent photo storage is available.
    

The Compose stack is located at:

`/opt/stacks/immich`

Follow the version-appropriate Immich upgrade instructions rather than assuming a generic image pull is sufficient.

Verify the application UI, database connectivity, and photo access after updating.

---

## Method 3 — Proxmox MCP on observer-prod-01

**Host:** observer-prod-01  
**IP:** 192.168.30.17  
**Container:** proxmox-mcp  
**Compose:** `/opt/stacks/proxmox-mcp/compose.yaml`  
**Image:** `ghcr.io/mattoddie/proxmox-mcp:latest`

The Proxmox MCP service runs as a dedicated Docker Compose stack on the AI Observer VM.

It is managed through SSH and the CLI.

Dockman is not installed on observer-prod-01 and does not manage this container remotely.

### Update Requirements

Before updating Proxmox MCP:

1. Review the upstream project's release notes and changes.
    
2. Verify that the target version remains compatible with the current MCP transport and tool configuration.
    
3. Verify that a usable recovery point exists for observer-prod-01 or establish an approved recovery procedure.
    
4. Check that the existing MCP and tunnel services are healthy.
    
5. Record the currently deployed image version or image ID.
    
6. Preserve the existing read-only configuration and credentials.
    

The current deployment uses the `latest` image tag.

Because `latest` is mutable, record the existing image ID before pulling a replacement. A prior Compose file alone may not identify the previously deployed image.

### Step 1 — Connect to the Observer

```
ssh gio@192.168.30.17
```

Navigate to the MCP stack:

```
cd /opt/stacks/proxmox-mcp
```

### Step 2 — Inspect the Existing Deployment

```
# Check the current container
sudo docker compose ps

# Show the currently referenced image
sudo docker compose images

# Record the current image ID
sudo docker image inspect \
  ghcr.io/mattoddie/proxmox-mcp:latest \
  --format '{{.Id}}'

# Validate the Compose configuration
sudo docker compose config --quiet
```

Record the image ID in the maintenance notes before proceeding.

Do not display or share the full contents of the credential-bearing `.env` file.

### Step 3 — Verify MCP Health

```
curl -fsS http://127.0.0.1:8080/health
```

Expected response:

```
{"status":"ok"}
```

Check the tunnel service:

```
sudo systemctl status tunnel-client --no-pager
```

Check tunnel readiness:

```
curl -fsS http://127.0.0.1:8081/readyz
```

Expected response:

```
ready
```

If either component is unhealthy before the update, investigate the cause first.

### Step 4 — Pull the Updated MCP Image

After reviewing the release and confirming backup readiness:

```
cd /opt/stacks/proxmox-mcp

sudo docker compose pull proxmox-mcp
```

This downloads the image referenced by the existing Compose configuration.

It does not restart the running container.

### Step 5 — Recreate the MCP Container

```
sudo docker compose up -d --no-deps proxmox-mcp
```

The service name is `proxmox-mcp`.

This updates the MCP container without deliberately restarting the separate OpenAI tunnel systemd service.

There may still be a brief interruption to MCP requests while the container is recreated.

### Step 6 — Verify the Updated Container

```
sudo docker compose ps

sudo docker compose logs --tail 100 proxmox-mcp

curl -fsS http://127.0.0.1:8080/health
```

Confirm:

- The container is running.
    
- The expected image is deployed.
    
- No startup or configuration errors are present.
    
- The health endpoint responds successfully.
    

### Step 7 — Verify Tunnel Connectivity

```
sudo systemctl status tunnel-client --no-pager

curl -fsS http://127.0.0.1:8081/readyz
```

Review tunnel logs if necessary:

```
sudo journalctl -u tunnel-client -n 100 --no-pager
```

The tunnel should remain active and able to reach the local MCP endpoint.

### Step 8 — Verify End-to-End MCP Functionality

From ChatGPT, request a live Proxmox query through the Homelab Proxmox plugin.

Example:

> Check the live Proxmox cluster status using the Homelab Proxmox integration.

Confirm that the tool successfully returns current cluster information.

This is the final functional verification after an update.

If the upstream release changes tool definitions or MCP discovery behavior, the ChatGPT plugin configuration may need to be reviewed.

Do not mark the update successful based solely on Docker container status or the local health endpoint.

### Step 9 — Verify Security Settings

Confirm that the deployed configuration continues to enforce:

- `PROXMOX_ALLOW_WRITES=false`
    
- `PROXMOX_ALLOW_DELETES=false`
    
- `PROXMOX_ALLOW_EXEC=false`
    
- Proxmox API access through the dedicated audit-only identity.
    
- MCP HTTP binding to `127.0.0.1:8080`.
    
- No new external listeners or unnecessary Docker privileges.
    

Do not enable additional permissions as part of a routine update.

### Updating the OpenAI Tunnel Client

**The tunnel client is not a Docker container.**

It runs as a native Linux systemd service:

`tunnel-client.service`

Its binary is installed at:

`/opt/tunnel-client/tunnel-client`

Do not use Docker Compose commands to update the tunnel client.

Tunnel-client upgrades require a separate procedure covering:

- Review of the official OpenAI tunnel-client release.
    
- Verification of the downloaded artifact and checksum.
    
- Preservation of `/etc/tunnel-client` configuration and protected credentials.
    
- Compatible binary installation.
    
- Controlled systemd restart.
    
- Readiness verification.
    
- End-to-end ChatGPT MCP testing.
    

The detailed procedure belongs in `runbooks/ai-observer-operations.md`.

**Do not update the MCP container and tunnel-client binary simultaneously during routine maintenance.**

Update and verify one component at a time to simplify troubleshooting and rollback.

---

## Updating Dockman Itself

**Host:** docker-prod-01  
**Stack:** `/opt/stacks/dockman`  
**Service:** `dockman`

Dockman should be updated through Docker Compose rather than through its own web UI.

### Procedure

SSH into docker-prod-01:

```
ssh gio@192.168.30.11
```

Navigate to its stack:

```
cd /opt/stacks/dockman
```

Inspect the current deployment:

```
docker compose ps
docker compose images
```

After reviewing the release and confirming configuration backup readiness, update the Dockman service:

```
docker compose pull dockman

docker compose up -d --no-deps dockman
```

Verify:

```
docker compose ps

docker compose logs --tail 50 dockman
```

Finally, reopen:

`https://dockman.giohosted.com`

Confirm that the web UI loads and can display the expected Compose stacks.

### Recovery — Replacement Container Created but Not Started

During a previous Dockman self-update, the existing container stopped and the replacement was created but not started.

The service was successfully restored with:

```
cd /opt/stacks/dockman

sudo docker compose up -d --no-deps dockman
```

If Dockman becomes unavailable after an update, inspect the Compose status and logs through SSH before attempting further changes.

Do not repeatedly perform self-updates through the unavailable UI.

See `services/infra/dockman.md`.

---

## Cleaning Up Old Images

After updating a container, unused images may remain on disk.

Inspect image usage:

```
docker images
```

When cleanup is appropriate:

```
docker image prune -f
```

This removes dangling images that are not associated with a running or stopped container.

**Important:** Do not automatically prune images immediately after an update.

Older images may be needed for troubleshooting or rollback, especially when a service uses a mutable tag such as `latest`.

Wait until the new version has been verified and the relevant rollback and backup requirements are satisfied.

Avoid broad image cleanup commands unless their effect is understood.

---

## Rollback Considerations

A Docker image rollback is not always equivalent to a complete application rollback.

For services without database migrations, restoring an earlier known-good image and configuration may be sufficient.

For services with database or schema changes, downgrading the image may fail or corrupt application state.

### Before Updating

Where practical:

- Record the existing image tag and image ID.
    
- Preserve the previous Compose configuration.
    
- Verify application backups.
    
- Review the upstream rollback guidance.
    
- Keep required older images until the new deployment has been verified.
    

### If an Update Fails

1. Stop further update attempts.
    
2. Check container status and logs.
    
3. Identify whether the failure is related to the image, configuration, network, or persistent application state.
    
4. Review whether a supported rollback path exists.
    
5. Restore the prior image or configuration only when compatible with the current persistent data.
    
6. Verify the service after recovery.
    

For Authentik, Immich, and other database-backed services, consult the appropriate service-specific documentation.

For AI Observer, preserve the separation between the Proxmox MCP container and the OpenAI tunnel client while investigating.

Do not assume that rolling back the MCP container will resolve a problem caused by tunnel authentication or network connectivity.

---

## Stack Locations by Host

|Host|IP|Stacks Path|Management|
|---|---|---|---|
|docker-prod-01|192.168.30.11|`/opt/stacks`|Dockman and CLI|
|auth-prod-01|192.168.30.13|`/opt/stacks`|CLI|
|immich-prod-01|192.168.30.14|`/opt/stacks`|CLI|
|observer-prod-01|192.168.30.17|`/opt/stacks`|CLI only|

### Observer-Specific Services

|Component|Deployment|Management|
|---|---|---|
|proxmox-mcp|Docker Compose|`sudo docker compose`|
|OpenAI tunnel-client|Native systemd|`sudo systemctl`|
|Tunnel configuration|Protected local files|SSH and CLI|

The observer is not managed by the Dockman instance on docker-prod-01.

---

## Notes

- Inspect the Git working tree before pulling configuration changes.
    
- Use `git pull --ff-only` when synchronizing a clean repository.
    
- On observer-prod-01, follow the credential-handling procedure in `runbooks/git-workflow.md`.
    
- A routine image pull does not require a Git commit.
    
- Commit only intentional version-controlled configuration changes.
    
- Do not commit `.env` files or other credentials.
    
- `docker compose up -d` generally recreates only services whose image or configuration changed.
    
- Use `--no-deps` for targeted updates only when dependencies do not require coordinated changes.
    
- Verify applications after updates, not just their container processes.
    
- Authentik updates must follow `runbooks/authentik-update.md`.
    
- Immich updates may require version-specific migration and backup procedures.
    
- Dockman should be updated through the CLI.
    
- Proxmox MCP updates must preserve read-only permissions and localhost binding.
    
- OpenAI tunnel-client updates are separate from Docker Compose.
    
- Do not mark an update successful until its functional checks pass.
    
- Never assume that Git alone provides complete service recovery.
    

---

## Related Documentation

- `runbooks/git-workflow.md` — Version control, multi-host synchronization, and secure GitHub credentials.
    
- `runbooks/new-stack-checklist.md` — Creating and configuring Docker stacks on docker-prod-01.
    
- `runbooks/authentik-update.md` — Authentik sequential upgrade and migration procedure.
    
- `runbooks/backup-schedule.md` — Backup schedules and verification requirements.
    
- `runbooks/ai-observer-operations.md` — AI Observer maintenance, troubleshooting, and recovery
    
- `services/infra/dockman.md` — Dockman deployment and management.
    
- `services/infra/ai-observer.md` — Proxmox MCP and OpenAI Secure MCP Tunnel architecture.