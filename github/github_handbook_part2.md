# 🐙 GitHub — Comprehensive Notes (Part 2)
> Part 2 of 2 | Covers: GitHub Actions deep dive, Security, Webhooks, API, Best Practices

---

## 10. GitHub Actions — Deep Dive

### Core Concepts

```
┌──────────────────────────────────────────────────────────────┐
│  GITHUB ACTIONS — COMPONENT HIERARCHY                        │
│                                                              │
│  Workflow (.github/workflows/ci.yml)                         │
│  └── Triggered by: push, PR, schedule, manual               │
│      │                                                       │
│      ├── Job 1: test                                        │
│      │   ├── runs-on: ubuntu-latest                         │
│      │   ├── Step 1: Checkout code                          │
│      │   ├── Step 2: Setup Python                           │
│      │   ├── Step 3: Run tests                              │
│      │   └── Step 4: Upload coverage report                 │
│      │                                                       │
│      ├── Job 2: lint (runs in parallel with test)           │
│      │   └── Steps...                                       │
│      │                                                       │
│      └── Job 3: deploy (needs: [test, lint])                │
│          └── Steps...                                       │
│                                                              │
│  Jobs run in PARALLEL by default.                           │
│  Use 'needs' to create dependencies between jobs.           │
│  Steps within a job run SEQUENTIALLY.                       │
└──────────────────────────────────────────────────────────────┘
```

### Workflow Syntax — Complete Reference

```yaml
name: Full Workflow Example

# ─── TRIGGERS ────────────────────────────────────────────────
on:
  push:
    branches: [main, 'release/**']
    tags: ['v*.*.*']
    paths-ignore: ['**.md', 'docs/**']

  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]  # default

  schedule:
    - cron: '0 2 * * 1-5'   # 2 AM Mon-Fri UTC

  workflow_dispatch:          # Manual trigger
    inputs:
      environment:
        description: Target environment
        required: true
        type: choice
        options: [dev, staging, prod]
        default: staging
      dry_run:
        type: boolean
        default: false

  workflow_call:              # Called by another workflow
    inputs:
      image_tag:
        type: string
        required: true
    secrets:
      DEPLOY_KEY:
        required: true

# ─── GLOBAL SETTINGS ─────────────────────────────────────────
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true   # Cancel older runs of same branch

permissions:                  # Least privilege at workflow level
  contents: read
  packages: write
  id-token: write

env:
  REGISTRY: ghcr.io
  NODE_VERSION: '20'

# ─── JOBS ────────────────────────────────────────────────────
jobs:
  test:
    name: Unit Tests
    runs-on: ubuntu-latest

    # Matrix: run job for multiple configs simultaneously
    strategy:
      fail-fast: false         # Don't cancel others if one fails
      matrix:
        python: ['3.10', '3.11', '3.12']
        os: [ubuntu-latest, macos-latest]

    # Override global permissions at job level
    permissions:
      contents: read

    # Set timeout to prevent runaway jobs
    timeout-minutes: 30

    # Environment variables for this job
    env:
      PYTHONDONTWRITEBYTECODE: 1

    # Services (sidecar containers — e.g. test DB)
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      # ── Step types ──────────────────────────────────────────

      # 1. Run shell commands
      - name: Print info
        run: |
          echo "Branch: ${{ github.ref_name }}"
          echo "SHA: ${{ github.sha }}"
          python --version

      # 2. Use a pre-built action
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0        # Full history (needed for SonarQube)

      # 3. Conditional step
      - name: Only on main
        if: github.ref == 'refs/heads/main'
        run: echo "This is main!"

      # 4. Continue on error
      - name: Non-critical step
        continue-on-error: true
        run: some-optional-command

      # 5. Step with id (to reference output later)
      - name: Get version
        id: version
        run: echo "tag=$(git describe --tags)" >> $GITHUB_OUTPUT

      - name: Use version
        run: echo "Version is ${{ steps.version.outputs.tag }}"

      # 6. Upload artifact
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results-${{ matrix.python }}
          path: test-results.xml
          retention-days: 7

  deploy:
    needs: [test]              # Wait for test job
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    environment: production    # Requires environment approval

    steps:
      # Download artifact from test job
      - uses: actions/download-artifact@v4
        with:
          name: test-results-3.11

      # Pass data between jobs using outputs
      - name: Deploy
        env:
          VERSION: ${{ needs.test.outputs.version }}
        run: echo "Deploying $VERSION"
```

