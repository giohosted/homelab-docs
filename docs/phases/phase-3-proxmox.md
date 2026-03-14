# Phase 3 — Proxmox Rebuild & Core VM Infrastructure

**Status:** Complete  
**Started:** 2026-03-13  
**Last Updated:** 2026-03-13

---

## Objectives

1. Fresh Proxmox install on MS-A2 (pve-prod-01) — ZFS RAID-1 boot mirror on 2x NVMe
2. Fresh Proxmox install on Optiplex (pve-prod-02)
3. Cluster both nodes, add pi-prod-01 as QDevice
4. Create AdGuard LXC (dns-prod-01) on VLAN 30 — 192.168.30.10
5. Create AdGuard LXC (dns-prod-02) on VLAN 30 — 192.168.30.15
6. Deploy adguardhome-sync on docker-prod-01
7. Create docker-prod-01 VM on VLAN 30 — 192.168.30.11
8. Create pbs-prod-01 VM on pve-prod-02 — 192.168.30.12
9. Install NUT clients on both Proxmox nodes

---

## Entry Criteria

- Phase 2 complete — nas-prod-01 healthy, all pools online, NFS exports verified
- All hardware racked and powered
- Network foundation complete — VLANs 10/20/30/40 configured, firewall rules validated

---

## Build Log

### pve-prod-01 — Minisforum MS-A2

- **Proxmox version:** 9.1.6
- **Kernel:** 6.17.13-1-pve
- **Boot storage:** ZFS RAID-1 mirror — Samsung 980 1TB + Sabrent Rocket 1TB (pool: rpool)
- **Management IP:** 192.168.10.11 (VLAN 10)
- **NIC used for management:** igc (Intel I226-V 2.5GbE) — connected to USW-Pro-Max-24 port 17-24 (2.5GbE ports)
- **vmbr0:** VLAN-aware bridge on igc — carries all VM/LXC traffic tagged per guest + management untagged
- Enterprise repos removed (pve-enterprise.sources, ceph.sources) — replaced with pve-no-subscription repo (trixie)
- System updated via apt dist-upgrade
- Subscription nag popup removed
- local-lvm ghost entry removed (`pvesm remove local-lvm`) — pve-prod-01 uses local-zfs
- local-zfs re-added after pvesm entry was lost: `pvesm add zfspool local-zfs --pool rpool/data --content rootdir,images`

#### Issues Encountered
- PVE 9 uses `.sources` format instead of `.list` files for apt repos — correct files were `pve-enterprise.sources` and `ceph.sources`
- MS-A2 initially plugged into 1GbE port on switch (ports 1-16) — moved to 2.5GbE port (ports 17-24), confirmed 2500Mb/s via ethtool
- local-zfs storage entry missing after install — re-added via pvesm

---

### pve-prod-02 — Dell Optiplex 3070 Micro

- **Proxmox version:** 9.1.6
- **Boot storage:** Single NVMe — ext4 (no redundancy — acceptable for secondary node)
- **Management IP:** 192.168.10.12 (VLAN 10)
- **NIC:** 1GbE RJ45 — connected to USW-Pro-Max-24 (any port, 1GbE max)
- **vmbr0:** VLAN-aware bridge — same config as pve-prod-01
- Enterprise repos removed, pve-no-subscription repo added (trixie)
- System updated, subscription nag removed
- local-zfs ghost entry removed: `pvesm remove local-zfs`
- local-lvm re-added: `pvesm add lvmthin local-lvm --vgname pve --thinpool data --content rootdir,images`

#### Issues Encountered
- Installer showed "no supported hard disk found" — resolved by changing SATA controller mode from RAID/RST to AHCI in BIOS (Dell F2 → System Configuration → SATA Operation)
- local-lvm missing from storage after install — re-added via pvesm

---

### Proxmox Cluster — homelab-v3

- **Cluster name:** homelab-v3
- **Transport:** knet
- **Secure auth:** on
- **Nodes:** pve-prod-01 (primary), pve-prod-02 (secondary)
- **HA:** NOT enabled — clustering for unified management UI only. Optiplex cannot handle MS-A2 workload.
- Cluster created on pve-prod-01 first, pve-prod-02 joined via join information token

---

### pi-prod-01 — QDevice Setup & VLAN Migration

- **Previous state:** VLAN 20, IP 192.168.20.8, running AdGuard Home (v2 role)
- **New state:** VLAN 10, IP 192.168.10.20
- **Network manager:** NetworkManager (not dhcpcd — Raspberry Pi OS Bookworm default)
- Static IP set via nmcli — dhcpcd.conf edits do nothing on this OS version
- Switch port changed to Mgmt-Only profile (access port, VLAN 10) on USW-Pro-Max-24
- `corosync-qdevice` + `corosync-qnetd` installed on Pi
- `corosync-qdevice` installed on both Proxmox nodes
- QDevice initialized via `pvecm qdevice setup 192.168.10.20` on pve-prod-01
- Root SSH enabled temporarily for QDevice setup, disabled after confirmation

