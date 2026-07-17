# 🪟 Windows & Azure CLI Commands Reference

---

## 1. Windows CMD — File & Directory

```cmd
:: List files (detailed)
dir
dir /a          :: show hidden files
dir /s          :: recursive
dir *.log       :: filter by extension

:: Navigate
cd C:\Users\john
cd ..           :: go up one level
cd /d D:\      :: change drive

:: Create / Delete
mkdir my-folder
mkdir "path\to\nested\folder"
rmdir /s /q my-folder         :: force delete non-empty folder
del file.txt
del /f /q /s *.log            :: force delete all .log recursively

:: Copy / Move
copy source.txt dest.txt
xcopy /s /e /i src\ dst\    :: copy folder recursively
move file.txt C:\NewPath\
robocopy src\ dst\ /E /Z /LOG:robocopy.log   :: reliable copy with logging

:: File content
type file.txt
more file.txt          :: paged view
find "error" file.log  :: search in file
findstr /i /r "error\|warn" *.log   :: regex search
fc file1.txt file2.txt :: compare files
```

---

## 2. Windows CMD — System & Process

```cmd
:: System info
systeminfo
systeminfo | findstr /i "OS Name\|Total Physical"
hostname
whoami
ipconfig
ipconfig /all
ipconfig /flushdns

:: Processes
tasklist
tasklist | findstr "nginx"
taskkill /PID 1234 /F
taskkill /IM nginx.exe /F

:: Services
sc query                      :: list all services
sc query wuauserv             :: specific service status
sc start "service-name"
sc stop "service-name"
net start "service-name"
net stop "service-name"

:: Network
netstat -an                   :: all connections + listening ports
netstat -ano | findstr :8080  :: who is using port 8080
nslookup myapp.example.com
tracert google.com
ping -t 8.8.8.8               :: continuous ping
curl http://localhost:8080/health
```

---

## 3. Windows CMD — Environment Variables

```cmd
:: View
set                           :: all env vars
set PATH                      :: specific var
echo %USERNAME%
echo %COMPUTERNAME%
echo %TEMP%

:: Set (session only)
set MY_VAR=hello
echo %MY_VAR%

:: Set permanently (user level)
setx MY_VAR "hello"

:: Set permanently (system level, needs admin)
setx MY_VAR "hello" /M

:: Add to PATH permanently
setx PATH "%PATH%;C:\my\new\path" /M

:: Delete variable
set MY_VAR=
reg delete HKCU\Environment /v MY_VAR /f   :: permanent delete
```

---

## 4. PowerShell — Essential Commands

```powershell
# Help system
Get-Help Get-Process
Get-Help Get-Process -Examples
Get-Command *service*           # find commands related to service

# File system
Get-ChildItem                   # ls equivalent
Get-ChildItem -Recurse -Filter *.log
Set-Location C:\path           # cd equivalent
New-Item -ItemType Directory -Path "C:\myfolder"
New-Item -ItemType File -Path "file.txt"
Remove-Item -Recurse -Force "folder"
Copy-Item src dest -Recurse
Move-Item src dest

# Content
Get-Content file.txt
Get-Content file.txt | Select-String "error"    # grep equivalent
Get-Content file.txt | Select-Object -Last 20   # tail
Set-Content file.txt "new content"
Add-Content file.txt "append this line"

# Variables
$myVar = "hello"
$env:MY_VAR = "value"           # env var
[Environment]::SetEnvironmentVariable("MY_VAR","value","User")   # permanent

# Objects & piping (PowerShell specialty)
Get-Process | Where-Object { $_.CPU -gt 10 } | Sort-Object CPU -Descending
Get-Service | Where-Object { $_.Status -eq "Running" }
Get-ChildItem *.log | Where-Object { $_.Length -gt 1MB }
```

---

## 5. PowerShell — System Administration

