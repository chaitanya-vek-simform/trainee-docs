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

---

## 25. Workflow & Integration Services

```
SERVICE              │ USE CASE                         │ ~COST/mo
─────────────────────┼──────────────────────────────────┼──────────
Logic Apps           │                                  │
  Consumption        │ Pay-per-action, event-driven     │ $0.000025/action
                     │ workflows connecting 400+ apps   │ (~$5-50/mo typical)
  Standard           │ Longer runs, VNet integration,   │ $192/mo (WS1)
                     │ stateful workflows               │
─────────────────────┼──────────────────────────────────┼──────────
Azure Functions      │ Code-based event processing      │ See §1
  (Durable)          │ Orchestrations, fan-out/fan-in   │
─────────────────────┼──────────────────────────────────┼──────────
Azure Data Factory   │ ETL pipelines, data movement     │ See §14
─────────────────────┼──────────────────────────────────┼──────────
Azure Service Bus    │ Reliable enterprise messaging    │ See §8
─────────────────────┼──────────────────────────────────┼──────────
Azure API Management │ API gateway, developer portal    │ See §11

WHEN Logic Apps vs Functions vs Durable Functions:
├── Logic Apps Consumption: business logic, connect SaaS apps (Salesforce,
│   Slack, SharePoint), non-developer friendly, drag-and-drop, no code
├── Logic Apps Standard: same but need VNet, long-running, stateful
├── Functions: custom code needed, developer-driven, any language
├── Durable Functions: complex orchestration (fan-out, human approval,
│   retries, saga pattern), long-running with checkpointing
└── Choose Functions when you need full code control;
    Logic Apps when connecting services quickly without code
```

---

## 26. Developer Tools & Configuration

```
SERVICE                 │ USE CASE                      │ ~COST/mo
────────────────────────┼───────────────────────────────┼──────────
App Configuration       │ Centralized feature flags,    │
  Free                  │ 10MB config, 1k req/day       │ $0
  Standard              │ Soft-delete, RBAC, 200k req/day│$44
────────────────────────┼───────────────────────────────┼──────────
Azure Static Web Apps   │ Host React/Vue/Angular + API  │
  Free                  │ 100GB bandwidth, custom domain│ $0
  Standard              │ SLA, private endpoints, auth  │ $9
────────────────────────┼───────────────────────────────┼──────────
Azure Dev Center        │ Dev environment mgmt, DevBoxes│
  Dev Box (per user)    │ Cloud PC for developers       │ $2.90/hr (8vCPU 32GB)
────────────────────────┼───────────────────────────────┼──────────
Azure Spring Apps       │ Managed Spring Boot hosting   │
  Basic                 │ Dev/test                      │ $0.04/vCPU-hr
  Standard              │ Production, SLA, monitoring   │ $0.10/vCPU-hr
  Enterprise            │ Tanzu components, observability│ $0.20/vCPU-hr

WHEN App Configuration vs Key Vault vs environment variables:
├── Environment vars: simple, per-app, no central management
├── App Configuration: feature flags, per-environment values,
│   dynamic config refresh (change without redeploy)
├── Key Vault: secrets and certificates (rotate without redeploy)
└── Use App Configuration + Key Vault references together:
    App Config stores non-secret config → references KV for secrets

WHEN Static Web Apps vs App Service vs Storage static:
├── Storage static + CDN: cheapest ($1-5/mo), no API needed
├── Static Web Apps: SPA + serverless API (Functions), GitHub integration
├── App Service: need server-side rendering, custom server logic
└── Static Web Apps Standard ($9): private endpoints, enterprise auth
```

---

## 27. IoT Services

```
SERVICE                │ USE CASE                       │ ~COST/mo
───────────────────────┼────────────────────────────────┼──────────
IoT Hub                │ Device connectivity, D2C/C2D   │
  Free (F1)            │ 8000 msgs/day, 500 devices     │ $0
  Basic (B1)           │ D2C only, limited features     │ $10/unit
  Standard (S1)        │ Full features, 400k msg/day    │ $25/unit
  Standard (S3)        │ 300M messages/day              │ $2,500/unit
───────────────────────┼────────────────────────────────┼──────────
IoT Central            │ Fully managed IoT platform,    │
  Free                 │ 5 devices, 30 days             │ $0
  Standard S0          │ 2 messages/device/day          │ $0.60/device
  Standard S1          │ 30 messages/device/day         │ $2/device
───────────────────────┼────────────────────────────────┼──────────
Digital Twins          │ Model physical environments    │ $0.06/operation
                       │ smart buildings, factories     │ + $0.06/message
───────────────────────┼────────────────────────────────┼──────────
Time Series Insights   │ BEING RETIRED → use ADX        │
Azure Data Explorer    │ Time-series, IoT telemetry     │ $0.39/hr (2vCPU)

DECISION — IoT Hub vs IoT Central:
├── IoT Hub: developer-driven, full control, build your own solution,
│   connect to custom pipelines, large scale (millions of devices)
├── IoT Central: no-code platform, dashboard out of box, rules engine,
│   faster to get started, limited customization
└── IoT Hub wins when: custom processing, complex routing, full SDK control
    IoT Central wins when: quick PoC, no custom backend needed
```

---

## 28. Virtual Desktop & Remote Access

```
SERVICE                  │ USE CASE                      │ ~COST/mo
─────────────────────────┼───────────────────────────────┼──────────
Azure Virtual Desktop    │ Cloud-hosted Windows desktops │
  (AVD)                  │ and apps for remote workers   │
  Personal host pool     │ 1 user → 1 VM (dedicated)    │ VM cost only
  Pooled host pool       │ Multiple users share VMs      │ VM cost only
                         │ + Microsoft 365 license       │
  AVD per user pricing   │ Per user/month access         │ $10/user or
                         │ (instead of buying VMs)       │ included in M365
─────────────────────────┼───────────────────────────────┼──────────
Azure Dev Box            │ Cloud PC for developers       │ $130-330/mo/user
                         │ Pre-configured dev envs       │ (fixed config)
─────────────────────────┼───────────────────────────────┼──────────
Azure Bastion            │ Secure VM access              │ See §5

AVD sizing guide:
├── Light workload (Office, browser): D2s_v5 shared (2-4 users/VM)
├── Medium (design tools): D4s_v5 dedicated per user
├── Heavy (video editing, 3D): NV GPU series VM
└── Start pooled to optimize cost, switch to personal if users clash

WHEN AVD vs physical desktops vs Dev Box:
├── AVD: remote workers need Windows desktop/apps, BYO hardware, lower cost
├── Dev Box: developers need pre-configured dev environment, project isolation
└── Physical: specialized hardware needs, offline requirements, high GPU needs
```

---

## 29. Media & Content

```
SERVICE                  │ USE CASE                      │ ~COST/mo
─────────────────────────┼───────────────────────────────┼──────────
Azure Media Services     │ BEING RETIRED (June 2024)     │
                         │ → Use 3rd party or Blob+CDN   │
─────────────────────────┼───────────────────────────────┼──────────
Azure CDN (Front Door)   │ Video streaming/delivery      │ $0.081/GB
─────────────────────────┼───────────────────────────────┼──────────
Video Indexer            │ AI-based video analysis       │
                         │ face detection, transcription │ $0.04/min (basic)
─────────────────────────┼───────────────────────────────┼──────────
Blob Storage (Hot)       │ Store video files             │ $0.018/GB
+ Azure CDN              │ Stream to users               │ $0.081/GB delivered

VIDEO STREAMING PATTERN:
Upload → Blob Storage → Encode (Functions/VM) → Store → Front Door CDN → Users
No need for Azure Media Services anymore — Blob + CDN is sufficient.
```

---

