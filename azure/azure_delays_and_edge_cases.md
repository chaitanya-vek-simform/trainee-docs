# ⏱️ Azure — Delays, Timing Behaviours & Edge Cases

> What actually happens under the hood, how long things take,
> and what can go wrong during common Azure operations.

---

## 1. App Service — Slot Swapping (Deep Dive)

### What Swap Actually Does (Step by Step)

```
┌──────────────────────────────────────────────────────────────┐
│  SLOT SWAP SEQUENCE (Microsoft's actual warm-up process)     │
│                                                              │
│  Stage 1: WARM UP SOURCE SLOT (staging)                     │
│  ├── Azure applies PRODUCTION slot settings to staging       │
│  │   (connection strings, app settings marked "slot sticky") │
│  ├── Sends HTTP requests to staging until it responds 200    │
│  │   (uses the app's Health Check path if configured)        │
│  └── Duration: depends on app startup time (30s - 10min)    │
│                                                              │
│  Stage 2: SWAP (atomic, instant)                            │
│  ├── Azure updates the routing rules in the load balancer    │
│  ├── Incoming traffic now goes to the former staging slot    │
│  └── Duration: ~30 seconds (invisible to users)             │
│                                                              │
│  Stage 3: COOL DOWN OLD PRODUCTION (now in staging slot)    │
│  ├── Old production slot is kept alive in staging            │
│  └── Allows instant swap-back if issues found               │
│                                                              │
│  TOTAL TIME: 2 min (simple app) → 20 min (heavy Java app)  │
└──────────────────────────────────────────────────────────────┘
```

### Slot Settings — "Sticky" vs "Swapped"

```
TWO TYPES of app settings:

1. SLOT SETTING (sticky = true):
   ├── Stays with the SLOT, does NOT swap
   ├── Mark: connection strings, slot-specific API keys
   └── Example: ASPNETCORE_ENVIRONMENT = "Production" → sticky to prod slot
               ASPNETCORE_ENVIRONMENT = "Staging" → sticky to staging slot

2. DEPLOYMENT SETTING (sticky = false):
   ├── Travels with the CODE during swap
   ├── Mark: feature flags, app version tags, config that follows the release
   └── Example: APP_VERSION = "v1.2.3" → moves to production after swap

IF YOU FORGET TO MARK DB CONNECTION AS STICKY:
   Staging slot uses staging DB.
   Swap happens → production slot now uses STAGING DB connection string.
   ALL PRODUCTION USERS are hitting staging database.
   This is a CRITICAL DATA CORRUPTION RISK.
```

### Edge Case: Two Databases (Staging DB + Production DB)

```
┌──────────────────────────────────────────────────────────────┐
│  SCENARIO: You have                                          │
│  ├── prod slot → production database (prod-db)              │
│  ├── staging slot → staging database (staging-db)           │
│  └── You run: az webapp deployment slot swap                │
│                                                              │
│  WHAT HAPPENS DEPENDS ON YOUR SETTINGS:                     │
│                                                              │
│  ✅ CORRECT setup (DB connection is SLOT-STICKY):            │
│  Before swap:                                                │
│    prod slot   → DB_URL = postgresql://prod-db/myapp        │
│    staging slot → DB_URL = postgresql://staging-db/myapp    │
│                                                              │
│  After swap:                                                 │
│    prod slot   → DB_URL = postgresql://prod-db/myapp (✅)   │
│    staging slot → DB_URL = postgresql://staging-db/myapp (✅)│
│  Each slot keeps its own DB connection. Code swaps, DB stays.│
│                                                              │
│  ❌ WRONG setup (DB connection is NOT sticky):               │
│  Before swap:                                                │
│    prod slot   → DB_URL = postgresql://prod-db/myapp        │
│    staging slot → DB_URL = postgresql://staging-db/myapp    │
│                                                              │
│  After swap:                                                 │
│    prod slot   → DB_URL = postgresql://staging-db/myapp (❌)│
│    staging slot → DB_URL = postgresql://prod-db/myapp (❌)  │
│  Production now writes to STAGING DB. Data is lost/mixed.  │
└──────────────────────────────────────────────────────────────┘

HOW TO MARK AS SLOT-STICKY (CLI):
az webapp config appsettings set \
  --resource-group rg-myapp \
  --name myapp \
  --slot-settings DB_URL   # marks this setting as slot-sticky

HOW TO VERIFY BEFORE SWAP:
az webapp config appsettings list \
  --resource-group rg-myapp \
  --name myapp \
  --query "[?slotSetting==true].name" -o tsv
```

