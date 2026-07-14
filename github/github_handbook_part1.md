# 🐙 GitHub — Comprehensive Notes (Part 1)
> Location: trainee-docs/github/
> Part 1 of 2 | Covers: GitHub platform, repos, PRs, branch protection, GitHub CLI, Actions basics

---

## 1. GitHub vs Git

```
┌──────────────────────────────────────────────────────────────┐
│  GIT vs GITHUB                                               │
│                                                              │
│  Git                           GitHub                        │
│  ──────────────────────        ──────────────────────────    │
│  A version control tool        A hosting platform for Git    │
│  Runs on your machine          Runs in the cloud             │
│  No UI (terminal only)         Web UI + API + CLI            │
│  No collaboration features     PRs, Issues, Actions, etc.    │
│  Free, open source             Free tier + paid plans        │
│  Works offline                 Requires internet             │
│                                                              │
│  Analogy:                                                    │
│  Git = the engine              GitHub = the car around it    │
│                                                              │
│  Alternatives to GitHub:                                     │
│  ├── GitLab (self-hosted or cloud, heavy CI/CD)             │
│  ├── Bitbucket (Atlassian, integrates with Jira)            │
│  └── Azure DevOps Repos (Microsoft ecosystem)               │
└──────────────────────────────────────────────────────────────┘
```

---

## 2. GitHub Repository Setup

### Creating and Configuring a Repo

```bash
# Create repo via GitHub CLI
gh repo create my-app --public --clone
gh repo create my-app --private --clone
gh repo create myorg/my-app --private  # org repo

# Clone existing repo
gh repo clone myorg/my-app
git clone https://github.com/myorg/my-app.git
git clone git@github.com:myorg/my-app.git   # SSH

# Fork a repo
gh repo fork owner/repo --clone

# View repo info
gh repo view myorg/my-app
gh repo view myorg/my-app --web   # open in browser

# List repos in org
gh repo list myorg --limit 50

# Archive a repo (read-only)
gh repo archive myorg/old-app
```

### Essential Files Every Repo Needs

```
my-repo/
├── README.md              # Project overview, setup instructions
├── .gitignore             # What NOT to commit
├── LICENSE                # Open source license
├── CONTRIBUTING.md        # How to contribute
├── CODEOWNERS             # Who reviews which files
├── .github/
│   ├── PULL_REQUEST_TEMPLATE.md   # PR description template
│   ├── ISSUE_TEMPLATE/
│   │   ├── bug_report.md
│   │   └── feature_request.md
│   └── workflows/         # GitHub Actions pipelines
│       ├── ci.yml
│       └── cd.yml
├── docs/                  # Documentation
└── src/                   # Source code
```

### .gitignore Best Practices

```gitignore
# Language-specific (Python example)
__pycache__/
*.pyc
*.pyo
.venv/
venv/
*.egg-info/
dist/
build/

# Environment & secrets (NEVER commit these)
.env
.env.local
.env.production
*.pem
*.key
secrets.yaml

# IDE files
.vscode/
.idea/
*.swp
.DS_Store

# Terraform
.terraform/
*.tfstate
*.tfstate.backup
*.tfplan

# Docker
.dockerignore   # this file itself is fine to commit

# Logs
*.log
logs/

# Node
node_modules/
npm-debug.log*
.npm

# Tip: Use gitignore.io to generate for your stack
# curl -sL https://www.toptal.com/developers/gitignore/api/python,node,terraform
```

### CODEOWNERS

```
# .github/CODEOWNERS
# Syntax: <path-pattern> <@user-or-@org/team>

# Default owner for all files
*                           @myorg/platform-team

# Backend team owns all Python files
*.py                        @myorg/backend-team

# Frontend team owns React components
src/components/             @myorg/frontend-team

# Security team must review any auth changes
src/auth/                   @myorg/security-team

# Infrastructure team owns all Terraform
infra/                      @myorg/infra-team
*.tf                        @myorg/infra-team

# Anyone can own docs (no required review)
docs/                       @myorg/docs-team
```

> **Gotcha:** CODEOWNERS only works if branch protection has "Require review from Code Owners" enabled. Without that setting, CODEOWNERS is just documentation.

---

## 3. Pull Requests

