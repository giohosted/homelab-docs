# Beszel

**Host (Hub):** pi-prod-01 (192.168.10.20)  
**Status:** Running — dashboard verified 2026-11-14  
**Last Updated:** 2026-11-14

---

## Overview

Beszel is a lightweight server monitoring tool. The hub runs as a systemd binary service on pi-prod-01 and collects metrics from agents installed on monitored hosts.

The monitoring architecture consists of:

- A central Beszel hub on pi-prod-01.
- Native systemd agents on supported infrastructure hosts.
- Docker Compose agents on application hosts.
- An Unraid Docker agent on nas-prod-01.

The hub provides centralized access to monitoring information, including CPU, memory, disk, and other available system metrics.

**Verified 2026-11-14:** The Beszel dashboard was accessible through its existing direct-IP address and displayed current monitoring metrics.

---

## Hub

- **Install method:** Binary (systemd service)
- **Service name:** `beszel-hub`
- **Binary location:** `/opt/beszel/beszel`
- **Host:** pi-prod-01 (192.168.10.20)
- **Web UI:** `http://192.168.10.20:8090`
- **Access:** From VLAN 20 (Trusted) — port 8090 is included in the `MGMT_ADMIN_PORTS` firewall group

### Useful Commands

```bash
sudo systemctl status beszel-hub
sudo systemctl restart beszel-hub
sudo systemctl stop beszel-hub
```

Use restart and stop operations only when required for approved maintenance or troubleshooting.

### Web Access

The established management URL is:

`http://192.168.10.20:8090`

The dashboard was verified working through this address on November 14, 2026.

An unused Traefik router previously existed for `beszel.giohosted.com`. That route has been disabled.

The direct-IP URL remains the documented management endpoint.

See the Traefik section below.

---

## Agents

The following agent inventory reflects the documented deployment. Not every agent was independently revalidated during the November 14 maintenance session.

| Host | IP | Install Method | Agent Port |
|---|---|---|---|
| pi-prod-01 | 192.168.10.20 | Binary (systemd) | 45876 |
| pve-prod-01 | 192.168.10.11 | Binary (systemd) | 45876 |
| pve-prod-02 | 192.168.10.12 | Binary (systemd) | 45876 |
| docker-prod-01 | 192.168.30.11 | Docker Compose | 45876 |
| auth-prod-01 | 192.168.30.13 | Docker Compose | 45876 |
| immich-prod-01 | 192.168.30.14 | Docker Compose | 45876 |
| nas-prod-01 | 192.168.10.10 | Unraid Docker (CA) | 45876 |

Docker Compose agents are located at:

`/opt/stacks/beszel-agent/`

Agent configuration uses Git-excluded `.env` files where required.

### Docker Compose Agent

The shared Compose configuration is maintained in the private `homelab-infra` repository:

`/opt/stacks/beszel-agent/compose.yaml`

The deployed agent image observed on immich-prod-01 was:

`henrygd/beszel-agent`

The agent uses a persistent bind mount:

```yaml
volumes:
  - ./beszel_agent_data:/var/lib/beszel-agent
```

The directory contains host-specific runtime state, including the agent's fingerprint.

**Important:** The Compose definition is shared through Git, but the persistent agent runtime state must remain local to each host.

### Host-Specific Runtime State

The agent runtime directory is:

`/opt/stacks/beszel-agent/beszel_agent_data/`

The fingerprint is stored at:

`/opt/stacks/beszel-agent/beszel_agent_data/fingerprint`

Each host must preserve its own runtime state.

Do not replace one host's fingerprint with a copy from another host.

Do not manually modify or delete the fingerprint as part of routine container updates.

### Git Tracking

**Corrected 2026-11-14**

The shared `homelab-infra` repository previously tracked:

`beszel-agent/beszel_agent_data/fingerprint`

This was inappropriate because the fingerprint represents host-specific runtime state rather than shared deployment configuration.

The repository's root `.gitignore` now contains:

```gitignore
# Beszel agent runtime state (host-specific)
/beszel-agent/beszel_agent_data/
```

The fingerprint was removed from Git tracking while retaining the local file on the repository maintenance host.

**Git commit:** `10848f0` — `Stop tracking host-specific Beszel agent state`

The Compose file remains tracked:

`beszel-agent/compose.yaml`

Verify the ignore rule on a Docker host:

```bash
cd /opt/stacks

git check-ignore -v \
  beszel-agent/beszel_agent_data/fingerprint
```

Expected behavior: Git identifies the repository's ignore rule for the runtime directory.

**Important:** Removing a previously tracked file from Git can delete the corresponding file during synchronization of older checkouts. A backup and bind-mount impact assessment may be required before applying such changes.

Removing a file from current Git tracking also does not erase historical Git objects.

---

## Firewall Rule

The existing UniFi firewall documentation records a targeted rule for Beszel monitoring traffic between VLAN 30 and the Management network.

