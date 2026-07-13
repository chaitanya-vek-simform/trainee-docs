# 🔷 Azure CLI (az) — Complete Command Reference

> All commands grouped by service with usage explanation.
> Run `az login` first. Use `az account set --subscription "<name>"` to switch subscriptions.

---

## 1. Authentication & Account

```bash
# Login interactively (opens browser)
az login

# Login with a service principal (for automation/CI)
az login --service-principal \
  --username <app-id> \
  --password <secret> \
  --tenant <tenant-id>

# Login using managed identity (inside Azure VM/AKS)
az login --identity

# List all subscriptions you have access to
az account list --output table

# Show currently active subscription
az account show

# Switch to a different subscription
az account set --subscription "My Production Sub"
az account set --subscription "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

# Show current signed-in user
az ad signed-in-user show --query userPrincipalName -o tsv

# Logout
az logout
```

---

## 2. Resource Groups

```bash
# List all resource groups
az group list --output table

# List resource groups with specific tag
az group list --tag environment=prod --output table

# Show details of a resource group
az group show --name rg-myapp-prod

# Create a resource group
az group create \
  --name rg-myapp-prod \
  --location eastus \
  --tags environment=prod team=backend

# Check if resource group exists
az group exists --name rg-myapp-prod

# Update tags on a resource group
az group update --name rg-myapp-prod \
  --set tags.cost-center=CC1234

# Export resource group as ARM template
az group export --name rg-myapp-prod > exported-template.json

# Delete resource group (deletes EVERYTHING inside — irreversible)
az group delete --name rg-myapp-dev --yes

# Delete without waiting (async)
az group delete --name rg-myapp-dev --yes --no-wait

# List all resources inside a resource group
az resource list --resource-group rg-myapp-prod --output table

# List resources by type across all groups
az resource list --resource-type Microsoft.Compute/virtualMachines --output table

# List resources by tag
az resource list --tag environment=dev --output table
```

---

## 3. Virtual Machines

```bash
# List all VMs (name, power state, location)
az vm list --output table

# List VMs with power state
az vm list -d --output table

# List only running VMs
az vm list -d --query "[?powerState=='VM running'].name" -o tsv

# List VMs in a resource group
az vm list --resource-group rg-myapp-prod --output table

# Show detailed info about a VM
az vm show --resource-group rg-myapp-prod --name vm-web01

# Create a Linux VM
az vm create \
  --resource-group rg-myapp-prod \
  --name vm-web01 \
  --image Ubuntu2204 \
  --size Standard_B2s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --vnet-name vnet-main \
  --subnet subnet-app \
  --public-ip-sku Standard \
  --tags environment=prod role=web

# Create VM with cloud-init script (bootstrap on first boot)
az vm create \
  --resource-group rg-myapp-prod \
  --name vm-web01 \
  --image Ubuntu2204 \
  --size Standard_B2s \
  --generate-ssh-keys \
  --custom-data ./cloud-init.yaml

# Start a VM
az vm start --resource-group rg-myapp-prod --name vm-web01

# Stop a VM (OS shutdown but still billed for compute)
az vm stop --resource-group rg-myapp-prod --name vm-web01

# DEALLOCATE a VM (stops billing for compute — use this!)
az vm deallocate --resource-group rg-myapp-prod --name vm-web01

# Restart a VM
az vm restart --resource-group rg-myapp-prod --name vm-web01

# Resize a VM (change size/SKU)
az vm resize \
  --resource-group rg-myapp-prod \
  --name vm-web01 \
  --size Standard_B4ms

# List available VM sizes in a region
az vm list-sizes --location eastus --output table

# Run a shell command on VM without SSH (Run Command)
az vm run-command invoke \
  --resource-group rg-myapp-prod \
  --name vm-web01 \
  --command-id RunShellScript \
  --scripts "df -h && free -h"

# Open SSH port 22 in NSG
az vm open-port --resource-group rg-myapp-prod --name vm-web01 --port 22

# Get VM public IP
az vm show -d \
  --resource-group rg-myapp-prod \
  --name vm-web01 \
  --query publicIps -o tsv

# Show VM extensions installed
az vm extension list --resource-group rg-myapp-prod --vm-name vm-web01

# Delete a VM (does NOT delete NIC, disk, public IP by default)
az vm delete --resource-group rg-myapp-prod --name vm-web01 --yes
```

