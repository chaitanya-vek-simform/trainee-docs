# 🚀 DevOps Practice Projects — Service-Based Company Focus

> Hands-on projects ordered by difficulty. Each project tells you WHAT to build,
> WHY it matters in a service company, and WHICH skills you'll prove.

---

## How to Use This

```
┌──────────────────────────────────────────────────────────────┐
│  FOR EACH PROJECT:                                           │
│                                                              │
│  1. Pick a project matching your current level              │
│  2. Set a deadline (1-2 weeks max per project)              │
│  3. Document everything in a README with screenshots        │
│  4. Push to YOUR GitHub — this IS your portfolio            │
│  5. Move to the next project                                │
│                                                              │
│  Service-company mindset:                                    │
│  Clients don't care about YOUR app — they hand you THEIRS.  │
│  You must be able to take ANY codebase and:                 │
│  ├── Containerize it                                        │
│  ├── Set up CI/CD for it                                    │
│  ├── Deploy it to cloud infrastructure                      │
│  ├── Monitor it                                             │
│  └── Hand it over with documentation                        │
│                                                              │
│  That's what these projects train you to do.                │
└──────────────────────────────────────────────────────────────┘
```

---

## 🟢 Beginner Projects (Week 1-3)

### Project 1: "Pick a Random GitHub Project and Host It"

**What:** Go to GitHub Explore, pick ANY open-source web app you've never seen before,
containerize it, and get it running on a VM.

**Why this matters:** In a service company, clients hand you codebases you've never seen.
You must figure out how to run them — fast.

**Steps:**
```
1. Pick a project (suggestions below)
2. Clone it, read the README, understand the stack
3. Write a Dockerfile from scratch (don't use theirs if they have one)
4. Write a docker-compose.yml with all dependencies (DB, cache, etc.)
5. Get it running locally with docker compose up
6. Create an Azure VM / AWS EC2 instance
7. Install Docker on the VM
8. Deploy the app on the VM
9. Configure Nginx as a reverse proxy
10. Access the app via the VM's public IP
```