```powershell
# Processes
Get-Process
Get-Process -Name nginx
Stop-Process -Name nginx -Force
Stop-Process -Id 1234 -Force
Start-Process "notepad.exe"

# Services
Get-Service
Get-Service -Name "wuauserv"
Start-Service -Name "MyService"
Stop-Service -Name "MyService"
Restart-Service -Name "MyService"
Set-Service -Name "MyService" -StartupType Automatic

# Network
Test-NetConnection -ComputerName google.com -Port 443
Test-NetConnection -ComputerName 10.0.0.5 -Port 5432  # test DB port
Resolve-DnsName myapp.example.com
Get-NetIPAddress
Get-NetAdapter
netstat -ano | findstr :8080

# Windows Firewall
New-NetFirewallRule -DisplayName "Allow HTTP" -Direction Inbound -Protocol TCP -LocalPort 80 -Action Allow
Remove-NetFirewallRule -DisplayName "Allow HTTP"
Get-NetFirewallRule | Where-Object { $_.Enabled -eq "True" }

# Remote
Enter-PSSession -ComputerName server01 -Credential (Get-Credential)
Invoke-Command -ComputerName server01 -ScriptBlock { Get-Service }
```

---

## 6. PowerShell — File & Text Processing

```powershell
# Search recursively
Get-ChildItem -Recurse | Select-String "ERROR" | Select-Object Filename,LineNumber,Line

# Process logs
Get-Content app.log |
  Where-Object { $_ -match "ERROR" } |
  ForEach-Object { $_ -replace "ERROR","[CRITICAL]" } |
  Out-File errors.log

# CSV / JSON
Import-Csv data.csv | Where-Object { $_.Status -eq "active" }
Get-Process | Select-Object Name,CPU,WorkingSet | Export-Csv procs.csv -NoTypeInformation
$json = Get-Content config.json | ConvertFrom-Json
$json | ConvertTo-Json | Set-Content config.json

# Count lines matching pattern
(Get-Content app.log | Select-String "ERROR").Count

# Find large files
Get-ChildItem -Recurse | Sort-Object Length -Descending | Select-Object -First 10 |
  Select-Object Name,@{N="SizeMB";E={[math]::Round($_.Length/1MB,2)}}

# Archive / Extract
Compress-Archive -Path "C:\logs" -DestinationPath "logs.zip"
Expand-Archive -Path "logs.zip" -DestinationPath "C:\extracted"
```

---

## 7. AZ CLI — Authentication & Context

```bash
# Login
az login                                      # interactive browser login
az login --service-principal -u <appId> -p <secret> --tenant <tenantId>
az login --identity                           # use managed identity (inside Azure)

# Account management
az account list -o table
az account show
az account set --subscription "My Sub Name"
az account set --subscription "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
az account get-access-token                   # get current token

# Tenant & service principal
az ad sp list --display-name "myapp" -o table
az ad sp create-for-rbac --name myapp --role Contributor   --scopes /subscriptions/<sub-id>
az ad sp show --id <appId>
```

---

## 8. AZ CLI — Resource Groups & Resources

```bash
# Resource groups
az group list -o table
az group show --name rg-myapp
az group create --name rg-myapp --location eastus
az group delete --name rg-myapp --yes --no-wait
az group exists --name rg-myapp

# Generic resource operations
az resource list --resource-group rg-myapp -o table
az resource list --resource-group rg-myapp --resource-type Microsoft.Compute/virtualMachines
az resource show --ids <resource-id>
az resource delete --ids <resource-id>
az resource tag --ids <resource-id> --tags env=prod team=backend

# Tags
az tag list --resource-id <resource-id>
az tag update --resource-id <id> --operation merge --tags env=prod
az resource list --tag env=production -o table
```

---

## 9. AZ CLI — Virtual Machines

```bash
# List & status
az vm list -o table
az vm list --resource-group rg-myapp -o table
az vm list -d --query "[].{Name:name,Status:powerState,RG:resourceGroup}" -o table
az vm show -g rg-myapp -n vm-web01

# Power operations
az vm start -g rg-myapp -n vm-web01
az vm stop -g rg-myapp -n vm-web01          # stops billing (deallocate)
az vm deallocate -g rg-myapp -n vm-web01    # explicit deallocate
az vm restart -g rg-myapp -n vm-web01

# Create VM
az vm create   --resource-group rg-myapp   --name vm-web01   --image Ubuntu2204   --size Standard_D2s_v5   --admin-username azureuser   --ssh-key-values ~/.ssh/id_rsa.pub   --vnet-name vnet-myapp   --subnet snet-app   --public-ip-address ""        # no public IP (use Bastion)

# Disk operations
az vm disk attach -g rg-myapp --vm-name vm-web01 --name my-disk --new --size-gb 64
az disk list --resource-group rg-myapp -o table
az disk list --query "[?diskState=='Unattached'].name" -o tsv   # orphaned disks

# VM extensions
az vm extension list --resource-group rg-myapp --vm-name vm-web01 -o table
az vm run-command invoke -g rg-myapp -n vm-web01 --command-id RunShellScript   --scripts "systemctl status nginx"
```