## 30. High Performance Computing (HPC)

```
SERVICE              │ USE CASE                          │ ~COST/mo
─────────────────────┼───────────────────────────────────┼──────────
Azure Batch          │ Parallel batch jobs, rendering,   │ Free service
                     │ simulations, on-demand nodes      │ Pay for VMs only
─────────────────────┼───────────────────────────────────┼──────────
HPC VMs:             │                                   │
  HC (44-core)       │ MPI, tightly coupled simulation   │ $3,000+
  HB (120-core)      │ Memory bandwidth intensive HPC    │ $2,000+
  ND (GPU+InfiniBand)│ ML training at scale              │ $7,000+
─────────────────────┼───────────────────────────────────┼──────────
CycleCloud           │ HPC cluster management (SLURM/PBS)│ Free (pay VMs)
─────────────────────┼───────────────────────────────────┼──────────
Azure Spot VMs       │ HPC on cheap preemptible capacity │ 60-90% savings

WHEN Azure Batch vs Spot VMs vs AKS:
├── Azure Batch: designed for HPC, auto-provisions nodes, job queuing,
│   built-in MPI support, scale to thousands of nodes
├── Spot VMs manually: more control, need to handle eviction yourself
├── AKS + Spot nodes: containerized batch, K8s job scheduling
└── Azure Batch wins: large parallel workloads, rendering, simulations
```

---

## 31. Comparison: Azure vs AWS — Service Mapping

```
CATEGORY           │ AZURE                    │ AWS EQUIVALENT
───────────────────┼──────────────────────────┼─────────────────────────
Compute VM         │ Virtual Machines         │ EC2
Serverless compute │ Azure Functions          │ Lambda
Containers         │ AKS, Container Apps      │ EKS, ECS/Fargate
App Platform       │ App Service              │ Elastic Beanstalk
───────────────────┼──────────────────────────┼─────────────────────────
Object Storage     │ Blob Storage             │ S3
Block Storage      │ Managed Disks            │ EBS
File Storage       │ Azure Files              │ EFS / FSx
Archival           │ Blob Archive             │ S3 Glacier
───────────────────┼──────────────────────────┼─────────────────────────
SQL relational     │ Azure SQL / PostgreSQL   │ RDS (SQL Server / Aurora)
NoSQL              │ Cosmos DB                │ DynamoDB
In-memory cache    │ Azure Cache for Redis    │ ElastiCache
───────────────────┼──────────────────────────┼─────────────────────────
Message Queue      │ Service Bus              │ SQS
Pub/Sub            │ Service Bus Topics       │ SNS
Event Streaming    │ Event Hub                │ Kinesis
Event Routing      │ Event Grid               │ EventBridge
───────────────────┼──────────────────────────┼─────────────────────────
CDN                │ Front Door / Azure CDN   │ CloudFront
DNS                │ Azure DNS                │ Route 53
L4 Load Balancer   │ Load Balancer Standard   │ NLB
L7 Load Balancer   │ Application Gateway      │ ALB
Global L7 LB       │ Front Door               │ Global Accelerator + ALB
Firewall           │ Azure Firewall           │ AWS Network Firewall
VPN                │ VPN Gateway              │ Site-to-Site VPN (VGW)
Dedicated conn.    │ ExpressRoute             │ Direct Connect
───────────────────┼──────────────────────────┼─────────────────────────
Container Registry │ ACR                      │ ECR
CI/CD              │ Azure DevOps / GitHub    │ CodePipeline / CodeBuild
IaC                │ Bicep / Terraform        │ CloudFormation / Terraform
───────────────────┼──────────────────────────┼─────────────────────────
Secrets            │ Key Vault                │ Secrets Manager
Identity           │ Azure AD (Entra ID)      │ IAM + Cognito
Managed Identity   │ Managed Identity         │ IAM Roles for EC2/EKS
Policy/Governance  │ Azure Policy             │ AWS Organizations SCPs
───────────────────┼──────────────────────────┼─────────────────────────
Monitoring         │ Azure Monitor            │ CloudWatch
Log management     │ Log Analytics (KQL)      │ CloudWatch Logs Insights
APM                │ Application Insights     │ X-Ray + CloudWatch
Dashboards         │ Managed Grafana          │ CloudWatch Dashboards
Alerts             │ Azure Monitor Alerts     │ CloudWatch Alarms
───────────────────┼──────────────────────────┼─────────────────────────
AI/ML Platform     │ Azure Machine Learning   │ SageMaker
OpenAI             │ Azure OpenAI Service     │ Bedrock (Anthropic/etc)
Search             │ AI Search (Cognitive)    │ OpenSearch Service
───────────────────┼──────────────────────────┼─────────────────────────
Landing Zones      │ CAF + Management Groups  │ Control Tower + AWS Orgs
GitOps             │ Arc + Flux / ArgoCD      │ EKS Anywhere + Flux
Hybrid             │ Azure Arc                │ AWS Outposts / ECS Anywhere
DR orchestration   │ Site Recovery (ASR)      │ AWS Elastic DR
Backup             │ Recovery Services Vault  │ AWS Backup
```

---

## 32. Quotas & Limits — Things That Bite You in Production

```
┌──────────────────────────────────────────────────────────────┐
│  CRITICAL LIMITS TO KNOW (per subscription per region)       │
│                                                              │
│  COMPUTE:                                                    │
│  ├── vCPUs per region: DEFAULT 10-20 (request increase!)    │
│  ├── VM per resource group: no hard limit                   │
│  ├── Public IPs: 800 (standard SKU)                        │
│  └── NICs per subscription: 65,536                          │
│                                                              │
│  NETWORKING:                                                 │
│  ├── VNets per subscription: 1,000                          │
│  ├── Subnets per VNet: 3,000                                │
│  ├── VNet peerings per VNet: 500                            │
│  ├── NSG rules per NSG: 1,000                               │
│  ├── NSGs per NIC: 1 + 1 on subnet                         │
│  └── Private endpoints per VNet: 1,000                      │
│                                                              │
│  STORAGE:                                                    │
│  ├── Storage accounts per region: 250                       │
│  ├── Containers per account: 5 million                      │
│  └── Max blob size: 190.7 TB                                │
│                                                              │
│  AKS:                                                        │
│  ├── Nodes per cluster: 5,000                               │
│  ├── Pods per node: 250 (Azure CNI) or 110 (Kubenet)       │
│  ├── Node pools per cluster: 100                            │
│  └── Services per cluster: no hard limit (~65,000 ClusterIPs)│
│                                                              │
│  RESOURCE MANAGER:                                           │
│  ├── Resource groups per subscription: 980                  │
│  ├── Resources per resource group: 800 per type             │
│  └── Tags per resource: 50 (name+value each max 512 chars)  │
│                                                              │
│  HOW TO CHECK YOUR QUOTAS:                                   │
│  az vm list-usage --location eastus -o table                │
│  az network list-usages --location eastus -o table          │
│                                                              │
│  HOW TO REQUEST INCREASE:                                    │
│  Portal → Subscriptions → Usage + Quotas → Request increase │
│  Or: az support tickets create ... (takes 1-3 business days)│
└──────────────────────────────────────────────────────────────┘
```

---

## 33. Azure Naming Conventions Reference

