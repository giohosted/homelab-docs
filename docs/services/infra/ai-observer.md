# AI Observer — Proxmox & Unraid MCP with OpenAI Secure MCP Tunnels

**Role:** AI-assisted, read-only infrastructure monitoring and diagnostics  
**Host:** observer-prod-01 (192.168.30.17)  
**Proxmox Host:** pve-prod-01 (192.168.10.11)  
**Proxmox VMID:** 107  
**Status:** Active — Proxmox and Unraid integrations verified end-to-end  
**Last Updated:** 2026-10-30

---

## Overview

AI Observer provides controlled access to Homelab v3 infrastructure from AI assistants, initially ChatGPT, through Model Context Protocol (MCP).

The observer runs on a dedicated Ubuntu virtual machine rather than sharing a Docker host with production applications. This separates AI integration services from existing infrastructure workloads and provides an independent security boundary.

The implemented integrations are:

- **Proxmox MCP:** Read-only monitoring of the Homelab v3 Proxmox cluster.
- **Unraid MCP:** Read-only monitoring of `nas-prod-01` over the Unraid GraphQL API.
- **OpenAI Secure MCP Tunnels:** Two independent outbound tunnel-client systemd services, one per MCP server.
- **ChatGPT plugins:** `Homelab Proxmox` and `Homelab Unraid`, each separately connected and tested with a live tool call.

The Proxmox integration was verified October 29–30, 2026. The Unraid Rust MCP 0.6.3 integration was deployed and verified October 30, 2026 (local CDT; the source NAS returned October 31 UTC timestamps).

A separate Docker-host MCP integration remains **planned**. The Unraid plugin can inspect Docker containers managed by Unraid, but it is not a replacement for the proposed Docker MCP integration covering the homelab's other Docker hosts.

---

## Architecture

### Request Flow

```
ChatGPT
   |                                     |
   | Homelab Proxmox plugin             | Homelab Unraid plugin
   v                                     v
OpenAI Secure MCP Tunnel              OpenAI Secure MCP Tunnel
   |                                     |
   | outbound HTTPS                       | outbound HTTPS
   v                                     v
observer-prod-01 (192.168.30.17, VLAN 30)
   |
   +-- tunnel-client.service            -> 127.0.0.1:8080/mcp (Proxmox MCP)
   |                                      -> HTTPS 192.168.10.11:8006
   |                                      -> Proxmox homelab-v3 cluster
   |
   +-- tunnel-client-unraid.service     -> 127.0.0.1:8082/mcp (Unraid MCP)
                                          -> verified HTTPS, nas-prod-01.local:443
                                             resolved to 192.168.30.16
                                          -> nas-prod-01 Unraid GraphQL API
```


Both MCP HTTP listeners are published exclusively to localhost on the observer VM. Inside their Docker networks, the services bind to container interfaces, not exclusively to loopback. Each tunnel has its own configuration, health port, protected credentials, and independently enabled systemd unit. Both tunnel processes run as the existing `tunnel-client` Linux account.

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
|Proxmox ChatGPT plugin|Homelab Proxmox|ChatGPT Plugins|
|Unraid MCP|dinglebear-ai/unraid-rmcp 0.6.3|Docker on observer-prod-01|
|Unraid MCP transport|Streamable HTTP|127.0.0.1:8082/mcp|
|Unraid MCP container|`unraid-mcp`|`/opt/stacks/unraid-mcp/`|
|Unraid tunnel service|OpenAI tunnel-client v0.0.18|`tunnel-client-unraid.service`|
|Unraid tunnel health endpoint|HTTP|127.0.0.1:8083/readyz|
|Unraid tunnel configuration|YAML and protected credential files|`/etc/tunnel-client-unraid/`|
|Unraid ChatGPT plugin|Homelab Unraid|ChatGPT Plugins|

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

## Unraid MCP

### Deployment and Implementation

