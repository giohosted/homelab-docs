# AI Observer — Operations & Recovery Runbook

**Service:** AI Observer  
**Host:** observer-prod-01 (`192.168.30.17`)  
**Proxmox VMID:** 107  
**Proxmox Node:** pve-prod-01  
**Environment:** Homelab v3  
**Last Updated:** 2026-10-30  
**Service Status:** Proxmox and Unraid MCP integrations implemented and end-to-end verified  
**Backup Status:** October 30 scheduled backup successful; integrity and restore testing pending

---

## Purpose

This runbook provides operational procedures for managing the Homelab v3 AI Observer.

It covers:

- Routine service health checks.
    
- Verifying ChatGPT-to-Proxmox and ChatGPT-to-Unraid connectivity.
    
- Managing the Proxmox and Unraid MCP containers.
    
- Managing two OpenAI Secure MCP Tunnel services using one installed tunnel-client binary.
    
- Troubleshooting common failures.
    
- Reviewing security and permissions.
    
- Performing controlled updates.
    
- Handling credentials and token rotation.
    
- Preparing for backup and disaster recovery.
    
- Validating recovery and returning the service to operation.
    

This document is an operational reference, not a deployment history or architecture specification.

For implementation details, component configuration, security design, and architectural decisions, refer to:

`services/infra/ai-observer.md`

### Operational Principles

1. Prefer read-only investigation before making changes.
    
2. Make one consequential change at a time.
    
3. Verify the effect of each change before proceeding.
    
4. Do not expose infrastructure credentials in logs, documentation, screenshots, or chat.
    
5. Preserve the read-only security model.
    
6. Do not expand network access or permissions during routine troubleshooting.
    
7. Verify complete end-to-end operation rather than relying only on process status.
    
8. Do not claim backup or recovery readiness until the relevant procedures have been tested.
    

---

## 1. Component Reference

### Infrastructure

|Component|Configuration|
|---|---|
|VM|observer-prod-01|
|Proxmox VMID|107|
|Proxmox node|pve-prod-01|
|Operating system|Ubuntu Server 26.04.1 LTS|
|IP address|192.168.30.17/24|
|VLAN|30 — Services|
|CPU|2 vCPU|
|Memory|2 GiB|
|Disk|20 GiB|
|Primary administration|SSH|
|Administrative user|gio|

### Services

|Component|Technology|Management|
|---|---|---|
|Proxmox MCP|Docker Compose|`sudo docker compose`|
|Unraid MCP (Rust 0.6.3)|Docker Compose|`sudo docker compose` in `/opt/stacks/unraid-mcp`|
|OpenAI Secure MCP Tunnels (Proxmox and Unraid)|Two native systemd services|`sudo systemctl`|
|ChatGPT integration|Homelab Proxmox and Homelab Unraid plugins|ChatGPT Plugins|
|Proxmox API|Proxmox VE HTTPS API|Dedicated read-only API identity|
|Unraid API|Unraid GraphQL HTTPS API|Dedicated 14-category READ_ANY API key|

### Key Paths

|Path|Purpose|
|---|---|
|`/opt/stacks/proxmox-mcp/`|MCP Compose stack|
|`/opt/stacks/proxmox-mcp/compose.yaml`|Version-controlled Compose configuration|
|`/opt/stacks/proxmox-mcp/.env`|MCP environment configuration and credentials|
|`/opt/stacks/unraid-mcp/compose.yaml`|Unraid MCP Compose stack|
|`/opt/stacks/unraid-mcp/.env`|Protected Unraid GraphQL API key and MCP bearer token|
|`/etc/unraid-mcp/unraid.crt`|Trusted self-signed Unraid server certificate|
|`/opt/tunnel-client/`|Installed OpenAI tunnel-client files|
|`/opt/tunnel-client/tunnel-client`|Tunnel-client executable|
|`/etc/tunnel-client/config.yaml`|Tunnel configuration|
|`/etc/tunnel-client/openai-runtime.key`|Restricted OpenAI runtime credential|
|`/etc/tunnel-client/mcp-authorization`|Local MCP bearer authorization|
|`/etc/systemd/system/tunnel-client.service`|Proxmox tunnel systemd service unit|
|`/etc/tunnel-client-unraid/config.yaml`|Unraid tunnel configuration|
|`/etc/tunnel-client-unraid/openai-runtime.key`|Unraid OpenAI runtime key|
|`/etc/tunnel-client-unraid/mcp-authorization`|Unraid MCP bearer header|
|`/etc/systemd/system/tunnel-client-unraid.service`|Unraid tunnel systemd service unit|

**Important:** Proxmox and Unraid each use a separate MCP container and a separate tunnel systemd service; both tunnel processes reuse the existing `tunnel-client` account and installed binary.

A healthy tunnel does not necessarily mean the MCP server is healthy, and a healthy MCP server does not guarantee that ChatGPT can reach it.

---

## 2. Normal Operating State

AI Observer is considered operational when all required components are available and a read-only query succeeds end-to-end.

### Expected State

|Check|Expected Result|
|---|---|
|Observer VM|Running|
|Docker engine|Available|
|Proxmox MCP container|Running|
|Unraid MCP container|Running and healthy|
|MCP health endpoint|HTTP 200|
|Tunnel systemd services|Both active and enabled|
|Tunnel readiness endpoint|HTTP 200|
|Local MCP authentication|Functioning|
|Proxmox API connection|Reachable and authenticated|
|Unraid GraphQL API connection|Reachable and authenticated with TLS verification|
|ChatGPT integration|Connected|
|Live MCP tool call|Successfully returns Proxmox and Unraid information using the respective plugins|
|MCP permissions|Read-only; Unraid exposes 26 read / 0 write tools|

### End-to-End Request Path

```
ChatGPT
  | Homelab Proxmox plugin -> outbound tunnel -> tunnel-client.service
  |                             -> 127.0.0.1:8080/mcp
  |                             -> Proxmox API 192.168.10.11:8006
  |
  | Homelab Unraid plugin  -> outbound tunnel -> tunnel-client-unraid.service
                                -> 127.0.0.1:8082/mcp
                                -> Unraid HTTPS API 192.168.30.16:443
```


MCP endpoints: Proxmox `http://127.0.0.1:8080/mcp`; Unraid `http://127.0.0.1:8082/mcp`.

Tunnel readiness endpoints: Proxmox `http://127.0.0.1:8081/readyz`; Unraid `http://127.0.0.1:8083/readyz`.

Neither endpoint requires a public DNS record or inbound internet port forwarding.

---

## 3. Routine Health Checks

These checks are read-only and are appropriate for initial troubleshooting.

### Step 1 — Connect to the Observer

From an authorized administrative workstation:

```
ssh gio@192.168.30.17
```

SSH access uses key-based authentication.

Direct root SSH login and SSH password authentication were disabled during deployment.

If SSH fails, investigate network and VM availability before changing SSH configuration.

### Step 2 — Check MCP Container Status

```
cd /opt/stacks/proxmox-mcp

sudo docker compose ps
```

Expected result: The `proxmox-mcp` container is running.

Check recent container logs:

```
sudo docker compose logs --tail 50 proxmox-mcp
```

Look for:

- Startup errors.
    
- Configuration validation failures.
    
- Authentication failures.
    
- Proxmox API connection errors.
    
- TLS verification failures.
    
- Unexpected container restarts.
    

**Security:** Logs may contain sensitive operational information. Review and redact them before sharing externally.

### Step 3 — Check MCP Health

```
curl -fsS --max-time 5 http://127.0.0.1:8080/health
```

Expected response:

```
{"status":"ok"}
```

This establishes that the MCP HTTP health endpoint is responding.

It does not independently verify Proxmox authentication, API permissions, or ChatGPT connectivity.

### Step 4 — Check Tunnel Service

```
sudo systemctl status tunnel-client --no-pager
```

Expected result: `active (running)`.

Check whether the service is enabled at boot:

```
systemctl is-enabled tunnel-client
```

Expected result:

```
enabled
```

Inspect recent logs:

```
sudo journalctl -u tunnel-client -n 50 --no-pager
```

Look for authentication failures, connection errors, repeated restarts, and configuration problems.

### Step 5 — Check Tunnel Readiness

```
curl -fsS --max-time 5 http://127.0.0.1:8081/readyz
```

Expected response:

```
ready
```

A successful readiness response confirms the tunnel client reports itself ready.

It does not replace a successful live MCP tool invocation.

### Step 6 — Verify ChatGPT Access

In ChatGPT, use the Homelab Proxmox plugin to request:

> Check the current Proxmox cluster status, including quorum and the status of both nodes.

A successful response should contain information retrieved through the live MCP integration.

The initially verified cluster configuration included:

- Cluster name: `homelab-v3`
    
- Two Proxmox nodes.
    
- Proxmox VE 9.1.6 at initial deployment.
    
- Cluster quorum present.
    

Versions and operational state may change over time. Treat these as historical reference values, not guaranteed current results.

**This is the final end-to-end health check for Proxmox.**

### Additional Check — Unraid MCP and Tunnel

On `observer-prod-01`, inspect the second integration independently (read-only commands):

```bash
cd /opt/stacks/unraid-mcp
sudo docker compose ps
sudo docker compose logs --tail 50 --no-color unraid-mcp
sudo docker compose config --quiet
systemctl is-active tunnel-client-unraid.service
systemctl is-enabled tunnel-client-unraid.service
curl -fsS --max-time 10 http://127.0.0.1:8083/readyz
```

Expected: healthy `unraid-mcp`, active and enabled Unraid tunnel, `ready` response. The container logs should show `trusting extra CA certificates from bundle count=1`. The local MCP endpoint normally returns HTTP 401 without a bearer token; that proves it is reachable, **not** that upstream GraphQL authentication is working.

Then, in ChatGPT, call the `Homelab Unraid` plugin with `unraid_info`. Confirm it returns live `nas-prod-01` system information. This end-to-end check is required before closing an Unraid MCP incident. Do not call write/management actions as a connectivity test.

---

## 4. Quick Diagnostic Command Reference

Run these commands on observer-prod-01 unless otherwise noted.

### Docker and MCP

```
cd /opt/stacks/proxmox-mcp

sudo docker compose ps
sudo docker compose images
sudo docker compose config --quiet
sudo docker compose logs --tail 100 proxmox-mcp

curl -fsS --max-time 5 http://127.0.0.1:8080/health
```

### Tunnel Client

```
sudo systemctl status tunnel-client --no-pager

systemctl is-enabled tunnel-client

sudo journalctl -u tunnel-client -n 100 --no-pager

curl -fsS --max-time 5 http://127.0.0.1:8081/readyz

# Unraid
sudo systemctl status tunnel-client-unraid --no-pager
systemctl is-enabled tunnel-client-unraid
curl -fsS --max-time 5 http://127.0.0.1:8083/readyz
```

### Operating System Resources

```
uptime

free -h

df -h /

df -h /opt

sudo docker stats --no-stream proxmox-mcp unraid-mcp
```

### Network

```
ip -br addr show ens18

ip route

ip route get 192.168.10.11

getent hosts api.openai.com
```

### Local Listening Ports

```
sudo ss -lntp
```

Verify that the MCP and tunnel readiness services are bound to localhost rather than all interfaces.

Expected local endpoints:

- `127.0.0.1:8080` — Proxmox MCP.
    
- `127.0.0.1:8081` — Proxmox tunnel readiness.
- `127.0.0.1:8082` — Unraid MCP.
- `127.0.0.1:8083` — Unraid tunnel readiness.
    

### Git Configuration

```
cd /opt/stacks

git status

git log --oneline -5
```

Do not run Git commands that overwrite local changes as part of basic diagnostics.

---

## 5. Troubleshooting Workflow

When ChatGPT cannot retrieve Proxmox or Unraid data, first identify the affected plugin and investigate its independent container, tunnel, and upstream API path in order.

### Diagnostic Sequence

1. Is observer-prod-01 running?
    
2. Can the observer be reached over SSH?

3. Which integration is affected — Proxmox, Unraid, or both?
    
4. Is the affected MCP container running?
    
5. Does its local health/MCP endpoint respond?
    
6. Is the corresponding tunnel-client systemd service running?
    
7. Does that tunnel readiness endpoint respond?
    
8. Can the MCP service reach and authenticate to its upstream API?
    
9. Does ChatGPT recognize and invoke the intended plugin?
    
10. Does a live read-only tool call succeed?
    

Do not restart all components simultaneously.

Use the failed check to identify the affected layer.

---

## 6. Troubleshooting — MCP Container Unavailable

### Symptoms

- Docker reports the MCP container as stopped or restarting.
    
- `127.0.0.1:8080/health` does not respond.
    
- Tunnel requests fail because the local MCP endpoint is unavailable.
    

### Investigation

```
cd /opt/stacks/proxmox-mcp

sudo docker compose ps

sudo docker compose logs --tail 100 proxmox-mcp

sudo docker compose config --quiet
```

Check Docker service status:

```
sudo systemctl status docker --no-pager
```

Check available resources:

```
free -h
df -h /
```

### Common Causes to Investigate

- Docker engine unavailable.
    
- Invalid Compose configuration.
    
- Missing or unreadable `.env` file.
    
- Invalid environment variables.
    
- Container image problems.
    
- Port-binding conflict.
    
- Memory or disk exhaustion.
    

### Recovery

After identifying and addressing the cause, an authorized administrator may start or recreate the affected service:

```
cd /opt/stacks/proxmox-mcp

sudo docker compose up -d --no-deps proxmox-mcp
```

This is a state-changing operation, not a read-only check.

After recovery, verify:

```
sudo docker compose ps

curl -fsS http://127.0.0.1:8080/health
```

Then confirm tunnel readiness and perform a live ChatGPT MCP query.

Do not assume that recreating the container resolves configuration or credential problems.

---

## 7. Troubleshooting — Tunnel Client Unavailable

### Symptoms

- `tunnel-client.service` is inactive or failed.
    
- Tunnel readiness endpoint does not respond.
    
- ChatGPT cannot reach MCP despite a healthy local MCP container.
    
- Systemd logs show repeated reconnect or authentication failures.
    

### Investigation