### Other Slot Swap Edge Cases

```
EDGE CASE 1: Health check path not configured
  Stage 1 warm-up sends requests to "/" only.
  If "/" returns 200 but /api/data is broken → swap completes with broken code.
  FIX: Always configure Health Check path in App Service settings.

EDGE CASE 2: Session affinity during swap
  If ARR Affinity is ON and users have active sessions:
  ├── During swap, some requests go to old prod, some to new code
  ├── Users mid-session may hit different slot = inconsistent state
  └── FIX: Use stateless apps + Redis for sessions (no ARR affinity needed)

EDGE CASE 3: Certificates and custom domains
  Custom domains and SSL certs are NOT slot-specific.
  They apply at the App Service level → both slots share same certificate.
  Fine for TLS, but don't expect slot-level cert isolation.

EDGE CASE 4: Swap during active deployment
  If a deployment is running (GitHub Actions pushing to staging) and
  you swap simultaneously → you may swap an incomplete deployment.
  FIX: Check deployment status before swapping. Use swap with preview.

EDGE CASE 5: Database schema migrations
  Staging has new code + ran migration (new schema).
  Swap happens → production now runs new code against old schema? NO.
  Production DB was never migrated because staging ran migration on staging DB.
  FIX: Run migration on production DB BEFORE swapping code.
  Pattern: migrate prod DB → swap slot → verify.
```

---

## 2. App Service — Start/Stop/Restart Delays

```
OPERATION            │ TYPICAL DURATION   │ WHAT HAPPENS
─────────────────────┼────────────────────┼──────────────────────────────
Start (from stopped) │ 2-5 minutes        │ VM allocation + app startup
Stop                 │ 30-60 seconds      │ Graceful shutdown signal sent
Restart              │ 1-3 minutes        │ Stop + Start sequentially
Scale out (add inst.)│ 3-8 minutes        │ New instance provisioned + warmed
Scale in (remove)    │ 30-60 seconds      │ Instance drained then removed
Deploy (zip deploy)  │ 1-5 minutes        │ Upload + extract + restart
Deploy (container)   │ 3-10 minutes       │ Pull image + swap container
SSL cert bind        │ 30-60 seconds      │ Certificate propagated
Custom domain verify │ 1-10 minutes       │ DNS verification polling
App Setting change   │ 30-60 seconds      │ App restarts (all instances!)

CRITICAL: Changing ANY app setting triggers a restart of all instances.
In a multi-instance App Service, instances restart one at a time (rolling).
During this rolling restart: some instances have old setting, some have new.
This causes inconsistent behavior for 1-3 minutes.

FIX for zero-downtime setting changes:
1. Add new setting alongside old (both exist temporarily)
2. Deploy code that reads new setting (falling back to old)
3. Remove old setting
4. This is the blue-green approach for config changes.
```

---

## 3. AKS — Deployment & Scaling Delays

```
OPERATION                    │ TYPICAL DURATION    │ NOTES
─────────────────────────────┼─────────────────────┼──────────────────────
Pod scheduling               │ 1-5 seconds         │ If node has capacity
Pod startup (small image)    │ 10-30 seconds       │ + image pull if not cached
Pod startup (large image)    │ 2-10 minutes        │ First pull on new node
Container ready (readiness)  │ Until probe passes  │ App-specific warm-up
Node auto-scale (add node)   │ 3-8 minutes         │ VM provision + K8s join
Node scale-in                │ 5-15 minutes        │ Drain + cordon + delete
Cluster upgrade (control)    │ 5-10 minutes        │ API server briefly unreachable
Node pool upgrade (per node) │ 3-8 min per node    │ Rolling, pods rescheduled
AKS cluster creation         │ 8-15 minutes        │
Private cluster DNS           │ 5-15 minutes        │ Private DNS propagation
PV (disk) attach             │ 30-60 seconds       │ Azure Disk attach to node

GOTCHA: Large Docker image (5GB+) on a new node
  First pod on a new node must PULL the full image.
  5GB image on slow node → 10+ min before pod starts.
  FIX: Use image pre-pulling DaemonSets or keep images small (<500MB).

GOTCHA: HPA (autoscaler) has a 3-minute default cooldown.
  Spike hits, HPA triggers scale-out at T+0.
  New pods ready at T+5min (node provision + pull + startup).
  Spike already over. You have spare capacity for 5 minutes then scale back.
  FIX: Set targetCPUUtilizationPercentage lower (50%) to scale earlier.

GOTCHA: PodDisruptionBudget blocks node drain
  If PDB requires minimum 1 replica and only 1 pod exists:
  Node drain BLOCKS indefinitely waiting for pod to be rescheduled.
  FIX: Always run >= 2 replicas in production. Set PDB minAvailable: 1.
```

