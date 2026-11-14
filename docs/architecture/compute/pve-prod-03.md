# pve-prod-03 — Third Proxmox Node

**Role:** Third compute and quorum member; hosts secondary AdGuard DNS (`dns-prod-02`, CT 102)  
**Status:** Active — added to Homelab v3 in November 2026  
**Last Updated:** 2026-11-13

---

## Hardware

|Component|Detail|
|---|---|
|**Model**|Lenovo ThinkCentre M90q Tiny|
|**CPU**|Intel Core i7-10700 (8 cores / 16 threads)|
|**RAM**|24 GB DDR4 SO-DIMM (16 GB + 8 GB); ~23.2 GiB usable in Proxmox|
|**Boot Drive 1**|Toshiba KXG60ZNV512G NVMe 512 GB (`/dev/nvme0n1`)|
|**Boot Drive 2**|SK hynix HFS512GDE9X081N NVMe 512 GB (`/dev/nvme1n1`)|
|**Boot Storage**|ZFS RAID-1 mirror (`rpool`) with `rpool/data` for guest disks|
|**Networking**|Built-in 1 GbE RJ45, USW-Pro-Max-24 port 16 (last documented connection)|
|**Firmware**|Lenovo M2WKT61A (last observed November 2026); newer M2WKT65A was identified for later review, **not verified installed**|

**Disk health verified November 13, 2026:** Both NVMe drives reported SMART `PASSED`. Toshiba showed 28% reported endurance used; SK hynix 0%. The ZFS mirror was `ONLINE` with no known data errors and zero read/write/checksum errors. These values are point-in-time observations.

---

## Software

|Field|Value|
|---|---|
|**OS**|Proxmox VE 9.2.26 (`pve-manager`), running kernel `7.0.14-28-pve`|
|**Debian base**|Trixie (Debian 13)|
|**Boot**|UEFI, Secure Boot disabled|
|**APT**|Debian Trixie and Proxmox no-subscription repositories; enterprise repositories disabled (previously verified)|

---

## Network

|Field|Value|
|---|---|
|**Hostname (FQDN)**|`pve-prod-03.local`|
|**IP**|`192.168.10.13/24`|
|**VLAN**|10 — Management, untagged on bridge|
|**Gateway**|`192.168.10.1`|
|**DNS**|`192.168.10.1`, search domain `local` (live-verified)|
|**Physical NIC**|`nic0`, 1 GbE|
|**Bridge**|`vmbr0`, VLAN-aware, allowed VLAN IDs `2-4094`|

### Network Interfaces

|Interface|Type|Role|
|---|---|---|
|`nic0`|Physical Ethernet|Management and guest bridge uplink|
|`vmbr0`|Linux bridge|Static host management `192.168.10.13/24`; tagged guest VLAN traffic|

The bridge was live-verified active November 13. `dns-prod-02` uses the Services VLAN (30) at `192.168.30.15`.

---

## Proxmox Configuration

### Cluster

- Member of `homelab-v3`, node ID `3`, Corosync address `192.168.10.13`.
    
- Joined as the third node after removal of the former Raspberry Pi QDevice.
    
- Quorum now uses three Proxmox voting members (`pve-prod-01`, `pve-prod-02`, `pve-prod-03`); no QDevice is configured.
    
- No Proxmox HA policies are enabled. Cluster quorum does not automatically restart a guest on another host.
    

### Storage

|Storage|Type|Notes|
|---|---|---|
|`local`|Directory|ISO/template and other configured directory content|
|`local-zfs`|ZFS|Local `rpool/data`, ZFS mirror; same storage ID on three nodes but physically separate disks|
|`nas-isos-p2`|NFS|Unraid `192.168.30.16:/mnt/user/isos`, shared ISO library, assigned to `pve-prod-02` and `pve-prod-03`|
|`pbs-prod-01`|PBS|Docker PBS on `nas-prod-01` at `192.168.30.16:8007`, datastore `nas-backups`|

`nas-isos` on `pve-prod-01` uses its direct DAC path (`10.0.0.2`) to the same Unraid ISO export. Nodes 02/03 deliberately use `nas-isos-p2` via the Services VLAN instead.

**ZFS maintenance:** The default ZFS cron schedule previously verified on this host trims on the first Sunday and scrubs on the second Sunday of each month at 00:24. `rpool` autotrim was `off`. No changes were required. A successful maintenance execution on the new install has not been specifically verified.

**Redundant boot:** `rpool` mirrors both NVMe disks. Independent EFI-boot failover on this host has not been documented as physically tested; do not equate an ONLINE mirror with a confirmed single-disk boot test.

### UPS Monitoring — NUT Client

**Implemented and operationally verified November 13, 2026.**

This node runs Network UPS Tools as a network client to the UPS server on `nas-prod-01`. The UPS is connected to Unraid via USB; no UPS driver or NUT server is configured locally on the Lenovo.

|Setting|Value|
|---|---|
|**Package**|`nut-client` 2.8.1-5|
|**Operating mode**|`netclient`|
|**NUT server**|`192.168.10.10` (`nas-prod-01`)|
|**UPS identifier**|`ups`|
|**MONITOR target**|`ups@192.168.10.10`|
|**MONITOR role**|`slave` / secondary|
|**Power value**|`1`|
|**MINSUPPLIES**|`1`|
|**SHUTDOWNCMD**|`/sbin/shutdown -h +0`|
|**POLLFREQ**|`5` seconds|
|**HOSTSYNC**|`15` seconds|
|**DEADTIME**|`15` seconds|
|**FINALDELAY**|`5` seconds|
|**POWERDOWNFLAG**|`/etc/killpower`|
|**Systemd service**|`nut-monitor.service`, enabled and active|

