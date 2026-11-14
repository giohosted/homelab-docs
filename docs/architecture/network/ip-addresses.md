# IP Address

**Last updated:** 2026-11-14

---

## VLAN 10 — Management (192.168.10.0/24)

Strictly for infrastructure control plane interfaces — devices you administer, not services you consume.

| Device | IP | Interface | Notes |
|---|---|---|---|
| UDM-SE | 192.168.10.1 | Built-in | Gateway — already configured, do not change |
| Core Switch (UniFi USW-Pro-Max-24) | 192.168.10.2 | Management | Switch management interface |
| UPS (if networked later) | 192.168.10.3 | — | Reserved — NUT handles shutdown via USB for now |
| nas-prod-01 — management interface | 192.168.10.10 | Onboard 2.5GbE | Unraid management UI. Docker PBS uses Host networking; possible TCP 8007 exposure on this interface has **not** been independently restricted/verified. NFS is not intentionally served here. |
| pve-prod-01 (Minisforum MS-A2) | 192.168.10.11 | 2.5GbE RJ45 | Primary Proxmox node — management UI and API |
| pve-prod-02 (Dell OptiPlex 5000 SFF) | 192.168.10.12 | 1GbE RJ45 | Secondary Proxmox node — replacement hardware installed November 13, 2026 |
| pve-prod-03 (Lenovo ThinkCentre M90q Tiny) | 192.168.10.13 | vmbr0 via nic0 | Third Proxmox voting node — management UI and API |
| pi-prod-01 (Raspberry Pi) | 192.168.10.20 | GbE RJ45 | Uptime Kuma monitoring; historical QDevice role. No longer participates in Proxmox quorum. |
| 192.168.10.30–.49 | — | — | Reserved — future expansion / IPMI / OOB |
| 192.168.10.100+ | — | — | DHCP pool if needed |

> **Note:** VLAN 10 outbound reach to all VLANs requires **Auto Allow Return Traffic** enabled on the Allow Management to All firewall rule — without it, return packets to VLAN 10 hosts are dropped. See `firewall-rules.md`.

### Proxmox Cluster

**Cluster:** `homelab-v3`  
**Topology:** Three voting Proxmox nodes  
**Quorum:** Two of three node votes required  
**External QDevice:** Removed; not configured  
**Proxmox HA:** Not enabled

Live verification on November 14, 2026 confirmed three online, quorate nodes:

| Node | Management IP | Hardware | Status |
|---|---|---|---|
| pve-prod-01 | 192.168.10.11 | Minisforum MS-A2 | Online |
| pve-prod-02 | 192.168.10.12 | Dell OptiPlex 5000 SFF | Online |
| pve-prod-03 | 192.168.10.13 | Lenovo ThinkCentre M90q Tiny | Online |

The `pve-prod-03` management bridge is `vmbr0`, configured with `192.168.10.13/24`, gateway `192.168.10.1`, and VLAN-aware bridging through `nic0`.

The original two-node cluster used a Raspberry Pi QDevice at `192.168.10.20`. That QDevice was removed before the Lenovo joined the cluster in November 2026.

The current cluster uses three Proxmox voting members, one vote each. The Raspberry Pi no longer contributes a vote, and its availability is not required for cluster quorum.

**Important:** Cluster quorum provides control-plane availability, not automatic guest failover. Proxmox HA is disabled, and guest disks reside primarily on node-local ZFS storage.

See:

- `architecture/compute/pve-prod-01.md`
- `architecture/compute/pve-prod-02.md`
- `architecture/compute/pve-prod-03.md`
- `architecture/compute/pi-prod-01.md`

---

## VLAN 20 — Trusted (192.168.20.0/24)

Personal laptops, phones, and trusted client devices.

The historical v2 migration entries below are retained from the original IP plan. Their current operating status and any associated decommissioning have not been reverified as part of this update.

| Device | IP | Notes |
|---|---|---|
| Personal devices | DHCP | Laptops, phones, trusted clients |

---

