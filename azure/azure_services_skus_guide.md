# 🔷 Azure Services, SKUs & Decision Guide

> When to use which service, which SKU, and approximate costs (East US, July 2025).
> Costs are estimates — always verify on Azure Pricing Calculator.

---

## 1. Compute

### Virtual Machines

```
SERIES  │ USE CASE                        │ EXAMPLE SKU      │ ~COST/mo (PAYG)
────────┼─────────────────────────────────┼──────────────────┼─────────────────
B-series│ Dev/test, low-traffic, burstable│ B2s (2vCPU 4GB)  │ $30
        │ DON'T: sustained CPU workloads  │ B2ms (2vCPU 8GB) │ $60
        │ WHY: accumulates credits when   │                  │
        │ idle, bursts when needed        │                  │
────────┼─────────────────────────────────┼──────────────────┼─────────────────
D-series│ General purpose, web servers,   │ D2s_v5 (2vCPU 8GB)│ $70
        │ APIs, app servers               │ D4s_v5 (4vCPU 16GB)│$140
        │ DON'T: memory-heavy or GPU work │ D8s_v5 (8vCPU 32GB)│$280
        │ WHY: balanced CPU/RAM ratio     │                  │
────────┼─────────────────────────────────┼──────────────────┼─────────────────
E-series│ Memory-optimized: databases,    │ E2s_v5 (2vCPU 16GB)│ $90
        │ in-memory caching, SAP          │ E4s_v5 (4vCPU 32GB)│$180
        │ DON'T: compute-heavy workloads  │                  │
        │ WHY: high RAM-to-CPU ratio      │                  │
────────┼─────────────────────────────────┼──────────────────┼─────────────────
F-series│ CPU-optimized: batch processing,│ F2s_v2 (2vCPU 4GB)│ $60
        │ gaming servers, ML inference    │ F4s_v2 (4vCPU 8GB)│$120
        │ DON'T: RAM-heavy workloads      │                  │
────────┼─────────────────────────────────┼──────────────────┼─────────────────
L-series│ Storage-optimized: big data,    │ L8s_v3 (8vCPU 64GB)│$500
        │ Elasticsearch, Cassandra        │ NVMe local SSDs  │
        │ DON'T: general web apps         │                  │
────────┼─────────────────────────────────┼──────────────────┼─────────────────
N-series│ GPU: ML training, rendering,    │ NC6s_v3 (6vCPU   │ $660
        │ video processing                │ 112GB, 1xV100)   │

COST SAVINGS:
├── Reserved Instance (1yr): ~35% off PAYG
├── Reserved Instance (3yr): ~55% off PAYG
├── Spot VM: up to 90% off (can be evicted anytime — batch only)
├── Azure Hybrid Benefit: use existing Windows/SQL licenses
└── Dev/Test pricing: no Windows license cost in dev subs
```

**Decision: VM vs App Service vs AKS vs Container Instance?**
```
┌──────────────────────────────────────────────────────────────┐
│ Need full OS control, custom software, legacy app?  → VM    │
│ Simple web app, no infra management wanted?    → App Service │
│ Microservices, auto-scaling, containers?       → AKS        │
│ Short-lived task, batch job, sidecar?    → Container Instance│
│ Event-driven, pay-per-execution?         → Functions         │
└──────────────────────────────────────────────────────────────┘
```

### App Service

```
TIER         │ USE CASE                     │ FEATURES                 │ ~COST/mo
─────────────┼──────────────────────────────┼──────────────────────────┼──────────
Free (F1)    │ Testing, hello world         │ 60min CPU/day, no SLA    │ $0
Shared (D1)  │ Dev/test only                │ Custom domain, no SLA    │ $10
Basic (B1)   │ Low-traffic dev/staging      │ Custom domain, SSL, 1.75GB│$13
             │ DON'T: production            │ No autoscale, no slots   │
Standard (S1)│ Production web apps          │ Autoscale, slots, VNet   │ $70
             │ Most common for production   │ Daily backups, 50GB      │
Premium v3   │ High-perf production         │ Faster CPU, more memory  │ $115+
(P1v3)       │ VNet integration required    │ Private endpoint support │
Isolated v2  │ Compliance, dedicated env    │ Dedicated ASE, full VNet │ $350+
(I1v2)       │ DON'T: unless compliance     │ isolation, highest SLA   │
             │ requires dedicated infra     │                          │

WHEN App Service vs VM:
├── App Service: web apps, REST APIs, no OS-level access needed
├── VM: need custom OS config, background services, specific ports
└── App Service wins: auto-patching, built-in CI/CD, managed SSL
```

### Azure Kubernetes Service (AKS)

```
TIER        │ USE CASE                      │ FEATURES                │ ~COST/mo
────────────┼───────────────────────────────┼─────────────────────────┼──────────
Free        │ Dev/test, learning            │ No SLA, no uptime guar. │ $0 (nodes only)
Standard    │ Production workloads          │ 99.95% SLA (AZ), managed│ $0.10/hr
            │                               │ control plane uptime    │ (~$73/mo)
Premium     │ Mission-critical              │ 99.95%+, longer support │ $0.60/hr
            │                               │ cycles, advanced scaling│ (~$438/mo)

COST: AKS control plane + node VM costs.
3x D4s_v5 nodes in Standard tier ≈ $73 + (3 × $140) = ~$493/mo

WHEN AKS vs App Service:
├── AKS: microservices, multi-container pods, custom networking,
│         need Helm/K8s ecosystem, team has K8s skills
├── App Service: single app, want zero infrastructure management,
│                small team, no container orchestration needed
└── AKS wins: granular scaling, pod-level networking, sidecar pattern
```

### Container Instances (ACI)

```
USE CASE: Run a container without managing infrastructure.
         Short-lived tasks, batch jobs, CI/CD agents, sidecars.
DON'T:   Long-running production services (use AKS or App Service).

COST: per vCPU-second + per GB-second (truly pay-per-use)
  1 vCPU, 1.5GB, running 1 hour ≈ $0.05
  Running 24/7 for a month ≈ $36 (at that point use a VM)
```

### Azure Functions

