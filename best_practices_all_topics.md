# ✅ DevOps Best Practices — All Topics

> One file covering best practices for every topic in your trainee-docs.
> Use this as a quick-reference checklist before and during real work.

---

## 1. Linux

```
✅ DO:
├── Always use absolute paths in scripts (/home/user not ~/...)
├── Use set -euo pipefail at the top of every bash script
├── Check disk space (df -h) before large operations
├── Use less or tail -f instead of cat for large log files
├── Prefer find + xargs over for loops for file operations
├── Use trap for cleanup on script exit
├── Redirect stderr: command 2>/dev/null or 2>&1
├── Test scripts with bash -n script.sh before running
├── Use sudo only when necessary — never run entire scripts as root
└── Check man pages: man command or command --help

❌ DON'T:
├── rm -rf without double checking the path
├── Run unknown scripts with: curl | bash (pipe attack risk)
├── chmod 777 anything in production
├── Use root account for day-to-day work
├── Hard code passwords in scripts
└── Ignore exit codes — always check $? or use set -e
```

**Key Commands to Always Know:**
```bash
# Before doing anything risky:
pwd                        # Confirm you're where you think you are
ls -la                     # Confirm file contents
echo "rm -rf $DIR"         # Dry run: print the command before executing
```

---

## 2. Linux Scripting

```
✅ DO:
├── Line 1: #!/usr/bin/env bash (not #!/bin/bash — more portable)
├── Line 2: set -euo pipefail  (MANDATORY in prod scripts)
├── Use functions for repeated logic
├── Validate all inputs at script start
├── Log with timestamps: echo "[$(date)] Starting deployment"
├── Use readonly for constants
├── Use "${variable}" not $variable (handles spaces in names)
├── Quote all variables: rm -rf "${DIR}" not rm -rf $DIR
├── Use [[ ]] not [ ] for conditionals (safer, more features)
├── Test: run script with bash -x to see every command
└── Add usage() function for --help flag

❌ DON'T:
├── Parse ls output — use find or glob patterns
├── Use backticks `` — use $() instead (nestable, readable)
├── Forget to handle errors after each critical command
├── Write scripts longer than 200 lines without functions
├── Use eval with user input (code injection risk)
└── Echo passwords to stdout (shows in logs)
```

**Script Template:**
```bash
#!/usr/bin/env bash
set -euo pipefail
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
log() { echo "[$(date '+%H:%M:%S')] $*"; }
err() { echo "[ERROR] $*" >&2; exit 1; }
usage() { echo "Usage: $0 <env> <version>"; exit 0; }
[[ $# -lt 2 ]] && usage
cleanup() { log "Cleaning up..."; }
trap cleanup EXIT
```

---

## 3. Networking

```
✅ DO:
├── Understand CIDR notation before touching any firewall rule
├── Test connectivity before blaming the app: ping, curl, nc
├── Always restrict Security Groups / NSGs to minimum required ports
├── Use private subnets for databases and internal services
├── Use VPN or Bastion for admin access — never open 22 to 0.0.0.0/0
├── Document all firewall rules with a comment/tag explaining why
├── Test DNS resolution separately from connectivity
└── Use nslookup / dig to verify DNS before debugging app

❌ DON'T:
├── Open port 22 to the world (0.0.0.0/0) — ever
├── Allow 0.0.0.0/0 inbound on any port except 80/443
├── Assign public IPs to database servers
├── Share VPN credentials between team members
└── Change firewall rules in production without a rollback plan
```

**Debugging Order:**
```
Can I ping the server? → Is DNS resolving? → Is port open (nc/telnet)?
→ Is service listening (ss -tlnp)? → Is firewall blocking? → App issue
```

---

## 4. Git

```
✅ DO:
├── Write commit messages in imperative: "Add login" not "Added login"
├── Commit small and often — one logical change per commit
├── Always pull --rebase to avoid merge commits on feature branches
├── Use git add -p to stage only what belongs in this commit
├── Tag releases with semantic version: git tag -a v1.2.0
├── Use git stash when switching context mid-work
├── Protect main branch — require PR + review
├── Use --force-with-lease instead of --force
└── Check git log --oneline before pushing to verify history

❌ DON'T:
├── Commit directly to main (ever, in team projects)
├── Commit .env files, keys, or passwords
├── Use git push --force on shared branches
├── Write "WIP" or "fix" as commit messages
├── Rebase branches that others have pulled
└── Ignore merge conflicts — resolve them properly
```

