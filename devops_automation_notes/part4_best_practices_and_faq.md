# ✅ DevOps Automation Notes — Part 4: Best Practices, Gotchas & Production FAQ

> **Series:** Part 4 of 4 | ← [Part 3](./part3_azure_automation.md)

---

## Table of Contents

1. [Best Practices by Domain](#1-best-practices-by-domain)
2. [Universal DevOps Anti-Patterns](#2-universal-devops-anti-patterns)
3. [Production Scenario FAQ — AWS](#3-production-scenario-faq--aws)
4. [Production Scenario FAQ — Azure](#4-production-scenario-faq--azure)
5. [Top Gotchas for Beginners](#5-top-gotchas-for-beginners)
6. [The DevOps Automation Checklist](#6-the-devops-automation-checklist)

---

## 1. Best Practices by Domain

---

### IaC (Terraform) Best Practices

```
┌────────────────────────────────────────────────────────────────┐
│  TERRAFORM BEST PRACTICES                                      │
│                                                                │
│  ✅ DO:                                                        │
│  ├── Always run: terraform fmt → validate → plan → apply      │
│  ├── Store state in remote backend (Azure Blob / S3)          │
│  ├── Pin provider versions with ~> (not >= )                  │
│  ├── Commit .terraform.lock.hcl (never commit .terraform/)    │
│  ├── Use modules for reusable infrastructure patterns         │
│  ├── Tag EVERY resource (env, team, cost-center)             │
│  ├── Use lifecycle { prevent_destroy = true } on prod DBs    │
│  ├── Run terraform plan -out=tfplan in CI, apply tfplan in CD │
│  ├── Scan IaC with tfsec / checkov before apply              │
│  └── Run scheduled drift detection (plan -refresh-only)       │
│                                                                │
│  ❌ DON'T:                                                     │
│  ├── Store secrets in .tfvars files committed to Git          │
│  ├── Use terraform apply -auto-approve manually in prod       │
│  ├── Directly edit tfstate file with a text editor           │
│  ├── Mix multiple environments in one state file             │
│  └── Rename resources without planning for destroy+recreate  │
└────────────────────────────────────────────────────────────────┘
```

---

### CI/CD Pipeline Best Practices

```
┌────────────────────────────────────────────────────────────────┐
│  CI/CD BEST PRACTICES                                          │
│                                                                │
│  ✅ DO:                                                        │
│  ├── Pin dependency versions (requirements.txt, package-lock) │
│  ├── Pin Docker base image versions (never :latest in prod)   │
│  ├── Scan containers with Trivy before pushing to registry   │
│  ├── Use environment-specific secrets (not one shared secret) │
│  ├── Require manual approval before deploying to production   │
│  ├── Tag images with commit SHA, not sequential numbers       │
│  ├── Run smoke tests after every deployment                   │
│  ├── Implement rollback mechanism (helm rollback / revert)    │
│  ├── Use OIDC for cloud authentication (no static keys)       │
│  └── Store pipeline as code: azure-pipelines.yml / .github/  │
│                                                                │
│  ❌ DON'T:                                                     │
│  ├── Store secrets in pipeline YAML files                     │
│  ├── Run the same pipeline agent in dev AND prod              │
│  ├── Skip tests to "deploy faster" (this always backfires)   │
│  ├── Deploy without a rollback plan                          │
│  └── Allow force-push to main/release branches               │
└────────────────────────────────────────────────────────────────┘
```

---

### Kubernetes (EKS/AKS) Best Practices

```
┌────────────────────────────────────────────────────────────────┐
│  KUBERNETES PRODUCTION BEST PRACTICES                          │
│                                                                │
│  Pod Configuration:                                            │
│  ├── Always set resource requests AND limits                  │
│  ├── Set readinessProbe (traffic) + livenessProbe (restart)  │
│  ├── Use startupProbe for slow-starting apps (Java, Python)   │
│  ├── Set terminationGracePeriodSeconds >= app shutdown time   │
│  └── Never use :latest image tag                              │
│                                                                │
│  Deployment Config:                                            │
│  ├── minReplicas >= 2 (for HA, no SPOF)                       │
│  ├── maxUnavailable=0, maxSurge=1 (zero-downtime rolling)    │
│  ├── Set pod disruption budgets (PDB) for critical services   │
│  ├── Use pod anti-affinity to spread across nodes/AZs        │
│  └── Set revisionHistoryLimit for rollback capability         │
│                                                                │
│  Security:                                                     │
│  ├── Use namespaces to isolate workloads                      │
│  ├── Apply RBAC: service accounts with least privilege        │
│  ├── Use Network Policies to restrict pod-to-pod traffic      │
│  ├── Run containers as non-root user                          │
│  └── Use secrets from Key Vault / Secrets Manager (CSI driver)│
└────────────────────────────────────────────────────────────────┘
```

---

### Monitoring & Alerting Best Practices

| Principle | What it Means |
|-----------|-------------|
| **Alert on symptoms, not causes** | Alert "user requests failing" not "CPU high" |
| **Every alert must be actionable** | If no one knows what to do with it, delete the alert |
| **Set up alert fatigue prevention** | Too many alerts = on-call ignores them all |
| **Use composite alerts** | CPU high AND memory high AND errors up = real incident |
| **Test your alerts** | Intentionally trigger them to verify they work |
| **Log structured JSON** | `{"level":"error","message":"DB timeout","trace_id":"abc"}` |
| **Centralize logs** | One Log Analytics Workspace / CloudWatch log group per environment |
| **Set log retention policies** | Prod: 90 days in hot, 1 year in cold. Don't pay for forever |

---

### Secrets Management Best Practices

```
┌────────────────────────────────────────────────────────────────┐
│  SECRETS MANAGEMENT — RULES                                    │
│                                                                │
│  NEVER:                                                        │
│  ├── Commit secrets to Git (even private repos)               │
│  ├── Hardcode secrets in Dockerfiles or docker-compose.yml    │
│  ├── Pass secrets as plain text environment variables in K8s  │
│  ├── Log secrets (even accidentally via debug prints)         │
│  └── Share service accounts between environments             │
│                                                                │
│  ALWAYS:                                                       │
│  ├── Use AWS Secrets Manager / Azure Key Vault                │
│  ├── Use Managed Identity / IRSA (no static credentials)      │
│  ├── Rotate secrets on a schedule (30-90 days)               │
│  ├── Use separate secrets for each environment               │
│  └── Audit who/what accessed secrets (Key Vault audit logs)   │
│                                                                │
│  Secret Hierarchy (most to least preferred):                  │
│  1. Managed Identity (no secret at all) ← best               │
│  2. Key Vault + Managed Identity (secret stored in vault)     │
│  3. CI/CD platform secrets (GitHub Secrets, ADO Library)      │
│  4. Environment variables in deployment (last resort)         │
│  5. Hardcoded in code ← NEVER DO THIS                        │
└────────────────────────────────────────────────────────────────┘
```

---

## 2. Universal DevOps Anti-Patterns

These are mistakes every beginner makes. Learn them early:

| Anti-Pattern | Why It's Wrong | Fix |
|--------------|---------------|-----|
| **ClickOps** | Clicking in portal = no audit trail, not reproducible | Use IaC (Terraform/Bicep) |
| **Snowflake servers** | Each server is unique, can't be replaced | Immutable infra + IaC |
| **Manual deployments** | Human error, no rollback, slow | Automated CI/CD pipelines |
| **:latest Docker tag** | Non-deterministic, different code runs after restart | Pin to commit SHA |
| **No resource limits in K8s** | One noisy app starves the whole node | Set `requests` and `limits` |
| **Secrets in Git** | Instantly compromised if repo is public or leaked | Use secrets manager |
| **No rollback plan** | Deploy fails, can't go back | Helm rollback / feature flags |
| **Alert storms** | 500 alerts at once = oncall ignores all | Group, deduplicate, prioritize |
| **No staging environment** | Test in prod = customers see bugs | Full staging environment |
| **Skipping plan in Terraform** | Blind apply destroys prod | Always run plan, review diffs |

---

## 3. Production Scenario FAQ — AWS

**Q: Our EC2 instances run out of disk. How do we automate disk space management?**

```
A: Three-layer approach:
1. Prevention: Set CloudWatch alarm "disk > 80%" → SNS → Slack/PagerDuty
2. Short-term: Lambda auto-extends EBS volume via modify-volume API
3. Long-term: Check why disk fills — automate log rotation with logrotate
   or move logs to S3 via CloudWatch Agent → no local log accumulation.

Alarm example (Terraform):
resource "aws_cloudwatch_metric_alarm" "disk_high" {
  alarm_name  = "ec2-disk-high"
  metric_name = "disk_used_percent"
  threshold   = 80
  alarm_actions = [aws_sns_topic.alerts.arn]
}
```

---

**Q: We use EKS and a deployment keeps crashing (CrashLoopBackOff). How do we debug systematically?**

```
Step-by-step automation debugging flow:

1. kubectl get pods -n production
   → See pod status: CrashLoopBackOff

2. kubectl describe pod <pod-name> -n production
   → Check Events section: OOMKilled? Image pull error? Config issue?

3. kubectl logs <pod-name> -n production --previous
   → Get logs from the crashed container (--previous = last crash)

4. If OOMKilled: increase memory limit or fix memory leak in app
5. If ImagePullBackOff: check ECR auth, image tag exists
6. If Config error: check ConfigMap/Secret mounted correctly

Prevention automation:
├── Set resource limits (no OOMKill surprises)
├── Set up CloudWatch Container Insights alerts on pod restarts
└── Add liveness probe so K8s can detect stuck (not just crashed) pods
```

---

**Q: Our AWS bill doubled this month. How do we investigate with automation?**

```
Automated cost investigation:
1. AWS Cost Explorer → "Group by Service" → find which service grew
2. "Group by Tag" → which team/app is spending more
3. Cost Anomaly Detection → was there a sudden spike?

Automation to prevent next time:
├── AWS Budgets: alert at 80% and 100% of monthly budget
├── Cost Anomaly Detection: ML alerts unusual spending
├── Tag enforcement via AWS Config (no tags = flag for review)
└── Weekly Lambda: email "top 5 most expensive resources" to team

Common culprits:
├── EC2: forgot to delete dev instances
├── NAT Gateway: high data transfer through NAT (optimize with VPC endpoints)
├── RDS: multi-AZ enabled where not needed
└── CloudWatch Logs: no retention policy = infinite log accumulation
```

---

**Q: We need zero-downtime deployments to ECS. How to implement?**

```
ECS Blue-Green Deployment (automated via CodeDeploy):

Blue (current):  Task v1 v1 v1  ← 100% traffic
                                  │
                                  ▼ CodeDeploy triggered
Green (new):     Task v2 v2 v2  ← 0% traffic initially
                                  │
                                  ▼ Health checks pass
Traffic shift:   v1: 90%, v2: 10%  ← canary phase (5 min)
                 v1: 0%, v2: 100%  ← complete shift
                                  │
                                  ▼ 1 hour bake time
Old blue deleted: v1 v1 v1 ← terminated

Rollback: CodeDeploy re-routes to old task set instantly
```

---

**Q: RDS database is slow during peak hours. What can we automate?**

```
Performance automation checklist:
├── Enable Performance Insights (built-in, 7 days free)
│   → See wait events, slow queries automatically
├── Create CloudWatch alarm: DatabaseConnections > 80% of max
│   → Alert before connection exhaustion
├── Enable Enhanced Monitoring (OS-level metrics)
├── Automate read replica creation for read-heavy apps
├── Enable RDS Proxy: pool connections, reduce DB load
└── Schedule automated snapshots + test restores monthly

For Aurora: Auto-scaling read replicas built-in
  → aurora:ReplicaLag metric → scale replicas on demand
```

---

## 4. Production Scenario FAQ — Azure

**Q: AKS pods are being OOMKilled. How do we handle this?**

```
OOMKilled = container exceeded its memory limit → kernel kills it

Immediate fix:
kubectl describe pod <name> -n production
→ Look for: "OOMKilled" in Last State section
→ Increase memory limit in deployment.yaml

Root cause investigation:
├── Check Azure Monitor Container Insights → Memory Working Set
├── Review app code for memory leaks
└── Use VPA (Vertical Pod Autoscaler) recommendation mode
    → VPA will tell you the right memory request/limit without enforcing

Prevention automation:
├── Set memory alerts in Azure Monitor (>80% of limit)
├── Use LimitRange in namespace (default requests/limits for all pods)
└── Enable VPA in recommendation mode → review weekly
```

---

**Q: Azure DevOps pipeline is slow (takes 30+ minutes). How do we speed it up?**

```
Pipeline optimization checklist:

1. Enable pipeline caching:
   - task: Cache@2
     inputs:
       key: 'pip | "$(Agent.OS)" | requirements.txt'
       path: $(PIP_CACHE_DIR)

2. Parallelize independent jobs:
   jobs:
   - job: UnitTests    # runs in parallel
   - job: Lint         # runs in parallel
   - job: SecurityScan # runs in parallel

3. Use self-hosted agents (faster machines, no setup time)
4. Split build and deploy: fail fast on tests before building image
5. Use Docker layer caching in ACR Tasks
6. Only run expensive steps on main branch (not every PR)

Common culprits:
├── Not caching pip/npm → reinstalls every run
├── Sequential jobs that could run in parallel
├── Building Docker image from scratch every time (no layer cache)
└── Running integration tests on every PR (move to merge-only)
```

---

**Q: Terraform apply failed midway in production. How do we handle this?**

```
Partial apply = state is inconsistent with reality

Step 1: DO NOT run terraform apply again blindly
Step 2: terraform state list → see what was created vs not
Step 3: terraform plan -refresh-only → see what Azure actually has
Step 4: For each partially created resource, decide:
        - Delete it manually → terraform state rm <resource>
        - Keep it → terraform import <address> <azure_id>
Step 5: Run terraform plan again (should show correct delta)
Step 6: Carefully apply remaining changes

Prevention:
├── Use create_before_destroy for zero-downtime resource replacement
├── Use -parallelism=1 during risky applies (serialize operations)
├── Test complex changes in staging FIRST
└── Keep prod state file backed up before risky applies
    az storage blob copy start ...
```

---

**Q: Azure VM in production is under DDoS attack. What's the automated response?**

```
Azure DDoS Protection automation:

Preventive (automate these before incidents):
├── Enable Azure DDoS Protection Standard on VNet (Terraform)
├── Enable Azure WAF on Application Gateway
├── Set rate limiting rules in WAF policy
└── Set NSG rules: only allow known CIDRs to management ports

During attack (automated response):
├── DDoS Protection triggers automatically (absorbs + mitigates)
├── WAF blocks malicious patterns automatically
├── Azure Monitor alert: "DDoS attack detected" → Action Group
├── Action Group → Logic App → post to #security Slack channel
└── Auto-scale VMSS to absorb legitimate traffic surge

Post-incident automation:
├── Extract attacker IPs from NSG Flow Logs → add to block list
└── Generate DDoS diagnostic report from Azure Monitor
```

---

**Q: Key Vault secret was accidentally deleted. How do we recover?**

```
Key Vault soft-delete saves you (enabled by default, 90 day retention):

# List soft-deleted secrets
az keyvault secret list-deleted --vault-name kv-myapp-prod

# Recover the secret
az keyvault secret recover \
  --name my-db-password \
  --vault-name kv-myapp-prod

# If purge protection was also enabled (more protection):
# Must wait out retention period OR file Microsoft support request

Prevention automation:
├── Enable purge protection in Terraform:
│   purge_protection_enabled = true
│   soft_delete_retention_days = 90
├── Replicate critical secrets across two Key Vaults in different regions
└── Azure Policy: enforce purge protection on all Key Vaults
```

---

## 5. Top Gotchas for Beginners

These are high-frequency mistakes. Read them. Remember them.

```
┌────────────────────────────────────────────────────────────────┐
│  TOP 15 DEVOPS GOTCHAS FOR BEGINNERS                           │
│                                                                │
│  IaC:                                                          │
│  1. Changing a resource NAME in Terraform often = destroy +    │
│     recreate. Always check "Forces new resource" in docs.     │
│  2. Never use terraform apply -auto-approve in production.    │
│  3. State file contains plaintext secrets. Use remote backend. │
│                                                                │
│  AWS:                                                          │
│  4. az vm stop still charges for compute. Use deallocate.     │
│     (Azure) / ec2 stop still charges for EBS (AWS).          │
│  5. S3 bucket names are globally unique across all accounts.   │
│  6. EC2 memory metrics need CloudWatch Agent — not built in.  │
│  7. IAM changes can take up to 60 seconds to propagate.       │
│                                                                │
│  Azure:                                                        │
│  8. az group delete destroys EVERYTHING instantly — irreversib.│
│  9. Key Vault soft-deleted vaults block reuse of same name.   │
│  10. AKS control plane is free but node VMs are NOT.          │
│                                                                │
│  Kubernetes:                                                   │
│  11. Liveness probe failure = pod restart. Don't check         │
│      external deps (DB) in liveness — use readiness instead.  │
│  12. Pod without resource limits = noisy neighbor = node OOM. │
│  13. :latest image tag = non-deterministic deployments.        │
│                                                                │
│  CI/CD:                                                        │
│  14. Secrets in pipeline YAML are visible in Git history.     │
│      Use secret variables or Key Vault references.            │
│  15. Not pinning dependency versions = "it worked yesterday"  │
│      mystery failures. Always pin pip/npm/helm chart versions. │
└────────────────────────────────────────────────────────────────┘
```

---

## 6. The DevOps Automation Checklist

Use this checklist when you join a new project or do a review:

### Infrastructure Checklist

- [ ] All infrastructure defined as code (Terraform/Bicep/CloudFormation)
- [ ] IaC stored in Git with PR reviews required
- [ ] Remote state backend configured with locking
- [ ] Production resources locked (prevent accidental deletion)
- [ ] All resources tagged (env, team, cost-center, managed-by)
- [ ] Naming convention documented and enforced

### CI/CD Checklist

- [ ] Every merge to main triggers automated build
- [ ] Tests run on every PR (unit, lint, security scan)
- [ ] Container images scanned for vulnerabilities before push
- [ ] IaC scanned with tfsec/checkov before apply
- [ ] Deployment to prod requires manual approval
- [ ] Rollback procedure documented and tested
- [ ] OIDC used for cloud authentication (no static secrets)

### Security Checklist

- [ ] No secrets in code or Git history
- [ ] Secrets stored in AWS Secrets Manager / Azure Key Vault
- [ ] Managed Identity / IRSA configured for workloads
- [ ] MFA enforced for all human accounts
- [ ] CloudTrail / Azure Activity Log enabled
- [ ] GuardDuty / Microsoft Defender enabled
- [ ] Least privilege IAM roles (no `*` on resources)
- [ ] NSG/Security Groups restrict traffic to minimum required

### Monitoring Checklist

- [ ] CPU, memory, disk alerts configured for all VMs
- [ ] Application error rate alerts configured
- [ ] Alerts route to Slack AND PagerDuty (on-call rotation)
- [ ] Log retention policy set (not logging forever)
- [ ] Runbooks written for top 5 most common alerts
- [ ] On-call rotation schedule set up

### Cost Checklist

- [ ] Budget alerts configured per subscription/account
- [ ] Dev instances auto-shutdown at night/weekends
- [ ] S3/Blob lifecycle policies in place
- [ ] Weekly orphaned resource cleanup automated
- [ ] Reserved instances purchased for stable prod workloads

---

*← [Part 3](./part3_azure_automation.md) | Series complete — start from [Part 1](./part1_intro_and_what_to_automate.md)*
