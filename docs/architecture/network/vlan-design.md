# VLAN Design — Homelab v3

**Last Updated:** 2026-11-14  
**Status:** Active — five configured networks across the Internal and Dmz firewall zones  
**Enforced by:** UniFi Dream Machine Special Edition (UDM-SE)  
**Verification:** Live UniFi Network API and Proxmox cluster inspection, November 14, 2026

---

## Overview

Homelab v3 uses network segmentation to separate infrastructure management, trusted clients, production services, IoT devices, and game-server workloads.

The UniFi Dream Machine SE handles inter-network routing and firewall enforcement.

The architecture uses five networks:

| Network | VLAN ID | Subnet | Gateway | Firewall Zone |
|---|---|---|---|---|
| Management | 1 (default/untagged) | 192.168.10.0/24 | 192.168.10.1 | Internal |
| Trusted | 20 | 192.168.20.0/24 | 192.168.20.1 | Internal |
| Services | 30 | 192.168.30.0/24 | 192.168.30.1 | Internal |
| IoT | 40 | 192.168.40.0/24 | 192.168.40.1 | Internal |
| Game Servers | 50 | 192.168.50.0/24 | 192.168.50.1 | Dmz |

All five networks were confirmed enabled through the UniFi API on November 14, 2026.

### Firewall Design

The intended security model prioritizes:

- Separation of management interfaces from application workloads.
- Restricted administrative access from trusted devices.
- Denial of general Services-to-Management connectivity.
- Isolation of untrusted IoT devices.
- Isolation of game servers through a separate DMZ firewall zone.
- Narrow, explicit exceptions for required infrastructure dependencies.
- Stateful return traffic for authorized connections.

**Important implementation detail:**

The current configuration is **not a universal default-deny inter-VLAN firewall**.

The live UniFi configuration includes an enabled system-defined `Allow All Traffic` fallback for Internal-to-Internal traffic.

Explicit user-defined block policies implement the primary restrictions between Management, Trusted, Services, and IoT.

Consequently, missing directional rules may permit traffic that the original design intended to restrict.

The design should not be considered fully security-validated until the effective policies and representative traffic paths have been checked.

Detailed IP allocations and firewall rules are documented separately:

- `architecture/network/ip-addresses.md`
- `architecture/network/firewall-rules.md`

---

## Management — 192.168.10.0/24

**UniFi network name:** Management  
**Actual VLAN ID:** 1 (default network)  
**Historical documentation name:** VLAN 10 — Management  
**Gateway:** 192.168.10.1  
**Firewall zone:** Internal  
**Role:** Infrastructure administration and control-plane services

This network is dedicated to infrastructure management interfaces and services.

### Key Devices

| Device | IP | Role |
|---|---|---|
| UDM-SE | 192.168.10.1 | Router, gateway, firewall, UniFi controller |
| USW-Pro-Max-24 | 192.168.10.2 | Core network switch |
| nas-prod-01 | 192.168.10.10 | Unraid management |
| pve-prod-01 | 192.168.10.11 | Primary Proxmox node |
| pve-prod-02 | 192.168.10.12 | Dell OptiPlex 5000 SFF Proxmox node |
| pve-prod-03 | 192.168.10.13 | Lenovo ThinkCentre M90q Proxmox node |
| pi-prod-01 | 192.168.10.20 | Monitoring host; former Proxmox QDevice |

The three-node Proxmox cluster was verified online and quorate on November 14, 2026.

### Management VLAN Naming Discrepancy

Earlier Homelab v3 documentation consistently referred to the Management network as **VLAN 10**.

However, the live UniFi Network API reports:

| Setting | Value |
|---|---|
| Name | Management |
| VLAN ID | 1 |
| Default network | Yes |
| IPv4 gateway | 192.168.10.1 |
| Prefix length | 24 |
| Firewall zone | Internal |

The current Management network is therefore UniFi's default/untagged network, despite using addresses in the `192.168.10.0/24` subnet.

**VLAN IDs and IP subnets are independent concepts.** An IP subnet using `192.168.10.0/24` does not require VLAN tag 10.

The documentation's historical "VLAN 10" terminology should not be interpreted as proof that the management uplink is tagged with IEEE 802.1Q VLAN ID 10.