---

## 10. AZ CLI — Networking

```bash
# Virtual Networks
az network vnet list -o table
az network vnet show -g rg-net -n vnet-myapp
az network vnet create -g rg-net -n vnet-myapp --address-prefix 10.1.0.0/16 --location eastus

# Subnets
az network vnet subnet list -g rg-net --vnet-name vnet-myapp -o table
az network vnet subnet show -g rg-net --vnet-name vnet-myapp -n snet-app
az network vnet subnet create -g rg-net --vnet-name vnet-myapp   -n snet-app --address-prefix 10.1.1.0/24

# NSG
az network nsg list -o table
az network nsg show -g rg-net -n nsg-app
az network nsg rule list -g rg-net --nsg-name nsg-app -o table
az network nsg rule create -g rg-net --nsg-name nsg-app   -n allow-http --priority 100 --direction Inbound   --protocol Tcp --destination-port-ranges 80 --access Allow

# Public IPs
az network public-ip list -o table
az network public-ip show -g rg-myapp -n pip-myapp
az network public-ip list --query "[?ipConfiguration==null].name" -o tsv  # unattached

# DNS
az network dns zone list -o table
az network dns record-set a list -g rg-dns -z example.com -o table
az network dns record-set a add-record -g rg-dns -z example.com -n myapp -a 1.2.3.4

# Private DNS
az network private-dns zone list -o table
az network private-dns link vnet show -g rg-net -z privatelink.postgres.database.azure.com -n my-link

# VNet Peering
az network vnet peering list -g rg-net --vnet-name vnet-hub -o table
az network vnet peering create -g rg-net   -n peer-hub-to-spoke --vnet-name vnet-hub   --remote-vnet /subscriptions/.../vnet-spoke   --allow-forwarded-traffic --allow-gateway-transit
```

---

## 11. AZ CLI — App Service

```bash
# List
az webapp list -o table
az webapp list --resource-group rg-myapp --query "[].{Name:name,State:state,URL:defaultHostName}" -o table

# Status
az webapp show -g rg-myapp -n myapp --query "state" -o tsv
az webapp start -g rg-myapp -n myapp
az webapp stop -g rg-myapp -n myapp
az webapp restart -g rg-myapp -n myapp

# Config & settings
az webapp config appsettings list -g rg-myapp -n myapp -o table
az webapp config appsettings set -g rg-myapp -n myapp   --settings MY_VAR=value DB_URL="postgresql://..."
az webapp config appsettings delete -g rg-myapp -n myapp --setting-names MY_VAR

# Slot operations
az webapp deployment slot list -g rg-myapp -n myapp -o table
az webapp deployment slot create -g rg-myapp -n myapp --slot staging
az webapp deployment slot swap -g rg-myapp -n myapp --slot staging --target-slot production
az webapp deployment slot swap -g rg-myapp -n myapp --slot staging --action preview  # preview first

# Logs
az webapp log tail -g rg-myapp -n myapp
az webapp log tail -g rg-myapp -n myapp --slot staging
az webapp log download -g rg-myapp -n myapp --log-file myapp-logs.zip

# Deploy
az webapp deploy -g rg-myapp -n myapp --src-path ./app.zip --type zip

# VNet integration
az webapp vnet-integration list -g rg-myapp -n myapp -o table
az webapp vnet-integration add -g rg-myapp -n myapp   --vnet vnet-myapp --subnet snet-app
```

---

## 12. AZ CLI — AKS

