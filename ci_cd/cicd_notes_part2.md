# 🔄 CI/CD Pipeline Notes — Part 2: Tech Stack Pipelines, Security & Best Practices

> **Series:** Part 2 of 2 | [← Part 1](./cicd_notes_part1.md)

---

## 1. Complete Pipeline by Tech Stack

---

### Python (FastAPI / Django / Flask)

```yaml
# GitHub Actions — Python full pipeline
name: Python CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - uses: actions/setup-python@v5
      with:
        python-version: '3.11'

    - name: Cache pip
      uses: actions/cache@v4
      with:
        path: ~/.cache/pip
        key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}

    - name: Install dependencies
      run: |
        pip install -r requirements.txt
        pip install -r requirements-dev.txt

    - name: Lint (ruff)
      run: ruff check .

    - name: Format check (black)
      run: black --check .

    - name: Type check (mypy)
      run: mypy src/

    - name: Unit tests with coverage
      run: |
        pytest tests/unit/ \
          --cov=src \
          --cov-report=xml \
          --cov-fail-under=80 \
          --junitxml=test-results.xml

    - name: Security scan (bandit)
      run: bandit -r src/ -ll

    - name: Dependency CVE scan
      run: pip-audit

    - name: Build Docker image
      run: docker build -t myapp:${{ github.sha }} .

    - name: Scan image (Trivy)
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: myapp:${{ github.sha }}
        exit-code: '1'
        severity: 'CRITICAL,HIGH'

    - name: Push to registry
      if: github.ref == 'refs/heads/main'
      run: |
        echo "${{ secrets.REGISTRY_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
        docker tag myapp:${{ github.sha }} ghcr.io/${{ github.repository }}:${{ github.sha }}
        docker push ghcr.io/${{ github.repository }}:${{ github.sha }}
```

---

### Node.js (Express / Next.js / NestJS)

```yaml
name: Node.js CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  ci:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18, 20]

    steps:
    - uses: actions/checkout@v4

    - uses: actions/setup-node@v4
      with:
        node-version: ${{ matrix.node-version }}
        cache: 'npm'                    # Built-in cache shortcut

    - name: Install dependencies
      run: npm ci                       # ci = strict install from lockfile

    - name: Lint
      run: npm run lint

    - name: Type check
      run: npm run type-check           # tsc --noEmit

    - name: Unit tests
      run: npm run test -- --coverage --ci

    - name: Check coverage threshold
      run: npx jest --coverageThreshold='{"global":{"lines":80}}'

    - name: Build
      run: npm run build

    - name: Audit dependencies
      run: npm audit --audit-level=high

    - name: Build Docker image
      run: docker build -t myapp:${{ github.sha }} .

    - name: Trivy scan
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: myapp:${{ github.sha }}
        exit-code: '1'
        severity: 'CRITICAL,HIGH'
```

---

### Go (Golang)

```yaml
name: Go CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - uses: actions/setup-go@v5
      with:
        go-version: '1.22'
        cache: true                     # Caches go module cache automatically

    - name: Download modules
      run: go mod download

    - name: Verify modules (check go.sum integrity)
      run: go mod verify

    - name: Lint (golangci-lint)
      uses: golangci/golangci-lint-action@v6
      with:
        version: latest

    - name: Build
      run: go build -v ./...

    - name: Test with race detector
      run: go test -race -coverprofile=coverage.out ./...

    - name: Check coverage
      run: |
        COVERAGE=$(go tool cover -func=coverage.out | grep total | awk '{print $3}' | tr -d '%')
        if (( $(echo "$COVERAGE < 80" | bc -l) )); then
          echo "Coverage $COVERAGE% below threshold 80%"
          exit 1
        fi

    - name: Security scan (gosec)
      uses: securego/gosec@master
      with:
        args: ./...
```

---

### Java (Spring Boot / Maven / Gradle)