Before changing terminology across all infrastructure documentation, confirm the relevant switch-port profiles and Proxmox management interface configurations.

**Do not change the working VLAN configuration merely to make its tag match the subnet number.**

### Access Policy

The live UniFi configuration includes:

- Management-to-Trusted: Allowed.
- Management-to-Services: Allowed.
- Management-to-IoT: Allowed.
- Trusted-to-Management: Restricted to approved administrative ports.
- Services-to-Management: Blocked except for targeted exceptions.
- IoT-to-Management: Blocked by the IoT-to-Internal rule.

Management hosts retain broad outbound access to the established Internal networks for administrative operations.

Inbound access from other networks is restricted through the applicable user-defined firewall rules.

### Management DNS

The live DHCP configuration supplies:

- Primary DNS: `9.9.9.9`
- Secondary DNS: `1.1.1.1`

This avoids making Management network clients dependent on Proxmox-hosted AdGuard services for DNS.

Hosts with static configurations may use their own DNS settings; those should be verified separately.

### AI Observer Exception

`observer-prod-01` resides on the Services network and requires access to the Proxmox API on the Management network.

A targeted firewall exception allows:

| Setting | Value |
|---|---|
| Source | 192.168.30.17 |
| Destination | 192.168.10.11 |
| Protocol | TCP |
| Destination port | 8006 |
| Action | ALLOW |
| Return traffic | Enabled |

This connection supports the read-only Proxmox MCP integration.

**November 14 verification:** The rule exists, is enabled, and is ordered before the general Services-to-Management block.

The exception does not grant unrestricted access from Services to Management.

It also does not explicitly allow direct access to the management APIs of `pve-prod-02` or `pve-prod-03`.

The observer currently accesses the cluster through the node 01 Proxmox API endpoint.

See `services/infra/ai-observer.md`.

### NAS Management Separation

The NAS uses `192.168.10.10` for its management interface.

Production NFS and service-facing traffic use its separate Services network interface at `192.168.30.16`.

This separates normal application storage access from the Unraid management plane.

Docker PBS uses Host networking, however, so its TCP 8007 listener may be reachable through multiple NAS interfaces. Actual binding and reachability remain to be verified.

---

## VLAN 20 — Trusted (192.168.20.0/24)

**VLAN ID:** 20  
**Gateway:** 192.168.20.1  
**Firewall zone:** Internal  
**Role:** Trusted personal devices

Dedicated to trusted client devices, including personal laptops, desktops, phones, and other approved endpoints.

### Network Configuration

| Setting | Value |
|---|---|
| Network name | Trusted |
| VLAN ID | 20 |
| Subnet | 192.168.20.0/24 |
| Gateway | 192.168.20.1 |
| DHCP | Enabled |
| DHCP range | 192.168.20.100–192.168.20.254 |
| Primary DNS | 192.168.30.10 |
| Secondary DNS | 192.168.30.15 |
| Firewall zone | Internal |

### Access Policy

Trusted clients are permitted to:

- Access production applications on the Services network.
- Access approved administrative services on Management.
- Access the internet.

The Trusted-to-Services rule allows general access to Services.

The Trusted-to-Management rule uses the `MGMT_ADMIN_PORTS` traffic-matching list.

**Verified administrative ports:**

- 22
- 443
- 8006
- 8090
- 3001

All other Trusted-to-Management traffic is covered by the corresponding explicit block rule.

### Trusted-to-IoT Verification Gap

The November 14 UniFi firewall inspection did not identify an explicit Trusted-to-IoT deny policy.

Because both networks belong to the Internal zone, and the system-defined Internal-to-Internal fallback is `ALLOW`, Trusted-to-IoT traffic should **not** be assumed blocked.

A safe connectivity test is required to establish effective behavior.

If the intended security policy is to deny Trusted-to-IoT traffic, a separate firewall change should be reviewed and explicitly authorized.

No change was implemented during the documentation review.

### Historical v2 Configuration

VLAN 20 was retained from Homelab v2.

During the v3 migration, the previous Proxmox and Docker hosts used this network:

- v2 Proxmox — `192.168.20.2`
- v2 Docker host — `192.168.20.10`

These addresses are historical references.

Their current operational status and decommissioning state were not reverified during this update.

---

## VLAN 30 — Services (192.168.30.0/24)