---

## 4. Azure Functions — Cold Start Delays

```
PLAN           │ COLD START     │ WARM START   │ WHEN IT HAPPENS
───────────────┼────────────────┼──────────────┼──────────────────
Consumption    │ 1-10 seconds   │ <100ms       │ After ~10min idle OR
               │ (up to 30s for │              │ first request on new instance
               │ large runtimes)│              │
Premium (EP1)  │ None           │ <100ms       │ Pre-warmed instances
               │                │              │ always available
Dedicated      │ None (if plan  │ <100ms       │ App always running
               │ always-on)     │              │

COLD START FACTORS (longer):
├── .NET isolated, Java → longer than Node.js, Python
├── Large number of packages → slower module loading
├── VNet integration → extra time for VNet join
└── Key Vault references in settings → extra time to resolve secrets

EDGE CASE: First request in a new region (deployment)
  After deploying a new Function App, the first request often cold-starts
  even on Premium plan because the pre-warmed instance is being provisioned.
  Wait 2-3 minutes after deployment before load testing.

EDGE CASE: Consumption plan + VNet = much longer cold start
  Consumption + VNet integration requires special infrastructure.
  Cold starts can reach 30-40 seconds.
  FIX: Use Premium plan if VNet integration + low latency needed.
```

---

## 5. Azure Key Vault — Secret Propagation Delays

```
OPERATION                     │ DELAY       │ NOTES
──────────────────────────────┼─────────────┼─────────────────────────────
Create/Update a secret        │ Immediate   │ Available via API right away
App Service reads KV reference│ 24 hours    │ Cached in App Service!
  (on setting change)         │ or restart  │ Must restart app to pick up
                              │             │ new secret value
AKS CSI driver refresh        │ 2 minutes   │ Default poll interval
  (secret rotation)           │             │ (configurable: secretsync)
Functions (KV reference)      │ 24 hours    │ Same as App Service
ARM deployment reads KV       │ Immediate   │ No caching at ARM level

CRITICAL GOTCHA: App Service Key Vault References
  You update a secret in Key Vault.
  Your App Service is NOT restarted.
  The app still reads the OLD cached value for up to 24 hours.
  
  FIX: After rotating a secret, restart the App Service.
  Or set: WEBSITE_SKIP_CONTENTSHARE_VALIDATION=1 to reduce caching.
  
  Better: Use versioned secret references:
  @Microsoft.KeyVault(SecretUri=https://kv.vault.azure.net/secrets/db-pass/VERSION)
  With version pinned → you control when new version takes effect.

EDGE CASE: Soft-deleted secret name conflict
  You delete secret "db-password" and try to create a new one with same name.
  Azure returns: "Secret db-password is currently in a deleted state."
  The name is reserved during soft-delete retention (90 days).
  FIX: Purge the deleted secret first OR use a different name.
  az keyvault secret purge --vault-name kv-myapp --name db-password
```

---

## 6. Azure Container Registry — Image Pull Delays

```
OPERATION                    │ DELAY          │ NOTES
─────────────────────────────┼────────────────┼──────────────────────────
Push image (100MB)           │ 30-60 seconds  │ Layer-based, cached layers
Push image (1GB)             │ 3-10 minutes   │ Depends on upload bandwidth
Pull image (first time)      │ 30s - 10 min   │ Depends on image size
Pull image (cached on node)  │ <5 seconds     │ Only downloads new layers
ACR Build (cloud build)      │ 2-15 minutes   │ Depends on Dockerfile stages
Geo-replication sync         │ 1-10 minutes   │ Replication to other region
Webhook trigger (on push)    │ 5-30 seconds   │ After push completes

EDGE CASE: AKS pulls from ACR in different region
  AKS in East US, ACR in West EU (before geo-replication).
  Every pod start = image pull across the Atlantic.
  1GB image = 5+ minutes for first pull on each node.
  FIX: Enable ACR geo-replication to match AKS region.
  az acr replication create --registry myacr --location eastus

EDGE CASE: ACR token expiry during long builds
  ACR access tokens expire after 3 hours.
  If your CI/CD pipeline takes >3 hours (unlikely but possible),
  the docker push at the end may fail with auth error.
  FIX: Re-authenticate at push step, not just at pipeline start.

EDGE CASE: Rate limiting on image pulls
  ACR Basic tier: 1000 pulls/day per repository.
  Large AKS cluster scaling up = 100 nodes × 5 services = 500 pulls at once.
  May hit rate limits → ImagePullBackOff errors.
  FIX: Use ACR Standard/Premium (higher pull rate limits) or image caching.
```

