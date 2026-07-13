# AI Context Handoff — Homelab v3.0

  

**Purpose:** This document exists so any AI assistant (or human) can pick up full working context on this project without prior conversation history. It's the companion to `00-roadmap.md` — the roadmap covers the plan, this covers current state, hard-won lessons, and how Gio likes to work.

**Last Updated:** 2026-07-13

---

## Project Snapshot

Gio is rebuilding his homelab from scratch (Homelab v3.0) — proper VLAN segmentation, virtualization, SSO, automated backups, and monitoring, with a future path toward Kubernetes. Two private GitHub repos:

- **homelab-docs** — this Obsidian vault, synced to GitHub, also mirrored to Google Drive
- **homelab-infra** — compose files, cloned at `/opt/stacks` on each Docker host

Phases 0–5 are the structure. Phases 0–4 are complete. Phase 5 (Hardening & Operational Readiness) is in progress — started 2026-03-17, Waves 1–6 complete (monitoring, backup automation, OIDC rollout, external access/CF Access, MAM seeding rules, service settings review). Waves 7 (Security Hardening) and 8 (Synology ABB) remain.

---

## Core Infrastructure Reference

**Network:** UniFi UDM-SE, USW-Pro-Max-24, U6-Pro AP
VLANs: 10 (Management), 20 (Trusted), 30 (Services), 40 (IoT)

**Compute:**
- pve-prod-01 — Minisforum MS-A2, Ryzen 9 7945HX, 192.168.10.11 — primary node
- pve-prod-02 — Dell Optiplex 3070, 192.168.10.12 — secondary node, no HA. Runs pbs-prod-01, dns-prod-02, and immich-prod-01 (moved here from pve-prod-01 pre-launch, due to RAM headroom — pve-prod-01 was already carrying docker-prod-01 + auth-prod-01)

**NAS:** nas-prod-01 — Unraid 7.2.4, mgmt 192.168.10.10, data 192.168.30.16, DAC link to pve-prod-01 via 10.0.0.x/30

**Support nodes:**
- pi-prod-01 — Raspberry Pi 4B, 192.168.10.20 — QDevice, Uptime Kuma, Beszel hub
- pbs-prod-01 — PBS VM, 192.168.30.12

**Docker hosts:**
- docker-prod-01 — 192.168.30.11 — primary, runs almost everything
- auth-prod-01 — 192.168.30.13 — Authentik, isolated on its own VM
- immich-prod-01 — 192.168.30.14 — Immich, isolated for resource tuning, runs as a VM on pve-prod-02 (not pve-prod-01)

**DNS:** dns-prod-01 (AdGuard primary, 192.168.30.10), dns-prod-02 (AdGuard secondary, 192.168.30.15), synced via adguardhome-sync

**Reverse proxy:** Traefik v3 on docker-prod-01, wildcard cert for `*.giohosted.com` via Cloudflare DNS-01. Cloudflare Tunnel for external access.

**SSO:** Authentik on auth-prod-01 — OIDC across Proxmox, PBS, Synology DSM, Beszel, qBitrr, ABS, CWA, Shelfmark, Seerr.

Full inventory, IP plan, and VLAN design live in `architecture/` — this doc doesn't duplicate those tables, just orients you to them.

---

## Deployed Services (all on docker-prod-01 unless noted)

- **Media:** Plex (on Unraid, QuickSync), Sonarr-TV, Sonarr-Anime, Radarr-1080p, Radarr-4K, Prowlarr, Bazarr, Profilarr, Maintainerr, Seerr, Tautulli, Flaresolverr
- **Torrent:** Gluetun (ProtonVPN WireGuard + port forwarding), qBittorrent, qBitrr
- **Books:** Audiobookshelf, Calibre-Web-Automated, Shelfmark
- **Photos:** Immich (on immich-prod-01, ~10k assets, buffalo_s facial recognition)
- **Monitoring:** Beszel (hub on pi-prod-01, agents on all 7 hosts), Uptime Kuma v2 — both on pi-prod-01
- **Infra:** Authentik, Traefik, cloudflared, Dockman, adguardhome-sync

---

## Where Things Stand Right Now

Phase 5 (Hardening) is in progress, started 2026-03-17. Waves 1–6 are complete; Waves 7 and 8 remain.

**Phase 5 completed so far:**