**VLAN ID:** 30  
**Gateway:** 192.168.30.1  
**Firewall zone:** Internal  
**Role:** Production services, application VMs/LXCs, and service-facing storage

### Network Configuration

| Setting | Value |
|---|---|
| Network name | Services |
| VLAN ID | 30 |
| Subnet | 192.168.30.0/24 |
| Gateway | 192.168.30.1 |
| DHCP | Enabled |
| DHCP range | 192.168.30.100–192.168.30.254 |
| Primary DNS | 192.168.30.10 |
| Secondary DNS | 192.168.30.15 |
| Firewall zone | Internal |

### Current Services

| Service | IP | Host | Purpose |
|---|---|---|---|
| dns-prod-01 | 192.168.30.10 | pve-prod-01 | Primary AdGuard |
| docker-prod-01 | 192.168.30.11 | pve-prod-01 | Primary Docker application host and Traefik |
| auth-prod-01 | 192.168.30.13 | pve-prod-03 | Authentik identity provider |
| immich-prod-01 | 192.168.30.14 | pve-prod-02 | Immich photo service |
| dns-prod-02 | 192.168.30.15 | pve-prod-03 | Secondary AdGuard |
| nas-prod-01 | 192.168.30.16 | Physical Unraid host | Data/NFS/PBS interface |
| observer-prod-01 | 192.168.30.17 | pve-prod-01 | AI Observer |

**Additional service:** Plex runs as a Docker container on `nas-prod-01` and has the documented service address `192.168.30.2`.

The retired PBS VM 103 previously used `192.168.30.12`. It was replaced by Docker PBS on Unraid and destroyed before the November 13 Proxmox node replacement.

The active PBS endpoint is:

`https://192.168.30.16:8007`

### Three-Node Workload Distribution

The Services network spans the three-node Proxmox cluster.

| Proxmox Node | Current Services VLAN Guests |
|---|---|
| pve-prod-01 | dns-prod-01 (CT 100), docker-prod-01 (VM 101), observer-prod-01 (VM 107) |
| pve-prod-02 | immich-prod-01 (VM 105) |
| pve-prod-03 | dns-prod-02 (CT 102), auth-prod-01 (VM 104) |

**November 14 verification:** The Proxmox cluster was online and quorate, and the guest assignments above matched the live inventory.

Primary and secondary DNS now reside on separate Proxmox hosts.

Authentik was moved from node 01 to node 03 on November 14, 2026.

The migration retained:

- Guest IP: `192.168.30.13`
- Gateway: `192.168.30.1`
- VLAN tag: 30
- Original virtual NIC MAC address
- Existing application routing and SSO configuration

Post-migration networking, application health, SSO, and PBS backup checks passed.

### Application Networking

Production Docker containers generally use their host's IP address and Docker networks rather than receiving individual VLAN 30 addresses.

Services published through the central Traefik instance use `docker-prod-01`.

Other services may be accessed directly through their respective hosts.

For example:

- `auth.giohosted.com` resolves to Traefik on `192.168.30.11`.
- Traefik forwards Authentik traffic to `192.168.30.13:9000`.

The Authentik host migration did not require DNS or reverse-proxy changes.

### NAS Data Interface

The NAS uses:

- Management: `192.168.10.10`
- Services data: `192.168.30.16`

Production VMs connect to Unraid's NFS exports through the Services interface.

The primary Docker host uses the documented NFSv3 export:

`192.168.30.16:/mnt/user/data`

The Immich VM uses the documented photos export:

`192.168.30.16:/mnt/user/photos`

PBS Docker accesses its datastore directly through the Unraid host's ZFS filesystem, rather than through the previous PBS VM's NFS mount.

The separate NAS management interface remains subject to Management network firewall restrictions.

### Network Restrictions

The live UniFi configuration includes explicit rules blocking Services-initiated traffic to:

- Management, except targeted exceptions.
- Trusted.
- IoT.

Services retains internet access for:

- Container image pulls.
- Software updates.
- External API calls.
- Cloudflare Tunnel.
- OpenAI Secure MCP Tunnels.
- Other approved application dependencies.

### Targeted Management Exceptions

Two specific Services-to-Management exceptions were confirmed enabled:

| Source | Destination | Port | Purpose |
|---|---|---|---|
| observer-prod-01 — 192.168.30.17 | pve-prod-01 — 192.168.10.11 | TCP 8006 | AI Observer Proxmox API |
| Services network | pi-prod-01 — 192.168.10.20 | Port 45876 | Beszel monitoring |

The Beszel rule uses the configured `Bezsel Ports` group.

Its current API policy did not explicitly restrict the protocol to TCP, despite the historical Beszel documentation describing TCP 45876.

These exceptions are evaluated before the general Services-to-Management block.

No general Services-to-Management allowance was added for the new Proxmox node.

---

## VLAN 40 — IoT (192.168.40.0/24)

**VLAN ID:** 40  
**Gateway:** 192.168.40.1  
**Firewall zone:** Internal  
**Role:** Smart home devices, printers, and other untrusted endpoints

### Network Configuration

| Setting | Value |
|---|---|
| Network name | IoT |
| VLAN ID | 40 |
| Subnet | 192.168.40.0/24 |
| Gateway | 192.168.40.1 |
| DHCP | Enabled |
| DHCP range | 192.168.40.100–192.168.40.254 |
| DHCP DNS | 9.9.9.9 |
| Internet access | Enabled |
| Firewall zone | Internal |

### Intended Security Policy

IoT devices should have internet access without unrestricted connectivity to trusted clients or infrastructure administration.

The live firewall configuration includes an enabled rule:

`Block IoT to All Internal`

The destination uses the `NET_INTERNAL` traffic-matching list containing `192.168.0.0/16`.

The rule is scoped to the Internal firewall zone and blocks IoT-initiated access to other Internal networks.

This is consistent with the original IoT isolation objective for Management, Trusted, and Services.

### IoT-to-DMZ Consideration

Game Servers is assigned to the separate Dmz firewall zone.

The IoT-to-Internal rule does not itself establish an IoT-to-Dmz block.

The live configuration also includes a system-defined Internal-to-Dmz `Allow All Traffic` policy.

Therefore, IoT-to-Game Servers access should not be assumed denied without reviewing effective zone policies and testing representative traffic.

This is a remaining security validation item.

### DNS Handling

The current UniFi DHCP configuration supplies:

`9.9.9.9`

Earlier documentation contained conflicting descriptions of IoT DNS behavior:

1. IoT uses public DNS directly.
2. The UDM-SE intercepts IoT DNS and redirects it to AdGuard.

The November 14 UniFi API verification establishes the DHCP-advertised resolver, but does not independently establish whether DNS interception or NAT redirection is configured.

No custom DNS policy objects were returned by the available DNS policy listing.

**Current status:**

- DHCP-advertised DNS: Verified as `9.9.9.9`.
- DNS interception/NAT behavior: Not verified.
- Actual resolver usage by individual IoT clients: Not verified.

Do not treat the older interception design as confirmed implementation.

### Historical Migration Plan

The original v3 migration plan identified seven IoT devices to move from VLAN 30 to VLAN 40 during Phase 1.

The completion status of those individual migrations has not been reverified during this update.

Preserve the historical Phase 1 record until the device assignments are independently checked.

---

## VLAN 50 — Game Servers (192.168.50.0/24)

**VLAN ID:** 50  
**Gateway:** 192.168.50.1  
**Firewall zone:** Dmz  
**Role:** Dedicated game-server network

The Game Servers network separates hosted game workloads from production services and infrastructure management.

Unlike Management, Trusted, Services, and IoT, this network belongs to UniFi's **Dmz** firewall zone.

### Verified Network Configuration

| Setting | Value |
|---|---|
| Network name | Game Servers |
| VLAN ID | 50 |
| Subnet | 192.168.50.0/24 |
| Gateway | 192.168.50.1 |
| DHCP | Enabled |
| DHCP range | 192.168.50.6–192.168.50.254 |
| Firewall zone | Dmz |
| Internet access | Enabled |
| Network isolation toggle | Disabled |
| Custom DHCP DNS override | None returned |

These settings were verified through the live UniFi Network API on November 14, 2026.

### Current Game-Server Guest

| Setting | Value |
|---|---|
| Guest | dragonwilds-server |
| VMID | 106 |
| Proxmox host | pve-prod-01 |
| Bridge | vmbr0 |
| VLAN tag | 50 |
| Guest state during November 14 check | Stopped |
| Guest IP | Not verified |

