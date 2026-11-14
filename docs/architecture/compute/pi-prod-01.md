
# pi-prod-01 — Raspberry Pi 4B

**Role:** Lightweight always-on infrastructure monitoring — Uptime Kuma; historical Beszel hub  
**Status:** Active monitoring host; individual service states not fully reverified  
**Last Updated:** 2026-11-13

---

## Hardware

| Component | Detail |
|-----------|--------|
| **Model** | Raspberry Pi 4B |
| **RAM** | 4GB |
| **Boot Storage** | SD card |
| **Networking** | 1x GbE RJ45 built-in |
| **OS** | Raspberry Pi OS (Debian GNU/Linux 12 Bookworm; last documented version) |

---

## Network

| Field | Value |
|---|---|
| **Hostname** | pi-prod-01 |
| **IP** | 192.168.10.20 |
| **VLAN** | 10 — Management |
| **Gateway** | 192.168.10.1 |
| **DNS** | 192.168.30.10 (AdGuard primary), 9.9.9.9 (fallback), per last documented configuration |
| **Switch port profile** | Mgmt-Only (access port, VLAN 10) |
| **Network manager** | NetworkManager (not dhcpcd) |

### Static IP Configuration

The Pi was configured with a static IP using NetworkManager and `nmcli`. Raspberry Pi OS Bookworm uses NetworkManager; the existing `/etc/dhcpcd.conf` file is not authoritative for this deployment.

The previously documented connection profile is `Wired connection 1`.

Before making network changes, verify the active connection, assigned address, gateway and DNS configuration:

```bash
nmcli connection show --active
nmcli device status
nmcli device show
nmcli connection show "Wired connection 1"
```

**Operational note:** NetworkManager changes can interrupt SSH connectivity. The actual active profile name, configured DNS entries and firewall reachability should be checked before applying any modifications.

The Pi remains on the Management VLAN because it hosts infrastructure monitoring services accessed from the administrative network.

The former QDevice requirement is no longer a reason to keep the Pi on VLAN 10. Any future network relocation would require a separate review of monitoring connectivity, access rules and management dependencies.

---

## Roles

### 1. Uptime Kuma — Primary Monitoring Role

**Status:** Operational according to the latest infrastructure inventory and operator confirmation. Container health and current application version were not independently rechecked during this documentation update.

Uptime Kuma is the Pi's always-on availability monitoring service. It monitors infrastructure hosts and applications independently of the Proxmox cluster.

**Last documented deployment:**

| Setting | Value |
|---|---|
| Installation method | Docker Compose |
| Image | `louislam/uptime-kuma:2` |
| Container | `uptime-kuma` |
| Compose directory | `/opt/stacks/uptime-kuma/` |
| Data directory | `/opt/appdata/uptime-kuma/` |
| Web UI | `http://192.168.10.20:3001` |
| Network mode | Host |
| Container restart policy | `unless-stopped` |
| Internal DNS | 192.168.30.10 and 192.168.30.15 |

The service was documented as monitoring Proxmox nodes, Docker hosts, DNS, NAS services, identity services, media applications and other homelab workloads.

The latest infrastructure inventory specifically confirms operator-observed Uptime Kuma monitoring for `pve-prod-02` and `pve-prod-03`.

The detailed monitoring inventory in `services/monitoring/uptime-kuma.md` dates from March 2026 and contains historical endpoints. It should not be treated as a verified current monitor export without checking the live Uptime Kuma configuration.

**Read-only inspection:**

```bash
sudo docker ps --filter name=uptime-kuma
sudo docker logs --tail 50 uptime-kuma
```

**Service documentation:** `services/monitoring/uptime-kuma.md`

### 2. Beszel — Historical Monitoring Hub

**Status:** Previously implemented and documented as running. Current hub process, version and monitoring coverage have not been independently reverified.

Beszel was deployed as the primary system-metrics monitoring platform. The hub historically ran directly on the Raspberry Pi as a systemd-managed binary, with agents installed on infrastructure hosts.

**Last documented deployment:**

