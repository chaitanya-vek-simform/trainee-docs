# Git — Level 3 (🔴 Hard)

## 1. Finding Bug-Introducing Commit (git bisect)

### Aim

Use `git bisect` to perform a binary search in commit history to identify the commit that introduced a bug.

### How It Works

Instead of checking every commit one by one, `git bisect` does a **binary search** — finds the bug in $O(\log n)$ steps instead of $O(n)$.

```
100 commits: Manual = up to 100 checks | Bisect = ~7 checks (log₂ 100 ≈ 7)
```

### Step-by-Step Implementation

#### Step 1 — Create a repo with a hidden bug

```bash
mkdir ~/git-bisect-lab && cd ~/git-bisect-lab
git init

echo "def calculate(x): return x * 2" > app.py
git add app.py && git commit -m "Commit 1: initial function"
echo "def greet(): return 'hello'" >> app.py
git add app.py && git commit -m "Commit 2: add greet"
echo "def validate(x): return x > 0" >> app.py
git add app.py && git commit -m "Commit 3: add validate"

# THE BUG IS INTRODUCED HERE
sed -i 's/return x \* 2/return x \* 0/' app.py
git add app.py && git commit -m "Commit 4: refactor calculate"

echo "def helper(): pass" >> app.py && git add app.py && git commit -m "Commit 5: add helper"
echo "# TODO: add tests" >> app.py && git add app.py && git commit -m "Commit 6: add todo"
echo "def logger(): print('log')" >> app.py && git add app.py && git commit -m "Commit 7: add logger"
echo "VERSION = '1.0'" >> app.py && git add app.py && git commit -m "Commit 8: add version"
```

#### Step 2 — Start bisecting

```bash
git bisect start
git bisect bad                # Current commit is BAD
git bisect good HEAD~7        # Commit 1 was GOOD

# Git checks out a middle commit. Test it:
python3 -c "exec(open('app.py').read()); print(calculate(5))"

# If output is 0: git bisect bad
# If output is 10: git bisect good
# Repeat until Git finds the exact bad commit
```

#### Step 3 — Automate bisect with a test script

```bash
cat > /tmp/test-bug.sh << 'EOF'
#!/bin/bash
RESULT=$(python3 -c "exec(open('app.py').read()); print(calculate(5))")
[ "$RESULT" = "10" ] && exit 0 || exit 1
EOF
chmod +x /tmp/test-bug.sh

git bisect start
git bisect bad HEAD
git bisect good HEAD~7
git bisect run /tmp/test-bug.sh    # Fully automated!
```

#### Step 4 — End bisecting

```bash
git bisect reset
```

#### Cleanup

```bash
rm -rf ~/git-bisect-lab
```

---

---

## 2. Repository Migration with Full History

### Aim

Migrate repositories using mirror clone while preserving full commit history, branches, and tags.

### Step-by-Step Implementation

#### Step 1 — Create a source repo with history

```bash
mkdir ~/git-migration-source && cd ~/git-migration-source
git init

echo "Initial code" > app.py && git add app.py && git commit -m "Initial commit"
git checkout -b develop
echo "Dev feature" >> app.py && git add app.py && git commit -m "Add dev feature"
git checkout -b feature/auth
echo "Auth module" >> app.py && git add app.py && git commit -m "Add auth module"
git checkout main
git tag -a v1.0 -m "Version 1.0 release"

git clone --bare ~/git-migration-source /tmp/source-remote.git
cd ~/git-migration-source
git remote add origin /tmp/source-remote.git
git push --all origin && git push --tags origin
```

#### Step 2 — Mirror clone (captures EVERYTHING)

```bash
cd /tmp
git clone --mirror /tmp/source-remote.git migration-mirror.git
```

| Feature | `--bare` | `--mirror` |
|---|---|---|
| All branches | ✅ | ✅ |
| All tags | ✅ | ✅ |
| All refs (notes, stashes) | ❌ | ✅ |
| Remote tracking | ❌ | ✅ |

#### Step 3 — Push to new destination

```bash
cd /tmp/migration-mirror.git
git remote set-url origin <NEW_DESTINATION_URL>
git push --mirror origin
```

#### Step 4 — Verify

```bash
git clone <NEW_DESTINATION_URL> /tmp/verify-migration
cd /tmp/verify-migration
git branch -a && git tag -l && git log --oneline --all --graph
```

#### Cross-platform migration (e.g., GitLab → GitHub)