### PR Lifecycle

```
┌──────────────────────────────────────────────────────────────┐
│  PULL REQUEST LIFECYCLE                                      │
│                                                              │
│  1. Developer creates feature branch                         │
│     git switch -c feature/add-login                         │
│                                                              │
│  2. Makes commits, pushes branch                            │
│     git push -u origin feature/add-login                    │
│                                                              │
│  3. Opens PR on GitHub                                       │
│     gh pr create --title "feat: add login" --body "..."     │
│                                                              │
│  4. CI pipeline auto-triggers on PR                         │
│     ├── lint checks                                         │
│     ├── unit tests                                          │
│     └── security scans                                      │
│                                                              │
│  5. Code review by teammates / CODEOWNERS                   │
│     ├── Request changes → developer pushes fixes            │
│     └── Approves → proceed to merge                        │
│                                                              │
│  6. All checks pass + required approvals = can merge        │
│                                                              │
│  7. Merge PR (squash / merge commit / rebase)               │
│                                                              │
│  8. Branch auto-deleted (if configured)                     │
│                                                              │
│  9. CD pipeline deploys to staging/prod                     │
└──────────────────────────────────────────────────────────────┘
```

### PR Template

```markdown
<!-- .github/PULL_REQUEST_TEMPLATE.md -->

## Summary
<!-- What does this PR do? Why? -->

## Changes
- [ ] Feature X added
- [ ] Bug Y fixed
- [ ] Tests updated

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update
- [ ] Infrastructure change

## Testing
<!-- How did you test this? -->
- [ ] Unit tests pass locally
- [ ] Tested manually in dev environment
- [ ] Added new tests for this change

## Checklist
- [ ] Code follows project style guidelines
- [ ] Self-reviewed my own code
- [ ] No hardcoded secrets or credentials
- [ ] Documentation updated if needed

## Related Issues
Closes #<issue-number>
```

### GitHub CLI — PR Commands

```bash
# Create a PR
gh pr create \
  --title "feat: add user authentication" \
  --body "Adds JWT-based login and registration" \
  --base main \
  --head feature/add-login \
  --reviewer alice,bob \
  --label "feature,needs-review"

# Create PR and open in browser
gh pr create --web

# List open PRs
gh pr list
gh pr list --state all      # includes merged + closed
gh pr list --author alice

# View a PR
gh pr view 42
gh pr view 42 --web

# Check PR status (CI checks)
gh pr checks 42

# Review a PR
gh pr review 42 --approve
gh pr review 42 --request-changes --body "Please add tests"
gh pr review 42 --comment --body "Looks good, minor nit"

# Checkout a PR locally (to test it)
gh pr checkout 42

# Merge a PR
gh pr merge 42 --squash        # squash commits into one
gh pr merge 42 --merge         # merge commit
gh pr merge 42 --rebase        # rebase and merge

# Close without merging
gh pr close 42

# Reopen a closed PR
gh pr reopen 42

# Add labels to PR
gh pr edit 42 --add-label "priority:high"

# Mark PR as draft
gh pr ready 42 --undo         # convert to draft
gh pr ready 42                # mark as ready for review
```

### Merge Strategies — When to Use Which

```
┌──────────────────────────────────────────────────────────────┐
│  MERGE STRATEGIES IN GITHUB                                  │
│                                                              │
│  1. MERGE COMMIT (default)                                   │
│  Creates a merge commit preserving full branch history       │
│  main: A──B──C──────G   (G = merge commit)                  │
│               └──D──E──F                                     │
│  Use when: you want to see branch history clearly            │
│                                                              │
│  2. SQUASH AND MERGE                                         │
│  Combines all PR commits into ONE commit on main             │
│  main: A──B──C──D'  (D' = squashed version of D+E+F)        │
│  Use when: PR has messy WIP commits, want clean history      │
│  Gotcha: loses individual commit context                     │
│                                                              │
│  3. REBASE AND MERGE                                         │
│  Replays PR commits on top of main (no merge commit)         │
│  main: A──B──C──D'──E'──F'  (linear history)                │
│  Use when: team prefers linear history, clean git log        │
│  Gotcha: rewrites commit hashes (new SHAs)                  │
└──────────────────────────────────────────────────────────────┘
```

