# 🔷 Azure — Implicit & Complex Topics: What Happens Under the Hood

> For every Azure resource or feature below, this note explains:
> - What it actually IS (mental model)
> - Every resource/configuration created AUTOMATICALLY
> - What YOU must configure manually
> - Gotchas that trip up beginners

---

## 1. Private Endpoint

### What It Is
A Private Endpoint gives a PaaS service (like Azure SQL, Storage, Key Vault) a **private IP address inside your VNet**. Traffic to that service no longer traverses the public internet.

```
┌─────────────────────────────────────────────────────────────────┐
│  WITHOUT Private Endpoint:                                      │
│                                                                 │
│  Your VM ──── public internet ────► storage.blob.core.windows.net │
│  (traffic exits VNet, goes over Microsoft backbone, but         │
│   still "public" — visible to DLP/firewall/sniffers)           │
│                                                                 │
│  WITH Private Endpoint:                                         │
│                                                                 │
│  Your VM ──── VNet (10.0.1.5) ────► NIC (10.0.2.8) ──► Storage │
│               private traffic          ↑                        │
│                               Private Endpoint NIC              │
│               Storage's public endpoint can now be DISABLED     │
└─────────────────────────────────────────────────────────────────┘
```

### Resources Created AUTOMATICALLY When You Create a Private Endpoint

| Resource | Created Where | Notes |
|----------|--------------|-------|
| **Network Interface Card (NIC)** | Your chosen subnet | Gets a private IP from subnet range. This NIC IS the private endpoint. |
| **Private DNS Zone record** (if you choose auto) | Your Private DNS Zone | Creates an A record: `mystorageacct.blob.core.windows.net → 10.0.2.8` |

### What You Must Configure Manually

```
1. Private DNS Zone (create once per service type per subscription):
   ├── blob:         privatelink.blob.core.windows.net
   ├── Key Vault:    privatelink.vaultcore.azure.net
   ├── SQL Server:   privatelink.database.windows.net
   ├── ACR:          privatelink.azurecr.io
   ├── AKS API:      privatelink.<region>.azmk8s.io
   └── Service Bus:  privatelink.servicebus.windows.net

2. VNet Link — link the Private DNS Zone to your VNet
   (without this, VMs in your VNet cannot resolve the private DNS record)

3. Disable public network access on the PaaS service
   (optional but recommended — otherwise private endpoint is useless
    because public access still works!)

4. NSG rules — by default NSGs don't affect private endpoint NICs
   (you need PrivateEndpointNetworkPolicies enabled on subnet to use NSGs)
```

### Full DNS Resolution Flow

```
VM in VNet queries: mystorageacct.blob.core.windows.net
      │
      ▼ (Azure DNS 168.63.129.16)
CNAME: mystorageacct.blob.core.windows.net
    → mystorageacct.privatelink.blob.core.windows.net
      │
      ▼ (Private DNS Zone: privatelink.blob.core.windows.net)
A record: mystorageacct.privatelink.blob.core.windows.net → 10.0.2.8
      │
      ▼
Traffic goes to private IP (10.0.2.8) inside VNet — never leaves!
```

> **Gotcha:** If you forget the VNet Link on the Private DNS Zone, DNS resolution fails silently — traffic falls back to the public IP. The app connects but via public internet. You'll only notice in network logs.

> **Gotcha:** One Private Endpoint = one NIC = one private IP. If you need a service in multiple VNets, you need multiple private endpoints.

---

## 2. Managed Identity

### What It Is
A Managed Identity is an Azure AD identity automatically managed by Azure for a resource (VM, AKS, App Service, etc.) so it can authenticate to other Azure services **without any password, secret, or credential**.