---

## 4. VM Scale Sets (VMSS)

```bash
# List scale sets
az vmss list --output table

# Show scale set details
az vmss show --resource-group rg-myapp-prod --name vmss-web

# List instances in a scale set
az vmss list-instances \
  --resource-group rg-myapp-prod \
  --name vmss-web --output table

# Manually scale (set instance count)
az vmss scale \
  --resource-group rg-myapp-prod \
  --name vmss-web \
  --new-capacity 5

# Update image on all instances
az vmss update-instances \
  --resource-group rg-myapp-prod \
  --name vmss-web \
  --instance-ids '*'

# Delete specific instance
az vmss delete-instances \
  --resource-group rg-myapp-prod \
  --name vmss-web \
  --instance-ids 2
```

---

## 5. Networking — VNet, Subnet, NSG

```bash
# --- Virtual Networks ---
az network vnet list --output table
az network vnet show --resource-group rg-myapp-prod --name vnet-main

az network vnet create \
  --resource-group rg-myapp-prod \
  --name vnet-main \
  --address-prefix 10.0.0.0/16 \
  --location eastus

# List subnets in a VNet
az network vnet subnet list \
  --resource-group rg-myapp-prod \
  --vnet-name vnet-main --output table

# Create a subnet
az network vnet subnet create \
  --resource-group rg-myapp-prod \
  --vnet-name vnet-main \
  --name subnet-app \
  --address-prefix 10.0.1.0/24

# --- Network Security Groups ---
az network nsg list --output table

az network nsg create \
  --resource-group rg-myapp-prod \
  --name nsg-app

# List NSG rules
az network nsg rule list \
  --resource-group rg-myapp-prod \
  --nsg-name nsg-app --output table

# Add inbound allow rule (allow HTTPS from internet)
az network nsg rule create \
  --resource-group rg-myapp-prod \
  --nsg-name nsg-app \
  --name allow-https \
  --priority 100 \
  --protocol Tcp \
  --destination-port-ranges 443 \
  --access Allow \
  --direction Inbound

# Add deny-all rule (block everything else)
az network nsg rule create \
  --resource-group rg-myapp-prod \
  --nsg-name nsg-app \
  --name deny-all \
  --priority 4000 \
  --protocol '*' \
  --destination-port-ranges '*' \
  --access Deny \
  --direction Inbound

# Associate NSG to subnet
az network vnet subnet update \
  --resource-group rg-myapp-prod \
  --vnet-name vnet-main \
  --name subnet-app \
  --network-security-group nsg-app

# --- Public IPs ---
az network public-ip list --output table

az network public-ip create \
  --resource-group rg-myapp-prod \
  --name pip-web01 \
  --sku Standard \
  --allocation-method Static

az network public-ip show \
  --resource-group rg-myapp-prod \
  --name pip-web01 \
  --query ipAddress -o tsv
```

---

## 6. Azure Kubernetes Service (AKS)