#### QDevice Status (confirmed working)
```
Expected votes: 3
Total votes:    3
Quorate:        Yes
Flags:          Quorate Qdevice
```

#### Issues Encountered
- dhcpcd.conf edits had no effect — Pi OS Bookworm uses NetworkManager, not dhcpcd
- Root SSH login disabled by default on Pi — had to enable PermitRootLogin temporarily
- corosync-qnetd package missing from Pi — caused certutil errors, installed separately
- corosync-qdevice package missing from Proxmox nodes — installed separately

---

### dns-prod-01 — AdGuard LXC (pve-prod-01)

- **CT ID:** 100
- **Host:** pve-prod-01
- **Template:** debian-12-standard
- **IP:** 192.168.30.10/24 (VLAN 30)
- **Resources:** 1 vCPU, 512MB RAM, 4GB disk (local-zfs)
- **AdGuard Home:** installed via official install script
- **Admin UI:** http://192.168.30.10:3000
- SSH enabled (openssh-server installed manually)

#### AdGuard Configuration
- **Upstream DNS:** `https://dns.quad9.net/dns-query`, `https://doh.opendns.com/dns-query`
- **Fallback DNS:** `9.9.9.10`, `1.1.1.1`
- **Bootstrap DNS:** `9.9.9.10`, `1.1.1.1`
- **Upstream mode:** Parallel requests
- **DNSSEC:** Enabled
- **Blocklist:** AdGuard DNS filter (default)
- **Custom filtering rules:**
```
@@||api.pubnative.net^
@@||api.sprig.com^
@@||api2.branch.io^$important
```
- **DNS rewrites:** None configured — will be added in Phase 4 pointing at Traefik (192.168.30.11)

#### UDM-SE DNS Configuration (updated)

| VLAN | DNS Server | Notes |
|------|------------|-------|
| 10 — Management | 9.9.9.9 | Public DNS — management must not depend on Proxmox |
| 20 — Trusted | 192.168.30.10 | AdGuard on dns-prod-01 |
| 30 — Services | 192.168.30.10 | AdGuard on dns-prod-01 |
| 40 — IoT | 9.9.9.9 | Public DNS — IoT fully isolated |

---

### dns-prod-02 — AdGuard LXC (pve-prod-02)

- **CT ID:** 102
- **Host:** pve-prod-02
- **Template:** debian-12-standard
- **IP:** 192.168.30.15/24 (VLAN 30)
- **Resources:** 1 vCPU, 512MB RAM, 4GB disk (local-lvm)
- **AdGuard Home:** installed via official install script
- Config synced from dns-prod-01 via adguardhome-sync
- SSH enabled, PermitRootLogin yes

#### Issues Encountered
- local-lvm not available in CT creation UI — local-zfs ghost entry had to be removed first from pve-prod-02

---

### adguardhome-sync (Docker, docker-prod-01)

- Syncs dns-prod-01 → dns-prod-02 every 30 minutes and on startup
- Compose: `/opt/stacks/adguardhome-sync/compose.yaml` (safe to commit)
- Config: `/opt/appdata/adguardhome-sync/adguardhome-sync.yaml` (gitignored — contains credentials)
- Uses `replica:` (single) config format, not `replicas:` list

#### Issues Encountered
- `%` character in AdGuard password caused YAML parse error — fixed by quoting the value in the config file

---

### docker-prod-01 — Docker Host VM (pve-prod-01)

- **VM ID:** 101
- **OS:** Ubuntu Server 24.04.2 LTS (no snaps, no minimized install)
- **IP:** 192.168.30.11/24 (VLAN 30)
- **Resources:** 4 cores (x86-64-v3), 6GB RAM, 64GB disk (local-zfs, write-back cache, discard)
- **System:** q35, OVMF UEFI, VirtIO SCSI Single, QEMU agent enabled
- Docker 29.3.0 installed via official script
- NFS client installed (nfs-common)
- NFS mount: `192.168.30.16:/mnt/user` → `/data` via fstab with `_netdev,x-systemd.automount`
- service:service user/group created at UID/GID 2000:2000
- gio added to service group

#### Directory Structure Created
```
/data/media/movies/1080p
/data/media/movies/4k
/data/media/tv
/data/media/anime
/data/media/books/ebooks
/data/media/books/audiobooks
/data/downloads/movies
/data/downloads/tv
/data/downloads/anime
/data/downloads/books/ebooks/downloads
/data/downloads/books/ebooks/ingest
/data/downloads/books/audiobooks/downloads
```