The documented rule is:

- **Rule name:** Allow Services to Beszel
- **Source:** VLAN 30 (Services)
- **Destination:** 192.168.10.20 (pi-prod-01)
- **Destination port:** 45876
- **Action:** ALLOW
- **Evaluation:** Before the general Services-to-Management block

The November 14 firewall review confirmed that this rule exists and is enabled.

**Protocol scope:** The original Beszel documentation described TCP port 45876. The corresponding firewall documentation states that the exact TCP/UDP restriction was not independently confirmed. Do not assume that the live rule is TCP-only without verification.

VLAN 10 hosts can communicate within the same subnet without traversing the inter-VLAN firewall.

### Monitoring Traffic Direction

There is an unresolved distinction between the documented firewall rule and the observed Beszel agent connection.

During the November 14 recovery of the Immich agent, its logs showed an incoming SSH connection from:

`192.168.10.20`

The agent reported:

```text
INFO SSH connected addr=192.168.10.20:35346
INFO SSH connection established
```

This indicates the hub successfully established an SSH connection to the agent.

The existing UniFi exception, however, is documented in the opposite direction: Services VLAN → hub IP, destination port 45876.

**Status:** The direction and necessity of the existing firewall exception have not been fully reconciled.

Do not change or remove the rule based solely on these observations. Verify live traffic requirements and the effective UniFi policies before making firewall changes.

See `network/firewall-rules.md` for the authoritative firewall configuration and security boundaries.

### Beszel UI Access

The hub's management interface uses:

`http://192.168.10.20:8090`

Port 8090 is included in `MGMT_ADMIN_PORTS` for authorized Trusted-to-Management access.

During the Traefik review, a direct request from docker-prod-01 (`192.168.30.11`) to the hub on port 8090 timed out.

This is consistent with restricted inter-VLAN access, but the exact cause was not independently established.

The existing direct-IP dashboard remained accessible from the operator's authorized client.

No firewall changes were made to enable Traefik access.

---

## Traefik Integration

### Retired Reverse-Proxy Route — 2026-11-14

An existing dynamic Traefik configuration was found at:

`/opt/appdata/traefik/config/beszel.yml`

It defined:

| Setting | Previous configuration |
|---|---|
| Hostname | `beszel.giohosted.com` |
| Backend | `http://192.168.10.20:8090` |
| EntryPoint | `websecure` |
| Middleware | `local-only@docker` |
| TLS resolver | `cloudflare` |

During the Traefik v3.7.19 upgrade review:

1. The hostname route produced a gateway timeout from an authorized client.
2. Direct connectivity from docker-prod-01 to `192.168.10.20:8090` timed out.
3. The operator confirmed that Beszel was normally accessed using the direct-IP bookmark.
4. The existing IP-based dashboard was verified operational.

There was no confirmed requirement for hostname-based access.

Rather than create additional inter-VLAN access, the unused Traefik configuration was moved outside the watched configuration directory.

**Previous path:**

`/opt/appdata/traefik/config/beszel.yml`

**Disabled configuration:**

`/opt/appdata/traefik/beszel.yml.disabled`

Traefik's file provider has watching enabled, so the router was removed without restarting the proxy.

### Verification

After disabling the route:

| Hostname | HTTP Response | Result |
|---|---|---|
| `auth.giohosted.com` | 302 | Authentik routing unaffected |
| `photos.giohosted.com` | 200 | Immich routing unaffected |
| `beszel.giohosted.com` | 404 | Beszel router removed |

All three HTTPS requests passed certificate verification.

**Current state:** Beszel is accessed directly through `http://192.168.10.20:8090`, not through Traefik.

The disabled router file is retained outside the watched configuration directory.

See `services/infra/traefik.md`.

---

## Recovery — immich-prod-01 Beszel Agent

**Incident date:** 2026-11-14  
**Status:** Resolved and verified

### Background

During Git synchronization, the Immich host was fast-forwarded to the commit that stopped tracking Beszel's fingerprint.

The older checkout still tracked the fingerprint.

Before synchronization, the original fingerprint was copied to a separate backup location and verified using matching SHA-256 hashes.

### What Happened

During the Git fast-forward:

1. Git removed the previously tracked fingerprint.
2. The now-empty `beszel_agent_data` directory was also removed.
3. The original fingerprint was restored from the verified backup after recreating the directory.
4. The existing Docker container continued running but still referenced the original bind-mounted directory.

The container was therefore running while its runtime directory no longer matched the active host path.

### Diagnosis

The host directory was:

`/opt/stacks/beszel-agent/beszel_agent_data`

The container directory was:

`/var/lib/beszel-agent`

A read-only inspection compared their device and inode identities.

Example:

```bash
cd /opt/stacks

pid=$(sudo docker inspect -f '{{.State.Pid}}' beszel-agent)

sudo stat -Lc '%d:%i %n' \
  beszel-agent/beszel_agent_data \
  "/proc/$pid/root/var/lib/beszel-agent"
```

