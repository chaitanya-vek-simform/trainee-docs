[🏠 Home](../README.md) · [Azure](README.md)

# ☁️ Microsoft Azure — Comprehensive DevOps Handbook

> **Audience:** DevOps / Cloud Engineers & Trainees | **Focus:** Azure Services, Architecture, Automation, Production Readiness
> **Covers:** Subscriptions & ARM, Compute, Networking, Storage, AKS, DevOps Pipelines, IaC (Bicep/Terraform), Security, HA/DR, FinOps
> **Tip:** Sections build on each other — read sequentially first, then use TOC for reference.

---

## Table of Contents

1. [Why Azure for DevOps](#1-why-azure-for-devops)
2. [Account, Subscription & Tenant Structure](#2-account-subscription--tenant-structure)
3. [Resource Groups & Azure Resource Manager](#3-resource-groups--azure-resource-manager)
4. [Azure CLI, Portal & Cloud Shell](#4-azure-cli-portal--cloud-shell)
5. [Compute Services](#5-compute-services)
6. [Networking Fundamentals](#6-networking-fundamentals)
7. [Storage Services](#7-storage-services)
8. [Azure AD (Entra ID) & RBAC](#8-azure-ad-entra-id--rbac)
9. [Azure Kubernetes Service (AKS)](#9-azure-kubernetes-service-aks)
10. [Azure DevOps & Pipelines](#10-azure-devops--pipelines)
11. [Container Registry & Container Instances](#11-container-registry--container-instances)
12. [Azure Functions & Serverless](#12-azure-functions--serverless)
13. [Monitoring & Observability](#13-monitoring--observability)
14. [IaC — ARM Templates, Bicep & Terraform](#14-iac--arm-templates-bicep--terraform)
15. [Advanced Networking](#15-advanced-networking)
16. [Security & Governance](#16-security--governance)
17. [Cost Management & FinOps](#17-cost-management--finops)
18. [HA & Disaster Recovery](#18-ha--disaster-recovery)
19. [Migration Strategies](#19-migration-strategies)
20. [Production Scenario FAQ](#20-production-scenario-faq)

---

## 1. Why Azure for DevOps

### 1.1 Azure's Position in the Cloud Market

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  CLOUD MARKET SHARE (2024)                                       │
  │                                                                  │
  │  AWS  ████████████████████████████  ~31%                        │
  │  Azure████████████████████████  ~25%                             │
  │  GCP  ████████████  ~11%                                        │
  │  Others████████  ~33%                                            │
  │                                                                  │
  │  KEY INSIGHT:                                                    │
  │  Azure is the #1 choice for enterprises already using:          │
  │  ├── Microsoft 365 / Office 365                                 │
  │  ├── Active Directory                                            │
  │  ├── Windows Server workloads                                   │
  │  ├── .NET / SQL Server applications                             │
  │  └── LinkedIn, GitHub, VS Code ecosystem                       │
  │                                                                  │
  │  Azure's DevOps advantage:                                       │
  │  ├── Azure DevOps (CI/CD built-in)                              │
  │  ├── GitHub integration (Microsoft-owned)                       │
  │  ├── Best hybrid cloud story (Azure Arc + Stack)               │
  │  └── Strongest enterprise compliance certifications            │
  └──────────────────────────────────────────────────────────────────┘
```

### 1.2 Azure vs AWS vs GCP — Quick Decision Framework

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  WHEN TO CHOOSE AZURE                                            │
  │                                                                  │
  │  Choose Azure if you have:                                       │
  │  ✅ Existing Microsoft ecosystem (AD, M365, Windows)            │
  │  ✅ Enterprise compliance requirements (FedRAMP, HIPAA)         │
  │  ✅ Hybrid cloud needs (on-prem + cloud)                        │
  │  ✅ .NET / SQL Server workloads                                  │
  │  ✅ Need Azure DevOps for project management + CI/CD           │
  │                                                                  │
  │  Choose AWS if you need:                                         │
  │  ✅ Broadest service catalog                                     │
  │  ✅ Most mature serverless (Lambda)                              │
  │  ✅ Largest partner ecosystem                                    │
  │                                                                  │
  │  Choose GCP if you need:                                         │
  │  ✅ Best Kubernetes experience (GKE)                             │
  │  ✅ Data/ML/AI workloads (BigQuery, Vertex AI)                  │
  │  ✅ Best global network                                          │
  │                                                                  │
  │  SERVICE NAME MAPPING:                                           │
  │  ┌──────────────┬──────────────┬──────────────┐                │
  │  │ AWS          │ Azure        │ GCP          │                │
  │  ├──────────────┼──────────────┼──────────────┤                │
  │  │ EC2          │ VMs          │ Compute Eng. │                │
  │  │ S3           │ Blob Storage │ Cloud Storage│                │
  │  │ VPC          │ VNet         │ VPC          │                │
  │  │ IAM          │ Entra ID     │ IAM          │                │
  │  │ EKS          │ AKS          │ GKE          │                │
  │  │ Lambda       │ Functions    │ Cloud Funct. │                │
  │  │ RDS          │ SQL Database │ Cloud SQL    │                │
  │  │ CloudWatch   │ Monitor      │ Cloud Monit. │                │
  │  │ CodePipeline │ DevOps Pipes │ Cloud Build  │                │
  │  │ CloudFront   │ Front Door   │ Cloud CDN    │                │
  │  │ Route 53     │ Azure DNS    │ Cloud DNS    │                │
  │  └──────────────┴──────────────┴──────────────┘                │
  └──────────────────────────────────────────────────────────────────┘
```

### 1.3 The Azure Certification Path

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  AZURE CERTIFICATION ROADMAP FOR DEVOPS                         │
  │                                                                  │
  │  Foundation:                                                     │
  │  └── AZ-900: Azure Fundamentals (entry level)                  │
  │                                                                  │
  │  Associate:                                                      │
  │  ├── AZ-104: Azure Administrator                                │
  │  ├── AZ-204: Azure Developer                                    │
  │  └── AZ-400: Azure DevOps Engineer Expert (★ KEY CERT)         │
  │                                                                  │
  │  Expert / Specialty:                                             │
  │  ├── AZ-305: Azure Solutions Architect Expert                  │
  │  ├── AZ-500: Azure Security Engineer                            │
  │  └── AZ-700: Azure Network Engineer                             │
  │                                                                  │
  │  For DevOps: AZ-900 → AZ-104 → AZ-400                         │
  └──────────────────────────────────────────────────────────────────┘
```

> **DevOps Relevance:** Azure's biggest advantage is seamless integration with the Microsoft ecosystem. If your company already uses Active Directory, Microsoft 365, and Windows Server — Azure provides the smoothest cloud migration path. Azure DevOps gives you an all-in-one platform (repos, boards, pipelines, artifacts) that competes with GitHub + Jenkins + Jira combined.

---

## 2. Account, Subscription & Tenant Structure

### 2.1 Azure's Organizational Hierarchy

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  AZURE HIERARCHY — FROM TOP TO BOTTOM                           │
  │                                                                  │
  │  ┌─────────────────────────────────┐                            │
  │  │  Azure AD Tenant (Entra ID)     │ ← Identity boundary       │
  │  │  (e.g., mycompany.onmicrosoft)  │   One per organization    │
  │  └──────────┬──────────────────────┘                            │
  │             │                                                    │
  │  ┌──────────▼──────────────────────┐                            │
  │  │  Management Groups             │ ← Policy & RBAC            │
  │  │  (organize subscriptions)       │   inheritance point        │
  │  │  e.g., "Production", "Dev"     │                             │
  │  └──────────┬──────────────────────┘                            │
  │             │                                                    │
  │  ┌──────────▼──────────────────────┐                            │
  │  │  Subscriptions                  │ ← Billing boundary         │
  │  │  (billing + resource limit)     │   Separate invoices        │
  │  │  e.g., "Prod-Sub", "Dev-Sub"   │   Per-sub quotas           │
  │  └──────────┬──────────────────────┘                            │
  │             │                                                    │
  │  ┌──────────▼──────────────────────┐                            │
  │  │  Resource Groups               │ ← Lifecycle boundary       │
  │  │  (logical container)            │   Deploy/delete together   │
  │  │  e.g., "rg-myapp-prod-eastus" │                             │
  │  └──────────┬──────────────────────┘                            │
  │             │                                                    │
  │  ┌──────────▼──────────────────────┐                            │
  │  │  Resources                      │ ← Actual cloud services   │
  │  │  (VMs, databases, etc.)         │   VMs, storage, DBs       │
  │  └────────────────────────────────┘                             │
  └──────────────────────────────────────────────────────────────────┘
```

### 2.2 Tenant & Subscription Deep Dive

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  MULTI-SUBSCRIPTION PATTERN (enterprise)                        │
  │                                                                  │
  │  Tenant: mycompany.onmicrosoft.com                              │
  │  │                                                               │
  │  ├── Management Group: "Production"                             │
  │  │   ├── Subscription: "Prod-Web" (budget: $50K/mo)           │
  │  │   │   ├── rg-frontend-prod-eastus                            │
  │  │   │   └── rg-backend-prod-eastus                             │
  │  │   └── Subscription: "Prod-Data" (budget: $30K/mo)          │
  │  │       ├── rg-sql-prod-eastus                                 │
  │  │       └── rg-analytics-prod-eastus                           │
  │  │                                                               │
  │  ├── Management Group: "Non-Production"                         │
  │  │   ├── Subscription: "Development" (budget: $5K/mo)          │
  │  │   │   └── rg-myapp-dev-eastus                                │
  │  │   └── Subscription: "Staging" (budget: $10K/mo)             │
  │  │       └── rg-myapp-staging-eastus                            │
  │  │                                                               │
  │  └── Management Group: "Sandbox"                                │
  │      └── Subscription: "Sandbox" (budget: $1K/mo)              │
  │          └── rg-experiments                                      │
  │                                                                  │
  │  WHY SEPARATE SUBSCRIPTIONS?                                     │
  │  ├── Billing isolation (charge back to teams)                  │
  │  ├── Resource quota separation (VMs, IPs, etc.)               │
  │  ├── RBAC boundary (devs can't touch prod)                    │
  │  └── Policy enforcement at different levels                    │
  └──────────────────────────────────────────────────────────────────┘
```

```bash
# List subscriptions
az account list --output table

# Set active subscription
az account set --subscription "Prod-Web"

# Show current subscription
az account show --output table

# List management groups
az account management-group list --output table
```

> **⚠️ Gotcha (subscription limits):** Each Azure subscription has default limits (quotas) — e.g., 25,000 VMs per subscription, 250 storage accounts per region. If you hit limits, you'll get deployment errors. Check limits with `az vm list-usage --location eastus -o table` and request increases through the Portal.

### 2.3 Naming Conventions

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  AZURE NAMING CONVENTION (Microsoft CAF recommended)            │
  │                                                                  │
  │  Format: {resource-type}-{workload}-{environment}-{region}     │
  │                                                                  │
  │  Examples:                                                       │
  │  ├── rg-myapp-prod-eastus          Resource Group              │
  │  ├── vm-web01-prod-eastus          Virtual Machine             │
  │  ├── vnet-main-prod-eastus         Virtual Network             │
  │  ├── nsg-web-prod-eastus           Network Security Group      │
  │  ├── st-myappprodeastus            Storage (no hyphens!)       │
  │  ├── sql-myapp-prod-eastus         SQL Database                │
  │  ├── aks-main-prod-eastus          AKS Cluster                │
  │  ├── kv-myapp-prod-eastus          Key Vault                   │
  │  ├── pip-lb-prod-eastus            Public IP                   │
  │  └── log-main-prod-eastus          Log Analytics Workspace     │
  │                                                                  │
  │  RULES:                                                          │
  │  ├── Lowercase only (most resources require it)                │
  │  ├── Storage accounts: no hyphens, 3-24 chars, globally unique│
  │  ├── Key Vaults: globally unique, 3-24 chars                   │
  │  └── Be consistent — pick a convention and stick to it         │
  └──────────────────────────────────────────────────────────────────┘
```

> **DevOps Relevance:** The organizational hierarchy is the foundation of your Azure governance. Getting subscriptions, management groups, and naming conventions right from the start saves months of refactoring later. The Cloud Adoption Framework (CAF) landing zone pattern is the golden standard for enterprise Azure setup.

---

## 3. Resource Groups & Azure Resource Manager

### 3.1 How ARM Works

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  AZURE RESOURCE MANAGER (ARM) — THE CONTROL PLANE              │
  │                                                                  │
  │  Every Azure interaction goes through ARM:                       │
  │                                                                  │
  │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────────┐                   │
  │  │Portal│  │ CLI  │  │SDKs  │  │Terraform │                   │
  │  │      │  │(az)  │  │(.NET,│  │(azurerm) │                   │
  │  │      │  │      │  │Python│  │          │                   │
  │  └──┬───┘  └──┬───┘  └──┬───┘  └────┬─────┘                   │
  │     │         │         │            │                          │
  │     ▼         ▼         ▼            ▼                          │
  │  ┌────────────────────────────────────────┐                     │
  │  │  Azure Resource Manager (ARM)          │                     │
  │  │  ├── Authenticates (via Entra ID)      │                     │
  │  │  ├── Authorizes (RBAC)                 │                     │
  │  │  ├── Validates request                 │                     │
  │  │  └── Routes to resource provider      │                     │
  │  └──────────────┬─────────────────────────┘                     │
  │                 │                                                │
  │     ┌───────────┼───────────┐                                   │
  │     ▼           ▼           ▼                                   │
  │  ┌──────┐  ┌──────┐  ┌──────┐                                  │
  │  │Compute│  │Network│  │Storage│                                │
  │  │Provider│ │Provider│ │Provider│                               │
  │  └──────┘  └──────┘  └──────┘                                  │
  └──────────────────────────────────────────────────────────────────┘
```

### 3.2 Resource Group Best Practices

```bash
# Create a resource group
az group create \
    --name rg-myapp-prod-eastus \
    --location eastus \
    --tags environment=production team=backend app=myapp

# List all resource groups
az group list --output table

# List resources in a group
az resource list \
    --resource-group rg-myapp-prod-eastus \
    --output table

# Delete a resource group (DELETES ALL RESOURCES INSIDE)
az group delete --name rg-myapp-dev-eastus --yes --no-wait

# Export resource group as ARM template
az group export \
    --name rg-myapp-prod-eastus \
    --output json > exported-template.json
```

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  RESOURCE GROUP STRATEGIES                                       │
  │                                                                  │
  │  Strategy 1: By Application + Environment                       │
  │  ├── rg-myapp-prod-eastus   (app resources)                    │
  │  ├── rg-myapp-staging-eastus                                    │
  │  └── rg-myapp-dev-eastus                                        │
  │  → Best for: clear ownership, easy cleanup                     │
  │                                                                  │
  │  Strategy 2: By Resource Type                                    │
  │  ├── rg-networking-prod     (all VNets, NSGs)                  │
  │  ├── rg-compute-prod        (all VMs)                           │
  │  └── rg-data-prod           (all databases)                    │
  │  → Best for: shared infrastructure teams                       │
  │                                                                  │
  │  Strategy 3: By Lifecycle                                        │
  │  ├── rg-myapp-persistent    (DB, storage — rarely deleted)     │
  │  └── rg-myapp-ephemeral     (web servers — frequently redeployed)│
  │  → Best for: IaC-managed environments                          │
  │                                                                  │
  │  RULES:                                                          │
  │  ├── A resource belongs to EXACTLY ONE resource group          │
  │  ├── Resources in a group CAN be in different regions          │
  │  ├── The RG location only stores metadata                       │
  │  └── Deleting a RG deletes ALL resources inside it             │
  └──────────────────────────────────────────────────────────────────┘
```

### 3.3 Tags, Locks & Policies

```bash
# --- Tagging ---
# Tag a resource group
az group update \
    --name rg-myapp-prod-eastus \
    --tags environment=production team=backend cost-center=CC1234

# Tag an individual resource
az resource tag \
    --tags environment=production owner=devops \
    --ids /subscriptions/{sub-id}/resourceGroups/rg-myapp-prod-eastus/providers/Microsoft.Compute/virtualMachines/vm-web01

# Find all resources with a specific tag
az resource list --tag environment=production --output table

# --- Resource Locks (prevent accidental deletion) ---
# Create a CanNotDelete lock on production database
az lock create \
    --name "protect-prod-db" \
    --lock-type CanNotDelete \
    --resource-group rg-myapp-prod-eastus \
    --resource-name sql-myapp-prod \
    --resource-type Microsoft.Sql/servers

# Lock types:
# CanNotDelete — can modify but can't delete
# ReadOnly     — can't modify OR delete

# List locks
az lock list --resource-group rg-myapp-prod-eastus --output table

# Delete a lock (must be removed before deleting protected resource)
az lock delete --name "protect-prod-db" \
    --resource-group rg-myapp-prod-eastus
```

> **⚠️ Gotcha (resource group deletion):** `az group delete` is the most dangerous command in Azure — it deletes EVERYTHING in the resource group immediately and irreversibly. ALWAYS use resource locks (`CanNotDelete`) on production resource groups. A single mistyped group name can wipe out your entire production environment.

> **DevOps Relevance:** Resource Groups are how you organize deployments in Azure. In IaC (Terraform/Bicep), you target a resource group for deployment. Tags are essential for cost allocation — without them, you can't answer "how much does app X cost?" Locks prevent the dreaded "someone deleted prod" scenario.

---

## 4. Azure CLI, Portal & Cloud Shell

### 4.1 Azure CLI (az) — Your Primary Tool

```bash
# --- Installation ---
# macOS
brew install azure-cli

# Linux (Ubuntu/Debian)
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Windows
winget install Microsoft.AzureCLI

# Docker
docker run -it mcr.microsoft.com/azure-cli

# --- Authentication ---
# Interactive login (opens browser)
az login

# Service principal login (CI/CD)
az login --service-principal \
    --username $AZURE_CLIENT_ID \
    --password $AZURE_CLIENT_SECRET \
    --tenant $AZURE_TENANT_ID

# Managed identity login (inside Azure VMs/AKS)
az login --identity

# Show current logged-in user
az account show --query user.name -o tsv
```

### 4.2 Essential az CLI Patterns

```bash
# --- Output formats ---
az vm list --output table         # Human-readable table
az vm list --output json          # Full JSON (for scripting)
az vm list --output tsv           # Tab-separated (for piping)
az vm list --output yaml          # YAML format

# --- JMESPath queries (--query) ---
# Get only VM names
az vm list --query "[].name" -o tsv

# Get name + resource group
az vm list --query "[].{Name:name, RG:resourceGroup, Size:hardwareProfile.vmSize}" -o table

# Filter by power state
az vm list -d --query "[?powerState=='VM running'].name" -o tsv

# Count resources
az resource list --query "length([?type=='Microsoft.Compute/virtualMachines'])"

# --- Common patterns ---
# Create and store ID for later use
VM_ID=$(az vm create \
    --resource-group rg-myapp-dev-eastus \
    --name vm-web01 \
    --image Ubuntu2204 \
    --size Standard_B2s \
    --admin-username azureuser \
    --generate-ssh-keys \
    --query id -o tsv)

echo "Created VM: $VM_ID"

# Wait for a long-running operation
az vm create ... --no-wait
az vm wait --name vm-web01 --resource-group rg-myapp-dev-eastus --created

# Find available VM sizes in a region
az vm list-sizes --location eastus --output table

# Find available images
az vm image list --publisher Canonical --offer 0001-com-ubuntu-server-jammy --all --output table
```

### 4.3 Cloud Shell & Azure PowerShell

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  AZURE CLI vs POWERSHELL vs CLOUD SHELL                         │
  │                                                                  │
  │  Azure CLI (az):                                                 │
  │  ├── Cross-platform (Linux, macOS, Windows)                    │
  │  ├── Bash-friendly (pipes, jq, grep)                           │
  │  ├── Consistent verb-noun: az {group} {action}                 │
  │  └── Preferred by: Linux/DevOps engineers                      │
  │                                                                  │
  │  Azure PowerShell (Az module):                                   │
  │  ├── PowerShell syntax: Get-AzVM, New-AzVM                    │
  │  ├── Object pipeline (not text)                                │
  │  ├── Better for complex logic                                   │
  │  └── Preferred by: Windows admins, .NET teams                  │
  │                                                                  │
  │  Azure Cloud Shell:                                              │
  │  ├── Browser-based (shell.azure.com)                           │
  │  ├── Pre-installed: az, kubectl, terraform, docker, git        │
  │  ├── Persists files in Azure storage ($HOME/clouddrive)        │
  │  ├── Choose Bash or PowerShell                                  │
  │  └── Free (you pay for storage backing)                        │
  │                                                                  │
  │  RECOMMENDATION FOR DEVOPS:                                      │
  │  Use az CLI (Bash) for scripts → Cloud Shell for quick tasks   │
  └──────────────────────────────────────────────────────────────────┘
```

```powershell
# Azure PowerShell equivalents
# Install module
Install-Module -Name Az -Scope CurrentUser

# Login
Connect-AzAccount

# List VMs
Get-AzVM | Format-Table Name, ResourceGroupName, Location

# Create a resource group
New-AzResourceGroup -Name "rg-myapp-dev-eastus" -Location "East US"

# Create a VM
New-AzVM `
    -ResourceGroupName "rg-myapp-dev-eastus" `
    -Name "vm-web01" `
    -Image "Ubuntu2204" `
    -Size "Standard_B2s" `
    -Credential (Get-Credential)
```

### 4.4 az CLI Configuration & Defaults

```bash
# Set defaults so you don't repeat --resource-group and --location
az configure --defaults group=rg-myapp-dev-eastus location=eastus

# Now these are equivalent:
az vm list --resource-group rg-myapp-dev-eastus
az vm list  # Uses default group

# Clear defaults
az configure --defaults group='' location=''

# Enable CLI auto-completion (bash)
echo 'source /etc/bash_completion.d/azure-cli' >> ~/.bashrc

# View configuration
az configure --list-defaults

# --- Useful extensions ---
az extension add --name aks-preview
az extension add --name front-door
az extension add --name account       # Management groups

# List installed extensions
az extension list --output table
```

> **⚠️ Gotcha (az login in CI/CD):** Never use `az login` (interactive) in CI/CD pipelines — it opens a browser. Use service principal login with `--service-principal` or managed identity with `--identity`. Store credentials as pipeline secrets, never in scripts.

> **DevOps Relevance:** The `az` CLI is your daily driver for Azure. Learn `--query` (JMESPath) — it eliminates the need for `jq` in most cases. Use `--no-wait` + `az ... wait` for long operations in scripts. Set defaults with `az configure` to reduce typing. Cloud Shell is your emergency tool when you need quick access from any browser.

---

## 5. Compute Services

### 5.1 Azure Compute Landscape

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  AZURE COMPUTE — DECISION TREE                                   │
  │                                                                  │
  │  What are you running?                                           │
  │  │                                                               │
  │  ├── Full OS control needed?                                    │
  │  │   ├── YES → Virtual Machines (IaaS)                         │
  │  │   │         ├── Single VM: az vm create                     │
  │  │   │         ├── Scale set: VM Scale Sets (VMSS)             │
  │  │   │         └── Spot VMs: up to 90% cost savings            │
  │  │   └── NO → Managed platform                                 │
  │  │                                                               │
  │  ├── Web app / API?                                              │
  │  │   ├── Simple: App Service (PaaS)                            │
  │  │   ├── Containers: App Service for Containers                │
  │  │   └── Microservices: AKS (Kubernetes)                       │
  │  │                                                               │
  │  ├── Event-driven / short-lived?                                │
  │  │   └── Azure Functions (serverless)                          │
  │  │                                                               │
  │  ├── Containers (no K8s)?                                       │
  │  │   └── Azure Container Instances (ACI)                       │
  │  │                                                               │
  │  └── Batch processing?                                           │
  │      └── Azure Batch                                             │
  │                                                                  │
  │  COST COMPARISON (approximate, per month):                       │
  │  ├── Functions: $0 (free tier) to pay-per-execution            │
  │  ├── App Service B1: ~$13/mo                                    │
  │  ├── VM Standard_B2s: ~$30/mo                                   │
  │  ├── VM Standard_D4s_v5: ~$140/mo                              │
  │  └── AKS (3-node D4s): ~$420/mo + management free             │
  └──────────────────────────────────────────────────────────────────┘
```

### 5.2 Virtual Machines

```bash
# --- Create a VM ---
az vm create \
    --resource-group rg-myapp-prod-eastus \
    --name vm-web01-prod-eastus \
    --image Ubuntu2204 \
    --size Standard_D2s_v5 \
    --admin-username azureuser \
    --generate-ssh-keys \
    --zone 1 \
    --nsg nsg-web-prod-eastus \
    --vnet-name vnet-main-prod-eastus \
    --subnet snet-web \
    --public-ip-address "" \
    --tags environment=production app=myapp

# --- VM Sizes (most common for DevOps) ---
# B-series: Burstable (dev/test)        Standard_B2s   (2 vCPU, 4 GB)
# D-series: General purpose (prod)      Standard_D4s_v5 (4 vCPU, 16 GB)
# E-series: Memory optimized (DBs)      Standard_E8s_v5 (8 vCPU, 64 GB)
# F-series: Compute optimized (CPU)     Standard_F4s_v2 (4 vCPU, 8 GB)
# L-series: Storage optimized           Standard_L8s_v3 (8 vCPU, 64 GB)

# List available sizes
az vm list-sizes --location eastus --output table | grep -E "Standard_(B|D|E).*s"

# Resize a VM
az vm resize \
    --resource-group rg-myapp-prod-eastus \
    --name vm-web01-prod-eastus \
    --size Standard_D4s_v5

# Start / Stop / Deallocate
az vm start --name vm-web01 --resource-group rg-myapp-prod-eastus
az vm stop --name vm-web01 --resource-group rg-myapp-prod-eastus       # Still billed!
az vm deallocate --name vm-web01 --resource-group rg-myapp-prod-eastus # Stops billing

# SSH into a VM
az ssh vm --name vm-web01 --resource-group rg-myapp-prod-eastus

# Run a command on a VM (without SSH)
az vm run-command invoke \
    --resource-group rg-myapp-prod-eastus \
    --name vm-web01 \
    --command-id RunShellScript \
    --scripts "df -h && free -m"
```

> **⚠️ Gotcha (stop vs deallocate):** `az vm stop` stops the OS but keeps the VM allocated — **you're still billed for compute**. `az vm deallocate` releases the compute resources and stops billing. In scripts, always use `deallocate` unless you need the VM to restart quickly.

### 5.3 VM Scale Sets (VMSS)

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  VM SCALE SETS — AUTO-SCALING VMs                               │
  │                                                                  │
  │        Load Balancer                                             │
  │             │                                                    │
  │    ┌────────┼────────┐                                          │
  │    ▼        ▼        ▼                                          │
  │  ┌────┐  ┌────┐  ┌────┐                                        │
  │  │VM-0│  │VM-1│  │VM-2│  ← identical instances                 │
  │  └────┘  └────┘  └────┘                                        │
  │                          ┌────┐                                  │
  │  Auto-scale rules:       │VM-3│  ← added when CPU > 70%       │
  │  CPU > 70% → add VM     └────┘                                  │
  │  CPU < 30% → remove VM  ┌────┐                                  │
  │  Min: 2, Max: 10        │VM-4│  ← added during peak           │
  │                          └────┘                                  │
  └──────────────────────────────────────────────────────────────────┘
```

```bash
# Create a VMSS
az vmss create \
    --resource-group rg-myapp-prod-eastus \
    --name vmss-web-prod \
    --image Ubuntu2204 \
    --vm-sku Standard_D2s_v5 \
    --instance-count 3 \
    --admin-username azureuser \
    --generate-ssh-keys \
    --zones 1 2 3 \
    --lb-sku Standard \
    --upgrade-policy-mode Rolling

# Configure auto-scale
az monitor autoscale create \
    --resource-group rg-myapp-prod-eastus \
    --resource vmss-web-prod \
    --resource-type Microsoft.Compute/virtualMachineScaleSets \
    --min-count 2 --max-count 10 --count 3

# Add scale-out rule (CPU > 70%)
az monitor autoscale rule create \
    --resource-group rg-myapp-prod-eastus \
    --autoscale-name vmss-web-prod \
    --condition "Percentage CPU > 70 avg 5m" \
    --scale out 2

# Add scale-in rule (CPU < 30%)
az monitor autoscale rule create \
    --resource-group rg-myapp-prod-eastus \
    --autoscale-name vmss-web-prod \
    --condition "Percentage CPU < 30 avg 5m" \
    --scale in 1
```

### 5.4 App Service (PaaS)

```bash
# Create an App Service Plan + Web App
az appservice plan create \
    --name asp-myapp-prod \
    --resource-group rg-myapp-prod-eastus \
    --sku P1v3 \
    --is-linux

az webapp create \
    --resource-group rg-myapp-prod-eastus \
    --plan asp-myapp-prod \
    --name myapp-prod-eastus \
    --runtime "PYTHON:3.12"

# Deploy from GitHub
az webapp deployment source config \
    --name myapp-prod-eastus \
    --resource-group rg-myapp-prod-eastus \
    --repo-url https://github.com/myorg/myapp \
    --branch main

# Deploy from Docker container
az webapp create \
    --resource-group rg-myapp-prod-eastus \
    --plan asp-myapp-prod \
    --name myapp-prod-eastus \
    --deployment-container-image-name myacr.azurecr.io/myapp:v2.3.1

# Set environment variables (App Settings)
az webapp config appsettings set \
    --resource-group rg-myapp-prod-eastus \
    --name myapp-prod-eastus \
    --settings DB_HOST=sql-myapp-prod.database.windows.net \
               REDIS_URL=redis-myapp-prod.redis.cache.windows.net

# Enable deployment slots (blue-green)
az webapp deployment slot create \
    --name myapp-prod-eastus \
    --resource-group rg-myapp-prod-eastus \
    --slot staging

# Swap staging → production (zero-downtime)
az webapp deployment slot swap \
    --name myapp-prod-eastus \
    --resource-group rg-myapp-prod-eastus \
    --slot staging --target-slot production
```

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  APP SERVICE TIERS — WHAT TO PICK                               │
  │                                                                  │
  │  Free/Shared (F1/D1): dev/test only, no SLA, shared infra     │
  │  Basic (B1-B3): dedicated VMs, no auto-scale, no slots         │
  │  Standard (S1-S3): auto-scale, deployment slots, VNet          │
  │  Premium (P1v3-P3v3): ★ production — more CPU/RAM, slots     │
  │  Isolated (I1v2-I3v2): dedicated environment (ASE)             │
  │                                                                  │
  │  FOR DEVOPS: Use Premium v3 for production                     │
  │  (deployment slots, auto-scale, VNet integration)              │
  └──────────────────────────────────────────────────────────────────┘
```

> **DevOps Relevance:** VMs + VMSS give you full control for legacy workloads or custom infrastructure. App Service is the fastest way to deploy web apps with built-in CI/CD, deployment slots, and auto-scaling. Use deployment slots for zero-downtime releases — deploy to staging, test, then swap to production.

---

## 6. Networking Fundamentals

### 6.1 Virtual Network (VNet) Architecture

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  AZURE VNET — THE NETWORKING FOUNDATION                         │
  │                                                                  │
  │  VNet: vnet-main-prod-eastus (10.0.0.0/16)                     │
  │  │                                                               │
  │  ├── Subnet: snet-web (10.0.1.0/24)                            │
  │  │   ├── NSG: nsg-web (allow 80, 443 inbound)                 │
  │  │   ├── vm-web01 (10.0.1.4)                                   │
  │  │   └── vm-web02 (10.0.1.5)                                   │
  │  │                                                               │
  │  ├── Subnet: snet-app (10.0.2.0/24)                            │
  │  │   ├── NSG: nsg-app (allow from snet-web only)              │
  │  │   ├── vm-app01 (10.0.2.4)                                   │
  │  │   └── vm-app02 (10.0.2.5)                                   │
  │  │                                                               │
  │  ├── Subnet: snet-db (10.0.3.0/24)                             │
  │  │   ├── NSG: nsg-db (allow 5432 from snet-app only)          │
  │  │   └── sql-myapp-prod (Private Endpoint)                     │
  │  │                                                               │
  │  ├── Subnet: snet-bastion (10.0.255.0/26)                      │
  │  │   └── Azure Bastion (secure SSH/RDP without public IP)      │
  │  │                                                               │
  │  └── Subnet: AzureFirewallSubnet (10.0.254.0/26)              │
  │      └── Azure Firewall (egress filtering)                     │
  │                                                                  │
  │  KEY RULES:                                                      │
  │  ├── VNets are region-specific (vnet-prod-eastus)              │
  │  ├── VNets do NOT span regions (use VNet peering)             │
  │  ├── VNet-to-VNet: peering (same/cross region)                │
  │  ├── VNet-to-OnPrem: VPN Gateway or ExpressRoute              │
  │  └── Each subnet gets 5 reserved IPs (Azure overhead)         │
  └──────────────────────────────────────────────────────────────────┘
```

```bash
# Create a VNet with subnets
az network vnet create \
    --resource-group rg-myapp-prod-eastus \
    --name vnet-main-prod-eastus \
    --address-prefixes 10.0.0.0/16 \
    --subnet-name snet-web \
    --subnet-prefixes 10.0.1.0/24

# Add more subnets
az network vnet subnet create \
    --resource-group rg-myapp-prod-eastus \
    --vnet-name vnet-main-prod-eastus \
    --name snet-app \
    --address-prefixes 10.0.2.0/24

az network vnet subnet create \
    --resource-group rg-myapp-prod-eastus \
    --vnet-name vnet-main-prod-eastus \
    --name snet-db \
    --address-prefixes 10.0.3.0/24

# VNet peering (connect two VNets)
az network vnet peering create \
    --resource-group rg-myapp-prod-eastus \
    --name peer-prod-to-staging \
    --vnet-name vnet-main-prod-eastus \
    --remote-vnet /subscriptions/{sub}/resourceGroups/rg-staging/providers/Microsoft.Network/virtualNetworks/vnet-staging \
    --allow-vnet-access
```

### 6.2 Network Security Groups (NSGs)

```bash
# Create an NSG
az network nsg create \
    --resource-group rg-myapp-prod-eastus \
    --name nsg-web-prod-eastus

# Allow HTTP/HTTPS inbound
az network nsg rule create \
    --resource-group rg-myapp-prod-eastus \
    --nsg-name nsg-web-prod-eastus \
    --name AllowHTTPS \
    --priority 100 \
    --direction Inbound \
    --access Allow \
    --protocol Tcp \
    --destination-port-ranges 443

az network nsg rule create \
    --resource-group rg-myapp-prod-eastus \
    --nsg-name nsg-web-prod-eastus \
    --name AllowHTTP \
    --priority 110 \
    --direction Inbound \
    --access Allow \
    --protocol Tcp \
    --destination-port-ranges 80

# Deny all other inbound (explicit)
az network nsg rule create \
    --resource-group rg-myapp-prod-eastus \
    --nsg-name nsg-web-prod-eastus \
    --name DenyAllInbound \
    --priority 4000 \
    --direction Inbound \
    --access Deny \
    --protocol '*' \
    --destination-port-ranges '*'

# Associate NSG with subnet
az network vnet subnet update \
    --resource-group rg-myapp-prod-eastus \
    --vnet-name vnet-main-prod-eastus \
    --name snet-web \
    --network-security-group nsg-web-prod-eastus

# View effective rules
az network nic list-effective-nsg \
    --resource-group rg-myapp-prod-eastus \
    --name vm-web01VMNic
```

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  NSG RULE PROCESSING ORDER                                       │
  │                                                                  │
  │  Rules processed by PRIORITY (lowest number = first):           │
  │                                                                  │
  │  Priority 100: Allow HTTPS (443)     ← checked first           │
  │  Priority 110: Allow HTTP (80)                                   │
  │  Priority 120: Allow SSH from Bastion                           │
  │  Priority 4000: Deny all other       ← catch-all deny          │
  │  Priority 65000: (Azure default) Allow VNet-to-VNet            │
  │  Priority 65001: (Azure default) Allow Azure LB               │
  │  Priority 65500: (Azure default) Deny all                      │
  │                                                                  │
  │  TIPS:                                                           │
  │  ├── Use increments of 10 (100, 110, 120) for room to insert  │
  │  ├── NSG can attach to subnet OR NIC (both evaluated)         │
  │  ├── Subnet-level NSG = team-wide, NIC-level = per-VM         │
  │  └── Use "Effective security rules" to debug connectivity     │
  └──────────────────────────────────────────────────────────────────┘
```

### 6.3 Load Balancing Options

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  AZURE LOAD BALANCING — DECISION TREE                           │
  │                                                                  │
  │  Is it HTTP/HTTPS traffic?                                       │
  │  │                                                               │
  │  ├── YES                                                         │
  │  │   ├── Single region? → Application Gateway (L7)             │
  │  │   │                     ├── URL-based routing                │
  │  │   │                     ├── SSL termination                  │
  │  │   │                     ├── WAF (Web App Firewall)           │
  │  │   │                     └── Cookie affinity                  │
  │  │   └── Multi-region? → Front Door (global L7)               │
  │  │                        ├── CDN + WAF                         │
  │  │                        ├── SSL offloading                    │
  │  │                        └── Instant failover                  │
  │  │                                                               │
  │  └── NO (TCP/UDP)                                                │
  │      ├── Single region? → Azure Load Balancer (L4)             │
  │      │                     ├── Basic (free, no SLA)             │
  │      │                     └── Standard (SLA, zone-redundant)   │
  │      └── DNS-based? → Traffic Manager                          │
  │                        ├── Weighted routing                     │
  │                        ├── Priority failover                    │
  │                        └── Geographic routing                   │
  └──────────────────────────────────────────────────────────────────┘
```

```bash
# Create a Standard Load Balancer
az network lb create \
    --resource-group rg-myapp-prod-eastus \
    --name lb-web-prod-eastus \
    --sku Standard \
    --frontend-ip-name frontend-ip \
    --backend-pool-name backend-pool \
    --public-ip-address pip-lb-prod-eastus

# Add health probe
az network lb probe create \
    --resource-group rg-myapp-prod-eastus \
    --lb-name lb-web-prod-eastus \
    --name health-probe-http \
    --protocol tcp \
    --port 80

# Add load balancing rule
az network lb rule create \
    --resource-group rg-myapp-prod-eastus \
    --lb-name lb-web-prod-eastus \
    --name http-rule \
    --protocol Tcp \
    --frontend-port 80 \
    --backend-port 80 \
    --frontend-ip-name frontend-ip \
    --backend-pool-name backend-pool \
    --probe-name health-probe-http
```

> **⚠️ Gotcha (Basic vs Standard LB):** Basic Load Balancer has NO SLA, doesn't support availability zones, and can't mix with Standard SKU resources. Always use **Standard** LB for production. Basic LB is being retired — Microsoft recommends upgrading all Basic LBs.

> **DevOps Relevance:** VNets are the foundation of every Azure architecture. Your job as a DevOps engineer is to design the subnet layout (web / app / data / bastion tiers), configure NSG rules (principle of least privilege), and pick the right load balancer. Use Azure Bastion instead of public IPs on VMs — it eliminates the SSH/RDP attack surface entirely.

---

## 7. Storage Services

### 7.1 Azure Storage Overview

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  AZURE STORAGE SERVICES                                          │
  │                                                                  │
  │  ┌────────────────┐  Object storage for unstructured data      │
  │  │  Blob Storage   │  Images, backups, logs, static websites   │
  │  │  (containers)   │  Hot / Cool / Cold / Archive tiers        │
  │  └────────────────┘                                              │
  │                                                                  │
  │  ┌────────────────┐  SMB/NFS file shares                       │
  │  │  Azure Files    │  Mount as network drive on VMs            │
  │  │  (shares)       │  Replace on-prem file servers             │
  │  └────────────────┘                                              │
  │                                                                  │
  │  ┌────────────────┐  Block storage for VMs                     │
  │  │  Managed Disks  │  OS disks, data disks                     │
  │  │                 │  Premium SSD / Standard SSD / HDD         │
  │  └────────────────┘                                              │
  │                                                                  │
  │  ┌────────────────┐  NoSQL key-value (for legacy/simple)       │
  │  │  Table Storage  │  Use Cosmos DB for new projects            │
  │  └────────────────┘                                              │
  │                                                                  │
  │  ┌────────────────┐  Message queues (simple pub/sub)           │
  │  │  Queue Storage  │  Use Service Bus for enterprise           │
  │  └────────────────┘                                              │
  └──────────────────────────────────────────────────────────────────┘
```

### 7.2 Blob Storage Deep Dive

```bash
# Create a storage account
az storage account create \
    --name stmyappprodeastus \
    --resource-group rg-myapp-prod-eastus \
    --location eastus \
    --sku Standard_ZRS \
    --kind StorageV2 \
    --access-tier Hot \
    --min-tls-version TLS1_2

# Create a container
az storage container create \
    --name backups \
    --account-name stmyappprodeastus \
    --auth-mode login

# Upload a file
az storage blob upload \
    --account-name stmyappprodeastus \
    --container-name backups \
    --name "db-backup-2024-01-15.sql.gz" \
    --file ./backup.sql.gz \
    --auth-mode login

# Upload entire directory
az storage blob upload-batch \
    --account-name stmyappprodeastus \
    --destination artifacts \
    --source ./build/

# List blobs
az storage blob list \
    --account-name stmyappprodeastus \
    --container-name backups \
    --output table --auth-mode login

# Generate SAS token (time-limited access)
az storage blob generate-sas \
    --account-name stmyappprodeastus \
    --container-name backups \
    --name "db-backup-2024-01-15.sql.gz" \
    --permissions r \
    --expiry $(date -u -d "+1 hour" '+%Y-%m-%dT%H:%MZ') \
    --auth-mode login
```

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  BLOB ACCESS TIERS — COST vs SPEED                              │
  │                                                                  │
  │  Tier     │ Access Cost │ Storage Cost │ Use Case               │
  │  ─────────┼─────────────┼──────────────┼───────────────────────│
  │  Hot      │ Cheapest    │ Most expensive│ Frequent access       │
  │  Cool     │ Moderate    │ Cheaper      │ 30+ day retention     │
  │  Cold     │ Higher      │ Cheaper      │ 90+ day retention     │
  │  Archive  │ Highest     │ Cheapest     │ Years (rehydrate hrs) │
  │                                                                  │
  │  LIFECYCLE MANAGEMENT (automate tier transitions):              │
  │  ├── After 30 days → Move to Cool                              │
  │  ├── After 90 days → Move to Cold                              │
  │  ├── After 365 days → Move to Archive                          │
  │  └── After 730 days → Delete                                   │
  │                                                                  │
  │  REDUNDANCY OPTIONS:                                             │
  │  ├── LRS:  3 copies, 1 datacenter (cheapest)                  │
  │  ├── ZRS:  3 copies, 3 availability zones (★ recommended)    │
  │  ├── GRS:  6 copies, 2 regions (DR capable)                   │
  │  └── GZRS: 6 copies, 3 zones + secondary region (best)       │
  └──────────────────────────────────────────────────────────────────┘
```

### 7.3 Azure Files & Managed Disks

```bash
# --- Azure Files (SMB share) ---
# Create a file share
az storage share create \
    --name config-share \
    --account-name stmyappprodeastus \
    --quota 100  # GiB

# Mount on Linux VM
sudo mount -t cifs \
    //stmyappprodeastus.file.core.windows.net/config-share \
    /mnt/config \
    -o vers=3.0,username=stmyappprodeastus,password=$STORAGE_KEY,serverino

# --- Managed Disks ---
# Create a Premium SSD data disk
az disk create \
    --resource-group rg-myapp-prod-eastus \
    --name disk-data01-prod \
    --size-gb 256 \
    --sku Premium_LRS \
    --zone 1

# Attach to VM
az vm disk attach \
    --resource-group rg-myapp-prod-eastus \
    --vm-name vm-web01-prod-eastus \
    --name disk-data01-prod

# Create a snapshot (backup)
az snapshot create \
    --resource-group rg-myapp-prod-eastus \
    --source disk-data01-prod \
    --name snap-data01-$(date +%Y%m%d)
```

> **⚠️ Gotcha (storage account naming):** Storage account names must be 3-24 characters, lowercase letters and numbers only — **no hyphens**. They must be globally unique across all of Azure. This often forces awkward names like `stmyappprodeastus` instead of the nice `st-myapp-prod-eastus`.

> **DevOps Relevance:** Blob Storage is where you store deployment artifacts, database backups, Terraform state, and log archives. Always enable lifecycle management to automatically tier old data to cheaper tiers. For secrets/config, never use Blob — use Key Vault. Use ZRS (zone-redundant) for production to survive datacenter failures.

---

## 8. Azure AD (Entra ID) & RBAC

### 8.1 Identity in Azure

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  AZURE IDENTITY — WHO CAN DO WHAT                               │
  │                                                                  │
  │  Identity Types:                                                 │
  │  ├── User: human person (engineer@company.com)                 │
  │  ├── Group: collection of users (devops-team)                  │
  │  ├── Service Principal: app identity (CI/CD pipeline)          │
  │  └── Managed Identity: Azure-assigned (VM, AKS, App Service)  │
  │                                                                  │
  │  RBAC Assignment = WHO + WHAT + WHERE                           │
  │                                                                  │
  │  ┌─────────┐  ┌──────────┐  ┌───────────┐                     │
  │  │ Identity │ + │   Role   │ + │   Scope   │                     │
  │  │ (who)    │  │ (what)   │  │ (where)   │                     │
  │  └─────────┘  └──────────┘  └───────────┘                     │
  │                                                                  │
  │  Example:                                                        │
  │  "devops-team" + "Contributor" + "/subscriptions/prod-sub"     │
  │  → DevOps team can create/modify resources in prod subscription│
  │                                                                  │
  │  SCOPE HIERARCHY (inheritance):                                  │
  │  Management Group → Subscription → Resource Group → Resource   │
  │  (broadest)                                       (narrowest)  │
  └──────────────────────────────────────────────────────────────────┘
```

### 8.2 Built-in RBAC Roles

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  KEY BUILT-IN ROLES                                              │
  │                                                                  │
  │  Role              │ Can Do                     │ Use When      │
  │  ──────────────────┼────────────────────────────┼──────────────│
  │  Owner             │ Everything + assign RBAC  │ Sub admins    │
  │  Contributor       │ Create/modify/delete      │ DevOps teams  │
  │  Reader            │ Read-only                  │ Auditors      │
  │  User Access Admin │ Manage RBAC only          │ IAM admins    │
  │                    │                            │               │
  │  AKS Cluster Admin │ Full AKS access           │ K8s admins    │
  │  AKS Cluster User  │ Get kubeconfig            │ Developers    │
  │  Storage Blob Data │ Read/write blobs          │ App data      │
  │  Contributor       │                            │ access        │
  │  Key Vault Secrets │ Manage secrets             │ Secret ops    │
  │  Officer           │                            │               │
  │  Network Contrib.  │ Manage networking          │ NetOps team   │
  │  SQL DB Contrib.   │ Manage SQL DBs             │ DBA team      │
  │                                                                  │
  │  PRINCIPLE OF LEAST PRIVILEGE:                                   │
  │  ├── Use Reader by default                                     │
  │  ├── Contributor only for teams that deploy                    │
  │  ├── Owner ONLY for subscription administrators               │
  │  └── Use resource-specific roles (Storage Blob, Key Vault)    │
  └──────────────────────────────────────────────────────────────────┘
```

```bash
# --- Assign roles ---
# Give devops-team Contributor on production subscription
az role assignment create \
    --assignee-object-id $(az ad group show --group devops-team --query id -o tsv) \
    --role "Contributor" \
    --scope "/subscriptions/${PROD_SUB_ID}"

# Give a user Reader on a resource group
az role assignment create \
    --assignee user@company.com \
    --role "Reader" \
    --resource-group rg-myapp-prod-eastus

# List role assignments for a scope
az role assignment list \
    --resource-group rg-myapp-prod-eastus \
    --output table

# List all built-in roles
az role definition list --output table --query "[?roleType=='BuiltInRole'].{Name:roleName, Description:description}"
```

### 8.3 Service Principals & Managed Identities

```bash
# --- Service Principal (for CI/CD pipelines) ---
# Create a service principal with Contributor role on subscription
az ad sp create-for-rbac \
    --name "sp-github-actions-prod" \
    --role Contributor \
    --scopes "/subscriptions/${PROD_SUB_ID}" \
    --sdk-auth

# Output (store as GitHub Secret: AZURE_CREDENTIALS):
# {
#   "clientId": "xxxx",
#   "clientSecret": "xxxx",
#   "subscriptionId": "xxxx",
#   "tenantId": "xxxx"
# }

# Create with MORE restrictive scope (recommended)
az ad sp create-for-rbac \
    --name "sp-deploy-myapp" \
    --role Contributor \
    --scopes "/subscriptions/${SUB_ID}/resourceGroups/rg-myapp-prod-eastus"
```

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  SERVICE PRINCIPAL vs MANAGED IDENTITY                           │
  │                                                                  │
  │  Service Principal:                                              │
  │  ├── Manual credential management (client ID + secret)        │
  │  ├── Secrets EXPIRE (1-2 years default)                        │
  │  ├── Must rotate secrets manually                               │
  │  ├── Use for: external CI/CD (GitHub, GitLab, Jenkins)        │
  │  └── Risk: secret leakage                                      │
  │                                                                  │
  │  Managed Identity:                                               │
  │  ├── Azure manages credentials automatically                   │
  │  ├── No secrets to manage or rotate                            │
  │  ├── Only works from Azure resources                           │
  │  ├── Two types:                                                 │
  │  │   ├── System-assigned: tied to resource lifecycle          │
  │  │   └── User-assigned: reusable across resources             │
  │  ├── Use for: Azure VMs, AKS, App Service, Functions          │
  │  └── ★ ALWAYS prefer over service principals inside Azure     │
  └──────────────────────────────────────────────────────────────────┘
```

```bash
# --- Managed Identity ---
# Enable system-assigned managed identity on a VM
az vm identity assign \
    --resource-group rg-myapp-prod-eastus \
    --name vm-web01-prod-eastus

# Grant the VM's identity access to Key Vault
VM_IDENTITY=$(az vm show \
    --resource-group rg-myapp-prod-eastus \
    --name vm-web01-prod-eastus \
    --query identity.principalId -o tsv)

az role assignment create \
    --assignee-object-id "$VM_IDENTITY" \
    --role "Key Vault Secrets User" \
    --scope "/subscriptions/${SUB_ID}/resourceGroups/rg-myapp-prod-eastus/providers/Microsoft.KeyVault/vaults/kv-myapp-prod-eastus"

# Create user-assigned managed identity (reusable)
az identity create \
    --resource-group rg-myapp-prod-eastus \
    --name id-myapp-prod

# Assign to App Service
az webapp identity assign \
    --resource-group rg-myapp-prod-eastus \
    --name myapp-prod-eastus \
    --identities /subscriptions/${SUB_ID}/resourceGroups/rg-myapp-prod-eastus/providers/Microsoft.ManagedIdentity/userAssignedIdentities/id-myapp-prod
```

> **⚠️ Gotcha (SP secret expiry):** Service principal secrets expire after 1 year by default. When they expire, your CI/CD pipelines silently fail with "AADSTS700024: Client assertion is not within its valid time range." Set a calendar reminder to rotate secrets, or better yet, use [federated credentials](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation) for GitHub Actions — no secrets needed.

> **DevOps Relevance:** Identity is the most critical security layer in Azure. Use managed identities everywhere inside Azure (VMs, AKS, App Service) — zero secrets to leak. Service principals are only for external systems (GitHub, Jenkins). Always assign roles at the narrowest scope possible (resource group, not subscription). Use Entra ID groups for role assignments — never assign roles to individual users.

---

## 9. Azure Kubernetes Service (AKS)

### 9.1 AKS Architecture

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  AKS ARCHITECTURE                                                │
  │                                                                  │
  │  ┌──────────────────────────────────┐                           │
  │  │  Control Plane (Azure-managed)   │  ← FREE (Azure manages)  │
  │  │  ├── API Server                  │     No access to VMs     │
  │  │  ├── etcd                        │     Auto-patched          │
  │  │  ├── Scheduler                   │     SLA: 99.95% (paid)   │
  │  │  └── Controller Manager          │     or 99.9% (free)      │
  │  └──────────────┬───────────────────┘                           │
  │                 │                                                │
  │  ┌──────────────▼───────────────────┐                           │
  │  │  Node Pool: system               │  ← You pay for these    │
  │  │  ├── node01 (Standard_D4s_v5)    │     VM nodes             │
  │  │  ├── node02                      │                           │
  │  │  └── node03                      │  Min 1 system pool      │
  │  └──────────────────────────────────┘                           │
  │                                                                  │
  │  ┌──────────────────────────────────┐                           │
  │  │  Node Pool: app (user pool)      │  ← Workload nodes       │
  │  │  ├── node04 (Standard_D8s_v5)    │     Separate from system │
  │  │  ├── node05                      │     Can auto-scale       │
  │  │  └── node06                      │     Taints supported     │
  │  └──────────────────────────────────┘                           │
  │                                                                  │
  │  ┌──────────────────────────────────┐                           │
  │  │  Node Pool: gpu (spot VMs)       │  ← Special workloads    │
  │  │  └── node07 (Standard_NC6s_v3)   │     ML/AI, batch jobs   │
  │  └──────────────────────────────────┘                           │
  └──────────────────────────────────────────────────────────────────┘
```

### 9.2 Creating & Managing AKS Clusters

```bash
# Create an AKS cluster
az aks create \
    --resource-group rg-myapp-prod-eastus \
    --name aks-main-prod-eastus \
    --node-count 3 \
    --node-vm-size Standard_D4s_v5 \
    --zones 1 2 3 \
    --network-plugin azure \
    --network-policy calico \
    --enable-managed-identity \
    --enable-cluster-autoscaler \
    --min-count 3 --max-count 10 \
    --kubernetes-version 1.29 \
    --generate-ssh-keys \
    --tags environment=production

# Get credentials (kubeconfig)
az aks get-credentials \
    --resource-group rg-myapp-prod-eastus \
    --name aks-main-prod-eastus

# Add a user node pool
az aks nodepool add \
    --resource-group rg-myapp-prod-eastus \
    --cluster-name aks-main-prod-eastus \
    --name apppool \
    --node-count 3 \
    --node-vm-size Standard_D8s_v5 \
    --zones 1 2 3 \
    --mode User \
    --labels workload=app \
    --enable-cluster-autoscaler \
    --min-count 2 --max-count 20

# Add a spot VM pool (cost savings for batch jobs)
az aks nodepool add \
    --resource-group rg-myapp-prod-eastus \
    --cluster-name aks-main-prod-eastus \
    --name spotpool \
    --node-count 0 \
    --node-vm-size Standard_D4s_v5 \
    --priority Spot \
    --spot-max-price -1 \
    --eviction-policy Delete \
    --enable-cluster-autoscaler \
    --min-count 0 --max-count 10

# Upgrade Kubernetes version
az aks get-upgrades \
    --resource-group rg-myapp-prod-eastus \
    --name aks-main-prod-eastus \
    --output table

az aks upgrade \
    --resource-group rg-myapp-prod-eastus \
    --name aks-main-prod-eastus \
    --kubernetes-version 1.30 \
    --yes
```

### 9.3 AKS Networking

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  AKS NETWORKING MODES                                            │
  │                                                                  │
  │  kubenet (Basic):                                                │
  │  ├── Pods get IPs from a separate CIDR (not VNet)              │
  │  ├── Uses UDR for pod-to-pod across nodes                      │
  │  ├── Simpler, fewer IPs needed                                  │
  │  └── Can't use: Network Policies, VNet peering to pods        │
  │                                                                  │
  │  Azure CNI (Advanced):                                           │
  │  ├── Every pod gets a VNet IP address                           │
  │  ├── Pods are directly reachable from VNet                     │
  │  ├── Supports Network Policies (Calico/Azure)                   │
  │  ├── Requires more IP addresses (plan subnets carefully!)      │
  │  └── ★ Recommended for production                              │
  │                                                                  │
  │  Azure CNI Overlay (New):                                        │
  │  ├── Pods get overlay IPs (not from VNet CIDR)                 │
  │  ├── Best of both — VNet integration without IP exhaustion     │
  │  └── ★ Best option for new clusters                            │
  │                                                                  │
  │  IP PLANNING for Azure CNI:                                      │
  │  Nodes: 10    × IPs per node: 30  = 300 IPs minimum           │
  │  Use /23 subnet (510 IPs) or larger for room to grow            │
  └──────────────────────────────────────────────────────────────────┘
```

### 9.4 AKS + Workload Identity

```bash
# Enable workload identity on cluster
az aks update \
    --resource-group rg-myapp-prod-eastus \
    --name aks-main-prod-eastus \
    --enable-oidc-issuer \
    --enable-workload-identity

# Create managed identity for the workload
az identity create \
    --name id-myapp-aks \
    --resource-group rg-myapp-prod-eastus

# Create federated credential
OIDC_ISSUER=$(az aks show -n aks-main-prod-eastus -g rg-myapp-prod-eastus \
    --query "oidcIssuerProfile.issuerUrl" -o tsv)

az identity federated-credential create \
    --name fc-myapp \
    --identity-name id-myapp-aks \
    --resource-group rg-myapp-prod-eastus \
    --issuer "$OIDC_ISSUER" \
    --subject "system:serviceaccount:myapp:myapp-sa" \
    --audience "api://AzureADTokenExchange"

# Grant identity access to Key Vault
IDENTITY_CLIENT_ID=$(az identity show -n id-myapp-aks -g rg-myapp-prod-eastus \
    --query clientId -o tsv)

az role assignment create \
    --assignee "$IDENTITY_CLIENT_ID" \
    --role "Key Vault Secrets User" \
    --scope "/subscriptions/${SUB_ID}/resourceGroups/rg-myapp-prod-eastus/providers/Microsoft.KeyVault/vaults/kv-myapp-prod"
```

> **⚠️ Gotcha (AKS upgrades):** AKS only supports N-2 Kubernetes versions. If you skip upgrades, you'll be forced to upgrade through each version sequentially. Test upgrades in a staging cluster first — breaking changes in K8s APIs (e.g., Ingress v1beta1 → v1) can break deployments.

> **DevOps Relevance:** AKS is Azure's managed Kubernetes. You don't manage the control plane (it's free), but you pay for worker nodes. Use multiple node pools (system vs app vs spot) for cost optimization. Workload Identity replaces pod-mounted service principal secrets — it's the secure way to let pods access Azure resources.

---

## 10. Azure DevOps & Pipelines

### 10.1 Azure DevOps Overview

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  AZURE DEVOPS — ALL-IN-ONE PLATFORM                             │
  │                                                                  │
  │  ┌─────────────┐  Git repos with branch policies, PRs          │
  │  │  Azure Repos  │  Alternative: GitHub                         │
  │  └─────────────┘                                                │
  │                                                                  │
  │  ┌─────────────┐  Agile boards (Kanban, Scrum, sprints)        │
  │  │  Azure Boards │  Alternative: Jira, Linear                  │
  │  └─────────────┘                                                │
  │                                                                  │
  │  ┌─────────────┐  CI/CD pipelines (YAML or classic)            │
  │  │  Pipelines    │  Alternative: GitHub Actions, Jenkins       │
  │  └─────────────┘                                                │
  │                                                                  │
  │  ┌─────────────┐  Package feeds (npm, NuGet, pip, Maven)       │
  │  │  Artifacts    │  Alternative: Artifactory, GitHub Packages  │
  │  └─────────────┘                                                │
  │                                                                  │
  │  ┌─────────────┐  Manual + exploratory testing                 │
  │  │  Test Plans   │  (paid add-on)                              │
  │  └─────────────┘                                                │
  │                                                                  │
  │  FREE TIER: 5 users, 1 parallel job (1800 min/mo)             │
  └──────────────────────────────────────────────────────────────────┘
```

### 10.2 YAML Pipelines — Multi-Stage CI/CD

```yaml
# azure-pipelines.yml — Production CI/CD pipeline
trigger:
  branches:
    include: [main, develop]
  paths:
    exclude: ['docs/*', '*.md']

pr:
  branches:
    include: [main]

variables:
  - group: production-secrets          # Variable group (linked to Key Vault)
  - name: imageName
    value: 'myapp'
  - name: acrName
    value: 'acrmyappprod'

stages:
  # --- Stage 1: Build & Test ---
  - stage: Build
    displayName: 'Build & Test'
    jobs:
      - job: BuildAndTest
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: UsePythonVersion@0
            inputs:
              versionSpec: '3.12'

          - script: |
              pip install -r requirements.txt
              pip install -r requirements-test.txt
            displayName: 'Install dependencies'

          - script: |
              pytest tests/ --cov=src --cov-report=xml --junitxml=test-results.xml
            displayName: 'Run tests'

          - task: PublishTestResults@2
            inputs:
              testResultsFiles: 'test-results.xml'

          - task: PublishCodeCoverageResults@1
            inputs:
              codeCoverageTool: Cobertura
              summaryFileLocation: 'coverage.xml'

          - task: Docker@2
            inputs:
              containerRegistry: 'acr-service-connection'
              repository: '$(imageName)'
              command: 'buildAndPush'
              Dockerfile: '**/Dockerfile'
              tags: |
                $(Build.BuildId)
                latest

  # --- Stage 2: Deploy to Staging ---
  - stage: DeployStaging
    displayName: 'Deploy to Staging'
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: DeployStaging
        environment: 'staging'
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureCLI@2
                  inputs:
                    azureSubscription: 'azure-prod-connection'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      az aks get-credentials -g rg-myapp-staging -n aks-staging
                      kubectl set image deployment/myapp \
                        myapp=$(acrName).azurecr.io/$(imageName):$(Build.BuildId) \
                        -n myapp
                      kubectl rollout status deployment/myapp -n myapp --timeout=300s

  # --- Stage 3: Deploy to Production (with approval) ---
  - stage: DeployProduction
    displayName: 'Deploy to Production'
    dependsOn: DeployStaging
    jobs:
      - deployment: DeployProd
        environment: 'production'       # Requires manual approval
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureCLI@2
                  inputs:
                    azureSubscription: 'azure-prod-connection'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      az aks get-credentials -g rg-myapp-prod-eastus -n aks-main-prod-eastus
                      kubectl set image deployment/myapp \
                        myapp=$(acrName).azurecr.io/$(imageName):$(Build.BuildId) \
                        -n myapp
                      kubectl rollout status deployment/myapp -n myapp --timeout=300s
```

### 10.3 Pipeline Variables & Secrets

```yaml
# --- Variable Groups linked to Key Vault ---
# In Azure DevOps UI: Pipelines → Library → Variable Groups
# Link to Azure Key Vault to auto-sync secrets

# Using variables in pipeline:
variables:
  - group: production-secrets  # Contains DB_PASSWORD, API_KEY, etc.
  - name: buildConfiguration
    value: 'Release'

# Access in script:
steps:
  - script: |
      echo "Deploying version $(Build.BuildId)"
      # Secret variables are automatically masked in logs
      ./deploy.sh --db-password $(DB_PASSWORD)
    displayName: 'Deploy'
    env:
      DB_PASSWORD: $(DB_PASSWORD)   # Map to env var for security
```

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  PIPELINE ENVIRONMENTS & APPROVALS                               │
  │                                                                  │
  │  Configure in: Pipelines → Environments → [env] → Approvals    │
  │                                                                  │
  │  staging:                                                        │
  │  └── Auto-deploy on main branch push                           │
  │                                                                  │
  │  production:                                                     │
  │  ├── Requires approval from: devops-leads group                │
  │  ├── Business hours only (optional gate)                       │
  │  └── Timeout: 72 hours                                         │
  │                                                                  │
  │  SERVICE CONNECTIONS:                                             │
  │  ├── Azure Resource Manager → deploy to Azure                  │
  │  ├── Docker Registry → push to ACR/DockerHub                  │
  │  ├── Kubernetes → deploy to AKS cluster                       │
  │  └── SSH → deploy to VMs                                       │
  └──────────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha (parallel jobs):** Free tier Azure DevOps gets **1 parallel job with 1800 free minutes/month**. You can request a free grant for open-source projects. For private projects, each additional parallel job costs ~$40/month. Self-hosted agents are unlimited parallelism but you manage the VMs.

> **DevOps Relevance:** Azure DevOps Pipelines gives you multi-stage CI/CD with built-in approvals, environments, and service connections. YAML pipelines are version-controlled alongside your code. The Key Vault integration for variable groups means secrets are synced from vault to pipeline automatically — no manual copy-paste.

---

## 11. Container Registry & Container Instances

### 11.1 Azure Container Registry (ACR)

```bash
# Create a Premium ACR (needed for geo-replication, private endpoints)
az acr create \
    --name acrmyappprod \
    --resource-group rg-myapp-prod-eastus \
    --sku Premium \
    --admin-enabled false

# Login to ACR
az acr login --name acrmyappprod

# Build image directly in ACR (no local Docker needed!)
az acr build \
    --registry acrmyappprod \
    --image myapp:v2.3.1 \
    --file Dockerfile .

# Push from local Docker
docker tag myapp:v2.3.1 acrmyappprod.azurecr.io/myapp:v2.3.1
docker push acrmyappprod.azurecr.io/myapp:v2.3.1

# List repositories and tags
az acr repository list --name acrmyappprod --output table
az acr repository show-tags --name acrmyappprod --repository myapp --orderby time_desc

# Enable geo-replication (Premium SKU)
az acr replication create \
    --registry acrmyappprod \
    --location westus2

# Clean up old images (keep last 10 tags)
az acr run --registry acrmyappprod --cmd "acr purge \
    --filter 'myapp:.*' \
    --keep 10 \
    --ago 30d \
    --untagged" /dev/null
```

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  ACR SKU COMPARISON                                              │
  │                                                                  │
  │  Basic:     10 GiB storage, 2 webhooks                         │
  │  Standard:  100 GiB, 10 webhooks                               │
  │  Premium:   500 GiB, 500 webhooks, geo-replication,            │
  │             private endpoints, content trust, retention         │
  │                                                                  │
  │  ACR TASKS (CI built into registry):                            │
  │  ├── Quick task:  az acr build (one-time build)                │
  │  ├── Auto task:   Trigger on git commit or base image update   │
  │  └── Multi-step:  YAML-based build + test + push pipeline      │
  │                                                                  │
  │  ATTACH ACR TO AKS:                                              │
  │  az aks update -n aks-main-prod -g rg-myapp-prod \              │
  │    --attach-acr acrmyappprod                                    │
  │  (Grants AKS managed identity AcrPull role)                    │
  └──────────────────────────────────────────────────────────────────┘
```

### 11.2 Azure Container Instances (ACI)

```bash
# Run a container instantly (no cluster needed)
az container create \
    --resource-group rg-myapp-dev-eastus \
    --name ci-migration-runner \
    --image acrmyappprod.azurecr.io/db-migrate:latest \
    --cpu 2 --memory 4 \
    --environment-variables \
        DB_HOST=sql-myapp-dev.database.windows.net \
    --secure-environment-variables \
        DB_PASSWORD="$DB_PASSWORD" \
    --restart-policy Never \
    --registry-login-server acrmyappprod.azurecr.io \
    --registry-username "$ACR_USERNAME" \
    --registry-password "$ACR_PASSWORD"

# Check logs
az container logs \
    --resource-group rg-myapp-dev-eastus \
    --name ci-migration-runner

# Attach to running container (interactive)
az container attach \
    --resource-group rg-myapp-dev-eastus \
    --name ci-migration-runner

# Delete when done
az container delete \
    --resource-group rg-myapp-dev-eastus \
    --name ci-migration-runner --yes
```

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  WHEN TO USE ACI vs AKS vs APP SERVICE                         │
  │                                                                  │
  │  ACI:                                                            │
  │  ├── Quick one-off tasks (DB migrations, batch jobs)           │
  │  ├── CI/CD build agents                                         │
  │  ├── Event-driven burst (scale from AKS via Virtual Kubelet)  │
  │  └── No infrastructure to manage, per-second billing           │
  │                                                                  │
  │  AKS:                                                            │
  │  ├── Microservices (many containers, service mesh)             │
  │  ├── Long-running workloads with auto-scaling                  │
  │  ├── Need network policies, RBAC, persistent storage           │
  │  └── Complex orchestration requirements                        │
  │                                                                  │
  │  App Service for Containers:                                     │
  │  ├── Single container web apps                                  │
  │  ├── Need deployment slots, custom domains, SSL                │
  │  └── Don't want to manage K8s                                  │
  └──────────────────────────────────────────────────────────────────┘
```

> **DevOps Relevance:** ACR is your private Docker registry in Azure — use `az acr build` to build images in the cloud without needing Docker locally. Attach ACR to AKS with one command for seamless image pulls. ACI is perfect for one-off tasks like database migrations, scheduled batch jobs, or CI/CD build agents — spin up, run, destroy.

---

## 12. Azure Functions & Serverless

### 12.1 Azure Functions Overview

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  AZURE FUNCTIONS — EVENT-DRIVEN COMPUTE                         │
  │                                                                  │
  │  Trigger (what starts it):                                       │
  │  ├── HTTP Request  → API endpoints, webhooks                   │
  │  ├── Timer         → Cron-like schedules ("0 */5 * * * *")    │
  │  ├── Blob Storage  → File uploaded/changed                     │
  │  ├── Queue Storage → Message received                          │
  │  ├── Service Bus   → Enterprise messaging                      │
  │  ├── Event Grid    → Azure resource events                     │
  │  ├── Cosmos DB     → Change feed                               │
  │  └── Event Hubs    → Streaming data                            │
  │                                                                  │
  │  Binding (automatic I/O):                                        │
  │  ├── Input:  Read from Blob, Cosmos DB, Table Storage          │
  │  └── Output: Write to Queue, Blob, Cosmos DB, SendGrid        │
  │                                                                  │
  │  HOSTING PLANS:                                                  │
  │  ├── Consumption: Pay per execution, auto-scale 0→200         │
  │  │                 Cold start: 1-10 seconds                     │
  │  ├── Premium:     Always warm, VNet, unlimited execution       │
  │  │                 Min 1 instance pre-warmed                    │
  │  └── Dedicated:   Runs on App Service plan (always on)         │
  │                                                                  │
  │  Languages: C#, JavaScript, Python, Java, PowerShell, Go      │
  └──────────────────────────────────────────────────────────────────┘
```

### 12.2 Function Examples

```python
# function_app.py — Python v2 programming model
import azure.functions as func
import json
import logging

app = func.FunctionApp()

# --- HTTP Trigger ---
@app.route(route="health", methods=["GET"])
def health_check(req: func.HttpRequest) -> func.HttpResponse:
    return func.HttpResponse(json.dumps({"status": "healthy"}),
                             mimetype="application/json")

# --- Timer Trigger (every 5 minutes) ---
@app.timer_trigger(schedule="0 */5 * * * *", arg_name="timer")
def cleanup_stale_sessions(timer: func.TimerRequest) -> None:
    logging.info("Running session cleanup...")
    # Delete sessions older than 24 hours
    delete_old_sessions(hours=24)

# --- Blob Trigger (process uploaded files) ---
@app.blob_trigger(arg_name="blob", path="uploads/{name}",
                  connection="AzureWebJobsStorage")
def process_upload(blob: func.InputStream) -> None:
    logging.info(f"Processing blob: {blob.name}, Size: {blob.length}")
    content = blob.read()
    # Process the uploaded file (resize image, parse CSV, etc.)

# --- Queue Trigger (process messages) ---
@app.queue_trigger(arg_name="msg", queue_name="deploy-queue",
                   connection="AzureWebJobsStorage")
def process_deploy_request(msg: func.QueueMessage) -> None:
    payload = json.loads(msg.get_body().decode())
    logging.info(f"Deploy request: {payload['app']} v{payload['version']}")
    trigger_deployment(payload["app"], payload["version"], payload["env"])
```

```bash
# --- Deploy Functions ---
# Create Function App
az functionapp create \
    --name func-myapp-prod-eastus \
    --resource-group rg-myapp-prod-eastus \
    --storage-account stmyappprodeastus \
    --consumption-plan-location eastus \
    --runtime python \
    --runtime-version 3.12 \
    --functions-version 4 \
    --os-type Linux

# Deploy from local
func azure functionapp publish func-myapp-prod-eastus

# Set app settings
az functionapp config appsettings set \
    --name func-myapp-prod-eastus \
    --resource-group rg-myapp-prod-eastus \
    --settings SLACK_WEBHOOK="https://hooks.slack.com/..." \
               DB_CONNECTION="@Microsoft.KeyVault(SecretUri=https://kv-myapp.vault.azure.net/secrets/db-conn)"
```

### 12.3 Serverless Decision Guide

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  AZURE SERVERLESS — WHAT TO USE WHEN                            │
  │                                                                  │
  │  Azure Functions:                                                │
  │  ├── Code-first, event-driven microservices                    │
  │  ├── Best for: APIs, webhooks, file processing, schedules     │
  │  └── You write: function code + bindings config               │
  │                                                                  │
  │  Logic Apps:                                                     │
  │  ├── Visual workflow designer (low-code)                       │
  │  ├── 400+ connectors (O365, Salesforce, SAP, etc.)            │
  │  ├── Best for: integration workflows, approvals, automation   │
  │  └── You build: drag-and-drop workflow in Portal              │
  │                                                                  │
  │  Event Grid:                                                     │
  │  ├── Event routing service (pub/sub)                           │
  │  ├── Routes events from Azure resources to handlers            │
  │  ├── Best for: reactive architectures                          │
  │  └── Example: blob created → trigger Function → notify Slack  │
  │                                                                  │
  │  COMMON DEVOPS PATTERN:                                          │
  │  Event Grid (source) → Function (process) → Logic App (notify)│
  └──────────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha (cold start):** Consumption plan Functions have cold start delays (1-10 seconds on first invocation after idle). For APIs that need consistent response times, use Premium plan with a minimum of 1 pre-warmed instance. Timer-triggered functions on Consumption plan may also experience up to 10-minute delays.

> **DevOps Relevance:** Functions are your automation glue in Azure. Use them for: alert-triggered auto-remediation, Slack notifications on deployment events, scheduled cleanup tasks, webhook processing from GitHub/Azure DevOps. The Key Vault reference syntax (`@Microsoft.KeyVault(...)`) in app settings means secrets are fetched at runtime — never stored in config.

---

## 13. Monitoring & Observability

### 13.1 Azure Monitor Architecture

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  AZURE MONITOR — THE OBSERVABILITY PLATFORM                     │
  │                                                                  │
  │  Data Sources:                                                   │
  │  ├── Application (App Insights SDK)                            │
  │  ├── OS / Guest (VM agent, diagnostics)                        │
  │  ├── Azure Resources (platform metrics & logs)                  │
  │  ├── Azure Subscriptions (Activity Log)                         │
  │  └── Azure AD (Entra ID sign-in/audit logs)                    │
  │                                                                  │
  │           ┌────────────────────────────┐                        │
  │  ────────►│  Azure Monitor             │                        │
  │           │  ┌──────────┐ ┌──────────┐│                        │
  │           │  │ Metrics   │ │   Logs   ││                        │
  │           │  │ (numeric, │ │ (text,   ││                        │
  │           │  │ real-time)│ │ KQL query││                        │
  │           │  └────┬─────┘ └────┬─────┘│                        │
  │           └───────┼────────────┼──────┘                        │
  │                   │            │                                 │
  │          ┌────────┼────────────┼────────┐                       │
  │          ▼        ▼            ▼        ▼                       │
  │  ┌──────────┐ ┌──────┐ ┌──────────┐ ┌──────────┐             │
  │  │  Alerts   │ │Dashb.│ │Workbooks │ │ Auto-    │             │
  │  │(action   │ │      │ │(reports) │ │ scale    │             │
  │  │ groups)  │ │      │ │          │ │          │             │
  │  └──────────┘ └──────┘ └──────────┘ └──────────┘             │
  └──────────────────────────────────────────────────────────────────┘
```

### 13.2 Log Analytics & KQL

```bash
# Create a Log Analytics workspace
az monitor log-analytics workspace create \
    --resource-group rg-myapp-prod-eastus \
    --workspace-name log-main-prod-eastus \
    --location eastus \
    --retention-time 90
```

```kusto
// --- KQL (Kusto Query Language) examples ---

// Find errors in container logs (AKS)
ContainerLogV2
| where TimeGenerated > ago(1h)
| where LogLevel == "error"
| project TimeGenerated, PodName, LogMessage
| order by TimeGenerated desc
| take 50

// CPU usage by VM (last 4 hours)
Perf
| where TimeGenerated > ago(4h)
| where ObjectName == "Processor" and CounterName == "% Processor Time"
| summarize AvgCPU = avg(CounterValue) by Computer, bin(TimeGenerated, 5m)
| render timechart

// Failed HTTP requests in App Insights
requests
| where timestamp > ago(24h)
| where success == false
| summarize FailedCount = count() by bin(timestamp, 1h), resultCode
| render columnchart

// Resource health changes
AzureActivity
| where TimeGenerated > ago(7d)
| where OperationNameValue contains "delete" or OperationNameValue contains "write"
| project TimeGenerated, Caller, OperationNameValue, ResourceGroup, _ResourceId
| order by TimeGenerated desc

// Slow queries (App Insights dependencies)
dependencies
| where timestamp > ago(1h)
| where duration > 1000  // ms
| project timestamp, name, type, duration, success
| order by duration desc
```

### 13.3 Application Insights

```bash
# Create Application Insights
az monitor app-insights component create \
    --app appi-myapp-prod-eastus \
    --location eastus \
    --resource-group rg-myapp-prod-eastus \
    --workspace log-main-prod-eastus \
    --application-type web

# Get the connection string (for your app)
az monitor app-insights component show \
    --app appi-myapp-prod-eastus \
    --resource-group rg-myapp-prod-eastus \
    --query connectionString -o tsv
```

```python
# Python — Add App Insights telemetry
# pip install azure-monitor-opentelemetry
from azure.monitor.opentelemetry import configure_azure_monitor

configure_azure_monitor(
    connection_string="InstrumentationKey=xxx;IngestionEndpoint=https://..."
)

# Now all requests, dependencies, exceptions are auto-tracked
# Custom telemetry:
from opentelemetry import trace
tracer = trace.get_tracer(__name__)

with tracer.start_as_current_span("process_order") as span:
    span.set_attribute("order.id", order_id)
    span.set_attribute("order.amount", amount)
    # Your business logic here
```

### 13.4 Alerts & Action Groups

```bash
# Create an action group (who gets notified)
az monitor action-group create \
    --resource-group rg-myapp-prod-eastus \
    --name ag-devops-critical \
    --short-name DevOpsCrit \
    --email-receiver name=lead email=devops-lead@company.com \
    --webhook-receiver name=slack uri="https://hooks.slack.com/services/xxx"

# Create a metric alert (CPU > 80%)
az monitor metrics alert create \
    --resource-group rg-myapp-prod-eastus \
    --name "alert-high-cpu" \
    --scopes "/subscriptions/${SUB_ID}/resourceGroups/rg-myapp-prod-eastus/providers/Microsoft.Compute/virtualMachines/vm-web01" \
    --condition "avg Percentage CPU > 80" \
    --window-size 5m \
    --evaluation-frequency 1m \
    --action ag-devops-critical \
    --severity 2

# Create a log-based alert (error spike)
az monitor scheduled-query create \
    --resource-group rg-myapp-prod-eastus \
    --name "alert-error-spike" \
    --scopes "/subscriptions/${SUB_ID}/resourceGroups/rg-myapp-prod-eastus/providers/Microsoft.Insights/components/appi-myapp-prod-eastus" \
    --condition "count > 50" \
    --condition-query "requests | where success == false | summarize count() by bin(timestamp, 5m)" \
    --evaluation-frequency 5m \
    --window-size 5m \
    --action-groups ag-devops-critical \
    --severity 1
```

> **⚠️ Gotcha (Log Analytics cost):** Log Analytics charges per GB ingested. A busy AKS cluster can generate 10+ GB/day of container logs. Use diagnostic settings to filter what you ingest, and set daily caps to avoid bill shock. Archive old logs to Storage instead of keeping them in Log Analytics.

> **DevOps Relevance:** Azure Monitor + Log Analytics + App Insights is the full observability stack. Learn KQL — it's the single query language for all Azure logs. Set up alerts with action groups for critical metrics (CPU, errors, latency). Use workbooks for custom dashboards shared across teams.

---

## 14. IaC — ARM Templates, Bicep & Terraform

### 14.1 IaC Options Comparison

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  AZURE IaC — WHAT TO USE                                        │
  │                                                                  │
  │  ARM Templates (JSON):                                           │
  │  ├── Native Azure format (every resource supports it)          │
  │  ├── Verbose JSON (hard to read/write)                         │
  │  ├── No state file (declarative via Azure deployment)          │
  │  └── Use when: exported from Portal, legacy projects           │
  │                                                                  │
  │  Bicep (★ Microsoft recommended):                              │
  │  ├── Transpiles to ARM JSON (1:1 mapping)                     │
  │  ├── Clean, readable DSL syntax                                │
  │  ├── Modules for reusability                                    │
  │  ├── No state file (uses Azure deployment stack)               │
  │  └── Use when: Azure-only shops, new projects                  │
  │                                                                  │
  │  Terraform (azurerm provider):                                   │
  │  ├── Multi-cloud (Azure + AWS + GCP + others)                 │
  │  ├── HCL syntax, state file required                           │
  │  ├── Largest community, most modules                           │
  │  ├── State must be stored somewhere (Azure Storage backend)    │
  │  └── Use when: multi-cloud, existing Terraform teams          │
  │                                                                  │
  │  Pulumi:                                                         │
  │  ├── Real programming languages (Python, Go, TS)              │
  │  └── Use when: developers who hate HCL/YAML                   │
  └──────────────────────────────────────────────────────────────────┘
```

### 14.2 Bicep Deep Dive

```bicep
// main.bicep — Deploy a web app with SQL database

// Parameters
@description('Environment name')
@allowed(['dev', 'staging', 'prod'])
param environment string

@description('Azure region')
param location string = resourceGroup().location

@secure()
@description('SQL admin password')
param sqlAdminPassword string

// Variables
var appName = 'myapp-${environment}-${location}'
var sqlServerName = 'sql-${appName}'
var tags = {
  environment: environment
  managedBy: 'bicep'
  team: 'devops'
}

// --- App Service Plan ---
resource appServicePlan 'Microsoft.Web/serverfarms@2023-01-01' = {
  name: 'asp-${appName}'
  location: location
  tags: tags
  sku: {
    name: environment == 'prod' ? 'P1v3' : 'B1'
  }
  kind: 'linux'
  properties: {
    reserved: true  // Required for Linux
  }
}

// --- Web App ---
resource webApp 'Microsoft.Web/sites@2023-01-01' = {
  name: 'app-${appName}'
  location: location
  tags: tags
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      linuxFxVersion: 'PYTHON|3.12'
      alwaysOn: environment == 'prod'
      appSettings: [
        { name: 'ENVIRONMENT', value: environment }
        { name: 'DB_CONNECTION', value: '@Microsoft.KeyVault(SecretUri=${keyVault::dbConnSecret.properties.secretUri})' }
      ]
    }
  }
  identity: {
    type: 'SystemAssigned'
  }
}

// --- SQL Server + Database ---
resource sqlServer 'Microsoft.Sql/servers@2023-05-01-preview' = {
  name: sqlServerName
  location: location
  tags: tags
  properties: {
    administratorLogin: 'sqladmin'
    administratorLoginPassword: sqlAdminPassword
    minimalTlsVersion: '1.2'
  }
}

resource sqlDb 'Microsoft.Sql/servers/databases@2023-05-01-preview' = {
  parent: sqlServer
  name: 'db-${appName}'
  location: location
  sku: {
    name: environment == 'prod' ? 'S2' : 'Basic'
  }
}

// --- Outputs ---
output webAppUrl string = 'https://${webApp.properties.defaultHostName}'
output webAppIdentityId string = webApp.identity.principalId
```

```bash
# Deploy Bicep
az deployment group create \
    --resource-group rg-myapp-prod-eastus \
    --template-file main.bicep \
    --parameters environment=prod sqlAdminPassword="$SQL_PASSWORD"

# Preview changes (what-if)
az deployment group what-if \
    --resource-group rg-myapp-prod-eastus \
    --template-file main.bicep \
    --parameters environment=prod sqlAdminPassword="$SQL_PASSWORD"

# Decompile ARM → Bicep
az bicep decompile --file template.json
```

### 14.3 Terraform on Azure

```hcl
# providers.tf
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.80"
    }
  }
  # Store state in Azure Storage
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "stterraformstate"
    container_name       = "tfstate"
    key                  = "myapp.prod.tfstate"
  }
}

provider "azurerm" {
  features {}
  subscription_id = var.subscription_id
}

# main.tf
resource "azurerm_resource_group" "main" {
  name     = "rg-myapp-${var.environment}-eastus"
  location = "East US"
  tags     = local.common_tags
}

resource "azurerm_service_plan" "main" {
  name                = "asp-myapp-${var.environment}"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  os_type             = "Linux"
  sku_name            = var.environment == "prod" ? "P1v3" : "B1"
}

resource "azurerm_linux_web_app" "main" {
  name                = "app-myapp-${var.environment}"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  service_plan_id     = azurerm_service_plan.main.id

  site_config {
    application_stack {
      python_version = "3.12"
    }
    always_on = var.environment == "prod"
  }

  identity {
    type = "SystemAssigned"
  }
}
```

```bash
# Terraform workflow
terraform init          # Initialize providers + backend
terraform plan -out=tfplan  # Preview changes
terraform apply tfplan      # Apply changes
terraform destroy           # Tear down everything

# Terraform state in Azure Storage
az storage account create --name stterraformstate --resource-group rg-terraform-state \
    --sku Standard_LRS --encryption-services blob
az storage container create --name tfstate --account-name stterraformstate
```

> **⚠️ Gotcha (Terraform state locking):** Azure Storage backend supports state locking via blob leases. If a `terraform apply` crashes mid-run, the lock can get stuck. Fix with: `az storage blob lease break --blob-name myapp.prod.tfstate --container-name tfstate --account-name stterraformstate`.

> **DevOps Relevance:** Pick Bicep for Azure-only, Terraform for multi-cloud. Both should be in CI/CD pipelines with plan → approve → apply workflow. Always use remote state (Azure Storage for Terraform, deployment stacks for Bicep). Never run `terraform apply` locally in production — always through pipelines.

---

## 15. Advanced Networking

### 15.1 Azure DNS

```bash
# Create a DNS zone
az network dns zone create \
    --resource-group rg-networking-prod \
    --name myapp.com

# Add an A record
az network dns record-set a add-record \
    --resource-group rg-networking-prod \
    --zone-name myapp.com \
    --record-set-name www \
    --ipv4-address 20.50.100.200

# Add a CNAME
az network dns record-set cname set-record \
    --resource-group rg-networking-prod \
    --zone-name myapp.com \
    --record-set-name api \
    --cname myapp-prod.azurewebsites.net

# Private DNS (for VNet-internal resolution)
az network private-dns zone create \
    --resource-group rg-networking-prod \
    --name privatelink.database.windows.net

# Link private DNS zone to VNet
az network private-dns link vnet create \
    --resource-group rg-networking-prod \
    --zone-name privatelink.database.windows.net \
    --name link-prod-vnet \
    --virtual-network vnet-main-prod-eastus \
    --registration-enabled false
```

### 15.2 Azure Front Door & CDN

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  AZURE FRONT DOOR — GLOBAL L7 LOAD BALANCER + CDN + WAF       │
  │                                                                  │
  │  Internet Users                                                  │
  │       │                                                          │
  │  ┌────▼──────────────────────┐                                  │
  │  │  Azure Front Door          │                                  │
  │  │  ├── Global anycast edge   │  ← 180+ edge locations         │
  │  │  ├── SSL termination       │                                  │
  │  │  ├── WAF policy            │  ← Block SQLi, XSS, bots      │
  │  │  ├── Caching (CDN)         │                                  │
  │  │  ├── URL routing           │                                  │
  │  │  └── Health probes         │                                  │
  │  └────┬─────────────┬────────┘                                  │
  │       │             │                                            │
  │  ┌────▼────┐   ┌────▼────┐                                     │
  │  │ East US  │   │West US  │                                     │
  │  │ (primary)│   │(failovr)│                                     │
  │  │ App Svc  │   │ App Svc │                                     │
  │  └─────────┘   └─────────┘                                     │
  │                                                                  │
  │  USE CASES:                                                      │
  │  ├── Global web app acceleration                               │
  │  ├── Multi-region failover (< 30 sec)                          │
  │  ├── WAF protection at the edge                                │
  │  └── Static content caching                                    │
  └──────────────────────────────────────────────────────────────────┘
```

### 15.3 Private Link & Private Endpoints

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  PRIVATE LINK — KEEP TRAFFIC ON AZURE BACKBONE                  │
  │                                                                  │
  │  WITHOUT Private Link:                                           │
  │  App (VNet) ──► Internet ──► sql-myapp.database.windows.net    │
  │                  (public)                                        │
  │                                                                  │
  │  WITH Private Link:                                              │
  │  App (VNet) ──► Private Endpoint (10.0.3.10) ──► SQL Database  │
  │                  (stays on Azure backbone)                       │
  │                                                                  │
  │  Supported services:                                             │
  │  ├── Azure SQL Database                                         │
  │  ├── Azure Storage (Blob, Files, Queue, Table)                 │
  │  ├── Azure Cosmos DB                                            │
  │  ├── Azure Key Vault                                            │
  │  ├── Azure Container Registry                                  │
  │  ├── Azure App Service                                          │
  │  └── 50+ more PaaS services                                   │
  └──────────────────────────────────────────────────────────────────┘
```

```bash
# Create a private endpoint for SQL Database
az network private-endpoint create \
    --name pe-sql-myapp-prod \
    --resource-group rg-myapp-prod-eastus \
    --vnet-name vnet-main-prod-eastus \
    --subnet snet-db \
    --private-connection-resource-id "/subscriptions/${SUB_ID}/resourceGroups/rg-myapp-prod-eastus/providers/Microsoft.Sql/servers/sql-myapp-prod" \
    --group-id sqlServer \
    --connection-name pe-conn-sql

# Disable public access on SQL Server
az sql server update \
    --name sql-myapp-prod \
    --resource-group rg-myapp-prod-eastus \
    --enable-public-network false
```

### 15.4 VPN Gateway & ExpressRoute

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  HYBRID CONNECTIVITY                                             │
  │                                                                  │
  │  VPN Gateway (Site-to-Site):                                     │
  │  ├── IPsec/IKE tunnel over public internet                    │
  │  ├── Up to 10 Gbps (VpnGw5)                                   │
  │  ├── Cost: ~$140-700/mo depending on SKU                      │
  │  └── Use for: dev/test, small offices, backup connectivity    │
  │                                                                  │
  │  ExpressRoute:                                                   │
  │  ├── Dedicated private connection (via ISP partner)            │
  │  ├── 50 Mbps to 100 Gbps                                      │
  │  ├── Low latency, high reliability, SLA-backed                │
  │  ├── Cost: $55-15,000/mo + ISP circuit costs                  │
  │  └── Use for: production workloads, compliance, large data    │
  │                                                                  │
  │  On-Prem ←─── VPN / ExpressRoute ───► Azure VNet              │
  │                                                                  │
  │  RECOMMENDATION:                                                 │
  │  Primary: ExpressRoute   Backup: Site-to-Site VPN              │
  └──────────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha (Private Endpoints + DNS):** After creating a private endpoint, you must configure DNS resolution so `sql-myapp.database.windows.net` resolves to the private IP (10.0.3.10), not the public IP. Use Azure Private DNS zones linked to your VNet. Forgetting this step is the #1 cause of "can't connect to database after enabling private endpoint."

> **DevOps Relevance:** In production, ALL PaaS services (SQL, Storage, Key Vault, ACR) should use Private Endpoints. This eliminates public internet exposure completely. Front Door gives you global HA with edge WAF protection. Plan your DNS strategy early — migrating from public to private endpoints requires careful DNS cutover to avoid downtime.

---

## 16. Security & Governance

### 16.1 Azure Key Vault

```bash
# Create a Key Vault
az keyvault create \
    --name kv-myapp-prod-eastus \
    --resource-group rg-myapp-prod-eastus \
    --location eastus \
    --enable-rbac-authorization \
    --sku standard

# Store secrets
az keyvault secret set \
    --vault-name kv-myapp-prod-eastus \
    --name "db-connection-string" \
    --value "Server=sql-myapp.database.windows.net;Database=mydb;..."

az keyvault secret set \
    --vault-name kv-myapp-prod-eastus \
    --name "api-key" \
    --value "sk-prod-xxxxxxxxxxxx" \
    --expires "2025-12-31T23:59:59Z"

# Retrieve a secret
az keyvault secret show \
    --vault-name kv-myapp-prod-eastus \
    --name "db-connection-string" \
    --query value -o tsv

# List secrets
az keyvault secret list \
    --vault-name kv-myapp-prod-eastus \
    --output table

# Import a certificate (SSL/TLS)
az keyvault certificate import \
    --vault-name kv-myapp-prod-eastus \
    --name "ssl-cert-myapp" \
    --file ./cert.pfx \
    --password "$CERT_PASSWORD"

# Enable soft-delete recovery (default: 90 days)
# Secrets can be recovered after accidental deletion
az keyvault secret recover \
    --vault-name kv-myapp-prod-eastus \
    --name "accidentally-deleted-secret"
```

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  KEY VAULT — ACCESS PATTERNS                                     │
  │                                                                  │
  │  From App Service / Functions:                                   │
  │  └── @Microsoft.KeyVault(SecretUri=https://kv.vault.../secret) │
  │      → Automatic secret injection via managed identity          │
  │                                                                  │
  │  From AKS:                                                       │
  │  └── CSI Secrets Store Driver + Workload Identity              │
  │      → Mount secrets as files in pods                           │
  │                                                                  │
  │  From Terraform:                                                 │
  │  └── data "azurerm_key_vault_secret" "db" { ... }             │
  │      → Read at plan/apply time                                  │
  │                                                                  │
  │  From Azure DevOps:                                              │
  │  └── Variable Group linked to Key Vault                        │
  │      → Auto-sync secrets to pipeline variables                 │
  │                                                                  │
  │  NEVER:                                                          │
  │  ├── Hardcode secrets in code, config files, or env vars      │
  │  ├── Store secrets in Blob Storage or Git                      │
  │  └── Share Key Vault across environments (dev/prod separate)  │
  └──────────────────────────────────────────────────────────────────┘
```

### 16.2 Defender for Cloud

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  MICROSOFT DEFENDER FOR CLOUD                                    │
  │                                                                  │
  │  Free tier (CSPM — Cloud Security Posture Management):         │
  │  ├── Secure Score (0-100 security rating)                      │
  │  ├── Security recommendations                                  │
  │  ├── Azure Policy compliance view                              │
  │  └── Asset inventory                                            │
  │                                                                  │
  │  Paid plans (CWPP — Cloud Workload Protection):                │
  │  ├── Defender for Servers: VM threat detection, VA scanning    │
  │  ├── Defender for Containers: AKS runtime protection          │
  │  ├── Defender for SQL: DB threat detection, VA                 │
  │  ├── Defender for Storage: malware upload detection            │
  │  ├── Defender for Key Vault: anomalous access patterns        │
  │  └── Defender for App Service: web attack detection            │
  │                                                                  │
  │  SECURE SCORE ACTIONS:                                           │
  │  ├── Enable MFA on all admin accounts (+10 pts)               │
  │  ├── Enable encryption at rest (+5 pts)                        │
  │  ├── Close management ports (SSH/RDP) (+8 pts)                │
  │  ├── Enable DDoS protection (+5 pts)                           │
  │  └── Enable diagnostic logging (+3 pts)                        │
  └──────────────────────────────────────────────────────────────────┘
```

### 16.3 Azure Policy & Governance

```bash
# List built-in policies
az policy definition list --query "[?policyType=='BuiltIn'].{Name:displayName}" --output table | head -20

# Assign a policy: Require tags on resource groups
az policy assignment create \
    --name "require-env-tag" \
    --display-name "Require environment tag on RGs" \
    --policy "/providers/Microsoft.Authorization/policyDefinitions/96670d01-0a4d-4649-9c89-2d3abc0a5025" \
    --scope "/subscriptions/${SUB_ID}" \
    --params '{"tagName": {"value": "environment"}}'

# Check compliance
az policy state list \
    --resource-group rg-myapp-prod-eastus \
    --query "[?complianceState=='NonCompliant'].{Resource:resourceId, Policy:policyDefinitionName}" \
    --output table
```

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  AZURE POLICY — COMMON DEVOPS RULES                             │
  │                                                                  │
  │  Enforce:                                                        │
  │  ├── Allowed VM sizes (prevent expensive VMs)                  │
  │  ├── Allowed regions (data residency compliance)               │
  │  ├── Required tags (environment, team, cost-center)            │
  │  ├── HTTPS only on App Service                                 │
  │  ├── Encryption at rest on storage accounts                    │
  │  └── No public IP on VMs                                       │
  │                                                                  │
  │  Policy Effects:                                                 │
  │  ├── Deny:       Block non-compliant resources               │
  │  ├── Audit:      Flag but don't block (start here)           │
  │  ├── DeployIfNotExists: Auto-remediate                       │
  │  ├── Modify:     Auto-add tags or settings                   │
  │  └── Disabled:   Turn off the policy                          │
  │                                                                  │
  │  GOVERNANCE HIERARCHY:                                           │
  │  Management Group (org-wide) → Subscription → Resource Group   │
  │  Policies inherit DOWN, exemptions can be applied at any level │
  └──────────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha (Policy deny vs audit):** Start new policies as `Audit` first to see what would be affected. Switching directly to `Deny` can break CI/CD pipelines that create non-compliant resources. Once you've fixed existing resources, switch to `Deny` to prevent future violations.

> **DevOps Relevance:** Key Vault is your single source of truth for secrets — every app, pipeline, and script should read from it. Defender for Cloud gives you a security score to track over time. Azure Policy is your automated compliance guardrail — it prevents misconfigurations before they happen. Start with audit, fix, then enforce.

---

## 17. Cost Management & FinOps

### 17.1 Azure Cost Management Overview

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  AZURE COST OPTIMIZATION — THE FINOPS FRAMEWORK                 │
  │                                                                  │
  │  1. INFORM (Visibility):                                         │
  │  ├── Cost Analysis: break down by resource, tag, service       │
  │  ├── Budgets: set monthly limits with alerts                   │
  │  └── Cost Allocation: tag resources for chargeback             │
  │                                                                  │
  │  2. OPTIMIZE (Savings):                                          │
  │  ├── Reserved Instances: 1yr = 30-40% off, 3yr = 60% off     │
  │  ├── Spot VMs: up to 90% off (evictable)                       │
  │  ├── Right-sizing: downsize over-provisioned VMs               │
  │  ├── Auto-shutdown: dev/test VMs off at night                  │
  │  └── Azure Advisor: automated recommendations                 │
  │                                                                  │
  │  3. GOVERN (Controls):                                           │
  │  ├── Policies: restrict expensive VM sizes                     │
  │  ├── Budgets + action groups: auto-alert on overspend          │
  │  └── Management groups: enforce across subscriptions           │
  │                                                                  │
  │  TYPICAL ENTERPRISE SAVINGS:                                     │
  │  ├── Reserved Instances: 30-60%                                │
  │  ├── Right-sizing VMs: 20-40%                                  │
  │  ├── Spot VMs for batch: 60-90%                                │
  │  ├── Dev auto-shutdown: 65% (16hr/day off)                    │
  │  └── Storage tiering: 30-50%                                   │
  └──────────────────────────────────────────────────────────────────┘
```

### 17.2 Cost Analysis & Budgets

```bash
# View costs for current month
az consumption usage list \
    --start-date $(date -d "first day of this month" +%Y-%m-%d) \
    --end-date $(date +%Y-%m-%d) \
    --query "[].{Resource:instanceName, Cost:pretaxCost, Currency:currency}" \
    --output table

# Create a budget with alerts
az consumption budget create \
    --budget-name "prod-monthly-budget" \
    --amount 10000 \
    --category Cost \
    --time-grain Monthly \
    --start-date 2024-01-01 \
    --end-date 2025-12-31 \
    --resource-group rg-myapp-prod-eastus

# Azure Advisor recommendations (cost savings)
az advisor recommendation list \
    --category Cost \
    --output table
```

### 17.3 Reserved Instances & Spot VMs

```bash
# --- Reserved Instances ---
# Purchase via Portal: Cost Management → Reservations
# Or via CLI:
az reservations reservation-order purchase \
    --sku Standard_D4s_v5 \
    --location eastus \
    --term P1Y \
    --billing-plan Monthly \
    --quantity 5 \
    --applied-scope-type Single \
    --applied-scope "/subscriptions/${SUB_ID}"

# --- Spot VMs (for batch/dev workloads) ---
az vm create \
    --resource-group rg-batch-prod-eastus \
    --name vm-batch-worker01 \
    --image Ubuntu2204 \
    --size Standard_D4s_v5 \
    --priority Spot \
    --max-price 0.08 \
    --eviction-policy Deallocate

# --- Auto-shutdown (dev/test VMs) ---
az vm auto-shutdown \
    --resource-group rg-myapp-dev-eastus \
    --name vm-dev01 \
    --time 1900 \
    --email devops@company.com
```

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  COST OPTIMIZATION CHEATSHEET                                    │
  │                                                                  │
  │  Quick Wins:                                                     │
  │  ├── Tag everything (environment, team, cost-center)           │
  │  ├── Auto-shutdown dev VMs at 7 PM                             │
  │  ├── Use B-series burstable for dev/test                       │
  │  ├── Delete unattached disks and unused public IPs             │
  │  └── Use Standard LRS for non-production storage               │
  │                                                                  │
  │  Medium Effort:                                                  │
  │  ├── Right-size VMs (check Azure Advisor weekly)               │
  │  ├── Enable storage lifecycle management                       │
  │  ├── Use Spot VMs for batch jobs and CI/CD agents             │
  │  └── Switch to Azure Container Apps for simple workloads       │
  │                                                                  │
  │  Strategic (high impact):                                        │
  │  ├── Buy Reserved Instances for stable workloads              │
  │  ├── Savings Plans for flexible compute                       │
  │  ├── Move infrequent storage to Cool/Archive tiers            │
  │  └── Consolidate Log Analytics workspaces                      │
  └──────────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha (orphaned resources):** Azure doesn't auto-delete disks, public IPs, or NICs when you delete a VM. These "orphaned resources" silently accumulate costs. Run `az disk list --query "[?managedBy==null].{Name:name,Size:diskSizeGb}" -o table` regularly to find them.

> **DevOps Relevance:** FinOps is a DevOps responsibility. Tag every resource in your IaC templates (Bicep/Terraform) with `environment`, `team`, and `cost-center`. Set up budget alerts before you deploy to production. Use Azure Advisor in your weekly ops review — it finds savings automatically.

---

## 18. HA & Disaster Recovery

### 18.1 Azure Availability Concepts

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  AVAILABILITY — FROM LOWEST TO HIGHEST                          │
  │                                                                  │
  │  Level 1: Single VM                                              │
  │  └── SLA: 99.9% (with Premium SSD)                             │
  │      ~8.76 hours downtime/year                                  │
  │                                                                  │
  │  Level 2: Availability Set (same datacenter)                    │
  │  ├── VMs spread across Fault Domains (racks)                   │
  │  ├── VMs spread across Update Domains (patches)                │
  │  └── SLA: 99.95%                                                │
  │      ~4.38 hours downtime/year                                  │
  │                                                                  │
  │  Level 3: Availability Zones (different datacenters)            │
  │  ├── 3 physically separate datacenters per region              │
  │  ├── Independent power, cooling, networking                    │
  │  └── SLA: 99.99%                                                │
  │      ~52 minutes downtime/year                                  │
  │                                                                  │
  │  Level 4: Multi-Region (different geographies)                  │
  │  ├── Active-Passive or Active-Active                           │
  │  ├── Azure Front Door / Traffic Manager for failover           │
  │  └── SLA: 99.999% (with proper setup)                          │
  │      ~5.26 minutes downtime/year                                │
  │                                                                  │
  │  FOR PRODUCTION: Always use Availability Zones (minimum)       │
  └──────────────────────────────────────────────────────────────────┘
```

### 18.2 DR Architecture Patterns

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  DR PATTERNS — RPO/RTO MATRIX                                   │
  │                                                                  │
  │  Pattern        │ RPO     │ RTO     │ Cost    │ Use Case       │
  │  ───────────────┼─────────┼─────────┼─────────┼───────────────│
  │  Backup/Restore │ Hours   │ Hours   │ $       │ Non-critical   │
  │  Pilot Light    │ Minutes │ Minutes │ $$      │ Important apps │
  │  Warm Standby   │ Seconds │ Minutes │ $$$     │ Critical apps  │
  │  Active-Active  │ Zero    │ Seconds │ $$$$    │ Mission-critical│
  │                                                                  │
  │  ACTIVE-PASSIVE (warm standby):                                 │
  │  ┌──────────┐           ┌──────────┐                           │
  │  │ East US   │           │ West US   │                          │
  │  │ (active)  │───sync──►│ (standby) │                          │
  │  │ Full stack│           │ Reduced   │                          │
  │  └─────┬────┘           └─────┬────┘                           │
  │        │                       │                                 │
  │  ┌─────▼───────────────────────▼─────┐                          │
  │  │       Azure Front Door            │                          │
  │  │   (auto-failover on health check) │                          │
  │  └──────────────────────────────────┘                           │
  │                                                                  │
  │  AZURE PAIRED REGIONS:                                           │
  │  ├── East US ← → West US                                       │
  │  ├── North Europe ← → West Europe                              │
  │  ├── Southeast Asia ← → East Asia                              │
  │  └── Platform updates roll to pair B after pair A              │
  └──────────────────────────────────────────────────────────────────┘
```

### 18.3 Azure Site Recovery (ASR) & Data Replication

```bash
# --- Azure Site Recovery (VM replication) ---
# Replicate VM from East US to West US
az site-recovery protected-item create \
    --resource-group rg-dr-prod \
    --vault-name rsv-dr-prod \
    --fabric-name eastus-fabric \
    --protection-container eastus-container \
    --name vm-web01-prod \
    --policy-id /subscriptions/${SUB_ID}/resourceGroups/rg-dr-prod/providers/Microsoft.RecoveryServices/vaults/rsv-dr-prod/replicationPolicies/24-hour-retention

# Test failover (non-disruptive)
az site-recovery recovery-plan execute \
    --resource-group rg-dr-prod \
    --vault-name rsv-dr-prod \
    --name recovery-plan-myapp \
    --failover-direction PrimaryToRecovery \
    --network-type VmNetworkAsInput

# --- Database Geo-Replication ---
# SQL Database auto-failover group
az sql failover-group create \
    --name fog-myapp-prod \
    --resource-group rg-myapp-prod-eastus \
    --server sql-myapp-prod-eastus \
    --partner-server sql-myapp-prod-westus \
    --partner-resource-group rg-myapp-prod-westus \
    --failover-policy Automatic \
    --grace-period 1

# Manual failover (for testing or planned maintenance)
az sql failover-group set-primary \
    --name fog-myapp-prod \
    --resource-group rg-myapp-prod-westus \
    --server sql-myapp-prod-westus

# --- Storage Geo-Redundancy ---
# Already configured at creation time:
# --sku Standard_GRS (geo-redundant)
# --sku Standard_GZRS (geo-zone-redundant)

# Initiate storage account failover
az storage account failover \
    --name stmyappprodeastus \
    --resource-group rg-myapp-prod-eastus
```

> **⚠️ Gotcha (storage failover):** Storage account failover is a destructive operation — the secondary becomes primary, and the original primary is lost. GRS replication is asynchronous, so you may lose the most recent writes. Design your apps to handle this (use queues, idempotent writes).

> **DevOps Relevance:** DR is not optional for production. At minimum, use Availability Zones (99.99% SLA). For critical apps, set up cross-region replication with SQL auto-failover groups and Azure Site Recovery. Test your DR plan quarterly — an untested DR plan is not a plan, it's a wish.

---

## 19. Migration Strategies

### 19.1 The 6 Rs of Cloud Migration

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  MIGRATION STRATEGIES (6 Rs)                                     │
  │                                                                  │
  │  1. Rehost (Lift & Shift):                                      │
  │  ├── Move VMs as-is to Azure                                   │
  │  ├── Fastest migration, minimal changes                        │
  │  ├── Use: Azure Migrate for assessment + migration             │
  │  └── Risk: you bring all the on-prem problems with you        │
  │                                                                  │
  │  2. Replatform (Lift & Optimize):                               │
  │  ├── Minor changes to leverage PaaS                            │
  │  ├── Example: SQL Server → Azure SQL, IIS → App Service       │
  │  └── Best balance of effort vs benefit                         │
  │                                                                  │
  │  3. Refactor (Re-architect):                                    │
  │  ├── Redesign for cloud-native (microservices, serverless)    │
  │  ├── Most effort, most benefit                                 │
  │  └── Example: monolith → AKS microservices + Functions        │
  │                                                                  │
  │  4. Repurchase (Replace):                                       │
  │  ├── Switch to SaaS (e.g., Exchange → Microsoft 365)          │
  │  └── No custom development needed                              │
  │                                                                  │
  │  5. Retire:                                                      │
  │  ├── Turn off apps nobody uses anymore                         │
  │  └── Most companies find 10-20% of apps can be retired        │
  │                                                                  │
  │  6. Retain (Do Nothing):                                        │
  │  ├── Keep on-prem for now (compliance, latency)               │
  │  └── Re-evaluate in 12-18 months                               │
  └──────────────────────────────────────────────────────────────────┘
```

### 19.2 Azure Migrate

```bash
# Azure Migrate — Discover, Assess, Migrate
# Step 1: Create a migration project
az migrate project create \
    --name migrate-mycompany \
    --resource-group rg-migration \
    --location eastus

# Step 2: Deploy appliance on-prem (downloads OVA/VHD)
# Discovers servers, databases, web apps automatically

# Step 3: Assess readiness
az migrate assessment create \
    --project-name migrate-mycompany \
    --resource-group rg-migration \
    --name assess-batch1 \
    --sizing-criterion PerformanceBased \
    --time-range LastMonth
```

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  AZURE MIGRATION TOOLS                                           │
  │                                                                  │
  │  Azure Migrate (hub):                                            │
  │  ├── Server Migration: VMware/Hyper-V/Physical → Azure VMs    │
  │  ├── Database Migration: SQL Server → Azure SQL               │
  │  │   └── Azure Database Migration Service (DMS)               │
  │  ├── Web App Migration: IIS → App Service                     │
  │  │   └── App Service Migration Assistant                       │
  │  └── Data Box: ship petabytes via physical device              │
  │                                                                  │
  │  MIGRATION PHASES:                                               │
  │  ────────────────────────────────────────────────                │
  │  1. Discover    │ Find all assets (servers, DBs, apps)         │
  │  2. Assess      │ Readiness, sizing, cost estimation           │
  │  3. Plan        │ Migration waves, dependency mapping          │
  │  4. Migrate     │ Replication + cutover                        │
  │  5. Optimize    │ Right-size, enable monitoring, security      │
  │  6. Secure      │ Defender, Policy, RBAC, Private Endpoints   │
  │                                                                  │
  │  TYPICAL TIMELINE:                                               │
  │  ├── Small (10-50 servers): 2-3 months                        │
  │  ├── Medium (50-200 servers): 4-6 months                      │
  │  └── Large (200+ servers): 6-18 months                        │
  └──────────────────────────────────────────────────────────────────┘
```

### 19.3 Database Migration

```bash
# --- SQL Server → Azure SQL ---
# Using Azure Database Migration Service
az dms project create \
    --resource-group rg-migration \
    --service-name dms-migration \
    --name sqlmigration \
    --source-platform SQL \
    --target-platform SQLDB

# For minimal downtime: use Online migration mode
# 1. Full backup replicated
# 2. Change data capture (CDC) streams changes
# 3. Cutover: stop source, apply final changes, switch connection string

# --- Web App Migration ---
# Install App Service Migration Assistant
# 1. Scan IIS site for compatibility
# 2. Generate migration report
# 3. One-click migrate to Azure App Service
# Download: https://appmigration.microsoft.com
```

> **⚠️ Gotcha (migration order):** Always migrate in waves, NOT big-bang. Start with low-risk apps (dev/test environments), then move to less critical production, then mission-critical last. Each wave teaches you lessons that improve the next wave.

> **DevOps Relevance:** As a DevOps engineer, you'll often lead or support migrations. Azure Migrate automates discovery and assessment, but the real work is dependency mapping (which apps talk to which databases), network cutover planning, and DNS switchover. Always plan for rollback — keep on-prem running in parallel during the migration window.

---

## 20. Production Scenario FAQ

### Scenario-Based Q&A — 15 Real-World Azure Problems

---

**Scenario 1: "Our Azure bill jumped 40% this month and nobody knows why."**

```
Root cause: Usually untagged resources, orphaned disks, or someone
left GPU VMs running in dev.

Fix:
1. Cost Analysis → Group by "Resource type" → identify spikes
2. Cost Analysis → Filter by "Tag: environment = none" → find untagged
3. az disk list --query "[?managedBy==null]" → find orphaned disks
4. az vm list -d --query "[?powerState!='VM deallocated']" → running VMs
5. Set up budget alerts: $8K warning, $10K critical
6. Policy: enforce required tags on all resource groups

Prevention:
├── Tag everything in IaC (environment, team, cost-center)
├── Weekly Azure Advisor cost review
├── Auto-shutdown dev VMs (7 PM daily)
└── Monthly FinOps review meeting
```

---

**Scenario 2: "We deployed to production and the app is returning 502 Bad Gateway."**

```
Diagnosis:
1. App Service → Diagnose and solve problems → Web App Down
2. az webapp log tail -n myapp -g rg-prod → check application logs
3. App Insights → Failures → check exception details
4. Check if deployment swapped all settings:
   az webapp config appsettings list -n myapp -g rg-prod -o table

Common causes:
├── Missing app settings in production slot
├── Health check endpoint returning non-200
├── Connection string pointing to wrong database
├── App startup exceeding timeout (230 seconds default)
└── Out of memory on App Service Plan

Fix: Swap back to previous slot immediately:
az webapp deployment slot swap -n myapp -g rg-prod \
    --slot production --target-slot staging
```

---

**Scenario 3: "Terraform plan shows 15 resources will be destroyed and I didn't change anything."**

```
Root cause: State drift — someone made changes via Portal/CLI
that aren't reflected in your Terraform code.

Fix:
1. terraform plan → review WHAT is being destroyed
2. terraform state list → check what's in state
3. For each unexpected destroy:
   terraform import azurerm_resource.name /subscriptions/.../resource
4. Update .tf files to match current state
5. terraform plan → confirm no changes

Prevention:
├── Azure Policy: deny manual changes via Portal (DeployIfNotExists)
├── Lock production resource groups (ReadOnly)
├── All changes through CI/CD pipeline only
└── Weekly terraform plan --detailed-exitcode in monitoring
```

---

**Scenario 4: "Our AKS cluster is out of IP addresses."**

```
Root cause: Azure CNI assigns VNet IPs to every pod.
With 10 nodes × 30 pods/node = 300 IPs.
A /24 subnet (254 IPs) is exhausted.

Fix (short-term):
1. Scale down non-critical pods to free IPs
2. Reduce maxPodsPerNode if over-provisioned
3. Add IPs: expand subnet or add a new one

Fix (long-term):
1. Migrate to Azure CNI Overlay (pods use overlay IPs, not VNet IPs)
   az aks create ... --network-plugin azure --network-plugin-mode overlay
2. Use /20 or larger subnets for AKS (4,094 IPs)

Prevention:
├── IP planning formula: (nodes × maxPods) + nodes + 5
├── Use Azure CNI Overlay for new clusters
└── Monitor available IPs: Metrics → VNet → Subnet IP availability
```

---

**Scenario 5: "Service principal for our CI/CD pipeline expired and deployments are failing."**

```
Error: AADSTS700024: Client assertion is not within its valid time range.

Fix (immediate):
1. az ad sp credential reset --name sp-github-actions-prod
2. Update the secret in CI/CD (GitHub Secret, Azure DevOps variable)
3. Trigger a test deployment

Fix (permanent):
1. Switch to Workload Identity Federation (no secrets):
   az ad app federated-credential create \
     --id $APP_ID \
     --parameters credential.json
2. GitHub Actions: use azure/login@v2 with OIDC

Prevention:
├── Use federated credentials (zero secrets to expire)
├── If using SP secrets: set expiry to 2 years, calendar reminder
├── Monitor: az ad sp credential list --id $SP_ID --query "[].endDateTime"
└── Automate rotation with Azure Function on timer trigger
```

---

**Scenario 6: "Database connection from App Service fails after enabling Private Endpoint."**

```
Root cause: DNS still resolves sql-myapp.database.windows.net
to the PUBLIC IP, but public access is now disabled.

Fix:
1. Create Private DNS zone: privatelink.database.windows.net
2. Link zone to VNet with App Service VNet integration
3. Verify DNS resolution:
   az webapp ssh → nslookup sql-myapp.database.windows.net
   Should return private IP (10.0.3.x), NOT public IP

4. If using App Service:
   az webapp vnet-integration add \
     -n myapp -g rg-prod \
     --vnet vnet-main-prod --subnet snet-app

Checklist:
├── Private Endpoint created in correct subnet
├── Private DNS zone exists and linked to VNet
├── App Service has VNet integration enabled
├── SQL Server public network access disabled
└── NSG allows traffic from app subnet to DB subnet on port 1433
```

---

**Scenario 7: "Azure Monitor shows high memory usage but our app seems fine."**

```
Diagnosis:
1. Distinguish PLATFORM memory vs APPLICATION memory
2. App Service: "Memory working set" includes all processes
3. Check App Service metrics → Memory Percentage
4. Check per-instance: App Insights → Live Metrics → per-server

Common causes:
├── Memory leak in application (check App Insights → Performance)
├── Too many worker processes on shared App Service Plan
├── Background tasks accumulating (async jobs not completing)
└── In-memory cache growing unbounded

Fix:
├── Scale UP: increase App Service Plan tier
├── Scale OUT: add more instances (if per-instance memory is OK)
├── App fix: add memory limits, fix leaks
└── Move background jobs to separate App Service or Functions
```

---

**Scenario 8: "We need to give an external contractor access to only one resource group."**

```
Solution: Scoped RBAC + Conditional Access

1. Create guest account:
   az ad user invite \
     --invited-user-email contractor@external.com \
     --invite-redirect-url https://portal.azure.com

2. Assign narrow role at resource group scope:
   az role assignment create \
     --assignee contractor@external.com \
     --role "Contributor" \
     --resource-group rg-myapp-dev-eastus

3. Add Conditional Access policy:
   - Require MFA
   - Block from non-approved countries
   - Session timeout: 8 hours
   - No persistent browser sessions

4. Set expiry (remove after contract ends):
   az role assignment delete \
     --assignee contractor@external.com \
     --resource-group rg-myapp-dev-eastus

Best practices:
├── Never give subscription-level access to contractors
├── Use time-bound access (PIM for JIT activation)
├── Enable Entra ID Access Reviews (quarterly)
└── Audit with: az monitor activity-log list --caller contractor@external.com
```

---

**Scenario 9: "Our AKS pods can't pull images from ACR after cluster upgrade."**

```
Root cause: AKS managed identity lost ACR pull permission,
or the attach was on the old identity.

Fix:
1. Re-attach ACR to AKS:
   az aks update \
     -n aks-main-prod -g rg-prod \
     --attach-acr acrmyappprod

2. Verify:
   az aks check-acr -n aks-main-prod -g rg-prod \
     --acr acrmyappprod.azurecr.io

3. If still failing, check kubelet identity:
   KUBELET_ID=$(az aks show -n aks-main-prod -g rg-prod \
     --query identityProfile.kubeletidentity.clientId -o tsv)
   az role assignment list --assignee $KUBELET_ID -o table
```

---

**Scenario 10: "We're running Terraform in a pipeline but it keeps timing out."**

```
Common causes:
├── State lock stuck from a previous failed run
├── Large state file (1000+ resources)
├── Slow API responses from Azure (throttling)
└── Self-hosted agent with poor network connectivity

Fix state lock:
az storage blob lease break \
    --blob-name myapp.prod.tfstate \
    --container-name tfstate \
    --account-name stterraformstate

Fix throttling:
- Add parallelism limit: terraform apply -parallelism=5
- Split state: separate networking, compute, data into modules
- Use -target for partial applies during incidents

Fix large state:
- Split monolithic state into multiple state files
- Use terraform state mv to reorganize
- Consider Terragrunt for multi-state management
```

---

**Scenario 11: "Alerts are firing but nobody is responding."**

```
Root cause: Alert fatigue — too many non-actionable alerts.

Fix:
1. Audit all alerts: az monitor metrics alert list -o table
2. Classify each alert:
   ├── P1: needs immediate response (page on-call)
   ├── P2: needs response within 4 hours (Slack channel)
   ├── P3: informational (email digest)
   └── DELETE: alerts nobody ever acts on

3. Restructure action groups:
   ├── ag-page-oncall: PagerDuty/OpsGenie (P1 only)
   ├── ag-slack-ops: Slack #ops channel (P1 + P2)
   └── ag-email-weekly: weekly digest (P3)

4. Tune thresholds:
   ├── CPU > 80% for 15 min (not 1 min — avoids spikes)
   ├── Error rate > 5% (not single errors)
   └── Use dynamic thresholds (ML-based) for variable workloads

Prevention:
├── Every alert must have a runbook
├── Review alert metrics monthly (how many fired, how many acted on)
└── Delete alerts with 0 actions taken in 90 days
```

---

**Scenario 12: "We need to encrypt data at rest and in transit for compliance."**

```
Azure encryption by default:
├── At rest: ALL Azure Storage, SQL, Cosmos DB, Managed Disks
│   are encrypted with Microsoft-managed keys (SSE) — automatic
├── In transit: TLS 1.2 enforced on all Azure services
└── You don't need to DO anything for basic encryption

For stricter compliance (BYOK):
1. Create Key Vault with HSM-backed keys:
   az keyvault key create \
     --vault-name kv-myapp-prod \
     --name cmk-storage \
     --kty RSA --size 2048

2. Configure CMK on storage:
   az storage account update \
     --name stmyappprod \
     --resource-group rg-prod \
     --encryption-key-source Microsoft.Keyvault \
     --encryption-key-vault https://kv-myapp-prod.vault.azure.net \
     --encryption-key-name cmk-storage

Compliance checklist:
├── ✅ TLS 1.2+ enforced (az policy: append TLS settings)
├── ✅ Storage encrypted at rest (default SSE)
├── ✅ SQL TDE enabled (default)
├── ✅ Key Vault for secret management
├── ✅ Disable public network access (Private Endpoints)
└── ✅ Enable Defender for Cloud (compliance dashboard)
```

---

**Scenario 13: "Our multi-region app has split-brain during failover."**

```
Problem: Both regions think they're primary, writing to their
local databases, causing data inconsistency.

Fix:
1. Use Azure SQL Failover Groups (single read-write endpoint)
   Connection string: fog-myapp.database.windows.net
   → Always points to current primary

2. Application reads from:
   fog-myapp.database.windows.net (primary, read-write)
   fog-myapp.secondary.database.windows.net (secondary, read-only)

3. Front Door health probes detect primary down:
   → Routes traffic to secondary region
   → SQL failover group promotes secondary to primary (automatic)
   → Connection string fog-myapp.database.windows.net now points to new primary

Prevention:
├── Never use hardcoded regional endpoints for databases
├── Use failover group listener endpoints
├── Test failover quarterly with az sql failover-group set-primary
└── Design apps for eventual consistency (async replication has lag)
```

---

**Scenario 14: "We need to migrate 50TB of data from on-prem to Azure."**

```
Options (by data size):
├── < 10 TB: AzCopy over network
│   azcopy copy '/data/*' 'https://st.blob.core.windows.net/import?SAS' \
│     --recursive --put-md5

├── 10-100 TB: Azure Data Box (physical appliance)
│   1. Order Data Box from Portal
│   2. Microsoft ships you an 80TB appliance
│   3. Copy data to Data Box via SMB/NFS
│   4. Ship back to Azure datacenter
│   5. Data uploaded to your storage account
│   Timeline: 7-10 days total

├── 100+ TB: Azure Data Box Heavy (1 PB per device)
│   Same process, larger device

├── Ongoing sync: Azure File Sync
│   Keeps on-prem file server in sync with Azure Files
│   Gradual migration over weeks/months

Network estimation:
├── 50TB over 1 Gbps = ~4.6 days (best case)
├── 50TB over 100 Mbps = ~46 days
└── Data Box: ~7-10 days regardless of internet speed
```

---

**Scenario 15: "Our Azure DevOps pipeline takes 45 minutes and developers are frustrated."**

```
Diagnosis:
1. Pipeline → Analytics → identify slowest stages
2. Common bottlenecks:
   ├── Docker build (no layer caching)
   ├── npm/pip install (downloading every time)
   ├── Test suite (running ALL tests)
   └── Agent provisioning (hosted agent cold start)

Fix:
1. Docker: use multi-stage builds + ACR Tasks (cloud build)
   az acr build --registry myacr --image myapp:$TAG .

2. Dependency caching:
   - task: Cache@2
     inputs:
       key: 'pip | requirements.txt'
       path: $(PIP_CACHE_DIR)

3. Parallel test execution:
   pytest tests/ -n auto  # pytest-xdist

4. Skip unchanged paths:
   trigger:
     paths:
       include: ['src/*']    # Only trigger on source changes
       exclude: ['docs/*']

5. Self-hosted agents (warm cache, faster network):
   az vmss create for agent pool → persistent disk cache

Target: < 10 minutes for CI, < 15 minutes for full CD
```

---

[🏠 Home](../README.md) · [Azure](README.md)