**Commit Message Format:**
```
<type>(<scope>): <subject>

type: feat | fix | docs | chore | refactor | test | ci
Example: feat(auth): add JWT refresh token endpoint
         fix(db): handle null pointer on empty result
         ci: add trivy scan to PR pipeline
```

---

## 5. GitHub

```
✅ DO:
├── Always use branch protection on main and release/* branches
├── Enable required status checks (CI must pass before merge)
├── Enable secret scanning + push protection on all repos
├── Use CODEOWNERS for critical paths (infra/, auth/, migrations/)
├── Enable Dependabot for dependency updates
├── Use OIDC for GitHub Actions → cloud auth (no stored keys)
├── Pin action versions to full SHA for security
├── Keep PRs small (<400 lines) for effective review
├── Link PRs to issues (Closes #N auto-closes issue on merge)
└── Set up environments with required reviewers for production

❌ DON'T:
├── Store secrets as repository variables (use secrets)
├── Use classic PATs — use fine-grained tokens
├── Use self-hosted runners for public repos (code injection risk)
├── Merge PRs without any CI running
├── Allow force pushes on protected branches
└── Use @v1 or @latest for actions — pin to SHA
```

---

## 6. Docker

```
✅ DO:
├── Always use multi-stage builds for compiled languages
├── Pin base image versions: python:3.11-slim not python:latest
├── Run containers as non-root user (USER appuser)
├── Add HEALTHCHECK to every production image
├── Use .dockerignore to exclude node_modules, .git, .env
├── Copy requirements/package files BEFORE source code (cache layer)
├── Chain RUN commands with && and clean up in same layer
├── Use exec form for ENTRYPOINT: ["python", "app.py"]
├── Set PYTHONUNBUFFERED=1 and PYTHONDONTWRITEBYTECODE=1 for Python
└── Scan images with Trivy before pushing: trivy image myapp:v1

❌ DON'T:
├── Use :latest tag in production Dockerfiles
├── RUN apt-get update in a separate layer from install
├── Store secrets in ENV instructions or ARG values
├── Run database and app in the same container
├── Use ADD when COPY will do
├── Bake environment-specific config into images
└── Use privileged: true in compose unless absolutely required
```

**Image Size Targets:**
```
Python API:     < 200MB (use slim base + multi-stage)
Node.js API:    < 150MB (use alpine + multi-stage)
Go binary:      < 20MB  (use distroless/scratch)
Static site:    < 30MB  (nginx:alpine + built files)
```

---

## 7. Docker Compose

```
✅ DO:
├── Always define healthchecks on DB/cache services
├── Use condition: service_healthy in depends_on (not just service name)
├── Set restart: unless-stopped for production services
├── Set resource limits (memory, CPU) for all services
├── Use named volumes for persistent data (not bind mounts)
├── Separate dev and prod with override files
├── Never publish DB ports to 0.0.0.0 — use 127.0.0.1:5432:5432
├── Use internal: true networks for backend services
├── Store secrets in .env file (not in compose YAML)
└── Add logging limits to prevent disk filling: max-size: 10m

❌ DON'T:
├── Hard code passwords in docker-compose.yml
├── Use depends_on without healthcheck (race condition!)
├── Bind mount entire project in production (dev pattern only)
├── Use restart: always (use unless-stopped instead)
├── Run docker compose down -v in prod (deletes all data!)
└── Expose all ports — only expose what external clients need
```

---

## 8. Kubernetes