The Proxmox VLAN tag was previously confirmed through the live Proxmox integration.

The guest's configured operating-system IP was not independently verified.

Do not infer the guest IP from the DHCP range.

### DMZ Firewall Behavior

The live UniFi firewall inspection returned the following system-defined zone policies:

| Direction | Observed Policy |
|---|---|
| Internal → Dmz | Allow All Traffic |
| Dmz → Internal | Block new traffic by default; established/related return traffic permitted |
| Dmz → External | Allow All Traffic |
| External → Dmz | Block new traffic by default; established/related return traffic permitted |

This provides an important security distinction:

**DMZ workloads are restricted from initiating new connections into the Internal zone, but the current default policy allows Internal networks to initiate connections toward the DMZ.**

Therefore, the DMZ is not fully isolated from internal clients in both directions.

The existing policy may be intentional for game-server administration, but its scope should be reviewed against the desired security model.

### Port Forwarding

The existence of VLAN 50 and its zone policies does not establish which game-server ports are exposed to the internet.

Specific inbound rules, port forwards, destination addresses, and application-level listening services must be verified separately.

The November 14 documentation review did not establish a complete Dragonwilds port-forward inventory.

### Remaining Verification

- Determine the guest's actual IP address.
- Review any game-server port forwards.
- Confirm the intended Internal-to-DMZ access requirements.
- Verify DMZ-to-Internal restrictions with safe tests.
- Determine whether IoT-to-DMZ access should be blocked.
- Confirm actual internet access and application behavior when the guest is running.

Do not modify the firewall during documentation reconciliation without a separate approved implementation plan.

---

## Firewall Zones

UniFi Network currently exposes six zones:

| Zone | Role in Homelab v3 |
|---|---|
| Internal | Management, Trusted, Services, IoT |
| Dmz | Game Servers |
| Gateway | Traffic involving the UniFi gateway |
| External | WAN/internet |
| Vpn | VPN-connected clients |
| Hotspot | Reserved/system-defined hotspot zone |

### Internal Zone

The Internal zone contains four networks that are further segmented through user-defined firewall policies.

A system-defined Internal-to-Internal fallback allows traffic not matched by higher-priority policies.

Consequently, explicit block rules are critical to the intended segmentation.

### Dmz Zone

Game Servers is assigned to the Dmz zone.

The zone has its own default traffic behavior, including restrictions on new DMZ-to-Internal connections.

This does not automatically provide complete isolation in every direction.

### Gateway, External, VPN, and Hotspot

These are UniFi's other system-defined zones.

Their existence and names were verified through the live API.

Detailed gateway service policy, VPN configuration, WAN exposure, and hotspot operation are outside the scope of this VLAN design document.

The firewall runbook records additional verified policies, including the existing VPN-to-Internal allowance and Plex external-access rule.

---

## Inter-VLAN Policy Summary

The table below summarizes the currently observed policy model for the primary internal networks.

| Source → Destination | Management | Trusted | Services | IoT | Internet |
|---|---|---|---|---|---|
| Management | — | Allow | Allow | Allow | Allow |
| Trusted | Admin ports only | — | Allow | No explicit deny verified | Allow |
| Services | Block except targeted exceptions | Block | — | Block | Allow |
| IoT | Block | Block | Block | — | Allow |

Game Servers is intentionally excluded from the Internal-to-Internal matrix because it belongs to the separate Dmz firewall zone.

For DMZ traffic:

- Internal → Dmz: System-defined allow.
- Dmz → Internal: System-defined block for new connections, with return-traffic exceptions.
- Dmz → External: System-defined allow.
- External → Dmz: System-defined block for new connections unless another policy or NAT/forwarding configuration permits it.

**Interpretation:** The matrix records retrieved policies and configuration, not a comprehensive packet-level verification of every network path.

See `architecture/network/firewall-rules.md` for detailed rule configuration and verification findings.

---

## DNS Architecture

Homelab v3 uses two AdGuard Home instances for resilient internal DNS.

| Role | Host | Proxmox Node | IP |
|---|---|---|---|
| Primary AdGuard | dns-prod-01 (CT 100) | pve-prod-01 | 192.168.30.10 |
| Secondary AdGuard | dns-prod-02 (CT 102) | pve-prod-03 | 192.168.30.15 |