```
RESOURCE TYPE         │ RECOMMENDED PREFIX  │ EXAMPLE
──────────────────────┼─────────────────────┼──────────────────────────
Resource Group        │ rg-                 │ rg-myapp-prod-eastus
Virtual Network       │ vnet-               │ vnet-myapp-prod
Subnet                │ snet-               │ snet-app-prod
NSG                   │ nsg-                │ nsg-app-prod
VM                    │ vm-                 │ vm-web01-prod
VM Scale Set          │ vmss-               │ vmss-web-prod
App Service Plan      │ asp-                │ asp-myapp-prod
App Service           │ app-                │ app-myapi-prod
Function App          │ func-               │ func-myprocessor-prod
AKS Cluster           │ aks-                │ aks-myapp-prod
Container Registry    │ cr or acr           │ crmyappprod (no dashes)
Azure SQL Server      │ sql-                │ sql-myapp-prod
Azure SQL Database    │ sqldb-              │ sqldb-myapp-prod
PostgreSQL            │ psql-               │ psql-myapp-prod
Cosmos DB             │ cosmos-             │ cosmos-myapp-prod
Redis Cache           │ redis-              │ redis-myapp-prod
Storage Account       │ st                  │ stmyappprod (no dashes)
Key Vault             │ kv-                 │ kv-myapp-prod
Service Bus Namespace │ sb-                 │ sb-myapp-prod
Event Hub Namespace   │ evhns-              │ evhns-myapp-prod
Log Analytics WS      │ log-                │ log-myapp-prod
Application Insights  │ appi-               │ appi-myapp-prod
Public IP             │ pip-                │ pip-myapp-prod
Load Balancer         │ lbi- / lbe-         │ lbe-myapp-prod (external)
Application Gateway   │ agw-                │ agw-myapp-prod
Azure Firewall        │ afw-                │ afw-hub-prod
Route Table           │ rt-                 │ rt-spoke-prod
Managed Identity      │ id-                 │ id-myapp-prod

FORMAT: {prefix}{workload}-{env}-{region(optional)}
RULES:
├── Lowercase only for most resources
├── Some (Storage Account, ACR) cannot have dashes or underscores
├── Max length varies: Storage=24, KV=24, RG=90, VM=15 (Windows)/64 (Linux)
└── Region suffix optional but helps in multi-region deployments
```

---

## 34. Azure Tags — Best Practices

```
MANDATORY TAGS (enforce via Azure Policy):
┌──────────────────┬─────────────────────────────────────────┐
│ Tag Name         │ Example Value                            │
├──────────────────┼─────────────────────────────────────────┤
│ environment      │ dev | staging | production              │
│ team             │ backend | frontend | data | devops      │
│ cost-center      │ CC-4521                                 │
│ project          │ ecommerce-platform                      │
│ managed-by       │ terraform | manual | bicep              │
│ created-date     │ 2025-07-16                              │
└──────────────────┴─────────────────────────────────────────┘

OPTIONAL BUT USEFUL:
│ owner            │ john.doe@company.com                    │
│ sla              │ 99.9% | 99.95% | 99.99%               │
│ data-class       │ public | internal | confidential        │
│ backup-policy    │ daily | weekly | none                   │
│ auto-shutdown    │ 7PM-UTC (dev VMs)                       │
└─────────────────────────────────────────────────────────────

Azure Policy to DENY resource creation without required tags:
{
  "if": {
    "anyOf": [
      {"field": "tags['environment']", "exists": "false"},
      {"field": "tags['team']",        "exists": "false"},
      {"field": "tags['cost-center']", "exists": "false"}
    ]
  },
  "then": {"effect": "deny"}
}

Query resources by tag (Resource Graph):
resources
| where tags['environment'] == 'production'
| where tags['team'] == 'backend'
| project name, type, resourceGroup, location
| order by type asc
```

---

## 35. Private Endpoint vs Service Endpoint — Deep Comparison

```
┌──────────────────────────────────────────────────────────────┐
│  SERVICE ENDPOINT                                            │
│  ├── Extends VNet identity to Azure PaaS service            │
│  ├── Traffic stays on Microsoft backbone (not internet)     │
│  ├── Source IP = VNet private IP (on the PaaS side)        │
│  ├── Service still has PUBLIC endpoint (just filtered)      │
│  ├── Free to use                                            │
│  └── Simple: enable per subnet per service type             │
│                                                              │
│  PRIVATE ENDPOINT                                            │
│  ├── Creates a NIC with private IP INSIDE your VNet         │
│  ├── Maps private IP to the specific PaaS resource          │
│  ├── Public endpoint CAN be fully disabled                  │
│  ├── Works across VNet peers, VPNs, ExpressRoute           │
│  ├── Requires Private DNS Zone + VNet link to resolve       │
│  └── Costs ~$7/mo per endpoint + data processing            │
│                                                              │
│  COMPARISON:                                                 │
│  Feature              │ Service Endpoint  │ Private Endpoint  │
│  ───────────────────  │ ────────────────  │ ───────────────  │
│  Cost                 │ Free              │ ~$7/mo           │
│  Private IP in VNet   │ No                │ Yes              │
│  Accessible from VPN  │ No (VNet only)    │ Yes (any network) │
│  Disable public access│ No (filter only)  │ Yes              │
│  DNS required         │ No                │ Yes              │
│  Cross-subscription   │ No                │ Yes              │
│  Works on-prem        │ No                │ Yes              │
│                                                              │
│  RULE:                                                       │
│  Dev/test, same VNet, budget matters → Service Endpoint     │
│  Production, compliance, hybrid, cross-sub → Private Endpoint│
└──────────────────────────────────────────────────────────────┘
```

---

## 36. Availability Zones vs Availability Sets vs Region Pairs

```
┌──────────────────────────────────────────────────────────────┐
│  AVAILABILITY SET                                            │
│  ├── Groups VMs across fault domains (separate racks)       │
│  ├── Groups VMs across update domains (separate patch waves)│
│  ├── Protects against: hardware failure, planned maintenance │
│  ├── Does NOT protect against datacenter-level failure      │
│  ├── SLA: 99.95% for 2+ VMs in Availability Set            │
│  └── Use when AZ not available in region or for old VMs     │
│                                                              │
│  AVAILABILITY ZONES                                          │
│  ├── Physically separate datacenters within a region        │
│  ├── Each zone has independent power, cooling, networking   │
│  ├── Distance: ~1-50km apart (low latency between zones)    │
│  ├── SLA: 99.99% for 2+ VMs across zones                   │
│  └── Supported in: East US, West EU, UK South, etc.        │
│                                                              │
│  REGION PAIRS                                                │
│  ├── Every Azure region is paired with another region       │
│  ├── East US ↔ West US 2                                    │
│  ├── UK South ↔ UK West                                     │
│  ├── Azure updates paired regions at different times        │
│  ├── GRS replicates to paired region                        │
│  └── Used for: DR, GRS storage replication, ASR failover    │
│                                                              │
│  WHEN TO USE WHAT:                                           │
│  Dev/test, no HA needed       → Single VM (no AS or AZ)    │
│  HA within datacenter         → Availability Set            │
│  HA within region (preferred) → Availability Zones          │
│  DR across regions            → Region Pairs + ASR/GRS      │
└──────────────────────────────────────────────────────────────┘

Cost impact:
├── Availability Set: free (just a grouping resource)
├── Availability Zone: may require Zone-redundant SKUs
│   ├── Zone-redundant Storage (ZRS): +25% over LRS
│   ├── Zone-redundant Redis Premium: ~10% premium
│   └── Zone-redundant App Service: requires Premium v2/v3
└── Cross-region (ASR): $25/VM/mo + storage + compute in DR region
```

---

## 37. Azure Networking Deep Dive — Route Tables & UDR

