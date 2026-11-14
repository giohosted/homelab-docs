# Homelab v3.0 — Master Architecture & Evolution Roadmap

**Version:** 8.0  
**Owner:** Gio  
**Status:** Operational — hardening in progress; AI-powered audit and observer enhancements planned  
**Last Updated:** 2026-11-14  
**Scope:** Current architecture, remaining operational work, AI Observer evolution, and future learning initiatives

---

> **ROADMAP PURPOSE**  
> This is the *forward-looking project plan and architecture overview*, not a second inventory, runbook, or historical implementation transcript. Detailed current configurations belong in `architecture/`, `services/`, and `runbooks/`; implementation history belongs in `phases/` and `architecture/decisions-log.md`. Where historical plans differ from verified production, the verified production state and its detailed documentation take precedence.

> **STATUS CONVENTIONS**  
> **Implemented / verified:** Tested or observed in the current environment.  
> **Implemented / partially verified:** Configured, but an important health, security, or recovery test remains.  
> **Planned:** Approved direction or candidate scope, but not deployed.  
> **Proposed:** A design to assess before committing to it.  
> **Historical / superseded:** Records an original plan, not an instruction for the current environment.

## SECTION 1 — DIRECTION AND DESIGN PRINCIPLES

### 1.1 Project Evolution

Homelab v3 began as a deliberate rebuild of the v2 environment: a dedicated Unraid NAS, redundant Proxmox boot storage, segmented networks, consistent Docker layouts, Authentik SSO, backups, and monitoring. The original roadmap assumed **two Proxmox nodes, a Raspberry Pi QDevice, and a PBS VM**. That design has materially changed.

As of November 14, 2026, Homelab v3 is an operating three-node Proxmox environment. The old QDevice has been removed; PBS runs as Docker on Unraid; the former Dell OptiPlex 3070 Micro has been replaced; and a dedicated AI Observer VM exposes four carefully scoped read-only infrastructure integrations to ChatGPT.

The roadmap now has **two active, parallel tracks**:

1. **Operational readiness:** Complete the remaining security, backup-recovery, power-protection, and documentation verification work.
2. **AI-powered enhancement and audit:** Use the existing read-only integrations for a comprehensive baseline audit, then add useful application-level visibility, constrained system/file inspection, and periodic, actionable reporting.

**Future Kubernetes exploration is retained**, but remains subordinate to a stable, recoverable Homelab and is not presumed to be the next immediate project.

### 1.2 Design Principles

| Principle | Current direction |
|---|---|
| Reliability over novelty | Protect backups, DNS, identity, storage, and restore ability before adding complexity |
| Documentation-first | Review authoritative Obsidian Markdown before proposing changes; use live integrations to verify current state |
| Read-only AI by default | AI may inspect and recommend; operational writes require an explicit, separately authorized workflow |
| Targeted access | Narrow identities, endpoints, API methods, directories, and ports; no unrestricted shell or Docker socket for the assistant |
| Evidence-driven work | Distinguish an observed issue from a historical incident, hypothesis, or untested recommendation |
| Incremental implementation | One consequential change at a time; verify, then update the relevant existing documentation |
| Home-lab scale | Avoid enterprise-grade machinery without a proportional benefit |
| No alert noise | Report actionable failures, risk changes, and materially useful enhancements—not every available update |
| Preserve the architecture | Extend `observer-prod-01` and existing service tooling rather than create a parallel AI management platform |
| Recovery matters | A completed backup or mounted mirror is not equivalent to a tested restore |

---

## SECTION 2 — VERIFIED CURRENT ARCHITECTURE

*Point-in-time summary: November 14, 2026. Consult the linked service and architecture files for details; this section is intentionally concise.*

### 2.1 Physical Hosts and Cluster

| Host | Hardware | Management IP | Current role / placement |
|---|---|---|---|
| `pve-prod-01` | Minisforum MS-A2, Ryzen 9 7945HX, 32 GiB class RAM | `192.168.10.11` | Primary Proxmox node; PVE 9.1.6; U5–U6 |
| `pve-prod-02` | Dell OptiPlex 5000 SFF, i7-12700, 48 GB RAM, mirrored NVMe | `192.168.10.12` | Replacement Proxmox node; PVE 9.2.26; U17; switch port 19 |
| `pve-prod-03` | Lenovo ThinkCentre M90q Tiny, i7-10700, 24 GB RAM, mirrored NVMe | `192.168.10.13` | Third Proxmox voting node; PVE 9.2.26; U7; switch port 16 |
| `nas-prod-01` | i5-13400, 32 GB RAM, Rosewill RSV-L4412U, Unraid 7.3.4 | `192.168.10.10` | NAS, Plex, Docker PBS, NUT server; U8–U11 |
| `pi-prod-01` | Raspberry Pi 4B | `192.168.10.20` | Beszel hub, Uptime Kuma; shares U7 shelf with Lenovo; **no QDevice** |

`homelab-v3` has **three online voting nodes, quorum of two, and no Proxmox HA**. Cluster quorum does not provide automatic recovery of guests stored on local ZFS. The physical boot redundancy of nodes 02/03 is implemented, but destructive single-drive boot-failover testing has not been performed.

The original OptiPlex 3070 Micro and PBS VM 103 are **retired**, not reserve production nodes.

### 2.2 Current Guest Placement

