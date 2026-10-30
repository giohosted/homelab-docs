# AI Observer — Proxmox MCP & OpenAI Secure MCP Tunnel

**Role:** AI-assisted, read-only infrastructure monitoring and diagnostics  
**Host:** observer-prod-01 (192.168.30.17)  
**Proxmox Host:** pve-prod-01 (192.168.10.11)  
**Proxmox VMID:** 107  
**Status:** Active — Proxmox integration verified end-to-end  
**Last Updated:** 2026-10-30

---

## Overview

AI Observer provides controlled access to Homelab v3 infrastructure from AI assistants, initially ChatGPT, through Model Context Protocol (MCP).

The observer runs on a dedicated Ubuntu virtual machine rather than sharing a Docker host with production applications. This separates AI integration services from existing infrastructure workloads and provides an independent security boundary.

The initial implementation uses:

- **Proxmox MCP:** Read-only access to the Homelab v3 Proxmox cluster.
    
- **OpenAI Secure MCP Tunnel:** Outbound connection allowing ChatGPT to reach the private MCP server.
    
- **ChatGPT Homelab Proxmox plugin:** Connected custom MCP integration.
    

The Proxmox integration was successfully tested both locally and directly from ChatGPT on October 29–30, 2026.

Unraid and Docker integrations are not yet deployed.

---

## Architecture

### Request Flow

```
ChatGPT
   |
   | Homelab Proxmox plugin
   v
OpenAI Secure MCP Tunnel Service
   ^
   | Outbound HTTPS (TCP 443)
   |
observer-prod-01 (192.168.30.17)
   |
   +-- OpenAI tunnel-client (systemd)
   |      |
   |      | Authenticated local HTTP
   |      v
   +-- Proxmox MCP (Docker)
          |
          | HTTPS TCP 8006
          | Pinned TLS certificate fingerprint
          | Read-only Proxmox API token
          v
pve-prod-01 (192.168.10.11)
          |
          v
Proxmox cluster: homelab-v3
   +-- pve-prod-01
   +-- pve-prod-02
```

The MCP HTTP endpoint is bound exclusively to localhost on the observer VM.

No publicly accessible MCP hostname, Traefik route, port forward, or inbound internet firewall rule is required.

### Component Inventory

|Component|Technology|Location|
|---|---|---|
|Observer VM|Ubuntu Server 26.04.1 LTS|pve-prod-01, VM 107|
|Proxmox MCP|mattoddie/proxmox-mcp|Docker on observer-prod-01|
|MCP transport|Streamable HTTP|127.0.0.1:8080/mcp|
|MCP health endpoint|HTTP|127.0.0.1:8080/health|
|Secure tunnel client|OpenAI tunnel-client v0.0.18|Native systemd service|
|Tunnel health endpoint|HTTP|127.0.0.1:8081/readyz|
|Tunnel configuration|YAML and protected credential files|/etc/tunnel-client|
|ChatGPT plugin|Homelab Proxmox|ChatGPT Plugins|

**Important distinction:** OpenAI Secure MCP Tunnel is separate from the homelab's existing Cloudflare Tunnel deployment on docker-prod-01. It does not provide a general-purpose public reverse proxy.

---

## Observer VM

### Virtual Hardware

|Setting|Value|
|---|---|
|VM name|observer-prod-01|
|VM ID|107|
|Proxmox node|pve-prod-01|
|CPU|2 vCPU, x86-64-v3|
|RAM|2 GiB|
|Disk|20 GiB|
|Disk storage|local-zfs|
|Machine|q35|
|Firmware|OVMF / UEFI|
|Disk controller|VirtIO SCSI|
|Network|VirtIO, vmbr0, VLAN tag 30|
|QEMU guest agent|Enabled|
|Start at boot|Enabled|
|Startup delay|90 seconds|

The allocation and Proxmox settings were verified through the live Homelab Proxmox integration.

### Operating System and Network

|Setting|Value|
|---|---|
|OS|Ubuntu Server 26.04.1 LTS|
|Hostname|observer-prod-01|
|Static IP|192.168.30.17/24|
|VLAN|30 — Services|
|Gateway|192.168.30.1|
|Primary DNS|192.168.30.10|
|Secondary DNS|192.168.30.15|
|Administration|SSH|
|Primary user|gio|

