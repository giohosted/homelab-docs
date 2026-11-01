# AI Observer — Proxmox, Unraid & Komodo MCP with OpenAI Secure MCP Tunnels

**Role:** AI-assisted, read-only infrastructure monitoring and diagnostics  
**Host:** observer-prod-01 (192.168.30.17)  
**Proxmox Host:** pve-prod-01 (192.168.10.11)  
**Proxmox VMID:** 107  
**Status:** Active — Proxmox, Unraid, and Komodo integrations verified end-to-end  
**Last Updated:** 2026-11-01

---

## Overview
AI Observer provides controlled access to Homelab v3 infrastructure from AI assistants, initially ChatGPT, through Model Context Protocol (MCP).

The observer runs on a dedicated Ubuntu virtual machine rather than sharing a Docker host with production applications. This separates AI integration services from existing infrastructure workloads and provides an independent security boundary.

**Implemented and end-to-end verified integrations:**

- **Proxmox MCP:** Read-only monitoring of the Homelab v3 Proxmox cluster; deployed server version `0.5.1`, **84 read tools**.
- **Unraid MCP:** Read-only monitoring of `nas-prod-01` over the Unraid GraphQL API; Rust MCP `0.6.3`, **26 read tools**.
- **Komodo MCP:** Read-only monitoring of Docker infrastructure on `docker-prod-01` via Komodo Core; MCP `1.5.0`, Komodo `2.3.4`, **40 read tools** with eight explicit exclusions.
- **OpenAI Secure MCP Tunnels:** Three separate outbound native systemd services, each with its own tunnel ID, restricted runtime credential, configuration, and health port.
- **ChatGPT plugins:** `Homelab Proxmox`, `Homelab Unraid`, and `Homelab Komodo`; all three connected and exercised using a live tool call. **Total discovered: 150 read tools, zero write tools.**

The Proxmox integration was verified October 29–30 and its compiled version/tool inventory rechecked November 1, 2026. The Unraid integration was deployed and verified October 30, 2026. Komodo integration and its third tunnel were verified end-to-end November 1, 2026. **Docker-host MCP is implemented, not merely planned.** Additional Docker hosts are not automatically enrolled: the verified Komodo server is `docker-prod-01`.

## Architecture
### Request Flow

```text
ChatGPT
  |-- Homelab Proxmox plugin -- OpenAI outbound tunnel: homelab-proxmox
  |     tunnel-client.service -> 127.0.0.1:8080/mcp
  |     Proxmox MCP -> HTTPS 192.168.10.11:8006 -> homelab-v3 cluster
  |
  |-- Homelab Unraid plugin -- OpenAI outbound tunnel: homelab-unraid
  |     tunnel-client-unraid.service -> 127.0.0.1:8082/mcp
  |     Unraid MCP -> verified HTTPS nas-prod-01.local:443 (192.168.30.16)
  |     -> nas-prod-01 GraphQL API
  |
  `-- Homelab Komodo plugin -- OpenAI outbound tunnel: homelab-komodo
        tunnel-client-komodo.service -> 127.0.0.1:8084/mcp
        Komodo MCP -> HTTPS komodo.giohosted.com (192.168.30.11)
        -> Komodo Core/Periphery -> Docker on docker-prod-01

All three MCP services and tunnel clients run on:
observer-prod-01 (192.168.30.17, VLAN 30, Proxmox VMID 107)
```

All three MCP HTTP listeners are published **only to loopback on the observer host**. Within the Docker network, the containerized MCP services bind to container interfaces. Each tunnel uses its own localhost health endpoint, protected OpenAI runtime key, and independent systemd unit. All tunnel processes run as the restricted `tunnel-client` Linux account using one installed binary.

No public MCP hostname, Traefik route, port forwarding, or inbound internet firewall rule is required. The dedicated Komodo MCP talks to the existing Komodo Core HTTPS endpoint; it does not require a second Docker socket or privileged observer access.

### Component Inventory

| Component | Technology | Location / endpoint |
|---|---|---|
| Observer VM | Ubuntu Server 26.04.1 LTS | `observer-prod-01`, VM 107 on `pve-prod-01` |
| Proxmox MCP | mattoddie/proxmox-mcp **0.5.1** | `/opt/stacks/proxmox-mcp/`, `127.0.0.1:8080/mcp` |
| Proxmox MCP health | HTTP | `127.0.0.1:8080/health` |
| Proxmox tunnel | OpenAI tunnel-client 0.0.18 | `tunnel-client.service`, `127.0.0.1:8081/readyz` |
| Proxmox tunnel configuration | YAML + key + bearer file | `/etc/tunnel-client/` |
| Proxmox ChatGPT plugin | Homelab Proxmox | 84 read tools |
| Unraid MCP | dinglebear-ai/unraid-rmcp **0.6.3** | `/opt/stacks/unraid-mcp/`, `127.0.0.1:8082/mcp` |
| Unraid tunnel | OpenAI tunnel-client 0.0.18 | `tunnel-client-unraid.service`, `127.0.0.1:8083/readyz` |
| Unraid tunnel configuration | YAML + key + bearer file | `/etc/tunnel-client-unraid/` |
| Unraid ChatGPT plugin | Homelab Unraid | 26 read tools |
| Komodo MCP | MP-Tool/komodo-mcp-server **1.5.0** | `/opt/stacks/komodo-mcp/`, `127.0.0.1:8084/mcp` |
| Komodo MCP readiness | HTTP | `127.0.0.1:8084/ready` |
| Komodo tunnel | OpenAI tunnel-client 0.0.18 | `tunnel-client-komodo.service`, `127.0.0.1:8085/readyz` |
| Komodo tunnel configuration | YAML + runtime key, no MCP bearer | `/etc/tunnel-client-komodo/` |
| Komodo ChatGPT plugin | Homelab Komodo | 40 read tools |
| Komodo Core | Komodo **2.3.4** | `docker-prod-01`, `https://komodo.giohosted.com` |

