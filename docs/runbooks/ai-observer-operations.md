# AI Observer — Operations & Recovery Runbook

**Service:** AI Observer  
**Host:** observer-prod-01 (`192.168.30.17`)  
**Proxmox VMID:** 107  
**Proxmox Node:** pve-prod-01  
**Environment:** Homelab v3  
**Last Updated:** 2026-11-01  
**Service Status:** Proxmox, Unraid and Komodo MCP integrations verified (84 + 26 + 40 read tools); three Komodo Docker hosts enrolled in Phase 3  
**Backup Status:** October 30 VM 107 task successful; fresh post-Komodo backup and observer reboot tests pending; isolated restore intentionally excluded from Phase 3

---

## Purpose
This runbook provides operational procedures for managing Homelab v3 AI Observer, including:

- Health checks and live ChatGPT verification for **Proxmox, Unraid and Komodo**.
- Independent management and troubleshooting of **three** Docker Compose MCP stacks.
- Independent management of **three** native OpenAI Secure MCP Tunnel systemd services sharing one installed binary.
- Review of upstream identity privileges, MCP tool inventories, authentication differences, and information-disclosure risks.
- Controlled updates, credential rotation, host configuration recovery, PBS backup verification, and isolated restoration.

This document is an operational reference, **not** an architecture specification or substitute for credential storage. Refer to `services/infra/ai-observer.md` for architecture, security decisions and exact inventories of **84 Proxmox**, **26 Unraid**, and **40 Komodo** tool names and exclusions.

### Operational Principles

1. Prefer read-only investigation before making changes.
2. Make **one consequential change at a time** and get operator authorization.
3. Verify each change before advancing.
4. Never expose tokens, `.env` contents or raw sensitive configuration in chat, screenshots or logs.
5. Preserve all three independent read-only security boundaries.
6. Do not expand firewall access or privileges merely to troubleshoot a failing tool.
7. Distinguish container health, tunnel readiness, upstream API access, tool discovery and actual ChatGPT tool success.
8. Do not claim current backups or restoration readiness from an old successful task alone.

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
| Component | Technology | Management |
|---|---|---|
| Proxmox MCP `0.5.1` / 84 read tools | Docker Compose | `/opt/stacks/proxmox-mcp/` |
| Unraid MCP `0.6.3` / 26 read tools | Docker Compose | `/opt/stacks/unraid-mcp/` |
| Komodo MCP `1.5.0` / 40 read tools | Docker Compose | `/opt/stacks/komodo-mcp/` |
| OpenAI Secure MCP Tunnels | Three native systemd services | `sudo systemctl` |
| ChatGPT plugins | Homelab Proxmox, Homelab Unraid, Homelab Komodo | ChatGPT Plugins |
| Proxmox API | Proxmox VE HTTPS | Dedicated `PVEAuditor` identity |
| Unraid API | Unraid GraphQL HTTPS | Dedicated 14-category `READ_ANY` key |
| Komodo API | Komodo Core `2.3.4` HTTPS | Dedicated non-admin Read + scoped Logs account |

### Key Paths
| Path | Purpose |
|---|---|
| `/opt/stacks/proxmox-mcp/compose.yaml`, `.env` | Proxmox MCP stack; protected local credentials in `.env` |
| `/opt/stacks/unraid-mcp/compose.yaml`, `.env` | Unraid MCP stack; protected local credentials in `.env` |
| `/etc/unraid-mcp/unraid.crt` | Trusted Unraid TLS certificate bundle |
| `/opt/stacks/komodo-mcp/compose.yaml`, `.env` | Komodo MCP stack; dedicated API key and secret in protected `.env` |
| `/opt/tunnel-client/tunnel-client` | Shared OpenAI tunnel-client binary |
| `/etc/tunnel-client/config.yaml` | Proxmox tunnel YAML |
| `/etc/tunnel-client/openai-runtime.key` | Proxmox restricted OpenAI key |
| `/etc/tunnel-client/mcp-authorization` | Proxmox MCP bearer header |
| `/etc/systemd/system/tunnel-client.service` | Proxmox tunnel unit |
| `/etc/tunnel-client-unraid/config.yaml` | Unraid tunnel YAML |
| `/etc/tunnel-client-unraid/openai-runtime.key` | Unraid restricted OpenAI key |
| `/etc/tunnel-client-unraid/mcp-authorization` | Unraid MCP bearer header |
| `/etc/systemd/system/tunnel-client-unraid.service` | Unraid tunnel unit |
| `/etc/tunnel-client-komodo/config.yaml` | Komodo tunnel YAML |
| `/etc/tunnel-client-komodo/openai-runtime.key` | Komodo restricted OpenAI key (`homelab-komodo-tunnel`) |
| `/etc/systemd/system/tunnel-client-komodo.service` | Komodo tunnel unit |

**Do not create `/etc/tunnel-client-komodo/mcp-authorization` as a routine repair:** Komodo local MCP is intentionally configured without bearer authentication, unlike the Proxmox and Unraid MCP servers. All MCPs remain loopback-bound on observer. A healthy tunnel does not independently prove the upstream API or ChatGPT tools work.

## 2. Normal Operating State

AI Observer is considered operational when all required components are available and a read-only query succeeds end-to-end.

### Expected State
| Check | Expected result |
|---|---|
| Observer VM and Docker | Running |
| MCP containers | `proxmox-mcp`, `unraid-mcp`, `komodo-mcp` running; Unraid/Komodo healthy |
| Proxmox MCP HTTP | `127.0.0.1:8080/health` HTTP 200 |
| Unraid MCP | `127.0.0.1:8082/mcp`, bearer-authenticated; unauthenticated HTTP 401 can be expected |
| Komodo MCP readiness | `127.0.0.1:8084/ready` returns ready; local MCP has **no bearer requirement** |
| Tunnel systemd services | All three `active` and `enabled` |
| Tunnel readiness | Ports **8081, 8083 and 8085**, each HTTP 200 `ready` |
| Proxmox upstream | Authenticated, pinned TLS certificate, auditor identity |
| Unraid upstream | Authenticated GraphQL over verified TLS |
| Komodo upstream | Authenticated HTTPS with non-admin Read/Logs account; Core covers all three enrolled Docker hosts |
| Remote Periphery | `immich-prod-01` and `auth-prod-01` each connected outbound over WSS, with persistent keys and no inbound host port |
| Registered app stacks | `immich` and `authentik` recognized from existing host-side Compose files; running at final verification |
| ChatGPT plugins | All three connected |
| Tool inventory | **84 Proxmox + 26 Unraid + 40 Komodo = 150 Read, 0 Write** at verification |
| End-to-end test | Live read-only tool call succeeds from each plugin |