```bash
git clone --mirror https://gitlab.com/old-org/old-repo.git
cd old-repo.git
git remote set-url origin https://github.com/new-org/new-repo.git
git push --mirror origin

# If repo has Git LFS:
git lfs fetch --all && git lfs push --all origin
```

#### Cleanup

```bash
rm -rf ~/git-migration-source /tmp/source-remote.git /tmp/migration-mirror.git /tmp/verify-migration
```

---

---

## 3. Cron — Auto-Pull & Change Logging

### Aim

Write a cron job script to check for remote updates, auto-pull changes, log activity, and detect merge conflicts.

### Step-by-Step Implementation

#### Step 1 — Create the script

```bash
sudo nano /usr/local/bin/git-auto-pull.sh
```

```bash
#!/bin/bash
# Git Auto-Pull Script — Checks for remote changes and pulls automatically

REPO_DIR="/home/$USER/Documents/projects/my-repo"
BRANCH="main"
LOG_FILE="/var/log/git-auto-pull.log"
CONFLICT_LOG="/var/log/git-auto-pull-conflicts.log"

log_message() {
    echo "[$(date "+%Y-%m-%d %H:%M:%S")] $1" >> "$LOG_FILE"
}

log_conflict() {
    echo "[$(date "+%Y-%m-%d %H:%M:%S")] $1" >> "$CONFLICT_LOG"
}

[ ! -d "$REPO_DIR/.git" ] && { log_message "ERROR: Not a git repo"; exit 1; }
cd "$REPO_DIR" || exit 1

CURRENT_BRANCH=$(git symbolic-ref --short HEAD 2>/dev/null)
[ "$CURRENT_BRANCH" != "$BRANCH" ] && { log_message "WARNING: On '$CURRENT_BRANCH', expected '$BRANCH'"; exit 0; }

git fetch origin "$BRANCH" 2>> "$LOG_FILE" || { log_message "ERROR: Fetch failed"; exit 1; }

LOCAL_HASH=$(git rev-parse HEAD)
REMOTE_HASH=$(git rev-parse "origin/$BRANCH")

if [ "$LOCAL_HASH" = "$REMOTE_HASH" ]; then
    log_message "NO CHANGE: Up to date ($LOCAL_HASH)"
    exit 0
fi

log_message "CHANGES DETECTED: Local ($LOCAL_HASH) vs Remote ($REMOTE_HASH)"
git log --oneline "$LOCAL_HASH..$REMOTE_HASH" >> "$LOG_FILE" 2>&1

# Stash uncommitted changes
STASHED=false
if ! git diff --quiet || ! git diff --cached --quiet; then
    log_message "WARNING: Stashing local changes"
    git stash push -m "Auto-stash $(date '+%Y%m%d_%H%M%S')" >> "$LOG_FILE" 2>&1
    STASHED=true
fi

# Attempt pull
PULL_OUTPUT=$(git pull origin "$BRANCH" 2>&1)
if [ $? -eq 0 ]; then
    log_message "PULL SUCCESS: Updated to $(git rev-parse HEAD)"
    [ "$STASHED" = true ] && git stash pop >> "$LOG_FILE" 2>&1
else
    log_message "PULL FAILED: Merge conflict!"
    log_conflict "MERGE CONFLICT — Files: $(git diff --name-only --diff-filter=U)"
    git merge --abort 2>> "$LOG_FILE"
    log_conflict "Steps: cd $REPO_DIR && git pull origin $BRANCH && resolve conflicts"
fi

log_message "--- End of check ---"
```

#### Step 2 — Set up

```bash
sudo chmod +x /usr/local/bin/git-auto-pull.sh
sudo touch /var/log/git-auto-pull.log /var/log/git-auto-pull-conflicts.log
sudo chown $USER:$USER /var/log/git-auto-pull.log /var/log/git-auto-pull-conflicts.log
```

#### Step 3 — Schedule with cron (every 5 minutes)

```bash
crontab -e
# Add: */5 * * * * /usr/local/bin/git-auto-pull.sh
```

---

---

## 4. Cron — Merged Branch Cleanup

### Aim

Automate deletion of merged branches starting with `temp/` using a scheduled cleanup script.

### Step-by-Step Implementation

```bash
sudo nano /usr/local/bin/git-cleanup-temp-branches.sh
```

