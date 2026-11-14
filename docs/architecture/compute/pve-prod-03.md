# pve-prod-03 — Third Proxmox Node

**Role:** Third compute and quorum member; hosts secondary AdGuard DNS (`dns-prod-02`, CT 102) and Authentik (`auth-prod-01`, VM 104)  
**Status:** Active — added to Homelab v3 in November 2026  
**Last Updated:** 2026-11-14

---

## Hardware

| Component        | Detail                                                                                                                       |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| **Model**        | Lenovo ThinkCentre M90q Tiny                                                                                                 |
| **CPU**          | Intel Core i7-10700 (8 cores / 16 threads)                                                                                   |
| **RAM**          | 24 GB DDR4 SO-DIMM (16 GB + 8 GB); ~23.2 GiB usable in Proxmox                                                               |
| **Boot Drive 1** | Toshiba KXG60ZNV512G NVMe 512 GB (`/dev/nvme0n1`)                                                                            |
| **Boot Drive 2** | SK hynix HFS512GDE9X081N NVMe 512 GB (`/dev/nvme1n1`)                                                                        |
| **Boot Storage** | ZFS RAID-1 mirror (`rpool`) with `rpool/data` for guest disks                                                                |
| **Networking**   | Built-in 1 GbE RJ45, USW-Pro-Max-24 port 16 (last documented connection; reverify after rack relocation)                     |
| **Firmware**     | Lenovo M2WKT61A (last observed November 2026); newer M2WKT65A identified for review, installation not independently verified |

**Disk health verified November 13, 2026:** Both NVMe drives reported SMART `PASSED`. Toshiba showed 28% reported endurance used; SK hynix 0%. These values are point-in-time observations.

**November 14 storage verification:** The ZFS mirror remained `ONLINE`, with no known data errors and zero read/write/checksum errors on either member.

### Physical Maintenance — November 14, 2026

The Lenovo was shut down, physically cleaned, reassembled, booted, and placed in the rack on a shelf.

The hardware maintenance was followed by live Proxmox health checks confirming:

- Node online and participating in cluster quorum.
- Proxmox services operational.
- `rpool` ONLINE with no known data errors.
- Secondary AdGuard CT 102 running.
- Shared NFS ISO and PBS storage available.
- NUT client functional after the reboot.

The exact rack-unit placement and current switch-port connection have not been independently documented following the physical relocation.

**Firmware verification remains open.** The earlier documented BIOS was M2WKT61A. Although a newer firmware package was reviewed before maintenance, successful installation of M2WKT65A has not been confirmed by a post-maintenance BIOS version readback.

---

## Software

| Field | Value |
|---|---|
| **OS** | Proxmox VE 9.2.26 (`pve-manager`) |
| **Kernel** | 7.0.14-28-pve |
| **Debian base** | Trixie (Debian 13) |
| **Boot** | UEFI, Secure Boot disabled |
| **APT** | Debian Trixie and Proxmox no-subscription repositories; enterprise repositories disabled (previously verified) |

**November 14 verification:** The live Proxmox integration confirmed `pve-manager/9.2.26`, running kernel `7.0.14-28-pve`, UEFI boot, and Secure Boot disabled.

---

## Network

| Field | Value |
|---|---|
| **Hostname (FQDN)** | `pve-prod-03.local` |
| **IP** | `192.168.10.13/24` |
| **VLAN** | 10 — Management, untagged on bridge |
| **Gateway** | `192.168.10.1` |
| **DNS** | `192.168.10.1`, search domain `local` (previously live-verified) |
| **Physical NIC** | `nic0`, 1 GbE |
| **Bridge** | `vmbr0`, VLAN-aware, allowed VLAN IDs `2-4094` |

### Network Interfaces

| Interface | Type | Role |
|---|---|---|
| `nic0` | Physical Ethernet | Management and guest bridge uplink |
| `vmbr0` | Linux bridge | Static host management `192.168.10.13/24`; tagged guest VLAN traffic |

The bridge was live-verified active following the November 14 maintenance.

Current guests on Services VLAN 30:

| Guest | VMID | IP | Purpose |
|---|---|---|---|
| dns-prod-02 | 102 | 192.168.30.15 | Secondary AdGuard Home |
| auth-prod-01 | 104 | 192.168.30.13 | Authentik identity provider |

The Authentik migration preserved its original VLAN tag, virtual MAC address, and guest IP address. No additional VLAN or routing configuration was required on the Lenovo.