| Node | Guest | ID | Services IP / notes |
|---|---|---|---|
| pve-prod-01 | `dns-prod-01` | CT 100 | `192.168.30.10` — primary AdGuard |
| pve-prod-01 | `docker-prod-01` | VM 101 | `192.168.30.11` — Docker/Traefik/media/books/torrent stacks; 7 GiB allocation |
| pve-prod-01 | `observer-prod-01` | VM 107 | `192.168.30.17` — dedicated AI Observer; 2 GiB |
| pve-prod-01 | `dragonwilds-server` | VM 106 | Game Servers VLAN 50; stopped at last check; guest IP not verified |
| pve-prod-02 | `immich-prod-01` | VM 105 | `192.168.30.14` — Immich; 6 GiB |
| pve-prod-03 | `dns-prod-02` | CT 102 | `192.168.30.15` — secondary AdGuard |
| pve-prod-03 | `auth-prod-01` | VM 104 | `192.168.30.13` — Authentik; 4 GiB |

Authentik was migrated offline to node 03 on November 14. Application and SSO checks passed, followed by a successful fresh PBS backup from the new host. Primary and secondary DNS now reside on different physical Proxmox nodes.

### 2.3 Network and Segmentation

| Network | Actual UniFi VLAN ID | Subnet | UniFi firewall zone |
|---|---|---|---|
| Management | **1** (default/untagged) | `192.168.10.0/24` | Internal |
| Trusted | 20 | `192.168.20.0/24` | Internal |
| Services | 30 | `192.168.30.0/24` | Internal |
| IoT | 40 | `192.168.40.0/24` | Internal |
| Game Servers | 50 | `192.168.50.0/24` | Dmz |

**Historical correction:** The original roadmap mandated a dedicated Management VLAN ID 10 and said VLAN 1 would not be used. Live UniFi inspection instead shows the Management network as default VLAN ID 1 with subnet `192.168.10.0/24`. The IP subnet and VLAN tag are not synonymous. **No change to the working management VLAN is proposed merely to match the original plan.**

**Firewall validation:** Internal-to-Internal has a system-defined fallback allow policy, with explicit user-defined deny/allow rules implementing much of the intended segmentation. Trusted-to-IoT has no confirmed explicit deny. Internal-to-Dmz is allowed by the current system-defined policy. These paths require risk review and safe testing; never claim a global default-deny policy is already enforced.

The AI Observer has a targeted exception from `192.168.30.17` to `192.168.10.11:8006/TCP` rather than unrestricted Services-to-Management access.

### 2.4 NAS, Data, and Backups

- **Parity array:** Two 12 TB parity drives and three XFS data drives; November 14 disk inventory reports all five `DISK_OK` with zero reported device errors. The October 31 breakout-cable/parity reconstruction incident is resolved; exact electrical root cause remains probable rather than proven.
- **ZFS mirror:** Two 4 TB WD Red Plus drives for critical `photos`, `backups`, and `system` shares; both member devices report `DISK_OK`. A fresh native `zpool status`, scrub history, and recent snapshot execution remain verification items.
- **Media:** Single `data` NFS export for media and downloads, presented as one `/data` mount to preserve hardlinks.
- **Network:** NAS management `192.168.10.10`; 10GbE Services data `192.168.30.16`; implemented point-to-point DAC `10.0.0.2/30` to `pve-prod-01` (`10.0.0.1/30`).
- **PBS:** `proxmox-backup-server` Docker container on Unraid, intended Services endpoint `192.168.30.16:8007`, direct-ZFS datastore `/mnt/zfs-mirror/backups/pbs`; former PBS VM 103 destroyed.
- **Cluster backup job:** Enabled, daily 02:00, snapshot mode, all seven current guests selected; stale exclusion for retired VM 103 remains. November 14 scheduled backups succeeded across nodes 01/02/03, and Authentik's post-migration manual backup succeeded.
- **Recovery assurance:** Prior targeted PBS integrity and file-level restore checks passed, but a full isolated VM/CT restore is not verified. PBS prune/GC/verify execution and notification delivery need separate confirmation.
- **Docker file backups:** Established nightly rsync of `/opt/appdata` and `/opt/stacks` from participating Docker VMs, distinct from PBS. Versioned off-host recovery and complete Plex appdata backup remain outstanding in Phase 5.

**Do not describe the original PBS NFS/`shfs`/FUSE datastore path as current.** It belongs to the retired VM-based design.

### 2.5 Identity, Ingress, and Applications

- **DNS:** Two AdGuard instances (primary on node 01, secondary on node 03), synchronized by `adguardhome-sync`.
- **Ingress:** Traefik on `docker-prod-01`, Cloudflare DNS-01 wildcard certificates, and Cloudflare Tunnel for approved external applications.
- **Identity:** Authentik is a dedicated VM on node 03; existing OIDC applications and SSO should be checked against their own service documentation.
- **Plex:** Native Unraid Docker container using Intel QuickSync; direct Plex access has a dedicated firewall/forwarding policy.
- **Containers:** Primary media/books/torrent workloads on `docker-prod-01`; Immich and Authentik run in dedicated guest VMs with Komodo Periphery for operations visibility.
- **Monitoring:** Beszel, Uptime Kuma, Healthchecks.io, Komodo, and the new AI Observer integrations complement rather than replace one another.

This roadmap does not maintain a duplicate complete container inventory or credentials list. Consult the service documentation and live tools for current operational details.

---

## SECTION 3 — ROADMAP STATUS AT A GLANCE