- **Wave 1 — Monitoring:** Beszel agents deployed on all 7 hosts, hub on pi-prod-01, OIDC via Authentik (`email_verified: true` custom scope required). Uptime Kuma running 30 monitors (10 ping, 20 HTTP).
- **Wave 2 — Backup automation:** Nightly rsync of `/opt/appdata` and `/opt/stacks` from docker-prod-01, auth-prod-01, and immich-prod-01 to NAS via SSH (dedicated backup key, cron at 03:00), with Healthchecks.io heartbeats per host. Rolling mirror with `--delete`, no versioning — that gap is what Wave 8 (Synology ABB) is meant to close. PBS retention finalized: keep last 3 / daily 7 / weekly 4 / monthly 3, GC and verify jobs running Saturdays.
- **Wave 3 — OIDC rollout:** Complete across Proxmox (both nodes + PBS), Immich, Beszel, Synology DSM, ABS, CWA, Shelfmark, qBitrr.
- **Wave 4 — External access:** Immich will get a *temporary* Cloudflare Tunnel route for a wedding in October (added the day before, removed a few days after) — no permanent exposure, guest access gated by Immich's built-in album password, no CF Access policy needed. ABS mobile app CF Access issue fixed via a service token + registering `audiobooth://oauth` as a mobile redirect URI.
- **Wave 5 — MAM seeding rules:** qBitrr configured with tracker-scoped rules (14-day min seed, 1.0 min ratio, auto-remove) — scoped specifically to `myanonamouse.net`, doesn't touch AudiobookBay or other trackers.
- **Wave 6 — Service settings review:** qBitrr, Immich (switched facial recognition `buffalo_l` → `buffalo_s`, enabled storage template), AdGuard (added Hagezi Threat Intelligence blocklist), and Plex (DB backup every 3 days) all tuned. Traefik and ARR stack needed no changes.

**Phase 5 remaining (Waves 7–8):**

- SSH key-based auth across all 7 hosts, password auth disabled — not started on any host yet
- NUT clients on auth-prod-01 and immich-prod-01 — still missing from Phase 4
- Firewall rule review — confirm nothing overly permissive survived from the migration period
- v2 decommission — shut down v2 Docker host and v2 Proxmox host, reclaim VLAN 20 IPs
- Synology ABB — versioned backups on top of the rsync mirror, schedule/retention still TBD
- Plex database backup — separate from the rsync scripts since Plex lives on Unraid directly, not in `/opt/appdata`

  
**Recently resolved issues (pre-Phase 5 / early troubleshooting):**

- ABS showing books as missing → manual library scan, resolved on its own
- PBS backups failing with `ESTALE: Stale file handle` → rebooted pbs-prod-01, hardened fstab with `soft,timeo=30,retrans=3,rsize=131072,wsize=131072`
- Immich container not running → manual `docker compose up -d`, confirmed restart policy `always`
- Uptime Kuma false positives on AdGuard/Authentik → fixed DNS resolution order on pi-prod-01, bumped dns-prod-01 LXC RAM to 1GB, trimmed AdGuard log retention to 24h
- ISP plan evaluation → WAN-to-UDM-SE link is 2.5GbE, that's the real ceiling for plan tier decisions, not internal LAN speed. UDM-SE IPS/IDS throughput still needs checking before assuming full gigabit-plus would actually be realized.

**On the horizon (post-Phase 5):**

- Adding pi-prod-01, auth-prod-01, immich-prod-01 as hosts in Dockman
- Long-term: local LLM inference hardware — RTX 3060 12GB is the preferred pick over Intel Arc A750, mainly for CUDA maturity and VRAM headroom
- Longer-term: possible Kubernetes migration (Phase 6, not started)

---

## Hard-Won Lessons (don't relearn these)

