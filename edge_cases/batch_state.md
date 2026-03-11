# 🔄 Edge Cases — Batch State Tracker

> This file tracks iterative generation of edge cases per topic.  
> **To continue:** Reply `continue` or `continue <topic>` to resume from the last checkpoint.  
> Last updated: 2025-07-15

---

## Overall Progress

| # | Topic | File | Edge Cases | Status | Last Line |
|---|-------|------|-----------|--------|-----------|
| 1 | Linux Admin | `linux_edge_cases.md` | 110 | ✅ Complete | — |
| 2 | Computer Networking | `networking_edge_cases.md` | 110 | ✅ Complete | — |
| 3 | Cloud & Data Centers | `cloud_edge_cases.md` | 110 | ✅ Complete | — |
| 4 | Git | `git_edge_cases.md` | 110 | ✅ Complete | — |
| 5 | Nginx | `nginx_edge_cases.md` | 110 | ✅ Complete | — |
| 6 | Apache | `apache_edge_cases.md` | 110 | ✅ Complete | — |
| 7 | IIS | `iis_edge_cases.md` | 0 | ⏳ Pending | — |
| 8 | Windows Server | `windows_server_edge_cases.md` | 0 | ⏳ Pending | — |

**Total edge cases so far:** 660 / 880+ target

---

## Batch 1 — Linux Admin (✅ Complete)

**File:** `edge_cases/linux_edge_cases.md`  
**Edge case count:** 110  
**Categories covered:**
- EC-001–015: File System & Disk
- EC-016–025: Process & Job Management
- EC-026–035: Users, Permissions & Security
- EC-036–045: Networking
- EC-046–055: SSH & Remote Access
- EC-056–065: systemd & Services
- EC-066–075: Package Management
- EC-076–085: Shell & Scripting
- EC-086–095: Performance & Resource Management
- EC-096–105: LVM, RAID & Storage
- EC-106–110: Kernel & Boot

**Status:** ✅ All 110 edge cases written, ASCII diagrams included, production FAQ at end.

---

## Batch 2 — Computer Networking (✅ Complete)

**File:** `edge_cases/networking_edge_cases.md`
**Edge case count:** 110
**Categories covered:**
- EC-001–015: TCP/IP & Socket Layer
- EC-016–030: DNS & Name Resolution
- EC-031–045: Firewall & iptables/nftables
- EC-046–058: TLS/SSL & Certificates
- EC-059–070: Load Balancers & Proxies
- EC-071–085: Container & Kubernetes Networking
- EC-086–095: VPN & Tunneling
- EC-096–105: Network Performance & Diagnostics
- EC-106–110: Cloud Networking
---
**Status:** ✅ All 110 edge cases written, ASCII diagrams included, production FAQ at end.

---

## Batch 3 — Cloud & Data Centers (✅ Complete)

**File:** `edge_cases/cloud_edge_cases.md`
**Edge case count:** 110
**Categories covered:**
- EC-001–015: IAM & Security
- EC-016–025: Compute (EC2/VMs)
- EC-026–035: Storage (S3/Blob/Object)
- EC-036–045: Databases (RDS/Cloud SQL)
- EC-046–055: VPC & Networking
- EC-056–065: Serverless & FaaS
- EC-066–075: Containers (ECS/EKS/GKE)
- EC-076–085: Infrastructure as Code (Terraform)
- EC-086–095: Monitoring & Observability
- EC-096–105: Cost & FinOps
- EC-106–110: Disaster Recovery

**Status:** ✅ All 110 edge cases written, ASCII diagrams included, production FAQ at end.

---

## Batch 4 — Git (✅ Complete)

**File:** `edge_cases/git_edge_cases.md`
**Edge case count:** 110
**Categories covered:**
- EC-001–015: Repository & Object Storage
- EC-016–025: Branching & Merging
- EC-026–035: Rebase & History Rewriting
- EC-036–045: Remote Operations & Authentication
- EC-046–055: Submodules & Subtrees
- EC-056–065: Hooks & Automation
- EC-066–075: Large Files & Performance
- EC-076–085: Workflow & Collaboration
- EC-086–095: CI/CD Integration
- EC-096–105: Recovery & Disaster
- EC-106–110: Advanced / Internals

**Status:** ✅ All 110 edge cases written, ASCII diagrams included, production FAQ at end.

---

## Batch 5 — Nginx (✅ Complete)

**File:** `edge_cases/nginx_edge_cases.md`
**Edge case count:** 110
**Categories covered:**
- EC-001–015: Configuration Syntax & Parsing
- EC-016–030: Reverse Proxy & Upstream
- EC-031–045: SSL / TLS & Certificates
- EC-046–058: Headers, Buffers & Body Handling
- EC-059–070: Caching
- EC-071–082: Rate Limiting & Access Control
- EC-083–090: Logging & Monitoring
- EC-091–100: Performance & Tuning
- EC-101–110: Containers, K8s & Cloud

**Status:** ✅ All 110 edge cases written, ASCII diagrams included, production FAQ at end.

---

## Batch 6 — Apache (✅ Complete)

**File:** `edge_cases/apache_edge_cases.md`
**Edge case count:** 110
**Categories covered:**
- EC-001–015: .htaccess & Per-Directory Config
- EC-016–030: mod_rewrite & URL Manipulation
- EC-031–042: Virtual Hosts & Server Configuration
- EC-043–055: SSL / TLS & Certificates
- EC-056–065: Modules & Extensions
- EC-066–078: MPM, Performance & Tuning
- EC-079–090: Reverse Proxy & mod_proxy
- EC-091–098: Authentication & Access Control
- EC-099–105: Logging, Monitoring & Troubleshooting
- EC-106–110: Containers, CI/CD & Cloud

**Status:** ✅ All 110 edge cases written, ASCII diagrams included, production FAQ at end.

---

## Batch 7 — IIS (⏳ Pending)

**Planned categories:** App pools, bindings, SSL, ARR, URL rewrite, auth, Windows auth, permissions, handlers  
**To start:** Reply `continue iis`

---

## Batch 8 — Windows Server (⏳ Pending)

**Planned categories:** AD, GPO, DNS, DHCP, PowerShell, WMI, registry, file system, networking, certificates  
**To start:** Reply `continue windows`

---

## Format Reference

Each edge case in any file follows this structure:

```
### EC-NNN — Title
**Category:** X | **Severity:** Critical/High/Medium/Low | **Env:** Prod/Dev/Both

**Scenario:** What triggers this edge case

**What breaks:** What symptom you observe

**Detection:**
  command or query

**Fix:**
  command or resolution

> ⚠️ **DevOps Gotcha:** Key warning or lesson
```

ASCII diagrams, comparison tables, and production FAQ sections are included where relevant.