---

## 4. Branch Protection Rules

### What They Are and Why They Matter

```
┌──────────────────────────────────────────────────────────────┐
│  BRANCH PROTECTION — STOPS BAD THINGS FROM HAPPENING         │
│                                                              │
│  Without protection on main:                                 │
│  ├── Anyone can push directly to main                       │
│  ├── Force push can rewrite prod history                    │
│  ├── Code can reach production unreviewed                   │
│  └── Broken code can break CI for everyone                  │
│                                                              │
│  With protection on main:                                    │
│  ├── All changes must go through a PR                       │
│  ├── PRs need N approvals before merging                    │
│  ├── CI must pass before merge allowed                      │
│  ├── CODEOWNERS must approve their sections                  │
│  └── No force pushes (history is sacred)                    │
└──────────────────────────────────────────────────────────────┘
```

### Configure via GitHub CLI

```bash
# View protection rules
gh api repos/myorg/my-app/branches/main/protection

# Set branch protection (requires admin)
gh api repos/myorg/my-app/branches/main/protection \
  --method PUT \
  --field required_status_checks='{"strict":true,"contexts":["ci/test","ci/lint"]}' \
  --field enforce_admins=true \
  --field required_pull_request_reviews='{"required_approving_review_count":1,"require_code_owner_reviews":true,"dismiss_stale_reviews":true}' \
  --field restrictions=null \
  --field allow_force_pushes=false \
  --field allow_deletions=false
```

### Recommended Settings for Production Branches

| Setting | Recommended Value | Why |
|---------|------------------|-----|
| Require PR before merge | ✅ ON | No direct pushes to main |
| Required approvals | 1-2 | Human review before merge |
| Dismiss stale reviews | ✅ ON | Re-review after new commits |
| Require CODEOWNERS review | ✅ ON | Domain experts review changes |
| Require status checks | ✅ ON (all CI jobs) | Tests must pass |
| Require up-to-date branch | ✅ ON | Branch must be current with main |
| Include administrators | ✅ ON | Admins follow same rules |
| Allow force pushes | ❌ OFF | Never rewrite shared history |
| Allow deletions | ❌ OFF | Protect main from deletion |

---

## 5. GitHub Issues

```bash
# Create an issue
gh issue create \
  --title "Bug: login fails on Safari" \
  --body "Steps to reproduce..." \
  --label "bug,priority:high" \
  --assignee alice

# List issues
gh issue list
gh issue list --label bug
gh issue list --assignee @me    # your assigned issues

# View issue
gh issue view 15
gh issue view 15 --web

# Close an issue
gh issue close 15
gh issue close 15 --comment "Fixed in PR #42"

# Reopen issue
gh issue reopen 15

# Add comment
gh issue comment 15 --body "Working on this now"

# Link issue to PR (in PR description or commit message)
# Use keywords: Closes #15, Fixes #15, Resolves #15
# GitHub auto-closes the issue when PR is merged
```

---

## 6. GitHub Releases & Tags

```bash
# Create an annotated git tag
git tag -a v1.2.0 -m "Release v1.2.0: add user auth"
git push origin v1.2.0

# Create a GitHub Release
gh release create v1.2.0 \
  --title "v1.2.0 — User Authentication" \
  --notes "## Changes\n- Added JWT login\n- Added registration" \
  --target main

# Create release from tag with auto-generated notes
gh release create v1.2.0 \
  --generate-notes \
  --target main

# Upload release assets (binaries, archives)
gh release create v1.2.0 \
  --notes "Release" \
  ./dist/app-linux-amd64 \
  ./dist/app-darwin-amd64

# List releases
gh release list

# View release
gh release view v1.2.0

# Download release assets
gh release download v1.2.0

# Delete a release
gh release delete v1.2.0

# Draft release (not publicly visible yet)
gh release create v1.3.0 --draft --notes "Coming soon"

# Prerelease (beta)
gh release create v2.0.0-beta.1 --prerelease
```

### Semantic Versioning (SemVer)

