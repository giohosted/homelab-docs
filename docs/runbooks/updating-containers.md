# Updating a Docker Container

**Applies to:** docker-prod-01, auth-prod-01, immich-prod-01, observer-prod-01  
**Last Updated:** 2026-11-14

---

## Overview

Docker containers in Homelab v3 are managed through Docker Compose stacks under `/opt/stacks`.

Updating a container generally involves:

1. Reviewing the target application's release notes.
2. Verifying configuration and backup readiness.
3. Identifying and downloading the intended Docker image.
4. Recreating the affected container.
5. Verifying the application and its dependencies.

Docker Compose updates can be performed through supported management interfaces or directly through the CLI.

Komodo provides centralized update visibility and stack management. Dockman remains documented for supported operations on docker-prod-01.

Not all services should follow the same update procedure. Applications with databases, migrations, or special dependencies may require service-specific runbooks.

**Important:** An image update is different from a Compose configuration change. Pulling a newer image does not require a Git commit unless a version-controlled file was modified.

A successful Docker deployment does not establish that the application is functioning correctly. Operational verification is always required.

---

## Update Management Strategy

### Komodo Update Monitoring

**Verified configuration — 2026-11-14**

Komodo is used to identify available image updates across managed Docker stacks.

The following settings were reviewed and configured:

| Setting | Configuration |
|---|---|
| Poll for Updates | Enabled on managed stacks |
| Auto Update | Disabled on stacks |
| Pre Pull Images | Enabled |
| Full Stack Auto Update | Disabled |
| Global Auto Update schedule | Daily, 03:00 |

The global schedule supports Komodo's update processing, but automatic stack deployment remains disabled under the verified per-stack settings.

**Operational policy:**

- Use Komodo to identify available updates.
- Review release notes and compatibility before deploying.
- Verify application-specific backup requirements.
- Prefer targeted updates for services with independent dependencies.
- Use specialized procedures for Authentik, Immich, reverse proxies, and infrastructure management services.
- Do not enable automatic deployment of critical application updates without a separate risk review.

**Important distinction:** Update notifications, image pre-pulling, and container recreation are different operations.

An available or downloaded image does not mean the running container has been updated.

### Moving Image Tags

Several Homelab v3 applications use moving image tags, including `latest` and application-specific stable-release tags.

Examples include:

- Traefik: `traefik:latest`
- Immich: `release`
- Other applications using `latest` or equivalent rolling tags

Moving tags are intentional where appropriate, but they introduce additional maintenance considerations.

A moving tag can reference a new major version without any change to the Compose file.

Before deploying an update:

1. Identify the currently running application version.
2. Review the version represented by the target tag.
3. Read the relevant migration notes.
4. Verify backup and rollback readiness.
5. Avoid silently pulling an unreviewed image during deployment.

For critical services, consider preserving a locally available version-specific image or recording the previous image digest as part of the rollback plan.

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
- Application-generated secrets, certificates, and persistent identifiers.

An existing backup schedule is not proof that a usable backup exists.

For database-backed or management-critical services, verify recent successful backups before proceeding with nonurgent updates.

**AI Observer:** VM 107 was included in the configured Proxmox backup-job selection, but no archive was visible during the October 30, 2026 UTC verification. Its recoverable backup status must be checked again before a routine update. This historical finding does not establish its present backup status.

See `runbooks/backup-schedule.md`.

### 3. Review the Git Working Tree

On hosts using `homelab-infra`:

```bash
cd /opt/stacks

git status -sb
git status --short --untracked-files=all
```

Before synchronizing, fetch and inspect the remote state:

```bash
git fetch origin

git status -sb

git log --oneline --left-right origin/main...HEAD

git diff --name-only HEAD..origin/main
```

If the working tree is clean and the branch can fast-forward:

```bash
git merge --ff-only origin/main
```

Alternatively, `git pull --ff-only` performs the fetch and fast-forward together.

On observer-prod-01, authenticated Git operations use the credential-helper-disabled procedure described in `runbooks/git-workflow.md`.

If local changes are present, review and reconcile them before pulling.

Do not reset or discard changes merely to make an update proceed.

#### Git Synchronization and Runtime Data

**Operational finding — 2026-11-14**

During synchronization of the four Docker hosts, the repository was discovered tracking:

`beszel-agent/beszel_agent_data/fingerprint`

The fingerprint belongs to host-specific Beszel agent runtime state and should not be distributed between hosts through Git.

The shared repository was corrected to ignore:

```gitignore
/beszel-agent/beszel_agent_data/
```

The previously tracked fingerprint was removed from Git's index without intentionally deleting the original file on the source host.

**Important:** Removing a tracked runtime file from Git can still cause Git to delete that file during a subsequent fast-forward on another host.