---

## Proxmox Configuration

### Cluster

- Member of `homelab-v3`, node ID `3`, Corosync address `192.168.10.13`.
- Joined as the third node after removal of the former Raspberry Pi QDevice.
- Quorum now uses three Proxmox voting members (`pve-prod-01`, `pve-prod-02`, `pve-prod-03`); no QDevice is configured.
- Two of the three node votes are required for quorum.
- No Proxmox HA policies are enabled. Cluster quorum does not automatically restart a guest on another host.

**November 14 verification:**

The live Homelab Proxmox integration confirmed all three nodes online and the cluster quorate, with configuration version `7`.

| Node | Management IP | Votes | State |
|---|---|---|---|
| pve-prod-01 | 192.168.10.11 | 1 | Online |
| pve-prod-02 | 192.168.10.12 | 1 | Online |
| pve-prod-03 | 192.168.10.13 | 1 | Online |

The Lenovo now hosts both CT 102 and VM 104.

This is a three-node compute cluster, not an automatic guest failover solution. The guests use node-local storage, and recovery after a host failure requires an appropriate manual migration, restoration, or recovery procedure.

### Storage

| Storage | Type | Notes |
|---|---|---|
| `local` | Directory | ISO/template and other configured directory content |
| `local-zfs` | ZFS | Local `rpool/data`, ZFS mirror; same storage ID on three nodes but physically separate disks |
| `nas-isos-p2` | NFS | Unraid `192.168.30.16:/mnt/user/isos`, shared ISO library, assigned to `pve-prod-02` and `pve-prod-03` |
| `pbs-prod-01` | PBS | Docker PBS on `nas-prod-01` at `192.168.30.16:8007`, datastore `nas-backups` |

`nas-isos` on `pve-prod-01` uses its direct DAC path (`10.0.0.2`) to the same Unraid ISO export. Nodes 02/03 deliberately use `nas-isos-p2` via the Services VLAN instead.

**November 14 live storage verification:**

- `rpool` state: `ONLINE`.
- Both NVMe mirror members: `ONLINE`.
- Read errors: 0.
- Write errors: 0.
- Checksum errors: 0.
- Known data errors: None.
- `local-zfs` available.
- `nas-isos-p2` available.
- `pbs-prod-01` available.
- Local ZFS usage: approximately 32.9 GiB of 454 GiB (7.3%).

The observed storage usage reflects the addition of VM 104's local ZFS virtual disks. Storage usage may change as guests write data.

**ZFS maintenance:** The default ZFS cron schedule previously verified on this host trims on the first Sunday and scrubs on the second Sunday of each month at 00:24. `rpool` autotrim was `off`. No changes were required. A successful maintenance execution on the new install has not been specifically verified.

**Redundant boot:** `rpool` mirrors both NVMe disks. Independent EFI-boot failover on this host has not been documented as physically tested; do not equate an ONLINE mirror with a confirmed single-disk boot test.

### UPS Monitoring — NUT Client

**Implemented and operationally verified November 13, 2026; rechecked after maintenance November 14.**

This node runs Network UPS Tools as a network client to the UPS server on `nas-prod-01`. The UPS is connected to Unraid via USB; no UPS driver or NUT server is configured locally on the Lenovo.

| Setting | Value |
|---|---|
| **Package** | `nut-client` 2.8.1-5 |
| **Operating mode** | `netclient` |
| **NUT server** | `192.168.10.10` (`nas-prod-01`) |
| **UPS identifier** | `ups` |
| **MONITOR target** | `ups@192.168.10.10` |
| **MONITOR role** | `slave` / secondary |
| **Power value** | `1` |
| **MINSUPPLIES** | `1` |
| **SHUTDOWNCMD** | `/sbin/shutdown -h +0` |
| **POLLFREQ** | `5` seconds |
| **HOSTSYNC** | `15` seconds |
| **DEADTIME** | `15` seconds |
| **FINALDELAY** | `5` seconds |
| **POWERDOWNFLAG** | `/etc/killpower` |
| **Systemd service** | `nut-monitor.service`, enabled and active |

**Configuration files:**

- `/etc/nut/nut.conf`: `MODE=netclient`.
- `/etc/nut/upsmon.conf`: UPS monitoring and shutdown settings; contains protected monitoring credentials.
- `/etc/systemd/system/nut-monitor.service.d/override.conf`: local systemd compatibility correction.

