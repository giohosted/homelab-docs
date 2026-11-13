# AdGuard Home

**Host:** dns-prod-01 (192.168.30.10) — primary  
**Host:** dns-prod-02 (192.168.30.15) — secondary  
**Container type:** LXC (Debian 12)  
**Status:** Running — both instances verified November 13, 2026  
**Last Updated:** 2026-11-13

---

## Deployment

- **Primary:** dns-prod-01 — LXC on pve-prod-01, CT ID 100
- **Secondary:** dns-prod-02 — LXC on pve-prod-03, CT ID 102, synced from primary via adguardhome-sync
- **Primary version:** Previously documented as v0.107.73; current version not independently verified
- **Secondary version:** v0.107.79 — verified from service logs November 12, 2026
- **Web UI (primary):** `http://192.168.30.10:3000`
- **Web UI (secondary):** `http://192.168.30.15:3000`
- **DNS port:** 53 (UDP/TCP)

### LXC Resources

| Setting | dns-prod-01 | dns-prod-02 |
|---|---|---|
| Proxmox host | pve-prod-01 | pve-prod-03 |
| Container ID | 100 | 102 |
| CPU | 1 core | 1 core |
| RAM | 1024 MiB | 1024 MiB |
| Swap | 512 MiB | 512 MiB |
| Root disk | 4 GiB | 4 GiB |
| Storage | local-zfs | local-zfs |
| VLAN | 30 — Services | 30 — Services |
| Start on boot | Enabled | Enabled |

Resource allocations and guest placement were verified through the live Proxmox integration on November 13, 2026.

> AdGuard Home runs as an LXC, not a Docker container. It is managed through Proxmox, not Dockman.

---

## Overview

AdGuard Home is the DNS server for the homelab's service and client networks. It provides:

- Ad, tracker, and malware/phishing domain blocking
- Split-horizon DNS — `*.giohosted.com` resolves internally to Traefik (192.168.30.11)
- DNS rewrites for internal services
- Encrypted upstream DNS resolution

The secondary instance (dns-prod-02) is managed as a configuration replica of dns-prod-01 through adguardhome-sync.

It provides DNS redundancy if the primary becomes unavailable, provided clients or their DNS configuration can use the secondary resolver.

The secondary instance now runs on pve-prod-03 instead of pve-prod-02.

---

## Network

| Instance | IP | VLAN | Proxmox Host |
|---|---|---|---|
| dns-prod-01 (primary) | 192.168.30.10 | 30 — Services | pve-prod-01 |
| dns-prod-02 (secondary) | 192.168.30.15 | 30 — Services | pve-prod-03 |

DHCP on the UDM-SE is documented as pushing `192.168.30.10` as the DNS server to client VLANs. dns-prod-02 is a failover resolver and was not pushed through DHCP by default in the documented configuration.

This DHCP behavior has not been independently reverified during the November 13 investigation.

---

## Upstream DNS

The previous documentation recorded DNS-over-TLS (DoT) resolvers:

| Resolver | Previously Documented Address |
|---|---|
| Cloudflare | `tls://1.1.1.1` |
| Quad9 | `tls://9.9.9.9` |

However, November 12 service logs from dns-prod-02 show DNS-over-HTTPS (DoH) requests to:

- `https://dns.quad9.net/dns-query`
- `https://cloudflare-dns.com/dns-query`

The logs also show bootstrap resolver requests to Quad9 addresses, including `9.9.9.10` and `149.112.112.10`.

**Documentation discrepancy:** The current secondary configuration demonstrably uses DoH for at least these upstream requests, while the previous service documentation describes DoT. The complete upstream settings for both instances have not been independently compared.

> Verify the configured upstream and bootstrap resolver lists in both AdGuard interfaces before replacing the historical resolver configuration.

> VLAN 10 (Management) uses public DNS independently of AdGuard. Proxmox management services should not depend on the DNS containers they host.

---

## Blocklists

| List | URL | Purpose |
|---|---|---|
| Hagezi Normal | `https://adguardteam.github.io/HostlistsRegistry/assets/filter_34.txt` | Ad and tracker blocking |
| Hagezi Threat Intelligence Feeds | `https://adguardteam.github.io/HostlistsRegistry/assets/filter_44.txt` | Malware and phishing domain blocking |