```
PLAN            │ USE CASE                    │ ~COST
────────────────┼─────────────────────────────┼────────────────────
Consumption     │ Event-driven, sporadic      │ First 1M exec free
                │ Auto-scales to zero         │ $0.20/million after
                │ DON'T: latency-sensitive    │ + $0.000016/GB-s
                │ (cold start 1-5s)           │
────────────────┼─────────────────────────────┼────────────────────
Premium (EP1)   │ No cold start, VNet needed  │ ~$155/mo (always-on)
                │ Pre-warmed instances        │
────────────────┼─────────────────────────────┼────────────────────
Dedicated       │ Already have App Service    │ Runs on your ASP
                │ Plan with spare capacity    │ (no extra cost)

WHEN Functions vs App Service:
├── Functions: event-driven (queue trigger, HTTP, timer, blob trigger)
├── App Service: long-running web server, persistent connections
└── Functions wins: auto-scale to zero, per-execution billing
```

---

## 2. Databases

### Azure SQL

```
MODEL               │ USE CASE                     │ ~COST/mo
────────────────────┼──────────────────────────────┼────────────
SQL Database        │ Single database, SaaS apps   │
  ├── Basic (5 DTU) │ Dev/test only                │ $5
  ├── Standard S0   │ Small production             │ $15
  ├── Standard S3   │ Medium workloads             │ $150
  ├── Premium P1    │ High IOPS, in-memory         │ $465
  └── Serverless    │ Intermittent/unpredictable   │ Auto-pause,
                    │ Auto-pauses when idle        │ pay per vCore-sec
────────────────────┼──────────────────────────────┼────────────
SQL Managed Instance│ Lift-and-shift on-prem SQL   │ $350+ (GP 4vCore)
                    │ Full SQL Server compatibility│
                    │ DON'T: new cloud-native apps │
────────────────────┼──────────────────────────────┼────────────
SQL Server on VM    │ Full control, custom config  │ VM cost + license
                    │ Legacy features, SQL Agent   │

DTU vs vCore model:
├── DTU: bundled (CPU+IO+memory), simpler, good for predictable workloads
├── vCore: choose CPU/memory independently, bring your own license
└── Start with DTU for simplicity, move to vCore when you need control

WHEN SQL vs PostgreSQL vs Cosmos DB:
├── Azure SQL: existing SQL Server apps, .NET ecosystem, enterprise features
├── PostgreSQL: open-source preference, JSON support, PostGIS, cost-sensitive
└── Cosmos DB: global distribution, multi-model, single-digit-ms latency
```

### Azure Database for PostgreSQL

```
DEPLOYMENT       │ USE CASE                      │ ~COST/mo
─────────────────┼───────────────────────────────┼────────────
Flexible Server  │ All new PostgreSQL workloads  │
  ├── Burstable  │ Dev/test, low traffic         │
  │  B1ms 1vC 2GB│                               │ $13
  │  B2s  2vC 4GB│                               │ $26
  ├── Gen Purpose │ Production web apps           │
  │  D2s  2vC 8GB│                               │ $100
  │  D4s  4vC 16G│                               │ $200
  └── Mem Optimzd│ Heavy analytics, large DBs    │
     E2s  2vC 16G│                               │ $130
     E4s  4vC 32G│                               │ $260
─────────────────┼───────────────────────────────┼────────────
Single Server    │ DEPRECATED — do not use       │ Migrating out
(legacy)         │ Migrate to Flexible Server    │

Storage: $0.115/GB/mo (first 32GB included in some tiers)
Backup: Included up to 100% of server storage

WHEN Burstable vs General Purpose:
├── Burstable: dev/test, apps with idle periods, < 50 connections
├── General Purpose: production, steady traffic, 50-500 connections
└── Memory Optimized: analytics, large working sets, in-memory caching
```

### Cosmos DB

```
MODEL            │ USE CASE                       │ ~COST/mo
─────────────────┼────────────────────────────────┼────────────
Provisioned RU   │ Predictable workloads          │ 400 RU/s = $23
                 │ (set capacity in advance)      │ 4000 RU/s = $233
                 │                                │ per container
─────────────────┼────────────────────────────────┼────────────
Autoscale RU     │ Variable traffic (scales 1-10x)│ Max 4000 RU/s=$350
                 │                                │ (charged per actual)
─────────────────┼────────────────────────────────┼────────────
Serverless       │ Dev/test, sporadic traffic     │ $0.25/million RU
                 │ DON'T: sustained high traffic  │ + $0.25/GB storage

WHEN Cosmos DB vs SQL vs PostgreSQL:
├── Cosmos DB: global distribution needed, <10ms latency SLA,
│              multi-model (document/graph/key-value), massive scale
├── DON'T use Cosmos DB when: relational queries, complex JOINs,
│   cost-sensitive (Cosmos is expensive at scale)
└── PostgreSQL/SQL: relational data, complex queries, cost-sensitive
```

---

## 3. Networking

### Load Balancers

```
SERVICE              │ LAYER  │ USE CASE                    │ ~COST/mo
─────────────────────┼────────┼─────────────────────────────┼──────────
Load Balancer Basic  │ L4     │ DON'T USE — being retired   │ Free
Load Balancer Standard│ L4    │ TCP/UDP load balancing       │ $18 + rules
                     │        │ Internal or public           │
─────────────────────┼────────┼─────────────────────────────┼──────────
Application Gateway  │ L7     │ HTTP/HTTPS, path routing,   │ $145+ (v2)
  ├── Standard_v2    │        │ cookie affinity, SSL offload│
  └── WAF_v2         │        │ + Web Application Firewall  │ $245+
─────────────────────┼────────┼─────────────────────────────┼──────────
Front Door Standard  │ L7     │ Global HTTP load balancing, │ $35 base
                     │ Global │ CDN, WAF, geo-routing       │ + per request
Front Door Premium   │        │ + Private Link origin,      │ $330 base
                     │        │   advanced WAF, bot prot.   │

DECISION:
├── Internal TCP/UDP balancing (VMs, ports)     → Load Balancer Standard
├── HTTP routing, SSL termination, single region→ Application Gateway
├── HTTP + WAF (OWASP protection)               → AppGW WAF_v2
├── Global HTTP, CDN, multi-region, DDoS        → Front Door
└── Global TCP/UDP (gaming, IoT)                → Traffic Manager (DNS-based)
```