### GitHub Actions — Expressions & Contexts

```yaml
# ─── CONTEXTS (objects available in workflows) ───────────────

# github context — info about the trigger event
${{ github.sha }}                # Full 40-char commit SHA
${{ github.ref }}                # refs/heads/main
${{ github.ref_name }}           # main
${{ github.ref_type }}           # branch or tag
${{ github.event_name }}         # push, pull_request, schedule
${{ github.actor }}              # Who triggered (username)
${{ github.repository }}         # owner/repo
${{ github.repository_owner }}   # owner
${{ github.run_id }}             # Unique run number
${{ github.run_number }}         # Sequential number
${{ github.server_url }}         # https://github.com
${{ github.api_url }}            # https://api.github.com
${{ github.workspace }}          # Path to checked-out code

# env context — environment variables
${{ env.MY_VARIABLE }}

# secrets context — secrets (masked in logs)
${{ secrets.MY_SECRET }}
${{ secrets.GITHUB_TOKEN }}      # Auto-provided, no setup needed

# steps context — outputs from previous steps in same job
${{ steps.my-step-id.outputs.result }}
${{ steps.my-step-id.outcome }}  # success, failure, skipped, cancelled

# needs context — outputs from upstream jobs
${{ needs.build.outputs.image_tag }}
${{ needs.build.result }}        # success, failure, etc.

# runner context
${{ runner.os }}                 # Linux, Windows, macOS
${{ runner.arch }}               # X64, ARM64
${{ runner.temp }}               # Temp directory
${{ runner.tool_cache }}         # Tool cache directory

# matrix context
${{ matrix.python-version }}

# ─── EXPRESSIONS ─────────────────────────────────────────────
# Operators
${{ 1 + 1 }}                     # Math
${{ 'hello' == 'hello' }}        # Comparison
${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}

# Functions
${{ contains(github.ref, 'release') }}
${{ startsWith(github.ref, 'refs/tags/v') }}
${{ endsWith(github.ref, 'main') }}
${{ format('Hello {0}!', github.actor) }}
${{ join(matrix.python, ', ') }}
${{ toJSON(github.event) }}
${{ fromJSON(steps.parse.outputs.json).key }}
${{ hashFiles('**/package-lock.json') }}

# Status functions (for if: conditions)
${{ success() }}     # Previous steps succeeded
${{ failure() }}     # A previous step failed
${{ cancelled() }}   # Workflow was cancelled
${{ always() }}      # Always run
```

### Setting Outputs & Environment Variables

```yaml
steps:
  # ── Set step output (pass to next steps in same job) ────────
  - name: Set image tag
    id: tag
    run: |
      TAG="sha-$(git rev-parse --short HEAD)"
      echo "image_tag=$TAG" >> $GITHUB_OUTPUT

  - name: Use output
    run: echo "Tag is ${{ steps.tag.outputs.image_tag }}"

  # ── Set env variable (available to later steps in same job) ──
  - name: Set version
    run: |
      echo "APP_VERSION=1.2.3" >> $GITHUB_ENV

  - name: Use env var
    run: echo "Version: $APP_VERSION"

  # ── Add to PATH ──────────────────────────────────────────────
  - name: Add custom tool to PATH
    run: echo "/opt/my-tool/bin" >> $GITHUB_PATH

  # ── Set job summary (appears in Actions UI) ──────────────────
  - name: Write summary
    run: |
      echo "## Test Results 🧪" >> $GITHUB_STEP_SUMMARY
      echo "| Test | Status |" >> $GITHUB_STEP_SUMMARY
      echo "|------|--------|" >> $GITHUB_STEP_SUMMARY
      echo "| Unit | ✅ Pass |" >> $GITHUB_STEP_SUMMARY
```

### Reusable Workflows