On immich-prod-01, the incoming change also removed the now-empty runtime directory. The original fingerprint was recovered from a verified backup, but the running Docker container retained a stale bind mount to the former directory.

The agent was recovered by recreating only the Beszel container and verifying that:

- The host and container bind-mounted directories had matching device and inode identities.
- The container could read the original fingerprint.
- The container was running without restart loops.
- The Beszel dashboard displayed current metrics.

**Prevention:**

- Do not commit generated runtime data or host-specific identifiers.
- Inspect incoming Git deletions before synchronizing production hosts.
- Preserve affected runtime files before an incoming deletion.
- Verify Docker bind mounts after changes that remove or recreate host directories.
- Do not assume a running container is healthy merely because its process remains active.

See `runbooks/git-workflow.md` and `services/monitoring/beszel.md`.

### 4. Check the Current Deployment

From the target stack directory:

```bash
docker compose ps

docker compose images

docker compose config --quiet
```

Use `sudo docker` where Docker access requires elevated privileges.

`docker compose config --quiet` validates the Compose configuration without printing its resolved environment values.

If the current configuration is invalid or the service is already unhealthy, investigate before updating.

For critical infrastructure services, also record the running application version and image reference:

```bash
docker inspect \
  --format '{{.Name}} => {{.Config.Image}}' \
  <container-name>
```

Where necessary, record the image ID or digest separately.

**Important:** `docker compose config --quiet` validates Docker Compose syntax and structure. It does not guarantee that the application inside the image accepts every configuration option.

Application-specific configuration should be validated independently when possible.

---

## Method 1 — Dockman (UI)

**Supported host:** docker-prod-01  
**URL:** `https://dockman.giohosted.com`

Dockman provides a web interface for managing Docker Compose stacks on docker-prod-01.

It does not manage Docker engines on observer-prod-01, auth-prod-01, or immich-prod-01.

Komodo is also available for centralized update monitoring and stack management. The procedures below describe Dockman's established behavior.

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
- Critical infrastructure services requiring controlled image and configuration verification

### Standard Stack Update

SSH into the host running the container.

```bash
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

This is a general procedure. Critical or migration-sensitive applications may require additional checks or a different update sequence.

### Updating One Compose Service

When a stack contains multiple services and only one needs updating:

```bash
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

### Deploying an Already-Reviewed Image

For a critical service using a moving image tag, the new image can be downloaded and inspected before deploying it.

Example:

```bash
docker pull <image>:<tag>
```

Where supported, use the application image to inspect the downloaded version without replacing the production container.

After verification:

```bash
docker compose up -d \
  --no-deps \
  --pull never \
  <service-name>
```

`--pull never` instructs Docker Compose not to download another image during that deployment operation.

This is useful when a moving tag might change between image review and deployment.

It does not guarantee rollback availability unless the previous image is retained separately.

### Important Compose Behavior

- `docker compose pull` downloads images but does not recreate running containers.
- `docker compose up -d` reconciles the stack against its Compose configuration and available images.
- Containers with changed images or configuration may be recreated.
- Unchanged containers are generally left running.
- `docker compose up -d --no-deps <service-name>` restricts the operation to the specified service.
- `docker compose up -d --pull never` avoids pulling a new image during deployment.
- A recreated container may be running before its application is actually ready.

Container status alone is not sufficient verification.

**Git synchronization and Docker deployment are separate operations.**

Pulling a changed Compose file does not automatically apply those changes to existing containers.

---

## Example — Updating the Media Stack on docker-prod-01

**Host:** docker-prod-01  
**IP:** 192.168.30.11

First identify the appropriate Compose stack and service.

For the existing media stack:

```bash
ssh gio@192.168.30.11

cd /opt/stacks/media

docker compose ps
docker compose images
```

For an approved full-stack update:

```bash
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

## Example — Updating Traefik on docker-prod-01

**Host:** docker-prod-01  
**IP:** 192.168.30.11  
**Stack:** `/opt/stacks/traefik`  
**Image:** `traefik:latest`

Traefik provides reverse-proxy and TLS services for multiple Homelab applications. Its updates should be deliberate because recreating the container temporarily interrupts HTTP and HTTPS routing.

**Verified implementation:** On 2026-11-14, Traefik was upgraded from v3.6.25 to v3.7.19.

The image tag was changed from `traefik:v3.6` to `traefik:latest`.

The upgrade was followed by additional HTTP header-alias and encoded-character security hardening.

See `services/infra/traefik.md` for the complete deployed configuration and implementation record.

### Step 1 — Review the Target Release

Identify the current running version:

```bash
sudo docker exec traefik traefik version
```

Read the target release notes and migration documentation.

Because `latest` is a moving tag, do not assume the next available image belongs to the same major release.

### Step 2 — Verify Persistent Data and Backups

Identify the critical configuration:

| Path | Purpose |
|---|---|
| `/opt/stacks/traefik/compose.yaml` | Compose configuration |
| `/opt/stacks/traefik/.env` | Cloudflare API token |
| `/opt/appdata/traefik/acme.json` | TLS certificate state and private keys |
| `/opt/appdata/traefik/config/` | Dynamic routing configuration |

Verify that suitable backups are available.

The November 14 upgrade used a root-protected archive containing these files.

Backup contents containing `.env` and certificate private keys must not be committed to Git or stored in an unprotected location.

### Step 3 — Establish a Routing Baseline

Before changing Traefik, verify important application routes and TLS behavior.

Example:

```bash
for host in \
  auth.giohosted.com \
  photos.giohosted.com