### Azure Firewall

```
TIER        │ USE CASE                          │ ~COST/mo
────────────┼───────────────────────────────────┼──────────
Standard    │ L3-L7 filtering, FQDN filtering  │ $912 + data
            │ Threat intel, NAT rules           │
Premium     │ TLS inspection, IDPS, URL filter  │ $1,825 + data
            │ Compliance workloads              │
Basic       │ Small/SMB, <250Mbps throughput    │ $274 + data
            │ DON'T: high throughput or TLS     │

WHEN Firewall vs NSG:
├── NSG: subnet/NIC level, IP+port rules only, free, always use
├── Firewall: centralized, FQDN filtering, TLS inspection, logging
└── Use BOTH: NSG on every subnet + Firewall in hub for egress control
```

### VPN Gateway vs ExpressRoute

```
SERVICE         │ USE CASE                        │ ~COST/mo
────────────────┼─────────────────────────────────┼──────────
VPN GW Basic    │ DON'T — no SLA, legacy          │ $27
VPN GW VpnGw1   │ S2S VPN, P2S VPN, dev/test     │ $138
VPN GW VpnGw2   │ Production, higher throughput   │ $360
VPN GW VpnGw3   │ High throughput (1.25Gbps)      │ $960
────────────────┼─────────────────────────────────┼──────────
ExpressRoute    │ Private connection to Azure     │ $55+/mo (port)
                │ (not over internet)             │ + circuit cost
                │ Sub-10ms latency, 50Mbps-10Gbps │ from ISP

WHEN VPN vs ExpressRoute:
├── VPN: budget-constrained, <1Gbps, okay with internet variability
├── ExpressRoute: compliance (data must not traverse internet),
│   consistent low latency, >1Gbps, mission-critical
└── Both can coexist: ExpressRoute primary, VPN as failover
```

---

## 4. Storage

```
TYPE                │ USE CASE                    │ ~COST/GB/mo
────────────────────┼─────────────────────────────┼────────────
Blob — Hot          │ Frequently accessed data    │ $0.018
Blob — Cool         │ Infrequent (30+ days)       │ $0.010
Blob — Cold         │ Rarely accessed (90+ days)  │ $0.0045
Blob — Archive      │ Long-term (180+ days)       │ $0.00099
                    │ DON'T: need instant access  │ (rehydrate takes hrs)
────────────────────┼─────────────────────────────┼────────────
Premium Block Blob  │ Low latency, high IOPS      │ $0.15
                    │ (analytics, AI, media)      │
────────────────────┼─────────────────────────────┼────────────
Azure Files Std     │ SMB/NFS file shares         │ $0.06
Azure Files Premium │ Low latency file shares     │ $0.16
                    │ (provisioned IOPS)          │
────────────────────┼─────────────────────────────┼────────────
Data Lake Gen2      │ Big data analytics          │ Same as Blob
                    │ Hierarchical namespace      │ + $0.065 per 10K ops
────────────────────┼─────────────────────────────┼────────────
Managed Disks:      │                             │
  Standard HDD      │ Dev/test, backup            │ $0.04/GB
  Standard SSD      │ Web servers, light prod     │ $0.075/GB
  Premium SSD (P10) │ Production databases, APIs  │ 128GB = $19
  Ultra Disk         │ SAP HANA, tier-1 DB         │ $0.12/GB + IOPS cost

REDUNDANCY:
├── LRS  (3 copies, 1 datacenter)      → Cheapest, no zone protection
├── ZRS  (3 copies, 3 zones)           → Zone-level HA, +25% cost
├── GRS  (6 copies, 2 regions)         → DR to paired region, +100% cost
└── GZRS (6 copies, 3 zones + 2nd region) → Best protection, most expensive
```

---

## 5. Security & Identity

```
SERVICE            │ USE CASE                        │ ~COST/mo
───────────────────┼─────────────────────────────────┼──────────
Key Vault Standard │ Secrets, keys, certificates     │ $0.03/10K ops
Key Vault Premium  │ HSM-backed keys (compliance)    │ $1/key/mo + ops
───────────────────┼─────────────────────────────────┼──────────
Defender for Cloud │                                 │
  Free tier        │ Security recommendations        │ $0
  Standard         │ Threat detection, JIT VM access │
    Servers        │ VM protection                   │ $15/server/mo
    App Service    │ App protection                  │ $15/instance/mo
    SQL            │ DB threat detection             │ $15/instance/mo
    Containers     │ AKS runtime protection          │ $7/vCore/mo
    Key Vault      │ Access anomaly detection        │ $0.02/10K ops
───────────────────┼─────────────────────────────────┼──────────
Azure Bastion      │ Secure RDP/SSH without public IP│
  Basic            │ Manual scaling, basic features  │ $139/mo
  Standard         │ Auto-scale, native client, IP   │ $277/mo
  DON'T:           │ open port 22/3389 to internet  │
───────────────────┼─────────────────────────────────┼──────────
DDoS Protection    │                                 │
  Network          │ VNet protection, auto-tuning    │ $2,944/mo (!!)
  IP               │ Per-IP protection (cheaper)     │ $199/mo per IP
```

---

## 6. Monitoring

```
SERVICE                 │ USE CASE                    │ ~COST
────────────────────────┼─────────────────────────────┼──────────
Azure Monitor Metrics   │ Auto-collected platform     │ FREE (93d retention)
                        │ metrics for all resources   │
────────────────────────┼─────────────────────────────┼──────────
Log Analytics Workspace │ Central log storage (KQL)   │ $2.76/GB ingested
                        │                             │ First 5GB/mo free
                        │                             │ Commitment tiers:
                        │                             │ 100GB/day = $196/day
────────────────────────┼─────────────────────────────┼──────────
Application Insights    │ APM for web apps (traces,   │ $2.76/GB (same as LAW)
                        │ requests, dependencies)     │ First 5GB free
────────────────────────┼─────────────────────────────┼──────────
Managed Grafana         │ Grafana dashboards (managed)│ Standard: $8.76/mo
────────────────────────┼─────────────────────────────┼──────────
Managed Prometheus      │ Prometheus metrics (managed)│ $0.30/million samples

COST TRAP: Log Analytics can get expensive fast.
├── 50 VMs logging everything = ~500GB/month = ~$1,380/mo
├── Solution: filter logs, set retention policies, use basic logs tier
└── Basic Logs: $0.65/GB (cheaper, limited KQL, no alerts)
```