```
HOW ROUTING WORKS IN AZURE:
Azure automatically creates system routes for:
├── Traffic within a VNet (local routing)
├── Traffic to internet (0.0.0.0/0 → Internet)
├── Traffic to Azure services via service endpoints
└── Traffic between peered VNets

CUSTOM/USER-DEFINED ROUTES (UDR):
Used to OVERRIDE system routes. Most common use: force all traffic
through Azure Firewall (or NVA) in hub-spoke topology.

Example: spoke subnet must route ALL internet traffic through firewall
┌─────────────────────────────────────────────────────────────┐
│  Route Table on spoke app-subnet:                           │
│  ┌────────────────┬──────────────┬────────────────────┐    │
│  │ Address Prefix │ Next Hop Type│ Next Hop Address   │    │
│  ├────────────────┼──────────────┼────────────────────┤    │
│  │ 0.0.0.0/0      │ VirtualApp   │ 10.0.1.4 (AzFW IP)│    │
│  │ 10.0.0.0/8     │ VirtualApp   │ 10.0.1.4 (AzFW IP)│    │
│  └────────────────┴──────────────┴────────────────────┘    │
│  Effect: VM in spoke cannot reach internet directly.       │
│  ALL traffic → Hub Firewall → inspected → allowed/denied   │
└─────────────────────────────────────────────────────────────┘

```bash
# Create route table and add UDR
az network route-table create \
  --name rt-spoke-prod \
  --resource-group rg-networking \
  --location eastus

az network route-table route create \
  --route-table-name rt-spoke-prod \
  --resource-group rg-networking \
  --name force-to-firewall \
  --address-prefix 0.0.0.0/0 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address 10.0.1.4

# Associate with subnet
az network vnet subnet update \
  --name snet-app \
  --vnet-name vnet-spoke-prod \
  --resource-group rg-spoke \
  --route-table rt-spoke-prod
```

NSG vs Route Table — different layers:
├── NSG: stateful firewall at subnet/NIC level (allow/deny by IP+port)
├── UDR: routing decision — WHERE does traffic go next
└── Use both: UDR to send traffic to firewall, firewall inspects it
```

---

## 38. AKS Networking Options — CNI Deep Dive

```
CNI OPTION           │ USE CASE                       │ KEY DIFFERENCE
─────────────────────┼────────────────────────────────┼───────────────────
Kubenet              │ Small clusters, limited IPs    │ Pods get private IPs
(legacy)             │ DON'T: if pods need direct     │ NOT routable from VNet
                     │ VNet access                    │ NAT happens at node
─────────────────────┼────────────────────────────────┼───────────────────
Azure CNI            │ Production, pods need direct   │ Each pod gets a VNet IP
(standard)           │ routing, private endpoints     │ Routable from anywhere
                     │ to pods                        │ Needs larger subnet (/22)
─────────────────────┼────────────────────────────────┼───────────────────
Azure CNI Overlay    │ Large clusters (IP shortage)   │ Pods get overlay IPs
(recommended new)    │ Same as CNI features but       │ Only nodes consume VNet IPs
                     │ saves VNet IP space            │ (best of both worlds)
─────────────────────┼────────────────────────────────┼───────────────────
Azure CNI + Cilium   │ Advanced network policies,     │ eBPF-based, no kube-proxy
                     │ high performance, observability│ Hubble for networking viz

IP PLANNING FOR AKS (most common gotcha):
  Azure CNI standard: each pod gets a VNet IP
  3 nodes × 30 pods max = 90 pod IPs + 3 node IPs = 93 IPs minimum
  /22 subnet = 1,024 addresses → fits ~100 pods with room to scale
  /24 subnet = 256 addresses → too small for auto-scaling clusters!

  Recommendation:
  ├── System node pool: dedicated /24 subnet
  ├── User node pool: /22 subnet (allow scaling)
  └── If using Overlay: /24 is sufficient (pods don't consume VNet IPs)

AKS Private Cluster:
  API server (control plane) gets private IP only.
  kubectl from VNet only (or via VPN/bastion/private runner).
  Required for production security — public cluster exposes API to internet.

  az aks create \
    --name aks-prod \
    --resource-group rg-myapp \
    --enable-private-cluster \
    --private-dns-zone system
```

---

## 39. Azure Firewall Rules — Priority & Types

```
AZURE FIREWALL RULE TYPES (processed in this order):
1. DNAT Rules     → Translate inbound public IP:port to private IP:port
2. Network Rules  → Allow/deny by IP, port, protocol (L3/L4)
3. Application Rules → Allow/deny by FQDN, URL, web categories (L7)

PROCESSING ORDER:
├── Rules within same type: LOWEST priority number processed FIRST
├── If matched → action applied, processing stops
├── No match in any rule → DENY (implicit deny-all at bottom)

PRIORITY NUMBERING RECOMMENDATION:
├── 100-199: Core infrastructure (DNS, NTP, Windows Update)
├── 200-299: Platform team approved services
├── 300-399: Application-specific rules
├── 400-499: Temporary/emergency rules (document and remove)
└── 65000:   Implicit deny all (always present, not shown)

EXAMPLE RULE STRUCTURE:
Rule Collection Group: rcg-prod-workloads (priority 200)
├── Network Rule Collection: net-allow-dns (priority 100, Allow)
│   └── Rule: allow DNS → 168.63.129.16:53 UDP
├── Application Rule Collection: app-allow-updates (priority 200, Allow)
│   ├── Rule: windows-update → windowsupdate.com, update.microsoft.com
│   └── Rule: ubuntu-updates → *.ubuntu.com, *.canonical.com
└── Network Rule Collection: net-deny-all (priority 65000, Deny)
    └── Rule: block-all → *:* (catch-all)

GOTCHA: Azure Firewall processes application rules BEFORE network rules
for HTTP/HTTPS traffic. If you have a network rule allowing port 443
and an application rule denying the FQDN → the app rule WINS (deny).
```

---

## 40. Azure Monitor — KQL Quick Reference

```kql
-- Active VMs by region
resources
| where type == "microsoft.compute/virtualmachines"
| summarize count() by location
| order by count_ desc

-- All resources missing required tags
resources
| where isnull(tags.environment) or isnull(tags['cost-center'])
| project name, type, resourceGroup, tags

-- Find oversized VMs (low CPU utilization)
-- (Requires Azure Monitor metrics integration)
Perf
| where ObjectName == "Processor" and CounterName == "% Processor Time"
| summarize avg(CounterValue) by Computer, bin(TimeGenerated, 1h)
| where avg_CounterValue < 5   -- less than 5% avg CPU
| project Computer, avg_CPU=avg_CounterValue

-- Application Insights: top 10 slowest requests
requests
| summarize
    count = count(),
    avg_duration = avg(duration),
    p95_duration = percentile(duration, 95)
  by name, operation_Name
| order by p95_duration desc
| take 10

-- Application Insights: error rate over time
requests
| where timestamp > ago(24h)
| summarize
    total = count(),
    failed = countif(success == false)
  by bin(timestamp, 1h)
| extend error_rate = (failed * 100.0) / total
| order by timestamp asc

-- Log Analytics: find VMs that restarted
Event
| where EventLog == "System"
       and EventID == 6009   -- system startup event
| where TimeGenerated > ago(7d)
| project TimeGenerated, Computer, RenderedDescription
| order by TimeGenerated desc

-- Containers: OOMKilled events in AKS
KubePodInventory
| where ContainerLastStatus == "OOMKilled"
| where TimeGenerated > ago(24h)
| project TimeGenerated, Namespace, Name, ContainerLastStatus
| order by TimeGenerated desc

-- Service Bus: dead letter queue depth alert
AzureMetrics
| where ResourceProvider == "MICROSOFT.SERVICEBUS"
       and MetricName == "DeadletteredMessages"
| where Maximum > 100
| project TimeGenerated, ResourceId, Maximum
```

---

## 41. Azure Cost Management — Practical CLI Commands