do
  echo "===== $host ====="

  curl --noproxy '*' -sS \
    --connect-timeout 5 \
    --max-time 15 \
    --resolve "$host:443:127.0.0.1" \
    -o /dev/null \
    -w 'HTTP=%{http_code} TLS=%{ssl_verify_result}\n' \
    "https://$host/"
done
```

The November 14 baseline was:

- Authentik: HTTP `302`
- Immich: HTTP `200`
- TLS verification: `0` for both

These values are historical expected responses, not a guarantee that future application versions will return identical status codes.

Also verify any other services relevant to the planned maintenance.

### Step 4 — Download and Inspect the Target Image

```bash
cd /opt/stacks/traefik

sudo docker pull traefik:latest
```

Inspect the downloaded version without replacing production:

```bash
sudo docker run --rm \
  --network none \
  --entrypoint traefik \
  traefik:latest version
```

On November 14, this confirmed v3.7.19 before deployment.

### Step 5 — Validate Configuration Changes

If a version-controlled Compose change is required:

```bash
sudo docker compose config --quiet

git -C /opt/stacks diff -- traefik/compose.yaml
```

For newly introduced static Traefik options, use a safe application-level validation method where practical.

Compose validation alone does not establish that the target Traefik binary recognizes every command-line flag.

Commit and push reviewed Compose changes before deployment when practical.

Never stage `.env`, certificate storage, or other secrets.

### Step 6 — Deploy Traefik

Once the image and configuration have been reviewed:

```bash
sudo docker compose up -d \
  --no-deps \
  --pull never \
  traefik
```

This recreates only the Traefik Compose service.

Expect a brief interruption to reverse-proxy access.

### Step 7 — Verify the Deployment

```bash
sudo docker compose ps traefik

sudo docker exec traefik traefik version

sudo docker logs --tail 100 traefik
```

Verify:

- The expected version is running.
- The container is not restarting repeatedly.
- Docker and file providers initialize correctly.
- ACME configuration loads without errors.
- TLS certificates validate.
- Previously working routes continue functioning.
- Relevant applications work through their normal browser interfaces.

After the November 14 upgrade and hardening:

- Authentik continued returning HTTP `302`.
- Immich continued returning HTTP `200`.
- Both applications worked normally in a browser.
- Encoded null characters (`%00`) and backslashes (`%5C`) returned HTTP `400`.
- The relevant security startup warnings were no longer present.

### Rollback

If Traefik fails after upgrading:

1. Stop further changes.
2. Inspect container startup errors.
3. Confirm the previous image is still available.
4. Review the previous Compose configuration.
5. Determine whether certificate or persistent configuration recovery is necessary.
6. Restore a compatible known-good configuration or image.
7. Recreate only Traefik.
8. Repeat TLS, routing, and application verification.

Do not blindly overwrite `acme.json` when recovering an unrelated configuration problem.

**Important:** Preserving the previous image and backup provides a possible rollback path. It does not establish that a complete rollback was tested.

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

**Verified deployment — 2026-11-14:**

Authentik was upgraded through the required intermediate versions to `2026.8.5`.

The final deployment was verified using:

- Docker Compose's resolved image configuration.
- Running server and worker image references.
- Authentik's administrative interface.
- A functional Immich SSO sign-in test.

The runtime version is explicitly provided through the Git-excluded `.env` file using `AUTHENTIK_TAG`.

The shared Compose fallback was also updated to `2026.8.5`.

Do not commit the Authentik `.env` file or assume changing a fallback tag automatically upgrades the running application.

See `services/infra/authentik.md` and `runbooks/authentik-update.md`.

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

**Additional reverse-proxy consideration:** Immich uses Traefik for HTTPS routing. When changing reverse-proxy timeouts, URL handling, request headers, or other shared entrypoint behavior, verify Immich functionality separately from container health.

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

The current documented deployment uses the `latest` image tag.

Because `latest` is mutable, record the existing image ID before pulling a replacement. A prior Compose file alone may not identify the previously deployed image.

### Step 1 — Connect to the Observer

```bash
ssh gio@192.168.30.17
```

Navigate to the MCP stack:

```bash
cd /opt/stacks/proxmox-mcp
```

### Step 2 — Inspect the Existing Deployment

```bash
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