| Workstream | Status (November 14, 2026) | Next outcome |
|---|---|---|
| Original Phase 0 — Procurement / physical build | **Historical build completed**, subsequently expanded | Maintain accurate hardware and rack records |
| Original Phase 1 — Networking foundation | **Operational; security verification incomplete** | Validate intended inter-network restrictions and VLAN terminology |
| Original Phase 2 — NAS / storage | **Operational**, later revised | Verify current ZFS jobs and recovery readiness |
| Original Phase 3 — Proxmox foundation | **Operational; superseded two-node plan** | Maintain three-node cluster; review node 01 upgrade readiness |
| Original Phase 4 — Service migration | **Completed in existing phase record**, with later guest moves | Maintain functioning services and independent backups |
| Phase 5 — Hardening and operational readiness | **In progress** | Complete Waves 7–8 and close restore/UPS/security gaps |
| AI Observer foundation (four MCP integrations and Komodo host coverage) | **Implemented and end-to-end verified** | Improve recovery assurance, then extend read-only visibility where justified |
| One-time AI-powered enhancement audit | **Planned; can begin with current access** | Evidence-based findings and prioritized recommendations |
| Observer Phase A — Application-level APIs | **Proposed / not implemented** | Narrow read-only Arr/media API discovery |
| Observer Phase B — System and file inspection | **Proposed / not implemented** | Safe allowlisted configuration/log inspection |
| Observer Phase C — Recurring audits and release intelligence | **Proposed / not configured** | Low-noise daily/weekly reporting after access and scheduling tests |
| Original Phase 6 — Kubernetes sandbox | **Deferred / not started** | Reassess only after operational readiness is established |

These are *roadmap labels*, not certifications that every original phase exit criterion was tested. The original phase documents retain historic work and the authoritative runbooks track outstanding verification.

---

## SECTION 4 — TRACK ONE: OPERATIONAL READINESS AND HARDENING

**Goal:** Make the current infrastructure maintainable, appropriately secured, and recoverable before layering on additional automation.

### 4.1 Phase 5 — Completed Configuration Work

The original Phase 5 record identifies Waves 1–6 as implemented, including:

- Beszel monitoring and Uptime Kuma service checks.
- Nightly Docker host rsync backups and Healthchecks.io heartbeats.
- Authentik/OIDC integrations.
- External access and Cloudflare Access adjustments.
- qBitrr tracker-specific seeding policy configuration.
- Service settings, media automation, and November reliability tuning.

**Verification caveat:** An implemented policy is not the same as end-to-end proof. Some MAM, recovery, and edge-case scenarios remain pending in the detailed Phase 5 document. Its older two-node/PBS VM statements are historical and must not override current backup/network documentation.

### 4.2 Phase 5 Wave 7 — Security and Operational Hardening

| Work item | Current position | Completion evidence required |
|---|---|---|
| SSH administrative access | Historical plan remains open; current all-host key/password policy not fully reverified | Verified keys, fallback/recovery path, staged host-by-host policy decisions |
| Proxmox host NUT clients | Configured on all three nodes | Routine status verified; a planned end-to-end UPS shutdown exercise is separate |
| Guest-level NUT requirement | Original Phase 5 calls for Authentik/Immich guest clients; architecture decision needs review first | Determine whether hypervisor shutdown ordering already meets the need; install only if justified |
| UniFi rules | Live rule inventory inspected | Test Trusted→IoT, Internal→Dmz/IoT→Dmz, and targeted exceptions; propose changes separately |
| Unraid/PBS access | PBS runs on Docker Host networking | Verify TCP 8007 exposure on Services and Management interfaces |
| Legacy v2 systems | Decommission status not independently reverified | Dependency scan, confirmed shutdown/decommission, DNS cleanup |
| Management VLAN description | Historical "VLAN 10" conflicts with UniFi VLAN ID 1 | Verify actual switch-port tagging; correct documentation, not arbitrary live VLANs |
| PVE software versions | Node 01 9.1.6; nodes 02/03 9.2.26 (last verified) | Review node 01 upgrade notes, backup/rollback readiness, then separately authorize upgrade |

**Priority rule:** Investigate read-only first. Change firewall, SSH, identity, or networking configuration only after impact and rollback are understood.

### 4.3 Phase 5 Wave 8 — Recovery Depth and Versioned Backups

| Work item | Current position | Completion evidence required |
|---|---|---|
| Full Proxmox restore exercise | Outstanding | Isolated VM/CT restore with startup and data checks; no conflict with production IDs/IPs |
| PBS maintenance | Schedules documented | Logs show successful recent prune, garbage collection, and verification jobs |
| PBS alerting | Warning/error matcher documented | Verified notification delivery on a safe test event |
| PBS Docker boot dependency | Mount-order behavior not fully tested | Confirm datastore available before service startup |
| AI Observer recovery | Configuration tracked in part in Git; secrets/tunnel runtime files protected separately | Fresh backup covers all four integrations; credential recovery plan; safe restart/reconnect verification |
| Plex full-appdata backup | Not verified implemented; built-in DB backup is not full protection | Consistent full-appdata archive, schedule, monitoring, and test restore |
| Synology ABB versioning | Original Phase 5 Wave 8 item remains open | Verified off-NAS versioned copies, retention, and sample restore |
| ZFS maintenance | Mirror member status healthy; job success not fully rechecked | Current `zpool` health, scrub results, snapshots and retention reviewed |
| Power-loss behavior | NUT clients responding; complete sequencing untested | Reviewed UPS maintenance test, physical outlet mapping, guest-before-host/NAS-last behavior |