The active configuration files were installed as `root:nut`, mode `0640`. Credentials were transferred directly from the working `pve-prod-01` configuration over SSH and are not recorded in Markdown.

**Systemd packaging workaround:**

The installed Debian NUT service referenced `/usr/lib/tmpfiles.d/nut-common-tmpfiles.conf`, which was absent. The actual installed tmpfiles definition was `/usr/lib/tmpfiles.d/nut-client.conf`, containing the `/run/nut` directory definition.

The following drop-in corrects the service's `ExecStartPre` command while leaving the vendor service file unchanged:

```ini
[Service]
ExecStartPre=
ExecStartPre=/usr/bin/systemd-tmpfiles --create /usr/lib/tmpfiles.d/nut-client.conf
```

`systemctl daemon-reload` and a monitoring-service restart were completed. The corrected pre-start step exited successfully (`0/SUCCESS`). Review the necessity of this override after future NUT package upgrades.

**November 13 verification:**

- Installed `nut-client` 2.8.1-5 from Debian Trixie.
- Queried the remote UPS; `upsc ups@192.168.10.10 ups.status` returned `OL`.
- Confirmed `/run/nut` exists with ownership `root:nut` and permissions `0770`.
- Verified the staged configuration matched the established primary-node settings.
- Restarted `nut-monitor.service` successfully; service status reported `active (running)` and enabled.
- Confirmed the service recognized `ups@192.168.10.10` as a secondary monitor, power value `1`.
- Reviewed the journal without finding access-denied or UPS communication-loss events after startup.
- Removed the temporary credential-bearing configuration copies under `/root/nut-reference/` after installation.

**November 14 post-maintenance verification:**

The operator executed the following commands on `pve-prod-03` after the node had rebooted and before the Authentik migration was considered complete:

```bash
systemctl is-enabled nut-monitor
systemctl is-active nut-monitor
upsc ups@192.168.10.10 ups.status
```

Results:

```text
enabled
active
Init SSL without certificate database
OL
```

These results confirm that the monitoring service remained enabled and active after the maintenance reboot and could successfully query the UPS server.

**Nonfatal startup messages:** An initial check for `/run/nut/upsmon.pid` reported that no previous PID file existed. NUT also logged `Init SSL without certificate database` and systemd notification messages from `upsnotify`. These did not prevent the monitor from starting. No additional suppression or notification changes were made.

**Validation boundary:** Healthy UPS queries and the active service are verified, but a real UPS-triggered shutdown has not been performed. Guest shutdown order, server-to-client forced-shutdown signaling, and restart behavior following restoration of utility power remain untested.

**Useful read-only checks:**

```bash
upsc ups@192.168.10.10 ups.status
systemctl is-enabled nut-monitor
systemctl is-active nut-monitor
systemctl status nut-monitor --no-pager -l
journalctl -u nut-monitor --no-pager -n 50
```

Avoid `upsmon -c fsd` during routine troubleshooting; it is a forced-shutdown command.

See `architecture/storage/nas-prod-01.md` for the documented NUT server and UPS-side shutdown policy.

### Backup

The cluster job `backup-54015f90-26e5` schedules daily 02:00 snapshot backups to `pbs-prod-01` for all current guests, retaining an obsolete exclusion for removed VM 103.

**November 13 scheduled backup:**

- Node `pve-prod-03` scheduled `vzdump` completed `TASK OK`.
- Its log included CT 102 and, at that time, VM 105.
- VM 105 was moved to the replacement `pve-prod-02` later on November 13.
- A full isolated CT restore has not been specifically verified for CT 102.

**November 14 — Authentik backup after migration:**

VM 104 was backed up by the scheduled cluster job on `pve-prod-01` early on November 14, before its migration.

After VM 104 was successfully migrated and started on `pve-prod-03`, the operator initiated a fresh snapshot backup from the Lenovo:

```bash
vzdump 104 --storage pbs-prod-01 --mode snapshot
```

**Result:** Backup job finished successfully.

| Metric | Result |
|---|---|
| Guest | `auth-prod-01` (VM 104) |
| Source | `pve-prod-03` |
| Destination | `pbs-prod-01` / `nas-backups` |
| Mode | Snapshot |
| Completion | November 14, 2026, 14:09:45 CST |
| Disk size processed | 32.00 GiB |
| Sparse zero data | 11.59 GiB (36%) |
| Backup data reused | 30.52 GiB (95%) |
| Duration | 44 seconds |
| Reported throughput | 744.7 MiB/s |
| Job status | Successful |