```
✅ DO:
├── Always set resource requests AND limits for every container
├── Set readinessProbe + livenessProbe on all pods
├── Use startupProbe for slow-starting apps (Java, ML models)
├── Set minReplicas >= 2 for any production workload
├── Use pod anti-affinity to spread replicas across nodes/zones
├── Set Pod Disruption Budgets (PDB) for critical services
├── Use namespaces to isolate workloads by team/environment
├── Never use :latest image tags — always pin to digest or SHA
├── Use Helm for packaging: enables versioning and rollback
├── Set terminationGracePeriodSeconds >= app shutdown time
├── Use Network Policies to restrict pod-to-pod communication
└── Use Secrets from Key Vault via CSI driver, not plain K8s secrets

❌ DON'T:
├── Run containers as root (securityContext.runAsNonRoot: true)
├── Expose Kubernetes API publicly without auth
├── Set CPU limit too low (causes CPU throttling, not OOMKill)
├── Use Liveness probe to check DB/external services (causes cascades)
├── Store sensitive data in ConfigMaps (use Secrets)
├── Skip resource limits (noisy neighbor problem)
└── kubectl apply in production without checking diff first
```

**Resource Limits Template:**
```yaml
resources:
  requests:
    memory: "128Mi"   # What the pod is GUARANTEED
    cpu: "100m"       # 0.1 CPU guaranteed
  limits:
    memory: "512Mi"   # Killed if exceeded (OOMKill)
    cpu: "500m"       # Throttled if exceeded (NOT killed)
```

---

## 9. Terraform

```
✅ DO:
├── Always run: fmt → validate → plan → apply (in that order)
├── Store state in remote backend (Azure Blob / AWS S3 + DynamoDB lock)
├── Pin provider versions: required_providers { azurerm = "~> 3.0" }
├── Commit .terraform.lock.hcl, never commit .terraform/ directory
├── Use modules for repeated patterns (networking, compute, monitoring)
├── Tag every resource: env, team, cost-center, managed-by=terraform
├── Use lifecycle { prevent_destroy = true } for production databases
├── Run terraform plan -out=tfplan in CI; apply tfplan in CD
├── Scan IaC with tfsec + checkov before every apply
├── Schedule regular terraform plan -refresh-only (drift detection)
└── Use terraform import for existing resources before touching them

❌ DON'T:
├── Store .tfstate in Git (contains plaintext secrets)
├── Use terraform apply -auto-approve in production manually
├── Store secrets in .tfvars files committed to Git
├── Mix multiple environments in one state file
├── Edit .tfstate file with a text editor
├── Rename Terraform resources without planning for destroy+recreate
└── Run terraform destroy without a clear backup/rollback plan
```

**State File Rules:**
```
One state file per:  environment (dev/staging/prod)
                     component (networking/compute/databases)
Never share state between environments.
Always enable state locking to prevent concurrent applies.
```

---

## 10. CI/CD Pipelines

```
✅ DO:
├── Run lint + tests on every PR (< 5 min feedback target)
├── Require all CI checks to pass before PR can merge
├── Tag images with git SHA — never :latest
├── Use OIDC for cloud authentication (no stored static keys)
├── Require manual approval before deploying to production
├── Run smoke tests immediately after every deployment
├── Implement rollback: helm rollback / kubectl rollout undo
├── Notify Slack on deployment success AND failure
├── Cache dependencies (pip, npm, go modules) to speed up builds
├── Parallelize independent jobs (lint, test, security scan)
└── Use environment-scoped secrets (dev secrets ≠ prod secrets)

❌ DON'T:
├── Store secrets in pipeline YAML files
├── Skip tests to "deploy faster" — this always ends badly
├── Deploy directly from feature branches to production
├── Run all pipeline stages sequentially when many can be parallel
├── Use the same agent/runner for dev and prod deployments
├── Deploy without a documented rollback procedure
└── Ignore security scan findings — block on CRITICAL/HIGH
```

---

## 11. Azure

