# homelab-docs

Welcome to the Homelab v3.0 documentation site.

This is a living knowledge base documenting the full design, build, and operation of a home infrastructure stack — including networking, storage, compute, services, and runbooks.

---

## Quick Links

- [Roadmap](app://-/00-roadmap.md) — Full phased build roadmap and architecture overview
    
- [Hardware Inventory](app://-/architecture/hardware/inventory.md) — Every device, specs, and role
    
- [Rack Layout](app://-/architecture/hardware/rack-layout.md) — Physical rack diagram and cabling
    
- [Decisions Log](app://-/architecture/decisions-log.md) — Why every major decision was made
    

---

## Operations & Monitoring

### AI Observer

- [AI Observer — Architecture & Configuration](app://-/services/infra/ai-observer.md) — Dedicated observer VM, read-only Proxmox MCP integration, OpenAI Secure MCP Tunnel, networking, and security
    
- [AI Observer — Operations & Recovery](app://-/runbooks/ai-observer-operations.md) — Health checks, maintenance, troubleshooting, credentials, backups, and disaster recovery
    

### Backup & Recovery

- [Backup Schedule](app://-/runbooks/backup-schedule.md) — Backup schedules, storage destinations, coverage, verification, and known issues
    
- [Backup & Restore Runbook](app://-/runbooks/backup-restore.md) — Procedures for recovering VMs, containers, application data, and services
    

---

## Structure

|Section|Contents|
|---|---|
|**Architecture**|Compute, hardware, network, and storage design docs|
|**Services**|Per-service documentation and runbooks|
|**Phases**|Chronological build log — one file per phase|
|**Runbooks**|Step-by-step operational procedures|
|**Kubernetes**|Phase 6 placeholder — populated later|

---

_Homelab v3.0 — Built intentionally._