The message `notification was not matched by any matcher` indicates the event did not match a configured notification rule; it did not indicate a failed backup.

**Current backup placement:**

| VMID | Guest | Host | Scheduled PBS coverage |
|---|---|---|---|
| 102 | dns-prod-02 | pve-prod-03 | Included |
| 104 | auth-prod-01 | pve-prod-03 | Included; fresh manual backup verified November 14 |

The cluster-wide backup job follows the current guest placement. Future scheduled backups of VM 104 should execute from node 03 rather than node 01.

**Remaining validation:**

- Verify future scheduled backup executions for both guests on node 03.
- Perform an isolated restore test for CT 102.
- Perform an isolated restore test for VM 104.
- Continue monitoring PBS datastore verification, prune, and garbage-collection tasks.
- Confirm actual backup warning/error notification delivery when practical.

A successful snapshot backup establishes that the backup operation completed; it does not independently establish full guest recovery.

See `runbooks/backup-schedule.md` and `runbooks/backup-restore.md`.

---

## Guests

The current guest inventory was verified through the live Homelab Proxmox integration on November 14, 2026.

| ID | Name | Type | VLAN | IP | Status |
|---|---|---|---|---|---|
| 102 | `dns-prod-02` | LXC (Debian 12) | 30 | `192.168.30.15` | Running; 1 vCPU, 1 GiB RAM, 4 GiB rootfs |
| 104 | `auth-prod-01` | VM (Ubuntu 24.04) | 30 | `192.168.30.13` | Running; 2 vCPU, 4 GiB RAM, 32 GiB virtual disk |

**Current guest count:** Two running guests — one LXC and one VM.

### dns-prod-02 — Secondary AdGuard

CT 102 is the **secondary AdGuard Home** instance.

The primary DNS instance, CT 100 (`dns-prod-01`), runs on `pve-prod-01`.

- Primary DNS: `192.168.30.10`.
- Secondary DNS: `192.168.30.15`.
- Secondary DNS Proxmox host: `pve-prod-03`.
- Configuration replication: `adguardhome-sync`.

Application synchronization was verified following the original relocation. Use both instances for DNS resilience rather than co-locating primary and secondary DNS on one Proxmox host.

The November 14 host maintenance and reboot completed without requiring CT 102 to be restored or re-created. The container was confirmed running afterward.

See `services/infra/adguard.md`.

### auth-prod-01 — Authentik Identity Provider

VM 104 hosts the Homelab v3 Authentik identity provider.

It was migrated from `pve-prod-01` to `pve-prod-03` on November 14, 2026.

**Current VM configuration:**

| Setting | Value |
|---|---|
| VMID | 104 |
| Guest hostname | auth-prod-01 |
| Operating system | Ubuntu Server 24.04 LTS |
| Proxmox host | pve-prod-03 |
| vCPU | 2 |
| CPU type | x86-64-v2-AES |
| RAM | 4 GiB |
| Memory ballooning | Disabled |
| Primary disk | 32 GiB on `local-zfs` |
| UEFI disk | On `local-zfs` |
| Machine | q35 |
| Firmware | OVMF / UEFI |
| SCSI controller | VirtIO SCSI |
| QEMU guest agent | Enabled |
| Network bridge | vmbr0 |
| VLAN tag | 30 |
| IP | 192.168.30.13/24 |
| MAC address | BC:24:11:CA:5D:1B |
| Start at boot | Enabled |
| Startup configuration | `order=4,up=60` |

**Application services:**

Authentik runs as a Docker Compose deployment on its dedicated VM.

The following six containers were verified running after migration:

- `authentik-server`
- `authentik-worker`
- `authentik-postgres`
- `authentik-redis`
- `beszel-agent`
- `komodo-periphery`

The Authentik server, worker, and PostgreSQL container health checks passed. The local application health endpoint returned HTTP 200.

The Komodo server connection returned `Ok`.

The migration preserved the original guest IP, VLAN, and MAC address. Existing Traefik routing and AdGuard DNS rewrites remained functional.

**Post-migration verification:**