```bash
curl -fsS http://127.0.0.1:8080/health
```

Expected response:

```json
{"status":"ok"}
```

Check the tunnel service:

```bash
sudo systemctl status tunnel-client --no-pager
```

Check tunnel readiness:

```bash
curl -fsS http://127.0.0.1:8081/readyz
```

Expected response:

```text
ready
```

If either component is unhealthy before the update, investigate the cause first.

### Step 4 — Pull the Updated MCP Image

After reviewing the release and confirming backup readiness:

```bash
cd /opt/stacks/proxmox-mcp

sudo docker compose pull proxmox-mcp
```

This downloads the image referenced by the existing Compose configuration.

It does not restart the running container.

### Step 5 — Recreate the MCP Container

```bash
sudo docker compose up -d --no-deps proxmox-mcp
```

The service name is `proxmox-mcp`.

This updates the MCP container without deliberately restarting the separate OpenAI tunnel systemd service.

There may still be a brief interruption to MCP requests while the container is recreated.

### Step 6 — Verify the Updated Container

```bash
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

```bash
sudo systemctl status tunnel-client --no-pager

curl -fsS http://127.0.0.1:8081/readyz
```

Review tunnel logs if necessary:

```bash
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

```bash
ssh gio@192.168.30.11
```

Navigate to its stack:

```bash
cd /opt/stacks/dockman
```

Inspect the current deployment:

```bash
docker compose ps
docker compose images
```

After reviewing the release and confirming configuration backup readiness, update the Dockman service:

```bash
docker compose pull dockman

docker compose up -d --no-deps dockman
```

Verify:

```bash
docker compose ps

docker compose logs --tail 50 dockman
```

Finally, reopen:

`https://dockman.giohosted.com`

Confirm that the web UI loads and can display the expected Compose stacks.

### Recovery — Replacement Container Created but Not Started

During a previous Dockman self-update, the existing container stopped and the replacement was created but not started.

The service was successfully restored with:

```bash
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

```bash
docker images
```

When cleanup is appropriate:

```bash
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

| Host | IP | Stacks Path | Management |
|---|---|---|---|
| docker-prod-01 | 192.168.30.11 | `/opt/stacks` | Komodo, Dockman, and CLI |
| auth-prod-01 | 192.168.30.13 | `/opt/stacks` | Komodo and CLI |
| immich-prod-01 | 192.168.30.14 | `/opt/stacks` | Komodo and CLI |
| observer-prod-01 | 192.168.30.17 | `/opt/stacks` | CLI; Komodo integration where configured |

Management interface availability and permissions should be confirmed before using a UI for a consequential operation.

### Observer-Specific Services

| Component | Deployment | Management |
|---|---|---|
| proxmox-mcp | Docker Compose | `sudo docker compose` |
| OpenAI tunnel-client | Native systemd | `sudo systemctl` |
| Tunnel configuration | Protected local files | SSH and CLI |

Dockman on docker-prod-01 does not manage the observer.

---

## Notes

- Inspect the Git working tree before pulling configuration changes.
- Use fast-forward-only Git synchronization when the branch history permits it.
- Review incoming deletions that could affect persistent runtime data or Docker bind mounts.
- On observer-prod-01, follow the credential-handling procedure in `runbooks/git-workflow.md`.
- Komodo update polling does not replace release-note review, backup validation, or post-deployment testing.
- A routine image pull does not require a Git commit.
- Commit only intentional version-controlled configuration changes.
- Do not commit `.env` files, certificates, credentials, or generated runtime state.
- `docker compose up -d` generally recreates only services whose image or configuration changed.
- Use `--no-deps` for targeted updates only when dependencies do not require coordinated changes.
- Use `--pull never` when deploying an image that was deliberately downloaded and reviewed in advance.
- Verify applications after updates, not just their container processes.
- Authentik updates must follow `runbooks/authentik-update.md`.
- Immich updates may require version-specific migration and backup procedures.
- Traefik updates require routing, TLS, and application verification.
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
- `runbooks/ai-observer-operations.md` — AI Observer maintenance, troubleshooting, and recovery.
- `services/infra/traefik.md` — Reverse-proxy configuration, verified upgrade, security hardening, and rollback.
- `services/monitoring/beszel.md` — Beszel hub, agents, and persistent runtime state.
- `services/infra/dockman.md` — Dockman deployment and management.
- `services/infra/ai-observer.md` — Proxmox MCP and OpenAI Secure MCP Tunnel architecture.