| Setting | Value |
|---|---|
| Role | Beszel monitoring hub |
| Installation method | Native binary |
| Service | `beszel-hub` |
| Binary | `/opt/beszel/beszel` |
| Web UI | `http://192.168.10.20:8090` |
| Pi agent | Native systemd service |
| Agent port | TCP 45876 |

The March 2026 documentation recorded agents on:

- pi-prod-01
- pve-prod-01
- pve-prod-02
- docker-prod-01
- auth-prod-01
- immich-prod-01
- nas-prod-01

The current deployment may include additional or changed hosts following the November 2026 Proxmox expansion and node replacement. No updated Beszel hub inventory was verified during this review.

**Read-only inspection:**

```bash
systemctl status beszel-hub --no-pager
systemctl status beszel-agent --no-pager
sudo ss -lntp | grep -E ':(8090|45876)\b'
```

An absent `beszel-agent` unit does not by itself establish that the agent is missing; verify the actual installation method if necessary.

**Network dependency requiring verification:**

Historical Beszel and firewall documentation describes a targeted inter-VLAN exception involving `pi-prod-01` and TCP 45876.

The exact direction of the required connection and the current UniFi firewall implementation have not been independently confirmed. Do not assume that the old description of agents initiating connections to the hub is accurate.

Verify the actual hub-to-agent connection behavior and live UniFi rule before modifying monitoring firewall policy.

**Service documentation:** `services/monitoring/beszel.md`

### 3. Proxmox QDevice — Retired Role

**Status:** Retired from the active Proxmox cluster in November 2026.

The Raspberry Pi originally acted as an external quorum tiebreaker for the two-node `homelab-v3` Proxmox cluster.

**Historical configuration:**

| Setting | Historical value |
|---|---|
| Role | Corosync QDevice / QNetd quorum tiebreaker |
| Pi IP | 192.168.10.20 |
| Cluster | homelab-v3 |
| Original PVE members | pve-prod-01 and pve-prod-02 |
| Historical setup command | `pvecm qdevice setup 192.168.10.20` |
| External quorum vote | 1 |
| Original expected quorum | 2 of 3 votes |

The Raspberry Pi historically provided a third vote so that the original two Proxmox nodes could establish majority quorum when needed.

**Removal:**

Before `pve-prod-03` joined the cluster in November 2026, the external QDevice was removed from the Proxmox cluster configuration.

The current cluster consists of:

| Node | IP | Votes |
|---|---|---|
| pve-prod-01 | 192.168.10.11 | 1 |
| pve-prod-02 | 192.168.10.12 | 1 |
| pve-prod-03 | 192.168.10.13 | 1 |

**Current quorum:** Two of three Proxmox node votes.

The live Homelab Proxmox integration verified three online voting nodes, a quorate cluster and a Corosync configuration with no external QDevice.

Consequently:

- The Pi no longer contributes a quorum vote.
- An outage of the Pi does not, by itself, affect Proxmox quorum.
- The cluster does not need the Pi to serve as a tiebreaker.
- The Pi's monitoring functions remain independent of quorum.
- Proxmox HA is still disabled; quorum availability does not provide automatic guest failover.

**Remaining local verification:** Removal from the active cluster has been confirmed, but the current state of any old `corosync-qnetd` packages, services, or configuration files on the Raspberry Pi itself has not been checked. Do not claim that the Pi-side software was uninstalled without verifying it.

For historical architecture rationale, see `architecture/decisions-log.md`.

---

## v2 → v3 Role Changes

| Role | v2 | v3 historical deployment | Current status |
|---|---|---|---|
| AdGuard Home primary | On Pi | Moved to dns-prod-01 | Runs on pve-prod-01 |
| AdGuard Home secondary | Not present | Added as dns-prod-02 | Runs on pve-prod-03 |
| Uptime Kuma | On Pi | Retained on Pi | Operational per latest inventory |
| Beszel hub | On Pi | Retained on Pi | Previously deployed; live state requires verification |
| Proxmox QDevice | Not present | Added for two-node cluster | Removed November 2026 |
| VLAN | 20 — Trusted | 10 — Management | VLAN 10, per latest documentation |
| IP | 192.168.20.8 | 192.168.10.20 | 192.168.10.20, per latest documentation |