**Project:** [dinglebear-ai/unraid](https://github.com/dinglebear-ai/unraid) — Rust server under `unraid-rs/`  
**Image:** `ghcr.io/dinglebear-ai/unraid-rmcp:latest`  
**Deployed binary version (verified):** `unraid-rmcp 0.6.3`  
**Container:** `unraid-mcp`  
**Compose file:** `/opt/stacks/unraid-mcp/compose.yaml`  
**Environment/credentials:** `/opt/stacks/unraid-mcp/.env` (mode 0600; excluded from Git)  
**Persistent named volume:** `unraid-mcp_unraid-mcp-data` (mounted at `/data`)  
**MCP transport:** Streamable HTTP with bearer-token authentication  
**Published port:** `127.0.0.1:8082:40010/tcp`

The `latest` tag is intentional for operator-controlled CLI updates; it does **not** automatically pull new releases. The original, outdated `ghcr.io/dinglebear-ai/runraid:latest` image ran v0.2.3 and lacked the required CA-bundle support. The correct Rust image is `ghcr.io/dinglebear-ai/unraid-rmcp:latest`.

The current Compose stack uses `restart: unless-stopped`, `no-new-privileges:true`, a 384 MiB memory limit, a 100-process limit, bounded JSON log retention (10 MiB × 3), and an explicitly localhost-bound published port. The read-only API configuration and MCP allowlist are the substantive privilege controls. No Unraid Docker socket or NAS filesystem is mounted into the container. This stack is not hardened identically to the Proxmox MCP stack; an earlier read-only-container configuration blocked writing runtime logs under `/data` and was removed.

### API, Transport, and TLS

| Setting | Implemented value / role |
|---|---|
| `UNRAID_API_URL` | `https://nas-prod-01.local/graphql` |
| `extra_hosts` | `nas-prod-01.local:192.168.30.16` (container-local name resolution) |
| `UNRAID_API_KEY` | Dedicated Unraid GraphQL key in protected `.env`; not documented in plaintext |
| `UNRAID_API_CA_BUNDLE` | `/certs/unraid.crt` |
| Host certificate path | `/etc/unraid-mcp/unraid.crt`, mounted read-only into container |
| `UNRAID_API_SKIP_TLS_VERIFY` | `false` |
| `UNRAID_RMCP_HOST` / `PORT` | `0.0.0.0` / `40010` inside container; Docker exposes only `127.0.0.1:8082` |
| `UNRAID_RMCP_AUTH_MODE` | `bearer` |
| `UNRAID_RMCP_TOKEN` | MCP client authentication token in `.env`; separate protected copy for tunnel Authorization header |
| `UNRAID_RMCP_PROJECTION` | `atomic` |
| `UNRAID_RMCP_ENABLED_TOOLS` | Explicit 26-action read-only allowlist (below) |

Unraid uses a self-signed server certificate (`CN=nas-prod-01.local`). The observed SHA-256 fingerprint was `67:F9:8F:19:E4:D9:10:51:95:4E:38:55:2D:F7:BA:78:14:49:54:7A:3D:EC:F6:18:2E:59:E9:52:67:CC:DB:9C`. The certificate was checked directly against the NAS and the accessible Services interface; startup logs reported `trusting extra CA certificates from bundle count=1`. A successful authenticated GraphQL query then confirmed the connection works **with** certificate verification enabled. Revalidate identity before replacing this certificate.

The API key was created with 14 `READ_ANY` permissions and no write grants: `ARRAY`, `DASHBOARD`, `DISK`, `DISPLAY`, `DOCKER`, `INFO`, `LOGS`, `ME`, `NETWORK`, `NOTIFICATIONS`, `ONLINE`, `SERVICES`, `SHARE`, `VARS`. This key is an upstream read-only boundary; it is independent of the MCP action filter. The bearer-token MCP authentication mode does not by itself guarantee read-only capabilities, so maintain the explicit allowlist as well.

### Currently Exposed MCP Tools — 26

The 26 atomic tools below were verified through local MCP `tools/list` and in ChatGPT's connected plugin view on October 30, 2026. All appear under the plugin's **Read** category; zero write tools were exposed. Tool discovery and one end-to-end `unraid_info` invocation were verified, **not** every individual action's upstream API permission.

| MCP tool | Read-only purpose |
|---|---|
| `unraid_array` | Array state, capacity, parity status, and disk summary |
| `unraid_disk` | Details for one disk by ID |
| `unraid_disks` | Disk inventory and SMART/temperature summaries |
| `unraid_docker` | Docker container inventory and state |
| `unraid_docker_logs` | Read Docker container log lines |
| `unraid_help` | Help for the currently enabled actions |
| `unraid_info` | Unraid release, kernel, CPU, and RAM |
| `unraid_log_file` | Read an available log file by path |
| `unraid_log_files` | List available log files |
| `unraid_me` | Identity associated with the authenticated API key |
| `unraid_metrics` | CPU, memory, and temperature metrics |
| `unraid_network` | Network URLs and address information |
| `unraid_network_interfaces` | Detailed network interface information |
| `unraid_notifications` | Read notifications and warning summaries |
| `unraid_online` | API reachability/online status |
| `unraid_parity_history` | Historical parity check results |
| `unraid_plugins` | Installed community plugin inventory |
| `unraid_server` | Unraid server identity and reachability metadata |
| `unraid_services` | Unraid system service status |
| `unraid_shares` | Share inventory, capacity, and placement configuration |
| `unraid_status` | Local Rust MCP server status/observability |
| `unraid_system_time` | System time and timezone details |
| `unraid_ups` | UPS devices and readings |
| `unraid_ups_config` | UPS monitoring configuration |
| `unraid_vars` | Unraid variables and configuration metadata |
| `unraid_vms` | Unraid VM inventory and state |

### Available Upstream but NOT Exposed

**Canonical action names** below are from the Rust server's [`ACTIONS` list](https://github.com/dinglebear-ai/unraid/blob/main/unraid-rs/src/mcp/schemas.rs). Atomic tool names add the prefix `unraid_`. This is a snapshot of the upstream source reviewed alongside deployed version 0.6.3; check release changes before modifying the allowlist.

**Additional read-scope actions not exposed — 35:**

`registration`, `flash`, `rclone`, `remote_access`, `connect`, `installed_unraid_plugins`, `is_sso_enabled`, `public_oidc_providers`, `oidc_providers`, `oidc_configuration`, `api_keys`, `api_key_possible_roles`, `api_key_possible_permissions`, `get_available_auth_actions`, `get_api_key_creation_form_schema`, `config`, `settings`, `display`, `customization`, `internal_boot_context`, `owner`, `servers`, `is_fresh_install`, `public_theme`, `time_zone_options`, `assignable_disks`, `plugin_install_operations`, `cloud`, `api_key`, `oidc_provider`, `ups_device_by_id`, `plugin_install_operation`, `validate_oidc_session`, `get_permissions_for_roles`, `preview_effective_permissions`
Some are potentially useful for future observability (`registration`, `flash`, `assignable_disks`, `installed_unraid_plugins`, `ups_device_by_id`, `plugin_install_operations`). Others could reveal sensitive configuration, access details, API-key identities, OIDC authentication information, or remote configuration (`api_keys`, `api_key`, `servers`, `rclone`, `remote_access`, `connect`, and other identity/settings endpoints). They are **not** currently enabled, and the current read-only API key may lack permission for some of them. Do not confuse a read-scope label with a guarantee that the returned data is non-sensitive.

**Write/admin-scope actions not exposed — 83:**

| Category | Not exposed (canonical action names) |
|---|---|
| Array and parity operations (10) | `array_set_state`, `array_add_disk_to_array`, `array_remove_disk_from_array`, `array_mount_array_disk`, `array_unmount_array_disk`, `array_clear_array_disk_statistics`, `parity_check_start`, `parity_check_pause`, `parity_check_resume`, `parity_check_cancel` |
| VM control (7) | `vm_start`, `vm_stop`, `vm_pause`, `vm_resume`, `vm_force_stop`, `vm_reboot`, `vm_reset` |
| Docker lifecycle and organization (21) | `docker_start`, `docker_stop`, `docker_restart`, `docker_pause`, `docker_unpause`, `docker_update_container`, `docker_remove_container`, `docker_update_containers`, `docker_update_all_containers`, `docker_create_folder`, `docker_create_folder_with_items`, `docker_set_folder_children`, `docker_delete_entries`, `docker_move_entries_to_folder`, `docker_move_items_to_position`, `docker_rename_folder`, `docker_update_view_preferences`, `docker_update_autostart_configuration`, `refresh_docker_digests`, `reset_docker_template_mappings`, `sync_docker_template_paths` |
| Notifications and alert state (9) | `delete_archived_notifications`, `archive_notification`, `create_notification`, `archive_notifications`, `unarchive_notifications`, `unread_notification`, `archive_all`, `unarchive_all`, `notify_if_unique` |
| API keys and Connect identity (8) | `api_key_create`, `api_key_add_role`, `api_key_remove_role`, `api_key_delete`, `api_key_update`, `update_server_identity`, `connect_sign_out`, `connect_sign_in` |
| Settings, network access, UPS, and UI (10) | `customization_set_locale`, `customization_set_theme`, `configure_ups`, `update_system_time`, `update_temperature_config`, `setup_remote_access`, `enable_dynamic_remote_access`, `update_api_settings`, `update_settings`, `update_ssh_settings` |
| Plugins and onboarding (14) | `unraid_plugins_install_plugin`, `unraid_plugins_install_language`, `onboarding_complete_onboarding`, `onboarding_reset_onboarding`, `onboarding_bypass_onboarding`, `onboarding_clear_onboarding_override`, `onboarding_close_onboarding`, `onboarding_open_onboarding`, `onboarding_resume_onboarding`, `onboarding_refresh_internal_boot_context`, `onboarding_set_onboarding_override`, `onboarding_create_internal_boot_pool`, `add_plugin`, `remove_plugin` |
| Remote backups (3) | `rclone_create_r_clone_remote`, `rclone_delete_r_clone_remote`, `initiate_flash_backup` |
| Other (1) | `recalculate_overview` |

**None of the 83 write/admin actions is part of the deployed MCP allowlist.** All operational changes to the array, parity, Docker containers, VMs, plugins, settings, API keys, and remote access remain excluded. In addition, the upstream Unraid key lacks write permissions.

**References and expansion policy:**

- [Rust MCP tool reference](https://github.com/dinglebear-ai/unraid/blob/main/unraid-rs/README.md#mcp-tool-reference) — documented query and management capabilities.
- [Canonical action names and scope classification](https://github.com/dinglebear-ai/unraid/blob/main/unraid-rs/src/mcp/schemas.rs) — source of truth for action availability and read/write classification.
- [MCP selector/allowlist implementation](https://github.com/dinglebear-ai/unraid/blob/main/unraid-rs/src/mcp/tool_filter.rs) — controls exposure and fail-closed validation for unknown selectors.
- Additional read tools require an explicit requirement, sensitivity and upstream-permission review, controlled Compose change, container recreation, exact `tools/list` comparison, and live read-only test. The operator must authorize any consequential capability expansion. Do not enable writes merely to make troubleshooting easier.
- `UNRAID_RMCP_ENABLED_TOOLS` restricts exposed MCP actions. It does not change the `runraid` command-line client's capabilities. Keep the dedicated Unraid key read-only and never put its value in documentation.

---

## OpenAI Secure MCP Tunnel

### Deployment

**Project:** [https://github.com/openai/tunnel-client](https://github.com/openai/tunnel-client)  
**Installed version:** 0.0.18  
**Installation directory:** `/opt/tunnel-client`  
**Binary:** `/opt/tunnel-client/tunnel-client`  
**Existing Proxmox service:** `tunnel-client.service`  
**Proxmox configuration:** `/etc/tunnel-client/config.yaml`  
**Unraid service:** `tunnel-client-unraid.service`  
**Unraid configuration:** `/etc/tunnel-client-unraid/config.yaml`

The official Linux AMD64 release was downloaded and its SHA-256 checksum successfully verified before installation.

The matching bundled `cloudflared` binary and manifest were installed alongside the tunnel client.

### Tunnel Identities

|Field|Value|
|---|---|
|Proxmox tunnel name / ID|homelab-proxmox — `tunnel_6aded6df36348191b926a1b2fed52974`|
|Unraid tunnel name / ID|homelab-unraid — `tunnel_6ae54db66f7c8191a73960ac6eb77981`|
|Control plane|OpenAI Platform|
|Proxmox MCP target / health|`http://127.0.0.1:8080/mcp` / `127.0.0.1:8081`|
|Unraid MCP target / health|`http://127.0.0.1:8082/mcp` / `127.0.0.1:8083`|
|Authentication to OpenAI|Restricted runtime API key|
|Authentication to MCP|Static bearer token|

Each tunnel uses a dedicated OpenAI runtime API key with restricted Tunnel Read and Use permissions. The Unraid runtime key was saved to Bitwarden and installed on the observer in a protected file. The Unraid API key is also stored in Bitwarden. Confirm that the separately generated `UNRAID_RMCP_TOKEN` is retained in Bitwarden before relying on it for recovery; its separate storage was not verified during deployment.

Administrative tunnel-management credentials are not required by the long-running daemon.

### Service Configuration

Both tunnel clients run as separate native systemd services using the same `tunnel-client` service account and binary.

**Systemd units:** `/etc/systemd/system/tunnel-client.service` and `/etc/systemd/system/tunnel-client-unraid.service`

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

Both tunnel services were verified `active` and `enabled`. Their unit files include `Restart=always` with a 5-second delay and `WantedBy=multi-user.target`; a full VM reboot/startup test has not been performed. Neither service depends on an interactive SSH session.

### Protected Configuration Files

|Path|Purpose|Ownership|Permissions|
|---|---|---|---|
|`/etc/tunnel-client/config.yaml`|Proxmox tunnel configuration|root:tunnel-client|0640|
|`/etc/tunnel-client/openai-runtime.key`|Proxmox OpenAI runtime API key|root:tunnel-client|0640|
|`/etc/tunnel-client/mcp-authorization`|Proxmox MCP bearer Authorization header|root:tunnel-client|0640|
|`/etc/tunnel-client/`|Proxmox protected directory|root:tunnel-client|0750|
|`/etc/tunnel-client-unraid/config.yaml`|Unraid tunnel configuration|root:tunnel-client|0640|
|`/etc/tunnel-client-unraid/openai-runtime.key`|Unraid OpenAI runtime API key|root:tunnel-client|0640|
|`/etc/tunnel-client-unraid/mcp-authorization`|Unraid MCP bearer Authorization header|root:tunnel-client|0640|
|`/etc/tunnel-client-unraid/`|Unraid protected directory|root:tunnel-client|0750|

Each MCP authorization file contains the complete `Bearer` header value for its own server. The corresponding tunnel client reads the file and sends it to the local MCP server.

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

### Unraid GraphQL API Access

The observer reaches Unraid over the Services VLAN at `192.168.30.16:443`, using hostname `nas-prod-01.local` and a verified certificate bundle. The API URL is `https://nas-prod-01.local/graphql`. The Unraid Docker container maps the hostname to the Services IP using `extra_hosts`, not a general DNS override.

No additional inter-VLAN management firewall exception or public listener was introduced for Unraid. Read-only requests were authenticated successfully over this route. Do not broaden the observer's access to Management VLAN as part of Unraid troubleshooting.


### OpenAI Connectivity

Both tunnel clients initiate outbound HTTPS traffic to OpenAI.

|Traffic|Purpose|
|---|---|
|Observer → api.openai.com:443|Secure MCP Tunnel control plane|
|Tunnel client → 127.0.0.1:8080|Local Proxmox MCP requests|
|Tunnel client → 127.0.0.1:8082|Local Unraid MCP requests|
|Unraid MCP → 192.168.30.16:443|Verified HTTPS to Unraid GraphQL API|
|Proxmox MCP → 192.168.10.11:8006|Proxmox API access|

No public listener, inbound port forwarding, or general inter-VLAN management access was introduced.

---

## ChatGPT Integration

### Plugins

**Name:** Homelab Proxmox  
**Description:** Read-only monitoring and diagnostics for Homelab v3 Proxmox infrastructure.  
**Connection:** Tunnel  
**Tunnel:** homelab-proxmox  
**ChatGPT authentication option:** No authentication  
**Status:** Connected and end-to-end verified

The ChatGPT plugin's _No authentication_ option refers to the absence of a separate OAuth login flow in ChatGPT. It does not disable the MCP server's bearer-token requirement.

Authentication to the local MCP endpoint is supplied by the protected tunnel-client configuration.

**Name:** Homelab Unraid  
**Description:** Read-only monitoring and diagnostics for Homelab v3 Unraid NAS infrastructure.  
**Connection:** Tunnel  
**Tunnel:** homelab-unraid (`tunnel_6ae54db66f7c8191a73960ac6eb77981`)  
**ChatGPT authentication option:** No authentication  
**Status:** Connected; 26 read tools/zero write tools discovered; `unraid_info` called successfully directly from ChatGPT.

For both plugins, ChatGPT's *No authentication* choice means no separate user OAuth flow. Local MCP HTTP requests still require a bearer token supplied by each tunnel's protected credential file.


### Usage

ChatGPT can automatically select the Homelab Proxmox or Homelab Unraid plugin when the relevant live infrastructure information is needed.

Each integration can also be selected explicitly using `@Homelab Proxmox` or `@Homelab Unraid`.

Typical supported requests include:

- Check Proxmox cluster health and quorum.
    
- Inspect nodes, VMs, and LXCs.
    
- Review resource utilization.
    
- Investigate task failures.
    
- Inspect guest configurations.
    
- Review Proxmox backup coverage.
    
- Check available package-update information.
    

Neither plugin currently exposes write operations. For Unraid, the MCP exposes exactly 26 allowlisted read tools and uses a separate read-only upstream API key. Do not assume that an *individual* read action returns nothing sensitive (for example logs or configuration variables).

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

### Unraid MCP Verification — October 30, 2026 (CDT)

| Test | Result |
|---|---|
| Correct Rust image and binary | `ghcr.io/dinglebear-ai/unraid-rmcp:latest`, `unraid-rmcp 0.6.3` |
| Docker Compose configuration | Validated |
| Unraid container | Healthy; `127.0.0.1:8082->40010/tcp` |
| CA bundle initialization | `trusting extra CA certificates from bundle count=1` |
| Local bearer-authenticated `unraid_info` | Passed; Unraid OS 7.3.2 |
| Local bearer-authenticated `unraid_array` | Passed; surfaced disabled first parity disk |
| Exact local MCP tool inventory | 26 expected, 26 exposed, none missing/unexpected |
| Unraid tunnel `doctor` | `RESULT ok` |
| Unraid tunnel `/readyz` | Returned `ready` |
| `tunnel-client-unraid.service` | Active and enabled at boot |
| ChatGPT plugin discovery | Connected; 26 read tools, zero write tools |
| Live `unraid_info` from ChatGPT | Passed (hostname `nas-prod-01`, Unraid 7.3.2) |

A healthy container, a ready tunnel, and connected-plugin discovery each check different layers. End-to-end verification was achieved through the live ChatGPT invocation. Full per-tool testing, automatic startup through an actual observer reboot, and recovery/restore testing remain outstanding.


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
- `/opt/stacks/unraid-mcp/compose.yaml` — Unraid MCP Compose configuration; should be tracked in `homelab-infra` once reviewed/committed (commit not verified).
- `/opt/stacks/unraid-mcp/.env` — Git-excluded Unraid MCP API key and bearer token.
- `/etc/unraid-mcp/unraid.crt` — Pinned trusted Unraid certificate bundle.
- `/etc/tunnel-client/config.yaml` — Git-excluded tunnel configuration.
- `/etc/tunnel-client/openai-runtime.key` — Git-excluded OpenAI credential.
- `/etc/tunnel-client/mcp-authorization` — Git-excluded MCP bearer credential.
- `/etc/tunnel-client-unraid/config.yaml` — Git-excluded Unraid tunnel configuration.
- `/etc/tunnel-client-unraid/openai-runtime.key` — Git-excluded Unraid tunnel runtime credential.
- `/etc/tunnel-client-unraid/mcp-authorization` — Git-excluded Unraid MCP bearer credential.
- `/etc/systemd/system/tunnel-client.service` — Native Proxmox tunnel systemd unit.
- `/etc/systemd/system/tunnel-client-unraid.service` — Native Unraid tunnel systemd unit.
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

### Unraid MCP

The Unraid MCP container is managed independently with Docker Compose under `/opt/stacks/unraid-mcp`. Its Docker image uses the `latest` tag, with operator-controlled updates and post-update version/tool-inventory verification. The existing Unraid NAS is **not** a Docker management target of this stack; it is queried remotely through GraphQL.


### Tunnel Client

The tunnel client is managed through systemd:

`tunnel-client.service` and `tunnel-client-unraid.service`

Updates are performed through an explicitly verified official release rather than Docker Compose.

The tunnel services and both MCP containers are separate processes; check the affected component before taking action. Both tunnels reuse the same installed binary, so a tunnel-client binary upgrade affects both integrations.

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

# Check Unraid MCP and its independent tunnel
cd /opt/stacks/unraid-mcp
sudo docker compose ps
sudo docker compose logs --tail 50 unraid-mcp
curl -fsS http://127.0.0.1:8083/readyz
systemctl is-enabled tunnel-client-unraid
systemctl is-active tunnel-client-unraid
```

Passing health checks confirms component availability, but a live MCP tool call remains the definitive end-to-end verification.

Full update, troubleshooting, service recovery, and credential-rotation procedures belong in `runbooks/ai-observer-operations.md`.

---

## Security Boundaries

The initial implementation deliberately minimizes privileges.

1. The Proxmox API account has audit-only permissions; the Unraid GraphQL API key has only the documented 14 `READ_ANY` permission families.
    
2. Proxmox MCP write, delete, and execution capabilities are disabled. Unraid MCP exposes an explicit 26-action read-only allowlist with no write/admin actions.
    
3. Both MCP endpoints are published to localhost only.
    
4. The observer-to-Proxmox firewall exception is restricted to one source, one destination, and TCP 8006.
    
5. The OpenAI tunnel is outbound-only.
    
6. OpenAI runtime credentials are restricted to tunnel Read and Use.
    
7. The tunnel service runs without root or Docker-group privileges.
    
8. Secrets are kept outside Git and are not reproduced in documentation. Each tunnel has its own restricted runtime key and protected MCP bearer header file.
    
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
|Unraid MCP — read-only local and ChatGPT access|Implemented and end-to-end verified (26 tools)|
|Docker-host MCP outside Unraid|Planned|
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

- `architecture/storage/nas-prod-01.md` — Unraid NAS configuration, storage inventory, and unresolved disabled primary-parity incident.
    
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
    
- Both Proxmox and Unraid MCP integrations are intentionally read-only. The Unraid MCP has 26 enabled actions with 118 other canonical actions withheld; see its inventory above.
    
- Unraid MCP is implemented and verified end-to-end. A separate Docker-host MCP remains planned.
    
- The October 30 observer VM backup task succeeded; independent snapshot verification, credential-recovery checks, and an isolated restoration test remain outstanding.
    
- Documentation should be updated when implementations change, but planned integrations must not be recorded as completed until tested.