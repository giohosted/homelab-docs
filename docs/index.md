# homelab-docs

Welcome to the Homelab v3.0 documentation site.

This is a living knowledge base documenting the full design, build, and operation of a home infrastructure stack — including networking, storage, compute, services, and runbooks.

---

## Quick Links

- [[00-roadmap|Roadmap]] — Full phased build roadmap and architecture overview
    
- [[architecture/hardware/inventory|Hardware Inventory]] — Every device, specs, and role
    
- [[architecture/hardware/rack-layout|Rack Layout]] — Physical rack diagram and cabling
    
- [[architecture/decisions-log|Decisions Log]] — Why every major decision was made
    

---

## Operations & Monitoring

### AI Observer

- [[services/infra/ai-observer|AI Observer — Architecture & Configuration]] — Dedicated observer VM, read-only Proxmox MCP integration, OpenAI Secure MCP Tunnel, networking, and security
    
- [[runbooks/ai-observer-operations|AI Observer — Operations & Recovery]] — Health checks, maintenance, troubleshooting, credentials, backups, and disaster recovery
    

### Backup & Recovery

- [[runbooks/backup-schedule|Backup Schedule]] — Backup schedules, storage destinations, coverage, verification, and known issues
    
- [[runbooks/backup-restore|Backup & Restore Runbook]] — Procedures for recovering VMs, containers, application data, and services
    

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