---

## 7. Azure SQL / PostgreSQL — Operation Delays

```
OPERATION                     │ DELAY           │ NOTES
──────────────────────────────┼─────────────────┼──────────────────────────
Flexible Server create        │ 3-8 minutes     │
Read replica create           │ 10-30 minutes   │ Initial sync of data
Failover (manual)             │ 20-60 seconds   │ For HA with standby
Failover (automatic)          │ 60-120 seconds  │ Detection + failover
Point-in-time restore         │ 5-60 minutes    │ Depends on DB size + restore point distance
Scale compute (vCores)        │ 3-10 minutes    │ Requires brief downtime
Scale storage                 │ 1-5 minutes     │ Usually no downtime
Maintenance window            │ Up to 15 min    │ Scheduled patching

CRITICAL GOTCHA: Scaling compute = brief downtime
  Scaling PostgreSQL Flexible from D2s to D4s causes ~60 second downtime.
  Apps get "connection refused" during this window.
  FIX: Schedule during maintenance window.
       Implement connection retry logic in app (exponential backoff).
       Use pgBouncer to absorb connection drops.

EDGE CASE: Long-running queries block maintenance
  Azure defers patching if long-running queries are active (>1 hour).
  Eventually forced maintenance overrides and kills them.
  FIX: Set statement_timeout in PostgreSQL.
       Monitor pg_stat_activity for long-running queries.

EDGE CASE: Read replica lag during failover
  Primary fails, you promote the read replica.
  Replica may be 10-30 seconds behind primary.
  Those last 10-30 seconds of writes are LOST.
  This is the RPO (Recovery Point Objective) of your database.
  FIX: Accept this loss OR use synchronous replication (higher latency).

EDGE CASE: Connection pool exhaustion after failover
  App has 100 open connections to old primary IP.
  Failover happens, new IP assigned.
  Apps still try old IP → connection errors.
  Apps open new connections to new IP → now 200 connections during reconnect.
  FIX: Use pgBouncer or connection pooling. Set connection timeouts.
       Configure max_connections carefully.
```

---

## 8. DNS Propagation Delays

```
OPERATION                        │ DELAY           │ NOTES
─────────────────────────────────┼─────────────────┼───────────────────────
Azure DNS record create/update   │ 30-60 seconds   │ Azure nameservers update fast
External DNS propagation         │ 1-48 hours      │ Depends on TTL of old record
App Service custom domain verify │ 1-10 minutes    │ Azure polls your DNS
Private DNS Zone record          │ 1-5 minutes     │ VNet link must exist first
Private Endpoint DNS record      │ 2-5 minutes     │ Auto-created if configured
CDN/Front Door custom domain     │ 5-30 minutes    │ Certificate also provisioned
cert-manager (K8s) cert issue    │ 1-5 minutes     │ DNS-01 or HTTP-01 challenge

GOTCHA: Low TTL before cutover is a MUST
  If your DNS record has TTL=3600 (1 hour) and you update it,
  old clients cache the old IP for up to 1 hour.
  FIX: Lower TTL to 60 seconds at least 24 hours before migration.
       After migration is stable, raise TTL back to 3600.

GOTCHA: Azure App Service custom domain verification
  You add CNAME to your domain registrar.
  Azure verification sometimes takes 5-10 minutes even after DNS propagates.
  Azure is polling your DNS from their servers — different resolvers.
  Be patient, or use TXT record verification as alternative.

EDGE CASE: Private Endpoint DNS without VNet Link
  Private Endpoint for PostgreSQL created → A record created in Private DNS Zone.
  But VNet Link to your app's VNet is NOT created.
  App tries to resolve "mydb.privatelink.postgres.database.azure.com" → fails.
  App falls back to public IP → either blocked by firewall or exposed publicly.
  FIX: ALWAYS verify VNet Link exists for EVERY spoke VNet that needs PaaS access.
```

---

## 9. Managed Identity & RBAC — Propagation Delays

