# 🔄 CI/CD Pipeline Notes — Part 1: Fundamentals & Core Concepts

> **Series:** Part 1 of 2 | [Part 2 →](./cicd_notes_part2.md)
> Covers: What CI/CD is, standard pipeline stages, trigger types, runner types, Docker in pipelines

---

## 1. What is CI/CD?

```
┌──────────────────────────────────────────────────────────────────┐
│  CI/CD = Continuous Integration + Continuous Delivery/Deployment │
│                                                                  │
│  CONTINUOUS INTEGRATION (CI):                                    │
│  Every developer's code change is automatically built,           │
│  tested, and validated — multiple times per day.                 │
│  Goal: Detect bugs EARLY, not in production.                    │
│                                                                  │
│  CONTINUOUS DELIVERY (CD):                                       │
│  After CI passes, the app is automatically prepared for          │
│  release. A human clicks "deploy to production."                 │
│  Goal: Release is always READY, deploy is a business decision.  │
│                                                                  │
│  CONTINUOUS DEPLOYMENT (CD):                                     │
│  After CI passes, the app is automatically deployed to           │
│  production with NO human intervention.                          │
│  Goal: Every passing commit ships to prod automatically.         │
│                                                                  │
│  Timeline of a change:                                           │
│                                                                  │
│  Developer writes code                                           │
│       │ git push / git commit                                    │
│       ▼                                                          │
│  CI kicks off automatically (webhook trigger)                    │
│  ├── Build (compile / docker build)                             │
│  ├── Unit tests                                                  │
│  ├── Static analysis / linting                                  │
│  ├── Security scans                                             │
│  └── Integration tests                                          │
│       │ all pass                                                 │
│       ▼                                                          │
│  CD kicks off                                                    │
│  ├── Deploy to DEV/STAGING                                      │
│  ├── Run smoke tests                                            │
│  └── Deploy to PROD (manual approval or automatic)              │
└──────────────────────────────────────────────────────────────────┘
```

**Why CI/CD matters:**

| Without CI/CD | With CI/CD |
|--------------|-----------|
| "It works on my machine" | Works everywhere reproducibly |
| Deploy every 2 weeks, high risk | Deploy many times/day, low risk |
| Find bugs in production | Find bugs before they reach users |
| Manual, error-prone deployments | Automated, consistent deployments |
| No visibility into what changed | Every deploy is traceable to a commit |
| Rollback = SSH + pray | Rollback = one command / click |

---

## 2. CI/CD Terminology

| Term | Meaning |
|------|---------|
| **Pipeline** | The full automated workflow (build → test → deploy) |
| **Stage** | A logical group of steps (e.g., "Build", "Test", "Deploy") |
| **Job** | A unit of work inside a stage, runs on one agent/runner |
| **Step / Task** | A single command or action inside a job |
| **Trigger** | What starts the pipeline (push, PR, schedule, manual) |
| **Artifact** | Output of a job passed to the next (compiled binary, Docker image) |
| **Agent / Runner** | The machine that executes pipeline jobs |
| **Environment** | A target deployment destination (dev, staging, prod) |
| **Approval Gate** | A manual checkpoint before deploying to production |
| **Workspace** | Shared filesystem on the runner during a pipeline run |
| **Secret / Variable** | Sensitive or dynamic values injected at runtime |
| **Cache** | Saved dependencies to speed up future runs |

---

## 3. Standard Pipeline Stage Sequence