```yaml
# ─── REUSABLE WORKFLOW (.github/workflows/deploy.yml) ────────
on:
  workflow_call:
    inputs:
      environment:
        type: string
        required: true
      image_tag:
        type: string
        required: true
    secrets:
      KUBE_CONFIG:
        required: true
    outputs:
      deployment_url:
        value: ${{ jobs.deploy.outputs.url }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    outputs:
      url: ${{ steps.deploy.outputs.url }}
    steps:
      - name: Deploy
        id: deploy
        env:
          KUBECONFIG_DATA: ${{ secrets.KUBE_CONFIG }}
        run: |
          echo "$KUBECONFIG_DATA" | base64 -d > kubeconfig
          helm upgrade --install myapp ./charts/myapp \
            --set image.tag=${{ inputs.image_tag }}
          echo "url=https://app-${{ inputs.environment }}.example.com" >> $GITHUB_OUTPUT

# ─── CALLER WORKFLOW ─────────────────────────────────────────
jobs:
  deploy-staging:
    uses: ./.github/workflows/deploy.yml
    with:
      environment: staging
      image_tag: sha-abc1234
    secrets:
      KUBE_CONFIG: ${{ secrets.STAGING_KUBE_CONFIG }}

  deploy-prod:
    needs: deploy-staging
    uses: ./.github/workflows/deploy.yml
    with:
      environment: production
      image_tag: sha-abc1234
    secrets:
      KUBE_CONFIG: ${{ secrets.PROD_KUBE_CONFIG }}
```

### Composite Actions (Custom Reusable Steps)

```yaml
# .github/actions/setup-python-env/action.yml
name: Setup Python Environment
description: Install Python and dependencies with caching

inputs:
  python-version:
    description: Python version to use
    required: false
    default: '3.11'
  requirements-file:
    description: Path to requirements file
    required: false
    default: requirements.txt

outputs:
  cache-hit:
    description: Whether cache was restored
    value: ${{ steps.cache.outputs.cache-hit }}

runs:
  using: composite
  steps:
    - uses: actions/setup-python@v5
      with:
        python-version: ${{ inputs.python-version }}

    - uses: actions/cache@v4
      id: cache
      with:
        path: ~/.cache/pip
        key: ${{ runner.os }}-pip-${{ hashFiles(inputs.requirements-file) }}

    - name: Install dependencies
      shell: bash
      run: pip install -r ${{ inputs.requirements-file }}

# Use the composite action:
steps:
  - uses: ./.github/actions/setup-python-env
    with:
      python-version: '3.12'
```

---

## 11. GitHub Security Features

### Secret Scanning & Push Protection

```
┌──────────────────────────────────────────────────────────────┐
│  SECRET SCANNING                                             │
│  Settings → Code security → Secret scanning                  │
│                                                              │
│  GitHub automatically scans for:                            │
│  ├── AWS access keys (AKIA...)                              │
│  ├── Azure storage keys                                     │
│  ├── GitHub tokens (ghp_...)                                │
│  ├── Stripe API keys                                        │
│  └── 200+ other secret patterns                             │
│                                                              │
│  PUSH PROTECTION (prevent secrets from entering):           │
│  If push contains a secret pattern → push is BLOCKED        │
│  Developer must either:                                      │
│  ├── Remove the secret from code                            │
│  ├── Mark as false positive                                 │
│  └── Request bypass (logged and audited)                    │
│                                                              │
│  Even if you delete the file — secrets in git history       │
│  are still exposed. Use 'git filter-repo' to remove.        │
└──────────────────────────────────────────────────────────────┘
```

### Dependabot

```yaml
# .github/dependabot.yml
version: 2
updates:
  # Python dependencies
  - package-ecosystem: pip
    directory: /
    schedule:
      interval: weekly
      day: monday
      time: "09:00"
      timezone: "Asia/Kolkata"
    open-pull-requests-limit: 5
    labels:
      - dependencies
      - python
    reviewers:
      - myorg/backend-team
    ignore:
      - dependency-name: "boto3"
        versions: ["1.30.x"]  # Skip specific version

  # Node.js dependencies
  - package-ecosystem: npm
    directory: /frontend
    schedule:
      interval: weekly
    groups:
      react-packages:
        patterns:
          - "react*"
          - "@types/react*"

  # Docker base images
  - package-ecosystem: docker
    directory: /
    schedule:
      interval: monthly

  # GitHub Actions versions
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
```

### Code Scanning (CodeQL)