```bash
# List AKS clusters
az aks list --output table

# Show details of a cluster
az aks show \
  --resource-group rg-myapp-prod \
  --name aks-main-prod

# Get credentials (update ~/.kube/config to connect with kubectl)
az aks get-credentials \
  --resource-group rg-myapp-prod \
  --name aks-main-prod

# Get credentials (overwrite if already exists)
az aks get-credentials \
  --resource-group rg-myapp-prod \
  --name aks-main-prod --overwrite-existing

# Create AKS cluster
az aks create \
  --resource-group rg-myapp-prod \
  --name aks-main-prod \
  --node-count 3 \
  --node-vm-size Standard_D2s_v3 \
  --enable-cluster-autoscaler \
  --min-count 2 --max-count 10 \
  --network-plugin azure \
  --generate-ssh-keys

# List node pools
az aks nodepool list \
  --resource-group rg-myapp-prod \
  --cluster-name aks-main-prod --output table

# Add a new node pool
az aks nodepool add \
  --resource-group rg-myapp-prod \
  --cluster-name aks-main-prod \
  --name spotpool \
  --node-count 2 \
  --node-vm-size Standard_D2s_v3 \
  --priority Spot \
  --eviction-policy Delete \
  --spot-max-price -1

# Scale a node pool
az aks nodepool scale \
  --resource-group rg-myapp-prod \
  --cluster-name aks-main-prod \
  --name nodepool1 \
  --node-count 5

# Upgrade cluster kubernetes version
az aks get-upgrades \
  --resource-group rg-myapp-prod \
  --cluster-name aks-main-prod --output table

az aks upgrade \
  --resource-group rg-myapp-prod \
  --name aks-main-prod \
  --kubernetes-version 1.30.0

# Attach ACR so AKS can pull images without secrets
az aks update \
  --resource-group rg-myapp-prod \
  --name aks-main-prod \
  --attach-acr myacr

# Enable addons
az aks enable-addons \
  --resource-group rg-myapp-prod \
  --name aks-main-prod \
  --addons monitoring \
  --workspace-resource-id <log-analytics-workspace-id>

# Show cluster credentials / admin access
az aks get-credentials \
  --resource-group rg-myapp-prod \
  --name aks-main-prod \
  --admin

# Stop cluster (saves cost for dev clusters — preserves config)
az aks stop \
  --resource-group rg-myapp-dev \
  --name aks-dev

# Start cluster
az aks start \
  --resource-group rg-myapp-dev \
  --name aks-dev

# Delete cluster
az aks delete \
  --resource-group rg-myapp-dev \
  --name aks-dev --yes
```

---

## 7. Azure Container Registry (ACR)

```bash
# List registries
az acr list --output table

# Create a registry
az acr create \
  --resource-group rg-myapp-prod \
  --name myacr \
  --sku Standard \
  --location eastus

# Login to ACR (sets up docker credentials)
az acr login --name myacr

# List repositories
az acr repository list --name myacr --output table

# List tags for a repository
az acr repository show-tags \
  --name myacr \
  --repository myapp --output table

# Show manifest details for an image
az acr repository show \
  --name myacr \
  --repository myapp --output table

# Build and push image using ACR Tasks (no local Docker needed)
az acr build \
  --registry myacr \
  --image myapp:v1.0 .

# Delete an image tag
az acr repository delete \
  --name myacr \
  --image myapp:old-tag --yes

# Show ACR admin credentials (enable if needed for legacy auth)
az acr credential show --name myacr

# List ACR webhooks
az acr webhook list --registry myacr --output table

# Purge old untagged images (cleanup)
az acr run \
  --registry myacr \
  --cmd "acr purge --filter 'myapp:.*' --untagged --ago 30d" \
  /dev/null
```

---

## 8. Azure Key Vault