```yaml
name: Java CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - uses: actions/setup-java@v4
      with:
        java-version: '21'
        distribution: 'temurin'
        cache: 'maven'

    - name: Build and test (Maven)
      run: mvn -B verify --no-transfer-progress

    # For Gradle:
    # - name: Build and test (Gradle)
    #   run: ./gradlew build test

    - name: Generate JAR
      run: mvn -B package -DskipTests

    - name: Upload JAR artifact
      uses: actions/upload-artifact@v4
      with:
        name: app-jar
        path: target/*.jar

    - name: Build Docker (multi-stage)
      run: docker build -t myapp:${{ github.sha }} .

    - name: OWASP Dependency Check
      uses: dependency-check/Dependency-Check_Action@main
      with:
        project: myapp
        path: .
        format: HTML
        failBuildOnCVSS: 7            # Fail on HIGH+ CVEs
```

---

### Terraform (Infrastructure Pipeline)

```yaml
name: Terraform CI/CD

on:
  push:
    branches: [main]
    paths: ['infra/**']
  pull_request:
    branches: [main]
    paths: ['infra/**']

permissions:
  id-token: write                     # OIDC for Azure/AWS auth
  contents: read
  pull-requests: write                # Post plan to PR comment

jobs:
  terraform:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: infra/

    steps:
    - uses: actions/checkout@v4

    - uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: 1.8.0

    - name: Azure Login (OIDC — no stored secrets)
      uses: azure/login@v2
      with:
        client-id: ${{ secrets.AZURE_CLIENT_ID }}
        tenant-id: ${{ secrets.AZURE_TENANT_ID }}
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

    - name: Terraform Format Check
      run: terraform fmt -check -recursive
      continue-on-error: true

    - name: Terraform Init
      run: terraform init
      env:
        ARM_USE_OIDC: true

    - name: Security scan (tfsec)
      uses: aquasecurity/tfsec-action@v1.0.0
      with:
        working_directory: infra/

    - name: IaC scan (checkov)
      uses: bridgecrewio/checkov-action@master
      with:
        directory: infra/
        soft_fail: false

    - name: Terraform Validate
      run: terraform validate

    - name: Terraform Plan
      id: plan
      run: terraform plan -out=tfplan -no-color 2>&1 | tee plan.txt
      env:
        ARM_USE_OIDC: true

    - name: Post plan to PR
      uses: actions/github-script@v7
      if: github.event_name == 'pull_request'
      with:
        script: |
          const fs = require('fs');
          const plan = fs.readFileSync('infra/plan.txt', 'utf8');
          github.rest.issues.createComment({
            issue_number: context.issue.number,
            owner: context.repo.owner,
            repo: context.repo.repo,
            body: `## Terraform Plan\n\`\`\`hcl\n${plan.slice(0, 60000)}\n\`\`\``
          });

    - name: Terraform Apply
      if: github.ref == 'refs/heads/main' && github.event_name == 'push'
      run: terraform apply -auto-approve tfplan
      env:
        ARM_USE_OIDC: true
```

---

### Docker Compose (Local Dev → Staging)

```yaml
name: Docker Compose Pipeline

on:
  push:
    branches: [main]

jobs:
  integration-test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Start all services
      run: docker compose -f docker-compose.test.yml up -d --wait

    - name: Wait for health checks
      run: |
        timeout 60 bash -c \
          'until curl -sf http://localhost:8080/health; do sleep 2; done'

    - name: Run integration tests
      run: docker compose -f docker-compose.test.yml run test-runner

    - name: Print service logs on failure
      if: failure()
      run: docker compose -f docker-compose.test.yml logs

    - name: Tear down
      if: always()
      run: docker compose -f docker-compose.test.yml down -v