## VLAN 30 — Services (192.168.30.0/24)

Production service VMs, LXCs, and the NAS data interface.

Containers inside VMs are generally accessed through their host VM's IP address rather than receiving individual VLAN IPs. Services exposed through the central Traefik instance use `docker-prod-01`.

Not every v3 VM is on VLAN 30. The Dragonwilds dedicated game server uses VLAN 50, which is documented separately below.

### Active Service Addressing

| Device | IP | Interface | Notes |
|---|---|---|---|
| dns-prod-01 (AdGuard LXC, pve-prod-01) | 192.168.30.10 | Virtual (VLAN tag 30) | Primary DNS — AdGuard Home, CT 100, 1 GiB RAM |
| docker-prod-01 (media/apps VM, pve-prod-01) | 192.168.30.11 | Virtual (VLAN tag 30) | Primary Docker host, including Traefik and media applications. Mounts `192.168.30.16:/mnt/user/data` as `/data` via NFSv3 — single mount covering media and downloads. |
| auth-prod-01 (Authentik VM, pve-prod-03) | 192.168.30.13 | Virtual (VLAN tag 30) | Authentik IdP — VM 104, 2 vCPU, 4 GiB RAM. Migrated from pve-prod-01 to pve-prod-03 on November 14, 2026. |
| immich-prod-01 (Immich VM, pve-prod-02) | 192.168.30.14 | Virtual (VLAN tag 30) | Immich — VM 105, 2 vCPU, 6 GiB RAM. Runs on replacement Dell OptiPlex 5000 SFF. NFS photos mount: `192.168.30.16:/mnt/user/photos` → `/data/photos`. |
| dns-prod-02 (AdGuard LXC, pve-prod-03) | 192.168.30.15 | Virtual (VLAN tag 30) | Secondary DNS — CT 102, 1 GiB RAM. Synced from dns-prod-01 via adguardhome-sync. |
| nas-prod-01 — **data/NFS/PBS** | 192.168.30.16 | X710 Port 2 SFP+ | NFS exports, Plex traffic, **PBS Docker at `https://192.168.30.16:8007`**. PBS datastore uses local direct ZFS, not NFS. |
| observer-prod-01 (AI Observer VM, pve-prod-01) | 192.168.30.17 | Virtual (VLAN tag 30, ens18) | AI Observer — VM 107, read-only infrastructure monitoring through Proxmox, Unraid, Komodo, and UniFi MCP integrations. Dedicated Proxmox API access to `192.168.10.11:8006`. |
| plex (Docker container on nas-prod-01) | 192.168.30.2 | X710 Port 2 SFP+ | Plex Media Server — Docker container on nas-prod-01, bound to eth1 (VLAN 30) |
| 192.168.30.100+ | — | — | DHCP / sandbox / test / future k3s nodes |

**Verification scope:**

The November 14, 2026 live Proxmox inventory confirmed the current host assignment and running status for CT 100, CT 102, and VMs 101, 104, 105, and 107.

Guest IP addresses remain those documented in their individual service and compute files. The Authentik guest address and default gateway were explicitly checked following its November 14 migration.

The November 13 Immich migration retained VM 105's existing service IP, VLAN, and virtual NIC configuration. Application and NFS access were verified afterward.

### Retired Service Addresses

| Former service | Historical IP | Status | Notes |
|---|---|---|---|
| pbs-prod-01 (original PBS VM 103) | 192.168.30.12 | Retired and deleted | The original VM was stopped during the November 3–5 PBS migration and destroyed before the November 13 replacement of the old pve-prod-02. |

**Important:** `192.168.30.12` is a historical allocation, not the active PBS endpoint.

The original PBS VM 103 no longer exists in the Proxmox cluster. It should not be listed as stopped, retained, or available as a rollback VM.

The active PBS service runs as a Docker container on `nas-prod-01`, reachable at:

`https://192.168.30.16:8007`

The previous PBS VM and its endpoint were retired after the datastore was migrated to direct ZFS access through Docker on Unraid.