```
┌─────────────────────────────────────────────────────────────────┐
│  WITHOUT Managed Identity (bad pattern):                        │
│                                                                 │
│  App code → uses CLIENT_SECRET stored in env var               │
│  → rotate secret? Update every deployment. Secret can leak.    │
│                                                                 │
│  WITH Managed Identity (correct pattern):                       │
│                                                                 │
│  App code → GET http://169.254.169.254/metadata/identity/...   │
│           ← Azure returns short-lived token (auto-renewed)     │
│           → Uses token to call Key Vault / Storage / etc.      │
│  No secret ever stored. Rotation is automatic.                 │
└─────────────────────────────────────────────────────────────────┘
```

### System-Assigned vs User-Assigned

| | System-Assigned | User-Assigned |
|---|---|---|
| Lifecycle | Tied to resource (deleted when resource deleted) | Independent resource, can be reused |
| One-to-many | One identity per resource | One identity → multiple resources |
| Use case | Single VM/App needing access | Multiple VMs sharing same permissions |

### Resources Created AUTOMATICALLY

**System-Assigned:**
| What | Where |
|------|-------|
| Azure AD Service Principal | Azure AD tenant (hidden, managed by Azure) |
| Object ID assigned to resource | Visible in Identity blade of the resource |

**User-Assigned:**
| What | Where |
|------|-------|
| A standalone Azure AD Service Principal | Azure AD tenant |
| A new Azure resource (`Microsoft.ManagedIdentity/userAssignedIdentities`) | Your resource group |
| Client ID + Object ID | Available for you to use in role assignments |

### What You Must Configure Manually

```
1. Role assignments — give the identity permission to access other resources:
   az role assignment create \
     --role "Key Vault Secrets User" \
     --assignee <object-id-of-managed-identity> \
     --scope /subscriptions/.../vaults/kv-myapp

2. For User-Assigned: attach it to the resource:
   az vm identity assign \
     --identities /subscriptions/.../userAssignedIdentities/myapp-identity \
     --name myvm --resource-group rg-prod

3. App code must use the IMDS endpoint or SDK:
   # Python (azure-identity SDK)
   from azure.identity import DefaultAzureCredential
   credential = DefaultAzureCredential()  # auto-uses managed identity in Azure
```

> **Gotcha:** Managed Identity only works when the code runs INSIDE Azure (VM, AKS pod, App Service). Running locally → `DefaultAzureCredential` falls back to Azure CLI / env vars.

> **Gotcha:** Deleting a resource with System-Assigned identity deletes the Service Principal in Azure AD. If you recreated the resource, you get a NEW identity with a DIFFERENT Object ID — all role assignments are lost and must be re-created.

---

## 3. VNet Peering

### What It Is
VNet Peering connects two VNets so resources in them can communicate using private IPs as if they were in the same network.

```
┌─────────────────────────────────────────────────────────────────┐
│  VNet A (10.0.0.0/16)       VNet B (10.1.0.0/16)              │
│  ┌────────────────┐          ┌────────────────┐                 │
│  │ VM: 10.0.1.5  │◄────────►│ VM: 10.1.1.8  │                 │
│  └────────────────┘  peering └────────────────┘                 │
│                                                                 │
│  IMPORTANT: Peering is NOT transitive.                         │
│                                                                 │
│  VNet A ──peered──► VNet B ──peered──► VNet C                  │
│  A CANNOT talk to C! Only A↔B and B↔C.                        │
│  To connect A↔C, you need ANOTHER peering or Azure VPN GW      │
└─────────────────────────────────────────────────────────────────┘
```

### Resources Created AUTOMATICALLY

| Resource | Notes |
|----------|-------|
| Peering link on VNet A side | `VNetA/peerings/VNetA-to-VNetB` |
| Peering link on VNet B side | `VNetB/peerings/VNetB-to-VNetA` (must be created separately!) |
| Route table entries | Azure automatically adds routes: VNet A knows 10.1.0.0/16 is reachable via peering |

### What You Must Configure Manually