```yaml
# .github/workflows/codeql.yml
name: CodeQL Security Analysis

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 6 * * 1'   # Weekly Monday 6 AM

jobs:
  analyze:
    name: Analyze
    runs-on: ubuntu-latest
    permissions:
      security-events: write
      contents: read

    strategy:
      matrix:
        language: [python, javascript]  # or: go, java, ruby, swift

    steps:
    - uses: actions/checkout@v4

    - name: Initialize CodeQL
      uses: github/codeql-action/init@v3
      with:
        languages: ${{ matrix.language }}
        queries: security-and-quality   # or: security-extended

    - name: Autobuild
      uses: github/codeql-action/autobuild@v3

    - name: Perform CodeQL Analysis
      uses: github/codeql-action/analyze@v3
      with:
        category: "/language:${{ matrix.language }}"
```

---

## 12. GitHub Webhooks

```
┌──────────────────────────────────────────────────────────────┐
│  WEBHOOKS — GitHub calls YOUR server on events              │
│                                                              │
│  GitHub (event happens)                                      │
│       │ HTTP POST with JSON payload                         │
│       ▼                                                      │
│  Your server / Lambda / Azure Function                      │
│  ├── Verify HMAC signature (secret)                         │
│  ├── Parse event type from X-GitHub-Event header            │
│  └── Take action (trigger Terraform, notify, etc.)          │
│                                                              │
│  Common webhook events:                                      │
│  ├── push          → code pushed to branch                  │
│  ├── pull_request  → PR opened/closed/merged                │
│  ├── release       → new release published                  │
│  ├── issues        → issue opened/closed                    │
│  ├── workflow_run  → Actions run completed                  │
│  └── check_run     → CI check completed                     │
└──────────────────────────────────────────────────────────────┘
```

```bash
# Create webhook via GitHub CLI
gh api repos/myorg/my-app/hooks \
  --method POST \
  --field name=web \
  --field active=true \
  --field "events[]=push" \
  --field "events[]=pull_request" \
  --field "config[url]=https://my-server.com/webhook" \
  --field "config[content_type]=json" \
  --field "config[secret]=mywebhooksecret"

# List webhooks
gh api repos/myorg/my-app/hooks

# Test webhook (redeliver)
gh api repos/myorg/my-app/hooks/123456/deliveries \
  --method GET
```

```python
# Verify webhook signature in Python
import hmac, hashlib

def verify_signature(payload: bytes, secret: str, signature: str) -> bool:
    expected = "sha256=" + hmac.new(
        secret.encode(), payload, hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)

# In Flask handler:
@app.route('/webhook', methods=['POST'])
def handle_webhook():
    sig = request.headers.get('X-Hub-Signature-256', '')
    if not verify_signature(request.data, WEBHOOK_SECRET, sig):
        return 'Unauthorized', 401

    event = request.headers.get('X-GitHub-Event')
    payload = request.json

    if event == 'push' and payload['ref'] == 'refs/heads/main':
        trigger_deployment()

    return 'OK', 200
```

---

## 13. GitHub API

```bash
# GitHub CLI wraps the API — use 'gh api' for direct API calls

# Get repo info
gh api repos/myorg/my-app

# List commits on main
gh api repos/myorg/my-app/commits?sha=main&per_page=10

# Get file content
gh api repos/myorg/my-app/contents/README.md \
  --jq '.content' | base64 -d

# Update a file
CONTENT=$(echo "new content" | base64)
SHA=$(gh api repos/myorg/my-app/contents/file.txt --jq '.sha')
gh api repos/myorg/my-app/contents/file.txt \
  --method PUT \
  --field message="Update file" \
  --field content="$CONTENT" \
  --field sha="$SHA"

# Trigger a workflow
gh api repos/myorg/my-app/actions/workflows/ci.yml/dispatches \
  --method POST \
  --field ref=main \
  --field "inputs[environment]=staging"

# List workflow runs
gh api repos/myorg/my-app/actions/runs \
  --jq '.workflow_runs[] | {id, status, conclusion, created_at}'

# Get PR diff
gh api repos/myorg/my-app/pulls/42 \
  --header "Accept: application/vnd.github.v3.diff"

# Search issues
gh api search/issues \
  --field q="repo:myorg/my-app is:issue is:open label:bug" \
  --jq '.items[] | {number, title}'

# Add comment to PR
gh api repos/myorg/my-app/issues/42/comments \
  --method POST \
  --field body="Automated comment from pipeline ✅"
```