### End-to-End Request Path
```text
ChatGPT
  |-- Homelab Proxmox -> homelab-proxmox tunnel -> tunnel-client.service
  |     -> 127.0.0.1:8080/mcp -> HTTPS 192.168.10.11:8006
  |-- Homelab Unraid -> homelab-unraid tunnel -> tunnel-client-unraid.service
  |     -> 127.0.0.1:8082/mcp -> HTTPS 192.168.30.16:443 (GraphQL)
  `-- Homelab Komodo -> homelab-komodo tunnel -> tunnel-client-komodo.service
        -> 127.0.0.1:8084/mcp -> HTTPS komodo.giohosted.com
        -> Komodo Core on docker-prod-01
        -> Docker on docker-prod-01 / immich-prod-01 / auth-prod-01 (via each Periphery)
```

Local MCP endpoints: **8080**, **8082**, **8084** (`/mcp`). Local tunnel readiness endpoints: **8081**, **8083**, **8085** (`/readyz`). No public MCP listener or inbound internet port forward is needed.

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

### Additional Check — Komodo MCP and Tunnel

Run on `observer-prod-01` (read-only):

```bash
cd /opt/stacks/komodo-mcp
sudo docker compose ps
sudo docker compose logs --tail 50 --no-color komodo-mcp
sudo docker compose config --quiet
curl -fsS --max-time 10 http://127.0.0.1:8084/ready
systemctl is-active tunnel-client-komodo.service
systemctl is-enabled tunnel-client-komodo.service
curl -fsS --max-time 10 http://127.0.0.1:8085/readyz
```

Expected: a healthy `komodo-mcp` container, `/ready` reporting connected, `active`, `enabled`, and a tunnel `ready` response. **Do not print `/opt/stacks/komodo-mcp/.env`, Compose-resolved secret values, full Docker Inspect or full stack configurations.** Then invoke `komodo_health_check` from ChatGPT, followed by `komodo_container_list` for `docker-prod-01` to confirm inventory. The local MCP has no bearer requirement, so do not mistake a public HTTP probe or local `doctor` result for complete upstream access verification.

---

## 4. Quick Diagnostic Command Reference

Run these commands on observer-prod-01 unless otherwise noted.

### Docker and MCP
```bash
# Proxmox
cd /opt/stacks/proxmox-mcp
sudo docker compose ps
sudo docker compose images
sudo docker compose config --quiet
sudo docker compose logs --tail 100 proxmox-mcp
sudo docker exec proxmox-mcp /proxmox-mcp version
curl -fsS --max-time 5 http://127.0.0.1:8080/health

# Unraid
cd /opt/stacks/unraid-mcp
sudo docker compose ps
sudo docker compose logs --tail 50 --no-color unraid-mcp
sudo docker compose config --quiet

# Komodo
cd /opt/stacks/komodo-mcp
sudo docker compose ps
sudo docker compose logs --tail 50 --no-color komodo-mcp
sudo docker compose config --quiet
curl -fsS --max-time 10 http://127.0.0.1:8084/ready
```

`docker compose config --quiet` validates without printing resolved environment variables. Avoid `docker compose config` without `--quiet` when `env_file` contains secrets.

### Tunnel Client
```bash
# Proxmox
systemctl is-active tunnel-client.service
systemctl is-enabled tunnel-client.service
curl -fsS --max-time 10 http://127.0.0.1:8081/readyz
sudo journalctl -u tunnel-client.service -n 50 --no-pager

# Unraid
systemctl is-active tunnel-client-unraid.service
systemctl is-enabled tunnel-client-unraid.service
curl -fsS --max-time 10 http://127.0.0.1:8083/readyz
sudo journalctl -u tunnel-client-unraid.service -n 50 --no-pager

# Komodo
systemctl is-active tunnel-client-komodo.service
systemctl is-enabled tunnel-client-komodo.service
curl -fsS --max-time 10 http://127.0.0.1:8085/readyz
sudo journalctl -u tunnel-client-komodo.service -n 50 --no-pager
```

Inspect journals for operational status but redact any unexpectedly logged secrets before sharing.

### Operating System Resources

```
uptime

free -h

df -h /

df -h /opt

sudo docker stats --no-stream proxmox-mcp unraid-mcp komodo-mcp
```

### Network

```
ip -br addr show ens18

ip route

ip route get 192.168.10.11

getent hosts api.openai.com
```

### Local Listening Ports
```bash
sudo ss -lntp
```

Verify that the MCP and tunnel readiness ports bind **only** to `127.0.0.1` on observer:

| Port | Purpose |
|---|---|
| `8080` | Proxmox MCP |
| `8081` | Proxmox tunnel readiness |
| `8082` | Unraid MCP |
| `8083` | Unraid tunnel readiness |
| `8084` | Komodo MCP |
| `8085` | Komodo tunnel readiness |

Do not expose these endpoints with public ports, Traefik routes or new firewall forwards.

### Git Configuration

```
cd /opt/stacks

git status

