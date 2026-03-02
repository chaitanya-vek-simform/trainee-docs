# Git — Level 1 (🟢 Easy)

## 1. Git Force Push

### Aim

Practice `git push --force` and `--force-with-lease`, understand risks of overwriting remote history and when it is safe to use.

### What is Force Push?

A normal `git push` fails when your local history diverges from the remote. A **force push** overwrites the remote history with your local one — **whether or not someone else pushed changes in the meantime**.

```
Normal push:        Local: A-B-C    Remote: A-B      ✅ Fast-forward
Diverged (fails):   Local: A-B-D    Remote: A-B-C    ❌ Rejected
Force push:         Local: A-B-D    Remote: A-B-C    ⚠️  Overwrites remote with A-B-D (C is LOST)
```

### When to Use

| Scenario | Safe? | Recommendation |
|---|---|---|
| You rebased your own feature branch | ✅ | Use `--force-with-lease` |
| Cleaning up commits before merge | ✅ | Use `--force-with-lease` |
| Pushing to `main`/`production` | ❌ | NEVER force push to shared branches |
| Collaborating with others on same branch | ❌ | Coordinate with team first |

### Step-by-Step Implementation

#### Step 1 — Create a practice repository

```bash
mkdir ~/git-force-push-lab && cd ~/git-force-push-lab
git init
echo "First commit" > file.txt
git add file.txt
git commit -m "Initial commit"
```

#### Step 2 — Create a remote (we'll simulate with a bare repo)

```bash
git clone --bare ~/git-force-push-lab /tmp/remote-repo.git
cd ~/git-force-push-lab
git remote add origin /tmp/remote-repo.git
git push -u origin main
```

#### Step 3 — Make a commit and push normally

```bash
echo "Second line" >> file.txt
git add file.txt
git commit -m "Second commit"
git push origin main
```

#### Step 4 — Simulate diverged history

```bash
echo "Different second line" > file.txt
git add file.txt
git commit --amend -m "Amended second commit"
```

#### Step 5 — Try a normal push (it will FAIL)

```bash
git push origin main
```

**Expected error:**

```
! [rejected]        main -> main (non-fast-forward)
```

#### Step 6 — Force push (DANGEROUS way)

```bash
git push --force origin main
```

> ⚠️ **WARNING:** If a teammate had pulled and was working on "Second commit", their work is now based on a commit that no longer exists!

#### Step 7 — Safe alternative: `--force-with-lease`

```bash
echo "Third line" >> file.txt
git add file.txt
git commit --amend -m "Amended again"
git push --force-with-lease origin main
```

**How `--force-with-lease` works:**
1. It checks: "Is the remote branch still where I think it is?"
2. If YES → proceeds with force push
3. If NO (someone else pushed) → rejects the push to prevent data loss

#### Step 8 — See protection from `--force-with-lease`

```bash
# Simulate another developer pushing
cd /tmp
git clone /tmp/remote-repo.git /tmp/other-dev
cd /tmp/other-dev
echo "Other dev's work" >> file.txt
git add file.txt
git commit -m "Other dev's commit"
git push origin main

# Back to your repo — try force-with-lease WITHOUT fetching first
cd ~/git-force-push-lab
echo "My new work" >> file.txt
git add file.txt
git commit --amend -m "My amended commit"
git push --force-with-lease origin main
```

**Expected:** The push is **REJECTED** because the remote changed.

#### Cleanup

```bash
rm -rf ~/git-force-push-lab /tmp/remote-repo.git /tmp/other-dev
```

---

---

## 2. Pre-Commit Hook

### Aim

Create a `.git/hooks/pre-commit` script to block commits based on custom conditions (lint errors, secret detection, formatting rules).

### What are Git Hooks?

Git hooks are **scripts that run automatically** at certain points in the Git workflow. They live in `.git/hooks/` inside your repository.

| Hook | When it runs |
|---|---|
| `pre-commit` | Before a commit is created (can block the commit) |
| `commit-msg` | After you write a commit message (can validate it) |
| `pre-push` | Before a push (can block the push) |
| `post-merge` | After a merge completes |

### Step-by-Step Implementation

#### Step 1 — Create a practice repository

```bash
mkdir ~/git-hooks-lab && cd ~/git-hooks-lab
git init
```

#### Step 2 — Create a pre-commit hook that blocks secrets

```bash
nano .git/hooks/pre-commit
```

Add this script:

```bash
#!/bin/bash
echo "🔍 Running pre-commit checks..."

SECRETS_PATTERN='(password|secret|api_key|apikey|token|aws_access_key|private_key)\s*=\s*["\x27].+'
STAGED_FILES=$(git diff --cached --name-only --diff-filter=d)

if [ -z "$STAGED_FILES" ]; then
    echo "✅ No files staged. Skipping checks."
    exit 0
fi

SECRETS_FOUND=0
for FILE in $STAGED_FILES; do
    MATCHES=$(git show ":$FILE" 2>/dev/null | grep -inE "$SECRETS_PATTERN")
    if [ -n "$MATCHES" ]; then
        echo "❌ POTENTIAL SECRET FOUND in $FILE:"
        echo "$MATCHES"
        SECRETS_FOUND=1
    fi
done

if [ $SECRETS_FOUND -eq 1 ]; then
    echo "🚫 COMMIT BLOCKED: Potential secrets detected!"
    exit 1
fi

# Check 2: Block large files (> 5MB)
MAX_SIZE=5242880
for FILE in $STAGED_FILES; do
    FILE_SIZE=$(git show ":$FILE" 2>/dev/null | wc -c)
    if [ "$FILE_SIZE" -gt "$MAX_SIZE" ]; then
        echo "❌ FILE TOO LARGE: $FILE"
        exit 1
    fi
done

# Check 3: Warn about committing to main directly
CURRENT_BRANCH=$(git symbolic-ref --short HEAD 2>/dev/null)
if [ "$CURRENT_BRANCH" = "main" ] || [ "$CURRENT_BRANCH" = "master" ]; then
    echo "⚠️  WARNING: You are committing directly to '$CURRENT_BRANCH'."
fi

echo "✅ All pre-commit checks passed!"
exit 0
```

#### Step 3 — Make the hook executable

```bash
chmod +x .git/hooks/pre-commit
```

#### Step 4 — Test with a secret (should be BLOCKED)

```bash
echo 'database_password = "super_secret_123"' > config.txt
git add config.txt
git commit -m "Add config"
# Expected: COMMIT BLOCKED
```

#### Step 5 — Test with a safe file (should PASS)

```bash
git reset HEAD config.txt
echo "Hello World" > safe-file.txt
git add safe-file.txt
git commit -m "Add safe file"
# Expected: All checks passed!
```

#### Step 6 — Share hooks with your team

```bash
mkdir -p .githooks
cp .git/hooks/pre-commit .githooks/pre-commit
git config core.hooksPath .githooks
git add .githooks/
git commit -m "Add shared git hooks"
```

#### Cleanup

```bash
rm -rf ~/git-hooks-lab
```

---

---

## 3. Branch Protection Rules

### Aim

Configure branch protection rules to prevent force pushes, enforce pull request reviews, and require status checks before merging.

### What are Branch Protection Rules?

Branch protection rules are **server-side** settings (configured on GitHub/GitLab) that enforce policies on important branches like `main`.

| Protection | What it prevents |
|---|---|
| **Require pull request reviews** | No direct pushes — changes must go through a PR with approvals |
| **Require status checks** | PRs can't merge unless CI/CD passes |
| **Require signed commits** | Only GPG-signed commits allowed |
| **Block force pushes** | Prevents history rewriting |
| **Block deletions** | Prevents accidental deletion of the branch |

### Step-by-Step Implementation on GitHub

1. Create a test repository on GitHub
2. Clone it locally and create a basic CI workflow:

```yaml
name: CI Check
on:
  pull_request:
    branches: [main]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: echo "All tests passed!"
```

3. Go to **Settings** → **Branches** → **Add branch protection rule**
4. Set branch name pattern: `main`
5. Enable: Require PRs, Require approvals, Require status checks, Block force pushes, Block deletions
6. Test by trying to push directly to main (should be REJECTED)
7. Test the proper workflow via a pull request

---

---

## 4. Pushing Tags to Remote

### Aim

Create lightweight and annotated tags and push them using `git push origin <tag>` or `git push --tags`.

### What are Tags?

| Type | Description | Use Case |
|---|---|---|
| **Lightweight** | Just a pointer to a commit | Quick, temporary markers |
| **Annotated** | Full object with author, date, message | Releases, important milestones |

### Step-by-Step Implementation

```bash
mkdir ~/git-tags-lab && cd ~/git-tags-lab
git init

echo "v1 code" > app.py && git add app.py && git commit -m "Version 1.0 release"
echo "v1.1 bugfix" >> app.py && git add app.py && git commit -m "Fix bug in v1.0"
echo "v2 code" >> app.py && git add app.py && git commit -m "Version 2.0 release"

# Create a lightweight tag
git tag v2.0
git tag v1.0 HEAD~2

# Create annotated tags (recommended for releases)
git tag -a v1.1 HEAD~1 -m "Bugfix release: fixes critical bug in v1.0"

# Push tags
git push origin v2.0         # Single tag
git push --tags               # ALL tags
git push --follow-tags        # Only annotated tags

# Delete a tag
git tag -d v2.0               # Local
git push origin --delete v2.0 # Remote

# Checkout a tag
git checkout v1.0             # Detached HEAD state
git checkout main             # Back to main
```