| Test | Result |
|---|---|
| VM started on pve-prod-03 | Passed |
| Original guest IP preserved | Passed |
| Default gateway 192.168.30.1 | Passed |
| Guest RAM total | 3.8 GiB |
| Guest RAM used | 1.3 GiB |
| Guest RAM available | 2.4 GiB |
| Guest swap occupied | 0 B |
| Authentik local health endpoint | HTTP 200 |
| Traefik backend check | HTTP 302 |
| Public URL check | HTTP 302 |
| DNS rewrite | auth.giohosted.com → 192.168.30.11 |
| Fresh SSO test | Passed, operator-confirmed |
| Komodo connection | Ok |
| Fresh PBS backup | Passed |

The HTTP 302 responses are expected application redirects, not evidence of failed health checks.

**Availability considerations:**

- Authentik is no longer dependent on `pve-prod-01` remaining online.
- Planned maintenance and reboots of the primary Proxmox node no longer directly stop the Authentik VM.
- Authentik is still a single identity-provider VM, with no automatic failover configured.
- The VM's local ZFS disks do not automatically replicate between Proxmox nodes.
- A node 03 failure would affect both secondary DNS and Authentik.
- PBS backups provide a recovery path but do not replace HA or independently validated restore procedures.

See `services/infra/authentik.md` for authoritative application configuration, secrets-handling requirements, container details, SSO providers, and the complete migration verification history.

### Previous Temporary Workload — immich-prod-01

VM 105 (`immich-prod-01`) was migrated here from the original `pve-prod-02` to vacate that node for replacement.

After the new Dell OptiPlex 5000 joined the cluster, Immich was shut down and migrated offline from `pve-prod-03` to the replacement `pve-prod-02` on November 13, 2026.

VM 105 is **not currently hosted on node 03**.

---

## Design Decisions

### Third Voting Node, No QDevice

A third compute node allows normal 2-of-3 cluster quorum. The former Raspberry Pi QDevice was removed prior to onboarding the Lenovo.

This is not equivalent to guest HA: the cluster still requires explicit recovery or migration procedures after a host failure.

### Mirrored NVMe Boot

Two reused 512 GB NVMe disks were configured as a ZFS RAID-1 mirror during installation, matching the preferred local boot redundancy of the other current compute nodes.

PBS is still required for recovery from non-disk failures and user/data corruption.

### Secondary DNS Placement

Placing CT 102 on node 03 separates it from primary AdGuard CT 100 on node 01.

This provides DNS service distribution across independent Proxmox hosts, rather than concentrating both DNS instances on the same hypervisor.

### Authentik Placement — November 14, 2026

**Decision (implemented):** Relocate VM 104 (`auth-prod-01`) from `pve-prod-01` to `pve-prod-03` using a controlled offline Proxmox migration.

**Rationale:**

- Reduce the number of critical services hosted on `pve-prod-01`.
- Allow maintenance and Proxmox updates on the primary node without directly shutting down Authentik.
- Use the available compute and memory capacity on the Lenovo.
- Retain Authentik as a dedicated VM, preserving its existing deployment and configuration.
- Avoid live CPU-state migration concerns between AMD and Intel hardware.

**Tradeoffs:**

- Authentik and secondary DNS now share a hypervisor.
- Loss of node 03 would affect both services.
- The primary DNS service remains on node 01, maintaining DNS distribution.
- There is no automatic VM recovery or shared guest-storage failover.

**Implementation:** Completed November 14, 2026. The VM booted successfully, application-level validation passed, and a fresh PBS snapshot backup completed successfully.

### Workload Distribution

As of November 14, 2026, the primary compute placement is:

| Node | Primary workloads |
|---|---|
| pve-prod-01 | Primary AdGuard, primary Docker host, AI Observer, stopped Dragonwilds VM |
| pve-prod-02 | Immich |
| pve-prod-03 | Secondary AdGuard, Authentik |

This distribution supports administrative flexibility and planned maintenance but does not constitute an HA deployment.

---

## Installation and Operational Notes

### Deployment — November 2026

- Added Lenovo ThinkCentre M90q to `homelab-v3` after preparing a mirrored ZFS installation and removing the earlier two-node QDevice.
- Configured `vmbr0` as VLAN-aware and connected it to USW-Pro-Max-24 (port 16 in the last verified switch placement).
- Relocated CT 102 here from the original `pve-prod-02`; performed a stop/backup/restore migration because the source used LVM-thin and the destination uses ZFS. Verified DNS function and AdGuard synchronization afterward.
- Temporarily hosted Immich VM 105 until completion of the Dell OptiPlex 5000 replacement on November 13.
- The scheduled November 13 02:00 PBS job completed successfully for CT 102 and VM 105 while both were still on node 03.
- On November 13, installed and configured `nut-client` 2.8.1-5 using the established `pve-prod-01` network-client configuration. Applied a systemd tmpfiles drop-in, verified UPS `OL` status and an active monitoring service, and removed temporary copies containing monitoring credentials.