```bash
# List key vaults
az keyvault list --output table

# Create a key vault
az keyvault create \
  --name kv-myapp-prod \
  --resource-group rg-myapp-prod \
  --location eastus \
  --enable-purge-protection true \
  --soft-delete-retention-days 90

# Show key vault details
az keyvault show --name kv-myapp-prod

# --- Secrets ---
# List secrets
az keyvault secret list --vault-name kv-myapp-prod --output table

# Add a secret
az keyvault secret set \
  --vault-name kv-myapp-prod \
  --name db-password \
  --value "sup3rs3cr3t"

# Get secret value
az keyvault secret show \
  --vault-name kv-myapp-prod \
  --name db-password \
  --query value -o tsv

# Get specific version of a secret
az keyvault secret show \
  --vault-name kv-myapp-prod \
  --name db-password \
  --version <version-id>

# List secret versions
az keyvault secret list-versions \
  --vault-name kv-myapp-prod \
  --name db-password --output table

# Delete a secret (goes to soft-delete)
az keyvault secret delete \
  --vault-name kv-myapp-prod \
  --name db-password

# List soft-deleted secrets
az keyvault secret list-deleted --vault-name kv-myapp-prod

# Recover soft-deleted secret
az keyvault secret recover \
  --vault-name kv-myapp-prod \
  --name db-password

# Purge permanently (cannot recover after this)
az keyvault secret purge \
  --vault-name kv-myapp-prod \
  --name db-password

# --- Keys ---
az keyvault key list --vault-name kv-myapp-prod
az keyvault key create \
  --vault-name kv-myapp-prod \
  --name myencryptionkey \
  --kty RSA --size 2048

# --- Certificates ---
az keyvault certificate list --vault-name kv-myapp-prod
az keyvault certificate show \
  --vault-name kv-myapp-prod \
  --name myapp-tls

# Grant a managed identity or user access to Key Vault
az keyvault set-policy \
  --name kv-myapp-prod \
  --object-id <managed-identity-object-id> \
  --secret-permissions get list

# Or use RBAC (preferred)
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee <managed-identity-object-id> \
  --scope /subscriptions/<sub-id>/resourceGroups/rg-myapp-prod/providers/Microsoft.KeyVault/vaults/kv-myapp-prod
```

---

## 9. Storage Accounts & Blob

```bash
# List storage accounts
az storage account list --output table

# Create a storage account
az storage account create \
  --name mystorageacct \
  --resource-group rg-myapp-prod \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2 \
  --allow-blob-public-access false \
  --min-tls-version TLS1_2

# Get connection string
az storage account show-connection-string \
  --name mystorageacct \
  --resource-group rg-myapp-prod -o tsv

# Get storage account keys
az storage account keys list \
  --account-name mystorageacct \
  --resource-group rg-myapp-prod

# --- Containers (Blob) ---
az storage container list \
  --account-name mystorageacct --output table

az storage container create \
  --name mycontainer \
  --account-name mystorageacct \
  --public-access off

# Upload a file
az storage blob upload \
  --account-name mystorageacct \
  --container-name mycontainer \
  --name myfile.txt \
  --file ./myfile.txt

# Upload entire directory
az storage blob upload-batch \
  --account-name mystorageacct \
  --destination mycontainer \
  --source ./dist/

# List blobs
az storage blob list \
  --account-name mystorageacct \
  --container-name mycontainer --output table

# Download a blob
az storage blob download \
  --account-name mystorageacct \
  --container-name mycontainer \
  --name myfile.txt \
  --file ./downloaded.txt

# Delete a blob
az storage blob delete \
  --account-name mystorageacct \
  --container-name mycontainer \
  --name myfile.txt

# Generate SAS (temp access URL)
az storage blob generate-sas \
  --account-name mystorageacct \
  --container-name mycontainer \
  --name myfile.txt \
  --permissions r \
  --expiry 2025-12-31T23:59:00Z \
  --https-only -o tsv
```

---

## 10. Azure Monitor & Log Analytics