```
1. BOTH sides of peering must be created:
   az network vnet peering create \
     --name VNetA-to-VNetB \
     --resource-group rg-a \
     --vnet-name VNetA \
     --remote-vnet /subscriptions/.../VNetB \
     --allow-vnet-access

   az network vnet peering create \
     --name VNetB-to-VNetA \
     --resource-group rg-b \
     --vnet-name VNetB \
     --remote-vnet /subscriptions/.../VNetA \
     --allow-vnet-access

2. NSG rules — peering opens the network path but NSGs still apply
   You must add NSG rules to allow the traffic between peered VNets.

3. Non-overlapping address spaces:
   VNet A: 10.0.0.0/16 and VNet B: 10.0.0.0/16 → CANNOT be peered
   Address ranges must be completely different.
```

> **Gotcha:** Peering is not transitive. Hub-spoke topology requires either VPN Gateway (with `UseRemoteGateways`) or Azure Firewall/Route Tables to route traffic between spokes.

> **Gotcha:** You cannot add address space to a VNet that has an existing peering without deleting and recreating the peering.

---

## 4. Azure Kubernetes Service (AKS) — What Gets Created

### Resources Created AUTOMATICALLY When You Create an AKS Cluster

```
┌─────────────────────────────────────────────────────────────────┐
│  YOU CREATE: az aks create --name aks-prod ...                  │
│                                                                 │
│  Azure creates TWO resource groups:                            │
│                                                                 │
│  1. YOUR resource group (rg-myapp):                            │
│     └── AKS cluster resource (Microsoft.ContainerService)      │
│                                                                 │
│  2. NODE resource group (MC_rg-myapp_aks-prod_eastus):         │
│     ├── Virtual Machine Scale Set (node pool VMs)              │
│     ├── Load Balancer (Standard, for Services type=LoadBalancer)│
│     ├── Public IP (for load balancer frontend)                 │
│     ├── NSG (attached to node subnet)                          │
│     ├── Virtual Network (if not using existing VNet)           │
│     ├── Route Table (for Azure CNI / kubenet routing)          │
│     ├── Managed Disks (OS disks for each node)                 │
│     └── Proximity Placement Group (if availability zones used) │
└─────────────────────────────────────────────────────────────────┘
```

### System-Created Azure AD Objects

| Object | Purpose |
|--------|---------|
| Managed Identity (kubelet) | Nodes use this to pull from ACR, write to disks |
| Managed Identity (control plane) | AKS control plane uses this to manage node resources |
| Service Principal (if not using MI) | Legacy — always prefer Managed Identity |

### Inside the Cluster — Automatically Deployed System Pods

```
kube-system namespace:
├── coredns                  → DNS resolution inside cluster
├── coredns-autoscaler       → Scales CoreDNS based on node count
├── azure-cni-networkmonitor → Monitors CNI networking
├── konnectivity-agent       → Tunnel between nodes and API server
├── metrics-server           → CPU/memory metrics (kubectl top)
├── cloud-node-manager       → Marks nodes with Azure zone labels
├── csi-azuredisk-node       → Azure Disk CSI driver (PVC support)
├── csi-azurefile-node       → Azure File CSI driver (PVC support)
└── azure-ip-masq-agent      → IP masquerading for internet access
```

### What You Must Configure Manually

```
1. ACR integration (so nodes can pull your images):
   az aks update --attach-acr myacr --name aks-prod -g rg-myapp

2. Ingress controller (AKS doesn't install one by default):
   helm install ingress-nginx ingress-nginx/ingress-nginx

3. cert-manager (for TLS certificates):
   helm install cert-manager jetstack/cert-manager

4. Monitoring addon (if you want Azure Monitor):
   az aks enable-addons --addons monitoring \
     --workspace-resource-id <law-id> \
     --name aks-prod -g rg-myapp

5. RBAC for your team (kubectl access via kubeconfig is admin by default):
   Create Kubernetes RoleBindings for your team members

6. Network Policies (AKS doesn't restrict pod-to-pod by default):
   Install Calico or use Azure CNI with Network Policy enabled at create time

7. Resource Quotas per namespace — not set by default
```