```
sudo systemctl status tunnel-client --no-pager

sudo journalctl -u tunnel-client -n 100 --no-pager

systemctl is-enabled tunnel-client
```

Check local readiness:

```
curl -fsS --max-time 5 http://127.0.0.1:8081/readyz
```

Confirm that the MCP service is available independently:

```
curl -fsS --max-time 5 http://127.0.0.1:8080/health
```

### Common Causes to Investigate

- Incorrect tunnel configuration.
    
- Expired, revoked, or invalid OpenAI runtime credential.
    
- Incorrect MCP Authorization header.
    
- Inaccessible credential files.
    
- Missing or incompatible tunnel-client binary.
    
- Outbound HTTPS connectivity failure.
    
- DNS resolution problems.
    
- Local MCP endpoint unavailable.
    

### Recovery

Once the cause is identified and the proposed action is approved, a controlled service restart may be appropriate:

```
sudo systemctl restart tunnel-client
```

Verify:

```
sudo systemctl status tunnel-client --no-pager

curl -fsS http://127.0.0.1:8081/readyz
```

Review logs after the restart:

```
sudo journalctl -u tunnel-client -n 50 --no-pager
```

Finish with an end-to-end ChatGPT MCP query.

Do not repeatedly restart the service when authentication or configuration errors remain unresolved.

---

## 8. Troubleshooting — Proxmox API Connectivity

### Symptoms

- MCP health responds but Proxmox tool calls fail.
    
- Connection timeout to Proxmox.
    
- Authentication or permission errors.
    
- TLS certificate verification failures.
    

### Network Path

|Source|Destination|Protocol|Port|
|---|---|---|---|
|observer-prod-01 — 192.168.30.17|pve-prod-01 — 192.168.10.11|TCP|8006|

The observer resides on VLAN 30, while the Proxmox management interface resides on VLAN 10.

A targeted UniFi firewall exception permits this connection.

### Investigation

Check observer networking:

```
ip -br addr show ens18

ip route get 192.168.10.11
```

Check MCP logs:

```
cd /opt/stacks/proxmox-mcp

sudo docker compose logs --tail 100 proxmox-mcp
```

If `nc` is installed, check TCP reachability:

```
nc -vz -w 5 192.168.10.11 8006
```

A successful TCP connection demonstrates reachability to the port. It does not prove that Proxmox authentication or TLS fingerprint verification is working.

### Firewall Verification

Confirm the documented rule exists and is evaluated before the general Services-to-Management deny rule.

**Documented rule:**

- Name: `Allow Observer to Proxmox API`
    
- Source: `192.168.30.17`
    
- Destination: `192.168.10.11`
    
- Destination port: TCP 8006
    
- Action: Allow
    

Do not substitute a broad VLAN 30-to-VLAN 10 allowance.

### Authentication Errors

If the API is reachable but authentication fails:

1. Verify the dedicated Proxmox account exists.
    
2. Verify the API token is still valid.
    
3. Confirm the MCP configuration references the expected token ID.
    
4. Confirm the secret is available to the container.
    
5. Review recent permission or token changes.
    

**Account:** `observer@pve`  
**Token ID:** `observer@pve!proxmox-mcp`  
**Role:** `PVEAuditor`

Do not print the token secret into logs or chat.

### TLS Fingerprint Errors

The MCP connection uses a pinned certificate fingerprint.

If Proxmox certificates change, a fingerprint mismatch may prevent connection.

Before updating a fingerprint:

1. Confirm the current Proxmox certificate through an authorized administrative interface.
    
2. Verify that any certificate change was expected.
    
3. Obtain and independently validate the replacement fingerprint.
    
4. Update the protected MCP configuration through the approved change process.
    
5. Recreate the MCP container if required.
    
6. Verify authenticated access.
    

Do not disable TLS verification as a shortcut.

---

## 9. Troubleshooting — ChatGPT Plugin

### Symptoms

- Homelab Proxmox does not appear as an available plugin.
    
- The plugin appears connected but tool calls fail.
    
- The plugin cannot retrieve live cluster information.
    
- Unexpected authentication or MCP tool-discovery errors occur.
    

### Investigation

First confirm the observer's local services are healthy:

```
curl -fsS http://127.0.0.1:8080/health

curl -fsS http://127.0.0.1:8081/readyz
```

Then inspect the tunnel logs:

```
sudo journalctl -u tunnel-client -n 100 --no-pager
```

In ChatGPT:

1. Confirm the Homelab Proxmox plugin remains connected.
    
2. Confirm it references the intended `homelab-proxmox` tunnel.
    
3. Retry a basic read-only cluster-status request.
    
4. Determine whether the failure affects all tools or a specific operation.
    

### Authentication Distinction

The ChatGPT plugin was configured with the **No authentication** option because it does not require a separate end-user OAuth flow.

This does not mean the local MCP endpoint is unauthenticated.

The tunnel client supplies the configured bearer token when forwarding requests to the MCP server.

Do not remove the MCP Authorization requirement to resolve a ChatGPT connection problem.

### Tool-Specific Errors

An individual tool failure does not necessarily indicate that the tunnel is broken.

Examples include:

- API privileges insufficient for a particular endpoint.
    
- Unsupported operation.
    
- Invalid tool arguments.
    
- Proxmox endpoint unavailable.
    
- Target guest not running.
    
- Guest-agent functionality unavailable.
    

Investigate the individual error before making changes to the global integration.

A read-only API account may legitimately be denied access to endpoints requiring administrative privileges. The Unraid API key is not intended for every potentially readable GraphQL endpoint, only the intended monitoring categories.

Do not grant administrator permissions merely to eliminate those errors.

---

### Unraid-Specific Failures

Use Section 21 for detailed Unraid MCP and tunnel commands. Diagnose independently of Proxmox:

- Rust MCP unavailable or port `8082` not responding: inspect `/opt/stacks/unraid-mcp/` Compose state and container logs.
- TLS trust failure: verify `UNRAID_API_CA_BUNDLE=/certs/unraid.crt`, the certificate mount, and `UNRAID_API_SKIP_TLS_VERIFY=false`. Do not disable certificate verification.
- HTTP 401 from local MCP without the bearer token is expected; an authenticated tool call or ChatGPT query is the functional test.
- Upstream `forbidden` / permission errors for an **individual tool** may mean the dedicated 14-category `READ_ANY` key does not grant that GraphQL field. Investigate before expanding permissions.
- Tool missing from ChatGPT: compare the actual 26-name tool inventory with the `UNRAID_RMCP_ENABLED_TOOLS` policy and check plugin tool refresh. Never remove the allowlist as a diagnostic shortcut.
- The Unraid array currently has a disabled first parity drive. This is a **NAS hardware/storage incident**, not an MCP fault; it is deliberately unresolved pending a separate controlled repair workflow in `architecture/storage/nas-prod-01.md`.

---

## 10. Service Management

AI Observer uses Docker Compose for its two MCP containers and native systemd services for its two secure tunnels. The Unraid and Proxmox services should be managed independently.

### Proxmox MCP — Docker Compose

**Directory:** `/opt/stacks/proxmox-mcp`

Read-only status:

```
cd /opt/stacks/proxmox-mcp

sudo docker compose ps
```

View logs:

```
sudo docker compose logs --tail 100 proxmox-mcp
```

Authorized service restart:

```
sudo docker compose restart proxmox-mcp
```

Authorized service recreation:

```
sudo docker compose up -d --no-deps proxmox-mcp
```

### Tunnel Client — systemd

Read-only status:

```
sudo systemctl status tunnel-client --no-pager
```

View logs:

```
sudo journalctl -u tunnel-client -n 100 --no-pager
```

Authorized restart:

```
sudo systemctl restart tunnel-client
```

### Unraid MCP — Docker Compose

Read-only checks:

```bash
cd /opt/stacks/unraid-mcp
sudo docker compose ps
sudo docker compose logs --tail 100 --no-color unraid-mcp
sudo docker compose config --quiet
```

Only with approval after identifying a fault:

```bash
cd /opt/stacks/unraid-mcp
sudo docker compose up -d --no-deps unraid-mcp
```

This affects only the Unraid MCP stack, not the NAS array or Proxmox MCP. Consult Section 21 for updates and verification.

### Unraid Tunnel — systemd

```bash
systemctl is-active tunnel-client-unraid.service
systemctl is-enabled tunnel-client-unraid.service
sudo journalctl -u tunnel-client-unraid -n 100 --no-pager
curl -fsS --max-time 10 http://127.0.0.1:8083/readyz
```

After confirming a tunnel-specific failure and receiving approval, restart only `tunnel-client-unraid.service`, then recheck readiness and a live ChatGPT tool call. Both tunnel services use `/opt/tunnel-client/tunnel-client`; do not upgrade that shared binary while debugging only one integration.


### Restart Order

If both services require recovery:

1. Identify why each service is unhealthy.
    
2. Restore Proxmox MCP availability.
    
3. Verify the MCP health endpoint.
    
4. Restore or verify the tunnel client.
    
5. Verify tunnel readiness.
    
6. Perform an end-to-end ChatGPT tool call.
    

Do not routinely restart both services when only one component is affected.

---

## 11. Updating Proxmox MCP

The MCP image currently uses:

`ghcr.io/mattoddie/proxmox-mcp:latest`

This is a mutable image tag. A newer image may be downloaded without any Compose file change.

### Before Updating

1. Review the upstream project and release notes.
    
2. Check the running version or image ID.
    
3. Confirm the required functionality remains supported.
    
4. Verify that a suitable backup or recovery procedure is available.
    
5. Confirm the existing MCP and tunnel health checks pass.
    
6. Confirm the update does not introduce new permissions or externally exposed listeners.
    

**Known limitation:** The scheduled Proxmox backup system failed on October 29, 2026 but completed successfully on both nodes on October 30 after an approved PBS proxy restart. Full recovery validation remains outstanding.

Do not assume that observer-prod-01 has a usable backup until PBS coverage has been verified.

### Update Procedure

Follow the established Docker update procedure in:

`runbooks/updating-containers.md`

The observer-specific commands are:

```
cd /opt/stacks/proxmox-mcp

sudo docker compose images

sudo docker compose config --quiet
```

After approval and backup verification:

```
sudo docker compose pull proxmox-mcp

sudo docker compose up -d --no-deps proxmox-mcp
```

### Post-Update Verification

```
sudo docker compose ps

sudo docker compose logs --tail 100 proxmox-mcp

curl -fsS http://127.0.0.1:8080/health
```

Then check tunnel readiness:

```
curl -fsS http://127.0.0.1:8081/readyz
```

Finish with a live ChatGPT MCP query.

### Security Verification

Confirm that the deployed configuration still has:

- `PROXMOX_ALLOW_WRITES=false`
    
- `PROXMOX_ALLOW_DELETES=false`
    
- `PROXMOX_ALLOW_EXEC=false`
    
- MCP listener bound to localhost.
    
- No Docker socket mount.
    
- Dedicated Proxmox API identity.
    
- TLS fingerprint verification enabled.
    

Do not expand privileges as part of routine updates.

---

## 12. Updating OpenAI Tunnel Client

The OpenAI tunnel client is installed as a native Linux binary.

**Binary:** `/opt/tunnel-client/tunnel-client`  
**Service:** `tunnel-client.service`

It is not updated through Docker Compose.

### Current Documented Version

`0.0.18`

This version was installed from an official release with a verified SHA-256 checksum.

### Required Update Process

1. Confirm the current installed version.
    
2. Review the official release notes.
    
3. Identify the correct Linux AMD64 release artifact.
    
4. Download the release through an approved channel.
    
5. Verify the artifact's published checksum.
    
6. Confirm compatibility with the existing tunnel configuration.
    
7. Preserve the current working binary and configuration for rollback.
    
8. Install the verified replacement.
    
9. Restart the systemd service through an approved maintenance action.
    
10. Verify readiness and end-to-end ChatGPT functionality.
    

**Official project:**

`https://github.com/openai/tunnel-client`

Use the release-specific official installation instructions rather than assuming future packages retain the same filenames or installation layout.

The deployed release also includes a matching `cloudflared` binary and manifest.

Do not update only one part of the release bundle unless the upstream release instructions explicitly support doing so.

### Verification

```
sudo systemctl status tunnel-client --no-pager

curl -fsS http://127.0.0.1:8081/readyz

sudo journalctl -u tunnel-client -n 50 --no-pager
```

Then perform a live Proxmox query through ChatGPT.

### Rollback

If a tunnel-client update fails:

- Review the systemd logs.
    
- Identify whether the failure is configuration-related, credential-related, or binary-related.
    
- Confirm the previous binary and required supporting files are available.
    
- Restore the previous known-good release only when compatible with the current configuration.
    
- Verify service health and ChatGPT integration afterward.
    

Do not overwrite protected credential files during routine binary updates.

---

## 13. Git Workflow

The observer uses the private `homelab-infra` repository.

**Local repository:** `/opt/stacks`

Only the intended infrastructure configuration is committed.

The Proxmox MCP `.env` file remains excluded from Git.

### Read-Only Git Checks

```
cd /opt/stacks

git status

git log --oneline -5
```

### Synchronizing Configuration

Before modifying version-controlled files:

```
cd /opt/stacks

git status
```

When the working tree is clean, an authorized synchronization may use:

```
git -c credential.helper= pull --ff-only
```

This disables configured credential helpers for that operation and avoids deliberately persisting GitHub authentication through the plaintext credential helper.

Do not put a GitHub token directly in a shell command or remote URL.

### Changes and Commits

Follow:

`runbooks/git-workflow.md`

Do not use `git reset --hard`, force-push, or overwrite local modifications as routine troubleshooting steps.

Git configuration does not replace backup of credentials, systemd units, or other host-level files.

---

## 14. Credential Management and Rotation

AI Observer uses several independent credentials.