```bash
# List Log Analytics workspaces
az monitor log-analytics workspace list --output table

# Create workspace
az monitor log-analytics workspace create \
  --resource-group rg-myapp-prod \
  --workspace-name law-myapp-prod \
  --location eastus

# Get workspace ID (needed for AKS monitoring addon)
az monitor log-analytics workspace show \
  --resource-group rg-myapp-prod \
  --workspace-name law-myapp-prod \
  --query id -o tsv

# Run a KQL query against Log Analytics
az monitor log-analytics query \
  --workspace <workspace-id> \
  --analytics-query "AzureActivity | where OperationNameValue contains 'delete' | limit 20"

# --- Metric Alerts ---
az monitor metrics alert list --output table

az monitor metrics alert create \
  --name "High CPU Alert" \
  --resource-group rg-myapp-prod \
  --scopes <vm-resource-id> \
  --condition "avg Percentage CPU > 85" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action <action-group-id>

az monitor metrics alert delete \
  --resource-group rg-myapp-prod \
  --name "High CPU Alert"

# --- Action Groups ---
az monitor action-group list --output table

az monitor action-group create \
  --resource-group rg-myapp-prod \
  --name ag-oncall \
  --short-name oncall \
  --email-receiver name=oncall email=oncall@company.com

# --- Activity Log ---
# Show recent activity log (who did what)
az monitor activity-log list \
  --resource-group rg-myapp-prod \
  --offset 24h \
  --output table

# Show delete operations in last 7 days
az monitor activity-log list \
  --resource-group rg-myapp-prod \
  --offset 7d \
  --query "[?operationName.localizedValue contains 'delete']" \
  --output table

# --- Diagnostic Settings ---
# List diagnostic settings for a resource
az monitor diagnostic-settings list \
  --resource <resource-id>

# Create diagnostic setting (send logs to Log Analytics)
az monitor diagnostic-settings create \
  --resource <resource-id> \
  --workspace <workspace-id> \
  --name send-to-law \
  --logs '[{"category":"AuditEvent","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]'
```

---

## 11. Azure DevOps (az devops)

```bash
# Configure default org and project
az devops configure \
  --defaults organization=https://dev.azure.com/myorg \
  project=myproject

# List projects
az devops project list --output table

# --- Pipelines ---
az pipelines list --output table

# Run a pipeline manually
az pipelines run --name "my-ci-pipeline"

# Show pipeline runs
az pipelines runs list --pipeline-name "my-ci-pipeline" --output table

# Show a specific run result
az pipelines runs show --id 42

# --- Repos ---
az repos list --output table

# Create a PR
az repos pr create \
  --title "Add feature X" \
  --source-branch feature/my-feature \
  --target-branch main \
  --description "This PR adds feature X"

# List PRs
az repos pr list --output table

# --- Artifacts ---
az artifacts universal download \
  --organization https://dev.azure.com/myorg \
  --feed myfeed \
  --name mypackage \
  --version 1.0.0 \
  --path ./download
```

---

## 12. Resource Locks

```bash
# List locks on a resource group
az lock list --resource-group rg-myapp-prod --output table

# Create CanNotDelete lock (cannot delete resource but can modify)
az lock create \
  --name protect-prod \
  --lock-type CanNotDelete \
  --resource-group rg-myapp-prod

# Create ReadOnly lock (no modifications or deletes)
az lock create \
  --name readonly-prod \
  --lock-type ReadOnly \
  --resource-group rg-myapp-prod

# Lock a specific resource (not the whole RG)
az lock create \
  --name protect-db \
  --lock-type CanNotDelete \
  --resource-group rg-myapp-prod \
  --resource-type Microsoft.Sql/servers \
  --resource myapp-sql-server

# Delete a lock
az lock delete \
  --name protect-prod \
  --resource-group rg-myapp-prod
```

---

## 13. Role Assignments (RBAC)

```bash
# List role assignments in a resource group
az role assignment list \
  --resource-group rg-myapp-prod --output table

# Assign a role to a user
az role assignment create \
  --role "Contributor" \
  --assignee user@company.com \
  --resource-group rg-myapp-prod

# Assign role to a managed identity / service principal
az role assignment create \
  --role "Storage Blob Data Contributor" \
  --assignee <object-id> \
  --scope /subscriptions/<sub-id>/resourceGroups/rg-myapp-prod/providers/Microsoft.Storage/storageAccounts/mystorageacct

# List built-in roles
az role definition list --query "[].roleName" -o tsv | sort

# Show what a role can do
az role definition show --name "Contributor"

# Remove a role assignment
az role assignment delete \
  --assignee user@company.com \
  --role "Contributor" \
  --resource-group rg-myapp-prod
```