```bash
#!/bin/bash
# Deletes merged branches starting with 'temp/'

REPO_DIR="/home/$USER/Documents/projects/my-repo"
MAIN_BRANCH="main"
LOG_FILE="/var/log/git-branch-cleanup.log"
DRY_RUN=false

log_message() {
    echo "[$(date "+%Y-%m-%d %H:%M:%S")] $1" >> "$LOG_FILE"
}

[ ! -d "$REPO_DIR/.git" ] && { log_message "ERROR: Not a git repo"; exit 1; }
cd "$REPO_DIR" || exit 1

git checkout "$MAIN_BRANCH" >> "$LOG_FILE" 2>&1
git fetch --prune origin >> "$LOG_FILE" 2>&1
log_message "Starting branch cleanup..."
DELETED_COUNT=0

# Clean LOCAL merged temp/ branches
for BRANCH in $(git branch --merged "$MAIN_BRANCH" | grep -E '^\s+temp/' | sed 's/^\s*//'); do
    if [ "$DRY_RUN" = true ]; then
        log_message "[DRY RUN] Would delete: $BRANCH"
    else
        git branch -d "$BRANCH" >> "$LOG_FILE" 2>&1 && {
            log_message "DELETED local: $BRANCH"
            DELETED_COUNT=$((DELETED_COUNT + 1))
        }
    fi
done

# Clean REMOTE merged temp/ branches
for BRANCH in $(git branch -r --merged "$MAIN_BRANCH" | grep -E 'origin/temp/' | sed 's/^\s*origin\///'); do
    if [ "$DRY_RUN" = true ]; then
        log_message "[DRY RUN] Would delete remote: $BRANCH"
    else
        git push origin --delete "$BRANCH" >> "$LOG_FILE" 2>&1 && {
            log_message "DELETED remote: origin/$BRANCH"
            DELETED_COUNT=$((DELETED_COUNT + 1))
        }
    fi
done

log_message "Cleanup complete. Deleted $DELETED_COUNT branch(es)."
```

#### Schedule (daily at 2 AM)

```bash
sudo chmod +x /usr/local/bin/git-cleanup-temp-branches.sh
crontab -e
# Add: 0 2 * * * /usr/local/bin/git-cleanup-temp-branches.sh
```

---

---

## 5. Cron — Repository Backup (7-Day Retention)

### Aim

Create a scheduled backup script that archives the repository daily and maintains 7-day retention.

### Step-by-Step Implementation

```bash
sudo nano /usr/local/bin/git-repo-backup.sh
```

```bash
#!/bin/bash
# Git Repository Backup — Daily compressed backups with 7-day retention

REPO_DIR="/home/$USER/Documents/projects/my-repo"
BACKUP_DIR="/home/$USER/backups/git-repos"
REPO_NAME=$(basename "$REPO_DIR")
RETENTION_DAYS=7
LOG_FILE="/var/log/git-backup.log"
DATE_STAMP=$(date "+%Y%m%d_%H%M%S")

log_message() {
    echo "[$(date "+%Y-%m-%d %H:%M:%S")] $1" >> "$LOG_FILE"
}

log_message "Starting backup for $REPO_NAME..."
mkdir -p "$BACKUP_DIR"

[ ! -d "$REPO_DIR/.git" ] && { log_message "ERROR: Not a git repo"; exit 1; }
cd "$REPO_DIR" || exit 1

git fetch --all >> "$LOG_FILE" 2>&1

# Create bundle backup
BUNDLE_FILE="$BACKUP_DIR/${REPO_NAME}_backup_${DATE_STAMP}.bundle"
git bundle create "$BUNDLE_FILE" --all >> "$LOG_FILE" 2>&1

if [ $? -eq 0 ]; then
    gzip "$BUNDLE_FILE"
    FINAL_BACKUP="${BUNDLE_FILE}.gz"
    log_message "BACKUP SUCCESS: $FINAL_BACKUP ($(du -h "$FINAL_BACKUP" | cut -f1))"
else
    # Fallback to tar
    BACKUP_FILE="$BACKUP_DIR/${REPO_NAME}_backup_${DATE_STAMP}.tar.gz"
    tar -czf "$BACKUP_FILE" -C "$(dirname "$REPO_DIR")" "$REPO_NAME" 2>> "$LOG_FILE"
    FINAL_BACKUP="$BACKUP_FILE"
    log_message "BACKUP SUCCESS (tar): $FINAL_BACKUP"
fi

# Delete old backups
DELETED_COUNT=0
while IFS= read -r OLD_BACKUP; do
    [ -n "$OLD_BACKUP" ] && rm -f "$OLD_BACKUP" && {
        log_message "DELETED old: $OLD_BACKUP"
        DELETED_COUNT=$((DELETED_COUNT + 1))
    }
done < <(find "$BACKUP_DIR" -name "${REPO_NAME}_backup_*" -type f -mtime +${RETENTION_DAYS})

log_message "Deleted $DELETED_COUNT old backup(s). Total: $(find "$BACKUP_DIR" -name "${REPO_NAME}_backup_*" | wc -l) backups"
```