```bash
# Current month spending by resource group
az consumption usage list \
  --start-date $(date -d "$(date +%Y-%m-01)" +%Y-%m-%d) \
  --end-date $(date +%Y-%m-%d) \
  --query "[].{RG:resourceGroup, Cost:pretaxCost, Currency:currency}" \
  -o table | sort -k3 -rn

# Set a monthly budget with email alert at 80% and 100%
az consumption budget create \
  --account-name "" \
  --budget-name "monthly-prod-budget" \
  --amount 5000 \
  --time-grain Monthly \
  --start-date "2025-07-01" \
  --end-date "2026-07-01" \
  --category Cost \
  --resource-group rg-myapp-prod

# Find all unattached (orphaned) resources costing money
# Unattached disks
az disk list \
  --query "[?diskState=='Unattached'].{Name:name,SizeGB:diskSizeGb,SKU:sku.name,RG:resourceGroup}" \
  -o table

# Unattached public IPs (still cost money even when not associated)
az network public-ip list \
  --query "[?ipConfiguration==null].{Name:name,SKU:sku.name,RG:resourceGroup}" \
  -o table

# Unused NICs
az network nic list \
  --query "[?virtualMachine==null].{Name:name,RG:resourceGroup}" \
  -o table

# Unused Application Gateways
az network application-gateway list \
  --query "[].{Name:name,State:operationalState,RG:resourceGroup}" \
  -o table

# Stopped (not deallocated) VMs — still charged for compute
az vm list -d \
  --query "[?powerState=='VM stopped'].{Name:name,State:powerState,RG:resourceGroup}" \
  -o table

# Get advisor cost recommendations
az advisor recommendation list \
  --category Cost \
  --query "[].{Title:shortDescription.solution,Impact:impact,Savings:extendedProperties.annualSavingsAmount}" \
  -o table | sort -k3 -rn
```

---

## 42. Azure Security — Key Concepts Summary

```
┌──────────────────────────────────────────────────────────────┐
│  DEFENCE IN DEPTH (Microsoft's layered security model)       │
│                                                              │
│         ┌─────────────────────────────┐                     │
│         │         DATA                │  ← Encrypt at rest  │
│         └──────────┬──────────────────┘    + in transit     │
│                    │                                        │
│         ┌──────────▼──────────────────┐                     │
│         │      APPLICATION            │  ← No secrets in   │
│         └──────────┬──────────────────┘    code, WAF        │
│                    │                                        │
│         ┌──────────▼──────────────────┐                     │
│         │        COMPUTE              │  ← Defender,        │
│         └──────────┬──────────────────┘    JIT VM access    │
│                    │                                        │
│         ┌──────────▼──────────────────┐                     │
│         │        NETWORK              │  ← NSG, Firewall,   │
│         └──────────┬──────────────────┘    Private Endpoint │
│                    │                                        │
│         ┌──────────▼──────────────────┐                     │
│         │       PERIMETER             │  ← DDoS, WAF        │
│         └──────────┬──────────────────┘                     │
│                    │                                        │
│         ┌──────────▼──────────────────┐                     │
│         │    IDENTITY & ACCESS        │  ← MFA, Cond.Access │
│         └──────────┬──────────────────┘    PIM, RBAC        │
│                    │                                        │
│         ┌──────────▼──────────────────┐                     │
│         │      PHYSICAL               │  ← Microsoft       │
│         └─────────────────────────────┘    manages this     │
│                                                              │
│  SHARED RESPONSIBILITY MODEL:                               │
│  IaaS (VMs):      You own OS, apps, data, network config   │
│  PaaS (App Svc):  You own apps and data only               │
│  SaaS (M365):     Microsoft owns almost everything          │
└──────────────────────────────────────────────────────────────┘

ZERO TRUST PRINCIPLES in Azure:
├── Verify explicitly: MFA + Conditional Access on all access
├── Least privilege: PIM, scope RBAC to minimum required
├── Assume breach: Defender for Cloud + network segmentation
│
KEY SECURITY SERVICES:
├── Microsoft Defender for Cloud: CSPM (posture) + CWPP (workload)
├── Microsoft Sentinel: SIEM + SOAR (detect + respond to threats)
├── Entra ID Protection: risky sign-in detection, risk-based CA
├── Azure DDoS: network layer protection (volumetric + protocol attacks)
└── Microsoft Purview: data governance, compliance, sensitivity labels
```

---

## 43. Service Reliability Patterns on Azure

```
PATTERN               │ HOW TO IMPLEMENT ON AZURE
──────────────────────┼────────────────────────────────────────────────
Circuit Breaker       │ Azure API Management policy (break-circuit)
                      │ Or app-level (Polly library for .NET)
──────────────────────┼────────────────────────────────────────────────
Retry with Backoff    │ Built into Azure SDK (DefaultAzureCredential)
                      │ Service Bus: built-in retry on receive
                      │ Functions: configurable retry policy
──────────────────────┼────────────────────────────────────────────────
Bulkhead              │ Separate App Service Plans per service
                      │ Separate AKS node pools per workload type
                      │ Separate Redis caches per app
──────────────────────┼────────────────────────────────────────────────
Health Endpoint       │ App Service health check path configuration
                      │ AKS liveness + readiness probes
                      │ Load Balancer health probes
──────────────────────┼────────────────────────────────────────────────
Graceful Degradation  │ Azure Cache for Redis as fallback for DB
                      │ Static content served from CDN if API down
                      │ Traffic Manager failover to static site
──────────────────────┼────────────────────────────────────────────────
Rate Limiting         │ APIM: rate-limit-by-key policy
                      │ Azure Functions: concurrency host settings
                      │ App Service: ARR affinity + custom code
──────────────────────┼────────────────────────────────────────────────
Saga Pattern          │ Azure Service Bus (choreography)
(distributed txn)     │ Azure Durable Functions (orchestration)
──────────────────────┼────────────────────────────────────────────────
CQRS                  │ Read replica (PostgreSQL/SQL read replica)
                      │ Separate read model in Cosmos DB
                      │ Event Hub → read-optimized store
──────────────────────┼────────────────────────────────────────────────
Event Sourcing        │ Event Hub (retain events 1-90 days)
                      │ + Azure Functions to project state
                      │ + Cosmos DB as read store
```

---

## 44. Quick Cheat Sheet — "Which Azure Service When"

```
SCENARIO                                          → SERVICE
──────────────────────────────────────────────────────────────
Host React SPA + backend Functions                → Static Web Apps
Run a Java Spring Boot app, don't want K8s       → Container Apps or App Service
Run multiple microservices, team knows K8s        → AKS Standard
Need GPU for ML training                          → NC/ND VM series or AML
Run a one-off script on a schedule               → Functions (timer trigger)
Store 10TB of files cheaply for 5 years          → Blob Archive
Stream 1M messages/sec from IoT devices          → Event Hub Standard (S3)
Queue messages between microservices              → Service Bus Standard
React to blob creation (process uploaded file)   → Event Grid + Functions
Broadcast real-time updates to thousands          → SignalR + Functions
Send OTP SMS to users                            → Communication Services SMS
Cache DB results for 10ms response times         → Redis Cache C1 Standard
Store secrets for an app running in AKS          → Key Vault (CSI driver)
Inspect outbound internet traffic from VMs       → Azure Firewall Standard
Connect 50 branch offices to Azure               → VPN Gateway VpnGw2 (mesh)
Migrate on-prem SQL Server to Azure              → SQL Managed Instance + DMS
Sync on-prem files to Azure (keep local cache)   → Azure File Sync
Protect web app from OWASP Top 10 attacks        → App Gateway WAF v2
Serve global users with <50ms response           → Front Door Standard + CDN
Archive logs for compliance (7yr retention)      → Storage Blob (Cool→Archive)
Get recommendations to save money on Azure        → Azure Advisor
Query all resources across subscriptions          → Azure Resource Graph
Deploy same policy to 100 subscriptions          → Management Groups + Policy
Manage on-prem K8s with Azure tools              → Azure Arc K8s
Connect on-prem database to Azure privately      → ExpressRoute + Private Link
Build data warehouse for BI reports              → Synapse Analytics
Run Apache Spark for ML feature engineering      → Databricks Premium
Detect anomalies in security logs                → Microsoft Sentinel
Enforce devs use pre-approved dev environments   → Azure Dev Center + Dev Box
Give dev team temporary access to prod           → Entra ID PIM (JIT access)
Enforce all VMs must have backup enabled         → Azure Policy (DINE effect)
Prevent teams from creating resources in EU      → Azure Policy (deny by location)
```