|Credential|Purpose|Storage|
|---|---|---|
|Proxmox API token|Read-only Proxmox API access|Protected MCP `.env` and Bitwarden|
|Proxmox MCP bearer token|Authenticates local Proxmox MCP requests|Proxmox `.env`, Proxmox tunnel authorization file, secure credential storage|
|Unraid MCP bearer token|Authenticates local Unraid MCP requests|Unraid `.env`, Unraid tunnel authorization file; Bitwarden retention pending verification|
|Proxmox OpenAI runtime API key|Connects Proxmox tunnel client to OpenAI|`/etc/tunnel-client/openai-runtime.key` and Bitwarden|
|Unraid OpenAI runtime API key|Connects Unraid tunnel client to OpenAI|`/etc/tunnel-client-unraid/openai-runtime.key` and Bitwarden|
|Unraid GraphQL API key|Read-only access to the Unraid GraphQL API|Unraid `.env` and Bitwarden|
|GitHub credential|Accesses `homelab-infra`|Bitwarden / interactive authentication|

Credential rotation is a consequential maintenance operation.

**Do not rotate multiple credentials simultaneously.**

### General Rotation Procedure

For each credential:

1. Confirm the current service is healthy.
    
2. Verify the credential's exact role and affected components.
    
3. Identify the new credential's required permissions.
    
4. Create the replacement through the appropriate authorized platform.
    
5. Store the new credential securely.
    
6. Update the relevant protected configuration.
    
7. Restart or recreate only the component that requires the new credential.
    
8. Verify the new credential works.
    
9. Revoke the old credential when no longer needed.
    
10. Confirm the old credential is no longer accepted.
    
11. Update the relevant credential-management record.
    

Do not include actual credential values in documentation.

### Proxmox API Token Rotation

The existing API identity is:

`observer@pve!proxmox-mcp`

The API account uses the `PVEAuditor` role.

Before rotating:

- Confirm the current Proxmox permissions.
    
- Confirm the replacement credential remains read-only.
    
- Determine whether the replacement uses a new token ID or the existing identity.
    
- Preserve a tested recovery path.
    

Update the protected MCP configuration.

Recreate the MCP service only when necessary.

Verify authenticated read-only Proxmox queries afterward.

**Never grant elevated privileges simply to complete a token rotation.**

### MCP Bearer Token Rotation

Each integration has its **own** MCP bearer token between its respective tunnel client and local MCP server. Do not accidentally copy the Proxmox token into the Unraid tunnel or vice versa.

Its configured value must be consistent on both sides.

The protected tunnel authorization file contains the complete Authorization header value, including the `Bearer` prefix.

Before changing either side:

1. Confirm the current token storage and configuration.
    
2. Prepare a new strong token.
    
3. Store the replacement securely.
    
4. Update the MCP environment and corresponding tunnel authorization file through an approved procedure.
    
5. Preserve ownership and permissions.
    
6. Recreate or restart the affected services in a controlled sequence.
    
7. Verify the tunnel can authenticate to MCP.
    
8. Perform a live ChatGPT tool call.
    

A brief interruption may occur while both components transition to the replacement token.

Do not display token values in terminal output.

### OpenAI Runtime API Key Rotation

Each tunnel client has its **own** runtime key restricted to the permissions required to read and use that tunnel.

It should not use broad administrative platform credentials.

During rotation:

1. Create a replacement key with the required restricted permissions.
    
2. Store it securely.
    
3. Update the appropriate protected key file (`/etc/tunnel-client/openai-runtime.key` for Proxmox; `/etc/tunnel-client-unraid/openai-runtime.key` for Unraid) through an approved secret-handling procedure.
    
4. Preserve the file's required ownership and permissions.
    
5. Restart only the affected tunnel service.
    
6. Verify tunnel readiness.
    
7. Verify ChatGPT connectivity.
    
8. Revoke the previous key after successful transition.
    

### Credential File Permissions

Documented tunnel credential paths:

- `/etc/tunnel-client/openai-runtime.key`
    
- `/etc/tunnel-client/mcp-authorization`
    
- `/etc/tunnel-client/config.yaml`
- `/etc/tunnel-client-unraid/openai-runtime.key`
- `/etc/tunnel-client-unraid/mcp-authorization`
- `/etc/tunnel-client-unraid/config.yaml`
    

The initial deployment used root-owned protected files with restricted permissions.

Before changing ownership or access permissions, inspect the effective systemd user, group, and file-access mechanism.

Do not assume another host's permissions are appropriate.

Read-only inspection:

```
sudo stat -c '%U %G %a %n' \
  /etc/tunnel-client \
  /etc/tunnel-client/config.yaml \
  /etc/tunnel-client/openai-runtime.key \
  /etc/tunnel-client/mcp-authorization \
  /etc/tunnel-client-unraid \
  /etc/tunnel-client-unraid/config.yaml \
  /etc/tunnel-client-unraid/openai-runtime.key \
  /etc/tunnel-client-unraid/mcp-authorization
```

Inspect service identity:

```
systemctl show tunnel-client tunnel-client-unraid \
  -p User \
  -p Group \
  -p SupplementaryGroups
```

These commands show metadata without printing secret contents.

Do not change ownership or permissions until the current access model is understood.

---

## 15. Security Verification

Periodically review whether the implementation continues to match its documented security boundaries.

### Proxmox Permissions

Verify:

- Dedicated account `observer@pve` exists.
    
- API token has only the intended permissions.
    
- Account role remains `PVEAuditor`.
    
- No elevated administrative permissions were introduced.
    

### Unraid API and MCP Permissions

Verify:

- The dedicated Unraid API key retains only the intended 14 `READ_ANY` categories; no write grants.
- The Rust MCP `UNRAID_RMCP_PROJECTION` remains `atomic`.
- The `UNRAID_RMCP_ENABLED_TOOLS` allowlist matches the exact 26 action names documented in `services/infra/ai-observer.md`.
- The ChatGPT plugin still lists 26 read tools and zero write tools.
- TLS CA-bundle trust is in use and `UNRAID_API_SKIP_TLS_VERIFY=false`.
- The MCP published port is bound to `127.0.0.1:8082`, not to every observer interface.
- No Unraid Docker socket or NAS filesystem is mounted into the MCP container.

### MCP Configuration — Proxmox

Verify:

- Write operations remain disabled.
    
- Delete operations remain disabled.
    
- Execution functionality remains disabled.
    
- Docker socket is not mounted.
    
- HTTP listener is restricted to localhost.
    
- Container security settings remain consistent with the architecture document.
    

### Tunnel Configuration

Verify:

- Both native tunnel systemd services remain enabled.
    
- Dedicated `tunnel-client` service account is used.
    
- Service account has no unnecessary administrative groups.
    
- Both sets of tunnel credential files remain protected.
    
- Both tunnels use outbound connectivity only.
    
- No unnecessary public listeners are present.
    

### Firewall

Verify the targeted exception remains limited to:

`192.168.30.17 → 192.168.10.11 TCP 8006`

Do not broaden this exception to the full Services VLAN or Management VLAN.

### Security Changes

Any change involving:

- API privileges.
    
- MCP tool permissions.
    
- Secret storage.
    
- Public network exposure.
    
- Tunnel authentication.
    
- Service account permissions.
    

requires a deliberate review and subsequent verification.

Update service documentation only after confirming the implemented state.

---

## 16. Backup and Recovery

### Current Status

**AI Observer VM 107 completed its October 30, 2026 scheduled PBS backup successfully**, confirmed through the pve-prod-01 task log. Its earlier October 29 backup failed with `connection reset` during the PBS stale-NFS-lock incident.

