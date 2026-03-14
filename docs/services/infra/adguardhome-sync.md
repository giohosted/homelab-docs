# adguardhome-sync

**Stack:** infra  
**Host:** docker-prod-01 (192.168.30.11)  
**Status:** Running  
**Last Updated:** 2026-03-13

---

## Purpose

Keeps dns-prod-02 (secondary AdGuard) in sync with dns-prod-01 (primary AdGuard). Runs on a 30-minute cron schedule and on container startup. Ensures both AdGuard instances have identical upstream resolvers, blocklists, DNS rewrites, and filtering rules.

---

## Compose File

**Path:** `/opt/stacks/adguardhome-sync/compose.yaml`
```yaml
services:
  adguardhome-sync:
    image: ghcr.io/bakito/adguardhome-sync
    container_name: adguardhome-sync
    command: run --config /config/adguardhome-sync.yaml
    volumes:
      - /opt/appdata/adguardhome-sync/adguardhome-sync.yaml:/config/adguardhome-sync.yaml
    restart: unless-stopped
```

---

## Config File

**Path:** `/opt/appdata/adguardhome-sync/adguardhome-sync.yaml`  
**Gitignored:** Yes — contains credentials, lives in appdata
```yaml
origin:
  url: http://192.168.30.10:3000
  username: admin
  password: "your_password"

replica:
  url: http://192.168.30.15:3000
  username: admin
  password: "your_password"

cron: "*/30 * * * *"
runOnStart: true
```

> Passwords with special characters (e.g. `%`) must be quoted in YAML or the parser will fail with "found character that cannot start any token".

---

## Sync Behavior

- **Origin:** dns-prod-01 at 192.168.30.10:3000
- **Replica:** dns-prod-02 at 192.168.30.15:3000
- **Schedule:** Every 30 minutes
- **Run on start:** Yes — syncs immediately when container starts
- **What gets synced:** Upstream DNS config, blocklists, filtering rules, DNS rewrites, client settings, query log config, stats config, general settings

---

## Management
```bash
# View logs
cd /opt/stacks/adguardhome-sync
docker compose logs -f

# Restart
docker compose restart

# Force immediate sync (restart triggers runOnStart)
docker compose down && docker compose up -d
```

---

## Notes

- Config file uses `replica:` (single struct) not `replicas:` (list) — simpler for one replica
- Compose file is safe to commit to homelab-infra — no secrets
- Config file is gitignored — backed up nightly to NAS via backup-docker.sh (Phase 5)
- TLSConfig sync is disabled by default — not needed since AdGuard is HTTP only internally