git log --oneline -5
```

Do not run Git commands that overwrite local changes as part of basic diagnostics.

---

## 5. Troubleshooting Workflow

When ChatGPT cannot retrieve Proxmox, Unraid, or Komodo data, first identify the affected plugin and investigate its independent container, tunnel, and upstream API path in order.

### Diagnostic Sequence

1. Is observer-prod-01 running?
    
2. Can the observer be reached over SSH?

3. Which integration is affected — Proxmox, Unraid, Komodo, or several?
    
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
    
- The affected local MCP health/readiness endpoint does not respond (`8080/health` for Proxmox; `8084/ready` for Komodo), or the Unraid MCP container is unhealthy.
    
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
    
- Incorrect MCP Authorization header (**Proxmox/Unraid only**; Komodo has no local MCP bearer header).
    
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

All three ChatGPT plugins use **No authentication** because they do not require separate end-user OAuth in ChatGPT. This option **does not define local MCP authorization**.

- **Proxmox and Unraid:** Their local MCP HTTP endpoints require the respective bearer token, supplied from each tunnel's protected `mcp-authorization` header file. Never remove that requirement as a quick fix.
- **Komodo:** The local MCP endpoint deliberately uses `MCP_AUTH_ENABLED=false`, with anonymous HTTP clients restricted to the upstream read-only tool scope. There is **no local MCP bearer**. The service still authenticates upstream to Komodo Core with a dedicated non-admin API key/secret. Loopback access alone does not authenticate a local caller.

All three have separate restricted OpenAI tunnel runtime API keys. Distinguish ChatGPT OAuth, local MCP authentication, and upstream API authentication during troubleshooting.

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
- **Historical NAS storage incident (resolved October 31, 2026):** The October 30 `DISK_DSBL` first-parity fault was not caused by the MCP integration. After breakout-cable replacement and simultaneous Parity 1/Disk 3 reconstruction, all array drives were verified `DISK_OK`. See `architecture/storage/nas-prod-01.md` for the completed recovery.

---

### Komodo-Specific Failures

Diagnose Komodo independently of Proxmox and Unraid:

- **Container stopped/unhealthy:** Inspect `/opt/stacks/komodo-mcp/`, container logs, Docker engine, and port `127.0.0.1:8084`; validate `sudo docker compose config --quiet`.
- **Komodo upstream unreachable:** Check observer DNS/TLS reachability to `https://komodo.giohosted.com` (`192.168.30.11` when verified). Do not disable TLS verification or broaden firewall access.
- **`komodo_health_check` configured but unhealthy:** Inspect Komodo Core availability and whether the dedicated `ai-observer-komodo` API credential is valid. Avoid printing `.env` or secrets.
- **Specific resource forbidden:** Review Read/Logs permissions for that resource and whether it belongs to the monitored server. Do not grant Inspect or admin privileges just to satisfy a request.
- **Missing tools:** The deployed allowlist excludes exactly eight tools; ChatGPT should show **40 Read, zero Write**. Compare against the inventory in `services/infra/ai-observer.md`, then refresh the plugin only after identifying a verified drift.
- **Logs missing or truncated:** Check `komodo_container_logs` or `komodo_container_search_logs` arguments, source retention and effective Logs permission; never turn on unrestricted shell/Inspect tools as a shortcut.
- **Komodo local MCP authentication:** `MCP_AUTH_ENABLED=false` is intentional. It does **not** mean the Komodo Core API is unauthenticated: the server uses protected upstream API credentials. It also means an observer-local process can invoke exposed read tools without a distinct MCP bearer header.

Follow the one-step-at-a-time workflow; recover only the failed layer and confirm a live ChatGPT read-only call. See Section 22 for detailed commands and security checks.

---

## 10. Service Management
AI Observer uses Docker Compose for **three** MCP containers and native systemd for **three** outbound secure tunnels. Isolate repairs; do not restart all three integrations when only one fails.

### Proxmox MCP — Docker Compose

```bash
cd /opt/stacks/proxmox-mcp
sudo docker compose ps
sudo docker compose logs --tail 100 proxmox-mcp
```

With specific authorization after diagnosis: `sudo docker compose restart proxmox-mcp` or `sudo docker compose up -d --no-deps proxmox-mcp`.

### Proxmox Tunnel — systemd

```bash
systemctl is-active tunnel-client.service
systemctl is-enabled tunnel-client.service
sudo journalctl -u tunnel-client.service -n 100 --no-pager
```

Approved recovery: `sudo systemctl restart tunnel-client.service`, followed by `8081/readyz` and a ChatGPT Proxmox read.

### Unraid MCP — Docker Compose

```bash
cd /opt/stacks/unraid-mcp
sudo docker compose ps
sudo docker compose logs --tail 100 --no-color unraid-mcp
sudo docker compose config --quiet
```

Only with approval: `sudo docker compose up -d --no-deps unraid-mcp`. Confirm TLS trust, read-only API and 26 tools after a configuration or image change.

### Unraid Tunnel — systemd

```bash
systemctl is-active tunnel-client-unraid.service
systemctl is-enabled tunnel-client-unraid.service
sudo journalctl -u tunnel-client-unraid.service -n 100 --no-pager
curl -fsS --max-time 10 http://127.0.0.1:8083/readyz
```

Approved recovery: restart **only** `tunnel-client-unraid.service`, then test `unraid_info` in ChatGPT.

### Komodo MCP — Docker Compose

```bash
cd /opt/stacks/komodo-mcp
sudo docker compose ps
sudo docker compose logs --tail 100 --no-color komodo-mcp
sudo docker compose config --quiet
curl -fsS --max-time 10 http://127.0.0.1:8084/ready
```

Only after diagnosis and approval: `sudo docker compose up -d --no-deps komodo-mcp`. Verify read-only scope, eight exclusions, 40 discovered tools and a live `komodo_health_check`.

### Komodo Tunnel — systemd

```bash
systemctl is-active tunnel-client-komodo.service
systemctl is-enabled tunnel-client-komodo.service
sudo journalctl -u tunnel-client-komodo.service -n 100 --no-pager
curl -fsS --max-time 10 http://127.0.0.1:8085/readyz
```

Approved recovery: restart only `tunnel-client-komodo.service`, then verify readiness and a live Komodo tool call.

### Restart Order

1. Determine which **single** integration and layer is failing.
2. Restore upstream availability and credentials without broadening permissions.
3. Restore the affected MCP container, if necessary, and verify local health plus a real tool invocation.
4. Restore the corresponding tunnel service, if necessary, and verify its `/readyz` response.
5. Perform a live read-only ChatGPT tool call; audit the tool inventory after changes.

An upgrade to the shared `/opt/tunnel-client/tunnel-client` binary affects all three systemd services after restart. Never conflate tunnel readiness with upstream API authentication.

## 11. Updating Proxmox MCP

The MCP image currently uses:

`ghcr.io/mattoddie/proxmox-mcp:latest`

**Deployed compiled version verified November 1, 2026: `0.5.1`.** The installed ChatGPT plugin exposes exactly **84 Read tools**, from nine configured toolsets. See the architecture file for all names and disabled toolset policy.

This is a mutable image tag. A newer image may be downloaded without any Compose file change.

### Before Updating

1. Review the upstream project and release notes.
    