An actual restore and recovery test is the exit criterion—not simply a green backup task.

### 4.4 Operational Readiness Exit Criteria

Phase 5 can be marked complete only when the selected scope is both implemented and verified:

- [ ] Critical data protection and sample restoration are demonstrated.
- [ ] Backup schedules, retention jobs, and alert paths have current evidence.
- [ ] Critical VM and NAS recovery dependencies are documented.
- [ ] Firewall intent matches effective policy, or exceptions are explicitly accepted.
- [ ] NUT/UPS operational behavior is verified to the agreed level, with untested areas documented.
- [ ] Remaining Phase 5 Wave 7/8 tasks are completed or consciously deferred with rationale.
- [ ] Related service, architecture, and runbook files reflect confirmed results.

---

## SECTION 5 — TRACK TWO: AI-POWERED ENHANCEMENT AND AUDIT

**Goal:** Turn existing observability into useful, evidence-based infrastructure improvement recommendations while maintaining the current **read-only AI boundary**.

This track has two components that can run in parallel:

1. **One-time comprehensive baseline audit:** Examine what can already be verified, identify problems and opportunities, and establish a prioritized findings register.
2. **Permanent read-only enhancement assistant:** Gradually expand useful visibility and, later, introduce scheduled reports if the access, privacy, and scheduler behavior are proven.

The initial audit **does not need to wait** for new application APIs or a host file collector. Do not install additional agents merely to start it.

### 5.1 Current AI Observer — Implemented Architecture

**VM:** `observer-prod-01` (VM 107, `192.168.30.17`, Services VLAN 30), on `pve-prod-01`. Its integration services live on a dedicated VM, separate from production application Docker workloads.

| Integration | Deployment | Exposed MCP tools | Verification |
|---|---|---|---|
| Proxmox | Dedicated MCP + outbound OpenAI Secure MCP Tunnel | 84 Read / 0 Write | Live cluster/API queries successful |
| Unraid | Dedicated MCP + outbound tunnel | 26 Read / 0 Write | Live NAS/array/Docker queries successful; individual operations may vary |
| Komodo | Dedicated MCP + outbound tunnel; one Core serving three Docker hosts | 40 Read / 0 Write | Live containers/stacks/log queries successful |
| UniFi | Dedicated MCP + outbound tunnel via UniFi cloud API | 55 Read / 0 Write | Live network, firewall, and device queries successful |
| **Total** | **Four independent MCP endpoints and outbound tunnels** | **205 Read / 0 Write** | Tool discovery and representative end-to-end operations verified |

**Komodo host coverage:** `docker-prod-01`, `immich-prod-01`, and `auth-prod-01`. The remote guests run outbound Komodo Periphery; no additional observer MCP tunnel was required for them.

**Important security qualification:** The 205/0 count describes tools exposed to ChatGPT, not the intrinsic rights of every upstream credential. In particular, the UniFi upstream key has not been independently proven read-only. Komodo's local MCP access and remote Periphery Docker permissions also require their existing compensating controls. Tool discovery is not proof every individual read operation works or every response is nonsensitive.

The observer uses localhost-only MCP listeners and outbound OpenAI tunnels. No public MCP reverse-proxy route or inbound port forward is part of the design.

**Authoritative implementation details:** `services/infra/ai-observer.md` and `runbooks/ai-observer-operations.md`.

### 5.2 What the Existing Integrations Can Do Now

| Capability | Current status | Boundary |
|---|---|---|
| Proxmox hardware, node, VM/LXC inventory and health | Available | Only exposed Proxmox read operations |
| Unraid array, drives, parity, capacity and basic Docker status | Available | Some API actions can fail upstream; verify per request |
| Komodo Docker container/stack state across three Docker hosts | Available | Not arbitrary guest shell/file access |
| Selected Docker container logs | Available | Logs do not expose all internal application state |
| Proxmox tasks and selected system logs | Available | Not unrestricted guest journal or file inspection |
| UniFi networks, devices, firewall rules and groups | Available | Configuration inspection, not policy writes or full packet testing |
| CPU/RAM/storage right-sizing recommendations | Available through analysis | Interpret actual utilization; recommendations are not automatic changes |
| Public release notes, advisories and version research | Available when sources are checked | Actual installed version must be grounded separately |
| Arbitrary VM configuration files | Limited | No general filesystem-access capability established |
| Sonarr/Radarr/Prowlarr/Plex/Tautulli internal API state | Limited | Not yet a dedicated approved read-only integration |
| Direct NAS media filesystem inspection | Not connected | Not available through current MCP scope |
| Automatic recurring infrastructure investigations | Not configured | Needs scheduler and data-access validation |
| Executing infrastructure changes | Intentionally disabled | Explicit approval and separate trusted execution would be required |

### 5.3 One-Time Comprehensive Baseline Audit — First Deliverable

**Status:** Planned; can start using current integrations and documentation.  
**Mode:** Read-only inspection and recommendations.  
**Deliverable:** One consolidated, prioritized audit findings report, not a stack of new permanent documentation files.

**Audit sequence:**