The former IP should not be considered automatically available for reassignment without checking current reservations, DHCP configuration, and network occupancy.

### Current Proxmox Guest Placement

**Last verified:** November 14, 2026.

| VMID | Guest | Proxmox host | VLAN | Service IP | State |
|---|---|---|---|---|---|
| 100 | dns-prod-01 | pve-prod-01 | 30 | 192.168.30.10 | Running |
| 101 | docker-prod-01 | pve-prod-01 | 30 | 192.168.30.11 | Running |
| 102 | dns-prod-02 | pve-prod-03 | 30 | 192.168.30.15 | Running |
| 104 | auth-prod-01 | pve-prod-03 | 30 | 192.168.30.13 | Running |
| 105 | immich-prod-01 | pve-prod-02 | 30 | 192.168.30.14 | Running |
| 106 | dragonwilds-server | pve-prod-01 | 50 | Not verified | Stopped |
| 107 | observer-prod-01 | pve-prod-01 | 30 | 192.168.30.17 | Running |

This inventory is based on live Proxmox guest placement and the documented guest addressing, not a new DHCP or IPAM scan.

### AI Observer — Network Configuration

The observer VM is deployed on `pve-prod-01` as VM 107.

| Setting | Value |
|---|---|
| Hostname | observer-prod-01 |
| IP address | 192.168.30.17/24 |
| VLAN | 30 — Services |
| Network interface | ens18 |
| Proxmox bridge | vmbr0 |
| Proxmox VLAN tag | 30 |
| Gateway | 192.168.30.1 |
| Primary DNS | 192.168.30.10 |
| Secondary DNS | 192.168.30.15 |

**Verification:** The IP address, subnet prefix, and interface name were verified through the live Proxmox QEMU guest agent on 2026-10-30 UTC. Gateway and DNS values reflect the documented deployment configuration and were not separately queried during this update.

The observer VM remained running on `pve-prod-01` during the November 14 cluster verification.

#### Network Access

The observer requires:

- Outbound HTTPS to the OpenAI Secure MCP Tunnel control plane.
- HTTPS access to the Proxmox API at `192.168.10.11:8006`.
- Localhost access to the Proxmox MCP server at `127.0.0.1:8080`.
- Localhost access to the tunnel health endpoint at `127.0.0.1:8081`.

The Proxmox API connection is permitted through a narrowly scoped UniFi firewall exception:

| Source | Destination | Protocol | Port | Action |
|---|---|---|---|---|
| 192.168.30.17 | 192.168.10.11 | TCP | 8006 | Allow |

This rule does not grant general Services-to-Management network access.

The Proxmox MCP accesses the cluster through the configured primary-node API endpoint. The existence of three Proxmox nodes does not imply that equivalent firewall exceptions were added to the other node management IPs.

No publicly exposed MCP endpoint, Traefik route, or inbound internet port forwarding is configured for AI Observer.

See:

- `services/infra/ai-observer.md`
- `architecture/network/firewall-rules.md`
- `runbooks/ai-observer-operations.md`

### NAS Dual-Interface Design

`nas-prod-01` has two logical network presences:

- **Onboard 2.5GbE at 192.168.10.10:** Management interface (Unraid UI); Docker PBS Host networking may also expose TCP 8007 here, pending access verification.
- **X710 Port 2 at 192.168.30.16:** Data plane (NFS, Plex, Docker PBS TCP 8007).

Services on VLAN 30 mount NFS from `192.168.30.16` without crossing into VLAN 10. Proxmox hosts also connect to the local Unraid PBS Docker service on `192.168.30.16:8007`; this backup traffic no longer traverses an NFS mount inside PBS.

The firewall denies general VLAN 30 → VLAN 10 access. Narrow, explicitly documented exceptions — such as the AI Observer's access to the Proxmox API — are permitted without opening the entire management network.

The AI Observer exception does not grant access to the Unraid management UI.

> **X710 Port 1** (DAC to MS-A2) is on a private point-to-point storage subnet — not on any VLAN, not reachable from the LAN. See Storage Network section below.