---

## 45. MySQL & MariaDB on Azure

```
SERVICE                  │ USE CASE                      │ ~COST/mo
─────────────────────────┼───────────────────────────────┼──────────
MySQL Flexible Server    │ WordPress, Laravel, Drupal,   │
  Burstable B1ms         │ dev/test, low traffic         │ $13
  Burstable B2s          │ small production              │ $26
  General Purpose D2s    │ Production web apps           │ $100
  General Purpose D4s    │ Medium production             │ $200
  Business Critical E4s  │ Heavy analytics, HA           │ $390
─────────────────────────┼───────────────────────────────┼──────────
MariaDB                  │ RETIRING Sept 2025            │
                         │ → Migrate to MySQL Flexible   │

WHEN MySQL vs PostgreSQL vs Azure SQL:
├── MySQL: PHP-based apps (WordPress, Laravel), open-source ecosystem,
│         JSON support, wide CMS compatibility
├── PostgreSQL: advanced SQL (window functions, CTEs), PostGIS, JSONB,
│             better for complex analytics, stronger ACID compliance
└── Azure SQL: .NET ecosystem, Azure AD auth built-in, full SQL Server
              compat, enterprise features (columnstore, in-memory OLTP)
```

---

## 46. Azure Container Apps — Deep Dive

```
┌──────────────────────────────────────────────────────────────┐
│  CONTAINER APPS KEY CONCEPTS                                 │
│                                                              │
│  Environment → shared boundary for apps (like a namespace)  │
│  App         → one microservice (can have multiple revisions)│
│  Revision    → immutable snapshot of app config + image     │
│  Replica     → running instance of a revision               │
│                                                              │
│  TRAFFIC SPLITTING (blue-green / canary):                   │
│  Revision A (v1.0) → 80% traffic                           │
│  Revision B (v1.1) → 20% traffic                           │
│  → gradually shift to 100% B when confident                │
│                                                              │
│  SCALING RULES (KEDA-based):                                │
│  ├── HTTP scaling: scale on concurrent requests             │
│  ├── CPU/Memory scaling: scale on resource usage            │
│  ├── Azure Queue scaling: scale on queue depth              │
│  ├── Service Bus scaling: scale on message count            │
│  ├── Event Hub scaling: scale on consumer lag               │
│  └── Custom KEDA scaler: any metric via ScaledObject       │
│                                                              │
│  DAPR INTEGRATION (optional sidecar):                       │
│  ├── Service invocation: call other apps by name (no DNS)  │
│  ├── State management: abstracted Redis/Cosmos state store  │
│  ├── Pub/Sub: abstracted Service Bus / Event Hub           │
│  ├── Bindings: trigger on external events (queue, blob)    │
│  └── Secrets: abstracted Key Vault access                  │
└──────────────────────────────────────────────────────────────┘

Container Apps Consumption pricing:
├── vCPU: $0.000024/vCPU-second
├── Memory: $0.000003/GB-second
├── Requests: $0.40/million requests
├── Scale to zero: no cost when idle (great for dev/test)
└── 180,000 vCPU-seconds FREE per month

Container Apps Dedicated pricing:
├── Pay for dedicated node (D4 workload profile ≈ $110/mo)
└── Apps run on your dedicated capacity (predictable, no cold start)

CLI: create and deploy a container app
az containerapp create \
  --name myapi \
  --resource-group rg-myapp \
  --environment my-env \
  --image ghcr.io/myorg/myapi:latest \
  --target-port 8000 \
  --ingress external \
  --min-replicas 0 \
  --max-replicas 10 \
  --scale-rule-name http-rule \
  --scale-rule-http-concurrency 50 \
  --env-vars "DB_URL=secretref:db-url" \
  --secrets "db-url=keyvaultref:https://kv-myapp.vault.azure.net/secrets/db-url"
```

---

## 47. Azure Lighthouse — Multi-Tenant Management

```
WHAT: Manage multiple customer Azure tenants from YOUR tenant.
      Service providers use this to manage client subscriptions
      without needing accounts in each client's tenant.

┌──────────────────────────────────────────────────────────────┐
│  WITHOUT LIGHTHOUSE:                                         │
│  Client A gives you a guest account in their tenant         │
│  Client B gives you a guest account in their tenant         │
│  Client C gives you a guest account in their tenant         │
│  → Separate logins, no unified view, hard to scale          │
│                                                              │
│  WITH LIGHTHOUSE:                                            │
│  Service Provider Tenant (YOUR org)                          │
│  └── Your engineers log in once to YOUR tenant              │
│      └── Can manage Client A, B, C subscriptions            │
│          from Azure portal/CLI without switching tenants     │
│                                                              │
│  CROSS-CUSTOMER VIEW:                                        │
│  ├── Azure Monitor: see metrics from all customers          │
│  ├── Azure Policy: apply policies across all customers      │
│  ├── Defender for Cloud: unified security posture           │
│  └── Azure Resource Graph: query all customer resources     │
└──────────────────────────────────────────────────────────────┘

COST: Azure Lighthouse itself is FREE.
      You pay for the resources in each customer subscription.

HOW TO ONBOARD A CUSTOMER:
1. Create an ARM template (offer definition) specifying:
   - Which principal (your engineers/groups)
   - What role (Contributor, Reader, etc.)
   - On which scope (subscription or resource group)
2. Customer deploys the template in THEIR subscription
3. Done — your tenant can now manage their resources

USE CASES IN A SERVICE COMPANY:
├── Centralized monitoring dashboard across all client environments
├── Apply security policies to all client subscriptions from one place
├── Run incident response across clients without login switching
└── Report cost and compliance across all clients
```

---

## 48. Service Principal vs Managed Identity — Full Comparison

```
┌──────────────────────────────────────────────────────────────┐
│  SERVICE PRINCIPAL                                           │
│  ├── An identity in Azure AD with client ID + secret/cert   │
│  ├── Works from ANYWHERE (local, external systems, on-prem) │
│  ├── YOU manage the secret (create, store, rotate)          │
│  ├── Secret can leak if stored insecurely                   │
│  ├── Secret expires → apps break if not rotated             │
│  └── Use when: external system, local dev, cross-tenant     │
│                                                              │
│  MANAGED IDENTITY                                            │
│  ├── Identity automatically managed by Azure                │
│  ├── No secret ever — Azure issues short-lived tokens       │
│  ├── Works ONLY when running INSIDE Azure                   │
│  ├── Rotation automatic — nothing to manage                 │
│  └── Use when: VM, AKS, App Service, Functions, ACI        │
│                                                              │
│  COMPARISON:                                                 │
│  Feature           │ Service Principal  │ Managed Identity  │
│  ─────────────────  │ ────────────────   │ ──────────────── │
│  Works from on-prem │ ✅ Yes             │ ❌ No            │
│  Works in Azure     │ ✅ Yes             │ ✅ Yes           │
│  Secret to manage   │ ✅ Yes (risk!)     │ ❌ No (great!)  │
│  Cross-tenant       │ ✅ Yes             │ ❌ No            │
│  Cost               │ Free              │ Free             │
│  Setup complexity   │ Medium            │ Low              │
│                                                              │
│  FEDERATION (Workload Identity Federation):                  │
│  Bridge between both — SP with no stored secret.            │
│  External system proves identity via OIDC token.            │
│  Azure AD validates token → issues access token.            │
│  ├── GitHub Actions → Azure (no stored secret in GH)       │
│  ├── Azure DevOps → Azure (no stored credentials)          │
│  └── Kubernetes (external) → Azure via OIDC               │
└──────────────────────────────────────────────────────────────┘

Workload Identity Federation setup (GitHub Actions example):
az ad app federated-credential create \
  --id <app-object-id> \
  --parameters '{
    "name": "github-actions-prod",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:myorg/myrepo:environment:production",
    "audiences": ["api://AzureADTokenExchange"]
  }'
# GitHub now authenticates to Azure WITHOUT storing any secret
```