1. **Inventory and topology:** Compare current Proxmox, Unraid, Komodo, and UniFi data with authoritative Markdown. Record stale paths, guest placement, version, and role discrepancies.
2. **Compute and resource allocation:** Review CPU/RAM allocations, host pressure, disk utilization, guest startup policy, idle overhead, and memory/swap evidence. Avoid tuning solely on one momentary metric.
3. **Backups and recovery:** Validate job coverage, tasks, datastore availability, retention execution, notifications, recovery gaps, and dependency on the NAS.
4. **Storage and hardware:** Inspect parity and disk health, ZFS evidence where accessible, pool headroom, NAS/network paths, and unusual recurring warnings.
5. **Containers and service operations:** Identify unhealthy containers, restart loops, error patterns, out-of-date deployment assumptions, missing checks, and noisy logs across all three Komodo hosts.
6. **Security and networking:** Review UniFi zone membership, rules and exceptions, sensitive upstream API access, exposure, and privileged container patterns. Do not infer enforcement from a configuration name alone.
7. **Media automation:** Review Docker-visible failures first; distinguish indexer limits and application-specific problems that require later Arr API access.
8. **Enhancement opportunities:** Look for specific new features or sensible configuration improvements relevant to hardware, versions, workloads, and user goals.
9. **Triage and validation:** Classify evidence as verified, likely, historical/resolved, or not observable; request narrowly targeted additional reads only when needed.

**Finding categories:**

- **Fix an existing issue:** An observed problem, such as persistent error patterns, failed imports, unhealthy backups, or recurring connectivity failures.
- **Improve an existing configuration:** A measurable resilience, security, performance, reliability, or maintainability gain.
- **Adopt a genuinely useful new capability:** A release feature or integration with a specific benefit for this exact Homelab configuration.

**Initial investigation candidates (not presumed active):**

- Historical Prowlarr/indexer rate-limit warnings and whether any current searches/imports remain affected.
- Historical Plex/Tautulli connection errors, if still present.
- PBS full-restore and maintenance-verification gaps.
- UniFi Trusted→IoT and Internal→Dmz policy scope.
- AI Observer credential backup and tunnel restart/recovery assurance.
- Recent Proxmox right-sizing decisions and remaining CPU-allocation review.

The audit must **not** re-open resolved qBitrr or parity incidents as ongoing faults without fresh evidence.

**Baseline audit exit criteria:**

- [ ] Current topology and integration reachability captured with timestamps.
- [ ] High-severity failures separated from historical alerts and unknowns.
- [ ] Findings ranked by benefit, risk, effort, and evidence quality.
- [ ] Each recommendation has exact proposed verification and rollback considerations.
- [ ] Nothing changed automatically.
- [ ] Findings point to their authoritative service/runbook documentation for future verified updates.

### 5.4 Enhancement Phase A — Application-Level API Access

**Status:** Proposed / not implemented.  
**Priority:** Highest-value observer extension after establishing the baseline audit.

Docker logs reveal container health and some application errors, but not necessarily the application state. A Sonarr container can be running while its import queue is blocked or series are missing files.

**Candidate read targets:**

| Application | Useful approved reads |
|---|---|
| Sonarr (TV and Anime) | Health warnings, queues, failed imports, activity/history, monitored/missing status, configured indexer status |
| Radarr (1080p and 4K) | Health, queue, failed imports, upgrades, monitored/missing items, indexer status |
| Prowlarr | Indexer health, sync warnings, rate-limit errors, statistics where exposed |
| Bazarr | Subtitle tasks, application health and warnings |
| Plex | Server health, libraries and activity metadata within approved privacy scope |
| Tautulli | Plex-related performance and connection history within approved privacy scope |
| qBittorrent / qBitrr | Queue and seeding-health state through a deliberately restricted read-only adapter |

**Security constraint:** Many application API keys allow both reads and writes. **Do not give raw keys or unrestricted proxy endpoints to ChatGPT.** Build or select an observer-local gateway/adapter that uses an explicit allowlist of safe read operations, blocks unapproved methods and parameters, limits response size, redacts sensitive results, and authenticates clients. Separate upstream credentials should be used where feasible. The adapter must not expose general HTTP requests simply because the user intends read-only use.

**Implementation sequence:**

1. Inventory current endpoints, permissions, and APIs using service documentation.
2. Choose **one pilot application** (Sonarr or Prowlarr is a reasonable starting point).
3. Define exact approved API reads and response fields; identify any read route with side effects.
4. Implement a minimal, authenticated, loopback-bound adapter on `observer-prod-01` or another justified controlled boundary.
5. Test allowed queries, denied writes, secrets redaction, error handling, and real application-state usefulness.
6. Extend to the remaining applications only when the pilot adds clear diagnostic value.

**Phase A exit criteria:** A representative failed-import/indexer/health question can be answered from approved application data, without revealing secrets or enabling arbitrary API operations; successful results and negative permission tests are documented.

### 5.5 Enhancement Phase B — Read-Only System, File, and Log Inspection

**Status:** Proposed / not implemented.  
**Objective:** Investigate approved host/VM configuration and operational evidence that existing MCP tools cannot access.

**Candidate read scope:**

- Systemd service state and selected journal entries.
- Approved Docker Compose configurations after secret redaction.
- Selected application configuration fields.
- Mount status, NFS health, filesystem type and capacity.
- Kernel and storage error summaries.
- Backup script exit status and bounded logs.
- Relevant NAS filesystem metadata or designated approved directories, without general browsing of media content.