The immediate failure was traced to a stale `/mnt/backups/pbs/.lock` handle retained by the long-running `proxmox-backup-proxy` process; a targeted operator-approved proxy restart at 00:55 CDT on October 30 was followed by successful manual and scheduled backups on both Proxmox nodes. The originating NFS/FUSE invalidation event remains unknown, and the operator elected to monitor Discord alerts without changing Unraid or NFS.

**Still unverified:** Independent integrity verification of the October 30 VM 107 snapshot, validation of recoverable host-level configuration and credentials, and a safe isolated restore. The successful backup task is not sufficient evidence of end-to-end observer recoverability. See `runbooks/backup-schedule.md` for detailed incident evidence.

---

### Important Distinction

The enabled backup job demonstrates intended coverage.

It does not demonstrate:

- Successful archive creation.
    
- Data integrity.
    
- Recovery-point availability.
    
- Successful restoration.
    
- Complete disaster recovery.
    

Do not mark the observer as recoverable until these have been verified.

### Required Recovery Assets

|Asset|Recovery Source|
|---|---|
|Observer virtual machine|October 30 PBS backup task succeeded; snapshot integrity/restore test pending|
|MCP Compose configurations (Proxmox and Unraid)|`homelab-infra` repository when committed; Unraid commit not yet verified|
|MCP `.env` files|Protected backup and separately retained credentials (both stacks)|
|Both tunnel configuration directories|Verified VM backup or protected configuration backup|
|Both OpenAI tunnel runtime keys|Secure credential storage|
|MCP bearer tokens|Secure credential storage; confirm Unraid bearer token has separately been saved to Bitwarden|
|Both tunnel systemd units|Verified VM backup or protected configuration backup|
|Shared tunnel binaries|Verified official release artifacts|
|Unraid server TLS certificate bundle|Verified VM backup or trusted certificate re-enrollment|

### Backup Verification Checklist

Before declaring observer recovery ready:

- **Completed:** Immediate PBS proxy stale-handle failure cleared; continue monitoring for recurrence.
    
- **Completed:** October 30 VM 107 scheduled backup task succeeded.
    
- **Pending:** Independently confirm the new VM 107 snapshot in PBS.
    
- Backup verification succeeds.
    
- Required host-level configuration is confirmed within the recovery source.
    
- Credentials are available through approved secure storage.
    
- A safe restoration test is completed.
    
- Restored services pass local health checks.
    
- End-to-end read-only Proxmox connectivity is verified.
    

Only items explicitly marked **Completed** are verified; all remaining checklist items are outstanding requirements.

### Independent Configuration Backup

A separate encrypted configuration backup has been considered but has not been implemented or verified.

If introduced later, it must protect sensitive configuration without placing credentials into Git or ordinary documentation.

It should include appropriate encryption, key recovery, retention, access controls, and restoration testing.

Do not assume existing Docker-host rsync jobs back up the observer.

---

## 17. Disaster Recovery — VM Restoration

**Status:** Procedure outline only — not yet tested.

This section describes the intended operational recovery sequence.

Do not execute a production restoration without first confirming a valid backup and assessing the effects on the existing observer deployment.

### Step 1 — Assess the Incident

Determine:

- Whether the original observer VM still exists.
    
- Whether it is running or stopped.
    
- Whether the virtual disks remain intact.
    
- Whether the problem is limited to a service.
    
- Whether a full VM restoration is necessary.
    

Prefer repairing a failed service when the underlying VM and data remain healthy.

### Step 2 — Verify Recovery Source

In Proxmox/PBS:

1. Locate the intended VM 107 backup.
    
2. Confirm its timestamp.
    
3. Confirm archive availability.
    
4. Check backup verification status.
    
5. Review the required restore destination and available storage.
    
6. Confirm that any necessary credentials are recoverable.
    

If no valid backup exists, do not assume a restoration is possible.

### Step 3 — Plan a Safe Restore

Before restoring:

- Confirm the intended target Proxmox node.
    
- Choose an appropriate VMID.
    
- Confirm available disk space.
    
- Account for the observer's existing static IP address.
    
- Prevent duplicate production observer instances.
    
- Prevent simultaneous use of the same tunnel identity during a test restoration.
    

**Important:** An isolated test VM must not start with the same production IP and active tunnel credentials on the same network as the original observer.

For a restoration test, use an isolated network or disconnected virtual NIC until the restored system can be safely inspected.

### Step 4 — Restore the VM

Follow the general Proxmox restore process in:

`runbooks/backup-restore.md`

The exact restore operation depends on the available backup archive and whether the operation is a test or a production replacement.

Do not overwrite a running VM without a reviewed recovery plan.

### Step 5 — Inspect Restored Configuration

Confirm:

- Operating system starts.
    
- Expected disks are mounted.
    
- Network configuration is present.
    
- Docker and Compose are installed.
    
- Both MCP Compose files exist (`proxmox-mcp` and `unraid-mcp`).
    
- Both MCP `.env` files exist with correct permissions and contain valid recovered credentials.
    
- Shared tunnel binaries and both systemd units exist.
    
- Both protected tunnel configuration directories are accessible to the `tunnel-client` service account.
- The trusted Unraid CA bundle exists and matches the expected NAS certificate.
    
- API and tunnel credentials are valid.
    

Do not immediately expose a test-restored observer to the production network.

### Step 6 — Restore Production Connectivity

For an approved production recovery:

1. Confirm the original observer is no longer actively using the production identity.
    
2. Restore the intended network configuration.
    
3. Confirm both MCP containers start correctly.
    
4. Confirm both tunnel services start correctly without duplicating production identities.
    
5. Verify the Proxmox and Unraid API connections.
    
6. Verify both ChatGPT plugins and their read-only tool inventories.
    
7. Confirm the restored instance is the only active production observer for that identity.
    

### Step 7 — Validate Recovery

Run the routine health checks in Section 3.

Perform a live read-only ChatGPT query through each of the Proxmox and Unraid plugins.

Only after these checks pass should the service be marked restored.

### Step 8 — Record the Outcome

Document:

- Failure cause, if established.
    
- Backup archive used.
    
- Recovery actions performed.
    
- Configuration or credential changes.
    
- Verification results.
    
- Remaining risks.
    
- Documentation updates required.
    

A restoration procedure is not considered tested merely because it has been written.

---

## 18. Recovery — Rebuilding Without a VM Backup

**Status:** Contingency approach, not a verified restoration procedure.

If no usable VM backup exists, the observer may require rebuilding from its documented architecture and recovered credentials.

A fresh installation would need:

1. A supported Ubuntu Server VM.
    
2. Appropriate Proxmox VM configuration.
    
3. VLAN 30 networking.
    
4. SSH key-based administration.
    
5. Docker and Compose installation.
    
6. The private `homelab-infra` repository.
    
7. The Proxmox and Unraid MCP Compose configurations.
    
8. Recovered environment variables and API/MCP bearer secrets for both containers; verified Unraid certificate bundle.
    
9. The verified OpenAI tunnel-client release.
    
10. Both tunnel systemd units and their separate protected configurations.
    
11. Separate restricted OpenAI runtime credentials for each tunnel.
    