---

## 14. GitHub Pages

```bash
# Enable GitHub Pages (serve static site from branch)
gh api repos/myorg/my-app/pages \
  --method POST \
  --field source='{"branch":"gh-pages","path":"/"}'

# Or serve from /docs folder on main branch
gh api repos/myorg/my-app/pages \
  --method POST \
  --field source='{"branch":"main","path":"/docs"}'

# Deploy to GitHub Pages via Actions
# .github/workflows/pages.yml
name: Deploy to Pages
on:
  push:
    branches: [main]
permissions:
  contents: read
  pages: write
  id-token: write
jobs:
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build docs
        run: mkdocs build --site-dir ./site
      - uses: actions/upload-pages-artifact@v3
        with:
          path: ./site
      - uses: actions/deploy-pages@v4
        id: deployment
```

---

## 15. Best Practices — GitHub

```
┌──────────────────────────────────────────────────────────────┐
│  GITHUB BEST PRACTICES                                       │
│                                                              │
│  Repos:                                                      │
│  ├── Always have README.md, .gitignore, LICENSE             │
│  ├── One repo per service/app (not mega-monolith)           │
│  ├── Archive repos instead of deleting                      │
│  └── Use topics/tags for discoverability                    │
│                                                              │
│  Branches:                                                   │
│  ├── Protect main + release/* branches always               │
│  ├── Delete merged branches automatically                   │
│  ├── Use short-lived feature branches (<1 week ideal)       │
│  └── Name branches: feature/, fix/, chore/, docs/           │
│                                                              │
│  Pull Requests:                                              │
│  ├── Keep PRs small (<400 lines of code)                    │
│  ├── Always use PR template                                 │
│  ├── Link PRs to issues (Closes #N)                        │
│  ├── Review your own PR before requesting review            │
│  └── Never merge your own PR in production repos            │
│                                                              │
│  Security:                                                   │
│  ├── Enable secret scanning + push protection               │
│  ├── Enable Dependabot + auto-merge for patches             │
│  ├── Enable CodeQL for all production repos                 │
│  ├── Use OIDC instead of stored secrets for cloud auth      │
│  ├── Rotate PATs (personal access tokens) regularly         │
│  └── Prefer fine-grained PATs over classic PATs             │
│                                                              │
│  GitHub Actions:                                             │
│  ├── Pin action versions to full SHA (not @v3)              │
│  │   BAD:  uses: actions/checkout@v4                        │
│  │   GOOD: uses: actions/checkout@11bd71901bbe5b1630ceea73d │
│  ├── Set minimum required permissions per workflow           │
│  ├── Never print secrets (even masked, avoid echo)          │
│  ├── Use concurrency to cancel duplicate runs               │
│  ├── Set timeout-minutes on all jobs                        │
│  └── Prefer GITHUB_TOKEN over PATs when possible           │
└──────────────────────────────────────────────────────────────┘
```

---

## 16. Quick Reference Card

```
TASK                              COMMAND
────────────────────────────────────────────────────────────
Create & clone repo               gh repo create myapp --private --clone
Create PR                         gh pr create --title "feat: X" --base main
Checkout PR locally               gh pr checkout 42
Approve PR                        gh pr review 42 --approve
Merge PR (squash)                 gh pr merge 42 --squash
Trigger workflow manually         gh workflow run ci.yml --field env=staging
Watch pipeline run                gh run watch
View run logs                     gh run view --log
Re-run failed jobs only           gh run rerun --failed
Set a secret                      gh secret set MY_KEY --body "value"
View all secrets                  gh secret list
Create a release                  gh release create v1.0.0 --generate-notes
View open issues                  gh issue list
Create issue                      gh issue create --title "Bug: X" --label bug
Search code in repo               gh search code "function login" --repo myorg/app
Call GitHub API                   gh api repos/myorg/app/commits --jq '.[0].sha'
Add webhook                       gh api repos/myorg/app/hooks --method POST ...
Check auth status                 gh auth status
Open repo in browser              gh repo view --web
Open PR in browser                gh pr view 42 --web
```

---

*← [Part 1: Repos, PRs, Branch Protection, CLI](./github_handbook_part1.md)*