---

## 49. Azure DNS Private Resolver

```
PROBLEM without DNS Private Resolver:
┌─────────────────────────────────────────────────────────────┐
│  On-Prem DNS Server                                         │
│  ├── Knows: server1.company.com → 192.168.1.10            │
│  ├── DOESN'T know: myapp.privatelink.postgres.database...  │
│  │   → On-prem users cannot resolve Azure private IPs      │
│  └── Azure VMs ALSO can't resolve on-prem DNS names       │
└─────────────────────────────────────────────────────────────┘

SOLUTION: Azure DNS Private Resolver
┌─────────────────────────────────────────────────────────────┐
│  Inbound Endpoint (10.0.5.4) ← On-prem forwards Azure DNS  │
│  On-prem DNS server:                                        │
│  ├── *.blob.core.windows.net → forward to 10.0.5.4        │
│  ├── *.database.windows.net → forward to 10.0.5.4         │
│  └── Azure Private Resolver resolves from Private DNS Zones│
│                                                              │
│  Outbound Endpoint (10.0.5.5) → Forwards to on-prem DNS   │
│  Azure VMs forwarding rules:                                │
│  ├── *.company.com → forward to 192.168.1.2 (on-prem DNS) │
│  └── Everything else → Azure public DNS                    │
└─────────────────────────────────────────────────────────────┘

COST: ~$175/mo per resolver (inbound + outbound endpoints)

WHEN DNS Private Resolver vs Custom DNS VM:
├── Custom DNS VM (old approach): needs high-availability setup,
│   OS patching, scalability management, your problem
├── DNS Private Resolver: fully managed, auto-scalable, no VMs to manage
└── Always prefer Private Resolver for new deployments
```

---

## 50. Azure Blob Storage — Advanced Features

```
FEATURE                  │ WHAT IT DOES                  │ COST IMPACT
─────────────────────────┼───────────────────────────────┼──────────
Versioning               │ Every overwrite creates new   │ Storage cost for
                         │ version (immutable old copies)│ ALL versions kept
                         │ Recover deleted/overwritten   │ Can add up fast!
─────────────────────────┼───────────────────────────────┼──────────
Soft Delete (blobs)      │ Deleted blobs kept X days     │ Storage cost during
                         │ before permanent removal      │ retention period
                         │ Default: 7 days retention     │
─────────────────────────┼───────────────────────────────┼──────────
Soft Delete (containers) │ Deleted containers recoverable│ Same as above
                         │ for retention period          │
─────────────────────────┼───────────────────────────────┼──────────
Immutability Policy      │ WORM: Write Once Read Many    │ No extra cost
  Time-based             │ Blobs cannot be deleted/mod   │
                         │ for locked retention period   │
  Legal Hold             │ Indefinite lock until removed │
                         │ (compliance, litigation)      │
─────────────────────────┼───────────────────────────────┼──────────
Lifecycle Mgmt Policy    │ Auto-move blobs between tiers │ Save on storage
                         │ based on age / last access    │ cost over time
─────────────────────────┼───────────────────────────────┼──────────
Object Replication       │ Async replicate blobs between │ Storage cost in
                         │ accounts (cross-region, cross │ destination account
                         │ subscription)                 │
─────────────────────────┼───────────────────────────────┼──────────
Point-in-Time Restore    │ Restore container to state    │ Requires versioning
                         │ from X days ago               │ + soft delete ON
─────────────────────────┼───────────────────────────────┼──────────
Change Feed              │ Ordered log of all blob       │ $0.004/10K transactions
                         │ creates/updates/deletes       │

RECOMMENDED SETTINGS FOR PRODUCTION:
├── Enable soft delete (blobs): 30-day retention
├── Enable soft delete (containers): 30-day retention
├── Enable versioning: only for critical buckets (cost!)
├── Enable immutability: for compliance/audit data
├── Enable lifecycle policy: auto-archive after 90 days
└── ENABLE SECURE TRANSFER (HTTPS only): always

COMMON GOTCHA:
Blob versioning + soft delete = versions of deleted blobs ALSO retained.
A table with 1M rows updated daily = 1M new versions daily = huge cost.
Use versioning selectively, not globally.
```

---

## 51. Azure Load Testing

```
WHAT: Run large-scale HTTP load tests using Apache JMeter scripts
      or URL-based quick tests. Identify performance bottlenecks
      before production.

WHEN TO USE:
├── Pre-launch: verify app handles expected peak load
├── After infrastructure changes: confirm performance maintained
├── Capacity planning: find saturation point (max users before failure)
└── Regression: compare performance before/after code changes

HOW IT WORKS:
Test Script (JMeter .jmx) → Azure Load Testing → Spins up engines
→ Generates HTTP load against target URL
→ Collects metrics (latency, error rate, throughput)
→ Integrates with App Insights for server-side metrics

COST: $10/virtual user hour
  100 virtual users for 1 hour = $1,000
  Use smaller tests ($10-100) to find bottlenecks first.

INTEGRATION WITH CI/CD (fail pipeline if latency too high):
- task: AzureLoadTest@1
  inputs:
    loadTestConfigFile: 'load-test.yaml'
    azureSubscription: 'my-service-connection'
    loadTestResource: 'myapp-load-test'
    failCriteria: |
      avg(response_time_ms) > 2000
      percentage(error) > 5
```

---

## 52. Azure Chaos Studio

```
WHAT: Intentionally inject failures to test resilience (chaos engineering).
      Verify your application actually handles failures as designed.

EXPERIMENTS (faults you can inject):
├── VM: CPU pressure, memory pressure, kill process, network disconnect
├── AKS: pod chaos, container kill, network partition, CPU stress on pods
├── Service Bus: latency, error injection
├── Cosmos DB: failover to secondary region
├── NSG: add deny rules (simulate network cut)
└── Azure Cache for Redis: connection disruption

COST: $0.10/hour per target (VM, AKS cluster, etc.)
  Run for 1 hour on 5 targets = $0.50 per experiment

WORKFLOW:
1. Define experiment: what fault, which targets, for how long
2. Run experiment during low-traffic period (not 3 PM Friday!)
3. Monitor app behavior during fault injection
4. Verify: did auto-healing work? Did alerts fire? Did failover happen?
5. Fix any gaps discovered
6. Re-run experiment to confirm fix works

EXAMPLE: Test AKS pod kill resilience
az rest --method PUT \
  --url "https://management.azure.com/.../experiments/kill-pods?api-version=2023-04-15-preview" \
  --body '{
    "properties": {
      "steps": [{
        "name": "Kill 30% of pods",
        "branches": [{
          "name": "Kill branch",
          "actions": [{
            "type": "continuous",
            "name": "urn:csci:microsoft:azureKubernetesServiceChaosMesh:podChaos/2.1",
            "duration": "PT10M",
            "parameters": [{"key": "jsonSpec", "value": "{mode:fixed-percent,value:30}"}]
          }]
        }]
      }]
    }
  }'
```