```
┌──────────────────────────────────────────────────────────────────┐
│  STANDARD PIPELINE — STAGE ORDER                                 │
│                                                                  │
│  Stage 1: SOURCE                                                 │
│  └── Checkout code from Git repository                          │
│       │                                                          │
│  Stage 2: VALIDATE                                               │
│  ├── Code linting (eslint, ruff, golangci-lint)                 │
│  ├── Code formatting check (prettier, black, gofmt)             │
│  └── Static type checking (mypy, tsc --noEmit)                  │
│       │                                                          │
│  Stage 3: TEST                                                   │
│  ├── Unit tests (fast, no external deps)                        │
│  ├── Code coverage check (fail if < 80%)                        │
│  └── Integration tests (with mocked services)                   │
│       │                                                          │
│  Stage 4: SECURITY SCAN                                          │
│  ├── SAST — scan source code (Semgrep, SonarQube)              │
│  ├── Dependency scan — CVEs in packages (Trivy, Snyk)          │
│  └── Secret detection — leaked creds (TruffleHog, gitleaks)    │
│       │                                                          │
│  Stage 5: BUILD                                                  │
│  ├── Compile / transpile source code                            │
│  ├── Build Docker image                                          │
│  └── Scan Docker image (Trivy image scan)                       │
│       │                                                          │
│  Stage 6: PUBLISH                                                │
│  ├── Push Docker image to registry (ECR / ACR / GHCR)         │
│  └── Upload build artifacts (S3, Azure Blob, Nexus)            │
│       │                                                          │
│  Stage 7: DEPLOY — DEV                                           │
│  ├── Deploy to dev environment (auto)                           │
│  └── Run smoke tests on dev                                     │
│       │                                                          │
│  Stage 8: DEPLOY — STAGING                                       │
│  ├── Deploy to staging (auto or light approval)                 │
│  ├── Run E2E tests                                              │
│  └── Run performance tests                                      │
│       │                                                          │
│  Stage 9: DEPLOY — PRODUCTION                                    │
│  ├── Manual approval required                                   │
│  ├── Deploy using chosen strategy (rolling/blue-green/canary)  │
│  ├── Run post-deploy smoke tests                                │
│  └── Alert on success or auto-rollback on failure              │
└──────────────────────────────────────────────────────────────────┘
```

> **Gotcha:** Don't run ALL stages on every PR. Run lint+test on PRs, run full security scan + deploy only on merge to main. This keeps PR feedback fast (< 5 min) and saves CI costs.

---

## 4. Pipeline Trigger Types

```bash
# --- GitHub Actions Trigger Examples ---

on:
  push:
    branches: [main, release/*]         # Push to specific branches
    paths-ignore: ['**.md', 'docs/**']  # Skip if only docs changed

  pull_request:
    branches: [main]                    # PR targeting main

  schedule:
    - cron: '0 2 * * *'                 # Nightly at 2 AM UTC

  workflow_dispatch:                    # Manual trigger via UI/API
    inputs:
      environment:
        description: 'Target environment'
        required: true
        default: 'staging'

  workflow_call:                        # Called by another workflow
    inputs:
      image_tag:
        type: string
        required: true
```

```yaml
# --- Azure DevOps Trigger Examples ---

trigger:
  branches:
    include: [main, release/*]
  paths:
    exclude: [docs/**, '*.md']

pr:
  branches:
    include: [main]

schedules:
  - cron: "0 2 * * *"
    displayName: Nightly build
    branches:
      include: [main]
    always: true                        # Run even if no code change
```

---

## 5. Runner / Agent Types

```
┌──────────────────────────────────────────────────────────────────┐
│  RUNNER TYPES — WHERE YOUR PIPELINE RUNS                         │
│                                                                  │
│  Type 1: Cloud-Hosted / Microsoft-Hosted                        │
│  ├── Managed by GitHub/Azure — zero maintenance                 │
│  ├── Fresh VM for every run (no leftover state)                 │
│  ├── Limited storage/RAM (e.g., 7GB RAM, 14GB SSD)             │
│  ├── Slower: download dependencies every time                   │
│  ├── No access to private VNet resources                        │
│  └── Use when: simple pipelines, public projects               │
│                                                                  │
│  GitHub:  runs-on: ubuntu-latest / windows-latest / macos-latest│
│  Azure:   vmImage: ubuntu-latest / windows-latest               │
│                                                                  │
│  Type 2: Self-Hosted Runner / Agent                              │
│  ├── Your own VM or container                                   │
│  ├── Persistent — tools stay installed between runs             │
│  ├── Faster: no setup time, large caches persist               │
│  ├── Can access private VNet, on-prem resources                │
│  ├── You manage patches, scaling, availability                  │
│  └── Use when: private infra, large builds, GPU workloads      │
│                                                                  │
│  Type 3: Kubernetes-based runners (ephemeral)                   │
│  ├── GitHub Actions Runner Controller (ARC) in K8s             │
│  ├── Azure DevOps Agent Pool on AKS                            │
│  ├── Best of both: scales to zero + private VNet access        │
│  └── Use when: cost matters + private network needed           │
└──────────────────────────────────────────────────────────────────┘
```