```bash
# Cluster operations
az aks list -o table
az aks show -g rg-myapp -n aks-myapp
az aks get-credentials -g rg-myapp -n aks-myapp       # merge into kubeconfig
az aks get-credentials -g rg-myapp -n aks-myapp --admin
az aks start -g rg-myapp -n aks-myapp
az aks stop -g rg-myapp -n aks-myapp

# Node pools
az aks nodepool list -g rg-myapp --cluster-name aks-myapp -o table
az aks nodepool show -g rg-myapp --cluster-name aks-myapp -n nodepool1
az aks nodepool scale -g rg-myapp --cluster-name aks-myapp -n nodepool1 --node-count 3
az aks nodepool add -g rg-myapp --cluster-name aks-myapp -n userpool   --node-count 2 --node-vm-size Standard_D4s_v5   --mode User --os-type Linux

# Upgrade
az aks get-upgrades -g rg-myapp -n aks-myapp -o table
az aks upgrade -g rg-myapp -n aks-myapp --kubernetes-version 1.29.0

# Credentials & access
az aks browse -g rg-myapp -n aks-myapp   # open dashboard
az aks check-acr -g rg-myapp -n aks-myapp --acr crmyapp   # test ACR access

# Enable addons
az aks enable-addons -g rg-myapp -n aks-myapp -a monitoring --workspace-resource-id <law-id>
az aks enable-addons -g rg-myapp -n aks-myapp -a azure-keyvault-secrets-provider
```

---

## 13. AZ CLI — Key Vault

```bash
# Vault operations
az keyvault list -o table
az keyvault show -n kv-myapp
az keyvault create -n kv-myapp -g rg-myapp --location eastus   --enable-rbac-authorization true --enable-soft-delete true

# Secrets
az keyvault secret list --vault-name kv-myapp -o table
az keyvault secret show --vault-name kv-myapp --name db-password
az keyvault secret show --vault-name kv-myapp --name db-password --query value -o tsv
az keyvault secret set --vault-name kv-myapp --name db-password --value "MySecret123"
az keyvault secret delete --vault-name kv-myapp --name db-password
az keyvault secret purge --vault-name kv-myapp --name db-password   # permanent

# Keys
az keyvault key list --vault-name kv-myapp -o table
az keyvault key create --vault-name kv-myapp -n mykey --kty RSA --size 2048

# Certificates
az keyvault certificate list --vault-name kv-myapp -o table
az keyvault certificate show --vault-name kv-myapp -n mycert

# Access policy (when not using RBAC mode)
az keyvault set-policy -n kv-myapp   --object-id <managed-identity-object-id>   --secret-permissions get list

# RBAC (when using RBAC mode — recommended)
az role assignment create   --assignee <object-id>   --role "Key Vault Secrets User"   --scope /subscriptions/.../resourceGroups/rg-myapp/providers/Microsoft.KeyVault/vaults/kv-myapp
```

---

## 14. AZ CLI — Storage

```bash
# Storage accounts
az storage account list -o table
az storage account show -n stmyapp -g rg-myapp
az storage account create -n stmyapp -g rg-myapp -l eastus --sku Standard_LRS
az storage account keys list -n stmyapp -g rg-myapp -o table
az storage account show-connection-string -n stmyapp -g rg-myapp -o tsv

# Containers (blob)
az storage container list --account-name stmyapp -o table
az storage container create --account-name stmyapp -n mycontainer
az storage container show --account-name stmyapp -n mycontainer

# Blobs
az storage blob list --account-name stmyapp -c mycontainer -o table
az storage blob upload --account-name stmyapp -c mycontainer   --name myfile.zip --file ./myfile.zip
az storage blob download --account-name stmyapp -c mycontainer   --name myfile.zip --file ./downloaded.zip
az storage blob delete --account-name stmyapp -c mycontainer --name myfile.zip

# SAS token (time-limited access)
az storage blob generate-sas   --account-name stmyapp -c mycontainer --name myfile.zip   --permissions r --expiry 2025-08-01T00:00:00Z --full-uri

# Copy between accounts
az storage blob copy start   --destination-container dst-container   --destination-blob myfile.zip   --source-uri "https://src.blob.core.windows.net/src-container/myfile.zip"
```

---

## 15. AZ CLI — Container Registry (ACR)

```bash
# Registry
az acr list -o table
az acr show -n crmyapp
az acr login -n crmyapp                       # docker login to ACR

# Images
az acr repository list -n crmyapp -o table
az acr repository show-tags -n crmyapp --repository myapp -o table
az acr repository delete -n crmyapp --repository myapp --yes

# Build in cloud (no local Docker needed)
az acr build --registry crmyapp --image myapp:v1.0 .
az acr build --registry crmyapp --image myapp:latest . --file Dockerfile.prod

# Webhooks
az acr webhook list -n crmyapp -o table
az acr webhook create -n mywebhook -r crmyapp   --uri https://myapp.example.com/deploy   --actions push

# Replication (Premium only)
az acr replication list -n crmyapp -o table
az acr replication create -n crmyapp --location westeurope
```