> **Gotcha:** Never manually modify resources in the `MC_` node resource group. AKS reconciles this group and will revert your changes. If you manually resize a VMSS, AKS will reset it during the next reconciliation.

> **Gotcha:** Deleting the node resource group manually corrupts the AKS cluster permanently. You must delete the cluster from the main resource group.

---

## 5. Azure Load Balancer vs Application Gateway vs Front Door

### The Confusion

All three "balance load" — but they work at completely different layers.

```
┌─────────────────────────────────────────────────────────────────┐
│  OSI Layer  │  Azure Service         │  What It Does            │
│─────────────┼────────────────────────┼──────────────────────────│
│  Layer 4    │  Azure Load Balancer   │  TCP/UDP — IP + Port     │
│  (Transport)│  (ALB)                 │  No HTTP awareness        │
│─────────────┼────────────────────────┼──────────────────────────│
│  Layer 7    │  Application Gateway   │  HTTP/HTTPS — paths,      │
│  (HTTP)     │  (AppGW)               │  headers, cookies, WAF    │
│─────────────┼────────────────────────┼──────────────────────────│
│  Global     │  Azure Front Door      │  Global HTTP, CDN,        │
│  (anycast)  │  (AFD)                 │  geo-routing, DDoS        │
└─────────────┴────────────────────────┴──────────────────────────┘
```

### Application Gateway — Resources Created Automatically

```
When you create an Application Gateway, Azure creates/manages:

Within the AppGW resource:
├── Frontend IP configuration     → Public or Private IP attached
├── Frontend port                 → Port 80 / 443
├── HTTP listener                 → Binds frontend IP + port
├── Backend pool                  → List of backend IPs/FQDNs
├── Backend HTTP settings         → Protocol, port, health probe
├── Request routing rule          → Listener → Backend pool mapping
└── Health probe                  → Pings /health on backends every 30s

Separate Azure Resources:
├── Public IP (Standard SKU)      → The AppGW's internet-facing IP
├── Subnet                        → AppGW needs its OWN dedicated subnet
│   (min /26 = 64 addresses)      → Cannot share subnet with VMs
└── WAF Policy (if WAF SKU)       → OWASP rules, custom rules, exclusions
```

> **Gotcha:** Application Gateway requires a **dedicated subnet**. You cannot put any other resource in the AppGW subnet. The subnet must be large enough: minimum /26 for Standard, /24 recommended for autoscaling.

---

## 6. Azure Service Bus — Implicit Concepts

### Namespace vs Queue vs Topic vs Subscription

```
┌─────────────────────────────────────────────────────────────────┐
│  Service Bus Namespace (the container — billable unit)          │
│  └── Queue (point-to-point: one sender, one receiver)          │
│      └── Message sits here until ONE consumer picks it up      │
│                                                                 │
│  └── Topic (pub/sub: one sender, MANY receivers)               │
│      ├── Subscription A (has its own copy of every message)    │
│      ├── Subscription B (independent copy, independent cursor) │
│      └── Subscription C (can have filter rules)                │
│                                                                 │
│  ANALOGY:                                                       │
│  Queue   = a mailbox (one person gets the letter)              │
│  Topic   = a mailing list (everyone subscribed gets a copy)    │
│  Sub     = your personal mailbox for that mailing list         │
└─────────────────────────────────────────────────────────────────┘
```

### Resources Created Automatically

| When You Create | Azure Also Creates |
|---|---|
| Namespace | Default auth rules (RootManageSharedAccessKey), metric alerts |
| Queue | Dead-letter sub-queue (`$deadletterqueue`) — auto-created |
| Topic | Nothing extra, but each Subscription creates its own internal queue |
| Subscription | Its own dead-letter queue, internal message store |

### Dead Letter Queue — What It Is