```
✅ DO:
├── Use Managed Identity for all Azure services (no stored credentials)
├── Store ALL secrets in Azure Key Vault
├── Apply CanNotDelete lock on all production resource groups
├── Tag every resource: environment, team, cost-center
├── Use Private Endpoints for PaaS services (SQL, Storage, Key Vault)
├── Enable Defender for Cloud on all subscriptions
├── Set budget alerts per subscription (80% and 100% thresholds)
├── Use Policy to enforce naming conventions and tags
├── Enable diagnostic settings → Log Analytics for all resources
└── Use az vm deallocate (not stop) to stop billing

❌ DON'T:
├── Use az group delete without a double confirmation
├── Grant Owner/Contributor at subscription level to developers
├── Create resources manually in production (always IaC)
├── Leave dev VMs running overnight (auto-shutdown policy)
├── Use the same Service Principal for multiple environments
└── Forget Key Vault soft-delete retention — purge blocks name reuse
```

---

## 12. Cloud General (AWS + Azure)

```
✅ DO:
├── Enable MFA for all human accounts (no exceptions)
├── Never use root/global admin for day-to-day work
├── Follow Principle of Least Privilege for all IAM/RBAC
├── Enable CloudTrail / Activity Log audit logging
├── Enable threat detection (GuardDuty / Defender for Cloud)
├── Use separate accounts/subscriptions per environment
├── Set up billing alerts before doing anything
├── Use managed services over self-hosted when possible
└── Always have a disaster recovery plan and test it

❌ DON'T:
├── Create long-lived access keys/secrets (use temporary creds)
├── Allow wildcard (*) resource in IAM policies
├── Open security groups to 0.0.0.0/0 on sensitive ports
├── Run databases with public endpoints
├── Ignore cost anomaly alerts
└── Deploy to production on a Friday afternoon
```

---

## 13. Servers & OS

```
✅ DO:
├── Keep OS patched: enable automatic security updates
├── Disable root SSH login (PermitRootLogin no in sshd_config)
├── Use SSH keys not passwords for authentication
├── Use Fail2ban to block brute force attempts
├── Rotate logs: configure logrotate for application logs
├── Monitor disk usage — set alert at 80%
├── Use NTP time synchronization (chrony/ntpd)
└── Firewall: only open ports the application actually uses

❌ DON'T:
├── SSH with root (use sudo from a regular user)
├── Leave default credentials on any service
├── Ignore failed SSH login alerts
├── Install software from untrusted PPAs/repos
└── Disable SELinux/AppArmor (harden, don't disable)
```

---

## 14. Windows

```
✅ DO:
├── Use PowerShell 7+ (cross-platform) not cmd.exe for scripts
├── Use Windows Update for Business for patch management
├── Enable Windows Defender + Firewall on all servers
├── Use Group Policy for consistent configuration
├── Enable event logging and ship to SIEM
└── Use WinRM over HTTPS (not HTTP) for remote management

❌ DON'T:
├── Disable Windows Defender to "fix" performance issues
├── Use local Administrator account for services (use gMSA)
├── Store passwords in plaintext in Registry or scripts
└── Allow RDP from the internet (0.0.0.0/0 on 3389)
```

---

## 15. Security (Cross-Cutting)

```
┌──────────────────────────────────────────────────────────────┐
│  UNIVERSAL SECURITY BEST PRACTICES                           │
│                                                              │
│  Secrets:                                                    │
│  ├── Never in code, Git, Docker images, or logs             │
│  ├── Always in secrets manager (Key Vault / Secrets Manager)│
│  ├── Rotate on schedule (30-90 days for passwords)          │
│  └── Audit access logs regularly                            │
│                                                              │
│  Access:                                                     │
│  ├── Least privilege — minimum needed, nothing more         │
│  ├── Time-bound — temporary access for temporary tasks      │
│  ├── MFA for all human logins                               │
│  └── No shared accounts — each person has their own         │
│                                                              │
│  Patching:                                                   │
│  ├── OS patches: weekly minimum                             │
│  ├── Container base images: rebuild monthly minimum         │
│  ├── Dependencies: Dependabot weekly PRs                    │
│  └── Never run end-of-life software in production           │
│                                                              │
│  Visibility:                                                 │
│  ├── All API calls logged (CloudTrail/Activity Log)         │
│  ├── Alerts for suspicious activity (GuardDuty/Defender)    │
│  ├── Container runtime scanning (Falco)                     │
│  └── Regular compliance audits (AWS Config / Azure Policy)  │
└──────────────────────────────────────────────────────────────┘
```