---

## 7. CI/CD & DevOps

```
SERVICE                │ USE CASE                     │ ~COST
───────────────────────┼──────────────────────────────┼──────────
Azure Container Regstr │                              │
  Basic                │ Dev/test, personal           │ $5/mo, 10GB
  Standard             │ Production, most teams       │ $20/mo, 100GB
  Premium              │ Geo-replication, throughput  │ $50/mo, 500GB
                       │ Private endpoint support     │
───────────────────────┼──────────────────────────────┼──────────
Azure DevOps           │ Repos, Boards, Pipelines     │
  Basic (5 users)      │ Small teams                  │ Free
  Basic Plan           │ Per user after 5             │ $6/user/mo
  MS-hosted parallel   │ CI/CD pipeline agents        │ 1 free, $40/extra
  Self-hosted parallel │ Your own agents              │ 1 free, $15/extra
───────────────────────┼──────────────────────────────┼──────────
GitHub Enterprise      │ GitHub + GHAS (security)     │ $21/user/mo

WHEN ACR Basic vs Standard vs Premium:
├── Basic: personal projects, <5 devs, no geo-replication
├── Standard: production, team of 5-50, webhook integrations
└── Premium: multi-region, need private endpoint, >100 devs, compliance
```

---

## 8. Messaging & Integration

```
SERVICE             │ USE CASE                      │ ~COST/mo
────────────────────┼───────────────────────────────┼──────────
Service Bus Basic   │ Simple queues only            │ $0.05/million ops
Service Bus Standard│ Queues + Topics, prod ready   │ $10 base + ops
Service Bus Premium │ Dedicated, predictable perf   │ $668/unit
                    │ Large messages (100MB)        │
────────────────────┼───────────────────────────────┼──────────
Event Hub Basic     │ Streaming, telemetry          │ $11
Event Hub Standard  │ Consumer groups, capture      │ $22
Event Hub Premium   │ Dedicated, compliance         │ $862
────────────────────┼───────────────────────────────┼──────────
Storage Queue       │ Simple queue, no ordering     │ $0.0004/10K ops
                    │ DON'T: need ordering/topics   │

DECISION:
├── Simple task queue, no ordering needed    → Storage Queue (cheapest)
├── Reliable messaging, ordering, topics     → Service Bus Standard
├── High-throughput streaming (millions/sec) → Event Hub
├── Real-time push to clients (WebSocket)    → SignalR
└── Workflow orchestration                   → Logic Apps / Durable Functions
```

---

## 9. Quick Decision Matrix

```
┌────────────────────┬─────────────────────────────────────────┐
│ "I need to..."     │ Use this service                        │
├────────────────────┼─────────────────────────────────────────┤
│ Host a web app     │ App Service (Standard S1)               │
│ Run containers     │ AKS (Standard) or ACI (short tasks)    │
│ Run serverless     │ Functions (Consumption)                 │
│ Store files        │ Blob Storage (Hot/Cool based on access)│
│ Store relational   │ PostgreSQL Flexible or Azure SQL        │
│ Store NoSQL        │ Cosmos DB (Serverless for dev)          │
│ Cache data         │ Redis Cache (C1 Standard for prod)     │
│ Queue messages     │ Service Bus Standard (or Storage Queue)│
│ Stream events      │ Event Hub Standard                      │
│ Load balance HTTP  │ App Gateway v2 (single region)         │
│                    │ Front Door (multi-region/global)        │
│ Load balance TCP   │ Load Balancer Standard                  │
│ Connect on-prem    │ VPN Gateway (budget) or ExpressRoute   │
│ Store secrets      │ Key Vault (Standard)                   │
│ Monitor apps       │ Application Insights (free 5GB)        │
│ Centralize logs    │ Log Analytics Workspace                 │
│ Protect web apps   │ App Gateway WAF v2 or Front Door WAF  │
│ Secure VM access   │ Azure Bastion (Basic)                  │
│ Store containers   │ ACR Standard (or Premium for PE)       │
│ Run CI/CD          │ Azure DevOps or GitHub Actions          │
│ Manage DNS         │ Azure DNS (public) + Private DNS Zones │
│ Send emails        │ Azure Communication Services            │
│ Schedule tasks     │ Functions (Timer trigger)               │
└────────────────────┴─────────────────────────────────────────┘
```

---

## 10. Caching

### Azure Cache for Redis

```
TIER        │ USE CASE                          │ ~COST/mo
────────────┼───────────────────────────────────┼──────────
Basic C0    │ Dev/test only (250MB, no SLA)     │ $16
Basic C1    │ Dev/test (1GB, no SLA)            │ $34
────────────┼───────────────────────────────────┼──────────
Standard C0 │ Small production (250MB, replicated)│ $40
Standard C1 │ Production sessions/cache (1GB)   │ $68
Standard C2 │ Medium prod (2.5GB)               │ $135
Standard C3 │ Larger cache (6GB)                │ $270
────────────┼───────────────────────────────────┼──────────
Premium P1  │ Persistence, clustering, VNet     │ $225
            │ (6GB, geo-replication, zones)     │
Premium P3  │ Large datasets (120GB)            │ $1,500
────────────┼───────────────────────────────────┼──────────
Enterprise E10│ Redis modules (RediSearch, etc.) │ $690
Enterprise Flash│ Large datasets on flash+DRAM  │ $680 (100GB)

WHEN Basic vs Standard vs Premium:
├── Basic: dev/test ONLY — no replication, no SLA, will lose data on failure
├── Standard: production — has replica, 99.9% SLA, automatic failover
├── Premium: need VNet integration, persistence (RDB/AOF), clustering,
│            geo-replication, or >53GB cache size
└── Enterprise: need RediSearch, RedisJSON, RedisTimeSeries modules

GOTCHA: Basic → Standard upgrade is possible. Standard → Premium
        requires RECREATION (different architecture). Plan ahead.

DECISION: Redis vs Cosmos DB for caching:
├── Redis: in-memory speed, sub-millisecond, session store, pub/sub,
│          leaderboards, rate limiting, simple key-value
└── Cosmos DB: need global distribution, documents, >120GB, query by fields
```