12. Valid read-only Proxmox and Unraid API credentials.
    
13. Successful local and end-to-end testing through both ChatGPT plugins.
    

Do not generate new credentials or replace existing production credentials unnecessarily.

A rebuild may require reestablishing or reauthorizing the tunnel and plugin configuration if the original credentials or identity are unavailable.

Because this procedure has not been tested, the complete time and effort required to rebuild the observer are not established.

---

## 19. Known Issues and Outstanding Work

### PBS Backup Failures

**Status:** Operational backup execution restored October 30; originating NFS/FUSE trigger and permanent prevention remain unresolved.

The October 29 backup attempt of VM 107 and other guests failed with connection resets. PBS logs identified a stale datastore `.lock` file descriptor in the proxy process (`ESTALE`); a targeted restart at 00:55 CDT on October 30 cleared the immediate failure. Both PVE nodes then completed their scheduled backup jobs, including VM 107 (`TASK OK`).

**Operational decision:** Keep Unraid's `Permit exclusive shares` setting at `No` and retain the existing FUSE-backed NFS configuration for now. The operator will monitor Discord alerts. No direct-ZFS export, NFS remount, storage migration, or other persistent modification was implemented.

**Next action:** Verify the VM 107 snapshot itself, plan a safe isolated restore, and respond to any recurrence using the read-only diagnostic workflow in `runbooks/backup-restore.md`. Do not equate job completion with proven recoverability.

---

### Observer Disaster Recovery

**Status:** Incomplete.

VM 107's October 30 backup task succeeded, but independent snapshot-integrity verification and restoration are not yet complete.

**Next action:** Now that backup execution has resumed, verify the observer snapshot and complete an isolated restore test.

### Additional MCP Integrations

|Integration|Status|
|---|---|
|Proxmox MCP — read-only|Implemented and verified|
|ChatGPT Homelab Proxmox plugin|Implemented and verified|
|ChatGPT Homelab Unraid plugin|Implemented, 26 read tools verified|
|OpenAI Secure MCP Tunnels (Proxmox + Unraid)|Both implemented and verified|
|Unraid MCP|Implemented, connected, 26 tools, end-to-end query verified|
|Other-host Docker MCP|Planned|
|Proxmox write operations|Deferred|
|Automated maintenance actions|Deferred|
|Codex access through tunnel|Not yet verified|

Do not document planned integrations as implemented until deployment and testing are complete.

---

## 20. Operational Completion Criteria

A routine service check is complete when:

- Observer VM is available.
    
- Both MCP containers are healthy.
    
- Both tunnel services are healthy and report ready.
    
- Proxmox and Unraid API connectivity succeeds with the intended read-only permissions.
    
- A live read-only ChatGPT MCP tool call succeeds through each plugin.
    
- No unexpected security or permission changes are identified.
    

A maintenance operation is complete when:

- The intended change has been applied.
    
- Affected services are healthy.
    
- End-to-end functionality is verified.
    
- Any security implications have been reviewed.
    
- The actual implemented change is recorded in the appropriate documentation.
    

A disaster recovery exercise is complete only when:

- A valid recovery source was used.
    
- Recovery operations finished successfully.
    
- Restored services passed operational checks.
    
- Production identity and network conflicts were prevented.
    
- End-to-end functionality was verified.
    
- Results and remaining risks were documented.
    

**Backup execution has recovered, but observer recovery readiness remains incomplete until archive integrity, credentials, and a safe restoration test are verified.**

---

## 21. Unraid MCP — Operations, Security, Expansion, and Recovery

This section governs the **implemented** Unraid Rust MCP v0.6.3 integration. Detailed architecture and the complete 26-enabled / 35-additional-read / 83-write-disabled action inventories live in `services/infra/ai-observer.md`, with links to the upstream Rust source. Do not duplicate or manually reclassify tool lists here.

### Deployment Identity

| Item | Implemented value |
|---|---|
| MCP stack | `/opt/stacks/unraid-mcp` (`compose.yaml`, protected `.env`) |
| Image | `ghcr.io/dinglebear-ai/unraid-rmcp:latest` |
| Binary version tested | `unraid-rmcp 0.6.3` |
| MCP HTTP endpoint | `http://127.0.0.1:8082/mcp` (localhost only) |
| Upstream API | `https://nas-prod-01.local/graphql` via `192.168.30.16:443` |
| Trusted CA bundle | `/etc/unraid-mcp/unraid.crt` -> `/certs/unraid.crt` |
| MCP projection / tools | `atomic`, exactly 26 allowlisted read-only actions |
| Tunnel ID | `tunnel_6ae54db66f7c8191a73960ac6eb77981` |
| Tunnel systemd unit | `tunnel-client-unraid.service` |
| Tunnel config directory | `/etc/tunnel-client-unraid/` |
| Tunnel health endpoint | `http://127.0.0.1:8083/readyz` |
| Linux service account | `tunnel-client` (shared with Proxmox tunnel) |
| ChatGPT plugin | `Homelab Unraid` (connected; 26 read tools and zero write tools) |

### Routine Read-Only Verification

```bash
cd /opt/stacks/unraid-mcp || exit 1
sudo docker compose ps
sudo docker compose config --quiet
sudo docker compose logs --tail 50 --no-color unraid-mcp

systemctl is-active tunnel-client-unraid.service
systemctl is-enabled tunnel-client-unraid.service
curl -fsS --max-time 10 http://127.0.0.1:8083/readyz

cd /opt/tunnel-client
sudo -u tunnel-client /opt/tunnel-client/tunnel-client doctor \
  --config /etc/tunnel-client-unraid/config.yaml
```

Expect a healthy Docker container; log entry showing one trusted extra CA certificate; `active`, `enabled`, and `ready`; `RESULT ok` from the tunnel diagnostic. The diagnostic may report MCP target HTTP 401 without bearer auth; this is expected reachability feedback, not an authenticated GraphQL test. Conclude with a read-only `unraid_info` query from ChatGPT.

### Checking Exposed Tools Without Printing Secrets

The container's `.env` contains a private bearer token. Do not print it, dump `docker compose config`, use shell tracing (`set -x`), or paste secrets in support requests. To list current tools, a local administrator may run:

```bash
cd /opt/stacks/unraid-mcp || exit 1
MCP_TOKEN="$(sed -n 's/^UNRAID_RMCP_TOKEN=//p' .env)"

printf 'header = "Authorization: Bearer %s"\n' "$MCP_TOKEN" | \
  curl --config - -fsS --max-time 20 \
    -H 'Content-Type: application/json' \
    -H 'Accept: application/json, text/event-stream' \
    -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}' \
    http://127.0.0.1:8082/mcp | \
  python3 -c 'import json,sys; d=json.load(sys.stdin); t=d["result"]["tools"]; print("Exposed:",len(t)); print("\n".join(x["name"] for x in t))'

unset MCP_TOKEN
```

Expected: exactly the 26 tools documented in `services/infra/ai-observer.md`, no management/write actions. Treat this as a diagnostic; do not use a tool discovery count to claim all 26 upstream queries were tested.

### Expand Read-Only Monitoring Deliberately