```
v MAJOR . MINOR . PATCH
   │        │       │
   │        │       └── Bug fixes, no breaking changes
   │        └─────────── New features, backward compatible
   └──────────────────── Breaking changes

Examples:
v1.0.0     → First stable release
v1.1.0     → Added new feature (no breaking change)
v1.1.1     → Fixed bug in v1.1.0
v2.0.0     → Breaking API change
v2.0.0-beta.1  → Pre-release (not stable)
```

---

## 7. GitHub CLI — Essential Commands

```bash
# Auth
gh auth login                  # Login to GitHub
gh auth login --hostname github.mycompany.com  # Enterprise
gh auth status                 # Check login status
gh auth token                  # Print current token

# Config
gh config set editor vim
gh config set git_protocol ssh
gh config list

# Repos
gh repo create                 # Interactive create
gh repo clone owner/repo
gh repo fork owner/repo --clone
gh repo list                   # Your repos
gh repo list myorg             # Org repos
gh repo view                   # Current repo
gh repo archive owner/repo
gh repo delete owner/repo --confirm

# PRs (see Section 3 above)

# Issues (see Section 5 above)

# Workflows (GitHub Actions)
gh workflow list               # List workflows
gh workflow run ci.yml         # Trigger manually
gh workflow run ci.yml --field environment=staging
gh workflow disable ci.yml
gh workflow enable ci.yml

# Runs
gh run list                    # Recent pipeline runs
gh run list --workflow=ci.yml
gh run view 1234567890         # View specific run
gh run view 1234567890 --log   # View full logs
gh run watch 1234567890        # Watch live
gh run rerun 1234567890        # Re-run
gh run rerun 1234567890 --failed  # Re-run only failed jobs
gh run cancel 1234567890       # Cancel running run
gh run download 1234567890     # Download artifacts

# Secrets
gh secret list                              # List repo secrets
gh secret set MY_SECRET                     # Set secret (prompts)
gh secret set MY_SECRET --body "value"      # Set inline
gh secret set MY_SECRET < ./secret.txt      # Set from file
gh secret delete MY_SECRET
gh secret list --org myorg                  # Org-level secrets
gh secret set DEPLOY_KEY --org myorg --visibility all

# Environments
gh api repos/myorg/my-app/environments      # List environments
```

---

## 8. GitHub Packages & Container Registry (GHCR)

```bash
# Login to GitHub Container Registry
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin

# Tag image for GHCR
docker tag myapp:latest ghcr.io/myorg/myapp:latest
docker tag myapp:v1.0 ghcr.io/myorg/myapp:v1.0

# Push image
docker push ghcr.io/myorg/myapp:v1.0

# Pull image
docker pull ghcr.io/myorg/myapp:v1.0

# In GitHub Actions (automatic auth):
- name: Login to GHCR
  uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}  # No extra secret needed!
```

### GHCR Visibility

```yaml
# Make package public via GitHub API
gh api \
  --method PATCH \
  /orgs/myorg/packages/container/myapp/versions/123 \
  --field visibility=public
```

---

## 9. GitHub Environments

```
┌──────────────────────────────────────────────────────────────┐
│  GITHUB ENVIRONMENTS — DEPLOYMENT GATES                      │
│                                                              │
│  Settings → Environments → New environment                   │
│                                                              │
│  Environment: development                                    │
│  ├── No protection rules                                    │
│  ├── Secrets: DEV_DB_URL, DEV_API_KEY                      │
│  └── Auto-deploy on every push to main                     │
│                                                              │
│  Environment: staging                                        │
│  ├── No reviewers needed                                    │
│  ├── Deployment branches: main only                         │
│  └── Secrets: STAGING_DB_URL                               │
│                                                              │
│  Environment: production                                     │
│  ├── Required reviewers: @alice, @bob (1 must approve)      │
│  ├── Wait timer: 5 minutes                                  │
│  ├── Deployment branches: main only                         │
│  └── Secrets: PROD_DB_URL, PROD_API_KEY                    │
└──────────────────────────────────────────────────────────────┘
```

```yaml
# Reference environment in GitHub Actions
jobs:
  deploy-prod:
    environment: production    # This triggers approval gate
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        env:
          DB_URL: ${{ secrets.PROD_DB_URL }}  # Env-specific secret
        run: ./deploy.sh
```

---

*Next → [Part 2: GitHub Actions deep dive, Security, Webhooks, API](./github_handbook_part2.md)*