---

## 11. API Management

```
TIER          │ USE CASE                         │ ~COST/mo
──────────────┼──────────────────────────────────┼──────────
Consumption   │ Serverless, pay-per-call         │ $3.50/million calls
              │ No SLA, cold starts              │ First 1M free
              │ DON'T: latency-sensitive         │
──────────────┼──────────────────────────────────┼──────────
Developer     │ Dev/test, no SLA                 │ $49
              │ DON'T: production                │
──────────────┼──────────────────────────────────┼──────────
Basic         │ Small prod (1,000 req/sec)       │ $151
              │ No VNet, no multi-region         │
──────────────┼──────────────────────────────────┼──────────
Standard      │ Production API gateway           │ $690
              │ Most common for production       │
              │ Azure AD auth, rate limiting     │
──────────────┼──────────────────────────────────┼──────────
Premium       │ Multi-region, VNet integration,  │ $2,794/unit
              │ self-hosted gateway, >99.95% SLA │

WHEN APIM vs direct exposure:
├── Use APIM: multiple backend APIs to unify, need rate limiting,
│   API versioning, OAuth gateway, developer portal, analytics
├── Skip APIM: single API, no external consumers, internal only
└── Use App Gateway instead: just need L7 routing + WAF, no API mgmt features
```

---

## 12. Content Delivery & DNS

### Azure CDN vs Front Door

```
SERVICE            │ USE CASE                       │ ~COST/mo
───────────────────┼────────────────────────────────┼──────────
Azure CDN Standard │ Static content caching         │ $0.081/GB (zone 1)
(Microsoft)        │ Simple acceleration            │ No base fee
───────────────────┼────────────────────────────────┼──────────
Azure CDN Akamai   │ BEING RETIRED — do not use     │
Azure CDN Verizon  │ BEING RETIRED — do not use     │
───────────────────┼────────────────────────────────┼──────────
Front Door Standard│ CDN + global LB + routing      │ $35 base
                   │ Custom domains, compression    │ + $0.081/GB
Front Door Premium │ + WAF, Private Link origins,   │ $330 base
                   │ bot protection, enhanced rules │ + $0.081/GB

DECISION:
├── Static site/images only, no routing logic  → Azure CDN Standard
├── Global load balancing + CDN + WAF          → Front Door Standard
├── Need private backends (Private Link)       → Front Door Premium
└── Front Door is replacing Azure CDN — prefer Front Door for new projects
```

### Azure DNS

```
SERVICE               │ USE CASE                    │ ~COST/mo
──────────────────────┼─────────────────────────────┼──────────
Azure DNS (public)    │ Host public DNS zones       │ $0.50/zone
                      │ Route53 equivalent          │ + $0.40/million queries
──────────────────────┼─────────────────────────────┼──────────
Private DNS Zones     │ DNS inside VNets            │ $0.25/zone
                      │ Required for private        │
                      │ endpoints (privatelink.*)   │
──────────────────────┼─────────────────────────────┼──────────
Traffic Manager       │ DNS-based global routing    │ $0.54/million queries
                      │ (failover, performance,     │
                      │ weighted, geographic)       │

DECISION: Traffic Manager vs Front Door:
├── Traffic Manager: DNS-level routing (any protocol), cheaper, simpler
│   DON'T: need WAF, caching, or SSL termination
├── Front Door: HTTP-level routing, CDN, WAF, SSL offload
│   More expensive but more capable
└── Both can do geo-routing and failover — Front Door is faster (anycast vs DNS TTL)
```

---

## 13. Identity & Governance

### Azure AD (Entra ID) Licensing

```
TIER          │ FEATURES                            │ ~COST/user/mo
──────────────┼─────────────────────────────────────┼──────────
Free          │ Basic SSO, MFA (security defaults)  │ $0
              │ 50,000 objects, basic reports        │
──────────────┼─────────────────────────────────────┼──────────
P1            │ Conditional Access, PIM,             │ $6
              │ Self-service password reset,         │
              │ Dynamic groups, App Proxy            │
              │ MOST COMMON for enterprise           │
──────────────┼─────────────────────────────────────┼──────────
P2            │ Identity Protection (risk-based      │ $9
              │ conditional access), Access Reviews, │
              │ Entitlement Management               │
              │ For compliance-heavy environments    │

WHEN P1 vs P2:
├── P1: need Conditional Access + PIM — sufficient for most orgs
├── P2: need risk-based sign-in detection, access review automation
└── Free: startup with <50 users, basic MFA via security defaults
```

### Management Groups & Subscriptions

```
Subscription limits to know:
├── 800 resource groups per subscription
├── vCPU quotas per region per subscription (default 20, increase via support)
├── 500 App Service Plans per subscription
├── 250 Storage Accounts per region per subscription
├── 10,000 NSG rules per subscription

WHEN to create a new subscription:
├── Hitting resource limits
├── Need billing separation (per client/department)
├── Need different policy enforcement (dev vs prod)
├── Need different RBAC boundaries (team isolation)
└── Compliance requires workload isolation
```

---

## 14. Data & Analytics

