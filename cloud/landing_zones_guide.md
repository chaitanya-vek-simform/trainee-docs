# 🏗️ Cloud Landing Zones — DevOps Engineer Walkthrough

> A landing zone is a pre-configured, governed cloud environment
> where workloads can be safely deployed from day one.

---

## 1. What Is a Landing Zone?

```
┌──────────────────────────────────────────────────────────────┐
│  LANDING ZONE = Foundation Before Any App Is Deployed        │
│                                                              │
│  Think of it like a construction site:                      │
│  Before you build a house, you need:                        │
│  ├── Land survey (identity & access)                        │
│  ├── Foundation & utilities (networking)                    │
│  ├── Building codes (policies & governance)                 │
│  ├── Security fence (firewalls & perimeters)                │
│  └── Address & mailbox (DNS, connectivity)                  │
│                                                              │
│  Without a landing zone, every team builds ad-hoc:          │
│  ├── VNets that can't peer (overlapping CIDRs)             │
│  ├── No central logging (can't investigate incidents)       │
│  ├── No policies (devs create public databases)             │
│  ├── No budget controls (surprise $50K bill)                │
│  └── No consistent naming (chaos)                           │
│                                                              │
│  Landing zone prevents ALL of the above.                    │
└──────────────────────────────────────────────────────────────┘
```

---

## 2. Azure Landing Zone Architecture (CAF)

```
┌──────────────────────────────────────────────────────────────┐
│  MANAGEMENT GROUP HIERARCHY                                  │
│                                                              │
│  Root (Tenant)                                               │
│  └── Company Root MG                                        │
│      ├── Platform MG                                        │
│      │   ├── Management Sub    → Log Analytics, Defender    │
│      │   ├── Identity Sub      → Azure AD, Domain Services  │
│      │   └── Connectivity Sub  → Hub VNet, Firewall, VPN    │
│      │                                                       │
│      ├── Landing Zones MG                                   │
│      │   ├── Production MG                                  │
│      │   │   ├── Client-A-Prod Sub                         │
│      │   │   └── Client-B-Prod Sub                         │
│      │   └── Non-Production MG                              │
│      │       ├── Client-A-Dev Sub                          │
│      │       └── Client-A-Staging Sub                      │
│      │                                                       │
│      ├── Sandbox MG                                         │
│      │   └── Experimentation Sub  (no policies, isolated)   │
│      │                                                       │
│      └── Decommissioned MG                                  │
│          └── Old-Project Sub  (locked, read-only)           │
│                                                              │
│  POLICIES are applied at MG level and INHERIT downward.     │
│  Apply strict policies at "Production MG" → every prod      │
│  subscription automatically gets them.                      │
└──────────────────────────────────────────────────────────────┘
```

---

## 3. Landing Zone Components — What You Build

### A. Identity & Access (RBAC)

```
WHO can do WHAT on WHICH resources?

Roles to assign at each level:
┌─────────────────────┬─────────────────────────────────────┐
│ Level               │ Role Assignment                     │
├─────────────────────┼─────────────────────────────────────┤
│ Company Root MG     │ Global Admin (break-glass only)     │
│ Platform MG         │ Platform Team → Contributor         │
│ Connectivity Sub    │ Network Team → Network Contributor  │
│ Landing Zone Sub    │ App Team → Contributor              │
│ Production MG       │ DevOps → Reader (deploy via CI/CD)  │
│ Resource Groups     │ Service Principals → Contributor    │
└─────────────────────┴─────────────────────────────────────┘

Key rules:
├── No one gets Owner on production subscriptions
├── Developers get Contributor on dev, Reader on prod
├── CI/CD service principals get scoped Contributor on their RG only
├── Use PIM (Privileged Identity Management) for just-in-time access
└── Break-glass account: 2 global admins, MFA, no daily use
```

```bash
# Create custom role (example: deploy-only, no delete)
az role definition create --role-definition '{
  "Name": "App Deployer",
  "Description": "Can deploy but not delete resources",
  "Actions": [
    "Microsoft.Resources/deployments/*",
    "Microsoft.ContainerService/managedClusters/read",
    "Microsoft.ContainerRegistry/registries/push/action"
  ],
  "NotActions": [
    "Microsoft.Resources/subscriptions/resourceGroups/delete",
    "*/delete"
  ],
  "AssignableScopes": ["/subscriptions/<sub-id>"]
}'
```

### B. Networking (Hub-Spoke)