**Preferred architecture:** A small host-side collector or existing trustworthy API that returns *predefined, filtered read-only facts*. The collector should have an explicit command/file allowlist, constrained OS identity, bounded output, audit logging, timeouts, and network restrictions. Prefer structured results over raw shell or arbitrary file-read endpoints.

**Must not expose:** Unrestricted root SSH, arbitrary shell commands, unrestricted filesystem traversal, private media contents by default, environment secrets, full Docker inspection output containing credentials, or broad remote Docker socket access.

**Implementation sequence:** Start with one proven operational gap from the baseline audit. Test data minimization and permission denial before deploying on another host. Add new collectors only where the value warrants their attack surface.

**Phase B exit criteria:** A previously inaccessible operational question can be answered accurately from a bounded approved source, with evidence that unapproved commands and paths cannot be accessed.

### 5.6 Enhancement Phase C — Recurring Health Audits and Release Intelligence

**Status:** Proposed / not configured.  
**Prerequisites:** Reliable source access, report thresholds, credential handling, and tested scheduler behavior.

Two proposed recurring outputs:

| Cadence | Report | Intended value |
|---|---|---|
| Daily | **Health audit** | New critical errors, failing backups, unhealthy containers, hardware alerts, significant resource changes, failed imports where app APIs exist |
| Weekly | **Optimization and release review** | Relevant software releases, security fixes, deprecations, configuration opportunities, resource tuning, and recurring unresolved problems |

**Scheduling options to evaluate:**

- ChatGPT scheduled tasks, **only after proving that the required connected-tool access works during scheduled runs**.
- An observer-local scheduler that collects bounded/sanitized facts and delivers them through an approved channel such as existing Discord notification infrastructure.

Neither cadence is active merely because it is listed here. Scheduler choice, delivery channel, timezone, report thresholds, secrets storage, and error handling require a separate implementation decision. Do not create new automations implicitly from this roadmap.

**Release intelligence should be architecture-aware:**

1. Establish the *actually installed* version and its host/container context.
2. Research the new version's release notes and security advisories.
3. Explain only changes relevant to this hardware, workload, and existing configuration.
4. Identify dependencies, compatibility issues, downtime, configuration migrations, and known upgrade risks.
5. Recommend **Update**, **Wait**, or **Investigate**, with an explicit reason.
6. Provide a staged implementation, validation plan, and rollback options if a change is worthwhile.
7. Remember or reference prior findings so reports do not repeatedly recommend the same dismissed item.

**Example of desired behavior:** An Immich release adds a hardware-acceleration capability. The report checks the installed Immich version, VM placement, host hardware, existing ML configuration, and authoritative release notes before deciding whether the feature materially benefits the deployment. It does not simply say "Immich has an update."

**Noise controls:**

- Do not report every image update as urgent.
- Prioritize newly observed changes since the last successful baseline.
- Separate actionable new failures from expected transient warnings.
- Suppress or explain known accepted risks, with a revisit condition.
- Use source timestamps; preserve the last successful run time and indicate stale data.
- Flag inaccessible sources as **unverified**, not healthy.

**Phase C exit criteria:** A real scheduled run successfully reads approved data, produces a useful report without secrets, distinguishes current vs. historical evidence, delivers as configured, and does not perform writes. A second run demonstrates deduplication and sensible thresholds.

### 5.7 Conceptual Target Architecture

```text
ChatGPT — interpretation, evidence review, recommendations
  |
  | OpenAI Secure MCP Tunnels (outbound, dedicated, authenticated)
  v
observer-prod-01 — independent read-only analysis gateway
  |
  +-- IMPLEMENTED: Proxmox MCP -----> Proxmox cluster
  +-- IMPLEMENTED: Unraid MCP ------> NAS/Unraid API
  +-- IMPLEMENTED: Komodo MCP ------> Docker hosts (3)
  +-- IMPLEMENTED: UniFi MCP -------> UniFi cloud / UDM-SE
  |
  +-- PROPOSED Phase A: approved application API adapter
  |      -> Sonarr / Radarr / Prowlarr / Plex / Tautulli / other pilots
  |
  +-- PROPOSED Phase B: bounded host/log/file collectors
  |      -> allowlisted systemd, journal, mounts, backup logs, configs
  |
  +-- PROPOSED Phase C: scheduler + release intelligence
         -> daily health findings / weekly improvements

Recommendations -> user approval -> manual/authorized implementation
  -> live verification -> update existing Obsidian docs
```

The lower three capabilities are **conceptual future work**, not current production MCP features.

### 5.8 Security and Change-Control Contract

**The current AI Observer integrations expose no write tools. Keep that design by default.**

1. Discover a failure, risk, or improvement with read-only evidence.
2. Identify affected infrastructure and review authoritative documentation.
3. Prepare exact proposed changes, commands/configuration, impact, prerequisites, validation, and rollback.
4. Obtain explicit user approval for consequential actions.
5. Apply changes through the established administrator workflow, not an unbounded AI tool.
6. Verify the result through live state or appropriate command output.
7. Update the relevant **existing** Obsidian Markdown files only after verification.
8. Commit through the established user-managed Git workflow.

Creating scripts, Compose files, systemd units, and proposed configuration is permitted as planning work. **Automatically executing or deploying them is not authorized by this roadmap.**

---

## SECTION 6 — RECOMMENDATIONS, AUDIT OUTPUTS, AND PRIORITIZATION

### 6.1 Standard Finding Format

Each material audit finding should contain:

| Field | Required contents |
|---|---|
| Title and category | Existing issue, configuration improvement, or new useful capability |
| Scope | Exact host, VM, service, software version and relevant dependency |
| Evidence | Observed logs/config/metrics with timestamp and source; historical vs current clearly marked |
| Confidence | Confirmed, probable, or needs investigation |
| Severity / benefit | Reliability, security, recovery, performance, operational efficiency, or functionality |
| Recommendation | Specific adjustment or explicit decision to leave unchanged |
| Change risk | Dependencies, downtime, external exposure, secrets/permissions effects |
| Implementation | Exact proposed steps, initially **not executed** |
| Validation | How to confirm the intended result |
| Rollback | Practical way to reverse a consequential change |
| Documentation | Existing service/architecture/runbook paths to update after verification |
| State | New, investigating, accepted, scheduled, verified resolved, or deferred |

### 6.2 Priority Model

| Priority | Examples | Expected handling |
|---|---|---|
| P0 — Immediate | Active data-loss risk, serious exposure, repeated critical backup failure | Investigate promptly; minimize destructive action |
| P1 — High | Unprotected critical data, recurring outages, material configuration risk | Address in the next focused work session |
| P2 — Medium | Verified resource waste, preventable alert noise, useful service improvement | Plan and evaluate benefit/risk |
| P3 — Optional | Cosmetic cleanup or speculative feature with no concrete benefit | Defer without repeated reminders |

A public software release alone is **not** a severity level.

### 6.3 Reporting Quality and History

- Do not claim an issue is fixed without verified post-change behavior.
- Do not claim a failed audit query proves a service is down.
- Avoid copying secrets, private tokens, and unnecessary application-level personal data into reports.
- Cite authoritative service docs, live observations, and external release sources as appropriate.
- Preserve a concise findings register within the chosen audit/report workflow; avoid creating duplicate permanent infrastructure inventory documents.
- Carry forward accepted decisions and mark obsolete findings as superseded.

---

## SECTION 7 — EXECUTION ORDER AND DEPENDENCIES

The two active tracks may progress concurrently, but neither should compromise infrastructure recovery or access boundaries.

| Sequence | Milestone | Dependency / gate | Success criterion |
|---|---|---|---|
| 1 | Complete documentation reconciliation | Existing sources and verified live state | Current topology accurate; obsolete PBS/two-node references removed from active docs |
| 2 | Perform one-time AI baseline audit | Four existing read-only integrations | Actionable, evidence-ranked findings without installing new agents |
| 3 | Close critical recovery and security gaps | Scoped Phase 5 work | Restore test, backup maintenance evidence, and agreed firewall priorities |
| 4 | Review `pve-prod-01` upgrade readiness | Backups, maintenance window, release notes | Approved, validated upgrade plan before execution |
| 5 | Pilot one application API adapter | Baseline shows meaningful missing application data | Narrow approved reads; verified write denial; useful diagnostics |
| 6 | Pilot one system/file collector if justified | Specific unanswered audit question | Allowlisted reads, safe data handling, no shell delegation |
| 7 | Pilot recurring health and release reporting | Approved sources, tested scheduling, alert thresholds | A successful report and a follow-up run without noise or data leaks |
| 8 | Reassess Kubernetes sandbox | Operational foundation stable; user wants to proceed | Separate lab plan that cannot impact production |

**No invented deadlines.** The milestone order is a priority and dependency guide, not a calendar commitment.

### 7.1 Near-Term Focus

The best immediate use of the existing integrations is **a baseline audit**, alongside completing the current documentation and selecting the next Phase 5 verification item. The baseline should tell us *which* additional API or file collector is truly worth deploying. Do not build collectors, schedulers, or extensive upgrade automation solely because they are possible.

### 7.2 Decisions Needed During Implementation

- Which one or two application APIs deliver the most valuable missing data?
- Which endpoints can be safely exposed as read-only facts despite upstream tokens that may have write privileges?
- Which filesystem/log questions are not answered by existing tools?
- Which findings warrant ongoing alerts versus a weekly report?
- Which scheduling and delivery mechanism can be reliably tested in this environment?
- What evidence is sufficient to close recovery gaps without unnecessary disruptive testing?

These decisions should be made when their corresponding implementation step is reached, using live evidence and explicit user authorization.

---

## SECTION 8 — FUTURE PHASE: KUBERNETES SANDBOX

**Original Phase 6:** Kubernetes with k3s/Talos, sandbox first.  
**Current status:** Deferred / not implemented.

The core original principle remains sound: **do not migrate production services into a Kubernetes environment until a separate learning cluster has been deliberately built, broken, restored, secured, and understood**.

The original two-node Talos layout is **historical**, because the cluster now has three heterogeneous Proxmox nodes. Reassess capacity, placement, networking, storage, and the actual learning objective when this phase is activated.

**Potential lab scope:**

- Lightweight VMs or other appropriately isolated lab guests.
- Kubernetes fundamentals: pods, deployments, namespaces, services, ingress, and storage.
- Helm and reproducible cluster configuration.
- GitOps using Flux or Argo CD if justified.
- Certificate management, upgrades, failures, and recovery exercises.
- Optional Talos, k3s, Longhorn or alternatives only after their trade-offs are understood.

**Entry criteria:**

- Phase 5's critical recovery/security gaps are resolved or explicitly accepted.
- Existing production guest resource commitments are understood.
- The lab cannot interrupt DNS, SSO, media, storage, backups, or the AI Observer.
- There is a clear learning/functional goal rather than a requirement to migrate everything.