```
Every Queue and every Subscription automatically has a dead-letter queue:
queue-name/$DeadLetterQueue

Messages go here automatically when:
├── Max delivery count exceeded (default: 10 attempts)
├── Message TTL expires before being consumed
├── Subscriber explicitly dead-letters the message
└── Filter evaluation fails (for topic subscriptions)

You must READ and process the dead-letter queue separately.
Messages there do NOT disappear — they pile up and cause billing.
Set up monitoring alerts on dead-letter queue depth!
```

---

## 7. Azure Key Vault — Implicit Behaviours

### Access Policies vs RBAC — The Two Models

```
┌─────────────────────────────────────────────────────────────────┐
│  Access Policy Model (legacy):                                  │
│  └── Permissions set ON the Key Vault itself                   │
│  └── Permissions: get, list, set, delete per object type       │
│  └── Assigned to: user, group, or service principal            │
│  └── Problem: no audit trail in Azure Activity Log             │
│                                                                 │
│  RBAC Model (recommended):                                      │
│  └── Uses standard Azure RBAC role assignments                 │
│  └── Built-in roles:                                           │
│      ├── Key Vault Secrets User     → get, list secrets        │
│      ├── Key Vault Secrets Officer  → full CRUD on secrets     │
│      ├── Key Vault Crypto User      → use keys (encrypt/decrypt│
│      ├── Key Vault Administrator    → full control             │
│  └── Can scope to: vault / secret / key / certificate          │
│  └── Full audit trail in Activity Log                          │
└─────────────────────────────────────────────────────────────────┘
```

### Soft Delete and Purge Protection — What Happens Automatically

```
When Key Vault is created (2023+ — soft delete is always ON):

SOFT DELETE (retention period, default 90 days):
├── Deleting a secret/key/cert → moves to "deleted" state
├── It is NOT gone — still exists in deleted state
├── Can be recovered: az keyvault secret recover ...
├── Takes up the same name (cannot create new secret with same name!)
└── After retention period → permanently purged

PURGE PROTECTION (optional — once enabled, CANNOT be disabled):
├── Prevents anyone (including admins) from purging during retention
├── Deleted items MUST wait full retention period
├── Use for compliance: ensures 90-day recovery window always exists
└── GOTCHA: If you delete a vault with purge protection, the name
           is RESERVED for 90 days. You cannot recreate it.

GOTCHA SCENARIO:
You delete kv-myapp-prod.
You immediately try: az keyvault create --name kv-myapp-prod
Result: ERROR — name already taken (by soft-deleted vault)
Fix: az keyvault purge --name kv-myapp-prod
     (only works if purge protection is OFF)
```

### Key Vault Firewall — What It Does

```
Key Vault Firewall settings:
├── "Disabled" → Public access allowed from anywhere
├── "Allow trusted Microsoft services" → Azure services bypass firewall
└── "Selected networks + IPs" → Whitelist specific VNets/IPs

IMPORTANT: "Allow trusted Microsoft services" is CRITICAL.
If you enable firewall and forget this checkbox:
├── Azure Backup fails to access Key Vault
├── Azure DevOps pipelines lose access
├── ARM template deployments that read secrets fail
└── Diagnostic settings fail to write to storage
```

---

## 8. Azure Storage Account — Implicit Behaviours

### Tiers and Automatic Tiering

```
Access Tiers (Blob Storage):
├── Hot    → Frequently accessed data. Higher storage cost, lower access cost.
├── Cool   → Infrequently accessed (30+ days). Lower storage cost, higher access.
├── Cold   → Rarely accessed (90+ days). Even lower storage, higher access.
└── Archive→ Almost never accessed. Lowest storage cost. Read takes HOURS.

Archive → Read Process (NOT instant!):
├── To read an archived blob, you must first "rehydrate" it
├── Standard priority: up to 15 hours
├── High priority: under 1 hour (extra cost)
└── While rehydrating: the blob is INACCESSIBLE

Lifecycle Management Policy (automatically transitions tiers):
If you configure a lifecycle rule:
├── Azure automatically moves blobs based on age/last-access
├── Example: Hot after upload → Cool after 30 days → Archive after 180 days
└── Deletion after 365 days
This runs AUTOMATICALLY — no manual action needed.
```