1. Identify the requested use case and canonical action name in the [Rust README](https://github.com/dinglebear-ai/unraid/blob/main/unraid-rs/README.md#mcp-tool-reference) and [scope classification](https://github.com/dinglebear-ai/unraid/blob/main/unraid-rs/src/mcp/schemas.rs). Review the deployed release, not just a potentially newer `main` branch.
2. Confirm whether returned metadata could expose secrets or sensitive authentication, network, or configuration information. A `Scope::Read` action is not necessarily safe to publish broadly.
3. Confirm the dedicated Unraid API key provides the required read permission. Prefer retaining the 14-category `READ_ANY` model, and don't add API privileges merely because a tool exists.
4. Review `/opt/stacks/unraid-mcp/compose.yaml` and append only specifically approved canonical action names to `UNRAID_RMCP_ENABLED_TOOLS`. Never replace the allowlist with `*`; do not expose any write/admin action in the read-only deployment.
5. Validate the Compose config (`sudo docker compose config --quiet`); review the expected tool count and changes **before** recreation. After approval, recreate only the Unraid container (`sudo docker compose up -d --force-recreate unraid-mcp`).
6. Verify container health, CA certificate loading, exact `tools/list` inventory, and a real read-only tool call. Check the plugin's newly discovered inventory and repeat a live ChatGPT invocation. Tool refresh or reconnect may be required for client-side discovery.
7. Update the canonical enabled/disabled inventory in `services/infra/ai-observer.md`, plus operations documentation as needed, only after verification.

Some read-only actions excluded by design could expose API keys, identity configuration, remote access settings, or detailed environment variables. All write/admin actions remain out of scope under the current policy. A future write integration would require a separately approved security design and new credentials, not just an allowlist edit.

### Updating the Unraid Rust Image

`latest` is a deliberate manually updated Docker tag. Do not assume it updates itself. Follow standard change/review practice, and check the upstream release notes for renamed tools or behavior changes.

```bash
cd /opt/stacks/unraid-mcp || exit 1
sudo docker compose config --quiet
sudo docker compose pull unraid-mcp
sudo docker run --rm --network none \
  --entrypoint /usr/local/bin/runraid \
  ghcr.io/dinglebear-ai/unraid-rmcp:latest --version
```

The `docker compose pull` changes the local image cache but does **not** restart the running service. After confirming the version and impact, obtain authorization before `sudo docker compose up -d --force-recreate unraid-mcp`. Then verify TLS, health, all enabled tool names, and an end-to-end ChatGPT read. Preserve the currently working image for rollback where feasible. Do not switch back to the old `ghcr.io/dinglebear-ai/runraid` image.

### Tunnel Runtime and Credential Files

Two tunnel services use the same `/opt/tunnel-client/tunnel-client` binary and the same Linux account, but separate unit files, tunnel IDs, health ports, OpenAI runtime keys, and MCP bearer tokens. Restarting `tunnel-client-unraid.service` does not intentionally restart `tunnel-client.service`; upgrading the shared binary may affect both after their next restart.

| Protected path | Purpose | Expected ownership/mode |
|---|---|---|
| `/etc/tunnel-client-unraid/` | Credential/config directory | `root:tunnel-client` `0750` |
| `/etc/tunnel-client-unraid/config.yaml` | Unraid tunnel configuration | `root:tunnel-client` `0640` |
| `/etc/tunnel-client-unraid/openai-runtime.key` | OpenAI runtime secret | `root:tunnel-client` `0640` |
| `/etc/tunnel-client-unraid/mcp-authorization` | `Bearer <token>` for local MCP | `root:tunnel-client` `0640` |
| `/opt/stacks/unraid-mcp/.env` | Unraid API key and MCP bearer secret | `gio` `0600` (verified at creation) |

The Unraid runtime API key and upstream Unraid API key were saved in Bitwarden. **Separate Bitwarden retention of `UNRAID_RMCP_TOKEN` remains to be verified**; do not mark credential recovery complete until confirmed. Never put the actual secret values into these Markdown files or Git.

### Known Unraid Storage Issue (Separate Incident)

The October 30 read-only `unraid_array` call discovered that the first 14 TB Seagate parity drive is `DISK_DSBL` / red X. Parity 2, both array data disks, and both ZFS mirror members appeared healthy. The operator saved Unraid diagnostics and plans to investigate a suspected SAS-to-SATA breakout cable in a **separate conversation**. No cable replacement, parity rebuild, disk reassignment, or array repair has been verified. Do **not** treat this storage issue as an MCP service failure. See `architecture/storage/nas-prod-01.md` for the live observation and unresolved status.

### Recovery and Completion

If Unraid MCP fails, inspect its Compose stack, local localhost binding, TLS bundle, the read-only GraphQL credentials, and `tunnel-client-unraid.service` one layer at a time. Avoid restarting Proxmox MCP as a troubleshooting shortcut.

For observer VM recovery, restore both tunnel configuration directories, **both** MCP Compose stacks and protected `.env` files, trusted Unraid certificate, both systemd units, and the shared tunnel binaries. Ensure the isolated restore does not duplicate production IP or tunnel identities. No full restore procedure has been tested; use Section 17's isolation safeguards.

Service readiness is demonstrated by healthy containers, ready tunnels, correct read-only tool inventory, and live ChatGPT tool execution. **Full disaster recovery readiness is still unverified.**

---

## Related Documentation

### Architecture and Service

- `services/infra/ai-observer.md` — AI Observer architecture, deployment, networking, authentication, security, and the complete Unraid MCP enabled/disabled tool inventory.
- `architecture/storage/nas-prod-01.md` — Unraid storage architecture and unresolved primary-parity fault.
    
- `architecture/compute/pve-prod-01.md` — Proxmox host and VM inventory.
    
- `architecture/network/ip-addresses.md` — IP and VLAN allocations.
    
- `architecture/network/firewall-rules.md` — Observer-to-Proxmox firewall exception.
    
- `architecture/network/vlan-design.md` — Network segmentation design.
    
- `architecture/decisions-log.md` — Architecture and security decisions.
    

### Operational Runbooks

- `runbooks/backup-schedule.md` — Backup jobs, October 2026 PBS stale-lock incident, and current recovery-validation status.
    
- `runbooks/backup-restore.md` — General Proxmox and Docker restoration procedures.
    
- `runbooks/git-workflow.md` — Git synchronization and credential handling.
    
- `runbooks/updating-containers.md` — Docker Compose updates.
    
- `services/infra/dockman.md` — Existing Docker UI management boundaries.
    

### Source Repositories

- `homelab-docs` — Obsidian documentation.
    
- `homelab-infra` — Version-controlled Docker Compose configuration.
    

---

## Final Operational Notes

AI Observer is intentionally a read-only monitoring and diagnostic integration.

Its purpose is to provide useful live infrastructure visibility while minimizing the risk of unintended changes.

The documented deployment connects ChatGPT to the Homelab v3 Proxmox cluster and `nas-prod-01` through separate local MCP services and outbound OpenAI Secure MCP Tunnels on a shared observer VM.

**Both integrations are operational; full backup and disaster recovery readiness remain unverified.**

All consequential service changes, credential rotations, and restoration procedures should follow the established one-step-at-a-time workflow with explicit authorization and post-change verification.