```
┌──────────────────────────────────────────────────────────────┐
│  HUB-SPOKE NETWORK TOPOLOGY                                 │
│                                                              │
│                    Internet                                  │
│                       │                                      │
│                  Azure Firewall                              │
│                       │                                      │
│              ┌────────┴────────┐                             │
│              │    HUB VNet     │  (Connectivity Sub)         │
│              │  10.0.0.0/16   │                              │
│              │                 │                              │
│              │ ├── AzFW Subnet │ 10.0.1.0/24                │
│              │ ├── GW Subnet   │ 10.0.2.0/24 (VPN/ER)      │
│              │ ├── Bastion Sub │ 10.0.3.0/24                │
│              │ └── DNS Resolver│ 10.0.4.0/24                │
│              └────────┬────────┘                             │
│           ┌───────────┼───────────┐                          │
│           │ (peering) │ (peering) │                          │
│     ┌─────┴─────┐ ┌──┴──────┐ ┌──┴──────┐                  │
│     │ Spoke A   │ │ Spoke B │ │ Spoke C │                   │
│     │ Client A  │ │ Client B│ │ Shared  │                   │
│     │10.1.0.0/16│ │10.2.0/16│ │10.3.0/16│                  │
│     │           │ │         │ │         │                    │
│     │├─App Sub  │ │├─App    │ │├─ACR    │                   │
│     │├─DB Sub   │ │├─DB     │ │├─KeyVault│                  │
│     │└─AKS Sub  │ │└─AKS   │ │└─Monitor│                   │
│     └───────────┘ └─────────┘ └─────────┘                   │
│                                                              │
│  Traffic flow: Spoke A → Hub Firewall → Spoke B             │
│  No direct spoke-to-spoke without going through hub.        │
│  All internet egress goes through Azure Firewall.           │
└──────────────────────────────────────────────────────────────┘
```

