# FlareSolverr

**Host:** docker-prod-01 (192.168.30.11)  
**Container name:** flaresolverr  
**Status:** Running  
**Last Updated:** 2026-03-16

---

## Deployment

- **Image:** ghcr.io/flaresolverr/flaresolverr:latest
- **Compose:** `/opt/stacks/arr/compose.yaml`
- **Appdata:** None — stateless, no persistent data
- **Network:** proxy
- **Config source:** Restored from v2 backup

## Container Variables

| Variable | Value |
|----------|-------|
| LOG_LEVEL | info |
| TZ | America/Chicago |

---

## Overview

FlareSolverr is a proxy server that bypasses Cloudflare and DDoS-Guard protection on torrent indexer sites. Prowlarr sends requests through FlareSolverr when an indexer requires JavaScript challenge solving.

FlareSolverr has no WebUI and is not exposed via Traefik — it is internal only, accessed by Prowlarr via container name.

---

## Prowlarr Configuration

In Prowlarr → Settings → Indexers → Add Proxy:

| Field | Value |
|-------|-------|
| Name | FlareSolverr |
| Type | FlareSolverr |
| Host | `http://flaresolverr:8191` |
| Tag | `flaresolverr` |

Indexers that require FlareSolverr are tagged with `flaresolverr` in Prowlarr. Only tagged indexers route through it.

---

## Notes

- No authentication — internal only, not reachable outside the proxy network
- Stateless — no appdata directory needed, no backup required
- If FlareSolverr goes down, only Cloudflare-protected indexers are affected — unprotected indexers continue working normally