#### Restore from backup

```bash
# From a bundle:
gunzip backup.bundle.gz
git clone backup.bundle /tmp/restored-repo

# From a tar:
tar -xzf backup.tar.gz -C /tmp/
```

#### Schedule (daily at 3 AM)

```bash
sudo chmod +x /usr/local/bin/git-repo-backup.sh
mkdir -p ~/backups/git-repos
crontab -e
# Add: 0 3 * * * /usr/local/bin/git-repo-backup.sh
```

---

---

## 6. Git Worktree — Multiple Working Directories

### Aim

Work on multiple branches simultaneously without stashing or switching, using separate working directories linked to the same repository.

### When to Use

- Reviewing a PR while still coding on your branch
- Running tests on one branch while developing on another
- Comparing code across branches side by side

### Step-by-Step Implementation

```bash
mkdir ~/git-worktree-lab && cd ~/git-worktree-lab
git init
echo "Main branch code" > app.py && git add app.py && git commit -m "Initial commit"

git checkout -b feature/new-ui
echo "New UI code" > ui.py && git add ui.py && git commit -m "Add new UI"
git checkout -b bugfix/login-error
echo "Login fix" > login.py && git add login.py && git commit -m "Fix login error"
git checkout main

# Create worktrees for other branches
git worktree add ../worktree-feature feature/new-ui
git worktree add ../worktree-bugfix bugfix/login-error

# Now you have three directories:
# ~/git-worktree-lab/  → main
# ~/worktree-feature/  → feature/new-ui
# ~/worktree-bugfix/   → bugfix/login-error

# Work in all three simultaneously
cd ~/git-worktree-lab && echo "Main update" >> app.py && git add . && git commit -m "Update main"
cd ~/worktree-feature && echo "UI improvement" >> ui.py && git add . && git commit -m "Improve UI"
cd ~/worktree-bugfix && echo "Better fix" >> login.py && git add . && git commit -m "Improve login fix"

# List all worktrees
cd ~/git-worktree-lab && git worktree list

# Remove a worktree when done
git worktree remove ../worktree-bugfix
git worktree prune    # Clean up if directory was manually deleted
```

#### Cleanup

```bash
cd ~
git -C ~/git-worktree-lab worktree remove ~/worktree-feature 2>/dev/null
git -C ~/git-worktree-lab worktree remove ~/worktree-bugfix 2>/dev/null
rm -rf ~/git-worktree-lab ~/worktree-feature ~/worktree-bugfix
```

---

---

## 7. Quick Reference — Essential Git Commands

| Category | Command | What it does |
|---|---|---|
| **Setup** | `git config --global user.name "Name"` | Set your name |
| | `git config --global user.email "email"` | Set your email |
| **Basics** | `git init` | Initialize a new repo |
| | `git clone <url>` | Clone a remote repo |
| | `git status` | See current state |
| | `git add .` | Stage all changes |
| | `git commit -m "msg"` | Commit staged changes |
| **Branching** | `git branch` | List branches |
| | `git checkout -b <name>` | Create and switch to new branch |
| | `git merge <branch>` | Merge branch into current |
| **Remote** | `git push origin <branch>` | Push to remote |
| | `git pull origin <branch>` | Pull from remote |
| | `git fetch --all` | Fetch all remotes |
| **History** | `git log --oneline --graph` | Visual commit history |
| | `git reflog` | All HEAD movements |
| | `git diff` | Show unstaged changes |
| **Undo** | `git reset --soft HEAD~1` | Undo last commit (keep staged) |
| | `git reset --hard HEAD~1` | Undo last commit (discard all) |
| | `git revert <hash>` | Create undo commit |
| **Advanced** | `git rebase -i HEAD~N` | Interactive rebase |
| | `git cherry-pick <hash>` | Copy a specific commit |
| | `git bisect start` | Binary search for bugs |
| | `git stash` | Save uncommitted work |
| | `git stash pop` | Restore saved work |