```
OPERATION                        │ DELAY           │ NOTES
─────────────────────────────────┼─────────────────┼───────────────────────
Create role assignment           │ 2-5 minutes     │ RBAC propagates across regions
Delete role assignment           │ 2-5 minutes     │ Same propagation
Enable system-assigned MI on VM  │ 30-60 seconds   │ SP created in Azure AD
User-assigned MI attach to VM    │ 30-60 seconds   │
Azure AD token lifetime          │ 1 hour          │ Apps cache token for 1hr
Conditional Access policy apply  │ 5-15 minutes    │ Policy cache refresh

CRITICAL GOTCHA: "Permission denied" immediately after role assignment
  You assign "Key Vault Secrets User" to a Managed Identity.
  App immediately tries to read secret → "Forbidden".
  Reason: RBAC changes take 2-5 minutes to propagate globally.
  FIX: Wait 2-5 minutes after assigning roles before testing.
       In pipelines, add a sleep/retry loop after role assignment.

EDGE CASE: Token caching after role removal
  You remove a role from a user.
  User already has a cached token valid for ~50 more minutes.
  They can STILL access the resource for up to 1 hour.
  FIX: For immediate revocation: use Conditional Access with token binding
       or revoke all sessions in Entra ID for that user.

EDGE CASE: Managed Identity unavailable during VM restart
  System-assigned MI token endpoint (169.254.169.254) is only available
  when the VM is fully started and IMDS is ready.
  Delay: 30-60 seconds after VM starts before tokens are issuable.
  FIX: Implement retry logic in app startup for first token acquisition.
```

---

## 10. Azure Policy — Evaluation & Enforcement Delays

```
OPERATION                        │ DELAY           │ NOTES
─────────────────────────────────┼─────────────────┼───────────────────────
Policy assignment takes effect   │ 5-15 minutes    │ After assignment creation
Policy evaluation (new resource) │ Near real-time  │ Evaluated at ARM request time
Policy evaluation (existing)     │ 24 hours        │ Standard scan cycle
On-demand evaluation trigger     │ 5-30 minutes    │ az policy state trigger-scan
DINE remediation task creation   │ 5-15 minutes    │ After non-compliance detected
DINE remediation completion      │ 5-60 minutes    │ Depends on what's deployed

GOTCHA: Policy assigned but not evaluating existing resources
  You assign "Deny public IP" policy to a subscription.
  Existing public IPs are NOT immediately flagged.
  Azure only evaluates EXISTING resources on the 24-hour cycle.
  FIX: Trigger manual scan:
  az policy state trigger-scan --resource-group rg-myapp

GOTCHA: DINE policy deploys to already-compliant resources
  DINE policy: "Deploy diagnostic settings to all VMs".
  VM already has diagnostic settings configured manually.
  Policy may still try to deploy its version → conflict or duplicate.
  FIX: Test DINE policies in audit mode before switching to enforcement.

EDGE CASE: Policy blocks emergency change
  Production is down. You need to create a public IP urgently.
  Policy "Deny public IP creation" blocks you.
  FIX: Temporarily exempt the resource or use the override mechanism.
  az policy exemption create \
    --name "emergency-public-ip" \
    --policy-assignment <assignment-id> \
    --scope /subscriptions/.../resourceGroups/rg-prod \
    --exemption-category Waiver \
    --expires-on "2025-07-17T00:00:00Z"
```

---

## 11. Azure Storage — Consistency & Delay Behaviours

```
OPERATION                        │ DELAY / BEHAVIOUR
─────────────────────────────────┼──────────────────────────────────────────
Blob upload (small, <100MB)      │ Immediate (single PUT)
Blob upload (large, >100MB)      │ Chunked upload; visible only after commit
Blob delete + soft delete        │ Blob "gone" from listing but recoverable X days
GRS replication lag              │ Seconds to minutes (eventual consistency)
GRS failover (Microsoft-managed) │ Hours (Microsoft decides when to failover)
GRS failover (customer-initiated)│ Minutes to initiate, then RPO = replication lag
Lifecycle policy execution       │ Once per day (not real-time)
RBAC change on storage           │ 2-5 minutes (same as all RBAC)
Firewall rule change             │ 30-60 seconds to propagate

CRITICAL GOTCHA: GRS is read-only in secondary region UNLESS failover
  GRS replicates to secondary region.
  You CANNOT read from secondary region during normal operation.
  (Use RA-GRS for read access to secondary — Read-Access GRS)
  During an outage: Microsoft-initiated failover can take HOURS.
  FIX for faster failover: use RA-GRS + customer-initiated account failover
  (you control when to failover, not Microsoft).

EDGE CASE: Lifecycle policy timing
  You create a lifecycle policy: "move to Cool after 30 days".
  Blob uploaded yesterday → lifecycle runs next day's scan → NOT moved yet.
  Policy is evaluated once daily, not immediately on upload.
  Expect up to 24 hours after the age threshold before actual tier change.

EDGE CASE: Blob lease preventing deletion
  A blob has an active lease (taken by another process, e.g., Terraform state lock).
  You try to delete or overwrite it → "LeaseIdMissing" or "LeaseAlreadyPresent".
  FIX: Break the lease (only if you're sure it's orphaned):
  az storage blob lease break --blob-name state.tfstate --container-name tfstate
```