```
SERVICE                │ USE CASE                       │ ~COST/mo
───────────────────────┼────────────────────────────────┼──────────
Azure Data Factory     │ ETL/ELT pipelines, data        │ $0.25/activity run
                       │ movement, orchestration        │ + $0.25/DIU-hr
                       │ 1000 runs/day ≈ $250/mo       │
───────────────────────┼────────────────────────────────┼──────────
Azure Synapse Analytics│ Data warehousing, big SQL      │
  Serverless SQL       │ Query on-demand (pay per TB)   │ $5/TB scanned
  Dedicated SQL Pool   │ Predictable performance        │ $1.20/DWU/hr
                       │ (100DWU = $870/mo)             │
  Spark Pool           │ Apache Spark workloads         │ Per node-hour
───────────────────────┼────────────────────────────────┼──────────
Azure Databricks       │ ML, analytics, Spark           │
  Standard             │ Basic notebooks + clusters     │ DBU: $0.07-0.55/hr
  Premium              │ RBAC, audit, Unity Catalog     │ + VM cost
───────────────────────┼────────────────────────────────┼──────────
Azure Stream Analytics │ Real-time stream processing    │ $0.11/SU/hr
                       │ IoT, real-time dashboards      │ (1SU = 1vCPU)

DECISION:
├── Batch ETL (move/transform data): Data Factory
├── SQL analytics on data lake: Synapse Serverless
├── Dedicated data warehouse: Synapse Dedicated Pool
├── ML training, complex Spark: Databricks Premium
└── Real-time processing (IoT/streaming): Stream Analytics or Event Hub + Functions
```

---

## 15. AI & Cognitive Services

```
SERVICE                  │ USE CASE                    │ ~COST
─────────────────────────┼─────────────────────────────┼──────────
Azure OpenAI Service     │ GPT-4, embeddings, DALL-E   │
  GPT-4o                 │ Text generation             │ $2.50/1M input tokens
  GPT-4o-mini            │ Cheaper text generation     │ $0.15/1M input tokens
  text-embedding-3-small │ Vector embeddings           │ $0.02/1M tokens
─────────────────────────┼─────────────────────────────┼──────────
Azure AI Search          │ Full-text + vector search   │
  Free                   │ Dev/test (50MB, 3 indexes)  │ $0
  Basic                  │ Small production            │ $75
  Standard S1            │ Production search           │ $250
  Standard S2            │ Large datasets              │ $1,000
─────────────────────────┼─────────────────────────────┼──────────
Form Recognizer          │ Document extraction (OCR+)  │ $1/1000 pages (prebuilt)
Computer Vision          │ Image analysis              │ $1/1000 images
Speech Services          │ Speech-to-text, text-speech │ $1/hr (STT)
Translator               │ Text translation            │ $10/million chars

GOTCHA: Azure OpenAI requires application approval.
        Not all regions have all models. Check availability first.
```

---

## 16. Cost Optimization Strategies

```
┌──────────────────────────────────────────────────────────────┐
│  COST SAVINGS SUMMARY                                        │
│                                                              │
│  1. Reserved Instances (1yr or 3yr commitment)              │
│     ├── VMs: 35-55% savings                                │
│     ├── SQL Database: 30-50% savings                       │
│     ├── Cosmos DB: up to 65% savings                       │
│     ├── App Service: 30-50% savings                        │
│     └── Redis Cache: up to 55% savings                     │
│                                                              │
│  2. Spot VMs (evictable, batch workloads only)              │
│     └── Up to 90% savings                                  │
│                                                              │
│  3. Auto-shutdown dev/test VMs                              │
│     └── az vm auto-shutdown --time 1900 (7PM)              │
│                                                              │
│  4. Right-size VMs (check Azure Advisor recommendations)    │
│     └── Many VMs run at <10% CPU — downsize to B-series    │
│                                                              │
│  5. Azure Hybrid Benefit                                     │
│     ├── Windows: use existing licenses → save ~40%         │
│     └── SQL Server: use existing licenses → save ~55%      │
│                                                              │
│  6. Dev/Test subscription pricing                            │
│     └── No Windows license cost, discounted services        │
│                                                              │
│  7. Storage lifecycle policies                               │
│     └── Auto-move blobs: Hot → Cool → Cold → Archive       │
│                                                              │
│  8. Serverless where possible                                │
│     ├── Functions Consumption: pay only when running        │
│     ├── Cosmos DB Serverless: auto-pause                    │
│     └── Azure SQL Serverless: auto-pause after idle         │
│                                                              │
│  9. Budget alerts (ALWAYS set these)                        │
│     az consumption budget create --amount 5000              │
│       --time-grain Monthly --category Cost                  │
│       --notifications "80=email:ops@company.com"            │
│                                                              │
│  10. Azure Advisor → check weekly for recommendations       │
│      ├── Right-size or shut down underutilized VMs         │
│      ├── Delete orphaned disks, IPs, NICs                  │
│      └── Purchase reservations for steady workloads        │
└──────────────────────────────────────────────────────────────┘
```

---

## 17. SLA Quick Reference

```
SERVICE                          │ SLA
─────────────────────────────────┼─────────────
VM (single, Premium SSD)         │ 99.9%
VM (Availability Set)            │ 99.95%
VM (Availability Zone)           │ 99.99%
App Service (Standard+)          │ 99.95%
AKS (Standard tier + AZ)        │ 99.95%
Azure SQL Database               │ 99.99%
Cosmos DB (single region)        │ 99.99%
Cosmos DB (multi-region)         │ 99.999%
Azure Functions (Consumption)    │ 99.95%
Storage (LRS)                    │ 99.9%
Storage (ZRS)                    │ 99.9999999999% (durability)
Key Vault                        │ 99.99%
Azure Cache for Redis Standard   │ 99.9%
Azure Cache for Redis Premium    │ 99.95% (zone redundant)
Load Balancer Standard           │ 99.99%
Application Gateway v2           │ 99.95%
Front Door                       │ 99.99%
VPN Gateway (VpnGw1+)           │ 99.95%
ExpressRoute                     │ 99.95%
Azure DNS                        │ 100%

COMPOSITE SLA = product of all component SLAs.
App Service (99.95%) × SQL (99.99%) × Redis (99.9%) = 99.84%
Every component you add REDUCES the composite SLA.
```

---

## 18. Backup & Disaster Recovery