**Important distinction:** OpenAI Secure MCP Tunnel is separate from the homelab's application-oriented Cloudflare Tunnel deployment on `docker-prod-01`; it is not a general-purpose public reverse proxy.

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
**Compiled running server version:** `0.5.1` (verified November 1, 2026 with `/proxmox-mcp version`)  
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

### Verified Proxmox MCP Tool Inventory — 84 Read Tools
The running container reported **`proxmox-mcp 0.5.1`** from `sudo docker exec proxmox-mcp /proxmox-mcp version` on November 1, 2026. The installed ChatGPT Homelab Proxmox plugin exposes **84 tools, all classified Read**, with no write, delete, or execution tools. The nine toolsets below match `PROXMOX_TOOLSETS=cluster,nodes,vms,containers,agent,snapshots,storage,backup,tasks` in the documented configuration. These are **discovered capabilities**, not proof that all 84 upstream API calls have individually succeeded.

| Toolset | Read tools | Purpose |
|---|---:|---|
| `cluster` | 11 | Cluster, version, quorum, nodes, guests, metric configuration, logs, schedule simulation |
| `nodes` | 25 | Node status, journals, network, services, package updates, ACME/certificates, time |
| `vms` | 7 | QEMU VM identity, configuration, status, metrics and feature/migration checks |
| `containers` | 7 | LXC inventory, configurations, interfaces, stats and feature/migration checks |
| `agent` | 5 | Read-only QEMU guest-agent information, networking, users, storage and memory |
| `snapshots` | 3 | List and inspect guest snapshots, including cluster-wide snapshot inventory |
| `storage` | 14 | Storage inventories, disk/SMART, volumes, ZFS, LVM and available templates |
| `backup` | 8 | Jobs, backup listings, defaults, coverage, and read-only prune preview |
| `tasks` | 4 | Task inventory, task details, logs and read-only waiting on task status |
| **Total** | **84** | **Currently exposed through ChatGPT** |

**Exact exposed names (as discovered November 1, 2026):**

**`cluster` (11):**
- `proxmox_get_cluster_config`
- `proxmox_get_cluster_log`
- `proxmox_get_cluster_options`
- `proxmox_get_cluster_status`
- `proxmox_get_next_vmid`
- `proxmox_get_version`
- `proxmox_list_guests`
- `proxmox_list_metric_servers`
- `proxmox_list_nodes`
- `proxmox_list_resources`
- `proxmox_simulate_schedule`

**`nodes` (25):**
- `proxmox_get_acme_account`
- `proxmox_get_acme_challenge_schema`
- `proxmox_get_acme_plugin`
- `proxmox_get_node_certificates`
- `proxmox_get_node_config`
- `proxmox_get_node_dns`
- `proxmox_get_node_hosts`
- `proxmox_get_node_journal`
- `proxmox_get_node_netstat`
- `proxmox_get_node_network_interface`
- `proxmox_get_node_package_versions`
- `proxmox_get_node_report`
- `proxmox_get_node_stats`
- `proxmox_get_node_status`
- `proxmox_get_node_subscription`
- `proxmox_get_node_syslog`
- `proxmox_get_node_time`
- `proxmox_get_package_changelog`
- `proxmox_list_acme_accounts`
- `proxmox_list_acme_directories`
- `proxmox_list_acme_plugins`
- `proxmox_list_node_network`
- `proxmox_list_node_repositories`
- `proxmox_list_node_services`
- `proxmox_list_node_updates`

**`vms` (7):**
- `proxmox_check_vm_feature`
- `proxmox_check_vm_migration`
- `proxmox_get_vm`
- `proxmox_get_vm_cloudinit`
- `proxmox_get_vm_config`
- `proxmox_get_vm_stats`
- `proxmox_list_vms`

**`containers` (7):**
- `proxmox_check_container_feature`
- `proxmox_check_container_migration`
- `proxmox_get_container`
- `proxmox_get_container_config`
- `proxmox_get_container_interfaces`
- `proxmox_get_container_stats`
- `proxmox_list_containers`

**`agent` (5):**
- `proxmox_get_guest_filesystems`
- `proxmox_get_guest_info`
- `proxmox_get_guest_memory_blocks`
- `proxmox_get_guest_network`
- `proxmox_get_guest_users`

**`snapshots` (3):**
- `proxmox_get_snapshot`
- `proxmox_list_all_snapshots`
- `proxmox_list_snapshots`

**`storage` (14):**
- `proxmox_get_disk_smart`
- `proxmox_get_storage`
- `proxmox_get_storage_stats`
- `proxmox_get_volume`
- `proxmox_get_zfs_pool`
- `proxmox_list_appliance_templates`
- `proxmox_list_directory_mounts`
- `proxmox_list_disks`
- `proxmox_list_lvm`
- `proxmox_list_lvm_thin`
- `proxmox_list_storage`
- `proxmox_list_storage_content`
- `proxmox_list_zfs_pools`
- `proxmox_scan_storage`

**`backup` (8):**
- `proxmox_get_backup_config`
- `proxmox_get_backup_job`
- `proxmox_get_vzdump_defaults`
- `proxmox_list_backup_files`
- `proxmox_list_backup_jobs`
- `proxmox_list_backups`
- `proxmox_list_unbacked_guests`
- `proxmox_preview_backup_prune`