```

---

## 2. GitOps Pipeline (Argo CD / Flux)

```
┌──────────────────────────────────────────────────────────────────┐
│  GITOPS — TWO REPOSITORY MODEL                                   │
│                                                                  │
│  App Repo (source code)       Infra Repo (K8s manifests)        │
│  ├── src/                     ├── apps/                         │
│  ├── Dockerfile               │   ├── dev/                      │
│  └── .github/workflows/       │   ├── staging/                  │
│      ci.yml                   │   └── prod/                     │
│                               └── base/                         │
│       │                              │                          │
│       │ CI passes                    │                          │
│       │ builds image                 │                          │
│       ▼                              │                          │
│  Opens PR on Infra Repo:             │                          │
│  Update image tag:                   │                          │
│  image: myapp:sha-abc123             │                          │
│       │                              │                          │
│       │ PR merged                    │                          │
│       ▼                              ▼                          │
│                          Argo CD watches Infra Repo             │
│                          Detects: cluster != Git state          │
│                          Syncs automatically to cluster         │
└──────────────────────────────────────────────────────────────────┘
```

```yaml
# GitHub Actions: auto-update image tag in Infra Repo after CI
- name: Update image tag in Infra Repo
  uses: actions/github-script@v7
  env:
    NEW_TAG: ${{ github.sha }}
  with:
    github-token: ${{ secrets.INFRA_REPO_TOKEN }}
    script: |
      const { Octokit } = require("@octokit/rest");
      const octokit = new Octokit({ auth: process.env.GITHUB_TOKEN });

      // Get current file
      const { data: file } = await octokit.repos.getContent({
        owner: 'myorg',
        repo: 'infra-repo',
        path: 'apps/staging/values.yaml'
      });

      // Update image tag
      const content = Buffer.from(file.content, 'base64').toString();
      const updated = content.replace(/tag: .+/, `tag: ${process.env.NEW_TAG}`);

      // Commit updated file
      await octokit.repos.createOrUpdateFileContents({
        owner: 'myorg',
        repo: 'infra-repo',
        path: 'apps/staging/values.yaml',
        message: `ci: update myapp to ${process.env.NEW_TAG}`,
        content: Buffer.from(updated).toString('base64'),
        sha: file.sha
      });
```

---

## 3. Security in CI/CD (DevSecOps)

### Full Security Gate Sequence

```
┌──────────────────────────────────────────────────────────────────┐
│  SECURITY GATES — WHERE IN PIPELINE                              │
│                                                                  │
│  Pre-commit (developer machine):                                 │
│  ├── detect-secrets: block committing secrets                   │
│  ├── gitleaks: scan for credential patterns                     │
│  └── hadolint: lint Dockerfile for security issues              │
│                                                                  │
│  On PR (fast feedback):                                          │
│  ├── SAST: Semgrep / SonarQube (5-10 min)                      │
│  ├── Secret scan: TruffleHog on git history                     │
│  └── Dependency audit: pip-audit / npm audit / go mod verify    │
│                                                                  │
│  On merge to main (full scan, OK if slower):                    │
│  ├── Container image scan: Trivy (CRITICAL = fail)              │
│  ├── IaC scan: tfsec + checkov                                  │
│  ├── SBOM generation: Syft → Grype CVE check                    │
│  └── DAST: OWASP ZAP against staging environment               │
│                                                                  │
│  On deploy to production:                                        │
│  ├── Sign container image: cosign                               │
│  └── Verify signature before K8s admission                      │
└──────────────────────────────────────────────────────────────────┘
```

### Pre-commit Hook Setup

```bash
# Install pre-commit framework
pip install pre-commit

# .pre-commit-config.yaml
repos:
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets

  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks

  - repo: https://github.com/hadolint/hadolint
    rev: v2.12.0
    hooks:
      - id: hadolint
        args: [--failure-threshold, error]

  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.92.0
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_tfsec

# Install hooks locally
pre-commit install
```

### Trivy — Container Scanning in Pipeline

```yaml
- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
    format: 'sarif'               # GitHub Security tab integration
    output: 'trivy-results.sarif'
    exit-code: '1'
    ignore-unfixed: true          # Skip CVEs with no fix yet
    vuln-type: 'os,library'
    severity: 'CRITICAL,HIGH'

- name: Upload Trivy results to Security tab
  uses: github/codeql-action/upload-sarif@v3
  if: always()
  with:
    sarif_file: 'trivy-results.sarif'