---

## 6. Docker in CI/CD Pipelines

### Build and Push Pattern

```yaml
# GitHub Actions — Build and push to GitHub Container Registry
- name: Log in to registry
  uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}

- name: Extract metadata
  id: meta
  uses: docker/metadata-action@v5
  with:
    images: ghcr.io/${{ github.repository }}
    tags: |
      type=sha,prefix=sha-               # sha-abc1234
      type=ref,event=branch              # main
      type=semver,pattern={{version}}    # v1.2.3 (from git tag)

- name: Build and push
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: ${{ steps.meta.outputs.tags }}
    labels: ${{ steps.meta.outputs.labels }}
    cache-from: type=gha                 # Use GitHub Actions cache
    cache-to: type=gha,mode=max
```

### Docker Layer Caching Strategy

```
┌──────────────────────────────────────────────────────────────────┐
│  OPTIMIZE DOCKERFILE FOR CI CACHING                              │
│                                                                  │
│  SLOW Dockerfile (no caching benefit):                           │
│  COPY . /app                    ← copies everything first       │
│  RUN pip install -r requirements.txt  ← reinstalls every time   │
│                                                                  │
│  FAST Dockerfile (layer cache works):                            │
│  COPY requirements.txt /app/    ← copy only deps file first     │
│  RUN pip install -r requirements.txt  ← cached if unchanged     │
│  COPY . /app                    ← code changes don't bust cache  │
│                                                                  │
│  Rule: Put things that change RARELY at the top.                │
│        Put things that change OFTEN at the bottom.              │
└──────────────────────────────────────────────────────────────────┘
```

---

## 7. Secrets & Variables Management

```
┌──────────────────────────────────────────────────────────────────┐
│  SECRET HIERARCHY IN PIPELINES                                   │
│                                                                  │
│  GitHub Actions:                                                 │
│  ├── Repository Secrets: for one repo only                      │
│  ├── Organization Secrets: shared across repos                  │
│  └── Environment Secrets: scoped to dev/staging/prod env        │
│                                                                  │
│  Azure DevOps:                                                   │
│  ├── Pipeline Variables: defined in YAML                        │
│  ├── Variable Groups: shared across pipelines                   │
│  └── Variable Group → Key Vault: secrets pulled from Key Vault  │
│                                                                  │
│  Accessing in pipeline:                                          │
│  GitHub:  ${{ secrets.MY_SECRET }}                              │
│  Azure:   $(MY_SECRET)                                          │
│                                                                  │
│  NEVER:                                                          │
│  ├── echo "${{ secrets.TOKEN }}"  ← prints to log (exposed!)   │
│  ├── Store in YAML files committed to Git                       │
│  └── Use same secret for dev and prod                           │
└──────────────────────────────────────────────────────────────────┘
```

```yaml
# GitHub Actions — secrets usage
env:
  DB_HOST: ${{ secrets.DB_HOST }}
  API_KEY: ${{ secrets.API_KEY }}

# Azure DevOps — variable group from Key Vault
variables:
  - group: prod-secrets          # Variable group linked to Key Vault
  - name: imageTag
    value: $(Build.BuildId)
```

---

## 8. Caching Dependencies

```yaml
# --- GitHub Actions: Python (pip) ---
- uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
    restore-keys: ${{ runner.os }}-pip-

# --- GitHub Actions: Node.js (npm) ---
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('package-lock.json') }}

# --- GitHub Actions: Go ---
- uses: actions/cache@v4
  with:
    path: |
      ~/go/pkg/mod
      ~/.cache/go-build
    key: ${{ runner.os }}-go-${{ hashFiles('go.sum') }}

# --- Azure DevOps: pip cache ---
- task: Cache@2
  inputs:
    key: 'pip | "$(Agent.OS)" | requirements.txt'
    restoreKeys: 'pip | "$(Agent.OS)"'
    path: $(PIP_CACHE_DIR)
```