---

## 53. Azure Dedicated Hosts

```
WHAT: Physical server reserved exclusively for your VMs.
      No other customer's VMs share your hardware.

WHY:
├── Compliance: regulations requiring single-tenant hardware
├── Security: isolate sensitive workloads at hardware level
├── Licensing: use existing per-socket/per-core Windows/SQL licenses
└── Maintenance: control when maintenance events happen

COST: Per host (not per VM) — very expensive
  DSv3-Type1 (56 vCPUs): ~$3,500/mo
  You can pack multiple VMs onto one host up to its vCPU limit.

WHEN NOT to use Dedicated Hosts:
├── No compliance requirement for dedicated hardware → waste of money
├── Spot VMs or reserved instances are almost always cheaper
└── Most compliance frameworks (SOC2, ISO27001) don't require this

WHEN YES:
├── FedRAMP High, HIPAA with BAA requirements for dedicated hardware
├── Financial services with specific hardware isolation requirements
└── Defense/government contracts requiring sole-tenancy
```

---

## 54. Azure Network Watcher — Diagnostic Tools

```
TOOL                     │ WHAT IT DIAGNOSES                 │ COST
─────────────────────────┼───────────────────────────────────┼──────────
IP Flow Verify           │ "Can VM A reach 10.0.2.5:443?"   │ Free
                         │ Tests NSG rules (allowed/denied)  │
─────────────────────────┼───────────────────────────────────┼──────────
Next Hop                 │ "What route does traffic take     │ Free
                         │ from VM A to 8.8.8.8?"           │
─────────────────────────┼───────────────────────────────────┼──────────
Connection Troubleshoot  │ End-to-end connectivity test      │ $0.01/check
                         │ (latency, hop count, port status) │
─────────────────────────┼───────────────────────────────────┼──────────
NSG Flow Logs           │ Log all inbound/outbound traffic  │ Storage cost
                         │ through NSG (allowed + denied)    │ + Log Analytics
─────────────────────────┼───────────────────────────────────┼──────────
Traffic Analytics        │ Visualize flow logs (top talkers, │ $0.10/GB processed
                         │ malicious IPs, ports used)        │
─────────────────────────┼───────────────────────────────────┼──────────
Packet Capture           │ Capture VM network packets        │ Storage cost
                         │ (like Wireshark in the cloud)     │
─────────────────────────┼───────────────────────────────────┼──────────
Topology                 │ Visual map of VNet resources      │ Free
                         │ and their connections             │
─────────────────────────┼───────────────────────────────────┼──────────
Connection Monitor       │ Continuous monitoring of          │ $0.01/check
                         │ connectivity between endpoints    │

DEBUGGING WORKFLOW:
1. VM can't reach database → IP Flow Verify (is NSG blocking it?)
2. Traffic going to wrong place → Next Hop (is UDR misconfigured?)
3. Intermittent connectivity → Connection Monitor (track over time)
4. "Something" is talking to strange IPs → NSG Flow Logs + Traffic Analytics
5. Deep packet analysis → Packet Capture

CLI quick checks:
# Is NSG allowing VM → DB on port 5432?
az network watcher test-ip-flow \
  --vm vm-web01 --resource-group rg-prod \
  --direction Outbound \
  --local 10.1.1.4:12345 \
  --remote 10.1.2.8:5432 \
  --protocol TCP

# What's the next hop from VM to internet?
az network watcher show-next-hop \
  --vm vm-web01 --resource-group rg-prod \
  --source-ip 10.1.1.4 \
  --dest-ip 8.8.8.8
```

---

## 55. Subnet Delegation — When Subnets Are "Owned" by Services

```
WHAT: Some Azure services require a dedicated subnet that is
      "delegated" to them. No other resources can be in that subnet.

SERVICES REQUIRING DEDICATED/DELEGATED SUBNETS:
┌──────────────────────────────────────────────────────────────┐
│ Service                    │ Delegation Required   │ Min Size│
├──────────────────────────────────────────────────────────────┤
│ Application Gateway        │ No (dedicated but not │ /26     │
│                            │ delegated)             │         │
│ Azure Bastion              │ Named "AzureBastionSubnet"│/26   │
│ Azure Firewall             │ Named "AzureFirewallSubnet"│/26  │
│ API Management (internal)  │ Dedicated, /27 min    │ /27     │
│ App Service VNet Integrate │ Delegation: web farms  │ /28    │
│ Azure Container Apps       │ No delegation but need │ /23    │
│                            │ dedicated subnet       │         │
│ Azure Databricks           │ Two dedicated subnets  │ /26 ea │
│ Azure NetApp Files         │ Delegation: netapp     │ /28    │
│ Azure SQL Managed Instance │ Dedicated subnet       │ /27    │
│ Azure PostgreSQL Flexible  │ Delegation: DBforPostgre│/28   │
│ Azure MySQL Flexible       │ Delegation: DBforMySQL │ /28    │
└──────────────────────────────────────────────────────────────┘

PLANNING TIP: Reserve dedicated subnets in your CIDR plan from day one.
Adding a subnet delegation AFTER VMs are already in a subnet = recreate everything.
This is a common architecture mistake.

az network vnet subnet update \
  --name snet-postgresql \
  --resource-group rg-myapp \
  --vnet-name vnet-myapp-prod \
  --delegations Microsoft.DBforPostgreSQL/flexibleServers
```

---

## 56. Azure DevOps vs GitHub — Full Comparison

```
FEATURE                    │ AZURE DEVOPS              │ GITHUB
───────────────────────────┼───────────────────────────┼─────────────────────
Repos                      │ Azure Repos (Git/TFVC)    │ GitHub Repos (Git)
CI/CD                      │ Azure Pipelines (YAML/GUI)│ GitHub Actions (YAML)
Project mgmt               │ Azure Boards (Kanban,     │ GitHub Issues + Projects
                           │ Sprints, Epics)           │ (simpler)
Test management            │ Azure Test Plans          │ Limited (3rd party)
Artifact storage           │ Azure Artifacts           │ GitHub Packages
Container registry         │ ACR (separate)            │ GHCR (built-in)
Security scanning          │ Defender DevOps (add-on)  │ GHAS (GitHub Adv Security)
Private runners            │ Self-hosted agents        │ Self-hosted runners
Hosted runners             │ Microsoft-hosted (Linux,  │ GitHub-hosted (Linux,
                           │ Windows, macOS)           │ Windows, macOS, larger)
Pricing (CI/CD)            │ 1 free parallel job       │ 2000 min/mo (private repo)
                           │ $40/extra parallel        │ $8/user/mo (Teams)
Integration                │ Deep Azure integration    │ Deep GitHub ecosystem
Compliance features        │ Strong (SOC2, ISO, FedRAMP│ GHAS adds compliance
───────────────────────────┼───────────────────────────┼─────────────────────

WHEN Azure DevOps:
├── Enterprise .NET shop, Microsoft ecosystem
├── Need Azure Boards for project management (Jira-like)
├── Complex pipeline dependencies (stages, environments, approvals)
├── Already using Azure Repos
└── Compliance requirements that Azure DevOps audit trails satisfy

WHEN GitHub:
├── Open-source or community-facing projects
├── Team prefers GitHub UI (most developers know GitHub)
├── Want GitHub Actions marketplace (thousands of actions)
├── Need GitHub Copilot for developers
└── Code is already on GitHub

HYBRID (common in enterprises):
Azure Boards + GitHub Repos + GitHub Actions → best of both
GitHub has native Azure DevOps board integration.
```