```

---

## 4. Notifications & Reporting

```yaml
# Slack notification on failure
- name: Notify Slack on failure
  if: failure()
  uses: slackapi/slack-github-action@v1.27.0
  with:
    payload: |
      {
        "text": "❌ Pipeline FAILED",
        "attachments": [{
          "color": "danger",
          "fields": [
            {"title": "Repo", "value": "${{ github.repository }}", "short": true},
            {"title": "Branch", "value": "${{ github.ref_name }}", "short": true},
            {"title": "Commit", "value": "${{ github.sha }}", "short": true},
            {"title": "Run", "value": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}", "short": false}
          ]
        }]
      }
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
    SLACK_WEBHOOK_TYPE: INCOMING_WEBHOOK

# Slack notification on success
- name: Notify Slack on success
  if: success() && github.ref == 'refs/heads/main'
  uses: slackapi/slack-github-action@v1.27.0
  with:
    payload: |
      {
        "text": "✅ Deployed to production: ${{ github.sha }}"
      }
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
    SLACK_WEBHOOK_TYPE: INCOMING_WEBHOOK
```

---

## 5. Rollback Strategies

```bash
# --- Kubernetes: Rollback Deployment ---
kubectl rollout undo deployment/myapp
kubectl rollout undo deployment/myapp --to-revision=5

# --- Helm: Rollback Release ---
helm rollback myapp                    # Rollback to previous
helm rollback myapp 3                  # Rollback to revision 3
helm history myapp                     # See all revisions

# --- AWS ECS: Rollback Task Definition ---
# Get previous task definition ARN from AWS Console or CLI
aws ecs update-service \
  --cluster prod-cluster \
  --service myapp-service \
  --task-definition myapp:42            # Previous working version

# --- Azure App Service: Swap Slots Back ---
az webapp deployment slot swap \
  --resource-group rg-myapp-prod \
  --name myapp-prod \
  --slot staging \
  --target-slot production

# --- GitHub Actions: Trigger Rollback Workflow ---
gh workflow run rollback.yml \
  --field image_tag=sha-abc123
```

---

## 6. Pipeline Best Practices

### Speed Optimizations

```
┌──────────────────────────────────────────────────────────────────┐
│  MAKE YOUR PIPELINE FAST                                         │
│                                                                  │
│  ✅ Parallelize independent jobs:                                │
│  jobs:                                                           │
│    lint:    (parallel)                                           │
│    test:    (parallel)                                           │
│    security:(parallel)                                           │
│    build:   dependsOn: [lint, test, security]                   │
│                                                                  │
│  ✅ Cache aggressively:                                          │
│  - pip/npm/go modules                                           │
│  - Docker layer cache                                           │
│  - Test result caches                                           │
│                                                                  │
│  ✅ Fail fast: run quickest checks first                        │
│  Order: lint (10s) → unit tests (1min) → build (3min) → scan   │
│                                                                  │
│  ✅ Skip unnecessary steps:                                     │
│  paths-ignore: ['**.md', 'docs/**', '.github/CODEOWNERS']      │
│                                                                  │
│  ✅ Use matrix builds wisely:                                   │
│  Only test on N versions for library code.                      │
│  For apps, test on your production version only.                │
│                                                                  │
│  ✅ Self-hosted runners for large builds:                       │
│  Docker layer cache persists between runs.                      │
│  No startup time overhead.                                      │
└──────────────────────────────────────────────────────────────────┘
```

### Pipeline as Code — File Structure

```
.github/
├── workflows/
│   ├── ci.yml               # PR: lint, test, security scan
│   ├── cd.yml               # Push to main: build, deploy
│   ├── nightly.yml          # Scheduled: full scan + E2E
│   └── rollback.yml         # Manual: rollback to tag
│
# For Azure DevOps:
azure-pipelines/
├── ci.yml                   # PR pipeline
├── cd-staging.yml           # Staging deploy
├── cd-prod.yml              # Production deploy (approval gated)
└── templates/
    ├── build-steps.yml      # Reusable build steps
    └── deploy-steps.yml     # Reusable deploy steps