```
SERVICE                  │ USE CASE                      │ ~COST/mo
─────────────────────────┼───────────────────────────────┼──────────
Recovery Services Vault  │ VM backup, file recovery,     │
  VM Backup              │ SQL in VM backup              │ $5/instance
                         │ + $0.10/GB protected data     │
  Azure File Share backup│ Snapshot-based file backup    │ Storage cost only
─────────────────────────┼───────────────────────────────┼──────────
Azure Site Recovery (ASR)│ VM replication to DR region   │ $25/instance/mo
                         │ Orchestrated failover         │ + storage + compute
                         │ RPO: continuous, RTO: minutes │ in DR region
─────────────────────────┼───────────────────────────────┼──────────
Azure Backup for:        │                               │
  PostgreSQL Flexible    │ Built-in, configurable        │ Included in service
                         │ Retention: 7-35 days (auto)   │ (long-term: storage)
  Azure SQL              │ Built-in automated            │ Included in service
                         │ PITR up to 35 days            │ (LTR: storage cost)
  AKS (Velero/native)   │ etcd + PV backup              │ Storage cost only
  Blob (soft delete)     │ Point-in-time restore         │ Storage cost only
                         │ Retention: 1-365 days         │

WHEN Backup vs ASR:
├── Backup: protect data (point-in-time restore, file-level recovery)
│   Recover individual files, databases, VMs from backups
├── ASR: protect entire environment (replicate to another region)
│   Failover entire workload in minutes during region outage
└── Use BOTH: Backup for data protection + ASR for disaster recovery

DR STRATEGY TIERS:
├── Tier 1 (cheapest): Backup only → restore from backup (RTO: hours)
├── Tier 2 (balanced): ASR warm standby → failover (RTO: 15-30 min)
├── Tier 3 (expensive): Active-active multi-region (RTO: <1 min)
└── Choose tier based on business impact and budget
```

---

## 19. Container Ecosystem — Full Picture

```
SERVICE                   │ USE CASE                     │ ~COST/mo
──────────────────────────┼──────────────────────────────┼──────────
AKS                       │ Full K8s orchestration       │ See §3
Container Instances (ACI) │ Serverless single containers │ See §3
Container Apps            │ Serverless containers with   │
                          │ KEDA scaling, Dapr, revisions│
  Consumption             │ Scale to zero, event-driven  │ vCPU-s + GB-s
  Dedicated               │ Predictable perf, workload   │ D4 node ≈ $100
                          │ profiles                     │
──────────────────────────┼──────────────────────────────┼──────────
Container Registry (ACR)  │ See §7                       │

WHEN Container Apps vs AKS vs ACI:
┌──────────────────────────────────────────────────────────────┐
│ Need full K8s control, Helm, custom operators? → AKS        │
│ Event-driven microservices, no K8s expertise?  → Container Apps│
│ One-off task, batch job, CI agent?             → ACI        │
│ Single web app, no containers expertise?       → App Service │
│                                                              │
│ Container Apps is the MIDDLE GROUND:                        │
│ ├── Runs on K8s under the hood (you don't manage it)       │
│ ├── Built-in Dapr for service-to-service communication     │
│ ├── KEDA-based autoscaling (scale on queue depth, HTTP, etc)│
│ ├── Revision management (traffic splitting, blue-green)    │
│ ├── Scale to zero (no cost when idle)                      │
│ └── Simpler than AKS, more capable than App Service        │
│                                                              │
│ DON'T use Container Apps when:                              │
│ ├── Need DaemonSets, StatefulSets, custom CRDs             │
│ ├── Need direct node access                                │
│ ├── Need custom CNI or network policies                    │
│ └── Already invested in Helm charts and K8s tooling        │
└──────────────────────────────────────────────────────────────┘
```

---

## 20. Notification & Communication

```
SERVICE                    │ USE CASE                    │ ~COST
───────────────────────────┼─────────────────────────────┼──────────
Azure Communication Svcs   │                             │
  Email                    │ Transactional emails        │ $0.25/1000 emails
  SMS                      │ OTP, notifications          │ $0.0075/msg (US)
  Chat                     │ In-app chat                 │ Free (1st 10K/mo)
  Voice/Video              │ Calling, video meetings     │ $0.004/min
───────────────────────────┼─────────────────────────────┼──────────
Notification Hub           │ Push notifications at scale │
  Free                     │ 1M pushes, 500 devices      │ $0
  Basic                    │ 10M pushes, 200K devices    │ $10
  Standard                 │ 10M pushes, unlimited       │ $200
───────────────────────────┼─────────────────────────────┼──────────
SignalR Service            │ Real-time WebSocket comms   │
  Free                     │ 20 connections, 20K msg/day │ $0
  Standard                 │ 1000 conn/unit              │ $49/unit
───────────────────────────┼─────────────────────────────┼──────────
Event Grid                 │ Event routing (reactive)    │
  Standard                 │ Azure resource events       │ $0.60/million ops
  Partner/Custom topics    │ Custom event-driven apps    │

DECISION:
├── Send emails from app: Communication Services (or SendGrid marketplace)
├── Push notifications (mobile): Notification Hub
├── Real-time updates (dashboard, chat): SignalR
├── React to Azure events (blob created, VM deleted): Event Grid
└── React to custom app events: Event Grid Custom Topics
```

---

## 21. Migration Services

```
SERVICE                    │ USE CASE                        │ ~COST
───────────────────────────┼─────────────────────────────────┼──────────
Azure Migrate              │ Discover + assess on-prem VMs   │ Free (assessment)
                           │ Migration hub for all tools     │ Pay for target infra
───────────────────────────┼─────────────────────────────────┼──────────
Database Migration Service │ Migrate databases to Azure      │
  (DMS) Standard           │ Offline migration               │ Free
  (DMS) Premium            │ Online (minimal downtime)       │ Per vCore pricing
───────────────────────────┼─────────────────────────────────┼──────────
Azure Data Box             │ Ship physical device to Azure   │
  Data Box Disk (8TB)      │ <40TB of data                   │ $80/order
  Data Box (100TB)         │ 40-500TB of data                │ $395/order
  Data Box Heavy (1PB)     │ >500TB of data                  │ $4,500/order
───────────────────────────┼─────────────────────────────────┼──────────
Azure File Sync            │ Sync on-prem file servers to    │ Free agent
                           │ Azure Files (tiering + cache)   │ + Azure Files cost

MIGRATION STRATEGIES (The 6 R's):
├── Rehost (lift-and-shift): VM → Azure VM (fastest, least change)
├── Replatform: VM → App Service or ACI (minor changes)
├── Refactor: Rewrite for cloud-native (most effort, most benefit)
├── Repurchase: Replace with SaaS (e.g., Exchange → M365)
├── Retain: Keep on-prem (not worth migrating yet)
└── Retire: Decommission (no longer needed)
```