### Physical Maintenance — November 14, 2026

- Shut down the Lenovo.
- Performed physical cleaning.
- Reassembled and booted the system.
- Placed the system on a rack shelf.
- Verified the Proxmox node returned online.
- Confirmed the ZFS mirror was healthy.
- Confirmed CT 102 was running.
- Verified NUT monitoring remained active and responsive.

The exact rack location and current switch-port assignment remain to be reverified and reflected in `architecture/hardware/rack-layout.md`.

The BIOS version has not been independently read back after this maintenance.

### Authentik Migration — November 14, 2026

**Source:** pve-prod-01  
**Destination:** pve-prod-03  
**Guest:** VM 104 (`auth-prod-01`)

The Authentik VM was gracefully shut down on the source node.

The following offline migration command was executed on `pve-prod-01`:

```bash
qm migrate 104 pve-prod-03 --with-local-disks --targetstorage local-zfs --online 0
```

The migration completed successfully.

The VM was confirmed registered on the Lenovo with its existing virtual disks, resource allocations, VLAN 30 network configuration, and MAC address preserved.

The operator started the VM on `pve-prod-03` and completed post-migration validation.

**Verified results:**

- VM running on destination.
- Guest IP and routing correct.
- Guest memory healthy, with 2.4 GiB available and zero swap occupied.
- All six Authentik-host containers running.
- Authentik local health endpoint HTTP 200.
- Existing reverse-proxy and DNS routing functioning.
- SSO authentication functioning.
- Komodo connection healthy.
- Destination NUT client enabled and active.
- ZFS mirror ONLINE.
- Fresh PBS snapshot backup completed successfully.

No changes to Authentik database contents, credentials, providers, Traefik routing, or DNS rewrites were required for the VM move.

### Previous Read-only Audit

- ZFS mirror ONLINE; NVMe SMART tests showed no media/data errors. Toshiba endurance usage was already 28%, so monitor wear over time without assuming failure is imminent.
- `pve-manager` 9.2.26 and kernel `7.0.14-28-pve` verified November 13 and rechecked November 14.
- Uptime Kuma ping monitoring for this host was previously added; monitoring delivery/history should be checked separately if host availability changes.
- The previously observed BIOS M2WKT61A is older than a later firmware release identified for review. Updating BIOS was initially deferred while Immich was temporarily resident. Firmware update completion has not subsequently been verified.
- Beszel agent deployment on the Proxmox host was deferred; do not mark it implemented merely because older monitoring documentation references Beszel.

### Open Validation Items

- Confirm the currently installed Lenovo BIOS version after the physical maintenance.
- Independently test boot from each SSD if redundant-boot recovery assurance is required.
- Re-check the physical switch port and exact rack-unit placement.
- Verify an isolated restore of CT 102 as part of the backup recovery test plan.
- Verify an isolated restore of VM 104.
- Confirm scheduled backups continue to run successfully for both node 03 guests.
- Plan and safely validate UPS-triggered guest and host shutdown, including recovery after power restoration; do not simulate a forced shutdown on the production UPS during routine service checks.
- Continue monitoring host memory, CPU load, and storage usage with both DNS and Authentik running.
- Reevaluate VM placement if future workloads or availability requirements justify it.

---

## Related Documentation

- `architecture/compute/pve-prod-01.md` — Primary Proxmox node and former Authentik host
- `architecture/compute/pve-prod-02.md` — Replacement Dell Proxmox node and Immich host
- `architecture/hardware/inventory.md` — Hardware inventory and workload placement
- `architecture/hardware/rack-layout.md` — Physical rack installation and network connections
- `architecture/decisions-log.md` — Cluster, quorum, and workload-placement decisions
- `services/infra/authentik.md` — Authentik service configuration and migration history
- `services/infra/adguard.md` — Primary and secondary DNS architecture
- `architecture/storage/nas-prod-01.md` — NAS storage and NUT server
- `runbooks/backup-schedule.md` — Backup scheduling and verified coverage
- `runbooks/backup-restore.md` — Recovery procedures