### Storage Account Endpoints — All Created Automatically

When you create a storage account, Azure creates endpoints for ALL services:
```
https://mystorageacct.blob.core.windows.net    → Blob storage
https://mystorageacct.file.core.windows.net    → File shares (SMB/NFS)
https://mystorageacct.queue.core.windows.net   → Queue storage
https://mystorageacct.table.core.windows.net   → Table storage
https://mystorageacct.dfs.core.windows.net     → Data Lake (if HNS enabled)
https://mystorageacct.web.core.windows.net     → Static website hosting
```

All endpoints exist even if you don't use them. This is why you need a Private Endpoint **per service type** (blob private endpoint does NOT protect the table endpoint).

> **Gotcha:** Enabling "Disable public access" on the storage account blocks ALL public endpoints. If you have a web app using the blob endpoint AND a function using the queue endpoint, both stop working. Plan private endpoints for each endpoint type you use.

---

## 9. Azure Monitor — Full Picture of What's Connected

```
┌─────────────────────────────────────────────────────────────────┐
│  AZURE MONITOR ARCHITECTURE                                     │
│                                                                 │
│  DATA SOURCES                                                   │
│  ├── Azure Resources → Platform Metrics (auto-collected)       │
│  ├── Azure Resources → Resource Logs (need Diagnostic Settings)│
│  ├── VMs → Agent (AMA) → Metrics + Logs                       │
│  ├── AKS → Container Insights → Pod/Node metrics               │
│  ├── Applications → Application Insights → APM traces          │
│  └── Azure Activity Log → Who did what in Azure                │
│                                                                 │
│  STORAGE                                                        │
│  ├── Metrics → Azure Monitor Metrics DB (free, 93 days)        │
│  ├── Logs → Log Analytics Workspace (KQL queryable, costs $)   │
│  └── Metrics → Storage Account / Event Hub (for export)        │
│                                                                 │
│  ACTIONS                                                        │
│  ├── Metric Alert → Action Group → Email/SMS/Slack/Webhook     │
│  ├── Log Alert → Runs KQL query on schedule → Action Group     │
│  └── Smart Detection (Application Insights) → auto anomalies  │
└─────────────────────────────────────────────────────────────────┘
```

### What Is Collected AUTOMATICALLY (No Configuration Needed)

| Resource | Auto-Collected Metrics | Retention |
|---|---|---|
| ALL Azure resources | Basic platform metrics (CPU, requests, errors) | 93 days |
| Azure Activity Log | All control-plane operations (create/delete/update) | 90 days |
| AKS | Node CPU/memory (if monitoring addon enabled) | Per workspace |

### What Requires MANUAL Configuration

```
Resource Logs (diagnostic logs) — NOT automatic:
For each resource, you must create a Diagnostic Setting:
az monitor diagnostic-settings create \
  --resource <resource-id> \
  --workspace <law-id> \
  --name "send-to-law" \
  --logs '[{"category":"AuditEvent","enabled":true}]'

Without this:
├── Key Vault access logs → NOT captured
├── Storage access logs → NOT captured
├── NSG flow logs → NOT captured (needs Network Watcher)
└── SQL audit logs → NOT captured

COMMON OVERSIGHT: Teams set up Azure Monitor but forget diagnostic
settings → they have metrics but no logs → can't investigate incidents.
```

---

## 10. Azure Policy — How Enforcement Works