- **Hardlinks need one NFS mount.** Media and downloads must share a single NFS export (`/mnt/user/data`) mounted as one filesystem — separate mounts silently break hardlinks. NFSv3 over NFSv4 specifically to avoid pseudo-root automounting recreating the separate-filesystem problem.
- **Traefik cross-host routing needs the file provider.** The Docker provider only sees containers on the same host. Anything routing to auth-prod-01 or immich-prod-01 needs a static file-provider route. Reference is `local-only@docker` — `local-only@file` doesn't exist and fails silently.
- **cloudflared → Traefik must be `https://`, not `http://`.** HTTP causes an infinite redirect loop (Traefik redirects to HTTPS, cloudflared follows, repeats). Use `https://192.168.30.11` with No TLS Verify — safe because the hop never leaves the LAN.
- **Container UID mismatches cause silent crash loops.** Maintainerr and Seerr v3 run internally as UID 1000, not the standard 2000. Appdata owned `2000:2000` causes crash loops or containers falling off the proxy network.
- **DNS dependency loops break monitoring.** Uptime Kuma's own host must not use the DNS server it's monitoring as its primary resolver, or a DNS blip cascades into false positives.
- **Docker containers don't inherit host DNS.** Must be set explicitly in compose (AdGuard IPs) — falling back to public DNS breaks internal hostname resolution.
- **VLAN 30 → pi-prod-01 (VLAN 10) is a firewall crossing.** Needs an explicit allow rule (TCP 45876 for Beszel) — Traefik on VLAN 30 can't reach pi-prod-01 by default.
- **PBS datastore should scope to a subdirectory,** not the share root — prevents GC failures from root-owned rsync folders at the share level. Concretely: the NAS backups share holds `pbs/`, `appdata/`, and `stacks/` as siblings — PBS's datastore must point at `/mnt/backups/pbs` specifically, not `/mnt/backups`. If it's scoped to the share root, PBS's GC job will error out on the root-owned `appdata`/`stacks` folders sitting next to it.
- **CF Access blocks mobile apps that can't complete a browser auth challenge.** Fixed for Audiobookshelf via a CF Access service token (Access controls → Service credentials) added as a `SERVICE AUTH` policy on the app, evaluated *before* other policies (policy order 1) — plus registering the app's custom redirect scheme (`audiobooth://oauth`) in both Authentik and ABS's OIDC mobile redirect URIs. The mobile app then sends `CF-Access-Client-Id` / `CF-Access-Client-Secret` as custom headers. This pattern generalizes to any mobile app sitting behind CF Access that hits an API endpoint directly instead of going through a browser.
- **WAN vs. LAN speed are different questions.** Internal LAN throughput is irrelevant to ISP plan decisions — only the WAN-to-router physical link and actual WAN-bound workloads matter.
- **qBitrr needs both `http://` and `https://` registered as Authentik redirect URIs** — it sends HTTP internally even when served over HTTPS externally.


---

## How Gio Works — Read This Before Doing Anything

- **One step at a time.** Confirm each step before moving to the next. Never batch multiple actions without explicit go-ahead.
- **Exact commands and click paths, always.** Gio is not strong with CLI — vague guidance like "update the config" is not useful. Give the literal command or the literal GUI path.
- **Pushback is expected and wanted.** Gio wants to be corrected before making a mistake, and will push back if something seems off. Don't just agree to be agreeable — engage with the technical reasoning.
- **He takes initiative mid-troubleshooting.** He's comfortable rebooting or acting before a step-by-step process finishes. Adapt to what he's already done rather than repeating a script.
- **Documentation happens at phase/wave boundaries.** After finishing work, Gio pastes existing docs back for a full rewrite in clean Markdown, ready to paste into Obsidian directly. Never overwrite — merge with what's there.
- **Resource skepticism.** He questions CPU/RAM allocations that look excessive and expects justification. Prefers not to over-provision idle capacity.
- **Homelab-appropriate complexity.** He pushes back on enterprise-grade solutions (e.g. strict SSH key policies) that add overhead without meaningful benefit at this scale. Match the solution to a homelab, not a company.

  

---

## Standing Conventions

- **Secrets:** always in `.env` files, always gitignored, never hardcoded in compose or committed.
- **File permissions:** compose files `chown gio:service` + `chmod 664`; `.env` files `chmod 660`. Never use `sudo nano` to create files — it creates root-owned files that break Dockman's autosave.
- **Git workflow:** always `git pull` before making changes on any host, on any of the three hosts that clone homelab-infra. Compose files are committed; `.env` files are gitignored.
- **Service user:** UID/GID 2000 (`service` group) is standard for self-hosted containers. Documented exceptions: Maintainerr, Seerr, and other node-based apps that hardcode UID 1000.
- **Reference the checklist for new stacks:** `runbooks/new-service-checklist.md` covers directory creation, permissions, git commit, and DNS rewrite steps in order — follow it exactly for every new deployment.

---

## What This Document Is Not

This isn't a replacement for the phase docs, architecture docs, or service docs — it's the orientation layer that sits on top of them. If you're an AI assistant picking this project up cold: read this first, then pull the specific `architecture/` or `services/` doc relevant to whatever Gio is asking about. Don't assume this doc has the full technical detail — it deliberately doesn't, to avoid drifting out of sync with the source-of-truth docs it points to.