---

## 14. Azure Policy

```bash
# List policy assignments
az policy assignment list --output table

# List built-in policy definitions
az policy definition list --query "[?policyType=='BuiltIn'].displayName" -o tsv | head -20

# Show a specific policy definition
az policy definition show --name "Require a tag on resource groups"

# Create a policy assignment (enforce a built-in policy)
az policy assignment create \
  --name "require-env-tag" \
  --display-name "Require environment tag" \
  --policy "871b6d14-10aa-478d-b590-94f262ecfa99" \
  --scope /subscriptions/<sub-id>

# Delete a policy assignment
az policy assignment delete --name "require-env-tag"

# List policy compliance state
az policy state list \
  --resource-group rg-myapp-prod \
  --query "[?complianceState=='NonCompliant']" --output table
```

---

## 15. Cost Management

```bash
# Show current month cost by resource group
az consumption usage list \
  --start-date 2025-07-01 \
  --end-date 2025-07-13 \
  --output table

# List budgets
az consumption budget list --output table

# Create a budget alert (alert at 80% of $500/month)
az consumption budget create \
  --budget-name monthly-budget \
  --amount 500 \
  --time-grain Monthly \
  --start-date 2025-07-01 \
  --end-date 2026-07-01 \
  --category Cost \
  --threshold 80 \
  --contact-emails oncall@company.com

# Show Azure Advisor recommendations
az advisor recommendation list --output table

# Show cost recommendations specifically
az advisor recommendation list \
  --category Cost --output table
```

---

## 16. Useful Query Tricks

```bash
# Get all resource names and types in a subscription
az resource list \
  --query "[].{Name:name, Type:type, RG:resourceGroup}" \
  -o table

# Find all VMs not tagged with 'environment'
az vm list \
  --query "[?!tags.environment].{Name:name, RG:resourceGroup}" \
  -o table

# Get all public IPs in subscription
az network public-ip list \
  --query "[].{Name:name, IP:ipAddress, RG:resourceGroup}" \
  -o table

# Count resources by type
az resource list \
  --query "length([?type=='Microsoft.Compute/virtualMachines'])"

# Get managed disk size for all VMs in an RG
az disk list \
  --resource-group rg-myapp-prod \
  --query "[].{Name:name, SizeGB:diskSizeGb, State:diskState}" \
  -o table

# Find all unattached (orphaned) managed disks
az disk list \
  --query "[?diskState=='Unattached'].{Name:name, SizeGB:diskSizeGb, RG:resourceGroup}" \
  -o table

# Set multiple environment variables from Key Vault
export DB_PASS=$(az keyvault secret show --vault-name kv-myapp-prod --name db-password --query value -o tsv)
export API_KEY=$(az keyvault secret show --vault-name kv-myapp-prod --name api-key --query value -o tsv)
```

---

## 17. Quick Reference Card

```
TASK                                  COMMAND
─────────────────────────────────────────────────────────────────────
Who am I logged in as?                az account show
Switch subscription                   az account set --subscription X
Stop billing for a VM                 az vm deallocate -g RG -n VM
Shell into VM without SSH             az vm run-command invoke ...
Pull from ACR in AKS                  az aks update --attach-acr ACR
Get kubeconfig for AKS                az aks get-credentials -g RG -n NAME
Get secret from Key Vault             az keyvault secret show --vault V --name N
List all unattached disks             az disk list --query "[?diskState=='Unattached']"
See who deleted a resource            az monitor activity-log list ...
Lock production RG from deletion      az lock create --lock-type CanNotDelete
List all resources without a tag      az resource list --query "[?!tags.environment]"
Show costs for this month             az consumption usage list --start-date ...
Assign Contributor role to user       az role assignment create --role Contributor ...
Check AKS upgrade options             az aks get-upgrades -g RG -n NAME
Find all public IPs                   az network public-ip list
```