```
┌─────────────────────────────────────────────────────────────────┐
│  POLICY EFFECTS — IN ORDER OF STRENGTH                         │
│                                                                 │
│  Audit       → Resource is created. Azure logs non-compliance. │
│                No blocking. Good for starting out.             │
│                                                                 │
│  Deny        → Resource creation/update is BLOCKED if it       │
│                violates the policy. Error returned to user.    │
│                                                                 │
│  Append      → Automatically ADDS fields to the resource       │
│                during creation (e.g., force a tag value)       │
│                                                                 │
│  Modify      → Automatically CHANGES fields during create/update│
│                (e.g., set diagnostic settings automatically)   │
│                                                                 │
│  DeployIfNotExists (DINE) → If resource is created WITHOUT     │
│                             a related resource, Azure creates  │
│                             it. Example: VM created without    │
│                             monitoring agent → policy deploys  │
│                             the agent automatically.           │
│                                                                 │
│  AuditIfNotExists → Like DINE but only logs non-compliance,    │
│                     doesn't fix it.                            │
└─────────────────────────────────────────────────────────────────┘
```

### DeployIfNotExists — Magic Behind the Scenes

```
Example: Policy "Deploy Log Analytics agent to VMs"
Effect: DeployIfNotExists

What happens when you create a VM:
1. VM is created (not blocked)
2. Azure Policy evaluates: does this VM have the Log Analytics extension?
3. If NO → Policy creates a Remediation Task automatically
4. Remediation Task deploys the VM extension to the VM
5. VM now has monitoring agent — without the user doing anything

This requires:
├── The policy assignment must have a Managed Identity
├── That identity needs "Contributor" or specific roles on the target scope
└── Remediation tasks can also be run manually for existing resources:
    az policy remediation create --policy-assignment <id> --name fix-existing
```

---

## 11. Azure DevOps Service Connections — What Gets Created

### Service Connection to Azure

When you create an Azure Resource Manager service connection in Azure DevOps:

```
Created in Azure AD:
├── App Registration (Service Principal)
├── Client Secret (or certificate)
└── Name: <OrganizationName>-<ProjectName>-<SubscriptionId>

Created in Azure:
└── Role Assignment: Contributor on the selected scope
    (subscription or resource group)

Created in Azure DevOps:
└── Service Connection record storing the SP credentials
```

> **Gotcha:** The auto-created Service Principal gets **Contributor** on the entire subscription by default. This is too broad. Best practice: use **Workload Identity Federation (OIDC)** service connections instead — no stored secret, federated to Azure AD via OIDC token. No secret rotation needed.

### OIDC Service Connection — How It Works

```
Azure DevOps Pipeline runs
      │ requests token from Azure AD
      ▼
Azure AD validates: "Is this request from the correct
Azure DevOps organization + project + pipeline?"
      │ YES → issues short-lived token
      ▼
Pipeline uses token to authenticate to Azure
No secret is ever stored anywhere.

Setup:
az ad app federated-credential create \
  --id <app-id> \
  --parameters '{
    "name": "ado-pipeline-cred",
    "issuer": "https://vstoken.dev.azure.com/<org-id>",
    "subject": "sc://<org>/<project>/<service-connection-name>",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

---

## 12. Quick Reference — Azure Resources Created Implicitly

| When You Create | Azure Also Creates (Automatic) |
|---|---|
| **Private Endpoint** | NIC with private IP, DNS A record (if auto DNS) |
| **AKS Cluster** | Node RG, VMSS, Load Balancer, Public IP, NSG, Route Table, 2 Managed Identities, system pods |
| **Application Gateway** | Public IP, dedicated subnet requirement |
| **Service Bus Queue/Topic** | Dead-letter queue |
| **Storage Account** | 6 endpoints (blob, file, queue, table, dfs, web) |
| **Key Vault** | Soft-delete always on, RootManageSharedAccessKey |
| **VNet** | Default route table (system routes), default DNS server |
| **VM** | NIC, OS Disk, optionally Public IP + NSG |
| **App Service Plan** | Underlying compute infrastructure (hidden) |
| **Azure SQL Server** | Master database, firewall rules resource |
| **Azure DevOps Service Connection** | App Registration + SP in Azure AD + Role Assignment |
| **Managed Identity (System)** | Service Principal in Azure AD |
| **Policy (DINE effect)** | Remediation Task → dependent resources auto-deployed |