**GitHub projects to try (pick one you haven't used before):**
```
Easy:
├── ghost/ghost         → Node.js blog platform + MySQL
├── hasura/graphql-engine → GraphQL API + PostgreSQL
├── strapi/strapi       → Node.js headless CMS + PostgreSQL
├── memos-hub/memos     → Go-based note app + SQLite

Medium:
├── calcom/cal.com      → Next.js scheduling app + PostgreSQL + Redis
├── nocodb/nocodb       → Airtable alternative + any DB
├── plausible/analytics → Elixir analytics + PostgreSQL + ClickHouse
├── outline/outline     → Wiki + PostgreSQL + Redis + S3

Hard:
├── sentry-org/sentry   → Python error tracking + PG + Redis + Kafka
├── posthog/posthog     → Python analytics + PG + Redis + ClickHouse + Kafka
└── mattermost          → Go chat platform + PostgreSQL
```

**Skills proved:** Docker, Docker Compose, Linux, Nginx, SSH, debugging unknown apps

---

### Project 2: "Personal CI/CD Pipeline from Scratch"

**What:** Take a simple app (your own or forked), set up a full CI/CD pipeline using
GitHub Actions that builds, tests, scans, and deploys automatically.

**Steps:**
```
1. Fork or create a simple web app (Flask/Express/Spring Boot)
2. Write unit tests (even 3-4 basic tests are enough)
3. Write a Dockerfile (multi-stage)
4. Create GitHub Actions workflow:
   ├── On PR: lint → test → security scan (Trivy)
   ├── On merge to main: build image → push to GHCR
   └── On tag (v*): deploy to VM via SSH
5. Set up branch protection on main
6. Configure GitHub Secrets for deployment
7. Make a PR, watch CI run, merge, watch CD deploy
```

```yaml
# Target pipeline structure:
# PR opened ─────► lint ──► test ──► trivy scan ──► ✅ can merge
# Merged to main ─► build ──► push to GHCR ──► deploy to staging
# Tag v*.*.* ──────► deploy to production (manual approval)
```

**Skills proved:** GitHub Actions, CI/CD, Docker, GHCR, automated testing

---

### Project 3: "Linux Server Hardening Playbook"

**What:** Spin up a fresh Ubuntu VM, manually harden it following a checklist,
then write a bash script that does it all automatically.

**Hardening checklist (automate ALL of these):**
```bash
# Your script should do all of this:
1. Create a non-root user with sudo access
2. Disable root SSH login
3. Configure SSH key-only authentication (disable passwords)
4. Change SSH port from 22 to a custom port
5. Install and configure UFW firewall (allow only SSH + HTTP + HTTPS)
6. Install and configure Fail2ban
7. Set up automatic security updates (unattended-upgrades)
8. Configure NTP time sync
9. Set up log rotation
10. Install and configure a monitoring agent
11. Disable unused services
12. Set up disk usage alerts (script that emails when >80%)
```

**Skills proved:** Linux, security, bash scripting, server administration

---

### Project 4: "Multi-Container App with Docker Compose"

**What:** Build a working local development environment for a full-stack app
using Docker Compose. No cloud — pure local containerization.

```
┌─────────────────────────────────────────────────────────┐
│  Architecture:                                           │
│                                                          │
│  Browser ──► Nginx (:80) ──► Frontend (:3000)           │
│                    │                                     │
│                    └──────► Backend API (:8000)          │
│                                    │                     │
│                              ┌─────┴─────┐              │
│                              ▼           ▼              │
│                         PostgreSQL    Redis              │
│                          (:5432)     (:6379)            │
│                              │                          │
│                              ▼                          │
│                          PgAdmin                        │
│                          (:5050)                        │
└─────────────────────────────────────────────────────────┘
```

**Requirements:**
```
1. Frontend: React/Vue/Angular (served by Nginx in production, dev server in dev)
2. Backend: Python/Node/Go API with DB access
3. Database: PostgreSQL with:
   ├── Named volume for data persistence
   ├── Healthcheck
   └── Init script that creates schema on first boot
4. Cache: Redis
5. Dev tools: PgAdmin (only in dev profile)
6. Two compose files:
   ├── docker-compose.yml (base)
   └── docker-compose.dev.yml (dev overrides: live reload, exposed ports)
7. Nginx config that proxies /api → backend, / → frontend
8. .env file for all configuration
```

**Skills proved:** Docker Compose, Nginx, multi-service architecture, dev/prod parity

---

## 🟡 Intermediate Projects (Week 4-8)

### Project 5: "Terraform Infrastructure as Code"

**What:** Provision a complete environment on Azure (or AWS) using only Terraform.
No portal clicking allowed.

**Infrastructure to build:**
```
┌─────────────────────────────────────────────────────────┐
│  Azure Resources to Create via Terraform:                │
│                                                          │
│  Resource Group                                          │
│  ├── VNet (10.0.0.0/16)                                │
│  │   ├── Subnet: public (10.0.1.0/24)                  │
│  │   ├── Subnet: private (10.0.2.0/24)                 │
│  │   └── Subnet: database (10.0.3.0/24)                │
│  ├── NSG (attached to subnets with proper rules)        │
│  ├── VM (public subnet, Nginx reverse proxy)            │
│  ├── Azure Container Instance or App Service            │
│  ├── Azure Database for PostgreSQL (private subnet)     │
│  ├── Key Vault (secrets for DB password, API keys)      │
│  ├── Storage Account (for backups)                      │
│  └── Log Analytics Workspace                            │
│                                                          │
│  Structure:                                              │
│  terraform/                                              │
│  ├── main.tf                                            │
│  ├── variables.tf                                       │
│  ├── outputs.tf                                         │
│  ├── terraform.tfvars (gitignored)                      │
│  ├── modules/                                           │
│  │   ├── networking/                                    │
│  │   ├── compute/                                       │
│  │   └── database/                                      │
│  └── environments/                                      │
│      ├── dev.tfvars                                     │
│      └── prod.tfvars                                    │
└─────────────────────────────────────────────────────────┘
```

**Rules:**
```
- Use modules (not one giant main.tf)
- Use remote backend (Azure Blob) for state
- Tag every resource with: environment, team, managed-by
- Use data sources to reference existing resources
- Output all important values (IPs, URLs, connection strings)
- README with architecture diagram and setup instructions
```

**Skills proved:** Terraform, IaC, cloud networking, modules, state management

---

### Project 6: "Blue-Green Deployment on VMs"

**What:** Set up two identical environments (blue/green) on VMs, with a load balancer
that switches traffic between them for zero-downtime deployments.

```
┌─────────────────────────────────────────────────────────┐
│  Blue-Green Setup:                                       │
│                                                          │
│  Client ──► Load Balancer / Nginx                       │
│                   │                                      │
│          ┌────────┴────────┐                             │
│          ▼                 ▼                              │
│    ┌──────────┐      ┌──────────┐                       │
│    │ BLUE VM  │      │ GREEN VM │                        │
│    │ (v1.0)   │      │ (v1.1)   │                       │
│    │ ACTIVE ✅│      │ STANDBY  │                        │
│    └──────────┘      └──────────┘                       │
│                                                          │
│  Deploy flow:                                            │
│  1. Deploy v1.1 to GREEN (inactive)                     │
│  2. Run smoke tests against GREEN                       │
│  3. Switch Nginx upstream to GREEN                      │
│  4. GREEN is now ACTIVE, BLUE becomes standby           │
│  5. If issues → switch back to BLUE instantly           │
└─────────────────────────────────────────────────────────┘
```

**Build:**
```
1. Two VMs (blue + green) running Docker
2. One VM running Nginx as load balancer
3. Bash script: deploy.sh <version> <blue|green>
   ├── Pulls new image on target VM
   ├── Runs smoke tests
   ├── Updates Nginx config to point to target VM
   ├── Reloads Nginx
   └── Verifies health of new active environment
4. Bash script: rollback.sh
   ├── Switches Nginx back to previous VM
   └── Verifies health
5. CI/CD pipeline triggers deploy.sh on merge to main
```

**Skills proved:** Deployment strategies, Nginx, load balancing, zero-downtime deployments

---

### Project 7: "Monitoring Stack from Scratch"

**What:** Deploy a complete monitoring stack and instrument an application.

```
┌─────────────────────────────────────────────────────────┐
│  Monitoring Stack:                                       │
│                                                          │
│  Your App ──► Prometheus (scrapes metrics)              │
│       │              │                                   │
│       │              ▼                                   │
│       │       Grafana (dashboards)                      │
│       │                                                  │
│       └──► Loki (log aggregation)                       │
│              │                                           │
│              ▼                                           │
│       Grafana (log viewer)                              │
│                                                          │
│  Alertmanager ──► Slack / Email                         │
└─────────────────────────────────────────────────────────┘
```

**Requirements:**
```
1. Docker Compose with:
   ├── Prometheus (metrics collection)
   ├── Grafana (visualization)
   ├── Loki (log aggregation)
   ├── Promtail (log shipping agent)
   ├── Alertmanager (alerting)
   ├── Node Exporter (host metrics)
   └── Your app (instrumented with metrics)

2. Dashboards in Grafana:
   ├── System: CPU, memory, disk, network
   ├── Docker: container stats
   ├── App: request rate, error rate, latency (RED metrics)
   └── Logs: error count over time

3. Alert rules:
   ├── CPU > 80% for 5 minutes
   ├── Disk > 85%
   ├── App error rate > 5%
   ├── Container restart detected
   └── Endpoint down for 2 minutes

4. Notifications to Slack channel
```

**Skills proved:** Prometheus, Grafana, Loki, observability, alerting

---

### Project 8: "Multi-Client Infrastructure (Service Company Simulation)"

**What:** Simulate managing infrastructure for 3 different "clients" using
separate Terraform workspaces, separate pipelines, and proper isolation.

```
┌─────────────────────────────────────────────────────────┐
│  Service Company Simulation:                             │
│                                                          │
│  Client A (E-commerce)         → Node.js + MongoDB      │
│  Client B (Blog Platform)      → WordPress + MySQL      │
│  Client C (REST API)           → Python + PostgreSQL    │
│                                                          │
│  YOUR repo structure:                                    │
│  infra/                                                  │
│  ├── modules/        (shared Terraform modules)         │
│  │   ├── vm/                                            │
│  │   ├── database/                                      │
│  │   └── networking/                                    │
│  ├── clients/                                           │
│  │   ├── client-a/                                      │
│  │   │   ├── main.tf  (uses shared modules)             │
│  │   │   ├── variables.tf                               │
│  │   │   └── terraform.tfvars                           │
│  │   ├── client-b/                                      │
│  │   └── client-c/                                      │
│  └── pipelines/                                         │
│      ├── client-a.yml                                   │
│      ├── client-b.yml                                   │
│      └── client-c.yml                                   │
│                                                          │
│  Each client:                                            │
│  ├── Separate Azure resource group                      │
│  ├── Separate state file                                │
│  ├── Separate CI/CD pipeline                            │
│  ├── Separate monitoring dashboard                      │
│  └── Documented runbook                                 │
└─────────────────────────────────────────────────────────┘
```

**Skills proved:** Multi-tenancy, Terraform modules, pipeline management, documentation

---

## 🔴 Advanced Projects (Week 9-16)

### Project 9: "Kubernetes from Scratch to Production"

**What:** Deploy a multi-service application to Kubernetes (AKS/EKS or Minikube)
with proper production patterns.

```
┌─────────────────────────────────────────────────────────┐
│  K8s Architecture:                                       │
│                                                          │
│  Ingress (Nginx Ingress Controller)                     │
│  ├── app.example.com ──► frontend (Deployment, 2 pods) │
│  ├── api.example.com ──► backend  (Deployment, 3 pods) │
│  └── admin.example.com ──► admin  (Deployment, 1 pod)  │
│                                                          │
│  Backend ──► PostgreSQL (StatefulSet or Azure DB)       │
│         ──► Redis (StatefulSet)                         │
│                                                          │
│  Supporting:                                             │
│  ├── cert-manager (auto TLS certificates)               │
│  ├── External Secrets Operator (sync from Key Vault)    │
│  ├── Horizontal Pod Autoscaler (CPU-based scaling)      │
│  ├── Pod Disruption Budget (availability guarantee)     │
│  ├── Network Policies (restrict pod-to-pod traffic)     │
│  └── RBAC (developer vs admin access)                   │
└─────────────────────────────────────────────────────────┘
```

**Deliverables:**
```
1. Helm chart for your application (not raw YAML)
2. Values files per environment (dev.yaml, prod.yaml)
3. CI/CD: build → push → helm upgrade (GitHub Actions)
4. Monitoring: Prometheus + Grafana (helm charts)
5. Logging: Loki + Promtail or EFK stack
6. README with:
   ├── Architecture diagram
   ├── Setup instructions
   ├── Runbook for common operations
   └── Disaster recovery procedure
```

**Skills proved:** Kubernetes, Helm, Ingress, TLS, autoscaling, RBAC, monitoring

---

### Project 10: "GitOps with ArgoCD"

**What:** Set up a GitOps workflow where Git is the single source of truth
for your Kubernetes cluster state.

```
┌─────────────────────────────────────────────────────────┐
│  GitOps Flow:                                            │
│                                                          │
│  Developer pushes code                                   │
│       │                                                  │
│       ▼                                                  │
│  CI Pipeline (GitHub Actions):                          │
│  ├── Build + test                                       │
│  ├── Build Docker image                                 │
│  ├── Push to GHCR/ACR                                   │
│  └── Update image tag in GitOps repo (kustomize/helm)   │
│       │                                                  │
│       ▼                                                  │
│  GitOps Repo (k8s manifests):                           │
│  └── commit: "chore: update myapp to sha-abc123"        │
│       │                                                  │
│       ▼                                                  │
│  ArgoCD detects change in GitOps repo                   │
│  └── Syncs cluster state to match Git                   │
│       │                                                  │
│       ▼                                                  │
│  Kubernetes cluster is updated automatically            │
│                                                          │
│  To rollback: git revert the commit → ArgoCD auto-syncs│
└─────────────────────────────────────────────────────────┘
```

**Repos structure:**
```
Repo 1: application code + CI pipeline (builds + pushes image)
Repo 2: GitOps manifests (Helm values / Kustomize overlays)
ArgoCD watches Repo 2 and syncs to cluster.
```

**Skills proved:** GitOps, ArgoCD, Kustomize, Helm, deployment automation

---

### Project 11: "Complete DevSecOps Pipeline"

**What:** Build a CI/CD pipeline that includes every security gate a service
company client would expect.

```
┌─────────────────────────────────────────────────────────┐
│  DevSecOps Pipeline Stages:                              │
│                                                          │
│  PR Created:                                             │
│  ├── Lint (eslint / flake8 / golint)                    │
│  ├── Unit Tests                                         │
│  ├── SAST: Semgrep (static code analysis)               │
│  ├── SCA: Trivy / Snyk (dependency vulnerabilities)     │
│  ├── Secrets scan: gitleaks (no secrets in code)        │
│  ├── IaC scan: tfsec / checkov (Terraform misconfigs)   │
│  └── License compliance check                           │
│                                                          │
│  Merge to main:                                         │
│  ├── Build Docker image                                 │
│  ├── Container scan: Trivy image (CVE check)            │
│  ├── Sign image (cosign)                                │
│  ├── Push to registry                                   │
│  ├── Deploy to staging                                  │
│  ├── DAST: OWASP ZAP (dynamic scanning against staging)│
│  └── Smoke tests                                        │
│                                                          │
│  Production deploy (manual approval):                   │
│  ├── Verify image signature                             │
│  ├── Deploy                                             │
│  └── Post-deploy security scan                          │
└─────────────────────────────────────────────────────────┘
```

**Skills proved:** SAST, DAST, SCA, container scanning, image signing, compliance

---

### Project 12: "Disaster Recovery Drill"

**What:** Build infrastructure, deploy an app, simulate failures, and recover.
Document the entire process as a runbook.

**Scenarios to simulate and recover from:**
```
1. Database corruption
   ├── Backup exists → restore from backup
   ├── Verify data integrity
   └── Document RTO (recovery time objective)

2. VM failure
   ├── Auto-scaling replaces the VM
   ├── OR manual: provision new VM from Terraform
   └── Deploy app, verify service restored

3. Kubernetes node failure
   ├── Pods automatically rescheduled to other nodes
   ├── Verify all pods healthy after rescheduling
   └── Add replacement node

4. Entire region outage (advanced)
   ├── DNS failover to secondary region
   ├── Promote DB replica in secondary region
   └── Verify application in secondary region

5. Secret/credential rotation
   ├── Rotate database password
   ├── Update Key Vault
   ├── Rolling restart of application pods
   └── Verify zero downtime during rotation

6. Accidental deletion
   ├── Delete a Kubernetes deployment "accidentally"
   ├── Recover using GitOps (ArgoCD auto-heals)
   ├── OR recover from Helm rollback
   └── Document prevention measures
```

**Deliverable: Runbook document with:**
```
For each scenario:
├── Detection: How do you know it happened? (alerts, metrics)
├── Impact: What is affected? (users, data, SLA)
├── Recovery steps: Exact commands to run
├── Verification: How do you confirm it's fixed?
├── Prevention: How to prevent it next time
└── RTO/RPO: How long did recovery take?
```

**Skills proved:** Disaster recovery, incident response, documentation, resilience

---

## 🏆 Capstone Project: "Full Platform for a Fictional Client"

### The Scenario

A fictional client "ShopEasy" is a small e-commerce startup. They have:
- A React frontend
- A Node.js backend API
- A PostgreSQL database
- A Redis cache for sessions

They need you to set up their ENTIRE infrastructure and CI/CD.

### What You Deliver

```
┌─────────────────────────────────────────────────────────┐
│  DELIVERABLES CHECKLIST:                                 │
│                                                          │
│  Infrastructure (Terraform):                             │
│  [ ] Azure resource group with tags                     │
│  [ ] VNet with public + private subnets                 │
│  [ ] AKS cluster (or VMs with Docker)                   │
│  [ ] Azure DB for PostgreSQL (private endpoint)         │
│  [ ] Azure Cache for Redis (private endpoint)           │
│  [ ] Key Vault for secrets                              │
│  [ ] Container Registry (ACR)                           │
│  [ ] Storage Account for backups                        │
│  [ ] Log Analytics Workspace                            │
│                                                          │
│  Containerization:                                       │
│  [ ] Dockerfile for frontend (multi-stage, Nginx)       │
│  [ ] Dockerfile for backend (multi-stage, non-root)     │
│  [ ] docker-compose.yml for local dev                   │
│  [ ] .dockerignore for both                             │
│                                                          │
│  CI/CD (GitHub Actions):                                │
│  [ ] PR pipeline: lint + test + scan                    │
│  [ ] Build pipeline: Docker build + push to ACR         │
│  [ ] Deploy pipeline: Helm upgrade to AKS              │
│  [ ] Staging auto-deploy on merge to main              │
│  [ ] Production deploy on tag with approval             │
│                                                          │
│  Kubernetes:                                             │
│  [ ] Helm chart for the application                     │
│  [ ] Ingress with TLS (cert-manager)                    │
│  [ ] HPA for backend (scale on CPU)                     │
│  [ ] Resource limits on all pods                        │
│  [ ] Health checks (readiness + liveness)               │
│  [ ] Network policies                                   │
│                                                          │
│  Monitoring:                                             │
│  [ ] Prometheus + Grafana (or Azure Monitor)            │
│  [ ] Dashboard: request rate, error rate, latency       │
│  [ ] Alerts: high error rate, pod restarts, disk full   │
│  [ ] Log aggregation (Loki or Azure Log Analytics)      │
│                                                          │
│  Security:                                               │
│  [ ] OIDC auth for CI/CD → Azure (no stored secrets)    │
│  [ ] Secrets in Key Vault (not in code or env vars)     │
│  [ ] Container scanning in pipeline                     │
│  [ ] Non-root containers                                │
│  [ ] Branch protection on main                          │
│                                                          │
│  Documentation:                                          │
│  [ ] Architecture diagram (ASCII or draw.io)            │
│  [ ] README with setup instructions                     │
│  [ ] Runbook for common operations                      │
│  [ ] Cost estimate for the infrastructure               │
│  [ ] Handover document for the "client"                 │
└─────────────────────────────────────────────────────────┘
```

**This capstone project alone, fully completed, is worth more on your resume
than any certification.**

---

## Quick Reference: What Each Project Teaches

| # | Project | Key Skills |
|---|---------|-----------|
| 1 | Host random GitHub project | Docker, Docker Compose, Nginx, Linux, debugging |
| 2 | CI/CD pipeline from scratch | GitHub Actions, GHCR, automated testing |
| 3 | Server hardening script | Linux security, bash scripting, automation |
| 4 | Multi-container local env | Docker Compose, dev/prod parity, Nginx |
| 5 | Terraform IaC | Terraform, modules, remote state, cloud networking |
| 6 | Blue-Green deployment | Deployment strategies, zero-downtime, Nginx |
| 7 | Monitoring stack | Prometheus, Grafana, Loki, alerting |
| 8 | Multi-client infra | Multi-tenancy, Terraform modules, isolation |
| 9 | K8s production setup | Kubernetes, Helm, Ingress, TLS, HPA, RBAC |
| 10 | GitOps with ArgoCD | GitOps, ArgoCD, Kustomize, declarative ops |
| 11 | DevSecOps pipeline | SAST, DAST, SCA, container scanning, compliance |
| 12 | Disaster recovery drill | Incident response, backups, runbooks, RTO/RPO |
| 🏆 | Capstone: full platform | Everything above combined into one project |

---

## Tips for Service-Company DevOps

```
┌──────────────────────────────────────────────────────────────┐
│  WHAT SERVICE COMPANIES VALUE:                               │
│                                                              │
│  1. Speed of onboarding to new projects                     │
│     → Practice Project 1 repeatedly with different apps     │
│                                                              │
│  2. Documentation quality                                    │
│     → Every project must have a README someone else can     │
│       follow without asking you questions                   │
│                                                              │
│  3. Reusable modules and templates                          │
│     → Build Terraform modules and Helm charts you can       │
│       copy across client projects                           │
│                                                              │
│  4. Security awareness                                       │
│     → Clients audit your infrastructure — show you know     │
│       secrets management, least privilege, scanning         │
│                                                              │
│  5. Cost consciousness                                       │
│     → Always include cost estimates in your proposals       │
│     → Know how to right-size VMs and auto-deallocate dev    │
│                                                              │
│  6. Multi-stack flexibility                                  │
│     → Don't just know Node.js. Practice with Python, Go,   │
│       Java, PHP — clients use everything                    │
│                                                              │
│  7. Handover-ready documentation                            │
│     → Projects end. Another engineer takes over.            │
│       Your docs should make that seamless.                  │
└──────────────────────────────────────────────────────────────┘
```