Both lists are maintained by Hagezi and updated regularly. Normal handles privacy (ads/trackers); Threat Intelligence Feeds handles security (malware/phishing). These complement each other — Normal is not a superset of Threat Intelligence Feeds.

### Observed Filter Update — November 12, 2026

The secondary AdGuard instance logged two completed filter updates at approximately 21:01 CST.

| Filter ID | Rules After Update |
|---|---:|
| 1773435342 | 209,073 |
| 1773544635 | 1,428,955 |
| **Total** | **1,638,028** |

These are summed per-filter rule counts, not a verified unique-rule count. The filter IDs have not been independently matched to the named Hagezi lists.

Shortly after the update, the secondary experienced sustained CPU and memory saturation, followed by an out-of-memory process termination.

See the incident record under Troubleshooting.

---

## Query Log & Statistics

| Setting | Value |
|---|---|
| Query log retention | 7 days |
| Statistics retention | 7 days |

These are the previously documented settings and were not reverified during the OOM investigation.

---

## DNS Rewrites

All documented `*.giohosted.com` service rewrites point to Traefik at `192.168.30.11`. Rewrites are managed on the primary (dns-prod-01) and synced to the secondary automatically.

| Domain | Answer |
|---|---|
| `traefik.giohosted.com` | 192.168.30.11 |
| `auth.giohosted.com` | 192.168.30.11 |
| `dockman.giohosted.com` | 192.168.30.11 |
| `sonarr-tv.giohosted.com` | 192.168.30.11 |
| `sonarr-anime.giohosted.com` | 192.168.30.11 |
| `radarr-1080p.giohosted.com` | 192.168.30.11 |
| `radarr-4k.giohosted.com` | 192.168.30.11 |
| `prowlarr.giohosted.com` | 192.168.30.11 |
| `bazarr.giohosted.com` | 192.168.30.11 |
| `profilarr.giohosted.com` | 192.168.30.11 |
| `maintainerr.giohosted.com` | 192.168.30.11 |
| `request.giohosted.com` | 192.168.30.11 |
| `tautulli.giohosted.com` | 192.168.30.11 |
| `qbittorrent.giohosted.com` | 192.168.30.11 |
| `qbitrr.giohosted.com` | 192.168.30.11 |
| `audiobooks.giohosted.com` | 192.168.30.11 |
| `cwa.giohosted.com` | 192.168.30.11 |
| `shelf.giohosted.com` | 192.168.30.11 |
| `photos.giohosted.com` | 192.168.30.11 |

> Always add new DNS rewrites to dns-prod-01 (primary) only — adguardhome-sync propagates them to dns-prod-02 automatically within 30 minutes.

---

## IoT DNS Interception

IoT devices on VLAN 40 are not permitted to reach VLAN 30 directly.

The UDM-SE intercepts DNS queries from VLAN 40 on port 53 and forwards them to AdGuard at 192.168.30.10.

IoT devices receive ad and malware blocking without needing a direct path to the Services VLAN.

---

## Adding a New DNS Rewrite

1. Go to the primary AdGuard Web UI at `http://192.168.30.10:3000`.
2. Navigate to Filters → DNS Rewrites → Add DNS Rewrite.
3. Domain: `service.giohosted.com`
4. Answer: `192.168.30.11`
5. Save — adguardhome-sync will propagate the change to dns-prod-02 within 30 minutes.

---

## Troubleshooting

### Internal hostnames not resolving

- Verify the DNS rewrite exists in AdGuard primary (dns-prod-01).
- Check that the client is using the intended AdGuard DNS resolver.
- If using Windows, flush DNS cache with `ipconfig /flushdns`.
- If using Chrome, review Secure DNS (DoH). External DoH providers may bypass AdGuard's internal rewrites.

### AdGuard Web UI not reachable

- Verify dns-prod-01 (CT 100) is running on pve-prod-01.
- Try the secondary at `http://192.168.30.15:3000`.
- Verify dns-prod-02 (CT 102) is running on pve-prod-03.
- Check the AdGuardHome systemd service and its logs.

### Out-of-Memory Incident — November 12, 2026

**Status:** Memory allocation increased and verified; recurrence and underlying trigger not yet fully ruled out.

**Affected service:** AdGuardHome  
**Affected container:** dns-prod-02 (CT 102)  
**Proxmox host:** pve-prod-03  
**Incident time:** 2026-11-12 21:22:33 CST  
**Previous RAM allocation:** 512 MiB  
**Current RAM allocation:** 1024 MiB