Secondary AdGuard configuration is synchronized from the primary using `adguardhome-sync`.

The two DNS containers reside on different physical Proxmox hosts.

### DHCP DNS Assignments

| Network | DHCP-Provided DNS |
|---|---|
| Management | 9.9.9.9, 1.1.1.1 |
| Trusted | 192.168.30.10, 192.168.30.15 |
| Services | 192.168.30.10, 192.168.30.15 |
| IoT | 9.9.9.9 |
| Game Servers | No custom DNS override returned |

The values above were retrieved from the live UniFi network settings on November 14.

These are DHCP-provided settings; individual statically configured devices may use different resolvers.

**Design benefit:** Primary DNS and secondary DNS reside on separate Proxmox nodes. A planned outage of node 01 does not directly shut down the secondary DNS container.

However, DNS redundancy does not guarantee every client will immediately fail over to the secondary resolver. Actual client behavior depends on its operating system, DNS configuration, and application behavior.

---

## Proxmox Network Architecture

The Proxmox cluster consists of three voting nodes:

| Node | Hardware | Management IP | Management Bridge |
|---|---|---|---|
| pve-prod-01 | Minisforum MS-A2 | 192.168.10.11 | vmbr0 |
| pve-prod-02 | Dell OptiPlex 5000 SFF | 192.168.10.12 | vmbr0 |
| pve-prod-03 | Lenovo ThinkCentre M90q Tiny | 192.168.10.13 | vmbr0 |

Proxmox node 03 was added after removal of the former Raspberry Pi QDevice.

The cluster has three voting members, with two votes required for quorum.

No Proxmox HA policy is enabled.

### Guest VLAN Tagging

Proxmox guests use tagged networks according to their assigned roles.

Examples:

- CT 100: Services VLAN 30.
- VM 101: Services VLAN 30.
- CT 102: Services VLAN 30.
- VM 104: Services VLAN 30.
- VM 105: Services VLAN 30.
- VM 106: Game Servers VLAN 50.
- VM 107: Services VLAN 30.

The physical bridges are VLAN-aware to support guest traffic.

The Management network is currently configured as UniFi's default/untagged network rather than a separately tagged VLAN 10.

The existing switch and host management traffic should be inspected before globally updating older references to VLAN 10.

### Storage Network

`pve-prod-01` and `nas-prod-01` also have a dedicated point-to-point 10 GbE DAC connection.

| Endpoint | IP |
|---|---|
| pve-prod-01 storage interface | 10.0.0.1/30 |
| nas-prod-01 storage interface | 10.0.0.2/30 |

This private connection is not routed through the UDM-SE and is not an additional VLAN.

It is used for dedicated NAS-to-primary-Proxmox storage traffic.

Nodes 02 and 03 access the NAS ISO export using the Services network instead.

---

## Design Principles

Homelab v3's VLAN architecture follows these principles:

1. **Separate management from services.** Production workloads should not have unrestricted access to infrastructure control interfaces.

2. **Keep trusted clients distinct.** Personal devices can access required services without sharing the same broadcast domain.

3. **Isolate untrusted devices.** IoT devices should not have general access to trusted clients or infrastructure management.

4. **Use targeted exceptions.** Required cross-network dependencies should use the narrowest practical source, destination, and port restrictions.

5. **Maintain dedicated workload segments where useful.** Game servers use VLAN 50 and the DMZ zone rather than sharing the general Services network.

6. **Distribute critical infrastructure.** Primary and secondary DNS are hosted on different Proxmox nodes, and Authentik is separated from the primary Docker host.

7. **Verify actual enforcement.** A VLAN assignment alone does not guarantee isolation. UniFi firewall policy and connectivity must also be checked.

8. **Distinguish design from implementation.** Intended security restrictions should not be described as enforced until verified in the current configuration.

9. **Avoid unnecessary complexity.** Prefer understandable, maintainable rules and minimal host-specific exceptions appropriate for a personal homelab.

---

## Open Verification and Documentation Items

### Network Configuration