```

### Reusable Steps (DRY Pipelines)

```yaml
# GitHub Actions — Reusable Workflow
# .github/workflows/build-docker.yml (reusable)
on:
  workflow_call:
    inputs:
      image_name:
        required: true
        type: string
    outputs:
      image_tag:
        value: ${{ jobs.build.outputs.tag }}
    secrets:
      REGISTRY_TOKEN:
        required: true

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      tag: ${{ steps.meta.outputs.version }}
    steps:
      - uses: actions/checkout@v4
      - name: Build and push
        id: meta
        # ... build steps

---

# Call the reusable workflow from another:
jobs:
  build:
    uses: ./.github/workflows/build-docker.yml
    with:
      image_name: myapp
    secrets:
      REGISTRY_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  deploy:
    needs: build
    steps:
      - run: echo "Image tag: ${{ needs.build.outputs.image_tag }}"
```

```yaml
# Azure DevOps — Template steps (reusable)
# templates/docker-build-steps.yml
parameters:
  - name: imageName
    type: string
  - name: tag
    type: string

steps:
  - task: Docker@2
    displayName: Build ${{ parameters.imageName }}
    inputs:
      command: build
      repository: ${{ parameters.imageName }}
      tags: ${{ parameters.tag }}

---
# Use template in main pipeline
steps:
  - template: templates/docker-build-steps.yml
    parameters:
      imageName: myapp
      tag: $(Build.BuildId)