> **Gotcha:** Cache keys must include a hash of the lockfile. If you just cache by OS, everyone shares the same cache — a dependency update won't be detected. Always use `hashFiles('requirements.txt')` or `hashFiles('package-lock.json')` as part of the key.

---

## 9. Artifacts — Passing Data Between Jobs

```yaml
# GitHub Actions — upload artifact from build job
- name: Upload build artifact
  uses: actions/upload-artifact@v4
  with:
    name: app-binary
    path: ./dist/
    retention-days: 7

# Download in another job
- name: Download build artifact
  uses: actions/download-artifact@v4
  with:
    name: app-binary
    path: ./dist/
```

```yaml
# Azure DevOps — publish and download artifacts
# In Build stage:
- task: PublishPipelineArtifact@1
  inputs:
    targetPath: $(Build.ArtifactStagingDirectory)
    artifact: drop
    publishLocation: pipeline

# In Deploy stage:
- task: DownloadPipelineArtifact@2
  inputs:
    artifact: drop
    path: $(Pipeline.Workspace)/drop
```

---

## 10. Deployment Strategies

```
┌──────────────────────────────────────────────────────────────────┐
│  DEPLOYMENT STRATEGIES — CHOOSE THE RIGHT ONE                    │
│                                                                  │
│  1. ROLLING DEPLOYMENT (default in Kubernetes)                   │
│  Old: [v1][v1][v1][v1]                                          │
│       ↓ replaces one at a time                                   │
│  New: [v2][v2][v2][v2]                                          │
│  ✅ Zero downtime  ✅ Simple  ❌ Both versions run together     │
│                                                                  │
│  2. BLUE-GREEN DEPLOYMENT                                        │
│  Blue (live):  [v1][v1][v1]  ← 100% traffic                    │
│  Green (new):  [v2][v2][v2]  ← 0% traffic                      │
│  Switch LB → Green instantly                                     │
│  ✅ Instant rollback  ✅ No mixed versions  ❌ 2x infrastructure │
│                                                                  │
│  3. CANARY DEPLOYMENT                                            │
│  [v1][v1][v1][v1][v1][v1][v1][v1][v1][v2]                      │
│  10% traffic to v2, monitor errors                              │
│  Gradually increase: 25% → 50% → 100%                          │
│  ✅ Low risk  ✅ Real traffic testing  ❌ Complex routing        │
│                                                                  │
│  4. RECREATE (simplest, but has downtime)                        │
│  Kill all v1 pods → start all v2 pods                           │
│  ✅ Simple  ❌ Downtime during switch                           │
│  Use for: dev environments only                                  │
│                                                                  │
│  WHEN TO USE:                                                    │
│  Rolling   → standard web services on K8s (default)            │
│  Blue-Green → critical services needing instant rollback        │
│  Canary    → high-traffic services, risk-averse teams           │
│  Recreate  → dev/staging, stateful apps that can't run 2x      │
└──────────────────────────────────────────────────────────────────┘
```

---

## 11. GitHub Actions — Full Reference

### Workflow Structure

```yaml
name: CI/CD Pipeline              # Workflow name (shown in UI)

on:                               # Triggers
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:                              # Global environment variables
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:                           # Job ID (must be unique)
    name: Run Tests               # Display name in UI
    runs-on: ubuntu-latest        # Runner type
    
    strategy:
      matrix:                     # Run job for multiple versions
        python-version: [3.10, 3.11, 3.12]
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run tests
        run: pytest tests/ --junitxml=results.xml

      - name: Publish test results
        uses: actions/upload-artifact@v4
        if: always()              # Upload even if tests fail
        with:
          name: test-results-${{ matrix.python-version }}
          path: results.xml

  build:
    name: Build & Push Image
    runs-on: ubuntu-latest
    needs: test                   # Only runs if 'test' job passes
    permissions:
      contents: read
      packages: write
      id-token: write             # For OIDC auth

    outputs:
      image_tag: ${{ steps.meta.outputs.version }}  # Pass to next job

    steps:
      - uses: actions/checkout@v4

      - name: Build and push
        id: meta
        # ... (build steps)

  deploy:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: build
    environment: production       # Requires environment approval
    if: github.ref == 'refs/heads/main'  # Only on main branch

    steps:
      - name: Deploy
        env:
          IMAGE_TAG: ${{ needs.build.outputs.image_tag }}
        run: |
          echo "Deploying image tag: $IMAGE_TAG"
```