---

## VLAN 40 — IoT (192.168.40.0/24)

Internet access only. No inter-VLAN routing.

| Device | IP | Notes |
|---|---|---|
| Gateway | 192.168.40.1 | UDM-SE |
| IoT devices | DHCP 192.168.40.100+ | Smart home, printers, untrusted endpoints |

---

## VLAN 50 — Game Servers

Dedicated network segment for game-server workloads.

The VLAN tag is verified in the Proxmox configuration for `dragonwilds-server` (VM 106).

The network subnet, gateway, DHCP configuration, firewall policies, and the guest's assigned IP address have not been verified during this documentation update.

| Device | VLAN | IP | Notes |
|---|---|---|---|
| dragonwilds-server (VM, pve-prod-01) | 50 | Not verified | RuneScape: Dragonwilds dedicated server. Proxmox VM 106, attached to vmbr0 with VLAN tag 50. VM was stopped during the November 14 live inventory. |

### Pending Network Verification

Before recording the full VLAN 50 IP plan, verify:

- Configured subnet and gateway in UniFi.
- Static or DHCP addressing for the game-server VM.
- Current guest IP address.
- Inter-VLAN firewall policies and isolation.
- Intended internet exposure and any port-forwarding rules.

Do not infer the subnet or IP address from VLAN ID 50 alone.

---

## Storage Network — Point-to-Point (No VLAN)

| Link | Interface A | Interface B | IP A | IP B | Subnet | Purpose |
|---|---|---|---|---|---|---|
| NAS ↔ MS-A2 | nas-prod-01 X710 Port 1 (eth2) | pve-prod-01 SFP+ Port 1 (nic3) | 10.0.0.2/30 | 10.0.0.1/30 | 10.0.0.0/30 | Dedicated 10GbE storage traffic. Not on LAN switch. Not routed. |

> This link is isolated — only these two devices can see it. Proxmox on pve-prod-01 uses this link for NFS storage mounts (`nas-isos`). VMs running on pve-prod-01 (including docker-prod-01) cannot access this link directly — they use VLAN 30 (`192.168.30.16`) for NFS instead.

Both `pve-prod-02` and `pve-prod-03` use `nas-isos-p2`, which mounts the same Unraid ISO export from `192.168.30.16` across the Services network.

The three-node architecture does not extend the private 10GbE point-to-point DAC link to nodes 02 or 03.

### Proxmox ISO Storage Mapping

| Storage ID | Proxmox nodes | NAS endpoint | Purpose |
|---|---|---|---|
| `nas-isos` | pve-prod-01 | `10.0.0.2:/mnt/user/isos` | ISO library over private DAC link |
| `nas-isos-p2` | pve-prod-02, pve-prod-03 | `192.168.30.16:/mnt/user/isos` | ISO library over Services VLAN |

Both definitions reference the same NAS ISO export through different network paths.

The November 14 live cluster check confirmed both storage definitions available on their assigned nodes.

---

## Recent Changes

### November 13, 2026 — Proxmox Node Replacement and DNS Placement

The Homelab v3 cluster was expanded and its original two-node architecture replaced with three voting compute nodes.

The original `pve-prod-02` (Dell OptiPlex 3070 Micro) was replaced with a Dell OptiPlex 5000 SFF, retaining hostname `pve-prod-02` and management IP `192.168.10.12`.

**Secondary DNS:**

- `dns-prod-02` (CT 102) was moved from the original `pve-prod-02` to `pve-prod-03`.
- Its service address remained `192.168.30.15`.
- Its memory allocation was increased from 512 MiB to 1024 MiB.
- AdGuard functionality and synchronization were verified following the relocation.

**Immich:**

- VM 105 was temporarily hosted on `pve-prod-03` while the old `pve-prod-02` was retired.
- After the replacement Dell joined the cluster, Immich was migrated offline to `pve-prod-02` on November 13.
- Its service address remained `192.168.30.14`.
- The VM configuration, application UI, Docker containers, and NFS photos mount were verified after migration.