**`tasks` (4):**
- `proxmox_get_task`
- `proxmox_get_task_log`
- `proxmox_list_tasks`
- `proxmox_wait_for_task`

### Proxmox MCP Exclusions and Read-Only Policy
The [deployed-version tool reference](https://github.com/mattoddie/proxmox-mcp/blob/v0.5.1/docs/tools.md) and [v0.5.1 configuration reference](https://github.com/mattoddie/proxmox-mcp/blob/v0.5.1/docs/configuration.md) describe 19 upstream toolsets and 300+ potential tools across capability levels. **Do not conflate upstream availability with deployed registration.** This deployment enables only the nine groups listed above. The remaining **ten entire toolsets** are not registered:

| Disabled toolset | Scope of withheld capabilities / reason |
|---|---|
| `access` | Users, roles, tokens, ACLs, realms and MFA; sensitive identity metadata and unnecessary for operational monitoring |
| `firewall` | Datacenter/node/guest firewall rules and management; privileged security configuration and unnecessary detailed disclosure |
| `ha` | HA configuration and resource management; not included in current observer monitoring scope |
| `sdn` | SDN zones, VNets, IPAM, routing/management metadata; unnecessary detailed network configuration |
| `ceph` | Ceph topology and management; not required for deployed storage observability |
| `pools` | Proxmox resource pool listings and changes; not part of current monitoring scope |
| `replication` | Replication jobs, logs and operations; not part of current monitoring scope |
| `notifications` | Notification targets, matchers, and management; endpoints can contain sensitive configuration |
| `hardware` | Host PCI/USB/device mappings; out of current scope |
| `raw` | General Proxmox API request escape hatch; explicitly excluded to prevent bypassing curated tool safety boundaries |

Within the **nine enabled toolsets**, tools marked write/delete/exec in upstream v0.5.1 are **also not registered** because `PROXMOX_ALLOW_WRITES=false`, `PROXMOX_ALLOW_DELETES=false`, and `PROXMOX_ALLOW_EXEC=false`. Consequently, start/stop/reconfigure/clone/migrate guests, create/delete snapshots or backups, run guest commands, edit networking/storage, and other changes are unavailable. The underlying `observer@pve!proxmox-mcp` identity retains `PVEAuditor` privileges, providing an independent upstream read-only boundary.

With v0.5.1 the upstream default `all` would expose more tools than this deployment; the existing explicit toolset policy intentionally limits information disclosure, prompt/tool complexity and blast radius. **No inference of 300+ locally deployed tools is valid.** A new upstream release or change to `PROXMOX_TOOLSETS` requires an exact `tools/list` audit, not only a version check. Read operations may still disclose sensitive infrastructure metadata (for example node reports, certificates, network details, VM configurations or logs); do not paste raw outputs into untrusted services.

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

## Komodo MCP

### Deployment and Verified Connectivity

**Project:** [MP-Tool/komodo-mcp-server](https://github.com/MP-Tool/komodo-mcp-server)  
**Image:** `ghcr.io/mp-tool/komodo-mcp-server:latest`  
**Deployed MCP version:** `1.5.0` (verified via `/ready` and live `komodo_health_check`)  
**Komodo Core version:** `2.3.4`  
**Container:** `komodo-mcp`  
**Compose:** `/opt/stacks/komodo-mcp/compose.yaml`  
**Protected environment:** `/opt/stacks/komodo-mcp/.env`, owned by `gio:gio` with mode `0600` at deployment  
**Local MCP endpoint:** `http://127.0.0.1:8084/mcp`, Docker published only on loopback (`127.0.0.1:8084:8000/tcp`)  
**Readiness:** `http://127.0.0.1:8084/ready`  
**Komodo API:** `https://komodo.giohosted.com` (resolved to `192.168.30.11` from observer at verification)

The image uses a manually managed mutable `latest` tag; it does not automatically update. The Compose security settings are `restart: unless-stopped`, `no-new-privileges:true`, `cap_drop: [ALL]`, `pids_limit: 100`, `mem_limit: 384m`, and bounded JSON logs (10 MiB × 3). No Docker socket is mounted into the observer MCP container. **Komodo Core and Periphery remain on `docker-prod-01`, not the observer.**

A dedicated Komodo user, `ai-observer-komodo`, holds Read permissions for monitored resources and Logs permissions for Server, Stack and Deployment. It is not an administrator and has no Inspect, Terminal, Attach, Processes, or create privileges. Its dedicated API key named `ai-observer-mcp` and secret are stored in the observer's protected `.env`; **do not publish them**. The local MCP queries Komodo Core over HTTPS with these credentials; the ChatGPT tool calls do not contact Docker directly.

### Komodo MCP Authentication and Security

| Setting | Implemented value / purpose |
|---|---|
| `MCP_TRANSPORT` | `http` |
| `MCP_PORT` | `8000` (in container) |
| `MCP_BIND_HOST` | `0.0.0.0` inside the container; host publication is loopback only |
| `MCP_AUTH_ENABLED` | `false` — intentional for this deployment; no separate bearer header on local MCP |
| `MCP_SECRET_SCRUB_ENABLED` | `true`; best-effort output scrubbing only |
| `MCP_TOOLS_EXCLUDED_TOOLS` | Eight explicitly withheld tools listed below |
| `LOG_LEVEL` | `info` |
| OpenAI tunnel MCP Authorization header | **Not configured**, unlike Proxmox and Unraid |

In the deployed upstream implementation, anonymous HTTP MCP clients are restricted to its read-only tool scope; write/manage/operate tools were not discovered. This **does not authenticate local MCP callers**: any process/user able to access observer loopback port `8084` may invoke exposed read tools. Localhost binding, the dedicated non-admin upstream account, disabled writes, selected tool exclusions, the observer OS boundary, and private OpenAI tunnel are distinct protections. A tool being labelled **Read** does not mean its output cannot include private information. Retained `*_info` tools can return whole resource configurations; operators deliberately chose to retain them for troubleshooting. Logs, scripts, network configuration, variables and metadata should always be reviewed before external sharing. Upstream scrubbing is defense-in-depth, **not** a guarantee that every secret will be removed.

### Current Resource Inventory and ChatGPT Verification

On November 1, 2026, local MCP health and server/container list calls passed. The dedicated account retrieved one Komodo server (`docker-prod-01`, state `Ok`) and **26 Docker containers**. A local resource-count audit found **9 stacks, 3 procedures, 1 builder**, and zero deployments, builds, repos, actions, alerters, resource syncs, swarms or variables at the time of checking. These are time-specific observations, not guaranteed current counts. The OpenAI tunnel and ChatGPT plugin then discovered **exactly 40 tools, all Read, zero Write**, and `komodo_health_check` plus `komodo_container_list` were successfully invoked live from ChatGPT.

### Exposed Komodo MCP Tools — 40

The exact ChatGPT-discovered tool inventory is grouped below. All entries are **tool names**, not guarantees of success for every resource or permission.

| Group | Count | Tool names |
|---|---:|---|
| Connectivity / server | 4 | `komodo_health_check`, `komodo_server_list`, `komodo_server_info`, `komodo_server_stats` |
| Containers and logs | 3 | `komodo_container_list`, `komodo_container_logs`, `komodo_container_search_logs` |
| Stacks and deployments | 3 | `komodo_stack_list`, `komodo_deployment_list`, `komodo_deployment_info` |
| Builds, repositories and builders | 7 | `komodo_build_list`, `komodo_build_info`, `komodo_build_logs`, `komodo_repo_list`, `komodo_repo_info`, `komodo_builder_list`, `komodo_builder_info` |
| Procedures and actions | 4 | `komodo_procedure_list`, `komodo_procedure_info`, `komodo_action_list`, `komodo_action_info` |
| Alerts / updates / resource sync | 6 | `komodo_alerter_list`, `komodo_alerter_info`, `komodo_update_list`, `komodo_update_info`, `komodo_resource_sync_list`, `komodo_resource_sync_info` |
| Docker objects and Swarm | 8 | `komodo_docker_image_list`, `komodo_docker_network_list`, `komodo_docker_network_inspect`, `komodo_docker_volume_list`, `komodo_swarm_list`, `komodo_swarm_info`, `komodo_swarm_nodes_list`, `komodo_swarm_services_list` |
| Identities, variables and tags | 5 | `komodo_user_list_api_keys`, `komodo_variable_list`, `komodo_variable_info`, `komodo_tag_list`, `komodo_tag_info` |
| **Total** | **40** | **All exposed as Read in ChatGPT; zero write tools** |

`komodo_container_logs` and `komodo_container_search_logs` remain intentionally enabled for useful diagnostics. `komodo_server_info`, `komodo_procedure_info`, and `komodo_builder_info` also remain enabled following the operator's explicit decision; these could reveal detailed connection or procedure configuration. `komodo_user_list_api_keys` returns key metadata, not a verified secret dump; review actual outputs carefully.

### Explicitly Excluded Komodo Tools — 8

| Excluded tool | Reason |
|---|---|
| `komodo_container_inspect` | Full Docker Inspect may return plaintext `Config.Env`, including Cloudflare and WireGuard secrets; confirmed in local audit. |
| `komodo_exec` | Arbitrary commands / execution path; outside the approved read-only model. |
| `komodo_docker_image_inspect` | Image configuration and labels could contain credentials. |
| `komodo_docker_volume_inspect` | Volume driver options could contain credential or endpoint information. |
| `komodo_docker_image_history` | Image build-step history may contain embedded secrets. |
| `komodo_stack_info` | Full stack/Compose configuration; local audit showed redaction, but operator elected not to rely on best-effort scrub for full Compose details. |
| `komodo_toml_export_all` | Bulk export of Komodo configuration; excessive disclosure surface. |
| `komodo_toml_export_resources` | Resource configuration export; excessive disclosure surface. |

Additionally, the upstream MCP's anonymous-HTTP read-only policy and dedicated non-admin Komodo account prevent exposure/use of management, deployment, destructive, and execution tools. The eight exclusions are **read-surface restrictions**, not the complete list of upstream write tools. Re-check all 40 names, exclusions, and upstream account permissions after updating the MCP image or changing configuration.

---

## OpenAI Secure MCP Tunnel
### Deployment

**Project:** [https://github.com/openai/tunnel-client](https://github.com/openai/tunnel-client)  
**Installed binary version:** `0.0.18` (verified during initial deployment)  
**Installation directory:** `/opt/tunnel-client/`  
**Binary:** `/opt/tunnel-client/tunnel-client`  
**Service account:** `tunnel-client` (noninteractive; not a member of the Docker group)

The original official Linux AMD64 release and its SHA-256 checksum were verified at installation; the matching `cloudflared` companion and manifest were installed with it. All three native services reuse that one installed binary. A binary update can affect **all three** integrations.

### Tunnel Identities

| Integration | Tunnel name | Tunnel ID | Local MCP target | Readiness endpoint |
|---|---|---|---|---|
| Proxmox | `homelab-proxmox` | `tunnel_6aded6df36348191b926a1b2fed52974` | `127.0.0.1:8080/mcp` | `127.0.0.1:8081/readyz` |
| Unraid | `homelab-unraid` | `tunnel_6ae54db66f7c8191a73960ac6eb77981` | `127.0.0.1:8082/mcp` | `127.0.0.1:8083/readyz` |
| Komodo | `homelab-komodo` | `tunnel_6ae6e984758881919f5c308207bab3d3` | `127.0.0.1:8084/mcp` | `127.0.0.1:8085/readyz` |

Each tunnel uses its own restricted OpenAI runtime API key with **Tunnels Read + Use**, not management privileges. The Komodo runtime key is named `homelab-komodo-tunnel` and was saved in Bitwarden. The Unraid runtime key and Unraid API key were saved in Bitwarden; separately retained Bitwarden storage of `UNRAID_RMCP_TOKEN` is **not yet verified**. Each tunnel has its own OpenAI identity, distinct from local MCP authentication.

### Service Configuration

All three tunnel processes are independent native systemd services. The **Komodo unit was inspected/verified** and follows the existing Unraid unit settings:

| Setting | Value |
|---|---|
| Service unit files | `/etc/systemd/system/tunnel-client.service`, `/etc/systemd/system/tunnel-client-unraid.service`, `/etc/systemd/system/tunnel-client-komodo.service` |
| `User` / `Group` | `tunnel-client:tunnel-client` |
| `WorkingDirectory` | `/opt/tunnel-client` |
| `ExecStart` | `/opt/tunnel-client/tunnel-client run --config /etc/<respective-tunnel-directory>/config.yaml` |
| `Restart` / `RestartSec` | `always` / `5` seconds |
| `NoNewPrivileges` | `true` |
| `UMask` | `0077` |
| `WantedBy` | `multi-user.target` |

On November 1, 2026, `tunnel-client-komodo.service` returned `active` and `enabled`, and `/readyz` returned HTTP 200 `ready`. The prior Proxmox/Unraid tunnels were previously verified operational. A complete observer VM reboot and automatic three-tunnel recovery test remains outstanding.

### Protected Configuration Files

| Path | Purpose | Ownership | Mode |
|---|---|---|---|
| `/etc/tunnel-client/` | Proxmox tunnel directory | `root:tunnel-client` | `0750` |
| `/etc/tunnel-client/config.yaml` | Proxmox tunnel YAML | `root:tunnel-client` | `0640` |
| `/etc/tunnel-client/openai-runtime.key` | Restricted OpenAI key | `root:tunnel-client` | `0640` |
| `/etc/tunnel-client/mcp-authorization` | Proxmox MCP `Bearer` Authorization value | `root:tunnel-client` | `0640` |
| `/etc/tunnel-client-unraid/` | Unraid tunnel directory | `root:tunnel-client` | `0750` |
| `/etc/tunnel-client-unraid/config.yaml` | Unraid tunnel YAML | `root:tunnel-client` | `0640` |
| `/etc/tunnel-client-unraid/openai-runtime.key` | Restricted OpenAI key | `root:tunnel-client` | `0640` |
| `/etc/tunnel-client-unraid/mcp-authorization` | Unraid MCP `Bearer` Authorization value | `root:tunnel-client` | `0640` |
| `/etc/tunnel-client-komodo/` | Komodo tunnel directory | `root:tunnel-client` | `0750` |
| `/etc/tunnel-client-komodo/config.yaml` | Komodo tunnel YAML | `root:tunnel-client` | `0640` |
| `/etc/tunnel-client-komodo/openai-runtime.key` | Restricted OpenAI key (`homelab-komodo-tunnel`) | `root:tunnel-client` | `0640` |

Komodo's key file and YAML permissions were **verified directly with `stat`**, and its key file was verified nonempty without displaying the value. **There is intentionally no `/etc/tunnel-client-komodo/mcp-authorization` file**: `MCP_AUTH_ENABLED=false` on the local Komodo MCP, unlike the bearer-protected Proxmox and Unraid MCP servers. The ChatGPT plugin's *No authentication* setting describes the lack of separate ChatGPT OAuth; it does not impose local MCP authentication. Credentials are not embedded in unit files or committed to Git.

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


### Komodo Core HTTPS API Access

The Komodo MCP container reaches existing Komodo Core at `https://komodo.giohosted.com`, resolving to `192.168.30.11` from the observer during setup. A local request completed with HTTP 200 and successful TLS verification. This is separate from the OpenAI control-plane outbound connection. The target server remains `docker-prod-01` on the Services VLAN; no direct Docker socket or privileged Docker API listener was added on the observer.

### OpenAI Connectivity
All three tunnel clients initiate outbound HTTPS traffic to OpenAI; none accepts inbound internet connections.

| Traffic | Purpose |
|---|---|
| Observer → `api.openai.com:443` | Outbound Secure MCP Tunnel control plane |
| Tunnel client → `127.0.0.1:8080` | Local Proxmox MCP requests |
| Tunnel client → `127.0.0.1:8082` | Local Unraid MCP requests |
| Tunnel client → `127.0.0.1:8084` | Local Komodo MCP requests |
| Proxmox MCP → `192.168.10.11:8006` | Proxmox HTTPS API via targeted UniFi firewall rule |
| Unraid MCP → `192.168.30.16:443` | Verified TLS to Unraid GraphQL API |
| Komodo MCP → `komodo.giohosted.com:443` (`192.168.30.11` at verification) | Komodo Core HTTPS API |

No inbound port forward, public MCP reverse proxy, or broad Services-to-Management firewall exception was introduced for these integrations.

## ChatGPT Integration
### Plugins

| Plugin | Tunnel | ChatGPT tool discovery | MCP authentication to local target | Live verification |
|---|---|---:|---|---|
| `Homelab Proxmox` | `homelab-proxmox` | **84 Read / 0 Write** | Tunnel injects protected MCP bearer header | `proxmox_get_cluster_status` succeeded |
| `Homelab Unraid` | `homelab-unraid` | **26 Read / 0 Write** | Tunnel injects protected MCP bearer header | `unraid_info` succeeded |
| `Homelab Komodo` | `homelab-komodo` | **40 Read / 0 Write** | **No local MCP bearer**; upstream anonymous HTTP MCP read scope plus dedicated non-admin Komodo API key | `komodo_health_check` and `komodo_container_list` succeeded |
| **Total** | **3 outbound tunnels** | **150 Read / 0 Write** | Per-integration controls | **All three integrations end-to-end verified** |

All three private ChatGPT plugins use **Connection: Tunnel** and **ChatGPT authentication option: No authentication** (no separate end-user OAuth flow). For Proxmox and Unraid, their MCP servers nevertheless demand their own bearer tokens. For Komodo, the local MCP is deliberately unauthenticated (`MCP_AUTH_ENABLED=false`); it remains loopback-bound and is restricted by upstream anonymous-read scope and the dedicated non-admin Komodo user. These controls are **not equivalent**: local processes may be able to query the Komodo MCP without a separate header.

### Usage

ChatGPT can select the relevant Homelab Proxmox, Homelab Unraid, or Homelab Komodo plugin automatically when live infrastructure context is useful. The corresponding plugins may also be invoked explicitly.

Representative read-only uses:

- Proxmox: inspect quorum and cluster status, guest and node health, storage capacity, backup job coverage, snapshots and task logs.
- Unraid: inspect array/parity/disk state, storage, shares, server status, metrics, container inventory and logs permitted by the 26-tool allowlist.
- Komodo: inspect `docker-prod-01` container inventory and states, search/read logs, read Komodo stacks, server statistics, and non-destructive resource metadata within the exposed 40-tool scope.

**Security:** A Read label means the tool is classified as non-mutating; it is **not a promise of non-sensitive output**. Container logs, node reports, resources, procedure scripts, webhook configuration, API-key metadata and host or guest configurations may reveal operational or private information. Review data before sending it to additional services. No write/management tool is currently exposed through these three plugins.

Documentation-first troubleshooting remains mandatory: consult the Markdown documentation in Google Drive and corroborate current state through the relevant live MCP integration.

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

### Proxmox MCP Audit — November 1, 2026

| Verification | Result |
|---|---|
| Container binary version | `0.5.1` verified with `sudo docker exec proxmox-mcp /proxmox-mcp version` |
| Live ChatGPT tool discovery | **84 Read**, **0 Write** |
| Configured toolsets | `cluster,nodes,vms,containers,agent,snapshots,storage,backup,tasks` (nine) |
| Excluded upstream toolsets | Ten: `access`, `firewall`, `ha`, `sdn`, `ceph`, `pools`, `replication`, `notifications`, `hardware`, `raw` |
| Write, delete, execution gates | All disabled in documented config |
| Reference version | [upstream v0.5.1 tools](https://github.com/mattoddie/proxmox-mcp/blob/v0.5.1/docs/tools.md), not future main branch |

The full 84-name inventory is recorded under **Proxmox MCP** above; tool discovery does not prove all 84 operations are authorized against Proxmox.

### Unraid MCP Verification — October 30, 2026 (CDT)

| Test | Result |
|---|---|
| Correct Rust image and binary | `ghcr.io/dinglebear-ai/unraid-rmcp:latest`, `unraid-rmcp 0.6.3` |
| Docker Compose configuration | Validated |
| Unraid container | Healthy; `127.0.0.1:8082->40010/tcp` |
| CA bundle initialization | `trusting extra CA certificates from bundle count=1` |
| Local bearer-authenticated `unraid_info` | Passed; Unraid OS 7.3.2 |
| Local bearer-authenticated `unraid_array` | Passed; surfaced disabled first parity disk on October 30 (historical observation; storage incident resolved October 31) |
| Exact local MCP tool inventory | 26 expected, 26 exposed, none missing/unexpected |
| Unraid tunnel `doctor` | `RESULT ok` |
| Unraid tunnel `/readyz` | Returned `ready` |
| `tunnel-client-unraid.service` | Active and enabled at boot |
| ChatGPT plugin discovery | Connected; 26 read tools, zero write tools |
| Live `unraid_info` from ChatGPT | Passed (hostname `nas-prod-01`, Unraid 7.3.2) |

A healthy container, a ready tunnel, and connected-plugin discovery each check different layers. End-to-end verification was achieved through the live ChatGPT invocation. Full per-tool testing, automatic startup through an actual observer reboot, and recovery/restore testing remain outstanding.


### Komodo MCP Verification — November 1, 2026 (CST)

| Verification | Result |
|---|---|
| Container and MCP readiness | Healthy `komodo-mcp`; `/ready` returned `ready:true` |
| MCP and Komodo versions | `1.5.0` / Core `2.3.4` |
| Local MCP calls | `komodo_health_check`, `komodo_server_list`, `komodo_container_list` succeeded |
| Server inventory | `docker-prod-01` state `Ok` |
| Docker inventory | 26 containers returned |
| Resource inventories | 9 stacks; 3 procedures; 1 builder; none of other queried types |
| Tool audit | **40 Read**, zero Write; eight explicit exclusions |
| Tunnel ID | `tunnel_6ae6e984758881919f5c308207bab3d3` |
| OpenAI runtime key | Restricted `homelab-komodo-tunnel`, saved in Bitwarden; file verified nonempty |
| Local tunnel diagnostic | `doctor --config /etc/tunnel-client-komodo/config.yaml` -> `RESULT ok` |
| Systemd service | `tunnel-client-komodo.service` active and enabled |
| Tunnel readiness | `127.0.0.1:8085/readyz` HTTP 200 `ready` |
| ChatGPT plugin discovery | `Homelab Komodo`: 40 Read tools |
| Live ChatGPT tool calls | `komodo_health_check` returned configured/healthy; `komodo_container_list` returned 26 containers |

The `doctor` result included an HTTP 400 reachability response from `/mcp`, accepted by the diagnostic as a valid reachability result; it was **not** used as proof of authenticated JSON-RPC. Successful MCP tool calls provide that proof. Full per-tool testing, cold-reboot recovery and isolated restore remain unverified.

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
The VM's configuration and credentials are spread across these locations:

- `/opt/stacks/proxmox-mcp/compose.yaml`, `/opt/stacks/proxmox-mcp/.env` — Proxmox Compose and protected credentials.
- `/opt/stacks/unraid-mcp/compose.yaml`, `/opt/stacks/unraid-mcp/.env` — Unraid Compose and protected credentials; repository commit status remains unverified.
- `/etc/unraid-mcp/unraid.crt` — Trusted Unraid certificate bundle.
- `/opt/stacks/komodo-mcp/compose.yaml`, `/opt/stacks/komodo-mcp/.env` — Komodo MCP Compose and dedicated Komodo API key/secret. **A repository commit or independent file backup for the new stack has not been verified.**
- `/etc/tunnel-client/` — Proxmox tunnel YAML, restricted runtime key, protected MCP bearer Authorization header.
- `/etc/tunnel-client-unraid/` — Unraid tunnel YAML, restricted runtime key, protected MCP bearer Authorization header.
- `/etc/tunnel-client-komodo/` — Komodo tunnel YAML and restricted OpenAI runtime key (no local MCP bearer header).
- `/etc/systemd/system/tunnel-client.service`, `tunnel-client-unraid.service`, `tunnel-client-komodo.service` — Three independent native systemd units.
- `/opt/tunnel-client/` — Shared tunnel-client binary, bundled cloudflared and release support files.

Existing Docker-host rsync backup jobs must **not** be assumed to protect observer VM host configuration or credentials. The historical October 30 successful VM 107 PBS backup task does not establish that the **November 1 Komodo additions** are in an intact, restorable backup. A fresh VM 107 snapshot, independent integrity check, secure credential recovery, and safe isolated restore still need verification.

Keep credentials in approved secure storage (Bitwarden where verified), not in Git or Obsidian. The separately retained Unraid MCP bearer token remains an identified recovery-check item. See `runbooks/backup-schedule.md` for PBS and `runbooks/ai-observer-operations.md` for recovery.

## Operations and Maintenance
### Management Approach

The observer is managed manually through SSH and the CLI. No Dockman instance is installed on `observer-prod-01`; the Dockman instance on `docker-prod-01` does not manage this observer's Docker engine. Docker Compose changes and service restarts require the operator's explicit authorization.

### MCP Stacks

| Integration | Compose directory | Expected container | Update approach |
|---|---|---|---|
| Proxmox | `/opt/stacks/proxmox-mcp/` | `proxmox-mcp` | Operator-controlled pull and recreate, then verify version and all 84 tools |
| Unraid | `/opt/stacks/unraid-mcp/` | `unraid-mcp` | Operator-controlled pull and recreate, then verify TLS and all 26 tools |
| Komodo | `/opt/stacks/komodo-mcp/` | `komodo-mcp` | Operator-controlled pull and recreate, then verify upstream permissions and all 40 tools |

All three images currently use mutable `latest` tags, which **do not automatically pull new releases**. Keep the existing local image available for rollback when feasible. The Komodo MCP talks to existing Komodo Core over HTTPS rather than mounting a Docker socket or managing the host directly.

### Tunnel Client

The tunnel-client is managed by **three independent systemd units**: `tunnel-client.service`, `tunnel-client-unraid.service`, and `tunnel-client-komodo.service`. They share one installed `/opt/tunnel-client/tunnel-client` binary, so an approved binary upgrade affects all three after restart. Review all three on upgrade and ensure each uses its respective restricted key and config.

### Basic Read-Only Health Checks

```bash
# Run on observer-prod-01
for dir in proxmox-mcp unraid-mcp komodo-mcp; do
  (cd "/opt/stacks/$dir" && sudo docker compose ps)
done

curl -fsS --max-time 5 http://127.0.0.1:8080/health
curl -fsS --max-time 5 http://127.0.0.1:8084/ready

for unit in tunnel-client.service tunnel-client-unraid.service tunnel-client-komodo.service; do
  systemctl is-active "$unit"
  systemctl is-enabled "$unit"
done

for port in 8081 8083 8085; do
  curl -fsS --max-time 10 "http://127.0.0.1:$port/readyz"
done
```

Then perform a live read-only call to each plugin. Passing container and tunnel health alone does **not** verify upstream API credentials, permissions, or end-to-end ChatGPT access. Full troubleshooting, credential-rotation, and recovery steps are in `runbooks/ai-observer-operations.md`.

## Security Boundaries
The implemented defense-in-depth model has **separate upstream identities, separate MCP surfaces, and separate outbound tunnels**:

1. **Upstream least privilege:** Proxmox's `observer@pve` account uses `PVEAuditor`; Unraid uses 14 `READ_ANY` families and no write grants; Komodo uses dedicated non-admin `ai-observer-komodo` with Read and scoped Logs permissions, excluding Inspect/Terminal/Attach/Processes and creation.
2. **MCP tools:** Proxmox exposes 84 read tools from nine enabled toolsets, with write/delete/exec switches false; Unraid exposes an exact 26-tool read allowlist; Komodo exposes 40 read tools with eight sensitive tools explicitly excluded and upstream anonymous-HTTP read scope. **Zero write tools** exposed across all three plugins at verification.
3. **Network:** All MCP endpoints are bound to observer loopback only, with no public MCP endpoint. The only targeted observer-to-Management firewall exception remains `192.168.30.17 → 192.168.10.11 TCP 8006` for Proxmox.
4. **Tunnel:** Three independent outbound OpenAI tunnels, each with a restricted Tunnels Read+Use runtime API key. All run as non-root `tunnel-client` with no Docker group membership.
5. **MCP authentication difference:** Proxmox and Unraid use local MCP bearer authentication supplied by protected tunnel headers. Komodo intentionally runs `MCP_AUTH_ENABLED=false`: **local clients on observer loopback are not individually authenticated**. Restrict observer access and preserve the dedicated upstream read-only account and tool exclusions.
6. **Secrets:** MCP `.env`, runtime key, and authorization files are excluded from Git. The third tunnel has **no** local MCP bearer-header file by design. No plaintext secrets are reproduced here.
7. **Information exposure:** Even non-mutating tools may return sensitive logs, host/network configuration, scripts, API-key metadata, or potentially secrets. Komodo's secret-scrubbing mechanism is best-effort; full Docker Inspect and Compose/TOML exports remain excluded. For retained detailed `*_info` tools, review actual output before sharing.
8. **Operational separation:** No Docker socket is mounted into any observer MCP container; the observer's Docker daemon is not remotely administered by Dockman or the Komodo MCP.

Any change to upstream privileges, toolsets, local authentication, excluded tools, secret storage, or network exposure requires explicit review and post-change tool-inventory and end-to-end verification. No write automation is implemented.

## Planned Extensions
| Integration / capability | Status |
|---|---|
| Proxmox MCP 0.5.1 / 84 read tools | **Implemented and verified** |
| Unraid MCP 0.6.3 / 26 read tools | **Implemented and verified** |
| Komodo MCP 1.5.0 / 40 read tools / eight exclusions | **Implemented and verified** on `docker-prod-01` |
| Three private ChatGPT plugins / three outbound secure tunnels | **Implemented and end-to-end verified** |
| Extend Komodo monitoring to further Docker hosts | **Not implemented/verified**; separate scope approval needed |
| Codex usage through these tunnels | Not yet verified |
| Proxmox, Unraid, or Komodo write operations | Deferred, not authorized |
| Automated maintenance and remediation | Deferred |
| Fresh observer backup/credential recovery and isolated restore test | **Pending verification** |

No elevated permissions should be introduced for convenience. Revisit information disclosure whenever enabling additional ostensibly read-only tools.

## Related Documentation

- `architecture/network/ip-addresses.md` — Observer IP allocation.
    
- `architecture/network/firewall-rules.md` — Targeted Proxmox API firewall exception.
    
- `architecture/compute/pve-prod-01.md` — Observer VM guest inventory.
    
- `architecture/decisions-log.md` — AI Observer security and architecture decisions.
    
- `runbooks/ai-observer-operations.md` — Operations, maintenance, troubleshooting, and recovery procedures.

- `architecture/storage/nas-prod-01.md` — Unraid NAS configuration, current array inventory, and resolved October 2026 parity/Disk 3 connection incident.
    
- `runbooks/git-workflow.md` — Version control and credentials workflow.
    
- `runbooks/updating-containers.md` — Docker Compose update practices.
    
- `runbooks/backup-schedule.md` — Backup coverage and verification.
    
- `services/infra/dockman.md` — Local Docker management separation.
    

**Source repositories:**

- `homelab-docs` — Obsidian infrastructure documentation.
    
- `homelab-infra` — Docker Compose infrastructure configuration.
    

---

## Notes
- AI Observer runs independently on VM 107 and is **not** a general-purpose production Docker host.
- OpenAI Secure MCP Tunnel does not replace existing homelab Cloudflare Tunnel or Traefik.
- The three installed integrations are intentionally **read-only**: Proxmox **84** tools, Unraid **26** tools and Komodo **40** tools, all discovered in ChatGPT. Tool discovery is not proof of every individual upstream query's success.
- Proxmox compiled binary `0.5.1` was verified November 1, 2026, and upstream tool classifications were checked against the **v0.5.1** reference, not just current `main`.
- Komodo intentionally has an unauthenticated **local** MCP endpoint (read-only scope, localhost only); its dedicated non-admin upstream API credentials and eight tool exclusions are critical security controls. No credential output should be assumed safe solely because secret scrubbing is enabled.
- The October 30 observer VM backup task succeeded historically. A backup containing the **November 1 changes**, integrity validation, credential-recovery checks and isolated restoration test remain unverified.
- Update authoritative docs after verified implementations; preserve explicit distinction between implemented, planned and unresolved work.