### Operating System Security

The observer was configured with SSH key-based authentication.

SSH password authentication, keyboard-interactive authentication, and direct root SSH login were disabled.

The VM uses a dedicated service account, `tunnel-client`, for the OpenAI tunnel daemon. This account has a noninteractive shell and no Docker group membership.

Docker is administered separately with elevated privileges when needed.

---

## Proxmox MCP

### Deployment

**Project:** [https://github.com/mattoddie/proxmox-mcp](https://github.com/mattoddie/proxmox-mcp)  
**Docker image:** `ghcr.io/mattoddie/proxmox-mcp:latest`  
**Container:** `proxmox-mcp`  
**Compose:** `/opt/stacks/proxmox-mcp/compose.yaml`  
**Environment:** `/opt/stacks/proxmox-mcp/.env`

The Compose file is maintained in the private `homelab-infra` GitHub repository.

The `.env` file is excluded from Git and contains the Proxmox connection settings and MCP authentication secret.

The `latest` image tag was deliberately selected for CLI-managed updates.

### Container Security

The deployed Compose configuration includes:

|Control|Configuration|
|---|---|
|Restart policy|unless-stopped|
|Published listener|127.0.0.1:8080:8080|
|Read-only root filesystem|Enabled|
|Linux capabilities|All dropped|
|No new privileges|Enabled|
|Memory limit|256 MiB|
|Process limit|100|

No Docker socket is mounted into the Proxmox MCP container.

### MCP Configuration

|Variable|Configured Value|
|---|---|
|PROXMOX_URL|[https://192.168.10.11:8006](https://192.168.10.11:8006/)|
|PROXMOX_TOKEN_ID|observer@pve!proxmox-mcp|
|PROXMOX_TLS_FINGERPRINT|Configured; value stored securely|
|PROXMOX_DEFAULT_NODE|pve-prod-01|
|PROXMOX_ALLOW_WRITES|false|
|PROXMOX_ALLOW_DELETES|false|
|PROXMOX_ALLOW_EXEC|false|
|MCP_TRANSPORT|http|
|MCP_HTTP_PORT|8080|
|MCP_AUTH_TOKEN|Configured; secret excluded from documentation|

**Enabled toolsets:**

`cluster,nodes,vms,containers,agent,snapshots,storage,backup,tasks`

### Proxmox API Permissions

A dedicated Proxmox account was created:

**Account:** `observer@pve`  
**API token:** `observer@pve!proxmox-mcp`  
**Assigned role:** PVEAuditor  
**ACL path:** `/`

Verified effective privileges include audit and inspection permissions such as:

- Sys.Audit
    
- VM.Audit
    
- VM.GuestAgent.Audit
    
- Datastore.Audit
    
- Pool.Audit
    
- Mapping.Audit
    
- SDN.Audit
    

The account does not have administrative write permissions.

Token privilege separation is disabled because the token belongs to a dedicated read-only account. This does not grant privileges beyond the account's existing permissions.

The API token is stored in Bitwarden and on the observer host in its protected environment configuration.

### TLS Verification

The MCP server connects to the Proxmox API using HTTPS and a pinned SHA-256 certificate fingerprint.

The fingerprint was obtained from the Proxmox node certificate interface.

TLS certificate verification remains enabled through fingerprint validation rather than being globally disabled.

---

## OpenAI Secure MCP Tunnel

### Deployment

**Project:** [https://github.com/openai/tunnel-client](https://github.com/openai/tunnel-client)  
**Installed version:** 0.0.18  
**Installation directory:** `/opt/tunnel-client`  
**Binary:** `/opt/tunnel-client/tunnel-client`  
**Service:** `tunnel-client.service`  
**Configuration:** `/etc/tunnel-client/config.yaml`

The official Linux AMD64 release was downloaded and its SHA-256 checksum successfully verified before installation.

The matching bundled `cloudflared` binary and manifest were installed alongside the tunnel client.

### Tunnel Identity

|Field|Value|
|---|---|
|Tunnel name|homelab-proxmox|
|Tunnel ID|tunnel_6aded6df36348191b926a1b2fed52974|
|Control plane|OpenAI Platform|
|MCP target|[http://127.0.0.1:8080/mcp](http://127.0.0.1:8080/mcp)|
|Health listener|127.0.0.1:8081|
|Authentication to OpenAI|Restricted runtime API key|
|Authentication to MCP|Static bearer token|

The OpenAI runtime API key was created with restricted Tunnel Read and Use permissions.

Administrative tunnel-management credentials are not required by the long-running daemon.

### Service Configuration

The tunnel client runs as a native systemd service.

**Systemd unit:** `/etc/systemd/system/tunnel-client.service`

Important service settings:

|Setting|Value|
|---|---|
|User / Group|tunnel-client|
|Working directory|/opt/tunnel-client|
|Startup|Enabled|
|Restart policy|always|
|Restart delay|5 seconds|
|NoNewPrivileges|true|
|UMask|0077|

The tunnel service starts automatically when the VM boots and is not dependent on an active SSH session.

### Protected Configuration Files

|Path|Purpose|Ownership|Permissions|
|---|---|---|---|
|`/etc/tunnel-client/config.yaml`|Tunnel configuration and secret-file references|root|0640|
|`/etc/tunnel-client/openai-runtime.key`|Restricted OpenAI runtime API key|root|0640|
|`/etc/tunnel-client/mcp-authorization`|Complete MCP Authorization header|root|0640|
|`/etc/tunnel-client/`|Protected configuration directory|root|0750|

The MCP authorization file contains the complete `Bearer` header value. The tunnel client reads this from the file and sends it to the local MCP server.

Secrets are not embedded directly into the systemd unit or committed to Git.

---

## Network and Firewall

### Proxmox API Access

The observer resides on VLAN 30, while Proxmox management resides on VLAN 10.

The default Services-to-Management policy denies this traffic.

A targeted UniFi firewall exception was implemented to permit only the required API connection.

|Field|Value|
|---|---|
|Rule name|Allow Observer to Proxmox API|
|Source|192.168.30.17|
|Destination|192.168.10.11|
|Destination port|TCP 8006|
|Action|Allow|
|Return traffic|Automatically allowed|
|Position|Above Block Services to Mgmt|

This exception does not provide unrestricted access from VLAN 30 to VLAN 10.

A connectivity test originally timed out before the rule was created. After the rule was implemented, the Proxmox API responded with HTTP 401 to an unauthenticated request, proving network reachability.

Authenticated read-only API access was verified afterward.

### OpenAI Connectivity

The tunnel client initiates outbound HTTPS traffic to OpenAI.

|Traffic|Purpose|
|---|---|
|Observer → api.openai.com:443|Secure MCP Tunnel control plane|
|Tunnel client → 127.0.0.1:8080|Local Proxmox MCP requests|
|Proxmox MCP → 192.168.10.11:8006|Proxmox API access|

No public listener, inbound port forwarding, or general inter-VLAN management access was introduced.

---

## ChatGPT Integration

### Plugin

**Name:** Homelab Proxmox  
**Description:** Read-only monitoring and diagnostics for Homelab v3 Proxmox infrastructure.  
**Connection:** Tunnel  
**Tunnel:** homelab-proxmox  
**ChatGPT authentication option:** No authentication  
**Status:** Connected and end-to-end verified

The ChatGPT plugin's _No authentication_ option refers to the absence of a separate OAuth login flow in ChatGPT. It does not disable the MCP server's bearer-token requirement.

Authentication to the local MCP endpoint is supplied by the protected tunnel-client configuration.

### Usage

ChatGPT can automatically select the Homelab Proxmox plugin when live infrastructure information is relevant.

The plugin may also be selected explicitly using `@Homelab Proxmox`.

Typical supported requests include:

- Check Proxmox cluster health and quorum.
    
- Inspect nodes, VMs, and LXCs.
    
- Review resource utilization.
    
- Investigate task failures.
    
- Inspect guest configurations.
    
- Review Proxmox backup coverage.
    
- Check available package-update information.
    

The plugin does not currently provide write access.

Documentation-first troubleshooting still applies: consult the relevant Markdown documentation in Google Drive and verify infrastructure state through the live MCP integration when appropriate.

---

## Implementation Verification

**Initial successful verification:** 2026-10-30 UTC

|Test|Result|
|---|---|
|Docker Compose configuration validation|Passed|
|Proxmox MCP container health|Healthy|
|Proxmox MCP `/health`|HTTP 200, `{"status":"ok"}`|
|Authenticated local MCP tool call|Passed|
|Proxmox cluster-status retrieval|Passed|
|Tunnel-client configuration diagnostic|`RESULT ok`|
|Tunnel systemd service|Enabled and running|
|Tunnel `/readyz`|HTTP 200, `ready`|
|ChatGPT custom plugin connection|Connected|
|Live tool call directly from ChatGPT|Passed|

### Verified Live Query

The `proxmox_get_cluster_status` tool was successfully invoked through the local MCP endpoint and subsequently through ChatGPT's installed Homelab Proxmox plugin.

The response identified:

- Cluster: `homelab-v3`
    
- Proxmox VE: 9.1.6
    
- Two Proxmox nodes online
    
- Cluster quorum present
    
- Live VM, container, storage, and resource information
    

This confirms the complete connection from ChatGPT through the tunnel, MCP server, and Proxmox API.

---

## Backup and Recovery Status

**Status: October 30 PBS backup task successful; archive integrity and restoration still require verification.**

### Proxmox VM Backup Coverage

The cluster has an enabled daily 02:00 Proxmox backup job targeting `pbs-prod-01`. It includes VM 107 and excludes the PBS server VM 103.

During the October 29 incident, the VM 107 backup failed with a `connection reset`. PBS investigations confirmed an invalid, long-lived NFS datastore `.lock` file descriptor in the backup proxy, which returned `ESTALE`. The operator restarted `proxmox-backup-proxy` on October 30 at 00:55 CDT, restoring backup execution; the original event that invalidated the handle is not known.

The **October 30, 2026 scheduled VM 107 backup completed successfully** on `pve-prod-01` at approximately 02:22 CDT; this was verified in its Proxmox task log. That task included the primary VM disk and EFI disk. This confirms a successful backup *task*, not a separately completed PBS snapshot-integrity check or restoration test.

- Successful VM 107 scheduled backup task: **verified** (October 30).
- Independent PBS snapshot-level integrity verification: **pending**.
- Safe isolated VM restoration and service validation: **pending**.
- Separate observer file-level backup: **not implemented/verified**.

The earlier read-only Proxmox archive query returned zero visible snapshots even when PBS showed older backup groups; it is not definitive evidence of archive absence. Check PBS datastore Content directly for the new observer snapshot before claiming it is independently visible and verified.

### Configuration and Credential Recovery

The VM's configuration and credentials are spread across several locations:

- `/opt/stacks/proxmox-mcp/compose.yaml` — version-controlled in `homelab-infra`.
- `/opt/stacks/proxmox-mcp/.env` — Git-excluded Proxmox MCP credentials and configuration.
- `/etc/tunnel-client/config.yaml` — Git-excluded tunnel configuration.
- `/etc/tunnel-client/openai-runtime.key` — Git-excluded OpenAI credential.
- `/etc/tunnel-client/mcp-authorization` — Git-excluded MCP bearer credential.
- `/etc/systemd/system/tunnel-client.service` — Native systemd unit.
- `/opt/tunnel-client/` — Installed tunnel-client binaries.

Existing Docker-host rsync backup jobs must **not** be assumed to protect the observer VM or its `/etc/tunnel-client` configuration.

A complete backup and restoration procedure for the observer has not yet been verified. A successful VM backup task reduces exposure but does not replace an integrity check, secure credential recovery, or an isolated restoration test.

Credentials remain separately retained in Bitwarden according to the established workflow. Avoid copying plaintext credentials into Git repositories or ordinary documentation.

See `runbooks/backup-schedule.md` for the October 2026 PBS incident and `runbooks/ai-observer-operations.md` for the recovery plan.

---

## Operations and Maintenance

### Management Approach

The observer is managed manually through SSH and the CLI.

No Dockman instance is installed on observer-prod-01.

The Dockman instance on docker-prod-01 does not manage this observer's Docker engine.

### Proxmox MCP

The MCP server is managed through Docker Compose:

`/opt/stacks/proxmox-mcp`

The image currently uses the `latest` tag. Updates must be deliberately performed through the CLI, with container status, health, and live API access verified afterward.

### Tunnel Client

The tunnel client is managed through systemd:

`tunnel-client.service`

Updates are performed through an explicitly verified official release rather than Docker Compose.

The tunnel client and Proxmox MCP server are separate processes and must each be checked when troubleshooting.

### Basic Read-Only Health Checks

```
# Check the MCP container
cd /opt/stacks/proxmox-mcp
sudo docker compose ps

# Check the MCP HTTP health endpoint
curl -fsS http://127.0.0.1:8080/health

# Check the tunnel service
sudo systemctl status tunnel-client --no-pager

# Check tunnel readiness
curl -fsS http://127.0.0.1:8081/readyz

# Inspect recent tunnel logs
sudo journalctl -u tunnel-client -n 50 --no-pager
```

Passing health checks confirms component availability, but a live MCP tool call remains the definitive end-to-end verification.

Full update, troubleshooting, service recovery, and credential-rotation procedures belong in `runbooks/ai-observer-operations.md`.

---

## Security Boundaries

The initial implementation deliberately minimizes privileges.

1. The Proxmox API account has audit-only permissions.
    
2. MCP write, delete, and execution capabilities are disabled.
    
3. The MCP endpoint is bound to localhost.
    
4. The observer-to-Proxmox firewall exception is restricted to one source, one destination, and TCP 8006.
    
5. The OpenAI tunnel is outbound-only.
    
6. OpenAI runtime credentials are restricted to tunnel Read and Use.
    
7. The tunnel service runs without root or Docker-group privileges.
    
8. Secrets are kept outside Git and are not reproduced in documentation.
    
9. Existing Docker administration and management services remain isolated.
    

These controls reduce the impact of a compromised or misused integration, but do not eliminate risks associated with granting an external AI service visibility into infrastructure metadata.

Changes to permissions, toolsets, authentication, or network exposure require deliberate review and verification.

---

## Planned Extensions

|Integration|Status|
|---|---|
|Proxmox MCP — read-only|Implemented and verified|
|ChatGPT custom plugin|Implemented and verified|
|OpenAI Secure MCP Tunnel|Implemented and verified|
|Codex usage through the tunnel|Not yet verified|
|Unraid MCP|Planned|
|Docker MCP|Planned|
|Proxmox write operations|Deferred|
|Automated maintenance actions|Deferred|

Write access should be considered only after the read-only integration has demonstrated reliable behavior and a separate authorization and approval model has been designed.

No elevated management privileges should be introduced merely for convenience.

---

## Related Documentation

- `architecture/network/ip-addresses.md` — Observer IP allocation.
    
- `architecture/network/firewall-rules.md` — Targeted Proxmox API firewall exception.
    
- `architecture/compute/pve-prod-01.md` — Observer VM guest inventory.
    
- `architecture/decisions-log.md` — AI Observer security and architecture decisions.
    
- `runbooks/ai-observer-operations.md` — Operations, maintenance, troubleshooting, and recovery procedures.
    
- `runbooks/git-workflow.md` — Version control and credentials workflow.
    
- `runbooks/updating-containers.md` — Docker Compose update practices.
    
- `runbooks/backup-schedule.md` — Backup coverage and verification.
    
- `services/infra/dockman.md` — Local Docker management separation.
    

**Source repositories:**

- `homelab-docs` — Obsidian infrastructure documentation.
    
- `homelab-infra` — Docker Compose infrastructure configuration.
    

---

## Notes

- The AI Observer is an independently managed infrastructure service, not a general-purpose production Docker host.
    
- The secure tunnel does not replace the existing homelab Cloudflare Tunnel or Traefik configuration.
    
- The Proxmox MCP service is intentionally read-only.
    
- Docker and Unraid MCP integrations remain future work.
    
- The October 30 observer VM backup task succeeded; independent snapshot verification, credential-recovery checks, and an isolated restoration test remain outstanding.
    
- Documentation should be updated when implementations change, but planned integrations must not be recorded as completed until tested.