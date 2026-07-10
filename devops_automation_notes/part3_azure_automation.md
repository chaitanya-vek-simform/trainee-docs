# 🔷 DevOps Automation Notes — Part 3: Azure Automation Deep Dive

> **Series:** Part 3 of 4 | ← [Part 2](./part2_aws_automation.md) | [Part 4 →](./part4_best_practices_and_faq.md)

---

## Table of Contents

1. [Azure Resource Manager & Governance Automation](#1-azure-resource-manager--governance-automation)
2. [Virtual Machine & VMSS Automation](#2-virtual-machine--vmss-automation)
3. [Azure Kubernetes Service (AKS) Automation](#3-azure-kubernetes-service-aks-automation)
4. [Azure DevOps Pipelines Automation](#4-azure-devops-pipelines-automation)
5. [Key Vault & Secrets Automation](#5-key-vault--secrets-automation)
6. [Azure Monitor & Log Analytics Automation](#6-azure-monitor--log-analytics-automation)
7. [Azure Functions Serverless Automation](#7-azure-functions-serverless-automation)
8. [Terraform on Azure — IaC Automation](#8-terraform-on-azure--iac-automation)
9. [Security Automation on Azure](#9-security-automation-on-azure)
10. [Cost Management Automation on Azure](#10-cost-management-automation-on-azure)

---

## 1. Azure Resource Manager & Governance Automation

### Azure Hierarchy — Know This First

```
┌────────────────────────────────────────────────────────────────┐
│  AZURE ORGANIZATIONAL HIERARCHY                                │
│                                                                │
│  Entra ID Tenant (your company's identity boundary)           │
│       │                                                        │
│       ▼                                                        │
│  Management Groups (group subscriptions for policy)           │
│  ├── Production MG                                            │
│  └── Non-Production MG                                        │
│       │                                                        │
│       ▼                                                        │
│  Subscriptions (billing boundary + quota limit)               │
│  ├── prod-subscription   (budget: $50K/mo)                    │
│  └── dev-subscription    (budget: $5K/mo)                     │
│       │                                                        │
│       ▼                                                        │
│  Resource Groups (lifecycle boundary — deploy/delete together)│
│  └── rg-myapp-prod-eastus                                     │
│       │                                                        │
│       ▼                                                        │
│  Resources (VMs, databases, VNets, etc.)                      │
└────────────────────────────────────────────────────────────────┘
```

### What to Automate for Governance

| Task | Tool |
|------|------|
| Create resource groups with tags | Terraform / az cli |
| Enforce naming conventions | Azure Policy (deny non-compliant names) |
| Enforce mandatory tags | Azure Policy (deny resources without team/env tag) |
| Set budget alerts per subscription | Terraform `azurerm_consumption_budget_subscription` |
| Lock production resource groups | `az lock create --lock-type CanNotDelete` |
| Auto-tag resources with creator identity | Azure Policy with `modify` effect |
| Export resource group as ARM template | `az group export` |

### Automated Tagging Policy (via Azure Policy)

```
┌────────────────────────────────────────────────────────────────┐
│  AZURE POLICY — AUTO-ENFORCE TAGS                              │
│                                                                │
│  Policy effect: "modify"                                       │
│  Condition: resource missing tag "environment"                │
│  Action: append tag "environment" = subscription name         │
│                                                                │
│  Scope: Applied at Management Group level                     │
│  → Inherits down to ALL subscriptions, resource groups        │
│                                                                │
│  Best Practices:                                               │
│  ├── "environment" tag: dev/staging/prod                      │
│  ├── "team" tag: backend/frontend/platform                    │
│  ├── "cost-center" tag: CC1234                                │
│  └── "managed-by" tag: terraform/manual                       │
└────────────────────────────────────────────────────────────────┘
```

> **Gotcha:** `az group delete` is the most dangerous Azure CLI command. It deletes EVERYTHING in the resource group instantly. Always place a `CanNotDelete` lock on production resource groups. One mistyped group name can wipe out your entire production environment.

---

## 2. Virtual Machine & VMSS Automation

### VM Automation Architecture

```
┌────────────────────────────────────────────────────────────────┐
│  AZURE VM AUTOMATION — FULL LIFECYCLE                          │
│                                                                │
│  Terraform provisions VM                                       │
│  ├── VNet + Subnet + NSG                                      │
│  ├── NIC attached to subnet                                   │
│  ├── VM with managed disk                                     │
│  └── Extensions (Custom Script, Azure Monitor Agent)          │
│       │                                                        │
│       ▼                                                        │
│  cloud-init / Custom Script Extension runs on first boot      │
│  ├── Install Docker, Nginx, app dependencies                  │
│  └── Pull app from ACR and start                              │
│       │                                                        │
│       ▼                                                        │
│  Azure Monitor Agent streams metrics + logs                   │
│  └── To Log Analytics Workspace                               │
│       │                                                        │
│       ▼                                                        │
│  Azure Automation / Update Manager patches OS weekly          │
└────────────────────────────────────────────────────────────────┘
```

### Key VM Automation Tasks

| Task | How |
|------|-----|
| Provision VMs | Terraform `azurerm_linux_virtual_machine` |
| Bootstrap software | `custom_data` (cloud-init) or Custom Script Extension |
| Auto-scale VMs | VM Scale Sets (VMSS) + autoscale rules |
| Patch OS automatically | Azure Update Manager (scheduled, zero-downtime) |
| Run commands without SSH | `az vm run-command invoke` or Azure Bastion |
| Auto start/stop dev VMs | Azure Automation Runbook + schedule |
| Take VM snapshots | Azure Backup + Recovery Services Vault |

### Auto Start/Stop Dev VMs (Cost Saver)

```bash
# Azure CLI: Stop all VMs tagged environment=dev
# Run this via Azure Automation Runbook on a schedule

az vm list \
  --query "[?tags.environment=='dev'].{name:name,rg:resourceGroup}" \
  -o tsv | \
while read name rg; do
  echo "Stopping $name in $rg"
  az vm deallocate --name "$name" --resource-group "$rg" --no-wait
done
```

> **Gotcha — stop vs deallocate:** `az vm stop` stops the OS but you're **still billed** for the compute. `az vm deallocate` releases the underlying hardware — billing stops. Always use `deallocate` in cost-saving scripts. Never just `stop`.

---

## 3. Azure Kubernetes Service (AKS) Automation

### AKS Architecture

```
┌────────────────────────────────────────────────────────────────┐
│  AKS — MANAGED KUBERNETES ON AZURE                             │
│                                                                │
│  Azure manages:                You manage:                    │
│  ├── Control plane (free!)     ├── Node pools (VM-backed)     │
│  ├── etcd backups              ├── Kubernetes workloads       │
│  ├── K8s API server HA         ├── Helm releases              │
│  └── Control plane upgrades    ├── Ingress controllers        │
│                                └── Workload Identity (RBAC)   │
│                                                                │
│  AKS Automation Flow:                                          │
│                                                                │
│  1. Terraform creates AKS cluster + node pools                │
│  2. Helm installs NGINX Ingress, cert-manager                 │
│  3. Azure DevOps pipeline:                                    │
│     build image → push to ACR → helm upgrade → AKS           │
│  4. KEDA scales pods based on Azure Service Bus queue depth   │
│  5. Cluster Autoscaler adds/removes nodes automatically       │
│  6. Azure Monitor Container Insights collects all metrics     │
└────────────────────────────────────────────────────────────────┘
```

### What to Automate with AKS

| Task | Tool |
|------|------|
| Create AKS cluster | Terraform `azurerm_kubernetes_cluster` |
| Create node pools | `azurerm_kubernetes_cluster_node_pool` |
| Enable cluster autoscaler | `enable_auto_scaling = true` in Terraform |
| Install NGINX Ingress | Helm via `helm_release` in Terraform |
| Install cert-manager | Helm (auto-issue Let's Encrypt TLS certs) |
| Pod identity (no secrets) | Azure Workload Identity + OIDC |
| Pull images from ACR | Attach ACR to AKS: `az aks update --attach-acr` |
| Rolling deployments | Azure DevOps pipeline → `helm upgrade` |
| Auto-scale pods | HPA on CPU, or KEDA on Azure Queue/Event Hub |
| Collect logs | Container Insights (Azure Monitor agent in AKS) |
| Upgrade K8s version | Terraform update `kubernetes_version` + plan/apply |

### AKS + Azure DevOps Pipeline Flow

```yaml
# azure-pipelines.yml — Build and deploy to AKS
trigger:
  branches:
    include: [main]

stages:
- stage: Build
  jobs:
  - job: BuildAndPush
    steps:
    - task: Docker@2
      inputs:
        command: buildAndPush
        containerRegistry: myACRServiceConnection
        repository: myapp
        tags: $(Build.BuildId)

- stage: Deploy
  dependsOn: Build
  jobs:
  - deployment: DeployToAKS
    environment: production
    strategy:
      runOnce:
        deploy:
          steps:
          - task: HelmDeploy@0
            inputs:
              command: upgrade
              chartType: filepath
              chartPath: ./charts/myapp
              releaseName: myapp
              overrideValues: image.tag=$(Build.BuildId)
```

> **Gotcha:** Never pull container images using `:latest` in AKS. Always use the commit SHA or build ID as the tag. If two pods restart at different times and `:latest` points to different images, you get split-brain deployments.

---

## 4. Azure DevOps Pipelines Automation

### Pipeline Architecture

```
┌────────────────────────────────────────────────────────────────┐
│  AZURE DEVOPS — ALL-IN-ONE DEVOPS PLATFORM                     │
│                                                                │
│  Azure DevOps Services:                                        │
│  ├── Repos: Git repositories (like GitHub)                    │
│  ├── Boards: Work items / sprints (like Jira)                 │
│  ├── Pipelines: CI/CD (the most important for DevOps)         │
│  ├── Artifacts: package feeds (npm, pip, NuGet)               │
│  └── Test Plans: manual + automated testing                   │
│                                                                │
│  Pipeline Types:                                               │
│  ├── Build Pipeline (CI): triggered by PR/push                │
│  └── Release Pipeline (CD): triggered by build artifact       │
│                                                                │
│  Agent Types:                                                  │
│  ├── Microsoft-hosted: fresh VM every run (no maintenance)    │
│  └── Self-hosted: your VM (faster, custom tools, private VNet)│
└────────────────────────────────────────────────────────────────┘
```

### Pipeline Automation Best Practices

```yaml
# Full YAML pipeline example (azure-pipelines.yml)

trigger:
  branches:
    include: [main, release/*]
  paths:
    exclude: [docs/**, '*.md']   # Don't trigger on doc changes

pool:
  vmImage: ubuntu-latest

variables:
  - group: prod-secrets          # Variable group from Azure Key Vault
  - name: imageTag
    value: $(Build.BuildId)

stages:
- stage: CI
  displayName: Build & Test
  jobs:
  - job: Test
    steps:
    - script: |
        pip install -r requirements.txt
        pytest tests/ --junitxml=results.xml
      displayName: Run Tests
    - task: PublishTestResults@2
      inputs:
        testResultsFiles: results.xml

  - job: SecurityScan
    steps:
    - script: |
        trivy image --exit-code 1 --severity HIGH,CRITICAL myapp:$(imageTag)
      displayName: Container Security Scan

- stage: DeployDev
  dependsOn: CI
  condition: succeeded()
  jobs:
  - deployment: Dev
    environment: dev      # Environment with approval gates
    strategy:
      runOnce:
        deploy:
          steps:
          - script: helm upgrade --install myapp ./charts/myapp

- stage: DeployProd
  dependsOn: DeployDev
  condition: succeeded()
  jobs:
  - deployment: Prod
    environment: production   # Requires manual approval
    strategy:
      runOnce:
        deploy:
          steps:
          - script: helm upgrade --install myapp ./charts/myapp
```

> **Gotcha — Service Connections:** Azure DevOps pipelines authenticate to Azure via "Service Connections." Use **Workload Identity Federation** (OIDC) for service connections — not service principal client secrets. Secrets expire and rotate; OIDC tokens do not need rotation.

---

## 5. Key Vault & Secrets Automation

### Key Vault Mental Model

```
┌────────────────────────────────────────────────────────────────┐
│  AZURE KEY VAULT — CENTRAL SECRETS STORE                       │
│                                                                │
│  What it stores:                                               │
│  ├── Secrets: DB passwords, API keys, connection strings      │
│  ├── Keys: Encryption keys (RSA, EC) for your apps           │
│  └── Certificates: TLS/SSL certs + auto-renewal               │
│                                                                │
│  Access Pattern (NO hardcoded secrets):                        │
│                                                                │
│  App running on AKS/VM                                         │
│       │ uses Workload Identity / Managed Identity              │
│       ▼                                                        │
│  Entra ID authenticates                                        │
│       │                                                        │
│       ▼                                                        │
│  Key Vault returns secret                                      │
│       │                                                        │
│       ▼                                                        │
│  App uses the secret (never stored in code or env vars)       │
└────────────────────────────────────────────────────────────────┘
```

### What to Automate with Key Vault

| Task | How |
|------|-----|
| Create Key Vault | Terraform `azurerm_key_vault` |
| Store secrets | Terraform `azurerm_key_vault_secret` |
| Grant app access | Terraform RBAC: `azurerm_role_assignment` (Key Vault Secrets User) |
| Auto-rotate secrets | Key Vault + Event Grid event → Logic App / Function |
| Inject secrets into AKS pods | Azure Key Vault Provider for Secrets Store CSI Driver |
| Pull secrets in pipeline | Variable group linked to Key Vault |
| Certificate auto-renewal | Key Vault + DigiCert/Let's Encrypt integration |

### Pipeline Secret Injection (Variable Group from Key Vault)

```
Azure DevOps Variable Group
       │ linked to
       ▼
Azure Key Vault (kv-myapp-prod-eastus)
├── SECRET: DB-PASSWORD
├── SECRET: API-KEY
└── SECRET: REDIS-CONN-STR
       │
       ▼ referenced in pipeline as
$(DB-PASSWORD) → available as env var in pipeline steps
```

> **Gotcha — Key Vault Soft Delete:** Azure Key Vault has "soft delete" enabled by default — deleted secrets/keys/vaults go to a recoverable state for 90 days. If you `terraform destroy` a Key Vault and create a new one with the same name in the same subscription, it will fail because the soft-deleted vault still occupies the name. You must first purge it: `az keyvault purge --name my-kv`.

---

## 6. Azure Monitor & Log Analytics Automation

### Observability Architecture

```
┌────────────────────────────────────────────────────────────────┐
│  AZURE MONITOR — COMPLETE OBSERVABILITY STACK                  │
│                                                                │
│  Data Sources:                                                 │
│  ├── VM: Azure Monitor Agent → metrics + logs                 │
│  ├── AKS: Container Insights → pod/node metrics + logs        │
│  ├── App: Application Insights → traces, requests, errors     │
│  └── Azure Services: built-in diagnostic settings            │
│       │                                                        │
│       ▼                                                        │
│  Log Analytics Workspace                                       │
│  └── All logs centralized here (query with KQL)               │
│       │                                                        │
│       ▼                                                        │
│  Azure Monitor Alerts                                          │
│  ├── Metric alert: CPU > 85% → Action Group                  │
│  ├── Log alert: ERROR count > 10/min → Action Group           │
│  └── Activity alert: "delete" operation → Action Group        │
│       │                                                        │
│       ▼                                                        │
│  Action Groups                                                 │
│  ├── Email: oncall@company.com                                │
│  ├── SMS: +1-555-000-0000                                     │
│  ├── Webhook: → PagerDuty / Slack                             │
│  └── Azure Function: auto-remediation                         │
└────────────────────────────────────────────────────────────────┘
```

### What to Automate in Azure Monitor

| Task | Tool |
|------|------|
| Create Log Analytics Workspace | Terraform `azurerm_log_analytics_workspace` |
| Enable diagnostic settings on all resources | Terraform `azurerm_monitor_diagnostic_setting` |
| Create metric alerts | `azurerm_monitor_metric_alert` |
| Create log search alerts | `azurerm_monitor_scheduled_query_rules_alert_v2` |
| Create action groups (Slack/email) | `azurerm_monitor_action_group` |
| Create dashboards | Azure Dashboard JSON → Terraform |
| Set log retention | `retention_in_days` in workspace Terraform |

### Automated Alert → Slack (Action Group + Logic App)

```
Azure Monitor Alert fires
       │
       ▼
Action Group: Webhook to Logic App URL
       │
       ▼
Logic App parses alert payload
       │
       ▼
HTTP POST to Slack incoming webhook
       │
       ▼
Slack message: "#alerts channel: CPU CRITICAL on vm-web01 (92%)"
```

> **Gotcha:** Azure Monitor log alerts have a minimum evaluation frequency of 1 minute, but log ingestion latency is typically 2-5 minutes. If you set a 1-minute evaluation window, you may miss events that haven't ingested yet. Use at least a 5-minute window for log-based alerts.

---

## 7. Azure Functions Serverless Automation

### Azure Functions for DevOps Automation

```
┌────────────────────────────────────────────────────────────────┐
│  AZURE FUNCTIONS — EVENT-DRIVEN AUTOMATION                     │
│                                                                │
│  Common Triggers for DevOps:                                   │
│  ├── Timer trigger: run every night at 2 AM                   │
│  ├── HTTP trigger: webhook from GitHub/Azure DevOps            │
│  ├── Blob trigger: file uploaded to storage                   │
│  ├── Service Bus trigger: message arrives in queue            │
│  ├── Event Grid trigger: Azure resource event                 │
│  └── Cosmos DB trigger: document changed                      │
│                                                                │
│  DevOps Automation Patterns:                                   │
│  ├── Nightly: delete untagged resources in dev subscription   │
│  ├── On PR merge: trigger Terraform plan and post to PR       │
│  ├── On Alert: auto-scale or restart unhealthy service        │
│  ├── On file upload: process CSV → insert to SQL              │
│  └── On schedule: generate weekly cost report → email         │
└────────────────────────────────────────────────────────────────┘
```

> **Gotcha — Consumption Plan Cold Starts:** Azure Functions on the Consumption plan cold-start in 1-10 seconds (Python/Java worse than Node/C#). For latency-sensitive webhooks, use the Premium plan (always-warm instances). For background DevOps automation tasks, cold starts don't matter.

---

## 8. Terraform on Azure — IaC Automation

### Terraform State on Azure (Remote Backend)

```
┌────────────────────────────────────────────────────────────────┐
│  TERRAFORM REMOTE STATE — AZURE BLOB STORAGE                   │
│                                                                │
│  Developer A ──┐                                               │
│                ├──► Azure Blob Storage (tfstate container)    │
│  Developer B ──┤    └── prod/terraform.tfstate               │
│                │                                               │
│  CI/CD ────────┘    Azure Blob provides:                      │
│                     ├── Blob Lease = state locking            │
│                     ├── Versioning = state history            │
│                     └── Encryption at rest = security        │
│                                                                │
│  Setup (do once manually — bootstrap problem):                 │
│  az group create -n tfstate-rg -l eastus                      │
│  az storage account create -n tfstateacct -g tfstate-rg       │
│  az storage container create -n tfstate --account tfstateacct │
└────────────────────────────────────────────────────────────────┘
```

### Standard Terraform Module Structure for Azure

```
infra/
├── main.tf          # Root module: calls child modules
├── variables.tf     # Input variables
├── outputs.tf       # Output values
├── providers.tf     # Azure provider + backend config
├── terraform.tfvars # Variable values (do NOT commit secrets)
└── modules/
    ├── networking/      # VNet, subnets, NSGs
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    ├── compute/         # VMs, VMSS, AKS
    ├── database/        # Azure SQL, CosmosDB
    └── monitoring/      # Workspace, alerts, action groups
```

### Terraform CI/CD Pipeline on Azure DevOps

```yaml
# terraform pipeline in azure-pipelines.yml
stages:
- stage: Plan
  jobs:
  - job: TerraformPlan
    steps:
    - script: terraform init
    - script: terraform fmt -check           # Fail if not formatted
    - script: terraform validate
    - script: |
        terraform plan -out=tfplan -detailed-exitcode
        echo "##vso[task.setvariable variable=PLAN_EXIT_CODE]$?"
    - task: PublishPipelineArtifact@1
      inputs:
        path: tfplan
        artifact: terraform-plan

- stage: Apply
  dependsOn: Plan
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  jobs:
  - deployment: TerraformApply
    environment: production       # Requires manual approval
    steps:
    - task: DownloadPipelineArtifact@2
      inputs:
        artifact: terraform-plan
    - script: terraform apply tfplan
```

> **Gotcha — Drift:** If someone makes a manual change in the Azure Portal to a Terraform-managed resource, Terraform will try to revert it on next `apply`. Enforce a policy: **no manual changes to Terraform-managed resources**. Use scheduled `terraform plan -refresh-only` runs to detect drift before it causes surprises.

---

## 9. Security Automation on Azure

### Azure Security Automation Stack

```
┌────────────────────────────────────────────────────────────────┐
│  AZURE SECURITY — AUTOMATION LAYERS                            │
│                                                                │
│  Layer 1: Identity & Access                                    │
│  ├── Workload Identity: pods/functions access resources        │
│  │   without secrets (OIDC-based)                             │
│  ├── Managed Identity: VMs/apps authenticate to Azure         │
│  ├── Conditional Access: enforce MFA, location policies       │
│  └── PIM (Privileged Identity Management): just-in-time admin │
│                                                                │
│  Layer 2: Network Security                                     │
│  ├── NSGs: auto-created by Terraform with least-privilege rules│
│  ├── Azure Firewall: centralized egress control               │
│  └── Private Endpoints: no public IPs for DBs/storage        │
│                                                                │
│  Layer 3: Monitoring & Detection                               │
│  ├── Microsoft Defender for Cloud: threat detection           │
│  ├── Azure Sentinel: SIEM + auto-response playbooks           │
│  └── Key Vault diagnostic logs: who accessed what secret      │
│                                                                │
│  Layer 4: Compliance                                           │
│  ├── Azure Policy: auto-deny non-compliant deployments        │
│  ├── Blueprints: package of policies + RBAC + ARM templates   │
│  └── Regulatory compliance: ISO 27001, SOC 2, PCI DSS maps   │
└────────────────────────────────────────────────────────────────┘
```

### Managed Identity Pattern (No Hardcoded Secrets)

```
┌────────────────────────────────────────────────────────────────┐
│  MANAGED IDENTITY — HOW IT WORKS                               │
│                                                                │
│  WITHOUT Managed Identity (bad):                               │
│  App config: DB_PASSWORD=sup3rs3cret  ← hardcoded/env var     │
│                                                                │
│  WITH Managed Identity (good):                                 │
│  VM/AKS pod has a Managed Identity (assigned in Terraform)    │
│       │                                                        │
│       │ Azure automatically provisions credentials             │
│       ▼                                                        │
│  App calls Azure SDK (no password in code)                    │
│  credential = DefaultAzureCredential()                        │
│       │                                                        │
│       ▼                                                        │
│  SDK gets token from local metadata endpoint (169.254.x.x)   │
│       │                                                        │
│       ▼                                                        │
│  Access Key Vault, Blob Storage, SQL with that token          │
└────────────────────────────────────────────────────────────────┘
```

> **Gotcha — Azure Policy vs RBAC:** Azure Policy controls what **resources can exist** (e.g., block creating VMs in unapproved regions). RBAC controls what **users/identities can do** (e.g., allow a user to read but not delete VMs). You need BOTH for a complete security posture.

---

## 10. Cost Management Automation on Azure

### Azure Cost Automation Toolkit

| Automation | Savings Potential | How |
|------------|------------------|-----|
| Dev VM auto-deallocate nights/weekends | 60-70% on dev | Azure Automation Runbook (schedule) |
| Spot node pools for AKS batch workloads | 60-90% | `priority = "Spot"` in node pool Terraform |
| Blob Storage lifecycle tiering | 50-90% on old data | `azurerm_storage_management_policy` |
| Azure Reservations | 30-72% on VMs/SQL | Azure Portal or Cost Management API |
| Delete unused managed disks | Stop orphan charges | Azure Resource Graph query + az cli |
| Budget alerts per subscription | Visibility | `azurerm_consumption_budget_subscription` |
| Azure Advisor auto-apply rightsizing | 10-40% | Azure Advisor recommendations API + automation |

### Automated Blob Lifecycle Policy (Terraform)

```hcl
resource "azurerm_storage_management_policy" "lifecycle" {
  storage_account_id = azurerm_storage_account.main.id

  rule {
    name    = "archive-old-logs"
    enabled = true
    filters {
      prefix_match = ["logs/"]
      blob_types   = ["blockBlob"]
    }
    actions {
      base_blob {
        tier_to_cool_after_days_since_modification_greater_than    = 30
        tier_to_archive_after_days_since_modification_greater_than = 90
        delete_after_days_since_modification_greater_than          = 365
      }
    }
  }
}
```

> **Gotcha — Reserved Instances on Azure:** Azure Reserved VM Instances are commitment-based (1 or 3 years). They apply automatically to matching VMs in your subscription — you don't assign them to specific VMs. If you delete the reserved VM and don't replace it with same-size VM, the reservation still charges you. Plan carefully before purchasing.

---

*← [Part 2](./part2_aws_automation.md) | Next → [Part 4: Best Practices & FAQ](./part4_best_practices_and_faq.md)*