#### Cleanup

```bash
rm -rf ~/git-tags-lab
```

---

---

## 5. Cherry-Pick Across Branches

### Aim

Copy specific commits from one branch to another without merging the entire branch.

### When to Use

- Applying a hotfix from `main` to a `release` branch
- Grabbing a specific feature commit without merging a whole branch
- Porting a bugfix to an older version

### Step-by-Step Implementation

```bash
mkdir ~/git-cherry-lab && cd ~/git-cherry-lab
git init
echo "Base code" > app.py && git add app.py && git commit -m "Initial commit"

# Create a feature branch with several commits
git checkout -b feature/many-changes
echo "Feature A" > feature-a.py && git add . && git commit -m "Add feature A"
echo "Critical bugfix" >> app.py && git add . && git commit -m "Fix critical bug in app.py"
echo "Feature B" > feature-b.py && git add . && git commit -m "Add feature B"

# Find the commit to cherry-pick
git log --oneline feature/many-changes

# Cherry-pick the bugfix to main
git checkout main
git cherry-pick <bugfix-commit-hash>

# Cherry-pick multiple commits
git cherry-pick <hash1> <hash2>

# Handle conflicts during cherry-pick
git add <resolved-file>
git cherry-pick --continue
# Or abort:
git cherry-pick --abort
```

#### Cleanup

```bash
rm -rf ~/git-cherry-lab
```

---

---

## 6. Resolving Merge Conflicts

### Aim

Understand what merge conflicts are, why they happen, and practice resolving them.

### What Causes a Merge Conflict?

A conflict occurs when **two branches modify the same line** in the same file.

### Step-by-Step Implementation

```bash
mkdir ~/git-conflict-lab && cd ~/git-conflict-lab
git init

cat > app.py << 'EOF'
def greet(name):
    return f"Hello, {name}"

def farewell(name):
    return f"Goodbye, {name}"
EOF
git add app.py && git commit -m "Initial code"

# Branch A changes the greeting
git checkout -b branch-a
sed -i 's/Hello/Hey there/' app.py
git add app.py && git commit -m "Branch A: casual greeting"

# Branch B also changes the greeting (differently)
git checkout main
git checkout -b branch-b
sed -i 's/Hello/Welcome/' app.py
git add app.py && git commit -m "Branch B: formal greeting"

# Merge and trigger conflict
git checkout main
git merge branch-a
git merge branch-b    # THIS CAUSES A CONFLICT!
```

#### Understanding the conflict markers

```python
<<<<<<< HEAD
    return f"Hey there, {name}"
=======
    return f"Welcome, {name}"
>>>>>>> branch-b
```

| Marker | Meaning |
|---|---|
| `<<<<<<< HEAD` | Start of YOUR current branch's version |
| `=======` | Separator |
| `>>>>>>> branch-b` | End of the INCOMING branch's version |

#### Resolve: Edit the file, remove markers, then:

```bash
git add app.py
git commit -m "Merge branch-b: combine greeting styles"
```

#### Useful conflict resolution commands

```bash
git diff --name-only --diff-filter=U    # See conflicting files
git mergetool                           # Visual merge tool
git merge --abort                       # Abort the merge entirely
git checkout --theirs app.py            # Take their version
git checkout --ours app.py              # Keep your version
```

#### Cleanup

```bash
rm -rf ~/git-conflict-lab
```

---

---

## 7. Git Stash — Save Work Without Committing

### Aim

Temporarily save uncommitted changes so you can switch branches or do other work, then restore them later.

### Step-by-Step Implementation

```bash
mkdir ~/git-stash-lab && cd ~/git-stash-lab
git init
echo "Main code" > app.py && git add app.py && git commit -m "Initial commit"

# Start working on something
echo "Work in progress..." >> app.py
echo "New file" > wip.txt
git add wip.txt

# Stash your work
git stash push -m "WIP: feature X progress"
git status    # Clean!

# Do other work...
git checkout -b hotfix/urgent
echo "Urgent fix" >> app.py
git add app.py && git commit -m "Apply urgent hotfix"
git checkout main && git merge hotfix/urgent

# Restore stashed work
git stash list
git stash pop    # Restore and remove from stack

# Advanced stash operations
git stash push --keep-index -m "Only unstaged changes"
git stash push --include-untracked -m "Include new files"
git stash apply stash@{0}          # Apply without removing
git stash show -p stash@{0}        # View stash diff
git stash drop stash@{1}           # Drop a specific stash
git stash branch feature/from-stash stash@{0}  # Create branch from stash
```

#### Cleanup

```bash
rm -rf ~/git-stash-lab
```
