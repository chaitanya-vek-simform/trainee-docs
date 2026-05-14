# Terraform for Azure — Complete Notes

> **Easy language. Small notes. Real examples. DevOps Gotchas included.**
> ASCII diagrams wherever a picture helps more than words.

---

## Table of Contents

1. [Terraform & IaC Fundamentals](#1-terraform--iac-fundamentals)
2. [Terraform Core Workflow & Commands](#2-terraform-core-workflow--commands)
3. [State Management](#3-state-management)
4. [HCL Language Features](#4-hcl-language-features)
5. [Modules & Architecture Patterns](#5-modules--architecture-patterns)
6. [Azure Identity & Security](#6-azure-identity--security)
7. [Azure Networking](#7-azure-networking)
8. [Azure Compute, Storage & Services](#8-azure-compute-storage--services)
9. [CI/CD, GitOps & Deployment Patterns](#9-cicd-gitops--deployment-patterns)

---

## 1. Terraform & IaC Fundamentals

### Infrastructure as Code (IaC)

**What it is:** Managing infrastructure (servers, networks, databases) using code/config files instead of clicking in a UI.

```
Without IaC                   With IaC
-----------                   --------
Click Azure portal  →  →  →   Write code → Run → Done
Repeat for every env          Same code runs in Dev/Test/Prod
Forget what you clicked       Version controlled in Git
```

**Why it matters:**
- Reproducible: run same code → get same infra, every time.
- Auditable: see who changed what via Git history.
- Automated: plug into CI/CD pipelines.

> **DevOps Gotcha 🚨:** IaC is only as good as its tests. Always run `terraform plan` before `apply`. Blindly running `apply` in prod is how outages happen.

---

### Terraform

**What it is:** An open-source IaC tool by HashiCorp. Write config → Terraform figures out what to create/change/delete in any cloud.

```
  .tf files  →  [Terraform]  →  Azure / AWS / GCP / Kubernetes
```

**Key idea:** Terraform is **declarative**. You say *what* you want, not *how* to do it.

```hcl
# Tell Terraform: "I want a resource group named my-rg"
resource "azurerm_resource_group" "example" {
  name     = "my-rg"
  location = "East US"
}
```

**Terraform versions:** Terraform is open-source; **OpenTofu** is its community fork (fully compatible).

---

### HCL (HashiCorp Configuration Language)

**What it is:** The language Terraform config is written in. Looks like JSON but more human-friendly.

```hcl
# Basic structure
resource "TYPE" "NAME" {
  argument = value
}

# Example
resource "azurerm_resource_group" "rg" {
  name     = "prod-rg"
  location = "eastus"
  tags = {
    env = "prod"
  }
}
```

**3 key constructs:**
| Construct | Purpose |
|-----------|---------|
| `resource` | Create/manage infra |
| `data` | Read existing infra |
| `variable` | Accept input values |
| `output` | Export values |
| `locals` | Define reusable expressions |

> **DevOps Gotcha 🚨:** HCL is case-sensitive. `"EastUS"` ≠ `"eastus"`. Azure locations must match exactly — use `az account list-locations -o table` to get valid names.

---

### Provider

**What it is:** A plugin that lets Terraform talk to a specific platform (Azure, AWS, GCP, Kubernetes, etc.).

```
  Terraform Core
       │
       ├─── azurerm provider  →  Azure REST API
       ├─── aws provider      →  AWS API
       └─── helm provider     →  Kubernetes Helm
```

**How to declare:**
```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {}
}
```

> **DevOps Gotcha 🚨:** Always pin provider versions (`~> 3.0` not just `>= 3.0`). A major version bump (3.x → 4.x) can break your entire codebase silently if unpinned.

---

### AzureRM Provider

**What it is:** The official Terraform provider for Azure. Written by HashiCorp + Microsoft. Covers 90%+ of Azure services.

```hcl
provider "azurerm" {
  features {
    key_vault {
      purge_soft_delete_on_destroy = false  # protect prod vaults!
    }
  }
  subscription_id = var.subscription_id
}
```

**Authentication options (in order of security preference):**
1. Managed Identity (best for pipelines)
2. OIDC / Workload Identity Federation
3. Service Principal + Client Secret
4. Azure CLI (dev machines only)

> **DevOps Gotcha 🚨:** The `features {}` block is **required** even if empty. Forgetting it = immediate error.

---

### Resource

**What it is:** A single piece of infrastructure managed by Terraform (a VM, a VNet, a Key Vault, etc.).

```hcl
resource "<provider>_<type>" "<local_name>" {
  # arguments
}

# Example: Create a storage account
resource "azurerm_storage_account" "data" {
  name                     = "mystorageacct001"
  resource_group_name      = azurerm_resource_group.rg.name
  location                 = azurerm_resource_group.rg.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
}
```

**Reference format:** `<TYPE>.<NAME>.<ATTRIBUTE>`
```hcl
azurerm_resource_group.rg.name     # → "my-rg"
azurerm_resource_group.rg.location # → "eastus"
```

> **DevOps Gotcha 🚨:** Changing a resource's `name` argument usually **destroys and recreates** it. Check the Terraform docs "Forces new resource" marker before editing names in prod.

---

### Data Source

**What it is:** Read-only lookup of existing infrastructure (things Terraform didn't create).

```hcl
# "Give me info about an existing VNet I didn't create"
data "azurerm_virtual_network" "shared" {
  name                = "hub-vnet"
  resource_group_name = "network-rg"
}

# Use it
resource "azurerm_subnet" "app" {
  virtual_network_name = data.azurerm_virtual_network.shared.name
  ...
}
```

**Use data sources when:**
- Referencing infra managed by another team/state
- Looking up subscription info, tenant ID, etc.

```hcl
# Get current client config
data "azurerm_client_config" "current" {}

output "tenant_id" {
  value = data.azurerm_client_config.current.tenant_id
}
```

> **DevOps Gotcha 🚨:** If the resource a `data` block points to doesn't exist yet, `terraform plan` fails. This is a common pain point with cross-team shared infra.

---

### Idempotency

**What it is:** Running Terraform multiple times produces the same result. If nothing changed in code, nothing changes in infra.

```
Run 1:  terraform apply  → Creates 5 resources
Run 2:  terraform apply  → "No changes. Infrastructure is up-to-date."
Run 3:  terraform apply  → "No changes. Infrastructure is up-to-date."
```

> **DevOps Gotcha 🚨:** Some resources are NOT truly idempotent (e.g., `null_resource` with `triggers`, custom scripts). Design your modules to be truly idempotent or you'll get surprise changes on every run.

---

### Immutable Infrastructure

**What it is:** Instead of modifying existing servers, you replace them with new ones.

```
MUTABLE (old way):            IMMUTABLE (Terraform way):
Server v1 exists              Server v1 exists
  └─ patch it                   └─ create Server v2
  └─ upgrade packages           └─ switch traffic
  └─ config drift!              └─ destroy Server v1
                                └─ no drift!
```

**In Terraform:** Use `create_before_destroy = true` in lifecycle rules to ensure zero-downtime replacement.

---

## 2. Terraform Core Workflow & Commands

### The Core Workflow

```
  Write .tf files
       │
       ▼
  terraform init       ← Download providers & modules
       │
       ▼
  terraform validate   ← Check syntax
       │
       ▼
  terraform plan       ← Preview changes (safe, read-only)
       │
       ▼
  terraform apply      ← Make changes in Azure
       │
       ▼
  terraform destroy    ← Tear down everything (DANGER!)
```

---

### `terraform init`

**What it does:** Initializes working directory. Downloads providers, sets up backend.

```bash
terraform init
```

Output:
```
Initializing the backend...
Initializing provider plugins...
- Installing hashicorp/azurerm v3.99.0...
Terraform has been successfully initialized!
```

**Flags:**
```bash
terraform init -upgrade          # Upgrade providers to latest allowed version
terraform init -reconfigure      # Reset backend config (state migration)
terraform init -backend=false    # Skip backend init (useful in CI for just validate)
```

> **DevOps Gotcha 🚨:** Always commit `.terraform.lock.hcl` to Git. This pins exact provider versions for all team members. Never commit the `.terraform/` directory (add to `.gitignore`).

---

### `terraform plan`

**What it does:** Shows what Terraform *would* do — no changes made. Always run before `apply`.

```bash
terraform plan
terraform plan -out=tfplan        # Save plan to file
terraform plan -var-file=prod.tfvars
```

**Plan output symbols:**
```
+ create       (green)
~ update       (yellow/orange)
- destroy      (red)
-/+ destroy and recreate
```

> **DevOps Gotcha 🚨:** Always use `-out=tfplan` in CI/CD pipelines and then `apply tfplan`. This ensures what you approved is exactly what gets applied — no race conditions.

---

### `terraform apply`

**What it does:** Executes the plan and makes real changes in Azure.

```bash
terraform apply              # Interactive (prompts "yes")
terraform apply -auto-approve  # Non-interactive (for CI/CD)
terraform apply tfplan         # Apply saved plan file
```

> **DevOps Gotcha 🚨:** Never use `-auto-approve` manually in prod. Use it only in automated pipelines after a human-approved plan. One fat-finger `apply -auto-approve` in prod = your worst day.

---

### `terraform destroy`

**What it does:** Destroys ALL resources managed by this state. **Extremely destructive.**

```bash
terraform destroy
terraform destroy -target=azurerm_resource_group.rg   # destroy only one resource
```

> **DevOps Gotcha 🚨:** Add `prevent_destroy = true` in lifecycle blocks for critical prod resources (databases, Key Vaults). This makes Terraform refuse to destroy them even if you run `destroy`.

```hcl
lifecycle {
  prevent_destroy = true
}
```

---

### `terraform validate`

**What it does:** Checks if your HCL syntax is valid. Does NOT check against Azure — purely local.

```bash
terraform validate
# Success: Configuration is valid.
```

Use in CI to catch typos before a plan runs.

---

### `terraform fmt`

**What it does:** Auto-formats `.tf` files to canonical HCL style. Like `prettier` for Terraform.

```bash
terraform fmt           # Format current directory
terraform fmt -recursive  # Format all subdirectories too
terraform fmt -check    # Exit non-zero if files need formatting (use in CI)
```

> **DevOps Gotcha 🚨:** Run `terraform fmt -check` in your CI pipeline. Enforce formatting consistency across the team, otherwise PR reviews get polluted with whitespace arguments.

---

### `terraform refresh`

**What it does:** Updates the state file to match real-world infra without making changes. Detects drift.

```bash
terraform refresh
```

> **DevOps Gotcha 🚨:** As of Terraform v0.15.4+, `refresh` is built into `plan` automatically (`-refresh=true` by default). Explicit `terraform refresh` is now mostly legacy. Use `terraform plan -refresh-only` instead.

---

### `terraform import`

**What it does:** Brings existing Azure resources under Terraform management (adds them to state).

```bash
# Syntax: terraform import <resource_address> <azure_resource_id>
terraform import azurerm_resource_group.rg /subscriptions/00000000/resourceGroups/my-rg
```

**Workflow:**
```
1. Write resource block in .tf file
2. terraform import <address> <id>
3. terraform plan (should show no changes if written correctly)
```

> **DevOps Gotcha 🚨:** `import` only adds to state — it does NOT generate `.tf` config. You must write the config manually (or use `terraform import` blocks in TF 1.5+ with `terraform plan`). Forgetting to write the config = next `apply` destroys the resource.

**New way (Terraform 1.5+):**
```hcl
import {
  to = azurerm_resource_group.rg
  id = "/subscriptions/xxx/resourceGroups/my-rg"
}
```

---

### `terraform output`

**What it does:** Reads output values from the state.

```bash
terraform output
terraform output resource_group_name   # specific output
terraform output -json                 # machine-readable
```

**Define outputs:**
```hcl
output "resource_group_name" {
  value       = azurerm_resource_group.rg.name
  description = "Name of the resource group"
}
```

---

### `terraform workspace`

**What it is:** Named environments within a single backend. Share code, isolate state.

```bash
terraform workspace list
terraform workspace new dev
terraform workspace select prod
terraform workspace show    # current workspace
```

**Use in code:**
```hcl
resource "azurerm_resource_group" "rg" {
  name = "rg-${terraform.workspace}"  # → rg-dev, rg-prod
}
```

> **DevOps Gotcha 🚨:** Workspaces share the **same backend** (same storage account). If you're not careful, a `destroy` in a wrong workspace can wipe prod. Many teams prefer **separate state files per env** over workspaces. Choose wisely.

---

### Execution Plan

**What it is:** The output of `terraform plan` — a detailed list of actions Terraform will take.

```
Terraform will perform the following actions:

  # azurerm_resource_group.rg will be created
  + resource "azurerm_resource_group" "rg" {
      + id       = (known after apply)
      + location = "eastus"
      + name     = "my-rg"
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

---

### Dependency Graph

**What it is:** Terraform builds an internal graph of all resource dependencies. Resources are created/destroyed in the right order.

```
  azurerm_resource_group.rg
         │
         ├──► azurerm_virtual_network.vnet
         │           │
         │           └──► azurerm_subnet.app
         │
         └──► azurerm_storage_account.data
```

**Explicit dependency:**
```hcl
resource "azurerm_subnet" "app" {
  depends_on = [azurerm_virtual_network.vnet]  # explicit, usually not needed
}
```

Most dependencies are implicit via attribute references.

```bash
terraform graph | dot -Tpng > graph.png   # Visualize the graph
```

---

### Lock File (`.terraform.lock.hcl`)

**What it is:** Records the exact provider versions downloaded during `init`. Ensures team consistency.

```hcl
# .terraform.lock.hcl
provider "registry.terraform.io/hashicorp/azurerm" {
  version     = "3.99.0"
  constraints = "~> 3.0"
  hashes = [
    "h1:abc123...",
  ]
}
```

> **DevOps Gotcha 🚨:** **Always commit** `.terraform.lock.hcl`. Never commit `.terraform/` directory. The lock file is your guarantee that CI/CD uses the same provider version as your local machine.

---

### Version Constraints

**What it is:** Controls which provider/module versions are acceptable.

| Constraint | Meaning |
|------------|---------|
| `= 3.99.0` | Exactly this version |
| `~> 3.0`   | Any 3.x (but not 4.x) |
| `>= 3.0`   | 3.0 or higher (dangerous!) |
| `>= 3.0, < 4.0` | Same as `~> 3.0` |

```hcl
terraform {
  required_version = ">= 1.5.0"   # Terraform CLI version
  required_providers {
    azurerm = {
      version = "~> 3.99"
    }
  }
}
```

> **DevOps Gotcha 🚨:** `>= 3.0` without an upper bound is a trap. When 4.0 releases with breaking changes, your next `init -upgrade` can break everything. Always use `~>` for providers.

---

### Parallelism

**What it is:** Terraform creates/destroys resources in parallel when there's no dependency between them. Default: 10 concurrent operations.

```bash
terraform apply -parallelism=20   # increase for faster applies
terraform apply -parallelism=1    # serial (debug mode)
```

> **DevOps Gotcha 🚨:** High parallelism can hit Azure API rate limits. If you see `429 Too Many Requests`, reduce parallelism. Some Azure services (like Policy assignments) have strict rate limits.

---

## 3. State Management

### State File (`terraform.tfstate`)

**What it is:** A JSON file that records what resources Terraform manages and their current configuration. It's Terraform's "memory".

```
  Your .tf files  ←→  terraform.tfstate  ←→  Real Azure Resources
    (desired)            (recorded)             (actual)
```

**Never manually edit it.** Use `terraform state` commands instead.

```bash
terraform state list                    # List all managed resources
terraform state show azurerm_vm.web     # Show a specific resource's state
terraform state rm azurerm_vm.web       # Remove from state (doesn't delete in Azure)
terraform state mv old_addr new_addr    # Rename/move resource in state
```

> **DevOps Gotcha 🚨:** The state file contains **secrets in plaintext** (passwords, connection strings). NEVER store it in Git. Always use a Remote Backend with encryption.

---

### Backend

**What it is:** Where Terraform stores the state file.

```
  Local Backend  →  stores terraform.tfstate on your laptop (default)
  Remote Backend →  stores state in Azure Blob / Terraform Cloud / S3
```

**Backend config:**
```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "tfstate-rg"
    storage_account_name = "tfstateacct001"
    container_name       = "tfstate"
    key                  = "prod/terraform.tfstate"
  }
}
```

---

### Remote Backend

**What it is:** Storing state in a shared, cloud location. Required for teams.

```
  Developer A ─┐
                ├──► Azure Blob Storage (tfstate) ◄── Source of truth
  Developer B ─┘
  CI/CD pipeline ─►──────────────────────────────┘
```

**Azure Blob Backend setup:**
```bash
# Create backend infrastructure (do this once manually or via bootstrap script)
az group create -n tfstate-rg -l eastus
az storage account create -n tfstateacct001 -g tfstate-rg --sku Standard_LRS
az storage container create -n tfstate --account-name tfstateacct001
```

> **DevOps Gotcha 🚨:** The backend infra itself (storage account, container) must be created **before** running `terraform init`. This is the "bootstrap problem" — you can't use Terraform to create its own backend. Create it with Azure CLI or a one-time script.

---

### Local Backend

**What it is:** Default backend — stores `terraform.tfstate` in the current directory.

**Only use for:**
- Local experiments
- Single-person projects
- Learning/demo

**Never use for:**
- Team projects (others can't see your state)
- Production (state lost if machine crashes)

---

### State Locking

**What it is:** Prevents two people (or two CI runs) from running `apply` simultaneously, which would corrupt state.

```
  Dev A runs apply   →  LOCK acquired
                         ├── Dev B tries apply → "Error: state locked by Dev A"
                         └── Dev A finishes → LOCK released → Dev B can run
```

**Azure Blob automatically supports locking via Blob Leases.**

> **DevOps Gotcha 🚨:** If a Terraform run crashes mid-apply, the lock can get stuck. To force-unlock:
> ```bash
> terraform force-unlock <LOCK_ID>
> ```
> Only do this when you're 100% sure no other apply is running. Breaking a lock during an active apply = corrupted state.

---

### Drift

**What it is:** When real Azure resources differ from what's in the Terraform state. Happens when someone makes manual changes in the portal.

```
  Terraform State: VM size = Standard_D2s_v3
  Azure Reality:   VM size = Standard_D4s_v3  ← someone changed it manually!
  
  → This is DRIFT
```

**Detect drift:**
```bash
terraform plan -refresh-only   # Shows what changed outside Terraform
terraform apply -refresh-only  # Updates state to match reality (doesn't fix infra)
```

> **DevOps Gotcha 🚨:** Drift is the #1 enemy of IaC. Enforce a rule: **no manual changes in Azure portal for Terraform-managed resources**. Use Azure Policy to restrict direct changes, or at minimum audit with scheduled `plan -refresh-only` runs.

---

### Drift Detection

**What it is:** A process to regularly check for drift. Best practice: run `terraform plan` on a schedule in CI/CD.

```yaml
# In GitHub Actions — scheduled drift detection
on:
  schedule:
    - cron: '0 8 * * *'   # every day at 8am

jobs:
  drift-detect:
    steps:
      - run: terraform plan -detailed-exitcode
        # exit 0 = no changes, exit 2 = changes (drift!), exit 1 = error
```

---

### Taint

**What it is:** Mark a resource for forced recreation on next `apply`. Even if nothing in config changed.

```bash
# Old way (deprecated in TF 1.0+)
terraform taint azurerm_virtual_machine.web

# New way
terraform apply -replace=azurerm_virtual_machine.web
```

**Use case:** VM is corrupted, you want to force a fresh one without changing config.

> **DevOps Gotcha 🚨:** `terraform taint` is deprecated since TF 0.15.2. Always use `-replace` flag instead. Tainting a resource marks it in state as dirty — if someone runs `apply` before you, the resource gets destroyed unexpectedly.

---

### Remote State Storage (Cross-team)

**What it is:** Reading another team's state output to get resource IDs.

```hcl
# Read the network team's state
data "terraform_remote_state" "network" {
  backend = "azurerm"
  config = {
    resource_group_name  = "tfstate-rg"
    storage_account_name = "tfstateacct001"
    container_name       = "tfstate"
    key                  = "network/terraform.tfstate"
  }
}

# Use their VNet ID
resource "azurerm_subnet" "app" {
  virtual_network_name = data.terraform_remote_state.network.outputs.vnet_name
  ...
}
```

> **DevOps Gotcha 🚨:** Remote state reads create **tight coupling** between teams. If the network team renames an output, your code breaks. Prefer using `data` sources + known resource names over remote state reads where possible.

---

### State Migration

**What it is:** Moving state from one backend to another (e.g., local → Azure Blob).

```bash
# 1. Update backend config in .tf file
# 2. Run:
terraform init -migrate-state
# Terraform asks: "Do you want to copy existing state to new backend? yes"
```

> **DevOps Gotcha 🚨:** Always back up the state file before migrating:
> ```bash
> cp terraform.tfstate terraform.tfstate.backup
> ```

---

### Workspace Isolation

**What it is:** Each workspace has its own state file. Resources in `dev` workspace don't affect `prod`.

```
  Storage Container: tfstate/
  ├── default.tfstate
  ├── env:/dev/terraform.tfstate
  ├── env:/staging/terraform.tfstate
  └── env:/prod/terraform.tfstate
```

> **DevOps Gotcha 🚨:** Many experienced teams **avoid workspaces** for environment separation. They use **separate state files + separate backends per environment** instead. Workspaces make it easy to accidentally `apply` to the wrong env.

---

### State Corruption

**What it is:** State file becomes inconsistent or invalid. Usually caused by:
- Concurrent applies (no state locking)
- Manual edits to state file
- Interrupted apply with non-locking backend

**Recovery:**
```bash
terraform state pull > backup.tfstate  # pull current state
# Manually fix or restore from Azure Blob versioning
terraform state push fixed.tfstate     # push corrected state
```

> **DevOps Gotcha 🚨:** **Enable versioning** on your Azure Blob container storing state. This lets you restore a previous state version if corruption happens:
> ```bash
> az storage blob versioning enable --account-name tfstateacct001 --container-name tfstate
> ```

---

### Import Existing Resources

(Covered in Commands section — `terraform import` command)

Additional pattern — **bulk import** using `for_each` + import blocks (TF 1.5+):
```hcl
locals {
  resource_groups = {
    "network-rg" = "/subscriptions/xxx/resourceGroups/network-rg"
    "app-rg"     = "/subscriptions/xxx/resourceGroups/app-rg"
  }
}

import {
  for_each = local.resource_groups
  to       = azurerm_resource_group.rgs[each.key]
  id       = each.value
}
```

---

## 4. HCL Language Features

### Variables

**What it is:** Input parameters for your Terraform config. Avoid hardcoding values.

```hcl
# Declare
variable "location" {
  description = "Azure region to deploy resources"
  type        = string
  default     = "eastus"
}

# Use
resource "azurerm_resource_group" "rg" {
  location = var.location
}
```

**Ways to pass values:**
```bash
terraform apply -var="location=westus"           # CLI flag
terraform apply -var-file="prod.tfvars"          # vars file
# OR set env vars:
export TF_VAR_location=westus
```

**`terraform.tfvars` auto-loaded:**
```hcl
# terraform.tfvars
location = "eastus"
env_name = "prod"
```

---

### Variable Types

```hcl
variable "name" {
  type = string          # "hello"
}

variable "count_val" {
  type = number          # 42
}

variable "enabled" {
  type = bool            # true / false
}

variable "tags" {
  type = map(string)     # { env = "prod", team = "devops" }
}

variable "cidrs" {
  type = list(string)    # ["10.0.0.0/16", "10.1.0.0/16"]
}

variable "vm_config" {
  type = object({
    size     = string
    os       = string
    count    = number
  })
}
```

> **DevOps Gotcha 🚨:** Always define `type` and `description` for every variable. Untyped variables default to `any` which can lead to confusing errors when wrong types are passed.

---

### Sensitive Variables

**What it is:** Variables whose values should never appear in logs or plan output.

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}
```

Plan output hides the value:
```
~ db_password = (sensitive value)
```

> **DevOps Gotcha 🚨:** `sensitive = true` doesn't encrypt the value — it just hides it from output. The value is still **in plaintext in state**. Always use a Remote Backend with encryption + access controls for sensitive vars.

---

### Local Values (`locals`)

**What it is:** Reusable named expressions within a module. Like a variable but calculated, not input.

```hcl
locals {
  env    = "prod"
  prefix = "myapp-${local.env}"
  
  common_tags = {
    environment = local.env
    managed_by  = "terraform"
    team        = "devops"
  }
}

resource "azurerm_resource_group" "rg" {
  name = "${local.prefix}-rg"
  tags = local.common_tags
}
```

> **DevOps Gotcha 🚨:** Don't use locals as a workaround for complex logic. If a local block becomes unreadable, it's a sign the module needs to be refactored.

---

### Outputs

**What it is:** Export values from a module so they can be used by other modules or displayed after `apply`.

```hcl
output "storage_account_name" {
  value       = azurerm_storage_account.data.name
  description = "Storage account name for app logs"
  sensitive   = false
}
```

**Cross-module usage:**
```hcl
# In root module
module "network" {
  source = "./modules/network"
}

resource "azurerm_subnet" "app" {
  virtual_network_name = module.network.vnet_name   # output from child module
}
```

---

### Interpolation

**What it is:** Embedding expressions inside strings using `${}`.

```hcl
name = "rg-${var.env}-${var.location}"     # → "rg-prod-eastus"
name = "vm-${count.index + 1}"             # → "vm-1", "vm-2"...

# Multi-line with heredoc
custom_data = <<-EOT
  #!/bin/bash
  echo "Hello from ${var.env} environment"
EOT
```

---

### Conditional Expressions

**What it is:** Ternary operator for conditional values.

```hcl
# Syntax: condition ? true_value : false_value

variable "is_prod" {
  default = false
}

resource "azurerm_storage_account" "data" {
  account_replication_type = var.is_prod ? "GRS" : "LRS"
}

# Conditional resource count
resource "azurerm_public_ip" "nat" {
  count = var.enable_nat ? 1 : 0
  ...
}
```

---

### Functions

**What it is:** Built-in Terraform functions for transforming values.

```hcl
# String functions
lower("HELLO")          → "hello"
upper("hello")          → "HELLO"
replace("hello", "l", "r") → "herro"
trimspace("  hi  ")     → "hi"
format("vm-%02d", 5)    → "vm-05"

# Collection functions
length(["a","b","c"])   → 3
join("-", ["a","b"])    → "a-b"
split(",", "a,b,c")     → ["a","b","c"]
merge({a=1}, {b=2})     → {a=1, b=2}
flatten([[1,2],[3]])     → [1,2,3]

# Type conversion
tostring(42)             → "42"
tonumber("42")           → 42

# File functions
file("./startup.sh")     → contents of file
```

**Useful for Azure naming:**
```hcl
locals {
  safe_name = lower(replace(var.name, " ", "-"))
}
```

---

### Count

**What it is:** Create multiple instances of a resource using an integer.

```hcl
resource "azurerm_virtual_machine" "web" {
  count = 3

  name = "vm-web-${count.index + 1}"   # vm-web-1, vm-web-2, vm-web-3
  ...
}

# Reference one: azurerm_virtual_machine.web[0]
# Reference all: azurerm_virtual_machine.web[*].id
```

> **DevOps Gotcha 🚨:** If you remove an item from the middle of a `count` list, Terraform renumbers all resources after it. This causes **unnecessary destroys and recreates**. For anything important, use `for_each` instead.

---

### For Each

**What it is:** Create resources from a map or set. Each item gets a stable key — no renumbering problem.

```hcl
variable "subnets" {
  default = {
    "frontend" = "10.0.1.0/24"
    "backend"  = "10.0.2.0/24"
    "database" = "10.0.3.0/24"
  }
}

resource "azurerm_subnet" "subnets" {
  for_each = var.subnets

  name             = each.key    # "frontend", "backend", "database"
  address_prefixes = [each.value]
  ...
}

# Reference: azurerm_subnet.subnets["frontend"].id
```

> **DevOps Gotcha 🚨:** `for_each` requires a **set or map** — not a list with duplicate values. Convert a list to a set: `for_each = toset(var.my_list)`. Also, `for_each` values must be known at plan time — they can't depend on resources that don't exist yet.

---

### Dynamic Blocks

**What it is:** Generate repeated nested blocks dynamically (like `count`/`for_each` but for blocks inside a resource).

```hcl
variable "inbound_rules" {
  default = [
    { port = 80,  priority = 100 },
    { port = 443, priority = 110 },
    { port = 22,  priority = 120 }
  ]
}

resource "azurerm_network_security_group" "nsg" {
  name = "my-nsg"
  ...

  dynamic "security_rule" {
    for_each = var.inbound_rules
    content {
      name                       = "rule-${security_rule.value.port}"
      priority                   = security_rule.value.priority
      direction                  = "Inbound"
      access                     = "Allow"
      protocol                   = "Tcp"
      destination_port_range     = tostring(security_rule.value.port)
      ...
    }
  }
}
```

---

### Lifecycle Rules

**What it is:** Control how Terraform creates, updates, and deletes resources.

```hcl
resource "azurerm_storage_account" "data" {
  ...
  lifecycle {
    prevent_destroy       = true      # refuse to destroy this resource
    create_before_destroy = true      # create new before destroying old
    ignore_changes        = [tags]    # don't track changes to tags
  }
}
```

| Rule | Use case |
|------|---------|
| `prevent_destroy` | Protect prod databases |
| `create_before_destroy` | Zero-downtime replacements |
| `ignore_changes` | Tags managed outside Terraform |
| `replace_triggered_by` | Force recreate when another resource changes |

> **DevOps Gotcha 🚨:** `ignore_changes = [all]` is tempting but dangerous — it means Terraform will never update this resource regardless of config changes. Use specific attribute names instead.

---

### Provisioners

**What it is:** Run scripts on resources after creation. **Last resort** — prefer cloud-native alternatives.

```hcl
resource "azurerm_virtual_machine" "web" {
  ...

  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y nginx",
    ]
    connection {
      type     = "ssh"
      user     = "azureuser"
      password = var.admin_password
      host     = self.public_ip_address
    }
  }
}
```

**Types:**
- `local-exec`: Runs on the Terraform machine
- `remote-exec`: SSH/WinRM into the resource
- `file`: Copy files to resource

> **DevOps Gotcha 🚨:** Provisioners break idempotency, require network access, and fail silently in complex scenarios. Use **Custom Script Extension**, **Cloud-init**, or **Azure VM Run Command** instead. HashiCorp themselves say provisioners are a "last resort".

---

### Null Resource

**What it is:** A resource with no real infrastructure. Used to run provisioners or trigger local scripts.

```hcl
resource "null_resource" "post_deploy" {
  triggers = {
    vm_id = azurerm_virtual_machine.web.id   # re-run if VM changes
  }

  provisioner "local-exec" {
    command = "echo 'VM created: ${azurerm_virtual_machine.web.name}'"
  }
}
```

> **DevOps Gotcha 🚨:** `null_resource` is being replaced by `terraform_data` resource in TF 1.4+. Prefer `terraform_data` for new code:
```hcl
resource "terraform_data" "post_deploy" {
  triggers_replace = [azurerm_virtual_machine.web.id]
  provisioner "local-exec" { ... }
}
```

---

## 5. Modules & Architecture Patterns

### Modules

**What it is:** A reusable, self-contained Terraform config. Like a function in programming.

```
  root module (your main code)
       │
       ├── module "network"   → ./modules/network/
       ├── module "compute"   → ./modules/compute/
       └── module "security"  → github.com/org/terraform-modules//keyvault
```

**Call a module:**
```hcl
module "network" {
  source   = "./modules/network"     # local path

  vnet_name  = "prod-vnet"
  location   = var.location
  cidr_block = "10.0.0.0/16"
}

# Use outputs
resource "azurerm_subnet" "app" {
  virtual_network_name = module.network.vnet_name
}
```

---

### Root Module

**What it is:** The top-level directory where you run Terraform commands. Contains `main.tf`, `variables.tf`, `outputs.tf`.

```
my-infra/
├── main.tf           ← Root module: calls child modules
├── variables.tf      ← Input variables
├── outputs.tf        ← Root outputs
├── providers.tf      ← Provider config
├── versions.tf       ← Version constraints
└── modules/
    ├── network/
    └── compute/
```

---

### Child Module

**What it is:** Any module called by the root (or another module). Receives input via `variable`, exports via `output`.

```hcl
# modules/network/main.tf
resource "azurerm_virtual_network" "vnet" {
  name          = var.vnet_name
  address_space = [var.cidr_block]
  ...
}

# modules/network/outputs.tf
output "vnet_name" {
  value = azurerm_virtual_network.vnet.name
}

# modules/network/variables.tf
variable "vnet_name" { type = string }
variable "cidr_block" { type = string }
```

> **DevOps Gotcha 🚨:** Don't nest modules more than 2-3 levels deep. Deep nesting makes debugging a nightmare. Outputs must be explicitly propagated up through every level — forgetting one = "undefined" errors.

---

### Module Registry

**What it is:** Public or private registry for sharing Terraform modules. [registry.terraform.io](https://registry.terraform.io) is the official one.

```hcl
# Use a community AKS module
module "aks" {
  source  = "Azure/aks/azurerm"
  version = "~> 7.0"

  resource_group_name = azurerm_resource_group.rg.name
  prefix              = "myaks"
}
```

> **DevOps Gotcha 🚨:** Community registry modules are convenient but can have opinions about naming, RBAC, or defaults that conflict with your org's standards. Always review module source before using in prod. Prefer your org's **internal private registry** for production.

---

### Modular Architecture

**What it is:** Structuring Terraform code into composable modules that can be mixed and matched.

```
Infrastructure Repository Structure:
  terraform/
  ├── environments/
  │   ├── dev/
  │   │   └── main.tf      ← calls modules with dev vars
  │   └── prod/
  │       └── main.tf      ← calls same modules with prod vars
  └── modules/
      ├── networking/
      ├── compute/
      ├── security/
      └── storage/
```

**Benefits:**
- DRY (Don't Repeat Yourself)
- Test modules independently
- Different teams own different modules

---

### Reusable Infrastructure

**What it is:** Write once, deploy many times with different inputs.

```hcl
# Same module, different environments
module "dev_network" {
  source     = "./modules/network"
  vnet_name  = "dev-vnet"
  cidr_block = "10.1.0.0/16"
}

module "prod_network" {
  source     = "./modules/network"
  vnet_name  = "prod-vnet"
  cidr_block = "10.0.0.0/16"
}
```

---

### Infrastructure Lifecycle Management

**What it is:** Managing infra through its entire life: create → update → scale → destroy.

```
  Plan → Apply → Monitor → Detect Drift → Update Plan → Apply Again → ...
                                                                    → Destroy
```

**Stages:**
1. **Provisioning**: `terraform apply` creates resources
2. **Configuration**: Cloud-init, CSE, Ansible configures VMs
3. **Maintenance**: Drift detection, updates via PRs
4. **Decommission**: `terraform destroy` or `state rm` + delete in Azure

---

### Terraform Cloud

**What it is:** HashiCorp's managed Terraform platform. Provides remote state, remote runs, team collaboration, and Sentinel policies.

**Free tier:** Up to 5 users, remote state, remote runs.

```hcl
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      name = "azure-prod"
    }
  }
}
```

**Features:**
- UI to see plan output
- Human approval workflow for apply
- VCS integration (GitHub, ADO)
- Cost estimation

---

### Terraform Enterprise

**What it is:** Self-hosted version of Terraform Cloud for orgs with strict data sovereignty needs.

**Adds:**
- SAML/SSO
- Audit logging
- Private network deployment
- Air-gapped support

> **DevOps Gotcha 🚨:** Terraform Cloud/Enterprise charges per resource under management. In large environments this gets expensive. Many orgs use **Azure DevOps** + **Azure Blob backend** as a free alternative.

---

### Sentinel Policies

**What it is:** Policy-as-code framework in Terraform Cloud/Enterprise. Enforces rules on Terraform plans before apply.

```python
# Example Sentinel policy: No public IPs allowed in prod
import "tfplan/v2" as tfplan

public_ips = filter tfplan.resource_changes as _, rc {
  rc.type is "azurerm_public_ip" and
  rc.mode is "managed" and
  (rc.change.actions contains "create" or rc.change.actions contains "update")
}

main = rule {
  length(public_ips) is 0
}
```

> **DevOps Gotcha 🚨:** Sentinel is only available on Terraform Cloud Team+ / Enterprise. For open-source Terraform, use **Open Policy Agent (OPA)** + `conftest` as a free alternative.

---

### Bootstrap Infrastructure

**What it is:** The foundational infra that must exist before Terraform can run (storage account for state, Service Principal, Key Vault for secrets).

```
Bootstrap (one-time, manual or script):
  1. Create Azure Storage Account for tfstate
  2. Create Service Principal with Contributor role
  3. Store SPN credentials in Key Vault
  4. Grant CI/CD system access to Key Vault

After bootstrap:
  → All other infra managed by Terraform
```

> **DevOps Gotcha 🚨:** Bootstrap infra is the "chicken and egg" problem. Document your bootstrap script clearly. Some teams put bootstrap in a separate `bootstrap/` Terraform module that uses a local backend (so at least it's code, even if not remote-state backed).

---

### Dependency Injection

**What it is:** Passing values into modules via variables instead of hardcoding them. Makes modules flexible and testable.

```hcl
# BAD: hardcoded
resource "azurerm_subnet" "app" {
  virtual_network_name = "prod-vnet"   # hardcoded = inflexible
}

# GOOD: injected via variable
variable "vnet_name" {}
resource "azurerm_subnet" "app" {
  virtual_network_name = var.vnet_name  # injected = reusable
}
```

---

## 6. Azure Identity & Security

### Azure Subscription

**What it is:** A billing and management boundary in Azure. All resources live inside a subscription.

```
  Azure Account (your Microsoft account)
       │
       ├── Tenant (your organization's Entra ID directory)
       │       │
       │       ├── Subscription: Dev   (separate billing)
       │       ├── Subscription: Test
       │       └── Subscription: Prod
```

**In Terraform:**
```hcl
provider "azurerm" {
  subscription_id = "00000000-0000-0000-0000-000000000000"
  features {}
}
```

---

### Tenant ID

**What it is:** Unique ID of your Microsoft Entra ID (formerly Azure AD) directory. Every organization has one tenant.

```bash
az account show --query tenantId -o tsv
```

---

### Client ID

**What it is:** The "username" of a Service Principal or App Registration. Also called `application_id`.

---

### Client Secret

**What it is:** The "password" of a Service Principal. Used with `client_id` to authenticate.

```hcl
provider "azurerm" {
  tenant_id       = var.tenant_id
  subscription_id = var.subscription_id
  client_id       = var.client_id       # SPN App ID
  client_secret   = var.client_secret   # SPN Password
  features {}
}
```

> **DevOps Gotcha 🚨:** Client secrets expire (default 6-24 months). Rotating them requires updating all pipelines. Use **Managed Identity** or **OIDC** in production to avoid secret rotation pain entirely.

---

### Service Principal (SPN)

**What it is:** A non-human identity in Entra ID used by applications/pipelines to authenticate to Azure. Like a robot user account.

**Create SPN:**
```bash
az ad sp create-for-rbac \
  --name "terraform-prod-sp" \
  --role Contributor \
  --scopes /subscriptions/<SUBSCRIPTION_ID>
# Output: appId (=client_id), password (=client_secret), tenant
```

> **DevOps Gotcha 🚨:** Don't give SPN `Owner` or broad `Contributor` at subscription level. Follow least privilege — give only the roles needed for the specific resources Terraform manages.

---

### Managed Identity

**What it is:** An identity automatically managed by Azure for a resource (VM, AKS node, pipeline agent). No secrets needed — Azure handles the credentials.

```
  VM with Managed Identity
       │
       └──► "I am this VM. Give me a token."
              │
       Azure  └──► validates and returns token
              │
       VM uses token to authenticate to Key Vault, Storage, etc.
       No password stored anywhere!
```

**Types:**
- **System-assigned**: Tied to a resource's lifecycle. Deleted when resource is deleted.
- **User-assigned**: Standalone. Can be assigned to multiple resources.

> **DevOps Gotcha 🚨:** Prefer **User-assigned Managed Identity** for Terraform pipelines running on Azure-hosted agents. System-assigned identity is deleted if the agent VM is recreated.

---

### OIDC Authentication

**What it is:** Passwordless authentication using OpenID Connect tokens from GitHub Actions / Azure DevOps to authenticate to Azure. **Best practice for CI/CD.**

```
  GitHub Actions workflow
       │
       └──► requests OIDC token from GitHub
              │
       Azure Federated Credential says "I trust this repo's tokens"
              │
       Azure issues access token
              │
       Terraform authenticates — no secrets stored!
```

**In GitHub Actions:**
```yaml
- uses: azure/login@v1
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

> **DevOps Gotcha 🚨:** OIDC requires creating a **Federated Credential** in Entra ID that trusts your specific repo + branch. Be specific: don't trust all branches, only `main` and specific environments.

---

### Azure CLI Authentication

**What it is:** Terraform uses your logged-in `az` CLI session for authentication. Dev machines only.

```bash
az login
az account set --subscription "my-sub"
terraform plan   # uses your az login session
```

> **DevOps Gotcha 🚨:** Never use Azure CLI auth in CI/CD pipelines. It requires interactive login or cached tokens that expire. Use SPN, Managed Identity, or OIDC in pipelines.

---

### Microsoft Entra ID

**What it is:** Azure's cloud identity and access management service (formerly Azure Active Directory). Manages users, groups, service principals, app registrations.

```
  Microsoft Entra ID (= Azure AD)
       │
       ├── Users & Groups
       ├── Service Principals (robot users)
       ├── App Registrations (apps)
       ├── Enterprise Applications
       └── Conditional Access Policies
```

---

### RBAC (Role-Based Access Control)

**What it is:** Control who can do what on which Azure resources. Based on 3 things: **Who** (principal) + **What** (role) + **Where** (scope).

```
  Role Assignment:
  ┌─────────┐    ┌─────────────┐    ┌──────────────────┐
  │  WHO    │    │    WHAT     │    │      WHERE       │
  │  (SPN)  │ + │ (Contributor)│ + │ (Subscription /  │
  │  (User) │    │ (Reader)    │    │  Resource Group / │
  │  (Group)│    │ (Owner)     │    │  Resource)       │
  └─────────┘    └─────────────┘    └──────────────────┘
```

**In Terraform:**
```hcl
resource "azurerm_role_assignment" "sp_contributor" {
  principal_id         = azurerm_user_assigned_identity.mi.principal_id
  role_definition_name = "Contributor"
  scope                = azurerm_resource_group.rg.id
}
```

---

### Role Assignment

**What it is:** The act of assigning a role to a principal at a scope.

**Built-in roles:**
| Role | Can do |
|------|--------|
| Owner | Everything + manage access |
| Contributor | Everything except manage access |
| Reader | View only |
| Storage Blob Data Contributor | Read/write blobs |
| Key Vault Secrets User | Read secrets |

> **DevOps Gotcha 🚨:** Role assignments take 1-5 minutes to propagate in Azure. If Terraform applies a role and immediately tries to use it, it can fail with `403 Forbidden`. Add explicit dependencies or a small `time_sleep` resource.

---

### Least Privilege

**What it is:** Give identities only the minimum permissions they need. Nothing more.

```
BAD:  Terraform SPN gets Owner on entire subscription
GOOD: Terraform SPN gets Contributor only on specific resource groups it manages
```

**Steps:**
1. Identify what resources Terraform creates
2. Check what Azure roles allow those operations
3. Assign only those roles, at the narrowest scope

---

### Key Vault

**What it is:** Azure's secret/key/certificate store. Like a secure safe for sensitive config.

```hcl
resource "azurerm_key_vault" "kv" {
  name                = "myapp-kv-prod"
  resource_group_name = azurerm_resource_group.rg.name
  location            = var.location
  tenant_id           = data.azurerm_client_config.current.tenant_id
  sku_name            = "standard"

  soft_delete_retention_days = 90
  purge_protection_enabled   = true   # CRITICAL: prevents permanent deletion
}
```

> **DevOps Gotcha 🚨:** Enable `purge_protection_enabled = true` on prod Key Vaults. Without it, a `terraform destroy` followed by a `purge` permanently deletes all secrets. With purge protection, even after "deletion" secrets survive the retention period.

---

### Secrets, Certificates, Access Policies

**Secrets:** Store passwords, connection strings, API keys.
```hcl
resource "azurerm_key_vault_secret" "db_password" {
  name         = "db-password"
  value        = var.db_password   # sensitive variable
  key_vault_id = azurerm_key_vault.kv.id
}
```

**Certificates:** Store TLS certificates. Can auto-renew from Let's Encrypt or internal CA.

**Access Policies (old model):**
```hcl
resource "azurerm_key_vault_access_policy" "sp" {
  key_vault_id = azurerm_key_vault.kv.id
  tenant_id    = var.tenant_id
  object_id    = var.sp_object_id

  secret_permissions = ["Get", "List"]   # Read-only
}
```

> **DevOps Gotcha 🚨:** Key Vault now supports **Azure RBAC** model (preferred) OR Access Policies. Don't mix them — pick one model and stick with it. RBAC model is recommended for new deployments.

---

### Zero Trust

**What it is:** Security model: "Never trust, always verify." Every access request is verified regardless of network location.

```
Traditional Security:       Zero Trust:
  ┌──────────────────┐       ┌──────────────────┐
  │  Inside network  │       │ Verify EVERY      │
  │  = Trusted       │  vs   │ request, from     │
  │  Outside         │       │ anywhere, always  │
  │  = Blocked       │       │ Least privilege   │
  └──────────────────┘       └──────────────────┘
```

**In Azure, implement via:**
- Private Endpoints (no public internet access)
- Managed Identities (no stored credentials)
- Conditional Access (verify context of every login)
- Just-in-Time VM Access

---

### Azure Policy

**What it is:** Rules enforced across your Azure environment. Prevents/audits non-compliant resources.

```hcl
resource "azurerm_policy_assignment" "no_public_ips" {
  name                 = "deny-public-ips"
  scope                = azurerm_resource_group.rg.id
  policy_definition_id = "/providers/Microsoft.Authorization/policyDefinitions/..."
  description          = "Deny creation of public IPs"
}
```

**Effects:** `Deny` | `Audit` | `Append` | `Modify` | `DeployIfNotExists`

> **DevOps Gotcha 🚨:** Azure Policy with `Deny` effect will block your Terraform apply if you try to create non-compliant resources. Always test new policies in `Audit` mode first, then switch to `Deny`.

---

### Management Group

**What it is:** Container for organizing multiple subscriptions. Policies applied here cascade down to all subscriptions.

```
  Management Group: Root
       │
       ├── Management Group: Corp
       │       ├── Subscription: Dev
       │       ├── Subscription: Test
       │       └── Subscription: Prod
       └── Management Group: Sandbox
               └── Subscription: Experiments
```

> **DevOps Gotcha 🚨:** Management Group changes (moving subs) can affect Policy assignments and RBAC inheritance. Always test in non-prod before reorganizing.

---

## 7. Azure Networking

### Resource Group

**What it is:** Logical container for Azure resources. Everything in Azure lives in a resource group.

```
  Resource Group: prod-app-rg
  ├── Virtual Network: prod-vnet
  ├── Storage Account: prodstg001
  └── Key Vault: prod-kv
```

```hcl
resource "azurerm_resource_group" "rg" {
  name     = "prod-app-rg"
  location = "eastus"
  tags     = local.common_tags
}
```

> **DevOps Gotcha 🚨:** Deleting a resource group deletes **everything** inside it instantly. Add `prevent_destroy = true` lifecycle rule to prod resource groups.

---

### Region

**What it is:** Physical data center location. E.g., `eastus`, `westeurope`, `southeastasia`.

```bash
az account list-locations -o table   # list all valid region names
```

> **DevOps Gotcha 🚨:** Not all Azure services/SKUs are available in all regions. Check availability before choosing a region for prod. Also, `East US` ≠ `East US 2` — two different regions.

---

### Availability Zone

**What it is:** Physically separate data centers within a region. Use for HA — if one zone fails, others stay up.

```
  Region: East US
  ├── Zone 1 (Data Center A)
  ├── Zone 2 (Data Center B)
  └── Zone 3 (Data Center C)
  
  Deploy VM in Zone 1 + Zone 2 → one zone goes down → VM in other zone still runs
```

```hcl
resource "azurerm_linux_virtual_machine" "web" {
  zone = "1"   # pin to zone 1
  ...
}
```

---

### Availability Set

**What it is:** Older HA mechanism — groups VMs across separate physical servers and racks (fault domains) within a single zone. Use Availability Zones instead for modern deployments.

```hcl
resource "azurerm_availability_set" "avset" {
  name                = "app-avset"
  resource_group_name = azurerm_resource_group.rg.name
  location            = var.location
  platform_fault_domain_count  = 2
  platform_update_domain_count = 5
}
```

> **DevOps Gotcha 🚨:** Availability Sets and Availability Zones are mutually exclusive. You can't use both. Zones provide stronger SLA (99.99% vs 99.95%). Prefer Zones unless the region doesn't support them.

---

### Virtual Network (VNet)

**What it is:** Your private network in Azure. Like having your own LAN in the cloud.

```
  VNet: 10.0.0.0/16
  ├── Subnet: frontend  10.0.1.0/24
  ├── Subnet: backend   10.0.2.0/24
  └── Subnet: database  10.0.3.0/24
```

```hcl
resource "azurerm_virtual_network" "vnet" {
  name                = "prod-vnet"
  resource_group_name = azurerm_resource_group.rg.name
  location            = var.location
  address_space       = ["10.0.0.0/16"]
}
```

---

### Subnet

**What it is:** A subdivision of a VNet. Resources (VMs, AKS nodes, etc.) live in subnets.

```hcl
resource "azurerm_subnet" "backend" {
  name                 = "backend-subnet"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.2.0/24"]
}
```

> **DevOps Gotcha 🚨:** Some Azure services require **dedicated subnets** (AKS, Azure Firewall, App Gateway, Bastion). Plan your IP address space in advance — you can't split or merge subnets after VNet creation without destroying resources.

---

### NSG (Network Security Group)

**What it is:** A firewall at the subnet or NIC level. Contains inbound/outbound rules.

```
  Internet
     │
     ▼
  NSG: web-nsg
  ├── Inbound: Allow 443 from Internet
  ├── Inbound: Allow 80 from Internet
  ├── Inbound: Deny all
  └── Outbound: Allow all
     │
     ▼
  Subnet: frontend (10.0.1.0/24)
     │
     └── VMs
```

```hcl
resource "azurerm_network_security_group" "web" {
  name                = "web-nsg"
  resource_group_name = azurerm_resource_group.rg.name
  location            = var.location

  security_rule {
    name                       = "allow-https"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "443"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
}

# Associate NSG with subnet
resource "azurerm_subnet_network_security_group_association" "web" {
  subnet_id                 = azurerm_subnet.frontend.id
  network_security_group_id = azurerm_network_security_group.web.id
}
```

> **DevOps Gotcha 🚨:** Azure has a hidden default "deny all inbound from internet" rule (priority 65500). Don't add a redundant deny rule — it wastes priority space. NSG rules process lowest priority number first.

---

### Route Table & UDR (User Defined Route)

**What it is:** Custom routing rules. Force traffic to go through Azure Firewall, NVA (Network Virtual Appliance), or specific hops.

```
  VM in backend subnet → wants to reach internet
  Default route: go directly out
  UDR: go through Azure Firewall first → inspected → then to internet
```

```hcl
resource "azurerm_route_table" "rt" {
  name                = "backend-rt"
  resource_group_name = azurerm_resource_group.rg.name
  location            = var.location

  route {
    name                   = "to-firewall"
    address_prefix         = "0.0.0.0/0"
    next_hop_type          = "VirtualAppliance"
    next_hop_in_ip_address = "10.0.0.4"   # Azure Firewall private IP
  }
}
```

---

### Public IP & Private IP

**Public IP:** Internet-routable address. Attach to VMs, Load Balancers, App Gateways.

```hcl
resource "azurerm_public_ip" "vm_pip" {
  name                = "vm-public-ip"
  resource_group_name = azurerm_resource_group.rg.name
  location            = var.location
  allocation_method   = "Static"    # or "Dynamic"
  sku                 = "Standard"  # use Standard for zone-redundant
}
```

**Private IP:** Internal VNet address. Automatically assigned from subnet CIDR.

> **DevOps Gotcha 🚨:** Minimize public IPs — they increase attack surface. Use Azure Bastion for VM access, Azure Firewall for egress, Private Endpoints for PaaS services. Audit public IPs regularly with: `az network public-ip list -o table`

---

### NAT Gateway

**What it is:** Provides outbound internet connectivity for resources in a subnet without needing public IPs on each resource.

```
  VMs (private IPs only)
       │
       ▼
  NAT Gateway (has Public IP)
       │
       ▼
  Internet  ← all outbound traffic appears from NAT Gateway's IP
```

> **DevOps Gotcha 🚨:** NAT Gateway and Azure Load Balancer outbound rules conflict. Use one or the other for outbound — not both. NAT Gateway is generally preferred for its simplicity.

---

### VNet Peering

**What it is:** Connects two VNets so resources can communicate privately. No VPN or gateway needed.

```
  Hub VNet (10.0.0.0/16)  ←──── Peering ────►  Spoke VNet A (10.1.0.0/16)
  Hub VNet (10.0.0.0/16)  ←──── Peering ────►  Spoke VNet B (10.2.0.0/16)
  (Spoke A and Spoke B cannot talk directly)
```

```hcl
resource "azurerm_virtual_network_peering" "hub_to_spoke" {
  name                      = "hub-to-spoke-a"
  resource_group_name       = "network-rg"
  virtual_network_name      = azurerm_virtual_network.hub.name
  remote_virtual_network_id = azurerm_virtual_network.spoke_a.id
  allow_forwarded_traffic   = true
  allow_gateway_transit     = true  # Hub can share VPN gateway
}
```

> **DevOps Gotcha 🚨:** Peering is **not transitive**. Spoke A → Hub → Spoke B doesn't work automatically. You need a network virtual appliance (like Azure Firewall) in the hub to route traffic between spokes. Also, peering CIDRs must not overlap.

---

### VPN Gateway

**What it is:** Connect on-premises network to Azure VNet securely over internet (IPSec VPN).

```
  On-Premises Network  ←── encrypted VPN tunnel ──►  VPN Gateway  →  Azure VNet
```

> **DevOps Gotcha 🚨:** VPN Gateway deployment takes **25-40 minutes** in Azure. Terraform will wait for it. Don't set short timeouts for gateway resources.

---

### ExpressRoute

**What it is:** Private, dedicated, high-bandwidth connection from on-premises to Azure. Does NOT go over the internet.

```
  On-Premises Data Center
           │
  (fiber/MPLS circuit via connectivity provider)
           │
  Microsoft Edge (peering location)
           │
  ExpressRoute Circuit → Azure VNet
```

> **DevOps Gotcha 🚨:** ExpressRoute circuits exist outside your subscription — they're managed by connectivity providers. In Terraform, you often `import` or use `data` sources to reference existing circuits rather than creating them.

---

### Bastion

**What it is:** Managed PaaS service to securely RDP/SSH to VMs without public IPs. Access via browser.

```
  Browser → HTTPS → Azure Bastion (in AzureBastionSubnet) → SSH/RDP → Private VM
  No public IP on VM needed!
```

```hcl
resource "azurerm_bastion_host" "bastion" {
  name                = "prod-bastion"
  resource_group_name = azurerm_resource_group.rg.name
  location            = var.location

  ip_configuration {
    name                 = "configuration"
    subnet_id            = azurerm_subnet.bastion.id   # must be AzureBastionSubnet
    public_ip_address_id = azurerm_public_ip.bastion.id
  }
}
```

> **DevOps Gotcha 🚨:** Bastion requires a subnet named **exactly** `AzureBastionSubnet` with `/26` or larger. No other resources can be in this subnet.

---

### Load Balancer

**What it is:** Distributes incoming traffic across multiple VMs/backends.

```
  Internet traffic
       │
  ┌────▼──────────┐
  │  Load Balancer │   ← Layer 4 (TCP/UDP)
  └────────────────┘
  ┌──────────────────────────┐
  │  Backend Pool             │
  │  ├── VM1 (10.0.1.4)      │
  │  ├── VM2 (10.0.1.5)      │
  │  └── VM3 (10.0.1.6)      │
  └──────────────────────────┘
```

> **DevOps Gotcha 🚨:** Azure Load Balancer (Standard SKU) doesn't allow outbound internet by default when using it. You must configure outbound rules or use a NAT Gateway alongside it.

---

### Application Gateway

**What it is:** Layer 7 HTTP/HTTPS load balancer. Can route by URL path, hostname, do SSL termination, and has WAF.

```
  https://myapp.com/api/*   →  Backend Pool: API Servers
  https://myapp.com/web/*   →  Backend Pool: Web Servers
  (URL-based routing — Application Gateway does this, basic LB cannot)
```

---

### Web Application Firewall (WAF)

**What it is:** Add-on to Application Gateway that blocks common web attacks (SQL injection, XSS, etc.) using OWASP rules.

**Modes:**
- **Detection**: Logs attacks but doesn't block
- **Prevention**: Blocks attacks (use in prod)

> **DevOps Gotcha 🚨:** WAF in Prevention mode can block legitimate traffic. Always start in Detection mode, review logs for false positives, then switch to Prevention. Common false positive triggers: special characters in query strings, large request bodies.

---

### DNS Zone & Private DNS Zone

**DNS Zone:** Manages public DNS records (A, CNAME, MX, etc.) for your domain.

```hcl
resource "azurerm_dns_zone" "public" {
  name                = "mycompany.com"
  resource_group_name = azurerm_resource_group.rg.name
}

resource "azurerm_dns_a_record" "app" {
  name    = "app"
  zone_name = azurerm_dns_zone.public.name
  ...
  records = [azurerm_public_ip.app.ip_address]
}
```

**Private DNS Zone:** DNS that only resolves inside your VNet. Used with Private Endpoints.

```hcl
resource "azurerm_private_dns_zone" "blob" {
  name                = "privatelink.blob.core.windows.net"
  resource_group_name = azurerm_resource_group.rg.name
}
```

---

### Service Endpoint vs Private Endpoint

```
  Service Endpoint:                    Private Endpoint:
  ┌──────────────┐                    ┌──────────────┐
  │  VM in VNet  │──►─────────────►  │  VM in VNet  │
  └──────────────┘  traffic goes      └──────────────┘
         │          through Azure              │
         │          backbone (still            │
         │          public IP of               ▼
         │          service)          ┌──────────────────┐
         ▼                           │  Private Endpoint │
  Storage Account                   │  (private IP      │
  (public IP, but                   │   in your VNet)   │
   restricted to VNet)              └──────────────────┘
                                           │
                                    Storage Account
                                    (no public access)
```

> **DevOps Gotcha 🚨:** Private Endpoints require **Private DNS Zone** configuration. Without it, your VM can connect to the private IP but DNS still resolves the public hostname. Configure `azurerm_private_dns_zone_virtual_network_link` or DNS resolution breaks.

---

### Azure Firewall

**What it is:** Managed, cloud-native network security service. Filters all traffic (in/out/east-west) with FQDN rules, threat intelligence.

```
  All spoke VNet traffic
         │
         ▼
  Azure Firewall (in Hub VNet)
  ├── Application Rules: Allow *.microsoft.com
  ├── Network Rules: Allow 443 to known IPs
  └── DNAT Rules: Translate public IP → private VM
         │
         ▼
  Internet / On-premises
```

> **DevOps Gotcha 🚨:** Azure Firewall is expensive (~$1.25/hr + data processing). Only use it when you need centralized inspection, FQDN filtering, or threat intelligence. For basic NSG filtering, use NSGs. Don't use Azure Firewall as a replacement for NSGs.

---

### DDoS Protection

**What it is:** Protects Azure resources from volumetric network attacks.

**Tiers:**
- **Basic**: Free, enabled by default, protects Azure infrastructure
- **Standard**: Paid, per-VNet, adaptive tuning, attack analytics, and SLA guarantee

> **DevOps Gotcha 🚨:** DDoS Standard is ~$2,944/month per VNet (as of 2024). Only enable for VNets with public-facing workloads handling sensitive data. Cost-justify before enabling.

---

### Hub and Spoke Architecture

**What it is:** The most common enterprise Azure network pattern. Centralize shared services in a "Hub" VNet, workloads in "Spoke" VNets.

```
                ┌─────────────────────────────┐
                │         HUB VNet             │
                │  ┌──────────┐  ┌──────────┐  │
                │  │ Firewall │  │  Bastion │  │
                │  └──────────┘  └──────────┘  │
                │  ┌──────────┐  ┌──────────┐  │
                │  │ VPN GW   │  │  DNS     │  │
                │  └──────────┘  └──────────┘  │
                └────────┬────────────┬─────────┘
                    Peering         Peering
              ┌─────────┴──┐   ┌────┴───────────┐
              │  Spoke VNet │   │   Spoke VNet   │
              │  (Dev App)  │   │   (Prod App)   │
              └─────────────┘   └────────────────┘
```

---

### Landing Zone

**What it is:** A pre-configured, secure Azure environment that follows best practices. The "foundation" that all workloads are deployed into.

**A Landing Zone includes:**
- Management Groups hierarchy
- Hub VNet + connectivity
- Azure Policy baseline
- RBAC structure
- Logging and monitoring

> **DevOps Gotcha 🚨:** Microsoft provides the **Azure Landing Zone Accelerator** (Terraform modules). Don't build everything from scratch — use it as a starting point and customize. Re-inventing the landing zone is a common time sink.

---

### Shared Services VNet

**What it is:** A VNet (often the Hub or a dedicated spoke) that hosts services used by all teams: DNS, AD DS, NTP, proxy servers.

---

## 8. Azure Compute, Storage & Services

### Virtual Machine (VM)

**What it is:** A cloud server. You pick the OS, size (CPU/RAM), and disk.

```hcl
resource "azurerm_linux_virtual_machine" "web" {
  name                = "web-vm-01"
  resource_group_name = azurerm_resource_group.rg.name
  location            = var.location
  size                = "Standard_D2s_v3"
  admin_username      = "azureuser"

  network_interface_ids = [azurerm_network_interface.web.id]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Premium_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts"
    version   = "latest"
  }

  admin_ssh_key {
    username   = "azureuser"
    public_key = file("~/.ssh/id_rsa.pub")
  }
}
```

> **DevOps Gotcha 🚨:** Avoid `version = "latest"` in production image references — it can cause unexpected OS updates on `apply`. Pin to a specific version: `version = "22.04.202401010"`.

---

### VM Scale Set (VMSS)

**What it is:** A group of identical VMs that can auto-scale based on load. Managed as a single unit.

```
  Load Balancer
       │
  ┌────▼───────────────────────────┐
  │        VM Scale Set             │
  │   ┌───┐  ┌───┐  ┌───┐  ┌───┐  │  ← auto adds/removes VMs
  │   │VM1│  │VM2│  │VM3│  │VM4│  │
  │   └───┘  └───┘  └───┘  └───┘  │
  └─────────────────────────────────┘
```

```hcl
resource "azurerm_linux_virtual_machine_scale_set" "web" {
  name                = "web-vmss"
  instances           = 2
  sku                 = "Standard_D2s_v3"
  ...

  automatic_instance_repair {
    enabled      = true
    grace_period = "PT30M"  # 30 minutes before repair
  }
}
```

> **DevOps Gotcha 🚨:** With VMSS, each VM gets a unique instance ID, not a fixed hostname. Don't design apps that rely on specific VM names/IPs — use Load Balancer frontends and service discovery.

---

### Managed Disk, OS Disk, Data Disk

**Managed Disk:** Azure-managed persistent storage for VMs. Comes in HDD (Standard), SSD (Standard/Premium), Ultra SSD.

**OS Disk:** The boot drive. Contains the operating system.
**Data Disk:** Additional storage for your app data, logs, databases.

```hcl
# Attach a data disk
resource "azurerm_managed_disk" "data" {
  name                 = "vm-data-disk"
  resource_group_name  = azurerm_resource_group.rg.name
  location             = var.location
  storage_account_type = "Premium_LRS"
  create_option        = "Empty"
  disk_size_gb         = 128
}

resource "azurerm_virtual_machine_data_disk_attachment" "data" {
  managed_disk_id    = azurerm_managed_disk.data.id
  virtual_machine_id = azurerm_linux_virtual_machine.web.id
  lun                = 10
  caching            = "ReadWrite"
}
```

> **DevOps Gotcha 🚨:** Increasing a disk's size in Terraform works without downtime. **Decreasing** a disk size is NOT supported — you'd need to destroy and recreate. Plan disk sizes with growth in mind.

---

### Image & Shared Image Gallery

**Image:** A snapshot of a VM's disk used to create new VMs with the same configuration.

**Shared Image Gallery (SIG):** A centralized repository for storing and distributing custom VM images across subscriptions and regions.

```
  Build VM → Run Packer → Create Image → Store in Shared Image Gallery
                                                │
              Dev Sub ◄── replicated ──────────┤
              Prod Sub ◄── replicated ──────────┘
```

```hcl
data "azurerm_shared_image" "golden" {
  name                = "ubuntu-22-04-base"
  gallery_name        = "OrgImageGallery"
  resource_group_name = "images-rg"
}
```

> **DevOps Gotcha 🚨:** Use Packer + Shared Image Gallery to create **golden images** with pre-installed agents, monitoring, and security hardening. Don't rely on `custom_script_extension` to install the same tools on every VM — it's slow and error-prone.

---

### Custom Script Extension

**What it is:** Runs a script on a VM after deployment. Used to install software, configure settings.

```hcl
resource "azurerm_virtual_machine_extension" "script" {
  name                 = "install-nginx"
  virtual_machine_id   = azurerm_linux_virtual_machine.web.id
  publisher            = "Microsoft.Azure.Extensions"
  type                 = "CustomScript"
  type_handler_version = "2.0"

  settings = jsonencode({
    commandToExecute = "apt-get update && apt-get install -y nginx"
  })
}
```

> **DevOps Gotcha 🚨:** Custom Script Extension runs only **once** at creation. If the script fails, Terraform shows the resource as failed but the VM still exists. You can't re-run it without recreating the extension (force replace). Prefer Cloud-init for initial config.

---

### Cloud-init

**What it is:** Industry-standard multi-cloud initialization tool. Runs scripts at first VM boot. Supported natively in Azure Linux VMs.

```hcl
resource "azurerm_linux_virtual_machine" "web" {
  custom_data = base64encode(<<-EOT
    #cloud-config
    package_upgrade: true
    packages:
      - nginx
      - curl
    runcmd:
      - systemctl enable nginx
      - systemctl start nginx
  EOT
  )
  ...
}
```

> **DevOps Gotcha 🚨:** `custom_data` is base64-encoded. If you change `custom_data` in Terraform, it forces VM recreation (it's set at creation time only). This is a breaking change in prod.

---

### Azure Virtual Desktop (AVD)

**What it is:** Desktop-as-a-service. Delivers Windows desktops and apps to users via browser or client.

```
  User → AVD Client → Host Pool → Session Host VMs
                                  (Windows 11 multi-session)
```

**Components:**
- **Host Pool**: Group of session host VMs
- **Session Host**: The VM users connect to
- **App Group**: The apps/desktops published to users
- **Workspace**: Container for app groups visible to users

---

### Host Pool, Session Host, FSLogix

**Host Pool types:**
- **Pooled**: Multiple users share a VM (Windows 11 Multi-session)
- **Personal**: One user per VM (persistent)

**FSLogix:** Manages user profiles for AVD. Stores profiles in Azure Files so they roam between session hosts.

```
  User logs in → FSLogix mounts profile from Azure Files → User sees their desktop
  User logs out → Profile saved back to Azure Files
  User logs in to different session host → same profile loaded
```

> **DevOps Gotcha 🚨:** FSLogix profiles in Azure Files need the correct SMB permissions. Grant `Storage File Data SMB Share Contributor` to AVD users at the **share level** AND configure NTFS permissions on the share. Missing either = profile load failure.

---

### Domain Join, Entra Join, Hybrid Join, Kerberos

**Domain Join (AD DS):** VM joins on-premises Active Directory. Requires line-of-sight to AD DC (via VPN/ExpressRoute or Azure AD DS).

**Entra Join:** VM joins Microsoft Entra ID (cloud-only). No on-prem AD needed. Modern approach for cloud-native.

**Hybrid Join:** Joined to both on-prem AD and Entra ID. Best for hybrid environments.

**Kerberos:** Authentication protocol used by AD. For AVD + FSLogix with Azure Files, you need Kerberos enabled to use Azure AD-based authentication to file shares.

> **DevOps Gotcha 🚨:** Entra-joined AVD session hosts cannot access traditional on-prem file shares (SMB). Use Entra Kerberos to allow Entra-joined VMs to authenticate to Azure Files with Kerberos tickets — this bridges the gap without requiring on-prem AD.

---

### Active Directory Domain Services (AD DS) & Azure AD DS

**AD DS:** Traditional on-premises Active Directory. Domain controllers run on Windows Server VMs.

**Azure AD DS:** Microsoft-managed AD DS in Azure. Get an AD domain without managing DCs yourself. Useful for legacy apps that need LDAP/Kerberos.

```hcl
resource "azurerm_active_directory_domain_service" "aadds" {
  name                = "corp.mycompany.com"
  resource_group_name = azurerm_resource_group.rg.name
  location            = var.location
  domain_name         = "corp.mycompany.com"
  sku                 = "Standard"
  ...
}
```

> **DevOps Gotcha 🚨:** Azure AD DS takes **~30-45 minutes** to provision. It also creates its own dedicated subnet requirements. Plan subnet IP space accordingly.

---

### UPN (User Principal Name)

**What it is:** User's login name in format `user@domain.com`. Used for authentication in Entra ID.

> **DevOps Gotcha 🚨:** In hybrid environments, on-prem UPN suffix must match the verified Entra ID domain for seamless SSO to work. Mismatched UPN suffixes are a common AVD login issue.

---

### Storage Account

**What it is:** Azure's general-purpose storage service. Stores blobs, files, queues, and tables.

```hcl
resource "azurerm_storage_account" "sa" {
  name                     = "myappstg001"   # globally unique, 3-24 chars, lowercase
  resource_group_name      = azurerm_resource_group.rg.name
  location                 = var.location
  account_tier             = "Standard"      # or Premium
  account_replication_type = "LRS"           # or GRS, ZRS, GZRS
  
  min_tls_version                = "TLS1_2"
  allow_nested_items_to_be_public = false    # block anonymous public access
  
  blob_properties {
    versioning_enabled = true
    delete_retention_policy {
      days = 30
    }
  }
}
```

> **DevOps Gotcha 🚨:** Storage account names are **globally unique** across all of Azure. If your name is taken, Terraform fails with a confusing error. Use random suffix: `name = "stg${random_string.suffix.result}"`.

---

### Blob Storage, Container, File Share, Queue, Table

**Blob Storage:** Unstructured object storage (files, images, backups, logs).
```hcl
resource "azurerm_storage_container" "logs" {
  name                  = "app-logs"
  storage_account_name  = azurerm_storage_account.sa.name
  container_access_type = "private"   # never use "blob" or "container" in prod
}
```

**File Share:** SMB file shares (Azure Files). Used for FSLogix, app configs.
**Queue Storage:** Message queue for async processing.
**Table Storage:** NoSQL key-value store. Cheap. For structured data with simple queries.

---

### Access Tier (Hot/Cool/Archive)

| Tier | Access Frequency | Storage Cost | Access Cost |
|------|-----------------|--------------|-------------|
| Hot | Frequent | High | Low |
| Cool | Infrequent (30+ days) | Medium | Medium |
| Archive | Rare (180+ days) | Very Low | High + rehydration time |

> **DevOps Gotcha 🚨:** Archive tier blobs must be **rehydrated** (moved to Hot/Cool) before you can read them. Rehydration can take hours. Don't use Archive for anything you need quickly.

---

### Replication (LRS/GRS/ZRS/GZRS)

| Type | Copies | Protects Against |
|------|--------|-----------------|
| LRS | 3 in one datacenter | Hardware failure |
| ZRS | 3 across zones | Zone failure |
| GRS | 3 local + 3 in paired region | Region failure |
| GZRS | 3 across zones + 3 in paired region | Zone + Region failure |

> **DevOps Gotcha 🚨:** GRS secondary region is **read-only** unless Microsoft declares a regional failover. For read access to secondary, use RA-GRS (`account_replication_type = "RAGRS"`). Terraform state backend = use ZRS or GRS minimum for resilience.

---

### SAS Token & User Delegation SAS

**SAS Token:** Shared Access Signature — a URL with permissions + expiry baked in. Grants limited access without account keys.

```bash
az storage blob generate-sas \
  --account-name mystorageacct \
  --container-name mycontainer \
  --name myfile.txt \
  --permissions r \
  --expiry 2024-12-31
```

**User Delegation SAS:** SAS backed by Entra ID credentials instead of storage account keys. More secure — no account key needed.

> **DevOps Gotcha 🚨:** SAS tokens can't be revoked once issued (unless you rotate storage account keys, which breaks ALL SAS tokens). Use short expiry times. For ongoing access, use Managed Identities + RBAC instead of SAS tokens.

---

### Lifecycle Management (Blob Lifecycle)

**What it is:** Automatically move blobs between access tiers or delete them based on age.

```hcl
resource "azurerm_storage_management_policy" "lifecycle" {
  storage_account_id = azurerm_storage_account.sa.id

  rule {
    name    = "logs-lifecycle"
    enabled = true
    filters {
      blob_types   = ["blockBlob"]
      prefix_match = ["logs/"]
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

---

### Control Plane vs Data Plane

**Control Plane:** Management operations — creating/deleting/configuring resources. Goes through Azure Resource Manager (ARM). Terraform uses the control plane.

**Data Plane:** The actual service operations — reading/writing blobs, secrets, queue messages.

```
  Control Plane:  Terraform creates a Key Vault   → ARM API → OK
  Data Plane:     App reads a secret from vault   → KV API  → OK
  
  (Different permissions! Contributor role ≠ Key Vault Secrets User)
```

> **DevOps Gotcha 🚨:** RBAC `Contributor` grants control plane access (create/delete vault) but NOT data plane access (read secrets). You need separate data plane roles like `Key Vault Secrets User`. This catches many DevOps engineers off-guard.

---

### Container Registry (ACR)

**What it is:** Private Docker registry in Azure. Store and manage your container images.

```hcl
resource "azurerm_container_registry" "acr" {
  name                = "myorgacr"
  resource_group_name = azurerm_resource_group.rg.name
  location            = var.location
  sku                 = "Premium"    # Premium for geo-replication, Private Link
  admin_enabled       = false        # use RBAC instead
}

# Grant AKS pull access
resource "azurerm_role_assignment" "aks_acr_pull" {
  principal_id         = azurerm_kubernetes_cluster.aks.kubelet_identity[0].object_id
  role_definition_name = "AcrPull"
  scope                = azurerm_container_registry.acr.id
}
```

> **DevOps Gotcha 🚨:** `admin_enabled = false` is best practice. Enable admin only for quick testing. In production, use `AcrPull` RBAC role for AKS + `AcrPush` for CI/CD pipelines.

---

### App Service & Function App

**App Service:** Managed PaaS for hosting web apps (.NET, Node, Python, Java, etc.) without managing VMs.

```hcl
resource "azurerm_service_plan" "asp" {
  name                = "my-app-plan"
  resource_group_name = azurerm_resource_group.rg.name
  location            = var.location
  os_type             = "Linux"
  sku_name            = "P1v3"
}

resource "azurerm_linux_web_app" "app" {
  name                = "my-webapp"
  resource_group_name = azurerm_resource_group.rg.name
  location            = var.location
  service_plan_id     = azurerm_service_plan.asp.id

  site_config {
    application_stack {
      node_version = "18-lts"
    }
  }
}
```

**Function App:** Serverless — code runs on demand. Pay per execution. Great for event-driven tasks.

> **DevOps Gotcha 🚨:** App Service slots (staging/prod) allow zero-downtime deployments via slot swap. Always use slots for production deployments — direct deploy to prod slot causes brief downtime during restart.

---

### AKS (Azure Kubernetes Service)

**What it is:** Managed Kubernetes cluster. Azure manages the control plane (free), you manage worker nodes.

```
  AKS Cluster
  ├── Control Plane (managed by Azure, free)
  │   ├── API Server
  │   ├── etcd
  │   └── Scheduler
  └── Node Pools (you pay for VMs)
      ├── System node pool (1+ nodes, for system pods)
      └── User node pool (your workloads)
```

```hcl
resource "azurerm_kubernetes_cluster" "aks" {
  name                = "prod-aks"
  resource_group_name = azurerm_resource_group.rg.name
  location            = var.location
  dns_prefix          = "prodaks"

  default_node_pool {
    name       = "system"
    node_count = 3
    vm_size    = "Standard_D4s_v3"
    zones      = ["1", "2", "3"]   # zone redundant
  }

  identity {
    type = "SystemAssigned"
  }

  network_profile {
    network_plugin = "azure"  # or "kubenet"
    network_policy = "azure"  # or "calico"
  }
}
```

> **DevOps Gotcha 🚨:** AKS cluster upgrades update the control plane first, then node pools (via rolling update). Always test upgrade compatibility in non-prod. Use node pool `max_surge` to control how fast nodes are replaced during upgrade.

---

### Docker

**What it is:** Platform to package apps in containers — portable, isolated environments with all dependencies.

```
  Dockerfile → docker build → Docker Image → docker run → Container
                                    │
                                docker push → ACR/Docker Hub → AKS pulls
```

```dockerfile
# Dockerfile example
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

---

### Kubernetes Core Concepts

**What it is:** Container orchestration — runs, scales, and manages containers across multiple nodes.

```
  Kubernetes Cluster
  ├── Node (VM)
  │   ├── Pod (one or more containers)
  │   │   └── Container (your app)
  │   └── Pod
  └── Node
  
  Services   → stable network endpoint to reach pods
  Deployments→ manage pod replicas, updates
  ConfigMaps → non-secret config
  Secrets    → sensitive config (consider Azure Key Vault integration)
```

> **DevOps Gotcha 🚨:** Kubernetes Secrets are base64-encoded, NOT encrypted by default. Use **Azure Key Vault Provider for Secrets Store CSI Driver** or **External Secrets Operator** to sync secrets from Key Vault to AKS pods securely.

---

### Ingress

**What it is:** HTTP/HTTPS routing to Kubernetes services. Sits at the edge of the cluster.

```
  Internet
     │
  Ingress Controller (NGINX / Azure App Gateway Ingress Controller)
     │
     ├── /api  → backend-service
     └── /web  → frontend-service
```

> **DevOps Gotcha 🚨:** By default, NGINX Ingress on AKS creates an Azure Load Balancer with a public IP. If you need private ingress, add annotation `service.beta.kubernetes.io/azure-load-balancer-internal: "true"`.

---

### Helm

**What it is:** Package manager for Kubernetes. Like `apt` for K8s. Packages are called "charts".

```bash
helm repo add nginx-stable https://helm.nginx.com/stable
helm install my-nginx nginx-stable/nginx-ingress --namespace ingress
```

```hcl
# Use Helm provider in Terraform
resource "helm_release" "nginx_ingress" {
  name       = "nginx-ingress"
  repository = "https://helm.nginx.com/stable"
  chart      = "nginx-ingress"
  namespace  = "ingress-nginx"

  set {
    name  = "controller.replicaCount"
    value = "2"
  }
}
```

> **DevOps Gotcha 🚨:** Terraform's Helm provider keeps state of Helm releases in Terraform state, not Helm's own release history. Mixing `helm` CLI and Terraform Helm provider causes conflicts. Pick one approach and stick with it.

---

## 9. CI/CD, GitOps & Deployment Patterns

### CI/CD

**What it is:** Continuous Integration + Continuous Delivery. Automate building, testing, and deploying infrastructure and apps.

```
  Developer pushes code
         │
  ┌──────▼──────────────────────────┐
  │  CI (Continuous Integration)    │
  │  ├── terraform fmt -check       │
  │  ├── terraform validate         │
  │  ├── terraform plan             │
  │  └── security scan (tfsec)      │
  └──────┬──────────────────────────┘
         │ Human approves plan
  ┌──────▼──────────────────────────┐
  │  CD (Continuous Delivery)       │
  │  └── terraform apply            │
  └─────────────────────────────────┘
```

> **DevOps Gotcha 🚨:** In Terraform CI/CD, always separate **Plan** (CI) and **Apply** (CD) stages with a **human approval gate** between them for production. Fully automated applies in prod without review = potential disasters.

---

### GitOps

**What it is:** Git is the single source of truth for infrastructure state. All changes go through Git (PR → review → merge → auto-apply).

```
  Dev writes Terraform code
        │
        └── Pull Request (PR)
              │
        Code Review + automated checks (fmt, validate, plan)
              │
        Merge to main branch
              │
        GitOps controller detects change
              │
        terraform apply (automated)
              │
        Infrastructure updated
```

**Tools:** ArgoCD (K8s), Flux, Atlantis (Terraform-specific GitOps)

> **DevOps Gotcha 🚨:** GitOps assumes the repo is ALWAYS the truth. If someone makes a manual change in Azure portal, the next GitOps sync will revert it. This is intentional — it enforces discipline. Train your team to never make manual changes.

---

### Pipelines

**What it is:** Automated sequences of steps that build, test, and deploy your code/infrastructure.

**Types:**
- **Build Pipeline**: Compile, test, package
- **Release Pipeline**: Deploy to environments
- **Infrastructure Pipeline**: `terraform plan` + `terraform apply`

---

### YAML Pipelines (Azure DevOps)

```yaml
# azure-pipelines.yml
trigger:
  - main

stages:
  - stage: Plan
    jobs:
      - job: TerraformPlan
        steps:
          - task: TerraformInstaller@1
            inputs:
              terraformVersion: '1.7.0'
          
          - task: TerraformTaskV4@4
            inputs:
              provider: 'azurerm'
              command: 'init'
              backendServiceArm: 'my-azure-service-connection'
              backendAzureRmResourceGroupName: 'tfstate-rg'
              backendAzureRmStorageAccountName: 'tfstateacct001'
              backendAzureRmContainerName: 'tfstate'
              backendAzureRmKey: 'prod.tfstate'
          
          - task: TerraformTaskV4@4
            inputs:
              command: 'plan'
              commandOptions: '-out=tfplan'
              environmentServiceNameAzureRM: 'my-azure-service-connection'

  - stage: Apply
    dependsOn: Plan
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: TerraformApply
        environment: 'production'   # ← manual approval configured here
        strategy:
          runOnce:
            deploy:
              steps:
                - task: TerraformTaskV4@4
                  inputs:
                    command: 'apply'
                    commandOptions: 'tfplan'
```

---

### GitHub Actions

```yaml
# .github/workflows/terraform.yml
name: Terraform CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  id-token: write   # for OIDC
  contents: read
  pull-requests: write

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.7.0"

      - run: terraform init
      - run: terraform fmt -check
      - run: terraform validate
      - run: terraform plan -no-color -out=tfplan
        id: plan

      # Comment plan output on PR
      - uses: actions/github-script@v7
        if: github.event_name == 'pull_request'
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              body: `\`\`\`\n${{ steps.plan.outputs.stdout }}\n\`\`\``
            })

      - run: terraform apply -auto-approve tfplan
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
```

---

### Azure DevOps Pipelines

**What it is:** Microsoft's CI/CD platform, tightly integrated with Azure.

**Key concepts:**
- **Service Connection**: Secure connection from ADO to Azure (SPN or Managed Identity)
- **Environments**: Deploy targets with approval gates
- **Variable Groups**: Shared variables/secrets across pipelines (backed by Key Vault)
- **Artifacts**: Built packages passed between pipeline stages

> **DevOps Gotcha 🚨:** Use **Variable Groups linked to Key Vault** for secrets in ADO pipelines — not hardcoded pipeline variables. Hardcoded variables are visible to anyone with read access to the pipeline.

---

### Pull Requests (PR)

**What it is:** A proposed change to the codebase. Code review happens here before merging to main.

**Terraform PR workflow:**
1. Developer opens PR with Terraform changes
2. CI runs: `fmt`, `validate`, `plan`
3. Plan output is posted as PR comment
4. Reviewer approves the plan + code
5. PR merges → pipeline runs `apply`

> **DevOps Gotcha 🚨:** Use **branch protection rules** to require: at least 1 reviewer, passing CI checks, and no direct pushes to `main`. Without this, someone can bypass the plan review and push broken infra directly.

---

### Environment Promotion

**What it is:** Progressive deployment through environments: Dev → Test → Staging → Production.

```
  Code merge to main
       │
       ▼
  [Dev environment apply]  → automated, instant
       │ passes tests
       ▼
  [Test environment apply] → automated
       │ passes tests
       ▼
  [Staging apply]          → automated
       │ human approval
       ▼
  [Production apply]       → manual approval required
```

> **DevOps Gotcha 🚨:** Use **the same Terraform module code** for all environments — only the `tfvars` differ (sizes, counts, names). If you have different `.tf` files per environment, you'll have drift between environments and bugs that only appear in prod.

---

### Secrets Management in Pipelines

**What it is:** Handling sensitive values (passwords, keys, tokens) safely in CI/CD.

**Options (best to worst):**
1. **OIDC + Managed Identity**: No secrets at all
2. **Azure Key Vault + Pipeline integration**: Secrets fetched at runtime
3. **Pipeline secret variables**: Masked in logs, not in code
4. **Hardcoded in code**: 🚫 NEVER

```yaml
# GitHub Actions: reference secrets
env:
  ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
  ARM_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}  # avoid if possible
  ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
  ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
```

> **DevOps Gotcha 🚨:** Even if you delete a secret from Git history with `git filter-branch`, you must still rotate the secret. Anyone who cloned the repo before deletion could have cached it. Secret exposure = rotate immediately, always.

---

### Environment Variables

**What it is:** Variables set in the pipeline runner's environment. Terraform reads `TF_VAR_*` prefixed env vars automatically.

```bash
export TF_VAR_location="eastus"
export TF_VAR_db_password="super_secret"  # still ends up in state!
terraform apply
```

```yaml
# In GitHub Actions
env:
  TF_VAR_location: "eastus"
```

---

### Tags

**What it is:** Key-value labels on Azure resources. Essential for cost management, automation, and governance.

```hcl
locals {
  common_tags = {
    environment   = var.environment    # dev/test/prod
    project       = var.project_name
    team          = var.team_name
    cost_center   = var.cost_center
    managed_by    = "terraform"
    last_updated  = timestamp()       # Gotcha: causes diff every plan!
  }
}
```

> **DevOps Gotcha 🚨:** Using `timestamp()` in tags causes Terraform to show a diff on EVERY `plan` (because the timestamp is always new). Use a fixed value or omit it. Set `ignore_changes = [tags["last_updated"]]` in lifecycle if you must keep it.

---

### Naming Convention

**What it is:** Standardized naming for all Azure resources. Critical for large teams.

**Common pattern:** `{resource-type}-{project}-{environment}-{region}-{sequence}`

```
  rg-myapp-prod-eus-001       (resource group)
  vnet-myapp-prod-eus-001     (virtual network)
  snet-frontend-prod-eus-001  (subnet)
  vm-web-prod-eus-001         (virtual machine)
  kv-myapp-prod-eus-001       (key vault)
  stg-myapp-prod-eus-001      (storage account — no dashes!)
```

> **DevOps Gotcha 🚨:** Storage account names have unique rules: 3-24 chars, only lowercase letters and numbers, globally unique. No dashes! Plan names carefully before creating — renaming usually requires destroying and recreating.

**Microsoft's official naming reference:** [Azure Naming Convention](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming)

---

### Monitoring, Log Analytics Workspace, Azure Monitor

**Log Analytics Workspace (LAW):** Central store for logs from all Azure resources.

```hcl
resource "azurerm_log_analytics_workspace" "law" {
  name                = "law-myapp-prod"
  resource_group_name = azurerm_resource_group.rg.name
  location            = var.location
  sku                 = "PerGB2018"
  retention_in_days   = 90
}
```

**Azure Monitor:** Collects metrics and logs from all Azure resources. Integrates with LAW.

**Diagnostic Settings:** Route resource logs/metrics to LAW.

```hcl
resource "azurerm_monitor_diagnostic_setting" "kv_diag" {
  name                       = "kv-diagnostics"
  target_resource_id         = azurerm_key_vault.kv.id
  log_analytics_workspace_id = azurerm_log_analytics_workspace.law.id

  enabled_log {
    category = "AuditEvent"
  }

  metric {
    category = "AllMetrics"
  }
}
```

> **DevOps Gotcha 🚨:** Log Analytics cost can surprise you — it's billed per GB ingested + per GB stored beyond retention. Enable diagnostic settings selectively. Don't send ALL logs from ALL resources — filter to what you actually need to query.

---

### Alerts

**What it is:** Notifications when something goes wrong (high CPU, 5xx errors, budget threshold, etc.).

```hcl
resource "azurerm_monitor_metric_alert" "cpu_alert" {
  name                = "high-cpu-alert"
  resource_group_name = azurerm_resource_group.rg.name
  scopes              = [azurerm_linux_virtual_machine.web.id]
  severity            = 2   # 0=Critical, 1=Error, 2=Warning

  criteria {
    metric_namespace = "Microsoft.Compute/virtualMachines"
    metric_name      = "Percentage CPU"
    aggregation      = "Average"
    operator         = "GreaterThan"
    threshold        = 80
  }

  action {
    action_group_id = azurerm_monitor_action_group.ops.id
  }
}
```

---

### Recovery Services Vault, Backup Policy, Site Recovery

**Recovery Services Vault:** Container for backup and disaster recovery configs.

```hcl
resource "azurerm_recovery_services_vault" "vault" {
  name                = "rsv-myapp-prod"
  resource_group_name = azurerm_resource_group.rg.name
  location            = var.location
  sku                 = "Standard"
  soft_delete_enabled = true
}
```

**Backup Policy:** Defines backup schedule and retention.

**Site Recovery (ASR):** Replicates VMs to another region for disaster recovery.

> **DevOps Gotcha 🚨:** Recovery Services Vault with `soft_delete_enabled = true` means even after `terraform destroy`, the vault can't be permanently deleted for 14 days. Plan for this in teardown scripts.

---

### Blue-Green Deployment

**What it is:** Run two identical environments (Blue = current prod, Green = new version). Switch traffic when Green is verified.

```
  Phase 1: Blue serves 100% traffic, Green being deployed
  ├── Blue (v1): ████████████████ 100%
  └── Green (v2): deploying...

  Phase 2: Switch traffic to Green
  ├── Blue (v1): ░░░░░░░░░░░░░░░░  0%
  └── Green (v2): ████████████████ 100%

  Phase 3: Blue becomes the new "old version" (keep for rollback)
```

**In Azure:** Use App Service slots, Azure Traffic Manager, or Azure Front Door weight routing.

---

### Rolling Updates

**What it is:** Update instances one-by-one (or in batches). No extra infrastructure needed.

```
  Step 1: Update VM1, others serve traffic
  Step 2: Update VM2, VM1 + VM3 serve traffic
  Step 3: Update VM3, all updated
```

**In Terraform/VMSS:**
```hcl
resource "azurerm_linux_virtual_machine_scale_set" "web" {
  upgrade_mode = "Rolling"
  rolling_upgrade_policy {
    max_batch_instance_percent              = 20
    max_unhealthy_instance_percent          = 20
    max_unhealthy_upgraded_instance_percent = 5
    pause_time_between_batches              = "PT0S"
  }
}
```

---

### Canary Deployment

**What it is:** Send a small % of traffic to the new version, monitor, then gradually increase.

```
  Traffic split:
  90% → v1 (stable)
  10% → v2 (canary) ← monitor errors/latency
  
  If canary looks good:
  50% → v1
  50% → v2
  
  All green:
  100% → v2
```

**In Azure:** Use Azure Front Door weighted routing or Traffic Manager.

---

### Dev, Test, Production Environments

```
  Dev:  Developer sandboxes. Small sizes, low cost. Break things freely.
         └── auto-apply on merge, no approval
  
  Test: Integration/QA testing. Mirrors prod architecture, smaller size.
         └── auto-apply on merge, smoke tests run
  
  Prod: Real users. Full size, HA config, monitoring, backups.
         └── manual approval required before apply
```

**In Terraform, use separate `tfvars` per environment:**
```
  environments/
  ├── dev.tfvars
  ├── test.tfvars
  └── prod.tfvars
```

---

### High Availability (HA)

**What it is:** System continues operating even when components fail.

```
  HA checklist for Azure:
  ☑ Resources spread across Availability Zones
  ☑ Load Balancer in front of multiple VMs
  ☑ Azure SQL with zone-redundant standby
  ☑ Storage with ZRS or GRS
  ☑ No single point of failure
```

**SLA levels:**
- Single VM: 99.9% (Premium SSD)
- VMs in Availability Set: 99.95%
- VMs in Availability Zones: 99.99%

---

### Scalability

**What it is:** Ability to handle increased load by adding resources.

**Vertical scaling:** Make the VM bigger (change SKU). Requires downtime for VMs.
**Horizontal scaling:** Add more VMs (VMSS auto-scale). No downtime.

```hcl
resource "azurerm_monitor_autoscale_setting" "vmss_scale" {
  name                = "vmss-autoscale"
  target_resource_id  = azurerm_linux_virtual_machine_scale_set.web.id
  ...

  profile {
    capacity {
      default = 2
      minimum = 2
      maximum = 10
    }

    rule {
      metric_trigger {
        metric_name = "Percentage CPU"
        operator    = "GreaterThan"
        threshold   = 75
      }
      scale_action {
        direction = "Increase"
        value     = "1"
        cooldown  = "PT5M"  # 5 minute cooldown
      }
    }
  }
}
```

---

### Fault Tolerance

**What it is:** System continues working even when individual components fail — automatically.

```
  Fault tolerant design:
  ├── If Zone 1 fails → traffic routes to Zone 2 (via Load Balancer)
  ├── If a VM becomes unhealthy → VMSS replaces it automatically
  ├── If a disk fails → Managed Disk replicates across hardware
  └── If a region fails → Geo-replication kicks in (GRS storage, paired region)
```

---

### Cost Optimization

**What it is:** Spend only what you need. Azure makes it easy to accidentally over-provision.

**Key techniques:**
1. **Right-sizing**: Use Azure Advisor recommendations
2. **Reserved Instances**: 1-3 year commitment = up to 72% discount on VMs
3. **Auto-scaling**: Scale down when traffic is low
4. **Spot VMs**: Up to 90% cheaper, but can be evicted (use for batch workloads)
5. **Lifecycle policies**: Automatically move blobs to Cool/Archive
6. **Schedule shutdown**: Dev VMs off on weekends (use Azure Automation)
7. **Tagging**: Tag resources with cost center to track spend

> **DevOps Gotcha 🚨:** The biggest Azure waste is **orphaned resources** — disks with no VM, public IPs with nothing attached, App Service Plans with no apps. Run regular audits: `az resource list -o table | grep <date>`. Terraform helps prevent this since destroy removes everything.

---

### Resource Recreation

**What it is:** When Terraform must destroy and recreate a resource because an argument that "forces new resource" was changed.

```bash
# In plan output:
-/+ resource "azurerm_virtual_machine" "web" {
    ~ name = "vm-old" → "vm-new"  # forces new resource!
```

> **DevOps Gotcha 🚨:** Before changing any argument in prod, check the provider docs for "**Forces new resource**" or "**Immutable**" markers. Common culprits: VM name, OS disk name, storage account name, subnet name. Add `lifecycle { prevent_destroy = true }` as a safeguard.

---

## Bonus Topics for DevOps Beginners

### Atlantis (Terraform GitOps Tool)

**What it is:** Open-source tool that runs Terraform plan/apply automatically via PR comments.

```
  PR opened → Atlantis posts plan as comment
  "atlantis apply" comment → Atlantis runs apply
  PR merged → Atlantis marks PR as deployed
```

---

### tfsec / Checkov / Terrascan

**What it is:** Security scanning tools for Terraform code. Detect misconfigurations before deploy.

```bash
# Scan for security issues
tfsec .
checkov -d .   # or checkov --compact -d .
```

> **DevOps Gotcha 🚨:** Run security scanners in CI. Common findings: storage account with public access, Key Vault without soft delete, NSG allowing all inbound traffic, unencrypted disks.

---

### Terragrunt

**What it is:** Wrapper around Terraform that adds DRY backend configs, dependency management between modules, and remote state helpers.

```hcl
# terragrunt.hcl - backend config once, applies to all modules
remote_state {
  backend = "azurerm"
  config = {
    resource_group_name  = "tfstate-rg"
    storage_account_name = "tfstateacct001"
    container_name       = "tfstate"
    key                  = "${path_relative_to_include()}/terraform.tfstate"
  }
}
```

---

### Infrastructure Testing (Terratest)

**What it is:** Go library to write automated tests for Terraform modules. Applies real infra, runs assertions, destroys.

```go
func TestVNetModule(t *testing.T) {
    opts := &terraform.Options{
        TerraformDir: "./modules/network",
        Vars: map[string]interface{}{
            "vnet_name": "test-vnet",
        },
    }
    defer terraform.Destroy(t, opts)
    terraform.InitAndApply(t, opts)

    vnetID := terraform.Output(t, opts, "vnet_id")
    assert.NotEmpty(t, vnetID)
}
```

---

### Cost Estimation (Infracost)

**What it is:** Shows estimated monthly cost of your Terraform changes before applying.

```bash
infracost breakdown --path .
# Output shows cost per resource and total monthly estimate
```

> **DevOps Gotcha 🚨:** Run Infracost in CI and post cost diff as PR comment. This catches surprise costs before they hit the bill. Especially useful when someone accidentally changes VM SKU from D2s to D32s.

---

*End of Terraform for Azure Notes*

---

> **📌 Quick Reference Commands**
> ```bash
> terraform init              # Setup
> terraform validate          # Check syntax
> terraform fmt -check        # Check formatting
> terraform plan -out=tfplan  # Preview changes
> terraform apply tfplan      # Apply changes
> terraform destroy           # Tear down
> terraform state list        # List managed resources
> terraform state show <res>  # Show resource state
> terraform import <addr> <id># Import existing resource
> terraform output            # Show outputs
> terraform workspace list    # List workspaces
> ```

---

## Appendix A — Azure Resource Abbreviations Cheat Sheet

Use these prefixes for naming conventions:

| Resource | Abbreviation | Example |
|----------|-------------|---------|
| Resource Group | `rg` | `rg-myapp-prod-eus` |
| Virtual Network | `vnet` | `vnet-myapp-prod-eus` |
| Subnet | `snet` | `snet-frontend-prod-eus` |
| Network Security Group | `nsg` | `nsg-frontend-prod-eus` |
| Route Table | `rt` | `rt-backend-prod-eus` |
| Public IP | `pip` | `pip-lb-prod-eus` |
| Load Balancer | `lb` | `lb-myapp-prod-eus` |
| Application Gateway | `agw` | `agw-myapp-prod-eus` |
| Azure Firewall | `afw` | `afw-hub-prod-eus` |
| VPN Gateway | `vpng` | `vpng-hub-prod-eus` |
| Virtual Machine | `vm` | `vm-web-prod-eus-001` |
| VM Scale Set | `vmss` | `vmss-web-prod-eus` |
| Managed Disk | `disk` | `disk-vm-data-prod-001` |
| AKS Cluster | `aks` | `aks-myapp-prod-eus` |
| Container Registry | `cr` | `crmyappprod` (no dashes!) |
| Storage Account | `st` | `stmyappprod001` (no dashes!) |
| Key Vault | `kv` | `kv-myapp-prod-eus` |
| App Service Plan | `asp` | `asp-myapp-prod-eus` |
| App Service | `app` | `app-myapp-prod-eus` |
| Function App | `func` | `func-myapp-prod-eus` |
| Log Analytics Workspace | `law` | `law-myapp-prod-eus` |
| Recovery Services Vault | `rsv` | `rsv-myapp-prod-eus` |
| Service Principal | `sp` | `sp-terraform-prod` |
| Managed Identity | `id` | `id-myapp-prod-eus` |
| Resource Group (tfstate) | — | `rg-tfstate-prod-eus` |

> **Reference:** [Microsoft CAF Abbreviations](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-abbreviations)

---

## Appendix B — Terraform File Structure Reference

### Single Environment Project
```
my-terraform/
├── main.tf           # Core resources
├── variables.tf      # Input variable declarations
├── outputs.tf        # Output declarations
├── providers.tf      # Provider + terraform block
├── locals.tf         # Local values
├── terraform.tfvars  # Variable values (don't commit secrets!)
└── .terraform.lock.hcl  # Provider lock file (COMMIT THIS)
```

### Multi-Environment with Modules
```
my-terraform/
├── environments/
│   ├── dev/
│   │   ├── main.tf         # calls modules
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── providers.tf
│   │   └── dev.tfvars
│   ├── test/
│   │   └── ... (same structure)
│   └── prod/
│       └── ... (same structure)
└── modules/
    ├── networking/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    ├── compute/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── security/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

### Recommended `.gitignore` for Terraform
```gitignore
# Local .terraform directory (provider binaries)
**/.terraform/*

# State files - NEVER commit these
*.tfstate
*.tfstate.*
*.tfstate.backup

# tfvars with secrets
*.auto.tfvars
prod.tfvars
secrets.tfvars

# Crash log
crash.log
crash.*.log

# Saved plan files
*.tfplan
tfplan

# Override files
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# CLI config
.terraformrc
terraform.rc
```

---

## Appendix C — Common Terraform Errors & Fixes

| Error Message | Cause | Fix |
|--------------|-------|-----|
| `Error: features is required` | Missing `features {}` in `provider "azurerm"` | Add empty `features {}` block |
| `Error: state file locked` | Previous apply crashed | `terraform force-unlock <ID>` (verify no active apply first) |
| `Error: 409 Conflict` | Resource name already exists in Azure | Use unique name, check globally unique requirements |
| `Error: 403 Forbidden` | SPN lacks permissions | Assign correct RBAC role at correct scope |
| `Error: can't determine type` | Variable without `type` is misused | Add explicit `type = ...` to variable declaration |
| `Error: Inconsistent values for sensitive` | Sensitive output used in non-sensitive context | Add `sensitive = true` to the output |
| `Error: Reference to undeclared resource` | Typo in resource address | Check `<TYPE>.<NAME>` spelling |
| `Error: No valid credential sources found` | Not authenticated | Run `az login` or set SPN env vars |
| `Error: A resource with the ID already exists` | Importing duplicate | Remove the `import` block, resource already in state |
| `Error: Expected block type` | Missing block, e.g., `os_disk {}` | Add required nested block |

---

## Appendix D — Terraform `azurerm` Provider Auth Methods Compared

| Method | Best For | Pros | Cons |
|--------|---------|------|------|
| **Azure CLI** (`az login`) | Dev laptops | Simple, no setup | Expires, not for CI/CD |
| **SPN + Client Secret** | CI/CD (legacy) | Works everywhere | Secret expires, needs rotation |
| **SPN + Client Certificate** | CI/CD (secure) | No plaintext secret | More complex setup |
| **Managed Identity** | Azure-hosted agents/VMs | No secrets at all | Only works on Azure resources |
| **OIDC / Workload Identity** | GitHub Actions, ADO | No secrets, no expiry | Requires Federated Credential setup |

**Recommended order:** OIDC > Managed Identity > Client Certificate > Client Secret > CLI

---

## Appendix E — AzureRM Provider: Key Resources Quick Lookup

| What you want | Terraform resource |
|--------------|-------------------|
| Resource Group | `azurerm_resource_group` |
| Virtual Network | `azurerm_virtual_network` |
| Subnet | `azurerm_subnet` |
| NSG | `azurerm_network_security_group` |
| NSG + Subnet association | `azurerm_subnet_network_security_group_association` |
| Public IP | `azurerm_public_ip` |
| Linux VM | `azurerm_linux_virtual_machine` |
| Windows VM | `azurerm_windows_virtual_machine` |
| VM Scale Set (Linux) | `azurerm_linux_virtual_machine_scale_set` |
| Managed Disk | `azurerm_managed_disk` |
| Disk Attachment | `azurerm_virtual_machine_data_disk_attachment` |
| AKS Cluster | `azurerm_kubernetes_cluster` |
| AKS Node Pool | `azurerm_kubernetes_cluster_node_pool` |
| Storage Account | `azurerm_storage_account` |
| Blob Container | `azurerm_storage_container` |
| File Share | `azurerm_storage_share` |
| Key Vault | `azurerm_key_vault` |
| Key Vault Secret | `azurerm_key_vault_secret` |
| Container Registry | `azurerm_container_registry` |
| App Service Plan | `azurerm_service_plan` |
| Linux Web App | `azurerm_linux_web_app` |
| Function App | `azurerm_linux_function_app` |
| Load Balancer | `azurerm_lb` |
| App Gateway | `azurerm_application_gateway` |
| Azure Firewall | `azurerm_firewall` |
| VNet Peering | `azurerm_virtual_network_peering` |
| Private Endpoint | `azurerm_private_endpoint` |
| Private DNS Zone | `azurerm_private_dns_zone` |
| Log Analytics Workspace | `azurerm_log_analytics_workspace` |
| Diagnostic Setting | `azurerm_monitor_diagnostic_setting` |
| Alert Rule | `azurerm_monitor_metric_alert` |
| RBAC Role Assignment | `azurerm_role_assignment` |
| Recovery Services Vault | `azurerm_recovery_services_vault` |
| Managed Identity | `azurerm_user_assigned_identity` |
| Policy Assignment | `azurerm_policy_assignment` |
| AVD Host Pool | `azurerm_virtual_desktop_host_pool` |
| Bastion Host | `azurerm_bastion_host` |

---

## Appendix F — Useful `terraform state` Commands

```bash
# List all resources in state
terraform state list

# Show full details of one resource
terraform state show azurerm_resource_group.rg

# Remove resource from state (leaves it in Azure unchanged)
terraform state rm azurerm_resource_group.old_rg

# Move/rename a resource in state
terraform state mv azurerm_resource_group.rg azurerm_resource_group.main

# Pull state as JSON (for inspection)
terraform state pull | jq .

# Push modified state back (use with extreme caution)
terraform state push modified.tfstate

# Show outputs stored in state
terraform output -json
```

---

## Appendix G — DevOps Gotchas Master List

Here's every 🚨 gotcha from these notes in one place:

1. Run `terraform plan` before EVERY `apply` — never skip it
2. Never commit `terraform.tfstate` to Git — use remote backend
3. Always commit `.terraform.lock.hcl` — never commit `.terraform/`
4. Pin provider versions with `~>` — never use bare `>=`
5. `features {}` block is required in `azurerm` provider — even if empty
6. Changing a resource `name` usually forces destroy + recreate
7. State file contains secrets in plaintext — enable backend encryption
8. State locking prevents concurrent applies — use `force-unlock` only when safe
9. Manual Azure portal changes cause drift — enforce "no manual changes" policy
10. `-auto-approve` only for pipelines, never manual prod apply
11. Add `prevent_destroy = true` to prod databases, Key Vaults, state storage
12. `data` source fails if referenced resource doesn't exist yet
13. `terraform import` only adds to state — you must write `.tf` config separately
14. Use `for_each` over `count` for anything important — avoids index renumbering
15. `for_each` values must be known at plan time — can't use computed values
16. `ignore_changes = [all]` stops all updates forever — use specific attributes instead
17. Provisioners break idempotency — prefer Cloud-init or Custom Script Extension
18. `null_resource` replaced by `terraform_data` in TF 1.4+
19. OIDC federated credentials should be scoped to specific repo+branch, not all branches
20. Never use Azure CLI auth in CI/CD pipelines
21. RBAC `Contributor` ≠ data plane access — add separate data plane roles (e.g., `Key Vault Secrets User`)
22. Key Vault: enable `purge_protection_enabled = true` in prod
23. Role assignments propagate in 1-5 minutes — add `depends_on` or `time_sleep` if needed
24. Client secrets expire — prefer Managed Identity or OIDC to avoid rotation
25. Storage account names: globally unique, no dashes, 3-24 chars, lowercase only
26. Bastion subnet must be named exactly `AzureBastionSubnet` with `/26`+
27. VNet peering is NOT transitive — need firewall/NVA for spoke-to-spoke routing
28. VPN Gateway deployment takes 25-40 minutes — don't set short timeouts
29. NAT Gateway and LB outbound rules conflict — use one, not both
30. Private Endpoints require Private DNS Zone + VNet link — don't forget DNS
31. Azure Firewall is expensive (~$1.25/hr) — don't use as NSG replacement
32. DDoS Standard is ~$2,944/month — only enable for public-facing sensitive workloads
33. Azure Policy `Deny` blocks Terraform apply — test in `Audit` first
34. Pin VM image versions in prod — `version = "latest"` causes surprise updates
35. Disk size can increase but NOT decrease in Terraform
36. FSLogix needs both share-level RBAC AND NTFS permissions
37. `timestamp()` in tags causes diff every plan — use static values
38. Storage SAS tokens can't be revoked without rotating account keys
39. Archive tier blobs require rehydration (hours) before read access
40. GRS secondary region is read-only unless Microsoft declares failover — use RA-GRS for read access
41. Log Analytics cost surprises — enable diagnostic settings selectively
42. Recovery Services Vault with soft delete: can't purge for 14 days after destroy
43. VMSS: don't rely on specific VM names/IPs — use LB frontends
44. AKS: upgrade control plane first, then node pools (test compatibility first)
45. Kubernetes Secrets are base64, not encrypted — use Key Vault CSI driver
46. NGINX Ingress creates public IP by default — use annotation for internal LB
47. Mixing Helm CLI and Terraform Helm provider causes conflicts — pick one
48. Workspaces share backend — wrong workspace + destroy = prod wiped
49. `terraform refresh` is legacy — use `terraform plan -refresh-only` instead
50. Secret exposure: always rotate immediately, even if deleted from Git history
51. High parallelism can hit Azure API rate limits (429) — reduce if needed
52. Bootstrap infra (state backend) must be created before `terraform init`
53. Deep module nesting (3+ levels) makes debugging painful
54. Community registry modules have opinions — review before using in prod
55. Sentinel policies are Terraform Cloud/Enterprise only — use OPA for open-source