Before correction, the directory identities differed.

The container also could not see the fingerprint in the recreated host directory.

This confirmed that the running container's bind mount was stale.

### Resolution

The original host-specific fingerprint was preserved and restored.

Only the Beszel agent container was recreated:

```bash
cd /opt/stacks/beszel-agent

sudo docker compose up -d \
  --no-deps \
  --pull never \
  --force-recreate \
  beszel-agent
```

This reestablished the bind mount using the current host directory.

The existing locally available image was reused.

Immich application containers were not restarted.

### Verification

After recreation, the host and container directory identities matched.

The container-visible fingerprint was confirmed identical to the restored original.

The verification used:

```bash
cd /opt/stacks

pid=$(sudo docker inspect -f '{{.State.Pid}}' beszel-agent)

sudo stat -Lc '%d:%i %n' \
  beszel-agent/beszel_agent_data \
  "/proc/$pid/root/var/lib/beszel-agent"

sudo cmp -s \
  beszel-agent/beszel_agent_data/fingerprint \
  "/proc/$pid/root/var/lib/beszel-agent/fingerprint" \
  && echo "PASS: Container sees the original fingerprint"
```

The successful output reported:

`PASS: Container sees the original fingerprint`

Additional verification showed:

- Container status: `running`
- Restart count: `0`
- Beszel SSH connection established from the hub
- Beszel dashboard: Immich host online with current metrics

**Final result:** Agent recovery was verified and monitoring resumed normally.

### WebSocket Warnings

The agent logs also contained timeouts when attempting a WebSocket connection to:

`192.168.10.20:8090`

An SSH connection from the hub was subsequently established.

The monitoring dashboard showed current metrics, indicating that the agent was functioning through the established connection path.

The WebSocket timeouts were not separately resolved during this maintenance session.

They should only be revisited if monitoring becomes unavailable or the warnings are associated with a reproducible operational problem.

---

## Operational Verification

### Docker Agent Status

On a Docker host running a Beszel agent:

```bash
sudo docker ps -a \
  --filter name=beszel-agent \
  --format 'table {{.Names}}\t{{.Status}}\t{{.Image}}'
```

Inspect container state:

```bash
sudo docker inspect \
  --format 'Status={{.State.Status}} Restarting={{.State.Restarting}} RestartCount={{.RestartCount}}' \
  beszel-agent
```

Inspect recent logs:

```bash
sudo docker logs --since 5m --tail 40 beszel-agent
```

### Effective Runtime Mount

When investigating missing runtime state or a stale bind mount:

```bash
cd /opt/stacks

pid=$(sudo docker inspect -f '{{.State.Pid}}' beszel-agent)

sudo stat -Lc '%d:%i %n' \
  beszel-agent/beszel_agent_data \
  "/proc/$pid/root/var/lib/beszel-agent"
```

Matching device and inode identities indicate that the inspected container path refers to the same filesystem directory as the host path.

Confirm the container can access its required runtime files.

### Dashboard Verification

From an authorized client, open:

`http://192.168.10.20:8090`

Confirm:

- The monitored host appears online.
- Metrics have recent timestamps.
- CPU, memory, and disk metrics update normally.
- There are no unexpected disconnects.

A container reporting `running` is not sufficient to establish successful monitoring.

---

## Notes

- nas-prod-01 temperature sensors `nct6798_auxtin2` and `nct6798_auxtin4` report 127°C — this is a sentinel value for unconnected sensor inputs, not a real reading. Ignore these.
- Hub port 8090 is included in `MGMT_ADMIN_PORTS` for Trusted-to-Management access.
- The established Beszel dashboard URL is `http://192.168.10.20:8090`.
- The unused `beszel.giohosted.com` Traefik route was disabled on 2026-11-14.
- Beszel Compose configuration is shared through `homelab-infra`; agent fingerprints and runtime state are host-specific.
- The `beszel_agent_data/` directory is excluded from Git as of commit `10848f0`.
- Do not remove a bind-mounted runtime directory during Git synchronization without assessing the effect on running containers.
- The immich-prod-01 Beszel agent was recreated and verified healthy after the November 14 bind-mount incident.
- The documented UniFi Beszel firewall rule direction and protocol scope should be reviewed before changing the policy.
- The documented agent inventory may need updating as additional Homelab v3 nodes are added or existing hosts are replaced.

---

## Related Documentation

- `services/infra/traefik.md` — Reverse proxy configuration and retired Beszel route.
- `runbooks/git-workflow.md` — Shared Git repository and host-specific runtime data.
- `runbooks/updating-containers.md` — Docker Compose updates and recovery considerations.
- `network/firewall-rules.md` — Inter-VLAN firewall policies and Beszel monitoring exceptions.
- `runbooks/backup-schedule.md` — Infrastructure backup and recovery procedures.