### November 2026 Architecture Changes

The Proxmox cluster was expanded and its original quorum model replaced:

1. The external Raspberry Pi QDevice was removed.
2. `pve-prod-03` joined `homelab-v3` as a third voting node.
3. The original `pve-prod-02` was subsequently replaced with newer hardware.
4. The active cluster now consists of three Proxmox nodes with one vote each.
5. Secondary AdGuard (`dns-prod-02`, CT 102) resides on `pve-prod-03`.

The Raspberry Pi now serves a monitoring role rather than a cluster-consensus role.

No Raspberry Pi hardware replacement, OS upgrade, Beszel reconfiguration or QDevice package removal was verified as part of this documentation update.

---

## SSH Access

Normal access:

```bash
ssh gdelgado@192.168.10.20
```

Historical SSH configuration:

- Direct root SSH login was temporarily enabled for the original QDevice setup.
- Root SSH login was subsequently disabled.
- `PermitRootLogin no` was documented in `/etc/ssh/sshd_config`.

The current effective SSH daemon configuration and authentication methods were not reverified during this update.

**Read-only verification:**

```bash
sudo sshd -T | grep -E '^(permitrootlogin|passwordauthentication|pubkeyauthentication) '
```

The original Phase 5 task to establish key-based SSH authentication across all hosts should be checked against the current configuration before being marked complete or carried forward as outstanding.

---

## Operational Notes

### Availability and Dependencies

The Pi is no longer required for Proxmox quorum.

However, if Uptime Kuma and Beszel are still hosted there, a Pi outage would affect the monitoring visibility those services provide.

An outage of the monitoring host does not necessarily indicate an outage of the systems being monitored.

### Storage

The Pi uses an SD card for boot and operating-system storage.

This was considered acceptable for its lightweight original role. However, persistent application data and monitoring databases can create ongoing writes.

Monitor SD card health, filesystem errors and available storage. Consider a USB SSD migration if storage reliability, performance or endurance becomes a concern.

No SD card health check or storage migration was verified during this review.

### Network and Firewall

The Pi is documented on VLAN 10 (Management).

Administrative access and monitoring traffic may involve connections from other VLANs.

Relevant documentation includes:

- `architecture/network/firewall-rules.md`
- `architecture/network/vlan-design.md`
- `services/monitoring/uptime-kuma.md`
- `services/monitoring/beszel.md`

The historical Beszel TCP 45876 exception and Trusted-to-Management access to TCP 8090 require live confirmation before being considered authoritative.

### QDevice Software Cleanup

The cluster-level QDevice configuration is removed and no further quorum changes are required for this architecture.

The Pi-side QNetd service/package state is unknown.

Any eventual cleanup should first confirm the local service and package state, preserve relevant historical documentation, and avoid interfering with Uptime Kuma or Beszel.

### Verification Status — November 13, 2026

| Item | Status | Evidence |
|---|---|---|
| Three-node Proxmox quorum | Verified | Live Proxmox cluster status and Corosync configuration |
| QDevice absent from cluster | Verified | Three voting nodes; no QDevice in current cluster configuration |
| Pi network identity | Documented | Existing inventory and architecture records |
| Uptime Kuma monitoring role | Reported operational | Latest inventory and operator confirmation |
| Uptime Kuma container health | Not reverified | No direct Pi host inspection |
| Beszel hub service | Not reverified | March deployment documentation only |
| Beszel current agent inventory | Not reverified | Older agent list predates cluster changes |
| Pi-side QNetd software removal | Not reverified | No direct Pi package/service inspection |
| Current SSH hardening | Not reverified | Existing configuration documentation only |
| SD card health | Not reverified | No current filesystem or device diagnostics |

---

## Related Documentation

- `architecture/compute/pve-prod-01.md`
- `architecture/compute/pve-prod-02.md`
- `architecture/compute/pve-prod-03.md`
- `architecture/decisions-log.md`
- `inventory.md`
- `services/monitoring/uptime-kuma.md`
- `services/monitoring/beszel.md`
- `architecture/network/firewall-rules.md`