**CIDR planning (critical — get this wrong and you'll re-IP later):**

```
Reserve large blocks upfront. You CANNOT change VNet address space
if peerings exist without deleting and recreating them.

Hub:       10.0.0.0/16   (65K addresses)
Spoke A:   10.1.0.0/16
Spoke B:   10.2.0.0/16
Spoke C:   10.3.0.0/16
...
Spoke Z:   10.26.0.0/16
On-prem:   172.16.0.0/12 (must not overlap with any Azure range)

Inside each spoke, standardize subnet layout:
├── app-subnet:     10.X.1.0/24  (256 addresses)
├── db-subnet:      10.X.2.0/24
├── aks-subnet:     10.X.4.0/22  (1024 addresses — AKS needs many IPs)
├── pe-subnet:      10.X.8.0/24  (private endpoints)
└── reserved:       10.X.0.0/24  (don't use — gateway/management)
```

```bash
# Terraform module for spoke VNet (reusable)
# modules/spoke-vnet/main.tf
resource "azurerm_virtual_network" "spoke" {
  name                = "vnet-${var.spoke_name}-${var.env}"
  resource_group_name = var.rg_name
  location            = var.location
  address_space       = [var.address_space]  # e.g. "10.1.0.0/16"
}

resource "azurerm_subnet" "app" {
  name                 = "snet-app"
  resource_group_name  = var.rg_name
  virtual_network_name = azurerm_virtual_network.spoke.name
  address_prefixes     = [cidrsubnet(var.address_space, 8, 1)]  # 10.X.1.0/24
}

resource "azurerm_virtual_network_peering" "spoke_to_hub" {
  name                      = "peer-${var.spoke_name}-to-hub"
  resource_group_name       = var.rg_name
  virtual_network_name      = azurerm_virtual_network.spoke.name
  remote_virtual_network_id = var.hub_vnet_id
  allow_forwarded_traffic   = true
  use_remote_gateways       = var.use_gateways
}
```

### C. Governance (Azure Policy)

```
Policies to apply at Landing Zones MG (inherited by all workloads):

DENY policies (block non-compliant resources):
├── Deny public IP creation
├── Deny resources in non-approved regions
├── Deny storage accounts without HTTPS
├── Deny VMs without managed disks
├── Deny resources without required tags (env, team, cost-center)
└── Deny SQL Server without private endpoint

DEPLOY-IF-NOT-EXISTS policies (auto-remediate):
├── Deploy diagnostic settings to Log Analytics
├── Deploy Azure Monitor agent on VMs
├── Deploy NSG flow logs
├── Deploy Microsoft Defender for Cloud
└── Deploy backup policy on VMs

AUDIT policies (log non-compliance without blocking):
├── Audit Key Vaults without soft-delete
├── Audit storage without encryption at rest
└── Audit resources missing tags
```

```bash
# Deny public IP creation (Terraform)
resource "azurerm_policy_assignment" "deny_public_ip" {
  name                 = "deny-public-ip"
  scope                = data.azurerm_management_group.landing_zones.id
  policy_definition_id = "/providers/Microsoft.Authorization/policyDefinitions/..."
  display_name         = "Deny Public IP Creation"
  enforcement_mode     = true
}
```

### D. Monitoring & Logging (Central)

```
┌──────────────────────────────────────────────────────────────┐
│  CENTRALIZED LOGGING ARCHITECTURE                            │
│                                                              │
│  Management Subscription:                                    │
│  └── Log Analytics Workspace (central — all subs send here) │
│      ├── Activity Logs (all subscriptions)                  │
│      ├── Diagnostic Logs (all resources via policy)          │
│      ├── VM Guest Logs (via AMA agent)                      │
│      ├── AKS Container Logs (Container Insights)            │
│      ├── NSG Flow Logs                                      │
│      └── Key Vault Access Logs                              │
│                                                              │
│  Alerts configured:                                          │
│  ├── Service Health alerts                                  │
│  ├── Budget alerts (80% and 100%)                           │
│  ├── Security alerts (Defender for Cloud)                   │
│  ├── Resource health alerts                                 │
│  └── Custom metric alerts per workload                      │
│                                                              │
│  Retention: 90 days online, archive to Storage after        │
└──────────────────────────────────────────────────────────────┘
```

### E. Security Baseline

```
Every landing zone subscription gets:
├── Microsoft Defender for Cloud (Standard tier)
├── Key Vault (one per workload, RBAC access model)
├── Managed Identities (no stored credentials)
├── Private Endpoints for all PaaS services
├── NSG on every subnet (deny-all-inbound default)
├── Azure Bastion for VM access (no public SSH)
├── DDoS Protection Standard on Hub VNet
└── Azure Firewall for egress filtering
```

---

## 4. Types of Landing Zones by Workload

### Type 1: VM-Based Workload Landing Zone

```
For: Lift-and-shift migrations, legacy apps, custom VMs

Resources provisioned:
├── Resource Group (tagged)
├── Spoke VNet (peered to hub)
│   ├── App subnet + NSG
│   ├── DB subnet + NSG
│   └── Management subnet + NSG
├── VM(s) in Availability Set or Zone
├── Azure Load Balancer (Standard)
├── Key Vault (for secrets)
├── Storage Account (for boot diag + backups)
├── Recovery Services Vault (VM backup)
├── Log Analytics (diagnostic settings)
└── Azure Bastion access (via hub)

Terraform structure:
landing-zone-vm/
├── main.tf          # VMs, disks, NICs
├── networking.tf    # spoke VNet, subnets, NSGs, peering
├── security.tf      # Key Vault, managed identity
├── monitoring.tf    # diagnostic settings, alerts
├── backup.tf        # Recovery Services Vault
├── variables.tf
├── outputs.tf
└── terraform.tfvars
```

### Type 2: AKS (Kubernetes) Landing Zone

```
For: Containerized microservices, modern cloud-native apps

Resources provisioned:
├── Resource Group
├── Spoke VNet (peered to hub)
│   ├── AKS subnet (/22 — needs many IPs for pods)
│   ├── Private endpoint subnet
│   └── Internal LB subnet
├── AKS Cluster
│   ├── System node pool (CriticalAddonsOnly taint)
│   ├── User node pool (app workloads)
│   ├── Azure CNI networking
│   ├── Private cluster (API not public)
│   ├── Managed Identity (not service principal)
│   └── Azure AD RBAC integration
├── ACR (Azure Container Registry, private endpoint)
├── Key Vault (secrets via CSI driver)
├── Azure Database for PostgreSQL (private endpoint)
├── Log Analytics + Container Insights
├── Azure Monitor Managed Grafana (optional)
└── Application Gateway / Nginx Ingress

Terraform structure:
landing-zone-aks/
├── main.tf           # AKS cluster, node pools
├── networking.tf     # VNet, subnets, peering, private DNS
├── acr.tf            # Container registry + private endpoint
├── database.tf       # PostgreSQL + private endpoint
├── keyvault.tf       # Key Vault + secrets
├── monitoring.tf     # Log Analytics, Container Insights
├── ingress.tf        # Application Gateway or Ingress
├── variables.tf
├── outputs.tf
└── environments/
    ├── dev.tfvars
    ├── staging.tfvars
    └── prod.tfvars
```

### Type 3: App Service Landing Zone

```
For: Web apps, APIs, low-ops workloads

Resources provisioned:
├── Resource Group
├── Spoke VNet (peered to hub)
│   ├── App Service VNet Integration subnet (/26 min)
│   └── Private endpoint subnet
├── App Service Plan (Isolated v2 for compliance)
├── App Service / Function App
│   ├── VNet integrated (outbound via VNet)
│   ├── Private endpoint (inbound only via VNet)
│   ├── Managed Identity enabled
│   └── Application Insights
├── Azure Front Door / App Gateway (WAF)
├── Azure SQL or PostgreSQL (private endpoint)
├── Key Vault (app config references)
├── Storage Account (private endpoint)
└── Log Analytics
```

### Type 4: Data Platform Landing Zone

```
For: Data engineering, analytics, ML workloads

Resources provisioned:
├── Resource Group
├── Spoke VNet (peered to hub)
├── Azure Data Lake Storage Gen2 (private endpoint)
├── Azure Databricks Workspace
│   ├── Private subnet (for worker nodes)
│   └── Public subnet (for cluster NAT)
├── Azure Synapse Analytics (private endpoint)
├── Azure Data Factory (managed VNet, private endpoints)
├── Azure Key Vault
├── Azure Purview / Microsoft Purview (data catalog)
├── Event Hub / IoT Hub (data ingestion)
└── Log Analytics + diagnostic settings
```

---

## 5. Landing Zone Deployment Automation

### Subscription Vending Machine

```
┌──────────────────────────────────────────────────────────────┐
│  SUBSCRIPTION VENDING = Automate new project onboarding     │
│                                                              │
│  Client request: "I need an environment for Project X"      │
│       │                                                      │
│       ▼                                                      │
│  Fill out YAML config:                                      │
│  ├── project_name: project-x                                │
│  ├── workload_type: aks | vm | appservice                   │
│  ├── environment: dev | staging | prod                      │
│  ├── region: eastus                                         │
│  ├── team: backend-team                                     │
│  ├── budget: 5000                                           │
│  └── cidr_block: 10.5.0.0/16                               │
│       │                                                      │
│       ▼                                                      │
│  CI/CD pipeline runs Terraform:                             │
│  ├── Creates subscription (or uses existing)                │
│  ├── Moves sub under correct Management Group               │
│  ├── Creates spoke VNet, peers to hub                       │
│  ├── Applies policies                                       │
│  ├── Creates Key Vault, storage, monitoring                 │
│  ├── Assigns RBAC roles                                     │
│  ├── Sets budget alerts                                     │
│  └── Outputs: VNet info, Key Vault URI, connection details  │
│       │                                                      │
│       ▼                                                      │
│  Team can deploy their workload immediately.                │
│  Time: 15 minutes instead of 2 weeks of manual setup.       │
└──────────────────────────────────────────────────────────────┘
```

```yaml
# landing-zone-request.yaml (input to vending pipeline)
project_name: "ecommerce-api"
workload_type: "aks"
environment: "production"
region: "eastus2"
team_name: "backend"
team_email: "backend@company.com"
budget_monthly_usd: 3000
cidr_block: "10.5.0.0/16"
features:
  database: "postgresql"
  cache: "redis"
  container_registry: true
  waf: true
rbac:
  contributors:
    - "backend-team-group-id"
  readers:
    - "management-group-id"
tags:
  cost_center: "CC-4521"
  data_classification: "confidential"
```

---

## 6. AWS Landing Zone (Control Tower)

```
┌──────────────────────────────────────────────────────────────┐
│  AWS CONTROL TOWER — Equivalent of Azure Landing Zones      │
│                                                              │
│  Organization Root                                          │
│  ├── Security OU                                            │
│  │   ├── Log Archive Account     → CloudTrail, Config logs  │
│  │   └── Audit Account           → GuardDuty, Security Hub  │
│  ├── Sandbox OU                                             │
│  │   └── Developer sandbox accounts                        │
│  ├── Workloads OU                                           │
│  │   ├── Prod OU                                           │
│  │   │   ├── Client-A-Prod Account                        │
│  │   │   └── Client-B-Prod Account                        │
│  │   └── Non-Prod OU                                       │
│  │       ├── Client-A-Dev Account                         │
│  │       └── Client-A-Staging Account                     │
│  └── Infrastructure OU                                      │
│      ├── Networking Account  → Transit GW, VPN, DNS        │
│      └── Shared Services Account → CI/CD, artifacts        │
│                                                              │
│  AWS equivalents:                                            │
│  Azure MG        → AWS OU (Organizational Unit)             │
│  Azure Sub       → AWS Account                              │
│  Azure Policy    → AWS SCP (Service Control Policy)         │
│  Azure Firewall  → AWS Network Firewall / TGW              │
│  VNet Peering    → Transit Gateway attachments              │
│  Azure Bastion   → AWS Systems Manager Session Manager     │
│  Azure AD PIM    → AWS IAM Identity Center                 │
└──────────────────────────────────────────────────────────────┘
```

```
AWS networking pattern (Transit Gateway):

On-Prem ──VPN──► Transit Gateway ◄── VPC A (10.1.0.0/16)
                      │
                 ┌────┴────┐
                 ▼         ▼
           VPC B          VPC C
        (10.2.0.0/16)  (10.3.0.0/16)

Transit Gateway = hub. All VPCs attach to it.
Route tables on TGW control who can talk to whom.
```

---

## 7. Landing Zone Anti-Patterns

```
┌──────────────────────────────────────────────────────────────┐
│  WHAT GOES WRONG WITHOUT PROPER LANDING ZONES               │
│                                                              │
│  ❌ "Just create a resource group and start"                │
│  → No VNet planning → overlapping CIDRs → can't peer       │
│  → Re-IP entire environment later (days of downtime)        │
│                                                              │
│  ❌ "Everyone gets Owner on the subscription"               │
│  → Junior dev deletes prod resource group                   │
│  → No audit trail of who changed what                       │
│                                                              │
│  ❌ "We'll add policies later"                              │
│  → 200 resources already non-compliant                      │
│  → Retroactive policy enforcement = mass outage             │
│                                                              │
│  ❌ "Each team manages their own logging"                   │
│  → Security incident → no central logs → can't investigate  │
│  → Compliance audit fails                                   │
│                                                              │
│  ❌ "One subscription for everything"                       │
│  → Hit subscription limits (800 RGs, vCPU quotas)          │
│  → Can't separate billing per client                        │
│  → Can't apply different policies to dev vs prod            │
│                                                              │
│  ❌ "We don't need a firewall in dev"                       │
│  → Dev environment becomes attack vector to prod            │
│  → Data exfiltration via dev subscription                   │
└──────────────────────────────────────────────────────────────┘
```

---

## 8. Checklist: Landing Zone Readiness

```
Identity & Access:
[ ] Management Group hierarchy created
[ ] RBAC roles defined per MG/subscription level
[ ] Break-glass accounts configured (2, MFA, monitored)
[ ] PIM enabled for privileged roles
[ ] Service Principals scoped to minimum required

Networking:
[ ] Hub VNet with Azure Firewall / NVA deployed
[ ] CIDR plan documented (no overlaps, room for growth)
[ ] Spoke VNet template ready (Terraform module)
[ ] Peering automated (spoke ↔ hub)
[ ] DNS: Private DNS Zones linked to hub
[ ] Bastion deployed in hub (no public SSH anywhere)
[ ] NSG baseline applied to all subnets
[ ] UDR forcing traffic through firewall

Governance:
[ ] Naming convention documented and enforced via policy
[ ] Tagging policy (env, team, cost-center) — deny if missing
[ ] Allowed regions policy
[ ] Deny public IP policy (or audit)
[ ] Deny public endpoints on PaaS policy

Security:
[ ] Defender for Cloud enabled (all subscriptions)
[ ] Key Vault per workload (RBAC model, purge protection)
[ ] Managed Identities (no stored secrets)
[ ] Private Endpoints for all PaaS services
[ ] DDoS Protection on hub VNet

Monitoring:
[ ] Central Log Analytics Workspace
[ ] Diagnostic settings auto-deployed via policy
[ ] Budget alerts on every subscription
[ ] Service Health alerts configured
[ ] Activity Log forwarded to central workspace

Automation:
[ ] Subscription vending pipeline (onboard new projects)
[ ] Spoke VNet Terraform module tested
[ ] Landing zone templates for each workload type
[ ] CI/CD pipeline for infrastructure changes
[ ] Drift detection (terraform plan -refresh-only on schedule)
```