**PBS:**

- The former PBS VM 103 had already been replaced by Docker PBS on Unraid.
- VM 103 was deleted before the original `pve-prod-02` was replaced.
- The active PBS endpoint remained `192.168.30.16:8007`.

The three-node cluster was confirmed online and quorate.

See `services/infra/adguard.md`, `services/photos/immich.md`, and the Proxmox compute documentation.

### November 14, 2026 — Authentik Migration

VM 104 (`auth-prod-01`) was migrated offline from `pve-prod-01` to `pve-prod-03`.

**Network configuration preserved:**

| Setting | Value |
|---|---|
| Guest hostname | auth-prod-01 |
| Service IP | 192.168.30.13/24 |
| Gateway | 192.168.30.1 |
| VLAN | 30 — Services |
| Bridge | vmbr0 |
| Virtual MAC | BC:24:11:CA:5D:1B |
| Proxmox host | pve-prod-03 |

**Verification completed:**

- VM 104 running on the Lenovo.
- Guest IP address and default route correct.
- Authentik application containers running.
- Local health endpoint returned HTTP 200.
- DNS rewrite for `auth.giohosted.com` resolved to `192.168.30.11`.
- Traefik backend check returned HTTP 302.
- Public Authentik URL returned HTTP 302.
- SSO functional test passed.
- A fresh PBS snapshot backup completed successfully from `pve-prod-03`.

No change to Authentik's service IP, gateway, VLAN, DNS rewrite, or Traefik route was necessary.

The migration reduced the number of critical services dependent on the primary Proxmox node ahead of its planned Proxmox upgrade.

### November 14, 2026 — Three-Node Network Verification

Live Proxmox checks confirmed:

- `pve-prod-01` online at `192.168.10.11`.
- `pve-prod-02` online at `192.168.10.12`.
- `pve-prod-03` online at `192.168.10.13`.
- Cluster quorate with all three voting members.
- No external QDevice configured.
- No Proxmox HA-managed guests.
- Primary DNS CT 100 on node 01.
- Secondary DNS CT 102 and Authentik VM 104 on node 03.
- Immich VM 105 on node 02.
- Primary Docker VM 101 and AI Observer VM 107 on node 01.
- PBS storage available on all three Proxmox nodes.
- ISO storage definitions available on their designated hosts.

No network addressing changes or UniFi firewall-rule modifications were performed during this documentation update.

---

## Open Verification Items

- Confirm the current physical switch-port connections for pve-prod-02 and pve-prod-03 following the November 14 rack maintenance.
- Confirm the full VLAN 50 subnet, gateway, VM addressing, and firewall configuration.
- Independently verify whether Docker PBS TCP 8007 is exposed on the NAS management interface as well as the intended data interface.
- Review historical or reserved IP addresses against current DHCP reservations and network occupancy before reassigning them.
- Verify any network access changes separately before changing the established Services-to-Management firewall boundaries.

---

## Related Documentation

- `architecture/network/firewall-rules.md` — Inter-VLAN security policies and AI Observer exception
- `architecture/network/vlan-design.md` — VLAN architecture
- `architecture/compute/pve-prod-01.md` — Primary Proxmox node
- `architecture/compute/pve-prod-02.md` — Replacement Dell node and Immich
- `architecture/compute/pve-prod-03.md` — Lenovo node, secondary DNS, and Authentik
- `architecture/hardware/inventory.md` — Current hardware and guest inventory
- `architecture/hardware/rack-layout.md` — Physical rack and cabling documentation
- `architecture/storage/nas-prod-01.md` — NAS networking and storage services
- `services/infra/authentik.md` — Authentik migration and networking
- `services/infra/adguard.md` — Primary and secondary DNS
- `services/photos/immich.md` — Immich configuration and migration
- `services/infra/ai-observer.md` — Read-only monitoring architecture
- `runbooks/backup-schedule.md` — Proxmox PBS backup coverage