```

---

## 7. Environment Protection Rules

```
┌──────────────────────────────────────────────────────────────────┐
│  ENVIRONMENT GATES — WHAT TO CONFIGURE                           │
│                                                                  │
│  GitHub Actions: Settings → Environments                        │
│  ├── DEV                                                        │
│  │   └── No protection (auto-deploy)                           │
│  ├── STAGING                                                    │
│  │   ├── Required reviewers: none                              │
│  │   └── Wait timer: 0 min                                     │
│  └── PRODUCTION                                                 │
│      ├── Required reviewers: 1 person from "prod-approvers"    │
│      ├── Wait timer: 5 min (cooldown)                          │
│      └── Deployment branches: main only                         │
│                                                                  │
│  Azure DevOps: Pipelines → Environments                         │
│  ├── Approvals: specific users or group must approve            │
│  ├── Branch control: only deploy from main/release/*            │
│  ├── Business hours gate: only deploy Mon-Fri 9am-5pm          │
│  └── Exclusive lock: only one deploy to prod at a time         │
└──────────────────────────────────────────────────────────────────┘
```

---

## 8. Common Pipeline Anti-patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| All stages sequential | Pipeline takes 45 min | Parallelize lint, test, security |
| No caching | Reinstalls deps every run | Cache pip/npm/go modules |
| `:latest` Docker tag | Can't trace which code is in prod | Tag with `git sha` |
| Secrets in YAML | Visible in Git history | Use secrets manager / vault |
| Deploy on every PR merge | Pushes broken code to prod | Only deploy from main branch |
| No rollback mechanism | Can't recover from bad deploy | Helm rollback / blue-green |
| Skip tests to "go faster" | Bugs reach production | Never skip — speed up tests instead |
| One giant pipeline file | Hard to maintain, slow to debug | Split into separate workflow files |
| No notifications | Team doesn't know about failures | Slack/email on failure + deploy |
| Force push to main | Destroys pipeline history | Protect branch: require PR |

---

## 9. Pipeline Debugging Commands

```bash
# --- GitHub Actions ---
# View workflow runs
gh run list --repo myorg/myapp --limit 20

# View run details
gh run view 1234567890

# Download logs from a failed run
gh run download 1234567890

# Re-run a failed run
gh run rerun 1234567890

# Re-run only failed jobs
gh run rerun 1234567890 --failed

# Trigger workflow manually
gh workflow run deploy.yml --field environment=staging

# List workflows
gh workflow list

# Watch run in terminal
gh run watch 1234567890

# --- Azure DevOps ---
# List recent runs
az pipelines runs list --pipeline-name "my-pipeline" --output table

# Show specific run
az pipelines runs show --id 42

# Re-queue (restart) a failed run
az pipelines runs tag add --run-id 42 --tag retry

# Download pipeline logs
az pipelines runs artifact download \
  --artifact-name drop --path ./downloaded --run-id 42

# Cancel a running pipeline
az pipelines runs cancel --id 42
```

---

## 10. Golden Pipeline Checklist

Use this before shipping any pipeline to production:

```
CI PIPELINE:
  [ ] Triggered on every PR to main/release branches
  [ ] Linting + formatting check (fail on violations)
  [ ] Unit tests with coverage threshold (≥80%)
  [ ] Dependency vulnerability scan (fail on CRITICAL/HIGH)
  [ ] Secret / credential detection scan
  [ ] SAST code analysis (Semgrep/SonarQube)
  [ ] Docker image build works
  [ ] Container vulnerability scan (Trivy)
  [ ] IaC scan if Terraform changes (tfsec/checkov)
  [ ] Results posted to PR comment or GitHub Security tab
  [ ] Pipeline completes in < 10 min for PRs

CD PIPELINE:
  [ ] Only triggered on merge to main (not every PR)
  [ ] Image tagged with git SHA (not :latest)
  [ ] Image pushed to private registry (ECR/ACR/GHCR)
  [ ] Auto-deploy to DEV after CI passes
  [ ] Smoke test runs on DEV after deploy
  [ ] Manual approval required for PRODUCTION
  [ ] Production deploy uses rolling/blue-green strategy
  [ ] Post-deploy health check (curl /health or readiness probe)
  [ ] Rollback mechanism documented and tested
  [ ] Slack/email notification on success AND failure
  [ ] No hardcoded secrets in pipeline YAML

SECURITY:
  [ ] Cloud authentication uses OIDC (no static keys)
  [ ] Secrets stored in secrets manager / Key Vault
  [ ] Each environment has separate secrets
  [ ] Pipeline has minimum required permissions (least privilege)
  [ ] Branch protection: main requires PR + review
```

---

## 11. Complete Summary — Quick Reference

```
┌──────────────────────────────────────────────────────────────────┐
│  CI/CD AT A GLANCE                                               │
│                                                                  │
│  Triggers:        push, PR, schedule, manual, webhook            │
│  Runner types:    cloud-hosted, self-hosted, K8s-based           │
│  Stage order:     checkout → lint → test → scan → build →        │
│                   publish → deploy-dev → deploy-staging →        │
│                   approve → deploy-prod → verify → notify        │
│  Image tag:       always git SHA, never :latest                  │
│  Secrets:         secrets manager / Key Vault, not YAML          │
│  Auth:            OIDC for cloud (no static keys)                │
│  Approval gate:   required for production environment            │
│  Deploy strategy: rolling (default), blue-green, canary          │
│  Rollback:        helm rollback / kubectl rollout undo            │
│  Speed tricks:    parallel jobs, caching, fail-fast              │
│  Notifications:   Slack on failure + successful prod deploy      │
│                                                                  │
│  Tools by job:                                                   │
│  Linting:     ruff, eslint, golangci-lint, hadolint              │
│  Testing:     pytest, jest, go test, JUnit                       │
│  SAST:        Semgrep, SonarQube, bandit, gosec                 │
│  Dep scan:    pip-audit, npm audit, Trivy, Snyk                 │
│  Image scan:  Trivy, Grype, Snyk Container                      │
│  IaC scan:    tfsec, checkov, terrascan                          │
│  Secrets:     TruffleHog, gitleaks, detect-secrets              │
│  Deploy:      kubectl, helm, aws ecs, az webapp                  │
│  GitOps:      Argo CD, Flux (Git = source of truth)              │
└──────────────────────────────────────────────────────────────────┘
```

---

*← [Part 1: Fundamentals & Core Concepts](./cicd_notes_part1.md)*