---

## 12. Terraform on Azure — Operation Timing

```
OPERATION                        │ TYPICAL DURATION  │ NOTES
─────────────────────────────────┼───────────────────┼──────────────────────
terraform plan                   │ 30s - 5 minutes   │ API calls to read state
terraform apply (VNet)           │ 1-3 minutes       │
terraform apply (AKS)            │ 8-15 minutes      │ Cluster provision
terraform apply (PostgreSQL)     │ 5-10 minutes      │
terraform apply (Private Endpoint│ 2-5 minutes       │ + DNS record creation
terraform destroy (AKS)          │ 10-20 minutes     │ Node pool deletion order
terraform state lock timeout     │ Default 0 (no TO)  │ Can wait forever!

GOTCHA: Resource created but Terraform shows failed
  AKS cluster creation timed out in Terraform (default provider timeout).
  Cluster is actually being provisioned in Azure (takes 10-15 min).
  Terraform state shows resource as tainted or deleted.
  FIX: Import the resource back: terraform import azurerm_kubernetes_cluster.main <id>
  Or increase provider timeout:
  resource "azurerm_kubernetes_cluster" "main" {
    timeouts { create = "30m" }
  }

GOTCHA: State file lock from failed pipeline
  Pipeline ran terraform apply, crashed mid-way.
  State file in Azure Blob has a lease (lock).
  Next pipeline run: "Error acquiring state lock".
  FIX: az storage blob lease break --blob-name terraform.tfstate \
         --container-name tfstate --account-name stterraformstate

EDGE CASE: RBAC propagation during terraform apply
  Terraform creates Managed Identity, then immediately creates role assignment.
  Then tries to use that identity (e.g., AKS attaching to ACR).
  AKS setup fails because RBAC hasn't propagated yet (needs 2-5 min).
  FIX: Add time_sleep resource after role assignment:
  resource "time_sleep" "wait_rbac" {
    depends_on = [azurerm_role_assignment.aks_acr]
    create_duration = "120s"
  }
```

---

## 13. Gotcha Quick Reference — "Why is this not working yet?"

```
┌──────────────────────────────────────────────────────────────┐
│  SYMPTOM                      │ LIKELY CAUSE + FIX           │
├──────────────────────────────────────────────────────────────┤
│ Role assigned but still 403   │ Wait 2-5 min (RBAC propagation)│
│ Secret updated but app reads  │ Restart app (KV cache: 24hr) │
│   old value                   │                              │
│ DNS record added but not      │ Wait for TTL + propagation   │
│   resolving                   │ Check VNet Link if private   │
│ Private endpoint created but  │ Create Private DNS Zone +    │
│   not reachable               │ VNet Link + A record         │
│ Slot swap took 15 minutes     │ Heavy app startup (normal)   │
│ Production reads staging DB   │ DB connection not slot-sticky│
│ Pod in ImagePullBackOff       │ ACR not attached to AKS, or  │
│                               │ image doesn't exist in ACR   │
│ Pod stuck in Pending          │ No node with enough CPU/mem  │
│                               │ or no node pool capacity     │
│ HPA not scaling               │ metrics-server not installed │
│                               │ or no resource requests set  │
│ Terraform state locked        │ Break blob lease (see §12)   │
│ Function cold start 30s       │ Consumption + VNet = slow    │
│                               │ Upgrade to Premium plan      │
│ AKS node drain stuck          │ PDB blocking, only 1 replica │
│                               │ Run >= 2 replicas            │
│ Policy assigned but not       │ Trigger manual evaluation    │
│   blocking resources          │ az policy state trigger-scan │
│ GRS secondary not readable    │ Use RA-GRS (not just GRS)    │
│ DB connection refused briefly │ After scaling compute,       │
│                               │ implement retry with backoff │
└──────────────────────────────────────────────────────────────┘
```