#### /opt Structure
```
/opt/stacks/{traefik,cloudflared,adguardhome-sync,dockman,arr,torrent,books}
/opt/appdata/{traefik,cloudflared,adguardhome-sync,dockman,beszel,sonarr-tv,sonarr-anime,radarr-1080p,radarr-4k,prowlarr,bazarr,profilarr,maintainerr,seerr,tautulli,flaresolverr,qbittorrent,gluetun,qbitrr,audiobookshelf,calibre-web-automated,shelfmark}
```

---

### NFS Configuration — Key Fix

Unraid's default "Public" NFS exports use `root_squash` and `all_squash`, which prevents root writes from clients. Fixed by setting all shares to **Private** security with explicit rule:
```
192.168.30.0/24(async,no_subtree_check,rw,sec=sys,insecure,anongid=100,anonuid=99,no_root_squash)
```

Verified with `exportfs -v` showing `no_root_squash,no_all_squash` on all six shares scoped to 192.168.30.0/24.

---

### pbs-prod-01 — Proxmox Backup Server VM (pve-prod-02)

- **VM ID:** 103
- **OS:** Proxmox Backup Server 4.1.0
- **IP:** 192.168.30.12/24 (VLAN 30)
- **Resources:** 1 core (x86-64-v2-AES), 2GB RAM, 16GB disk (local-lvm)
- Enterprise repo removed, no-subscription repo added (trixie)
- Subscription nag removed
- qemu-guest-agent installed
- NFS mount: `192.168.30.16:/mnt/user/backups` → `/mnt/backups` via fstab
- Datastore `nas-backups` created at `/mnt/backups`
- Prune schedule: daily — keep last 3, daily 7, weekly 4, monthly 3
- PBS storage added to Proxmox Datacenter (cluster-wide)
- Backup job: all VMs/LXCs, daily 02:00, snapshot mode, zstd compression, retention managed by PBS prune schedule

#### Issues Encountered
- PBS repo suite was set to bookworm but PBS runs on trixie — corrected pbs-no-subscription.list
- x86-64-v2-AES chosen (not v3) — Optiplex i5-9500T is 9th gen and does not support v3

---

### NUT Clients — pve-prod-01 and pve-prod-02

- nut-client installed on both nodes
- `/etc/nut/nut.conf`: `MODE=netclient`
- `/etc/nut/upsmon.conf`: `MONITOR ups@192.168.10.10 1 slaveuser slavepass slave`
- Both nodes confirmed running and connected to NAS NUT server

---

## Firewall Changes

Added rule in UDM-SE: **Allow Management to All**
- Source: Management network (192.168.10.0/24)
- Destination: Trusted + Services + IoT
- Action: Allow
- **Auto Allow Return Traffic: ✅ enabled** — required for return packets to reach VLAN 10 hosts

Without this rule, pve-prod-01 (VLAN 10) could not reach VMs on VLAN 30. Physical NAS interface (192.168.30.16) was reachable because it's a direct switch connection, but VM interfaces behind vmbr0 were not.

---

## Decisions Made

| Decision | Rationale |
|----------|-----------|
| x86-64-v3 CPU type for VMs on pve-prod-01 | MS-A2 supports v3 but not v4. Better than host for VM portability. |
| x86-64-v2-AES for pbs-prod-01 | Optiplex i5-9500T is 9th gen — does not support v3 |
| service:service at UID/GID 2000 | Clean break from v2, matches NAS share permissions |
| No snaps on docker-prod-01 | Clean Ubuntu install, Docker via official apt repo |
| adguardhome-sync config in appdata | Credentials kept out of git — config file gitignored, compose file safe to commit |
| PBS retention managed by PBS prune schedule | More granular than Proxmox job-level retention — No Limit set on job |
| pi-prod-01 stays on VLAN 10 | QDevice must be on management network alongside Proxmox nodes |
| Root SSH disabled on pi-prod-01 | QDevice setup complete — no reason to leave root SSH open |

---

## Deferred to Phase 4

- TrueNAS VM on pve-prod-01 to migrate media/photos/books from old ZFS drives (drives currently unplugged on desk)
- All service migration (ARR stack, Plex, books, Immich, etc.)
- DNS rewrites in AdGuard pointing at Traefik (192.168.30.11)

---

## Pending (Phase 5)

- SSH key-based authentication on all hosts
- Beszel firewall rule: VLAN 30 → 192.168.10.20 TCP 45876 ALLOW

---

## Exit Criteria

- ✅ Proxmox healthy on both nodes
- ✅ Cluster quorate with QDevice tiebreaker
- ✅ AdGuard DNS running and serving VLAN 20 and VLAN 30
- ✅ dns-prod-02 synced from dns-prod-01 via adguardhome-sync
- ✅ docker-prod-01 VM running with NFS mounts verified and directory structure created
- ✅ pbs-prod-01 running with backup job configured
- ✅ NUT clients connected on both Proxmox nodes
- ✅ Root SSH disabled on pi-prod-01