**Configuration files:**

- `/etc/nut/nut.conf`: `MODE=netclient`.
    
- `/etc/nut/upsmon.conf`: UPS monitoring and shutdown settings; contains protected monitoring credentials.
    
- `/etc/systemd/system/nut-monitor.service.d/override.conf`: local systemd compatibility correction.
    

The active configuration files were installed as `root:nut`, mode `0640`. Credentials were transferred directly from the working `pve-prod-01` configuration over SSH and are not recorded in Markdown.

**Systemd packaging workaround:**

The installed Debian NUT service referenced `/usr/lib/tmpfiles.d/nut-common-tmpfiles.conf`, which was absent. The actual installed tmpfiles definition was `/usr/lib/tmpfiles.d/nut-client.conf`, containing the `/run/nut` directory definition.

The following drop-in corrects the service's `ExecStartPre` command while leaving the vendor service file unchanged:

```
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
    

**Nonfatal startup messages:** An initial check for `/run/nut/upsmon.pid` reported that no previous PID file existed. NUT also logged `Init SSL without certificate database` and systemd notification messages from `upsnotify`. These did not prevent the monitor from starting. No additional suppression or notification changes were made.

**Validation boundary:** Healthy UPS queries and the active service are verified, but a real UPS-triggered shutdown has not been performed. Guest shutdown order, server-to-client forced-shutdown signaling, and restart behavior following restoration of utility power remain untested.

**Useful read-only checks:**

```
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

- November 13, 2026, 02:00: node `pve-prod-03` scheduled `vzdump` completed `TASK OK`.
    
- Its log included CT 102 and, at that time, VM 105. VM 105 was moved to the replacement `pve-prod-02` later on November 13.
    
- The backup job now covers CT 102 on node 03; new node 02 has VM 105.
    
- A full isolated CT restore has not been specifically verified for CT 102.
    

See `runbooks/backup-schedule.md` and `runbooks/backup-restore.md`.

---

## Guests

|ID|Name|Type|VLAN|IP|Status|
|---|---|---|---|---|---|
|102|`dns-prod-02`|LXC (Debian 12)|30|`192.168.30.15`|Running; 1 vCPU, 1 GiB RAM, 4 GiB rootfs|

CT 102 is the **secondary AdGuard Home** instance. The primary DNS instance, CT 100, runs on `pve-prod-01`; application synchronization was verified following the original relocation. Use both nodes for DNS resilience, rather than co-locating primary and secondary DNS on one Proxmox host.

**Temporary workload:** VM 105 (`immich-prod-01`) was migrated here from the original `pve-prod-02` to vacate that node for replacement. After the new Dell joined the cluster, Immich was shut down and migrated offline from `pve-prod-03` to the new `pve-prod-02` on November 13, 2026. It is **not** currently hosted on node 03.

---

## Design Decisions

### Third Voting Node, No QDevice

A third compute node allows normal 2-of-3 cluster quorum. The former Raspberry Pi QDevice was removed prior to onboarding the Lenovo. This is not equivalent to guest HA: the cluster still requires explicit recovery or migration procedures after a host failure.

### Mirrored NVMe Boot

Two reused 512 GB NVMe disks were configured as a ZFS RAID-1 mirror during installation, matching the preferred local boot redundancy of the other current compute nodes. PBS is still required for recovery from non-disk failures and user/data corruption.

### Secondary DNS Placement

Placing CT 102 on node 03 separates it from primary AdGuard CT 100 on node 01 and leaves node 02 dedicated to Immich at present.

---

## Installation and Operational Notes

### Deployment — November 2026

- Added Lenovo ThinkCentre M90q to `homelab-v3` after preparing a mirrored ZFS installation and removing the earlier two-node QDevice.
    
- Configured `vmbr0` as VLAN-aware and connected it to USW-Pro-Max-24 (port 16 in the last verified switch placement).
    
- Relocated CT 102 here from the original `pve-prod-02`; performed a stop/backup/restore migration because the source used LVM-thin and the destination uses ZFS. Verified DNS function and AdGuard synchronization afterward.
    
- Temporarily hosted Immich VM 105 until completion of the Dell OptiPlex 5000 replacement on November 13.
    
- The scheduled November 13 02:00 PBS job completed successfully for CT 102 and VM 105 while both were still on node 03.
    
- On November 13, installed and configured `nut-client` 2.8.1-5 using the established `pve-prod-01` network-client configuration. Applied a systemd tmpfiles drop-in, verified UPS `OL` status and an active monitoring service, and removed temporary copies containing monitoring credentials.
    

### Previous Read-only Audit

- ZFS mirror ONLINE; NVMe SMART tests showed no media/data errors. Toshiba endurance usage was already 28%, so monitor wear over time without assuming failure is imminent.
    
- `pve-manager` 9.2.26 and kernel `7.0.14-28-pve` verified November 13.
    
- Uptime Kuma ping monitoring for this host was previously added; monitoring delivery/history should be checked separately if host availability changes.
    
- The previously observed BIOS M2WKT61A is older than a later firmware release identified for review. Updating BIOS was intentionally deferred while Immich was temporarily resident; firmware update completion has not been verified.
    
- Beszel agent deployment was deferred; do not mark it implemented merely because older monitoring documentation references Beszel.
    

### Open Validation Items

- Independently test boot from each SSD if redundant-boot recovery assurance is required.
    
- Re-check physical switch port and firmware versions when doing a future hardware/BIOS maintenance window.
    
- Verify an isolated restore of CT 102 as part of the backup recovery test plan.
    
- Plan and safely validate UPS-triggered guest and host shutdown, including recovery after power restoration; do not simulate a forced shutdown on the production UPS during routine service checks.