#### Observed Behavior

1. At approximately 21:01 CST, the secondary completed updates for two DNS filtering lists.
2. At approximately 21:02 CST, container memory usage began increasing rapidly.
3. From approximately 21:03 through 21:22 CST, container RAM remained at its 512 MiB limit.
4. During this period, CPU usage remained near 99% of the container's allocated core.
5. AdGuard logged repeated upstream and bootstrap DNS resolution timeouts.
6. At 21:22:33 CST, the Linux kernel killed the AdGuardHome process because the container's memory cgroup exceeded its available limit.
7. systemd automatically restarted AdGuardHome at approximately 21:22:43 CST.
8. DNS listeners were started successfully at approximately 21:22:46 CST.

#### Kernel Evidence

The Proxmox host journal confirmed:

`oom-kill:constraint=CONSTRAINT_MEMCG`

`oom_memcg=/lxc/102`

`task=AdGuardHome,pid=1058449`

`Memory cgroup out of memory: Killed process 1058449 (AdGuardHome)`

The terminated process had approximately 508,364 KiB of anonymous resident memory (~496 MiB).

This was a container-specific memory exhaustion event, not a host-wide RAM shortage.

#### Probable Cause

The timing strongly implicates the blocklist update or subsequent processing of approximately 1.64 million reported filter rules.

However, the exact reason AdGuard remained CPU-saturated for approximately 20 minutes is unconfirmed.

Potential contributing factors include:

- Insufficient RAM for the filtering workload
- Expensive filter parsing or rebuilding
- A software defect or unusually costly processing operation

The accompanying DNS upstream timeouts may have resulted from service saturation, but an independent DNS/network problem has not been excluded.

#### Corrective Action — November 13, 2026

The RAM allocation of CT 102 was increased from 512 MiB to 1024 MiB:

`pct set 102 --memory 1024`

**Verification:**

- Proxmox reported 1024 MiB configured.
- The running container immediately reflected the new 1 GiB limit.
- No pending configuration changes were reported.
- No LXC restart or reboot was required.
- Memory usage after the change was approximately 257 MiB.
- Swap usage was 0 MiB out of 512 MiB.
- Both AdGuard instances remained running.

The primary already had a 1 GiB allocation, so the secondary now matches its RAM configuration.

CPU remains at 1 core. No CPU allocation changes were made.

#### Follow-Up Verification

- [ ] Observe the next scheduled filter refresh.
- [ ] Verify whether memory remains below the new 1 GiB limit.
- [ ] Check for sustained CPU saturation or additional OOM events.
- [ ] Investigate abnormal filter processing if the problem recurs.
- [ ] Confirm AdGuard DNS responses during and after the next refresh.
- [ ] Compare primary and secondary upstream DNS configuration.
- [ ] Review any additional systemd restart history, since the incident logs showed a restart counter of 3.

**Resolution status:** The resource limit change is implemented and verified. Long-term resolution remains pending observation of a subsequent filter refresh.

### AdGuard Directory Permissions Warning

On November 12, the secondary instance logged:

`permcheck: warning: found unexpected permissions type=directory path=/opt/AdGuardHome perm=0777 want=0700`

The directory permissions are more permissive than AdGuard expects.

This warning was not established as a cause of the OOM event.

**Status:** Open for separate investigation. Inspect ownership, service account, and directory requirements before changing permissions.

---

## Notes

- Rewrites added directly to dns-prod-02 will be overwritten by the next adguardhome-sync run — always manage configuration through the primary.
- adguardhome-sync runs every 30 minutes — see `services/infra/adguardhome-sync.md`.
- Secondary AdGuard now runs on pve-prod-03, not pve-prod-02.
- CT 102's memory allocation was increased live from 512 MiB to 1 GiB on November 13, 2026.
- Both AdGuard containers have 1 GiB RAM and 1 CPU core as of the latest Proxmox verification.
- The November 12 OOM was confined to CT 102; pve-prod-03 had substantial free host memory.
- The filter-refresh relationship remains a strong hypothesis, not a proven software root cause.
- Chrome Secure DNS can bypass AdGuard — review client DNS configuration if internal hostnames fail.
- Client-specific filtering rules were not documented; global blocking is the established configuration.