### Useful Built-in Variables

```bash
# GitHub Actions context variables
${{ github.sha }}              # Full commit SHA (abc123def...)
${{ github.ref }}              # refs/heads/main or refs/tags/v1.0
${{ github.ref_name }}         # main or v1.0
${{ github.actor }}            # Username who triggered
${{ github.repository }}       # owner/repo
${{ github.run_id }}           # Unique run ID
${{ github.run_number }}       # Sequential run number
${{ github.event_name }}       # push, pull_request, schedule
${{ runner.os }}               # Linux, Windows, macOS
${{ env.MY_VAR }}              # Access env variable set earlier
```

### Conditional Steps

```yaml
# Run only on main branch
if: github.ref == 'refs/heads/main'

# Run even if previous steps failed
if: always()

# Run only if previous step failed
if: failure()

# Run only if step was cancelled
if: cancelled()

# Run on specific event type
if: github.event_name == 'push'

# Run only if file changed
if: contains(github.event.commits[0].modified, 'src/')

# Combined conditions
if: github.ref == 'refs/heads/main' && github.event_name == 'push'
```

---

## 12. Azure DevOps Pipelines — Full Reference

### Pipeline Structure

```yaml
trigger:
  branches:
    include: [main, release/*]
  paths:
    exclude: [docs/**, '*.md']

pool:
  vmImage: ubuntu-latest

variables:
  - group: prod-secrets
  - name: imageTag
    value: $(Build.BuildId)
  - name: dockerRegistry
    value: myacr.azurecr.io

stages:
- stage: CI
  displayName: Build and Test
  jobs:
  - job: Test
    displayName: Run unit tests
    steps:
    - checkout: self
      fetchDepth: 0             # Full git history (needed for SonarQube)

    - task: UsePythonVersion@0
      inputs:
        versionSpec: '3.11'

    - script: |
        pip install -r requirements.txt
        pytest tests/ --junitxml=$(Build.ArtifactStagingDirectory)/results.xml
      displayName: Run Tests

    - task: PublishTestResults@2
      condition: always()
      inputs:
        testResultsFiles: '$(Build.ArtifactStagingDirectory)/results.xml'
        testRunTitle: Unit Tests

  - job: Build
    displayName: Build Docker image
    dependsOn: Test
    steps:
    - task: Docker@2
      displayName: Build and push
      inputs:
        command: buildAndPush
        containerRegistry: myACRServiceConnection
        repository: myapp
        dockerfile: ./Dockerfile
        tags: |
          $(imageTag)
          latest

- stage: DeployStaging
  displayName: Deploy to Staging
  dependsOn: CI
  condition: succeeded()
  jobs:
  - deployment: Staging
    displayName: Deploy to AKS Staging
    environment: staging
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
              namespace: staging
              overrideValues: image.tag=$(imageTag)

- stage: DeployProd
  displayName: Deploy to Production
  dependsOn: DeployStaging
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  jobs:
  - deployment: Production
    displayName: Deploy to AKS Production
    environment: production     # Manual approval gate configured in Azure DevOps UI
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
              namespace: production
              overrideValues: image.tag=$(imageTag)
```

### Useful Azure DevOps Variables

```bash
$(Build.BuildId)               # Unique build number
$(Build.BuildNumber)           # Build number (e.g., 20250713.1)
$(Build.SourceBranch)          # refs/heads/main
$(Build.SourceBranchName)      # main
$(Build.SourceVersion)         # Full commit SHA
$(Build.Repository.Name)       # Repository name
$(Build.ArtifactStagingDirectory)  # Temp folder for artifacts
$(Pipeline.Workspace)          # Root workspace folder
$(Agent.OS)                    # Linux, Windows, Darwin
$(System.DefaultWorkingDirectory)  # Checkout directory
$(System.PullRequest.PullRequestId)  # PR number (if PR trigger)
```

---

*Next → [Part 2: Tech Stack Pipelines, GitOps, Security, Advanced Patterns](./cicd_notes_part2.md)*