**Workloads not automatically scheduled for migration:** ARR stack, qBittorrent/Gluetun, and books hardlink/ingest workflows retain their established Docker design unless a compelling, tested alternative exists. Production Immich, Authentik, monitoring, and ingress are **candidates to evaluate**, not promises to migrate.

---

## SECTION 9 — DOCUMENTATION, REPOSITORIES, AND DECISION HISTORY

### 9.1 Documentation System

The Obsidian vault is the authoring source of truth and is mirrored to a private `homelab-docs` GitHub repository and Google Drive. The Drive mirror is the primary reference when consulting current documentation from ChatGPT.

**Authoritative paths:**

| Subject | Reference |
|---|---|
| Architecture rationale/history | `architecture/decisions-log.md` |
| Hardware and rack | `architecture/hardware/inventory.md`, `architecture/hardware/rack-layout.md` |
| Proxmox nodes | `architecture/compute/pve-prod-01.md`, `pve-prod-02.md`, `pve-prod-03.md` in the same directory |
| VLAN/IP/firewall | `architecture/network/vlan-design.md`, `ip-addresses.md`, `firewall-rules.md` |
| NAS and storage | `architecture/storage/nas-prod-01.md` |
| AI Observer architecture | `services/infra/ai-observer.md` |
| AI Observer operations | `runbooks/ai-observer-operations.md` |
| PBS backups and recovery | `runbooks/backup-schedule.md`, `runbooks/backup-restore.md` |
| Existing Phase 5 checklist | `phases/phase-5-hardening.md` |
| Original build history | Other existing `phases/` documents |
| Service-specific configuration | Corresponding Markdown files in `services/` |

`01-context-handoff.md` has been intentionally deleted because it duplicated current information and repeatedly became stale. **Do not recreate a permanent handoff file.** When changing conversations, produce a temporary on-demand handoff prompt and consult the actual documentation again.

This roadmap should not become a second copy of full service configuration, passwords, port lists, or the entirety of the decision log.

### 9.2 Infrastructure Repository

The private `homelab-infra` repository manages Compose deployments and non-secret configuration. Its working tree is used on participating Docker hosts, including the AI Observer.

Typical layout:

```text
/opt/stacks/<stack>/compose.yaml   # Version-controlled deployment
/opt/stacks/<stack>/.env           # Sensitive local environment, gitignored
/opt/appdata/<app>/                # Application state; backed up separately
```

The AI Observer's four MCP deployment definitions and sanitized tunnel/systemd templates have been committed to the infrastructure repository. Runtime keys, active tunnel identities, protected `.env` files, and necessary host state are not guaranteed recoverable from Git alone.

Maintain the existing separation between source-controlled configuration and secrets. Consult `runbooks/git-workflow.md` and `services/infra/ai-observer.md` before altering those workflows.

### 9.3 Naming Conventions

Use `{role}-{env}-{index}` for host and guest names: e.g., `pve-prod-01`, `nas-prod-01`, `observer-prod-01`. Docker instances use role-specific names such as `sonarr-tv`, `sonarr-anime`, `radarr-1080p`, and `radarr-4k`.

The PBS Proxmox storage ID `pbs-prod-01` is retained for compatibility even though the old PBS VM was deleted; it is **not** evidence that a VM by that name remains active.

### 9.4 Documentation Change Procedure

1. Read every affected existing Markdown document before proposing its replacement.
2. Verify current configuration through live integrations or operator command output.
3. Distinguish completed work from planned work, hypotheses, and historical incidents.
4. Preserve valuable decision and troubleshooting history.
5. Provide complete updated Markdown files for the operator to apply in Obsidian.
6. Do not directly edit Google Drive or GitHub without an explicit request.
7. Do not mark an implementation complete until the expected behavior was verified.

---

## SECTION 10 — ROADMAP CHANGE RECORD

| Date | Change | Interpretation |
|---|---|---|
| Original v7.0 plan | Clean-slate two-node Proxmox, QDevice, PBS VM, VLAN 10/20/30/40, services migration, Phase 5 hardening, future Kubernetes | **Historical design**, superseded in several key areas |
| November 3–5, 2026 | PBS moved from VM 103 to Docker on Unraid using direct ZFS | **Implemented**; old VM later deleted |
| November 2026 | Third Proxmox node added; Raspberry Pi QDevice removed | **Implemented** three-vote quorum; HA remains disabled |
| November 13, 2026 | Old node 02 replaced by Dell OptiPlex 5000 SFF; Immich on new node | **Implemented and verified** |
| November 14, 2026 | Authentik moved to Lenovo node 03; updated backup and rack verification | **Implemented and verified** |
| November 2026 | Four MCP integrations plus Komodo coverage of three Docker hosts | **Implemented and end-to-end verified** at the defined read-only tool boundary |
| November 14, 2026 | Unified roadmap revised for a one-time AI enhancement audit and future observer phases A/B/C | **Roadmap updated; new audit/extension phases are not yet implemented** |

**Roadmap maintenance rule:** Update this file when project goals, milestone status, or architectural direction changes. Record detailed implementation procedures and verified service settings in their existing dedicated documents. The roadmap is a plan, not proof that a future capability is already deployed.

---

_Homelab v3.0 Master Roadmap | v8.0 | Current plan, verified architecture, and AI-assisted evolution | 2026-11-14_