---

## 22. Governance & Management Tools

```
SERVICE                   │ USE CASE                      │ ~COST
──────────────────────────┼───────────────────────────────┼──────────
Azure Policy              │ Enforce rules on resources    │ Free
                          │ Deny, Audit, Modify, DINE     │
──────────────────────────┼───────────────────────────────┼──────────
Azure Blueprints          │ Package: policies + RBAC +    │ Free
(being replaced by        │ ARM templates for landing zone│ (Use Deployment
 Deployment Stacks)       │ DON'T: new projects           │ Stacks instead)
──────────────────────────┼───────────────────────────────┼──────────
Management Groups         │ Hierarchy above subscriptions │ Free
──────────────────────────┼───────────────────────────────┼──────────
Cost Management           │ Budget tracking, forecasting, │ Free (for Azure)
                          │ anomaly detection, exports    │
──────────────────────────┼───────────────────────────────┼──────────
Azure Advisor             │ Right-sizing, cost, security, │ Free
                          │ performance recommendations   │
──────────────────────────┼───────────────────────────────┼──────────
Resource Graph             │ Query all resources at scale  │ Free
                          │ (KQL across subscriptions)    │
──────────────────────────┼───────────────────────────────┼──────────
Azure Arc                 │ Manage on-prem/multi-cloud    │
  Arc-enabled Servers     │ Azure policy on-prem VMs      │ Free (basic)
  Arc-enabled K8s         │ GitOps on non-AKS clusters    │ Free (basic)
  Arc-enabled SQL         │ Manage SQL anywhere           │ ESU licensing
  Arc-enabled Data Svcs   │ Azure SQL MI on-prem          │ Per vCore

DECISION:
├── Enforce standards across subscriptions → Azure Policy
├── Query all resources quickly (cross-sub) → Resource Graph
├── Track costs + get optimization tips → Cost Management + Advisor
├── Manage on-prem or multi-cloud resources → Azure Arc
└── Package repeatable environments → Deployment Stacks (replacing Blueprints)
```

---

## 23. Scenario-Based Decision Trees

### "Client wants to host a web application"

```
Is it a simple marketing/content site?
├── YES → Static Web App ($0-9/mo) or Storage static site + CDN
└── NO → Does it need a backend API?
    ├── YES → How many services?
    │   ├── 1-2 services → App Service Standard ($70)
    │   ├── 3+ microservices, team knows K8s → AKS Standard (~$500+)
    │   └── 3+ microservices, no K8s skills → Container Apps
    └── NO → App Service Basic ($13) or Functions
```

### "Client wants a database"

```
What kind of data?
├── Relational (SQL)
│   ├── Need SQL Server compatibility? → Azure SQL
│   ├── Want open-source, JSON support? → PostgreSQL Flexible
│   └── Simple MySQL app? → MySQL Flexible
├── Document/NoSQL
│   ├── Need global distribution? → Cosmos DB
│   ├── Simple document store? → Cosmos DB Serverless (dev), MongoDB Atlas
│   └── Key-value only? → Redis or Cosmos DB Table API
├── Time-series
│   └── Azure Data Explorer or InfluxDB on VM
└── Graph
    └── Cosmos DB Gremlin API or Neo4j on VM
```

### "Client needs to connect on-prem to Azure"

```
Budget and latency requirements?
├── Budget (<$200/mo), occasional use → VPN Gateway VpnGw1 ($138)
├── Production, <50ms latency → VPN Gateway VpnGw2 ($360)
├── Compliance (no internet), <10ms → ExpressRoute ($55+ port + ISP)
└── Both (ER primary, VPN failover) → ExpressRoute + VPN Gateway coexist
```

### "Client wants CI/CD"

```
What source control?
├── GitHub → GitHub Actions (free for public repos, 2000 min/mo private)
├── Azure Repos → Azure Pipelines (1 free parallel, $40/extra)
├── GitLab → GitLab CI (built-in)
└── Mixed → GitHub Actions or Azure Pipelines (both work with any SCM)

Where to build?
├── Microsoft-hosted agents → quick start, no maintenance, $40/parallel
├── Self-hosted agent on VM → more control, caching, $15/parallel license
└── Self-hosted on AKS → scalable, containerized builds
```

---

## 24. Common Architecture Patterns with Costs

### Pattern 1: Simple Web App (~$150/mo)

```
App Service (S1)        $70
Azure SQL (S0)          $15
Redis Cache (C0 Std)    $40
Key Vault (Standard)     $1
Storage (10GB Hot)       $2
App Insights (5GB)       $0 (free tier)
Azure DNS               $1
                    ─────────
Total:                ~$130/mo
```

### Pattern 2: Containerized Microservices (~$700/mo)

```
AKS Standard (3x D2s_v5)  $283 (nodes + control plane)
ACR Standard                $20
PostgreSQL Flexible (D2s)  $100
Redis Cache (C1 Std)        $68
Key Vault                    $1
App Gateway (Standard v2)  $145
Log Analytics (10GB)        $28
Storage (50GB Hot)           $1
Azure DNS                    $1
                         ─────────
Total:                    ~$650/mo
```

### Pattern 3: Enterprise Multi-Region (~$3,500/mo)

```
AKS Standard (3x D4s_v5 × 2 regions)  $966
ACR Premium (geo-replicated)             $50
PostgreSQL Flexible (D4s + read replica)$400
Redis Premium P1 (zone-redundant)       $225
Front Door Premium (WAF)                $330
Key Vault × 2 regions                     $2
Log Analytics (50GB)                    $138
Azure Firewall Standard                 $912
Bastion Basic                           $139
Storage (GRS, 100GB)                      $4
                                     ─────────
Total:                               ~$3,200/mo
```