---

## 16. AZ CLI — RBAC & Identity

```bash
# Role assignments
az role assignment list --assignee <object-id> -o table
az role assignment list --scope /subscriptions/<sub-id> -o table
az role assignment list --resource-group rg-myapp -o table
az role assignment create   --assignee <object-id-or-email>   --role "Contributor"   --scope /subscriptions/<sub-id>/resourceGroups/rg-myapp
az role assignment delete   --assignee <object-id> --role "Contributor"   --scope /subscriptions/<sub-id>/resourceGroups/rg-myapp

# Role definitions
az role definition list -o table
az role definition list --name "Key Vault Secrets User"
az role definition create --role-definition @role.json

# Managed identities
az identity list -o table
az identity show -g rg-myapp -n id-myapp
az identity create -g rg-myapp -n id-myapp --location eastus

# Users & groups (Entra ID)
az ad user show --id user@company.com
az ad group show --group "backend-team"
az ad group member list --group "backend-team" -o table
```

---

## 17. AZ CLI — Monitoring & Diagnostics

```bash
# Activity log (what happened in the subscription)
az monitor activity-log list --max-events 50 -o table
az monitor activity-log list --resource-group rg-myapp   --start-time 2025-07-01 -o table

# Metrics
az monitor metrics list   --resource <resource-id>   --metric "Percentage CPU"   --start-time 2025-07-16T00:00:00Z   --end-time 2025-07-17T00:00:00Z   --interval PT1H -o table

# Alerts
az monitor alert list -o table
az monitor alert show -n my-alert -g rg-myapp

# Log Analytics
az monitor log-analytics workspace list -o table
az monitor log-analytics workspace show -g rg-myapp -n log-myapp
az monitor log-analytics query   --workspace <workspace-id>   --analytics-query "AzureActivity | where Level == 'Error' | take 20"   -o table

# Diagnostic settings
az monitor diagnostic-settings list --resource <resource-id> -o table
az monitor diagnostic-settings create   --name diag-myapp   --resource <resource-id>   --workspace <log-analytics-workspace-id>   --metrics '[{"category":"AllMetrics","enabled":true}]'   --logs '[{"category":"AppServiceHTTPLogs","enabled":true}]'
```

---

## 18. AZ CLI — Useful Patterns & Tips

```bash
# ── OUTPUT FORMATS ────────────────────────────────────────────
az vm list -o table     # human-readable table
az vm list -o json      # full JSON
az vm list -o tsv       # tab-separated (good for scripting)
az vm list -o jsonc     # JSON with color

# ── JQ-STYLE QUERIES ──────────────────────────────────────────
# --query uses JMESPath syntax
az vm list --query "[].name" -o tsv
az vm list --query "[?location=='eastus'].{Name:name,Size:hardwareProfile.vmSize}" -o table
az storage account list --query "[?kind=='StorageV2'].{Name:name,SKU:sku.name}" -o table

# ── COMMON PATTERNS ───────────────────────────────────────────
# Get just the value (no quotes, no JSON wrapper)
az keyvault secret show --vault-name kv-myapp --name db-pass --query value -o tsv

# Wait for async operations
az vm create ... --no-wait        # submit and continue
az vm wait --created -g rg-myapp -n vm-web01  # wait for created state

# What-if (preview changes before apply)
az deployment group what-if   --resource-group rg-myapp   --template-file main.bicep

# Export resource as ARM template
az group export --name rg-myapp > rg-template.json

# Find resource ID
az resource show -g rg-myapp -n kv-myapp   --resource-type Microsoft.KeyVault/vaults --query id -o tsv

# Bulk tag all resources in RG
az resource list -g rg-myapp --query "[].id" -o tsv |   xargs -I{} az resource tag --ids {} --tags env=prod

# Set default resource group & location (avoid repeating in every command)
az configure --defaults group=rg-myapp location=eastus
az configure --defaults group=      # clear defaults

# Enable verbose output for debugging
az vm list --debug 2>&1 | head -50

# Use --only-show-errors to suppress warnings
az webapp list --only-show-errors -o table
```