2. Check the running compiled version (`sudo docker exec proxmox-mcp /proxmox-mcp version`) and image ID; review [the matching upstream release](https://github.com/mattoddie/proxmox-mcp/blob/v0.5.1/docs/tools.md), not only the latest branch.
    
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

After updating, verify `sudo docker exec proxmox-mcp /proxmox-mcp version`, compare the **exact 84-tool inventory** (or review/authorize differences), and finish with a live ChatGPT MCP query.

### Security Verification

Confirm that the deployed configuration still has:

- `PROXMOX_ALLOW_WRITES=false`
    
- `PROXMOX_ALLOW_DELETES=false`
    
- `PROXMOX_ALLOW_EXEC=false`
    
- `PROXMOX_TOOLSETS=cluster,nodes,vms,containers,agent,snapshots,storage,backup,tasks` and exactly the approved 84 Read tools (before any authorized change)
    
- MCP listener bound to localhost.
    
- No Docker socket mount.
    
- Dedicated Proxmox API identity.
    
- TLS fingerprint verification enabled.
    

Do not expand privileges as part of routine updates.

---

## 12. Updating OpenAI Tunnel Client

The OpenAI tunnel client is installed as a native Linux binary.

**Binary:** `/opt/tunnel-client/tunnel-client`  
**Services:** `tunnel-client.service`, `tunnel-client-unraid.service`, `tunnel-client-komodo.service`

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
    
9. Restart each of the three affected systemd services through an approved maintenance action; validate all three, as they share the binary.
    
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

Then also check `8083/readyz` and `8085/readyz`, and perform live Proxmox, Unraid and Komodo queries from ChatGPT. Do not assume one ready tunnel validates the others.

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

All three MCP `.env` credential files must remain excluded from Git. Repository tracking of the newly deployed Komodo Compose file is **not yet verified**.

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
AI Observer uses distinct credentials for its three integrations. Handle one credential at a time; never display values in ChatGPT, shell tracing, support tickets, Git, or Obsidian.

| Credential | Purpose | Verified storage / outstanding verification |
|---|---|---|
| Proxmox API token `observer@pve!proxmox-mcp` | Audit-only Proxmox API access | Proxmox protected `.env` + Bitwarden |
| Proxmox local MCP bearer | Authenticates Proxmox MCP requests | Protected Proxmox `.env`, `/etc/tunnel-client/mcp-authorization`, secure credential record |
| Unraid GraphQL API key | 14-category `READ_ANY` access | Protected Unraid `.env` + Bitwarden |
| Unraid local MCP bearer | Authenticates Unraid MCP requests | Protected Unraid `.env`, `/etc/tunnel-client-unraid/mcp-authorization`; **separate Bitwarden retention unverified** |
| Komodo `ai-observer-mcp` API key/secret | Authenticates Komodo MCP → Komodo Core as `ai-observer-komodo` | Protected Komodo `.env`; created/saved outside chat, **independent secure recovery copy must be verified** |
| Proxmox OpenAI runtime key | Tunnel control-plane Read+Use | `/etc/tunnel-client/openai-runtime.key` + Bitwarden |
| Unraid OpenAI runtime key | Tunnel control-plane Read+Use | `/etc/tunnel-client-unraid/openai-runtime.key` + Bitwarden |
| Komodo OpenAI runtime key `homelab-komodo-tunnel` | Tunnel control-plane Read+Use | `/etc/tunnel-client-komodo/openai-runtime.key` + Bitwarden (verified saved, nonempty on host) |
| GitHub credential | Version-control operations | Bitwarden/interactive authentication |

### General Rotation Procedure

1. Confirm the **affected integration** currently works and its original credential can be revoked safely.
2. Confirm the upstream identity, minimum privilege, intended storage, and dependencies.
3. Create one replacement credential through the authorized UI; store it in Bitwarden/approved secret storage.
4. Enter it locally into the relevant protected file, without logging or printing the value.
5. Check **metadata only** (ownership, mode, nonempty), not secret contents.
6. Restart/recreate only the component needing the new value, with approval.
7. Verify upstream API authentication, MCP tool execution, tunnel readiness, and live ChatGPT access.
8. Revoke the old credential only after successful replacement and confirm revocation.
9. Record rotation date and secure-storage location without documenting values.

### Proxmox API Token Rotation

Account: `observer@pve`; token ID: `observer@pve!proxmox-mcp`; role: `PVEAuditor`. Maintain audit-only permissions. Update the protected Proxmox `.env`; recreate `proxmox-mcp` only after approval and verify live read tools. Never grant administrator privileges to make a diagnostic pass.

### MCP Bearer Token Rotation

**Only Proxmox and Unraid use local MCP bearer tokens**. Each tunnel's `mcp-authorization` file contains the complete `Bearer <token>` header. Replace the MCP server-side token and respective tunnel header together in a controlled sequence, verify permissions and test the live ChatGPT plugin. Do not inadvertently copy one integration's token to another. **Komodo has no local MCP bearer token** with its current intentionally unauthenticated MCP configuration; never create one without an approved design change and matching server configuration.

### OpenAI Runtime API Key Rotation

Each of the three tunnel services has its **own** restricted Tunnels Read+Use runtime key. For the correct service, replace **only** the matching `/etc/tunnel-client[-unraid|-komodo]/openai-runtime.key`, preserve `root:tunnel-client` ownership and `0640` mode, restart only that tunnel after approval, check `/readyz`, then perform a live ChatGPT tool call. Revoke the old key after successful validation. The runtime key does not grant tunnel-management rights.

### Komodo API Key Rotation

Komodo MCP uses `KOMODO_API_KEY` and `KOMODO_API_SECRET` (verify current environment **variable names** locally before changing them; do not print the values) in `/opt/stacks/komodo-mcp/.env`. Replace credentials only with the same non-admin `ai-observer-komodo` resource permissions. Recreate only `komodo-mcp` after approval; `komodo_health_check` and `komodo_container_list` must work before revoking the prior key. Verify all 40 Read tools and eight exclusions still apply.

### Credential File Permissions

Expected tunnel directories are `root:tunnel-client` `0750`, with configuration and runtime key files `root:tunnel-client` `0640`. Proxmox and Unraid have additional protected `mcp-authorization` files; Komodo **does not**.

```bash
# Metadata only — never `cat` the key files
sudo stat -c '%A %U:%G %n' \
  /etc/tunnel-client \
  /etc/tunnel-client/config.yaml \
  /etc/tunnel-client/openai-runtime.key \
  /etc/tunnel-client/mcp-authorization \
  /etc/tunnel-client-unraid \
  /etc/tunnel-client-unraid/config.yaml \
  /etc/tunnel-client-unraid/openai-runtime.key \
  /etc/tunnel-client-unraid/mcp-authorization \
  /etc/tunnel-client-komodo \
  /etc/tunnel-client-komodo/config.yaml \
  /etc/tunnel-client-komodo/openai-runtime.key

systemctl show tunnel-client.service tunnel-client-unraid.service tunnel-client-komodo.service \
  -p User -p Group -p WorkingDirectory
```

The Komodo tunnel `0750`/`0640` permissions and nonempty runtime key were directly verified on November 1, 2026. The Compose `.env` mode was verified `0600` during deployment. Check all permissions again after any recovery or migration.

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

### Komodo API and MCP Permissions

Verify non-admin Komodo user `ai-observer-komodo` retains Read and scoped Logs permissions, without Inspect/Terminal/Attach/Processes/Create, and the observer `.env` is mode `0600`. The MCP config must retain `MCP_AUTH_ENABLED=false` **only** while bound to loopback, `MCP_SECRET_SCRUB_ENABLED=true`, and the **eight blocked tools** listed in the architecture doc. Confirm the ChatGPT plugin still has exactly **40 Read, zero Write**; note remaining `*_info` and log tools may still expose secrets. Never enable a management/write MCP action merely because it appears upstream.

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

- All **three** native tunnel systemd services remain enabled.
    
- Dedicated `tunnel-client` service account is used.
    
- Service account has no unnecessary administrative groups.
    
- All three sets of tunnel config/runtime-key files remain protected; Proxmox/Unraid have bearer header files, Komodo intentionally does not.
    
- All three tunnels use outbound connectivity only.
    
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
| Asset | Recovery source / verification requirement |
|---|---|
| Observer VM 107 | October 30 PBS backup task succeeded, **but predates Nov 1 Komodo deployment**; verify a fresh intact snapshot and isolated restore |
| Proxmox MCP stack and `.env` | `homelab-infra` Compose (where committed) plus protected credential recovery |
| Unraid MCP stack and `.env` | Compose repository commit still unverified; protected credential recovery |
| Komodo MCP stack and `.env` | New November 1 stack; Git tracking/secure file-level backup not verified; restore dedicated Komodo API key/secret |
| Three tunnel directories | `/etc/tunnel-client/`, `/etc/tunnel-client-unraid/`, `/etc/tunnel-client-komodo/` including protected files |
| Three OpenAI runtime keys | Secure credential storage; Komodo `homelab-komodo-tunnel` confirmed saved |
| Proxmox/Unraid MCP bearer tokens | Secure storage; separately saved Unraid bearer remains unverified |
| Three systemd service units | `/etc/systemd/system/tunnel-client*.service` (verify exact set) |
| Shared tunnel binaries | Verified official release bundle and rollback copy |
| Unraid TLS certificate bundle | `/etc/unraid-mcp/unraid.crt` or re-enroll and validate host identity |

### Backup Verification Checklist

Before declaring observer recovery ready:

- **Completed:** Immediate PBS proxy stale-handle failure cleared; continue monitoring for recurrence.
    
- **Completed:** October 30 VM 107 scheduled backup task succeeded **before Komodo deployment**.

- **Pending:** A new successful, independently checked VM 107 snapshot containing the November 1 three-integration configuration.
    
- **Pending:** Independently confirm the new VM 107 snapshot in PBS.
    
- Backup verification succeeds.
    
- Required host-level configuration is confirmed within the recovery source.
    
- Credentials are available through approved secure storage.
    
- A safe restoration test is completed.
    
- Restored services pass local health checks.
    
- End-to-end read-only Proxmox, Unraid **and Komodo** connectivity is verified after the restore.
    

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
    
- **All three MCP Compose files** exist (`proxmox-mcp`, `unraid-mcp`, `komodo-mcp`).

- **All three protected MCP `.env` files** exist, are appropriately restricted and contain recoverable API credentials. Check metadata only; do not reveal values.

- The shared tunnel-client release bundle and **all three systemd units** exist.

- **All three protected tunnel directories** and their restricted OpenAI runtime keys are present and accessible to the `tunnel-client` service account. Proxmox/Unraid additionally have MCP bearer header files; Komodo does not.

- The trusted Unraid CA bundle exists and matches the expected NAS certificate.

- Komodo MCP is still bound to loopback and remains limited to its approved **40 read tools / eight exclusions**, with a dedicated non-admin `ai-observer-komodo` upstream account.
    
- API and tunnel credentials are valid.
    

Do not immediately expose a test-restored observer to the production network.

### Step 6 — Restore Production Connectivity

For an approved production recovery:

1. Confirm the original observer is no longer actively using the production identity.
    
2. Restore the intended network configuration.
    
3. Confirm **all three MCP containers** start correctly.

4. Confirm **all three tunnel services** start correctly without duplicating any of the three production tunnel identities.

5. Verify authenticated read-only upstream connections to Proxmox, Unraid and Komodo Core. Preserve TLS verification.

6. Verify **all three ChatGPT plugins** and their approved **84 + 26 + 40 read-tool inventories**, with zero write tools.

7. Confirm the restored instance is the only active production observer using these identities.
    

### Step 7 — Validate Recovery

Run the routine health checks in Section 3.

Perform a live read-only ChatGPT query through **each of the three plugins**: Proxmox, Unraid and Komodo.

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
    
7. **All three** MCP Compose configurations: Proxmox, Unraid, and Komodo.

8. All three protected `.env` credential files or approved recovered replacements; Proxmox/Unraid local MCP bearer tokens, dedicated Komodo API key/secret, and validated Unraid TLS certificate bundle.

9. The verified OpenAI tunnel-client release and matching companion files.

10. **All three** native tunnel systemd units and their protected YAML configurations.

11. Separate restricted OpenAI runtime API keys for **all three** tunnels. Proxmox/Unraid have protected MCP bearer header files; Komodo has no local MCP bearer.

12. Valid non-admin Proxmox, Unraid and Komodo upstream identities, with their original read-only permissions.

13. Successful local and end-to-end tool testing through **all three** ChatGPT plugins and audit of the 84/26/40 tool inventory.
    

Do not generate new credentials or replace existing production credentials unnecessarily.

A rebuild may require reestablishing or reauthorizing the tunnel and plugin configuration if the original credentials or identity are unavailable.

Because this procedure has not been tested, the complete time and effort required to rebuild the observer are not established.

---

## 19. Known Issues and Outstanding Work

### PBS Backup Failures

**Status:** Operational backup execution restored October 30; originating NFS/FUSE trigger and permanent prevention remain unresolved.

The October 29 backup attempt of VM 107 and other guests failed with connection resets. PBS logs identified a stale datastore `.lock` file descriptor in the proxy process (`ESTALE`); a targeted restart at 00:55 CDT on October 30 cleared the immediate failure. Both PVE nodes then completed their scheduled backup jobs, including VM 107 (`TASK OK`).

**Operational decision:** Keep Unraid's `Permit exclusive shares` setting at `No` and retain the existing FUSE-backed NFS configuration for now. The operator will monitor Discord alerts. No direct-ZFS export, NFS remount, storage migration, or other persistent modification was implemented.

**Next action:** Verify a recent VM 107 snapshot and respond to any recurrence using the read-only diagnostic workflow in `runbooks/backup-restore.md`. The isolated restore exercise is deferred outside the agreed Phase 3 scope; do not equate job completion with proven recoverability.

---

### Observer Disaster Recovery

**Status:** Incomplete.

VM 107's October 30 backup task succeeded, but independent snapshot-integrity verification and restoration are not yet complete.

**Next action:** Verify a new post-November 1 observer backup and its integrity/credential coverage; separately verify VM reboot and all three tunnel services reconnect. An isolated restore exercise is excluded from the agreed Phase 3 scope, remains untested, and should not be represented as completed disaster-recovery validation.

### Additional MCP Integrations

| Integration | Status |
|---|---|
| Proxmox MCP (`0.5.1`) and Homelab Proxmox plugin | Implemented, end-to-end verified; **84 Read** tools, nine toolsets |
| Unraid Rust MCP (`0.6.3`) and Homelab Unraid plugin | Implemented, end-to-end verified; **26 Read** tools |
| Komodo MCP (`1.5.0`) and Homelab Komodo plugin | Implemented November 1, end-to-end verified; **40 Read** tools, eight exclusions; initial 26-container count was for `docker-prod-01` only |
| OpenAI Secure MCP Tunnels | **Three independent** tunnels implemented, ready and exercised via ChatGPT |
| Monitor `immich-prod-01` and `auth-prod-01` | **Implemented and verified November 1**; three Komodo servers `Ok` on Periphery `2.3.4`; both application stacks adopted |
| Proxmox, Unraid or Komodo write actions | Deferred and not authorized |
| Automated maintenance | Deferred |
| Codex tunnel usage | Not yet verified |
| Fresh protected observer backup including Komodo additions | **Pending**; snapshot integrity and credentials must be checked |
| Observer reboot and tunnel recovery | **Pending live verification** |
| Isolated restoration exercise | **Not performed**; explicitly outside agreed Phase 3 scope; disaster-recovery proof remains incomplete |

Do not mark optional future integrations or additional hosts as implemented before verifying them.

---

## 20. Operational Completion Criteria

A routine service check is complete when:

- Observer VM is available.
    
- **All three MCP containers** are healthy / operational.

- **All three tunnel services** are active and report `ready` on ports `8081`, `8083`, and `8085`.

- Proxmox, Unraid and Komodo upstream API connectivity succeeds with the intended read-only permissions.

- A live read-only ChatGPT tool call succeeds through **each** plugin; the inventory remains **84 / 26 / 40 Read and zero Write** unless an authorized change has been made.
    
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
    

**Backup execution has recovered, but observer recovery readiness remains incomplete until a fresh backup, archive integrity, credentials and recovery testing are verified. The isolated restore exercise was not part of the agreed Phase 3 implementation scope.**

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

All **three** tunnel services use the same `/opt/tunnel-client/tunnel-client` binary and Linux account, but have separate unit files, tunnel IDs, health ports, OpenAI runtime keys and YAML. Only the Proxmox and Unraid tunnels use MCP bearer tokens; Komodo does not. Restarting `tunnel-client-unraid.service` does not intentionally restart the other services. Upgrading the shared binary may affect **all three** after their next restart.

| Protected path | Purpose | Expected ownership/mode |
|---|---|---|
| `/etc/tunnel-client-unraid/` | Credential/config directory | `root:tunnel-client` `0750` |
| `/etc/tunnel-client-unraid/config.yaml` | Unraid tunnel configuration | `root:tunnel-client` `0640` |
| `/etc/tunnel-client-unraid/openai-runtime.key` | OpenAI runtime secret | `root:tunnel-client` `0640` |
| `/etc/tunnel-client-unraid/mcp-authorization` | `Bearer <token>` for local MCP | `root:tunnel-client` `0640` |
| `/opt/stacks/unraid-mcp/.env` | Unraid API key and MCP bearer secret | `gio` `0600` (verified at creation) |

The Unraid runtime API key and upstream Unraid API key were saved in Bitwarden. **Separate Bitwarden retention of `UNRAID_RMCP_TOKEN` remains to be verified**; do not mark credential recovery complete until confirmed. Never put the actual secret values into these Markdown files or Git.

### Resolved Unraid Storage Incident (October 31, 2026)

The October 30 read-only `unraid_array` call correctly surfaced an **at-that-time** `DISK_DSBL` / red X on the first parity slot (then a 14 TB Seagate). Subsequent independent storage troubleshooting established that Parity 1 and Disk 3 shared a suspected faulty SAS-to-SATA breakout cable. The cable was replaced, original 12 TB WD Red Pro Parity 1 (`AAHKXP9H`) reinstalled, and existing WD Red Pro Disk 3 (`5PK7HDXF`) reassigned. A simultaneous Parity 1/Disk 3 reconstruction completed October 31 with 0 reported errors; all five array disks returned `DISK_OK`. Both affected disks passed fresh SMART short tests, CRC counters stayed unchanged, and Disk 3's XFS filesystem mounted and traversed successfully.

**Storage incident status: Resolved.** The replaced shared cable/connection path is the probable, not conclusively isolated, cause. This was a NAS storage incident and **not** an MCP failure. Keep the detailed incident evidence and verification limits in `architecture/storage/nas-prod-01.md` rather than duplicating the recovery runbook here.

### Recovery and Completion

If Unraid MCP fails, inspect its Compose stack, local localhost binding, TLS bundle, the read-only GraphQL credentials, and `tunnel-client-unraid.service` one layer at a time. Avoid restarting Proxmox MCP as a troubleshooting shortcut.

For observer VM recovery, restore **all three** tunnel configuration directories, **all three** MCP Compose stacks and protected `.env` files, trusted Unraid certificate, **all three** systemd units, and the shared tunnel binaries. Ensure an isolated restore does not duplicate the production IP or any of the three tunnel identities. No full restore procedure has been tested; use Section 17's isolation safeguards.

Service readiness is demonstrated by healthy containers, ready tunnels, correct read-only tool inventory, and live ChatGPT tool execution. **Full disaster recovery readiness is still unverified.**

---

## 22. Komodo MCP — Operations, Security, Expansion, and Recovery

### Deployment Identity

- Source: [MP-Tool/komodo-mcp-server](https://github.com/MP-Tool/komodo-mcp-server).
- Local container `komodo-mcp`, Compose directory `/opt/stacks/komodo-mcp/`, image `ghcr.io/mp-tool/komodo-mcp-server:latest`.
- Deployed MCP `1.5.0`; upstream Komodo Core `2.3.4` at verification.
- Core is on `docker-prod-01` (Komodo server resource ID `6ae6ba185b2d3aff47284bcc` at verification); Core URL `https://komodo.giohosted.com`. Two additional Periphery agents run on `immich-prod-01` and `auth-prod-01`.
- Local MCP `/mcp` `127.0.0.1:8084`; readiness `/ready` on same port.
- Tunnel `homelab-komodo`, ID `tunnel_6ae6e984758881919f5c308207bab3d3`; `tunnel-client-komodo.service`; config `/etc/tunnel-client-komodo/config.yaml`; readiness `127.0.0.1:8085/readyz`.
- Restricted OpenAI runtime credential named `homelab-komodo-tunnel`, saved in Bitwarden and installed as `/etc/tunnel-client-komodo/openai-runtime.key` (`root:tunnel-client`, `0640`).
- `MCP_AUTH_ENABLED=false`: **no local MCP bearer authentication**; Read tool enforcement and a dedicated non-admin Komodo user are essential, and all local clients on observer loopback may access the permitted reads.

### Routine Read-Only Verification

```bash
cd /opt/stacks/komodo-mcp
sudo docker compose ps
sudo docker compose logs --tail 50 --no-color komodo-mcp
sudo docker compose config --quiet
curl -fsS --max-time 10 http://127.0.0.1:8084/ready

systemctl is-active tunnel-client-komodo.service
systemctl is-enabled tunnel-client-komodo.service
curl -fsS --max-time 10 http://127.0.0.1:8085/readyz

# Read-only diagnostic, under the actual systemd account
sudo -u tunnel-client /opt/tunnel-client/tunnel-client doctor \
  --config /etc/tunnel-client-komodo/config.yaml
```

Expected: healthy container, `ready:true`, active/enabled tunnel, `ready`, and `RESULT ok` from doctor. The local diagnostic may legitimately report MCP HTTP 400 as **reachable**; it is not an authenticated tool invocation. From ChatGPT invoke `komodo_health_check`, `komodo_server_list`, `komodo_stack_list`, and `komodo_container_list` for each of `docker-prod-01`, `immich-prod-01` and `auth-prod-01`. The initial November 1 check returned 26 containers on `docker-prod-01`; later Phase 3 checks returned three `Ok` servers, `immich` and `authentik` stacks running, and container visibility on both new hosts. Resource counts (initially 9 stacks, 3 procedures, 1 builder) are historical checkpoints, not expected permanent numbers.

### Remote Periphery — Two Additional Docker Hosts (Implemented and Verified)

This is an **extension of the existing Komodo Core**, not a fourth MCP integration. No observer Docker socket, additional tunnel, MCP image or OpenAI credential is required. The `Homelab Komodo` plugin remains **40 Read / 0 Write**; interactive container/host terminals are enabled **only for authorized Komodo administrators through the Komodo UI**.

| Host | IP | Agent Compose | Persistent identity keys | Existing adopted stack |
|---|---|---|---|---|
| `immich-prod-01` | `192.168.30.14` | `/opt/stacks/komodo-periphery/compose.yaml` | `/opt/appdata/komodo/periphery/keys/` | `immich` |
| `auth-prod-01` | `192.168.30.13` | `/opt/stacks/komodo-periphery/compose.yaml` | `/opt/appdata/komodo/periphery/keys/` | `authentik` |

**Verified remote-agent settings (November 1 CST):** `ghcr.io/moghtech/komodo-periphery:2` / actual version `2.3.4`, `PERIPHERY_CORE_ADDRESS=wss://komodo.giohosted.com`, `PERIPHERY_CONNECT_AS=<exact server name>`, `PERIPHERY_SERVER_ENABLED=false`, `PERIPHERY_ROOT_DIRECTORY=/opt`, Core public key at `file:/config/keys/core.pub`, terminal and container-terminal access enabled, no published TCP port 8120. TLS certificate verification is enabled. Agent containers mount the local Docker socket, `/proc` read-only, `/opt/stacks`, and persistent keys. These are privileged host-management agents: protect administrative access and do not mount the Docker socket on the observer MCP container.

**Read-only host checks (execute separately on each remote host, using `ssh gio@192.168.30.14` or `ssh gio@192.168.30.13`):**

```bash
hostname
sudo docker ps -a --filter name=komodo-periphery \
  --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'
sudo docker compose -f /opt/stacks/komodo-periphery/compose.yaml config --quiet
sudo docker compose -f /opt/stacks/komodo-periphery/compose.yaml logs --tail 30 --no-color periphery
sudo stat -c '%A | %U:%G | %n' /opt/appdata/komodo/periphery/keys/{core.pub,periphery.pub,periphery.key}
sudo docker inspect komodo-periphery --format '{{json .HostConfig.PortBindings}}'
```

Expected: container running, no published port bindings, agent login to Core logged, permanent keys `core.pub`, `periphery.pub`, `periphery.key` present and root-owned. Never print `periphery.key` or `.env` contents. `core.pub` is public, but there is normally no need to paste it. The Periphery startup log reports `onboarding_key: None` after completed enrollment.

**Read-only Core/MCP checks:** use `komodo_server_list` to confirm three servers `Ok` at `2.3.4`; `komodo_stack_list` to confirm `immich` and `authentik`; `komodo_container_list(server=...)` to confirm each application's four containers plus Beszel and Periphery; and `komodo_container_logs` only as needed, with awareness that app logs may contain sensitive data. Do **not** use Komodo's **Deploy**, **Update**, container restart or terminal operations while merely verifying monitoring.

**Initial onboarding (historical implementation):** Each new host was registered using a separately created **non-privileged** Komodo onboarding key, stored temporarily in `/opt/appdata/komodo/periphery/secrets/onboarding-key` with root-only mode `0600`, bind-mounted read-only into Periphery, and supplied using `PERIPHERY_ONBOARDING_KEY_FILE`. The key was removed from Compose, the agent recreated and proven able to log in with `onboarding_key: None`, the host credential file removed, and the key revoked under Komodo **Settings → Onboarding**. For future re-enrollment, do not recycle the old identity or generate new keys without investigating the existing server public-key mapping and secure recovery plan. Never share onboarding credentials in chat or Git.

**Existing Compose registration:** In Komodo use **Stacks → New Stack → Files on Server**, target the correct server, run directory `/opt/stacks/immich` or `/opt/stacks/authentik`, file path `compose.yaml`, project name `immich` or `authentik` respectively. Confirm those values using Docker `com.docker.compose.project`, `.config_files` and `.working_dir` labels first. Merely saving the Komodo stack resource registers the existing Compose project; **do not click Deploy to accomplish adoption**. The Periphery-only Compose deployments under `/opt/stacks/komodo-periphery/` were managed independently at enrollment.

**Security and recovery:** The host-local Periphery private keys under `/opt/appdata/komodo/periphery/keys/` are essential for reconnecting. Protect them with appropriate host backups and access controls; backup coverage of these new key directories was not separately verified. The onboarding keys and temporary credential files were deliberately removed. A failed WSS handshake should be investigated through DNS/TLS, Traefik, and Periphery logs before making any firewall or Core changes; a bare `/ws/periphery` upgrade request without `?server=<name>` can return HTTP 400 even when routing is healthy. A complete WebSocket upgrade probe including the query returned HTTP 101 on both hosts during rollout.

**Immich incident qualification:** During integration Komodo later showed `immich_server` in Docker state `created` and its stack `unhealthy`; that issue was handled separately. A subsequent live MCP query showed `immich_server` and the stack running. Do not derive an unverified cause or recovery steps from this document.

### Checking the Tool Boundary

Before and after updates, verify ChatGPT shows **40 Read tools, zero Write**. Use the architecture document's exact list to compare. These **eight tools must remain excluded**:

```text
komodo_container_inspect
komodo_exec
komodo_docker_image_inspect
komodo_docker_volume_inspect
komodo_docker_image_history
komodo_stack_info
komodo_toml_export_all
komodo_toml_export_resources
```

The exclusions were chosen after checking sensitivity of Docker Inspect (`Config.Env` exposed Cloudflare/WireGuard secrets), stack configuration, image/volume inspection, image history, arbitrary exec, and broad TOML exports. Keep container logs and search, server stats, and selected `*_info` tools for diagnostics, but do not assume all returned data is nonsensitive. The operator deliberately retained `komodo_server_info`, `komodo_procedure_info`, and `komodo_builder_info`; revisit these if actual output reveals secrets. **Never paste full Inspect, Compose, `.env` contents, or credentials into ChatGPT.** No automated secret detector can guarantee safe disclosure.

### Updating Komodo MCP

Treat `latest` as a **manual** update target. Do not pull or recreate an image without approval and verifying available rollback and backups. First review upstream release notes for changes to anonymous HTTP read-only behavior, tool names, tool-exclusion settings, and output sanitization. Check the existing Compose and permissions without printing secrets:

```bash
cd /opt/stacks/komodo-mcp
sudo docker compose ps
sudo docker compose images
sudo docker compose config --quiet
sudo docker compose logs --tail 30 --no-color komodo-mcp
```

Only after an approved change: `sudo docker compose pull komodo-mcp` and `sudo docker compose up -d --no-deps komodo-mcp`. Then verify its `/ready`, the non-admin Komodo user, expected **40 Read/zero Write** names and eight exclusions, tunnel `8085/readyz`, and live ChatGPT health plus inventory calls. Do not introduce an `MCP_AUTH_ENABLED` change without separately planning both server and tunnel/client authentication. Avoid changing Komodo Core during an MCP-only maintenance task.

### Credential, Tunnel, and Recovery Notes

Protected `.env` contains Komodo API key and secret, not a token to paste into chat. The OpenAI runtime key is separately stored in `/etc/tunnel-client-komodo/openai-runtime.key`; there is no `mcp-authorization` file in this tunnel. The systemd unit runs as `tunnel-client`, without root or Docker group privileges, and has the same Restart/NoNewPrivileges/UMask policy as the other tunnel services.

After failure, isolate whether the cause is the Komodo container, DNS/TLS/API credentials, per-resource permissions, the tunnel service, or ChatGPT plugin discovery. Make a single authorized recovery change, then verify each layer. Preserve the original Proxmox/Unraid services. Restore and verify the complete new Komodo stack, protected `.env`, dedicated runtime key, tunnel YAML and systemd unit during VM recovery. The October 30 observer backup does not prove November 1 Komodo files are recoverable.

---

## Related Documentation

### Architecture and Service

- `services/infra/ai-observer.md` — AI Observer architecture, deployment, networking, authentication, security, and complete Proxmox, Unraid, and Komodo tool inventories/exclusions.
- `architecture/storage/nas-prod-01.md` — Unraid storage architecture, current five-disk array, and resolved October 2026 parity/Disk 3 incident.
    
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
- `services/media/immich.md` — Immich host and its adopted Komodo stack.
- `services/infra/authentik.md` — Authentik host and its adopted Komodo stack.
    

### Source Repositories

- `homelab-docs` — Obsidian documentation.
    
- `homelab-infra` — Version-controlled Docker Compose configuration.
    

---

## Final Operational Notes

AI Observer is intentionally a read-only monitoring and diagnostic integration.

Its purpose is to provide useful live infrastructure visibility while minimizing the risk of unintended changes.

The documented deployment connects ChatGPT to the Homelab v3 Proxmox cluster, `nas-prod-01`, and Komodo Core on `docker-prod-01` through three separate local MCP services and outbound OpenAI Secure MCP Tunnels on a shared observer VM.

**All three integrations have passed live ChatGPT tool calls; full backup and disaster recovery readiness remain unverified.**

All consequential service changes, credential rotations, and restoration procedures should follow the established one-step-at-a-time workflow with explicit authorization and post-change verification.