- Verify the physical switch-port profiles and host interface tagging for the default Management network.
- Reconcile historical "VLAN 10" terminology with UniFi's actual VLAN ID 1 configuration.
- Confirm the effective Trusted-to-IoT firewall behavior.
- Review whether the Internal-to-DMZ default allowance is appropriate.
- Verify the intended IoT-to-DMZ security policy.
- Confirm IoT DNS interception/NAT behavior.
- Confirm the game-server VM's actual IP address and port forwards.
- Validate representative cross-network restrictions with safe tests.
- Verify the interface-level exposure of Docker PBS TCP 8007.

### Firewall Housekeeping

- Review the `Proxmox host` UniFi address group, which currently contains only `192.168.10.11` and `192.168.10.12`.
- Confirm whether `192.168.10.13` should be included after checking the group's consumers.
- Identify the service requiring Management administrative port 3001.
- Confirm protocol-specific restrictions on relevant traffic-matching-list policies.
- Review any inherited or unnecessary firewall objects from Homelab v2.

No firewall modifications were performed during the November 14 documentation review.

### Related Documentation

The following documents should be reconciled where they retain older terminology or assumptions:

- `architecture/network/ip-addresses.md`
- `architecture/network/firewall-rules.md`
- `architecture/hardware/inventory.md`
- `architecture/hardware/rack-layout.md`
- `architecture/compute/pve-prod-01.md`
- `architecture/compute/pve-prod-02.md`
- `architecture/compute/pve-prod-03.md`

Do not change host networking or UniFi VLAN IDs solely to make previously written documentation consistent.

---

## Related Documentation

- `architecture/network/ip-addresses.md` — Infrastructure IP allocations, guest placement, and subnets.
- `architecture/network/firewall-rules.md` — Inter-VLAN policies, rule priorities, and security exceptions.
- `architecture/hardware/inventory.md` — Current physical hardware and guest distribution.
- `architecture/hardware/rack-layout.md` — Rack placement, physical connections, and switch-port mapping.
- `architecture/compute/pve-prod-01.md` — Primary Proxmox node.
- `architecture/compute/pve-prod-02.md` — Replacement Dell Proxmox node.
- `architecture/compute/pve-prod-03.md` — Lenovo third voting node.
- `architecture/storage/nas-prod-01.md` — NAS interfaces and dedicated storage networking.
- `services/infra/ai-observer.md` — AI Observer network access and security.
- `services/infra/adguard.md` — Primary and secondary DNS.
- `services/infra/authentik.md` — Identity provider and November 14 host migration.
- `services/monitoring/beszel.md` — Beszel monitoring connectivity.

---

## Verification Status — November 14, 2026

### Live-Verified

- All five UniFi networks exist and are enabled.
- Management is the default network with VLAN ID 1.
- Trusted, Services, and IoT use VLAN IDs 20, 30, and 40.
- Game Servers uses VLAN ID 50 and belongs to the Dmz firewall zone.
- Management, Trusted, Services, and IoT belong to the Internal zone.
- Current subnets and gateways for all five networks.
- Current DHCP ranges and advertised DNS settings.
- Ten enabled user-defined Internal-to-Internal firewall policies.
- System-defined Internal-to-Internal Allow All Traffic fallback.
- Internal-to-Dmz allowance and Dmz-to-Internal blocking behavior at the policy level.
- AI Observer Proxmox API exception.
- Beszel monitoring exception.
- Three-node Proxmox cluster, online and quorate.
- Current VM/CT placement, including Authentik on pve-prod-03.
- Primary and secondary DNS on separate Proxmox nodes.

### Documented but Not Fully Reverified

- Exact physical switch-port profiles and Management uplink tagging.
- Effective packet-level firewall restrictions across every network pair.
- Current operating-system IP of dragonwilds-server.
- Game-server external port-forward configuration.
- Complete IoT device migration from the original Phase 1 plan.
- Actual client DNS selection and IoT interception behavior.

### Unresolved

- Whether the Trusted-to-IoT traffic path is intentionally permitted.
- Whether broader Internal-to-Dmz access is desirable.
- Whether the old Management VLAN 10 terminology should be globally replaced after port-profile verification.
- Whether the Proxmox address group should be expanded to include node 03.
- Whether Docker PBS port 8007 is reachable through unintended NAS interfaces.

**Documentation assessment:** The primary network and firewall-zone configuration is now recorded from current live evidence. Remaining discrepancies are identified explicitly rather than assumed resolved.

**Infrastructure changes performed:** None. This update documents the existing configuration only.
