[🏠 Home](../README.md) · [Edge Cases](.)

# 🔀 Git — Edge Cases (110)

> **Audience:** DevOps engineers, developers, and SREs working with Git in production CI/CD pipelines.  
> **Coverage:** Repository corruption, merge nightmares, rebase traps, submodule pitfalls, hook failures, LFS quirks, and CI/CD integration surprises.

---

## Table of Contents

1. [Repository & Object Storage (EC-001–015)](#repository--object-storage)
2. [Branching & Merging (EC-016–025)](#branching--merging)
3. [Rebase & History Rewriting (EC-026–035)](#rebase--history-rewriting)
4. [Remote Operations & Authentication (EC-036–045)](#remote-operations--authentication)
5. [Submodules & Subtrees (EC-046–055)](#submodules--subtrees)
6. [Hooks & Automation (EC-056–065)](#hooks--automation)
7. [Large Files & Performance (EC-066–075)](#large-files--performance)
8. [Workflow & Collaboration (EC-076–085)](#workflow--collaboration)
9. [CI/CD Integration (EC-086–095)](#cicd-integration)
10. [Recovery & Disaster (EC-096–105)](#recovery--disaster)
11. [Advanced / Internals (EC-106–110)](#advanced--internals)
12. [Production FAQ](#production-faq)

---

## Repository & Object Storage

### EC-001 — `.git` Directory Accidentally Deleted — Entire History Lost
**Category:** Repository | **Severity:** Critical | **Env:** Dev/Prod

**Scenario:** Developer runs `rm -rf .git` thinking it just deletes "tracking info". Entire repository history, branches, stash, and reflog are gone. Working directory files remain but without any version control.

```
  .git/ structure:
  ├── HEAD          ← current branch pointer
  ├── config        ← remote URLs, settings
  ├── objects/      ← ALL commits, trees, blobs (your entire history!)
  ├── refs/         ← branch and tag pointers
  ├── hooks/        ← pre-commit, post-receive scripts
  └── logs/         ← reflog (undo history)
  
  rm -rf .git = delete ALL of the above = no recovery without remote
```

**Fix:**
```bash
# If remote exists, re-clone:
git clone <remote-url> .
# Copy your uncommitted working files back into the fresh clone

# If NO remote:
# Check if any backup, Time Machine, or filesystem snapshots exist
# .git objects are NOT recoverable without backup
```

> ⚠️ **DevOps Gotcha:** Add `.git` to your backup inclusion list. CI/CD runners that clean workspaces with `rm -rf *` instead of `git clean -fdx` can delete `.git` accidentally if glob expansion matches dotfiles.

---

### EC-002 — Repository Size Explodes After Committing Large Binary
**Category:** Repository | **Severity:** High | **Env:** Dev/Prod

**Scenario:** Developer commits a 500MB database dump. Realizes mistake, deletes file, commits deletion. Repository is STILL 500MB because Git stores the blob in history forever.

**Detection:**
```bash
# Find large objects in repository:
git rev-list --objects --all | \
  git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' | \
  sort -k3 -rn | head -20

# Or use git-sizer:
git-sizer --verbose
```

**Fix:**
```bash
# Remove from all history with git filter-repo (recommended over filter-branch):
pip install git-filter-repo
git filter-repo --invert-paths --path huge-dump.sql
# Force push to remote (coordinate with team!):
git push --force --all
git push --force --tags
```

> ⚠️ **DevOps Gotcha:** After filter-repo, ALL team members must re-clone. Their local repos have the old history. `git pull` will NOT remove the large blob from their local `.git/objects`. This is the single most disruptive Git operation you can do to a team.

---

### EC-003 — Shallow Clone Missing History Breaks `git log` and `git blame`
**Category:** Repository | **Severity:** Medium | **Env:** Prod (CI/CD)

**Scenario:** CI pipeline uses `git clone --depth 1` for speed. Build passes but `git log --oneline` shows only 1 commit. `git blame` fails. Changelog generation script produces empty output.

```
  Full clone: commit A → B → C → D → E (HEAD)
  Shallow (depth=1):              E (HEAD) ← no parent history!
  
  What breaks:
  - git log (shows 1 commit)
  - git blame (incomplete or fails)
  - git describe (can't find tags)
  - merge-base calculations (can't find common ancestor)
```

**Fix:**
```bash
# Unshallow when you need history:
git fetch --unshallow
# Or clone with enough depth for your needs:
git clone --depth 50 <repo>
# For git describe to work, fetch tags:
git fetch --tags
```

> ⚠️ **DevOps Gotcha:** GitHub Actions uses `actions/checkout` with `fetch-depth: 1` by default. Set `fetch-depth: 0` for full history if your pipeline needs `git log`, `git describe`, or semantic versioning tools.

---

### EC-004 — Corrupt Pack File After Disk Full During `git gc`
**Category:** Repository | **Severity:** Critical | **Env:** Dev/Prod

**Scenario:** `git gc` runs (manually or auto). Disk fills during pack file generation. Pack file is incomplete. Subsequent git operations fail with "bad object" or "packfile corrupt".

**Detection:**
```bash
git fsck --full   # checks object integrity
# Look for: "error in tree", "broken link", "bad object"
```

**Fix:**
```bash
# If remote is healthy:
mv .git/objects/pack .git/objects/pack.bak
git fetch origin
git fsck --full

# Last resort — re-clone from remote:
cd .. && mv repo repo.bak && git clone <remote-url> repo
```

---

### EC-005 — `.gitignore` Doesn't Ignore Already-Tracked Files
**Category:** Repository | **Severity:** High | **Env:** Dev/Prod

**Scenario:** Add `config/secrets.yml` to `.gitignore`. File still shows up in `git status`. Because the file was ALREADY tracked, `.gitignore` has no effect on tracked files.

```
  .gitignore only affects UNTRACKED files:
  ┌──────────────────────────────────────────────────┐
  │  Tracked file + .gitignore rule = STILL tracked  │
  │  Untracked file + .gitignore rule = Ignored ✅   │
  └──────────────────────────────────────────────────┘
```

**Fix:**
```bash
# Remove from tracking but keep on disk:
git rm --cached config/secrets.yml
git commit -m "Stop tracking secrets.yml"
# Now .gitignore will take effect
# The file remains in working directory but is no longer tracked
```

> ⚠️ **DevOps Gotcha:** The file still exists in git history! Anyone can see it by checking old commits. If it contained secrets, rotate them immediately. Then use `git filter-repo` to purge from history.

---

### EC-006 — `git clean -fdx` Deletes Untracked IDE Config and Build Artifacts
**Category:** Repository | **Severity:** Medium | **Env:** Dev

**Scenario:** Developer runs `git clean -fdx` to get a pristine workspace. `-x` flag removes even `.gitignore`-d files. IDE settings (`.idea/`, `.vscode/`), `node_modules/`, build caches — all deleted.

**Fix:**
```bash
# Preview what would be deleted FIRST:
git clean -fdxn   # -n = dry run
# To keep certain ignored files:
git clean -fd     # without -x, keeps .gitignore'd files
# Or use exclusion:
git clean -fdx -e .idea/ -e .vscode/
```

---

### EC-007 — Object Database Corruption After Force Power-Off
**Category:** Repository | **Severity:** High | **Env:** Dev

**Scenario:** Laptop battery dies during `git commit` or `git push`. Loose objects partially written. Next `git status` shows "fatal: bad object HEAD" or "error: object file is empty".

**Fix:**
```bash
# Find and remove corrupt objects:
git fsck --full 2>&1 | grep "corrupt\|missing\|bad"
# Remove broken loose objects:
find .git/objects -size 0 -delete
# Reset HEAD to last known good:
git reflog   # find last good commit
git reset --hard <good-commit>
```

---

### EC-008 — Repo Contains Sensitive Data in Object Store Even After Delete
**Category:** Repository | **Severity:** Critical | **Env:** Prod

**Scenario:** AWS credentials committed, then deleted in next commit. `git log --all -S "AKIA"` still finds them. `git show <old-commit>:path/to/file` exposes them.

**Detection:**
```bash
# Search entire history for secrets:
git log --all -S "AKIA" --oneline
# Or use trufflehog / gitleaks:
gitleaks detect --source=. --verbose
trufflehog git file://. --since-commit HEAD~100
```

> ⚠️ **DevOps Gotcha:** GitHub automatically scans for known secret patterns (GitHub Secret Scanning). AWS rotates exposed keys when detected. But your repo is cloned on N developer machines — ALL of them have the secret in their local history.

---

### EC-009 — `git gc --aggressive` Takes Hours on Large Repo
**Category:** Repository | **Severity:** Medium | **Env:** Dev/Prod

**Scenario:** Developer reads "gc --aggressive optimizes repo". Runs it on a 10GB monorepo. Process takes 4+ hours and locks the repository for the duration.

**Detection:**
```bash
git count-objects -v   # check pack size and loose objects
# If pack size > 5GB, gc --aggressive will be slow
```

> ⚠️ **DevOps Gotcha:** `git gc --aggressive` repacks ALL objects with maximum compression. For large repos, use `git repack -a -d --depth=50 --window=250` with controlled parameters instead. Or schedule gc during off-hours.

---

### EC-010 — Alternates Object Store Breaks After Path Change
**Category:** Repository | **Severity:** High | **Env:** Prod (CI/CD)

**Scenario:** CI system uses `objects/info/alternates` to share object store between builds. CI workspace path changes (new agent, new directory). Alternates path is now invalid. All fetches re-download everything.

**Detection:**
```bash
cat .git/objects/info/alternates
# If path doesn't exist → alternates broken → slow clones
```

---

### EC-011 — `git archive` Doesn't Include Submodule Contents
**Category:** Repository | **Severity:** Medium | **Env:** Prod (CI/CD)

**Scenario:** `git archive HEAD | tar -x -C /deploy` used for deployment. Submodule directories exist but are empty. `git archive` archives only the main repository, not submodules.

**Fix:**
```bash
# Archive with submodules manually:
git submodule foreach 'git archive HEAD --prefix=$sm_path/ | tar -x -C /deploy'
# Or use rsync-based deployment instead of git archive
```

---

### EC-012 — `.git/config` Contains Credentials in Plaintext
**Category:** Security | **Severity:** Critical | **Env:** Dev/Prod

**Scenario:** `git remote set-url origin https://user:password@github.com/org/repo.git`. Password stored in `.git/config` in plaintext. Anyone with read access to the filesystem can extract it.

**Fix:**
```bash
# Use credential helper instead:
git config --global credential.helper store   # stores in ~/.git-credentials (still plaintext)
git config --global credential.helper cache --timeout=3600  # in-memory for 1 hour
# Best: use SSH keys or credential manager:
git config --global credential.helper /usr/lib/git-core/git-credential-libsecret
```

> ⚠️ **DevOps Gotcha:** In CI/CD, use short-lived tokens (GitHub App installation tokens, GitLab CI_JOB_TOKEN). Never embed long-lived PATs in repository URLs.

---

### EC-013 — Bare Repository Cannot `git checkout` or Show Working Tree
**Category:** Repository | **Severity:** Low | **Env:** Prod

**Scenario:** Server-side Git repo (e.g., GitLab/GitHub self-hosted) is a bare repo. Admin tries `git checkout main` to inspect files. "fatal: this operation must be run in a work tree".

**Fix:**
```bash
# View files without checkout:
git show main:path/to/file
git ls-tree -r main --name-only
# Or create a temporary work tree:
git worktree add /tmp/inspection main
```

---

### EC-014 — `git init` in Home Directory Tracks Everything
**Category:** Repository | **Severity:** Medium | **Env:** Dev

**Scenario:** Developer accidentally runs `git init` in `~`. Now `git status` from any subdirectory shows thousands of untracked files (entire home directory).

**Detection:**
```bash
git rev-parse --show-toplevel   # shows repo root
# If it shows /home/user → accidental init in home dir
```

**Fix:**
```bash
rm -rf ~/.git   # removes the accidental repo (no data lost — it was empty)
```

---

### EC-015 — `core.autocrlf` Causes Phantom File Changes on Cross-Platform Teams
**Category:** Repository | **Severity:** High | **Env:** Dev

**Scenario:** Windows developers have `core.autocrlf=true`. Linux/Mac developers have `core.autocrlf=input`. Every `git diff` shows entire files changed because line endings differ.

```
  Windows (CRLF):   Hello\r\n
  Linux/Mac (LF):   Hello\n
  
  With autocrlf mismatch:
  Windows dev commits → CRLF stored
  Linux dev pulls → sees LF, converts → diff shows every line changed!
```

**Fix:**
```bash
# Create .gitattributes in repo root (overrides local settings):
* text=auto
*.sh text eol=lf
*.bat text eol=crlf
*.png binary
*.jpg binary

# Normalize existing files:
git add --renormalize .
git commit -m "Normalize line endings"
```

> ⚠️ **DevOps Gotcha:** `.gitattributes` committed to the repo is the ONLY reliable way to handle line endings. Per-developer `core.autocrlf` settings are fragile and inconsistent. Always commit `.gitattributes`.

---

## Branching & Merging

### EC-016 — Merge Conflict in Binary Files — No Visual Diff Possible
**Category:** Merge | **Severity:** High | **Env:** Dev

**Scenario:** Two developers modify the same Excel file / Photoshop file / compiled binary. Git can't auto-merge binary files and can't show a meaningful diff.

```
  Git merge with binary conflict:
  <<<<<<< HEAD
  (binary content — unreadable)
  =======
  (binary content — unreadable)
  >>>>>>> feature-branch
  
  You MUST choose one version entirely — can't merge parts.
```

**Fix:**
```bash
# Choose ours (current branch version):
git checkout --ours path/to/file.xlsx
# Choose theirs (incoming branch version):
git checkout --theirs path/to/file.xlsx
git add path/to/file.xlsx
git commit
```

> ⚠️ **DevOps Gotcha:** Use Git LFS for binary files with file-level locking (`git lfs lock file.xlsx`). This prevents concurrent edits on un-mergeable files. Alternatively, don't version-control binaries — use artifact repositories (Nexus, Artifactory).

---

### EC-017 — Fast-Forward Merge Loses Merge Context
**Category:** Merge | **Severity:** Medium | **Env:** Dev

**Scenario:** Feature branch merged with fast-forward. History shows a linear sequence — no visible merge point. Can't tell when the feature was integrated or revert the feature as a unit.

```
  Fast-forward:           No-fast-forward:
  A → B → C → D (main)   A → B --------→ M (main)
  (linear, no merge         \           /
   commit visible)            C → D → E (feature)
                           Merge commit M = clear integration point
```

**Fix:**
```bash
# Force merge commit even when fast-forward is possible:
git merge --no-ff feature-branch
# In GitHub: disable "Allow squash merging" if you want merge commits
# In GitLab: set merge method to "Merge commit" not "Fast-forward"
```

---

### EC-018 — Merge Commit Reverted — But Revert Prevents Future Merge
**Category:** Merge | **Severity:** High | **Env:** Dev/Prod

**Scenario:** Feature branch merged to main. Bug found, merge commit reverted. Later, bug fixed on feature branch. Try to merge again — Git says "Already up to date" because those commits are already in main's history (just reverted).

```
  A → B → M (merge) → R (revert of M) → main
       \         /
        C → D → E (feature)
  
  Now: git merge feature → "Already up to date"
  Because C, D, E are already reachable from main (through M, even though R reverted the changes)
```

**Fix:**
```bash
# Revert the revert:
git revert <revert-commit-hash>
# This re-applies the original merge changes
# Then merge any new commits from feature branch
```

> ⚠️ **DevOps Gotcha:** This is one of the most confusing Git scenarios. "Revert the revert" is the standard solution. Alternatively, cherry-pick individual commits from the feature branch instead of re-merging.

---

### EC-019 — Octopus Merge Fails With More Than 2 Conflicting Branches
**Category:** Merge | **Severity:** Medium | **Env:** Dev

**Scenario:** Trying to merge 3+ branches simultaneously: `git merge branch-a branch-b branch-c`. If ANY conflict exists between any pair, the entire octopus merge fails.

**Fix:**
```bash
# Merge one at a time:
git merge branch-a
git merge branch-b
git merge branch-c
# Resolve conflicts incrementally
```

---

### EC-020 — `git merge --squash` Doesn't Create Merge Relationship
**Category:** Merge | **Severity:** Medium | **Env:** Dev

**Scenario:** `git merge --squash feature` creates a single commit with all feature changes. But Git doesn't record a merge relationship. Future `git merge feature` will try to re-merge the same changes (conflicts likely).

```
  Regular merge:          Squash merge:
  main → A → M            main → A → S (squashed)
          \ /                        (no merge link!)
  feat → B → C            feat → B → C (still diverged)
  
  After squash: feature branch still looks unmerged to Git
```

> ⚠️ **DevOps Gotcha:** After squash merge, DELETE the source branch. Never try to re-merge a squash-merged branch. GitHub/GitLab "Squash and merge" button handles this correctly by deleting the branch.

---

### EC-021 — Merge Strategy `ours` Silently Discards Incoming Changes
**Category:** Merge | **Severity:** High | **Env:** Dev

**Scenario:** `git merge -s ours hotfix-branch` merges but keeps ONLY the current branch's content. All changes from `hotfix-branch` are silently discarded. Merge commit is created but content is unchanged.

```
  -s ours:   keeps current branch content ONLY (discards theirs entirely)
  -X ours:   auto-resolves conflicts by keeping ours (keeps non-conflicting theirs)
  
  These are VERY different! -s ours is nuclear, -X ours is targeted.
```

---

### EC-022 — Deleted Branch Re-Appears After Merge From Stale Clone
**Category:** Merge | **Severity:** Medium | **Env:** Dev

**Scenario:** Branch `feature/old` deleted on remote. Developer with stale clone pushes their local `feature/old` to remote. Branch re-created with old commits.

**Fix:**
```bash
# Always fetch and prune before pushing:
git fetch --prune
# Or set auto-prune:
git config --global fetch.prune true
```

---

### EC-023 — Merge Conflict Markers Left in Code — Passes Linter, Breaks Runtime
**Category:** Merge | **Severity:** Critical | **Env:** Prod

**Scenario:** Developer resolves merge conflict but accidentally leaves `<<<<<<<`, `=======`, `>>>>>>>` markers in a JSON/YAML file. Linter doesn't catch them (they're valid strings). Application fails at runtime parsing the config.

**Detection:**
```bash
# Add to pre-commit hook or CI:
grep -rn "^<<<<<<<\|^=======\|^>>>>>>>" --include="*.yml" --include="*.json" .
# Or use git's built-in:
git diff --check   # detects leftover conflict markers
```

> ⚠️ **DevOps Gotcha:** Add a CI step that searches for conflict markers in ALL files: `grep -rn "^<<<<<<<" . && exit 1`. This is cheap insurance against a surprisingly common mistake.

---

### EC-024 — `rerere` Cache Applies Wrong Resolution to Similar Conflict
**Category:** Merge | **Severity:** Medium | **Env:** Dev

**Scenario:** `git config rerere.enabled true` caches conflict resolutions. A structurally similar but semantically different conflict in another file gets the cached (wrong) resolution applied silently.

**Fix:**
```bash
# Forget a specific rerere resolution:
git rerere forget path/to/file
# Or clear all cached resolutions:
rm -rf .git/rr-cache/*
```

---

### EC-025 — Merge Commit Has Different Content Than Either Parent
**Category:** Merge | **Severity:** High | **Env:** Dev

**Scenario:** During manual conflict resolution, developer accidentally introduces new code (or deletes code) that wasn't in either branch. The merge commit contains "evil merge" changes not visible in normal `git log`.

**Detection:**
```bash
# Show only changes introduced during merge (not from either parent):
git show --diff-filter=M <merge-commit> --cc
# Or use:
git diff <merge-commit>^1 <merge-commit>
git diff <merge-commit>^2 <merge-commit>
```

---

## Rebase & History Rewriting

### EC-026 — `git rebase` on Shared Branch Causes Duplicate Commits
**Category:** Rebase | **Severity:** Critical | **Env:** Dev

**Scenario:** Developer rebases a branch that others are working on. Force-pushes. Other developers pull and see duplicate commits — their local copies have old commit hashes, remote has new ones.

```
  Before rebase:
  main: A → B → C
  feat: A → B → D → E (shared with teammate)
  
  After your rebase + force push:
  feat: A → B → C → D' → E' (new hashes!)
  
  Teammate pulls:
  feat: A → B → D → E → D' → E' (DUPLICATES!)
```

> ⚠️ **DevOps Gotcha:** NEVER rebase branches that other people are working on. The golden rule: only rebase YOUR OWN local branches that haven't been pushed (or have been pushed but nobody else has pulled).

---

### EC-027 — Interactive Rebase `fixup` Drops Important Commit Message
**Category:** Rebase | **Severity:** Medium | **Env:** Dev

**Scenario:** `git rebase -i` with `fixup` (not `squash`). `fixup` discards the commit message of the fixup commit. If the fixup had an important note, it's lost silently.

```
  squash:  combines commits, lets you EDIT the combined message
  fixup:   combines commits, DISCARDS the fixup commit's message
  fixup -C: combines commits, KEEPS the fixup commit's message (drops original)
```

---

### EC-028 — Rebase Conflict on Every Commit in Long Branch
**Category:** Rebase | **Severity:** High | **Env:** Dev

**Scenario:** Feature branch has 50 commits. `git rebase main` replays each commit. Same area of code modified in many commits — you get the SAME conflict over and over for each commit.

**Fix:**
```bash
# Use rerere to cache resolutions:
git config rerere.enabled true
git rebase main   # resolve first occurrence — rerere caches it

# Or squash before rebase to reduce conflicts:
git rebase -i HEAD~50   # squash into fewer commits first
git rebase main          # fewer commits = fewer potential conflicts
```

---

### EC-029 — `git commit --amend` Changes Commit Hash Even With Same Content
**Category:** Rebase | **Severity:** Medium | **Env:** Dev

**Scenario:** `git commit --amend` to fix a typo in commit message. Even though file content is identical, the commit hash changes (commit includes timestamp and message in hash). Force push required.

> ⚠️ **DevOps Gotcha:** `--amend` on the last commit is safe if you haven't pushed. If you HAVE pushed, you must `--force-push` — which affects anyone who has pulled that commit.

---

### EC-030 — `git filter-branch` Is Deprecated and Unreliable
**Category:** History Rewriting | **Severity:** High | **Env:** Dev/Prod

**Scenario:** Using `git filter-branch --env-filter` to change author email across all commits. Filter-branch is slow, error-prone, and creates backup refs that bloat the repo.

**Fix:**
```bash
# Use git-filter-repo instead (official recommendation):
pip install git-filter-repo
# Change author email:
git filter-repo --mailmap mailmap.txt
# mailmap.txt: Old Name <old@email.com> New Name <new@email.com>
```

---

### EC-031 — `git reset --hard` Deletes Uncommitted Work Permanently
**Category:** History Rewriting | **Severity:** Critical | **Env:** Dev

**Scenario:** Developer has unstaged changes. Runs `git reset --hard HEAD` expecting it to "undo last commit". All uncommitted changes in working directory are deleted permanently — not recoverable via reflog (reflog tracks commits, not working directory).

```
  reset --soft:  moves HEAD, keeps staged + working dir
  reset --mixed: moves HEAD, unstages files, keeps working dir
  reset --hard:  moves HEAD, unstages files, DELETES working dir changes!
```

> ⚠️ **DevOps Gotcha:** `git stash` before `git reset --hard`. Or use `git checkout -- .` to discard only working directory changes without moving HEAD. Reflog CANNOT recover uncommitted changes.

---

### EC-032 — Force Push Overwrites Teammate's Commits on Shared Branch
**Category:** History Rewriting | **Severity:** Critical | **Env:** Dev/Prod

**Scenario:** Developer force-pushes to `develop` branch after rebase. Teammate had pushed 3 commits that were not in the force-push. Those 3 commits are now unreachable on remote.

**Fix:**
```bash
# Use --force-with-lease instead of --force:
git push --force-with-lease
# This fails if remote has commits you haven't fetched
# Prevents accidentally overwriting others' work

# Server-side protection:
# GitHub: branch protection → "Restrict force pushes"
# GitLab: "Protected branches" → disallow force push
```

> ⚠️ **DevOps Gotcha:** Enable branch protection on ALL shared branches (main, develop, release/*). Disallow force push. Require PR reviews. This is the single most important Git repository setting for team safety.

---

### EC-033 — `git reflog` Entries Expire — Old Recovery Points Lost
**Category:** History Rewriting | **Severity:** Medium | **Env:** Dev

**Scenario:** Developer needs to recover a commit from 3 months ago via reflog. Reflog entries expire after 90 days (default for reachable) or 30 days (unreachable). Recovery point is gone.

**Detection:**
```bash
git reflog --date=iso   # check dates of reflog entries
git config gc.reflogExpire          # default: 90 days
git config gc.reflogExpireUnreachable  # default: 30 days
```

**Fix:**
```bash
# Extend reflog retention:
git config gc.reflogExpire 365.days
git config gc.reflogExpireUnreachable 180.days
```

---

### EC-034 — Rebasing Merge Commits Flattens Branch Topology
**Category:** Rebase | **Severity:** Medium | **Env:** Dev

**Scenario:** Branch has merge commits from pulling upstream. `git rebase main` linearizes history — all merge commits become regular commits. Branch topology and integration points are lost.

**Fix:**
```bash
# Preserve merge commits during rebase:
git rebase --rebase-merges main
# (replaces deprecated --preserve-merges)
```

---

### EC-035 — `git cherry-pick` Creates Duplicate Commit With Different Hash
**Category:** History Rewriting | **Severity:** Medium | **Env:** Dev

**Scenario:** Cherry-pick commit `abc123` from `feature` to `main`. Later, merge `feature` into `main`. Git sees two copies of the same change (different hashes). May or may not conflict depending on context.

```
  Original: feature: A → B → C (cherry-pick C to main)
  Main after cherry-pick: X → Y → C' (different hash, same diff)
  Main after merge: X → Y → C' → M (merge of feature)
  
  Git usually handles this cleanly, but:
  - If C was later modified on feature → conflict
  - git log shows "duplicate" looking commits
```

---

## Remote Operations & Authentication

### EC-036 — HTTPS Push Prompts for Password Every Time
**Category:** Remote | **Severity:** Low | **Env:** Dev

**Scenario:** Using HTTPS remote URL. Every `git push` asks for username/password. Developer types credentials 20 times/day.

**Fix:**
```bash
# Use credential cache (in-memory, expires):
git config --global credential.helper 'cache --timeout=86400'
# Or use system keychain:
# macOS: git config --global credential.helper osxkeychain
# Linux: git config --global credential.helper libsecret
# Windows: git config --global credential.helper wincred

# Best: switch to SSH:
git remote set-url origin git@github.com:org/repo.git
```

---

### EC-037 — SSH Key Authentication Fails Due to Agent Not Running
**Category:** Remote | **Severity:** Medium | **Env:** Dev

**Scenario:** `git push` fails with "Permission denied (publickey)". SSH key exists in `~/.ssh/` but `ssh-agent` is not running or key is not added.

**Fix:**
```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
# Verify:
ssh -T git@github.com
# Add to ~/.bashrc for persistence:
# eval "$(ssh-agent -s)" && ssh-add ~/.ssh/id_ed25519
```

---

### EC-038 — GitHub Deploy Key Only Works for One Repository
**Category:** Remote | **Severity:** Medium | **Env:** Prod (CI/CD)

**Scenario:** Add SSH deploy key to Repo A. Try to use same key for Repo B. GitHub rejects — deploy keys are unique per repository. CI pipeline accessing multiple repos fails.

**Fix:**
```bash
# Use GitHub App installation token (works across repos in org)
# Or use SSH config with different keys per host alias:
# ~/.ssh/config:
Host github-repo-a
  HostName github.com
  IdentityFile ~/.ssh/id_repo_a
Host github-repo-b
  HostName github.com
  IdentityFile ~/.ssh/id_repo_b
```

---

### EC-039 — `git push --mirror` Overwrites All Remote Branches and Tags
**Category:** Remote | **Severity:** Critical | **Env:** Prod

**Scenario:** Using `git push --mirror` to sync repos. Mirror push DELETES remote branches and tags that don't exist locally. Entire team's feature branches deleted.

```
  git push --all:    pushes all local branches (safe — doesn't delete remote-only branches)
  git push --mirror: makes remote an EXACT copy of local (DELETES remote branches not in local!)
```

> ⚠️ **DevOps Gotcha:** `--mirror` is for repo migration only, not daily syncing. Use `--all --tags` for syncing without deleting.

---

### EC-040 — GitHub Personal Access Token Expired — CI Pipeline Fails
**Category:** Remote | **Severity:** High | **Env:** Prod (CI/CD)

**Scenario:** CI pipeline uses PAT for git clone. PAT expires. All pipelines fail simultaneously with "remote: Invalid credentials". Whoever created the PAT is on vacation.

**Fix:**
```bash
# Use GitHub App installation tokens (auto-rotating, org-controlled)
# Or use fine-grained PATs with documented owners and expiry calendar
# GitLab: use CI_JOB_TOKEN (auto-generated per job)
```

> ⚠️ **DevOps Gotcha:** Never use personal PATs for CI/CD. Create machine user accounts or use GitHub Apps. Set PAT expiry alerts in your monitoring. Document PAT ownership in your runbook.

---

### EC-041 — `git fetch` From Fork Doesn't Include Upstream Tags
**Category:** Remote | **Severity:** Low | **Env:** Dev

**Scenario:** Working with a fork. `git fetch origin` gets fork's tags. Upstream tags (version tags) not available. `git describe` fails or shows wrong version.

**Fix:**
```bash
# Add upstream remote and fetch tags:
git remote add upstream https://github.com/original/repo.git
git fetch upstream --tags
```

---

### EC-042 — Remote URL Changed — All Developers Must Update Manually
**Category:** Remote | **Severity:** Medium | **Env:** Dev/Prod

**Scenario:** Organization renamed (github.com/old-name → github.com/new-name). GitHub redirects HTTP requests but not SSH. All developers with SSH remotes must update.

**Fix:**
```bash
# Update for each developer:
git remote set-url origin git@github.com:new-name/repo.git
# Verify:
git remote -v
```

---

### EC-043 — `pre-receive` Hook on Server Rejects Push Without Useful Error
**Category:** Remote | **Severity:** Medium | **Env:** Dev/Prod

**Scenario:** Push rejected by server-side hook. Error message: "remote: pre-receive hook declined". No useful information about WHY.

**Detection:**
```bash
GIT_TRACE=1 git push origin main   # verbose trace output
# Or check server-side hook logs (if you have access)
```

---

### EC-044 — Large Push Fails With "pack exceeds maximum allowed size"
**Category:** Remote | **Severity:** High | **Env:** Dev/Prod

**Scenario:** Initial push of large repository fails. GitHub has a pack size limit (~2GB). Large monorepos exceed this on first push.

**Fix:**
```bash
# Push incrementally using partial history:
git push origin HEAD~1000:refs/heads/main   # push in chunks
git push origin HEAD~500:refs/heads/main
git push origin main                        # push the rest
```

---

### EC-045 — Forked Repository Falls Behind — No Auto-Sync
**Category:** Remote | **Severity:** Low | **Env:** Dev

**Scenario:** Fork made 6 months ago. Original repo has 500 new commits. Fork shows "This branch is 500 commits behind main". Developer works on stale code.

**Fix:**
```bash
git remote add upstream https://github.com/original/repo.git
git fetch upstream
git merge upstream/main
git push origin main
# Or use GitHub's "Sync fork" button
```

---

## Submodules & Subtrees

### EC-046 — `git clone` Doesn't Clone Submodules By Default
**Category:** Submodules | **Severity:** High | **Env:** Dev/Prod (CI/CD)

**Scenario:** Repository uses submodules. `git clone repo` clones main repo only. Submodule directories exist but are empty. Build fails with missing dependency.

**Fix:**
```bash
# Clone with submodules:
git clone --recurse-submodules <repo-url>
# Or init after clone:
git submodule update --init --recursive

# In CI (GitHub Actions):
- uses: actions/checkout@v4
  with:
    submodules: recursive
```

> ⚠️ **DevOps Gotcha:** EVERY CI pipeline that clones a repo with submodules needs `--recurse-submodules` or explicit `submodule update`. This is the #1 reason submodule-based builds fail in CI.

---

### EC-047 — Submodule Pinned to Old Commit — Not Latest Branch
**Category:** Submodules | **Severity:** High | **Env:** Dev

**Scenario:** Submodule tracks a specific commit hash, not a branch. Developer expecting `git pull` to update the submodule to latest. Submodule stays pinned to old commit.

```
  Parent repo stores: submodule → commit abc123
  Submodule upstream has: commit abc123 → def456 → ghi789
  
  git pull (parent): updates parent files, submodule pointer UNCHANGED
  git submodule update: checks out abc123 (still old!)
  
  To update submodule pointer:
  cd submodule && git pull origin main && cd ..
  git add submodule && git commit -m "Update submodule to latest"
```

---

### EC-048 — Detached HEAD Inside Submodule — Commits Lost
**Category:** Submodules | **Severity:** High | **Env:** Dev

**Scenario:** `git submodule update` checks out the pinned commit — which puts the submodule in DETACHED HEAD state. Developer makes changes and commits in the submodule. When parent `git submodule update` runs again, those commits become unreachable.

**Detection:**
```bash
cd submodule
git status   # "HEAD detached at abc123"
```

**Fix:**
```bash
# Always checkout a branch in the submodule before making changes:
cd submodule
git checkout main
# Make changes, commit, push
# Then in parent: git add submodule && git commit
```

---

### EC-049 — Submodule URL Changed — All Team Members Must Sync
**Category:** Submodules | **Severity:** Medium | **Env:** Dev

**Scenario:** Submodule URL changed in `.gitmodules`. Team members pull. Their local `.git/config` still has the old URL. `git submodule update` fails because it uses the OLD URL.

**Fix:**
```bash
# After pulling .gitmodules change:
git submodule sync --recursive
git submodule update --init --recursive
```

---

### EC-050 — Nested Submodules Break at Third Level
**Category:** Submodules | **Severity:** Medium | **Env:** Dev/Prod

**Scenario:** Repo A has submodule B, which has submodule C. `git submodule update --init` initializes B but not C. `--recursive` is required but often forgotten.

**Fix:**
```bash
git submodule update --init --recursive   # always use --recursive
```

---

### EC-051 — `git subtree` Push Recomputes History Each Time
**Category:** Subtrees | **Severity:** Medium | **Env:** Dev

**Scenario:** Using `git subtree push` to split and push a subdirectory. Each push recomputes the split history from scratch. On large repos, this takes minutes.

**Fix:**
```bash
# Pre-compute split and cache it:
git subtree split --prefix=lib/ --annotate="subtree: " -b subtree-branch
git push <subtree-remote> subtree-branch:main
# Subsequent splits are faster because Git has the cached split branch
```

---

### EC-052 — Submodule Removal Leaves Ghost References
**Category:** Submodules | **Severity:** Medium | **Env:** Dev

**Scenario:** Remove a submodule with `git rm submodule-dir`. Entry still exists in `.git/config` and `.git/modules/` directory. Stale references cause confusing warnings.

**Fix:**
```bash
# Complete submodule removal:
git submodule deinit -f path/to/submodule
git rm -f path/to/submodule
rm -rf .git/modules/path/to/submodule
git commit -m "Remove submodule"
```

---

### EC-053 — Submodule Branch Tracking Ignored by `git submodule update`
**Category:** Submodules | **Severity:** Medium | **Env:** Dev

**Scenario:** `.gitmodules` has `branch = main` set. `git submodule update` still checks out the pinned commit hash, not the branch tip.

**Fix:**
```bash
# Use --remote to track branch:
git submodule update --remote
# This checks out the latest commit on the tracked branch
```

---

### EC-054 — Moving a Submodule Path Requires Manual Fixup
**Category:** Submodules | **Severity:** Medium | **Env:** Dev

**Scenario:** Need to move a submodule from `vendor/lib` to `deps/lib`. Simply using `git mv vendor/lib deps/lib` doesn't update `.gitmodules` and `.git/config`.

**Fix:**
```bash
# Git 1.8.5+ supports:
git mv vendor/lib deps/lib
# Then manually verify .gitmodules was updated
# Older Git: remove submodule, re-add at new path
```

---

### EC-055 — Subtree Merge Conflict Harder to Resolve Than Submodule
**Category:** Subtrees | **Severity:** Medium | **Env:** Dev

**Scenario:** `git subtree merge` brings upstream changes into a subdirectory. Merge conflicts between your local modifications to the subtree and upstream changes are hard to untangle because the commit history is mixed.

> ⚠️ **DevOps Gotcha:** Choose submodules when: upstream changes frequently, you rarely modify the dependency. Choose subtrees when: you need to modify the dependency heavily and don't want separate repo management overhead.

---

## Hooks & Automation

### EC-056 — `pre-commit` Hook Bypassed With `--no-verify`
**Category:** Hooks | **Severity:** High | **Env:** Dev

**Scenario:** Carefully crafted pre-commit hook validates code style, runs tests. Developer runs `git commit --no-verify` to skip the hook. Bad code committed and pushed.

**Detection:**
```bash
# Search history for commits that bypassed hooks:
# Can't detect retroactively — hooks run client-side only
```

> ⚠️ **DevOps Gotcha:** Client-side hooks are NOT enforceable. They're a convenience, not a security measure. Always duplicate critical checks in CI/CD (server-side). GitHub/GitLab server-side hooks or branch protection rules ARE enforceable.

---

### EC-057 — `post-receive` Hook Fails Silently — Deployment Doesn't Trigger
**Category:** Hooks | **Severity:** High | **Env:** Prod

**Scenario:** `post-receive` hook triggers deployment. Hook script has a bug (wrong path, missing dependency). Push succeeds but deployment doesn't happen. No error visible to the pusher.

**Fix:**
```bash
# Add error handling and logging to hooks:
#!/bin/bash
set -e
exec >> /var/log/git-hooks/post-receive.log 2>&1
echo "$(date): Deployment triggered by push"
/usr/local/bin/deploy.sh || {
    echo "DEPLOYMENT FAILED!"
    # Send alert
    curl -X POST https://slack-webhook-url -d '{"text":"Deployment failed!"}'
}
```

---

### EC-058 — `pre-commit` Hook Runs on Staged Content, Not Working Directory
**Category:** Hooks | **Severity:** Medium | **Env:** Dev

**Scenario:** Developer stages partial changes (`git add -p`). Pre-commit hook runs linter. Linter sees the FULL working directory file (including unstaged changes), not just staged content. Hook passes but committed code fails lint.

**Fix:**
```bash
# In pre-commit hook, stash unstaged changes first:
git stash --keep-index --include-untracked
# Run linter
pylint $(git diff --cached --name-only --diff-filter=ACMR)
# Restore unstaged changes:
git stash pop
```

---

### EC-059 — Hooks Not Portable — `.git/hooks/` Not Committed
**Category:** Hooks | **Severity:** Medium | **Env:** Dev

**Scenario:** Hooks configured in `.git/hooks/` directory. New developer clones — their `.git/hooks/` is empty (default hooks only). Your team's pre-commit checks don't run for them.

**Fix:**
```bash
# Use pre-commit framework (pre-commit.com):
# .pre-commit-config.yaml (committed to repo):
repos:
- repo: https://github.com/pre-commit/pre-commit-hooks
  rev: v4.4.0
  hooks:
  - id: trailing-whitespace
  - id: check-merge-conflict

# Or configure hooks path to a committed directory:
git config core.hooksPath .githooks
# Then commit .githooks/ directory
```

> ⚠️ **DevOps Gotcha:** Use `pre-commit` framework for portable, version-controlled hooks. Add `pre-commit install` to your project README's getting-started guide. Or use Husky for Node.js projects.

---

### EC-060 — `commit-msg` Hook Rejected But File Already Saved in Editor
**Category:** Hooks | **Severity:** Low | **Env:** Dev

**Scenario:** `commit-msg` hook validates commit message format. Developer writes a long, detailed message. Hook rejects (wrong format). Message is lost — editor doesn't keep it.

**Fix:**
```bash
# Git saves the rejected message in .git/COMMIT_EDITMSG:
cat .git/COMMIT_EDITMSG   # your rejected message is here
# Copy it, fix the format, recommit
```

---

### EC-061 — `prepare-commit-msg` Hook Modifies Message — Developer Confused
**Category:** Hooks | **Severity:** Low | **Env:** Dev

**Scenario:** `prepare-commit-msg` hook auto-prepends JIRA ticket number from branch name. Developer sees their commit message changed and thinks something is wrong.

---

### EC-062 — Server-Side `update` Hook Blocks Push — No Rollback Path
**Category:** Hooks | **Severity:** High | **Env:** Prod

**Scenario:** Server-side `update` hook rejects a push after partially accepting some refs. Some branches updated, others not. Repository is in an inconsistent state from the pusher's perspective.

---

### EC-063 — Hook Script Missing Execute Permission
**Category:** Hooks | **Severity:** Low | **Env:** Dev

**Scenario:** Hook created with correct content but without execute permission (`chmod +x`). Hook is silently ignored. Developer wonders why checks aren't running.

**Fix:**
```bash
chmod +x .git/hooks/pre-commit
# Or for committed hooks:
chmod +x .githooks/pre-commit
git add .githooks/pre-commit
```

---

### EC-064 — Hook Environment Different From Shell Environment
**Category:** Hooks | **Severity:** Medium | **Env:** Dev

**Scenario:** Hook script uses `nvm use 18` or `pyenv activate`. Git hooks run in a minimal shell environment — `nvm`, `pyenv`, and other shell-managed tools aren't available.

**Fix:**
```bash
#!/bin/bash
# Source NVM in hook:
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"
nvm use 18
npm test
```

---

### EC-065 — `pre-push` Hook Can't Prevent Force Push
**Category:** Hooks | **Severity:** Medium | **Env:** Dev

**Scenario:** `pre-push` hook checks for force push but `--no-verify` bypasses it. Client-side hooks cannot reliably prevent force push.

> ⚠️ **DevOps Gotcha:** Force push prevention MUST be server-side. Use GitHub branch protection "Restrict force pushes" or GitLab protected branches. Client-side hooks are advisory only.

---

## Large Files & Performance

### EC-066 — Git LFS Pointer Files Committed Instead of Actual Content
**Category:** LFS | **Severity:** High | **Env:** Dev/Prod

**Scenario:** Developer commits a large file. LFS is configured in `.gitattributes` but `git lfs install` was never run. The file's LFS pointer (text file with hash) is committed instead of the actual binary.

```
  Normal file in repo:        LFS pointer in repo:
  (500MB binary content)      version https://git-lfs.github.com/spec/v1
                               oid sha256:abc123...
                               size 524288000
                               ← 3 lines of text instead of 500MB!
```

**Detection:**
```bash
git lfs ls-files          # shows tracked LFS files
git lfs status            # shows LFS migration state
file large-asset.psd      # should show binary, if ASCII → pointer committed
```

**Fix:**
```bash
# Install LFS and re-track:
git lfs install
git lfs track "*.psd"
git rm --cached large-asset.psd
git add large-asset.psd
git commit -m "Fix LFS tracking"
```

---

### EC-067 — LFS Storage Quota Exceeded — Pushes Rejected
**Category:** LFS | **Severity:** High | **Env:** Prod

**Scenario:** GitHub free LFS: 1GB storage, 1GB/month bandwidth. Team pushes large assets regularly. Storage quota hit — all LFS pushes rejected.

**Detection:**
```bash
# Check LFS usage:
# GitHub: Settings → Billing → Git LFS Data
git lfs env   # shows storage endpoint
```

> ⚠️ **DevOps Gotcha:** GitHub LFS bandwidth resets monthly but storage accumulates. Delete old LFS objects from server to reclaim storage. Or use external LFS servers (AWS S3 + lfs-s3) for larger storage at lower cost.

---

### EC-068 — `git clone` of LFS Repo Downloads All LFS Objects
**Category:** LFS | **Severity:** Medium | **Env:** Dev/CI

**Scenario:** Repository has 50GB of LFS objects. `git clone` downloads ALL of them. CI only needs the code (not the assets). Clone takes 30 minutes.

**Fix:**
```bash
# Skip LFS during clone:
GIT_LFS_SKIP_SMUDGE=1 git clone <repo-url>
# Only download specific LFS files when needed:
git lfs pull --include="assets/needed-file.psd"
```

---

### EC-069 — Monorepo Performance Degrades With 100k+ Files
**Category:** Performance | **Severity:** High | **Env:** Dev

**Scenario:** Large monorepo with 200k files. `git status` takes 15 seconds. `git diff` takes 10 seconds. Developer productivity tanks.

**Fix:**
```bash
# Enable filesystem monitor:
git config core.fsmonitor true   # uses Watchman or built-in FSMonitor
# Enable untracked cache:
git config core.untrackedCache true
# Use sparse checkout for partial repo:
git sparse-checkout init --cone
git sparse-checkout set src/my-service
```

---

### EC-070 — `git log --all --graph` Hangs on Repository With 500k Commits
**Category:** Performance | **Severity:** Medium | **Env:** Dev

**Scenario:** Large repository with long history. `git log --all --graph` computes the entire DAG topology. Terminal hangs for minutes.

**Fix:**
```bash
# Limit output:
git log --oneline --graph -50   # last 50 commits only
git log --oneline --graph main..feature   # specific range
# Use commit-graph for faster traversal:
git commit-graph write --reachable
```

---

### EC-071 — Sparse Checkout Excludes Files Needed by Build
**Category:** Performance | **Severity:** High | **Env:** Prod (CI/CD)

**Scenario:** CI uses sparse checkout to clone only `services/my-app/`. Build fails because shared libraries in `libs/common/` are not checked out.

**Fix:**
```bash
git sparse-checkout set services/my-app libs/common build-scripts
# List all directories needed by the build — not just your service
```

---

### EC-072 — `git blame` Slow on Files With Thousands of Changes
**Category:** Performance | **Severity:** Low | **Env:** Dev

**Scenario:** `git blame` on a frequently-edited file (e.g., CHANGELOG.md) takes 30+ seconds. Each line requires traversing commit history to find when it was last changed.

**Fix:**
```bash
# Use commit-graph to speed up blame:
git commit-graph write --reachable
# Skip formatting-only commits:
git blame --ignore-rev <formatting-commit-hash>
# Or create .git-blame-ignore-revs file:
echo "<commit-hash>" >> .git-blame-ignore-revs
git config blame.ignoreRevsFile .git-blame-ignore-revs
```

---

### EC-073 — Binary Delta Compression Ineffective for Large Binaries
**Category:** Performance | **Severity:** Medium | **Env:** Dev

**Scenario:** Version-controlling a 200MB database dump. Each small change creates a new 200MB blob. Pack file delta compression works poorly for binaries (unlike text diffs).

> ⚠️ **DevOps Gotcha:** Binary files that change frequently should NOT be in Git (even LFS adds overhead). Use artifact registries (Nexus, Artifactory, S3) for binaries. Git excels at text; it's terrible at changing binaries.

---

### EC-074 — `git gc` Auto-Triggered at Worst Time
**Category:** Performance | **Severity:** Medium | **Env:** Dev/CI

**Scenario:** After a `git fetch`, Git auto-triggers `git gc` if loose object threshold is exceeded. On CI with concurrent builds, this locks the repo briefly and can cause race conditions.

**Fix:**
```bash
# Disable auto-gc in CI:
git config gc.auto 0
# Run gc manually during maintenance:
git gc --auto   # only if needed
```

---

### EC-075 — LFS Pre-Push Hook Doubles Push Time
**Category:** LFS/Performance | **Severity:** Medium | **Env:** Dev

**Scenario:** LFS pre-push hook scans all objects to determine what needs uploading. With many LFS objects, this scan takes significant time even when no new LFS objects are being pushed.

**Fix:**
```bash
# If no LFS objects in current push:
git push --no-verify   # skip LFS scan (use cautiously)
# Or optimize LFS cache:
git lfs prune   # remove old local LFS objects
```

---

## Workflow & Collaboration

### EC-076 — PR Approved But Base Branch Changed — Stale Approval
**Category:** Workflow | **Severity:** High | **Env:** Prod

**Scenario:** PR reviewed and approved on Monday. On Wednesday, base branch (`main`) gets a critical security fix. PR merged Friday with stale approval — security fix potentially regressed.

**Fix:**
```bash
# GitHub: Enable "Dismiss stale pull request approvals when new commits are pushed"
# Also enable "Require branches to be up to date before merging"
# GitLab: Settings → Merge Requests → "Remove all approvals when commits are added"
```

> ⚠️ **DevOps Gotcha:** Configure branch protection to BOTH dismiss stale approvals AND require up-to-date branch. Without both, either stale reviews or stale code can slip into production.

---

### EC-077 — `git stash` Applied to Wrong Branch
**Category:** Workflow | **Severity:** Medium | **Env:** Dev

**Scenario:** Developer stashes changes on `feature-A`, switches to `feature-B`, runs `git stash pop`. Changes from A applied to B — potentially conflicting and hard to untangle.

**Fix:**
```bash
# List stashes with context:
git stash list   # shows branch name where stash was created
# Apply specific stash:
git stash apply stash@{2}   # don't pop — keep in stash list until verified
# Delete only after confirming:
git stash drop stash@{2}
```

---

### EC-078 — `.gitkeep` Is Not a Git Feature — Just a Convention
**Category:** Workflow | **Severity:** Low | **Env:** Dev

**Scenario:** Team uses `.gitkeep` files to track empty directories. New developer sees them and asks "what are these?" Git fundamentally cannot track empty directories — `.gitkeep` is a community convention.

---

### EC-079 — Wrong Author Email on Commits — GitHub Shows No Avatar
**Category:** Workflow | **Severity:** Low | **Env:** Dev

**Scenario:** Developer's Git config has personal email. Commits to work repo show personal email. GitHub can't link commits to work account — no avatar, no contribution graph entry.

**Fix:**
```bash
# Set per-directory git config:
# ~/.gitconfig:
[includeIf "gitdir:~/work/"]
    path = ~/.gitconfig-work
# ~/.gitconfig-work:
[user]
    email = dev@company.com
    name = Dev Name
```

---

### EC-080 — Commit Message Convention Not Enforced — Changelog Generation Fails
**Category:** Workflow | **Severity:** Medium | **Env:** Dev/Prod

**Scenario:** Team uses Conventional Commits for auto-changelog. Some developers write "fixed stuff" instead of "fix(auth): resolve token expiry". Changelog generator produces garbage.

**Fix:**
```bash
# Enforce with commitlint:
# .commitlintrc.yml:
extends: ['@commitlint/config-conventional']
# Add as commit-msg hook (or use pre-commit framework)
# CI check:
commitlint --from=HEAD~1 --to=HEAD
```

---

### EC-081 — `git bisect` Breaks When Commits Don't Build
**Category:** Workflow | **Severity:** Medium | **Env:** Dev

**Scenario:** Using `git bisect` to find a regression. Some intermediate commits don't compile. `git bisect` needs a good/bad verdict for each commit but you can't test a broken build.

**Fix:**
```bash
# Skip uncompilable commits:
git bisect skip
# Or automate with a script:
git bisect run ./test.sh
# test.sh should return 125 to skip (can't test) instead of 1 (bad)
```

---

### EC-082 — Stale Feature Branch Merge Creates Massive Diff
**Category:** Workflow | **Severity:** Medium | **Env:** Dev

**Scenario:** Feature branch started 6 months ago. Never rebased. Merge to main creates a 10,000-line diff — impossible to review meaningfully.

> ⚠️ **DevOps Gotcha:** Enforce "branch age" policies. Require branches to be rebased/merged-up within the last week before merging. Some teams use bots to auto-close PRs older than 30 days.

---

### EC-083 — `git tag` Created Locally Not Pushed to Remote
**Category:** Workflow | **Severity:** High | **Env:** Prod

**Scenario:** Developer creates release tag `v2.1.0`. Pushes commits. Tags are NOT pushed by default. CI/CD triggered by tag push never fires. Release not deployed.

**Fix:**
```bash
# Push tags explicitly:
git push origin v2.1.0
# Or push all tags:
git push origin --tags
# Configure push to include tags automatically:
git config --global push.followTags true
```

---

### EC-084 — Signed Commits Not Verified — "Unverified" Badge in GitHub
**Category:** Workflow | **Severity:** Medium | **Env:** Dev/Prod

**Scenario:** Team requires signed commits. Developer's GPG key not uploaded to GitHub. Commits show "Unverified" badge. Branch protection rule "Require signed commits" blocks merge.

**Fix:**
```bash
# Upload GPG key to GitHub:
gpg --armor --export <key-id> | pbcopy
# GitHub → Settings → SSH and GPG keys → New GPG key
# Configure Git to sign:
git config --global user.signingkey <key-id>
git config --global commit.gpgsign true
```

---

### EC-085 — CODEOWNERS File Not in Root — Reviews Not Assigned
**Category:** Workflow | **Severity:** Medium | **Env:** Dev

**Scenario:** `CODEOWNERS` file placed in `docs/CODEOWNERS` or `.github/CODEOWNERS`. GitHub only checks specific locations. File in wrong location is ignored — no review assignments.

```
  GitHub checks these locations (in order):
  1. .github/CODEOWNERS
  2. CODEOWNERS (repo root)
  3. docs/CODEOWNERS
  
  GitLab: CODEOWNERS must be in repo root
```

---

## CI/CD Integration

### EC-086 — GitHub Actions `actions/checkout` Uses Detached HEAD
**Category:** CI/CD | **Severity:** Medium | **Env:** Prod

**Scenario:** `actions/checkout@v4` checks out in detached HEAD state. Scripts that use `git branch --show-current` return empty string. `git push` fails because no branch is checked out.

**Fix:**
```yaml
- uses: actions/checkout@v4
  with:
    ref: ${{ github.head_ref }}   # checkout the actual branch
# Or manually:
- run: git checkout ${{ github.head_ref }}
```

---

### EC-087 — CI Pipeline Triggered on Every Commit Including Docs-Only Changes
**Category:** CI/CD | **Severity:** Medium | **Env:** Prod

**Scenario:** Full build + test pipeline runs even when only `README.md` changed. Wastes CI minutes and slows queue for real code changes.

**Fix:**
```yaml
# GitHub Actions path filter:
on:
  push:
    paths-ignore:
      - '*.md'
      - 'docs/**'
      - '.github/CODEOWNERS'
# GitLab:
rules:
  - changes:
    - "src/**"
    - "Dockerfile"
```

---

### EC-088 — Git Tag-Based Versioning Breaks With Shallow Clone in CI
**Category:** CI/CD | **Severity:** High | **Env:** Prod

**Scenario:** CI uses `git describe --tags` to generate version. Shallow clone has no tags. `git describe` fails with "fatal: No names found, cannot describe anything."

**Fix:**
```yaml
# GitHub Actions:
- uses: actions/checkout@v4
  with:
    fetch-depth: 0    # full history including tags
    fetch-tags: true
```

---

### EC-089 — Concurrent CI Pushes to Same Branch Cause Race Condition
**Category:** CI/CD | **Severity:** High | **Env:** Prod

**Scenario:** Two developers push to `main` within seconds. Two CI pipelines triggered. Both deploy simultaneously. Second deployment may overwrite the first's changes (if deployment is not idempotent).

**Fix:**
```yaml
# GitHub Actions concurrency control:
concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: false   # queue, don't cancel
```

---

### EC-090 — `git diff --name-only` Returns Empty for Merge Commits
**Category:** CI/CD | **Severity:** Medium | **Env:** Prod

**Scenario:** CI pipeline uses `git diff --name-only HEAD~1` to detect changed files. For a merge commit, `HEAD~1` is only the first parent. Changes from the merged branch are not listed.

**Fix:**
```bash
# For merge commits, diff against merge base:
git diff --name-only $(git merge-base HEAD~1 HEAD) HEAD
# Or diff against both parents:
git diff-tree --no-commit-id --name-only -r HEAD
```

---

### EC-091 — GitHub Actions Secret Not Available in Fork PRs
**Category:** CI/CD | **Severity:** High | **Env:** Prod

**Scenario:** Open source project. CI pipeline uses `${{ secrets.DEPLOY_KEY }}`. Fork PR triggers CI but secrets are empty (security feature — forks can't access repo secrets). Pipeline fails.

> ⚠️ **DevOps Gotcha:** GitHub intentionally doesn't expose secrets to fork PRs (prevents malicious forks from exfiltrating secrets). Use `pull_request_target` event (runs in the context of the base repo) with careful permission scoping.

---

### EC-092 — `git diff` Shows No Changes After `git merge --squash`
**Category:** CI/CD | **Severity:** Medium | **Env:** Prod

**Scenario:** CI uses `git diff origin/main...HEAD` to determine changed files. After squash merge, the diff is computed against the squash commit which is already on main. Diff shows nothing.

---

### EC-093 — Renovate/Dependabot Creates Too Many PRs — Overwhelms CI
**Category:** CI/CD | **Severity:** Medium | **Env:** Dev/Prod

**Scenario:** Dependabot enabled for all dependencies. 50 PRs created simultaneously. Each PR triggers CI. CI queue saturated for hours.

**Fix:**
```yaml
# .github/dependabot.yml:
version: 2
updates:
  - package-ecosystem: npm
    directory: "/"
    schedule:
      interval: weekly
    open-pull-requests-limit: 5   # max 5 concurrent PRs
    groups:                        # group related updates
      minor-and-patch:
        update-types: ["minor", "patch"]
```

---

### EC-094 — GitLab Runner `git clean -ffdx` Deletes Docker Volumes
**Category:** CI/CD | **Severity:** High | **Env:** Prod

**Scenario:** GitLab Runner configured with `GIT_CLEAN_FLAGS`. Runner cleans up workspace aggressively. Docker-in-Docker volumes or build caches stored in the workspace are wiped.

**Fix:**
```yaml
# In .gitlab-ci.yml:
variables:
  GIT_CLEAN_FLAGS: -ffdx -e .cache -e node_modules
# Or disable clean entirely:
  GIT_CLEAN_FLAGS: none
```

---

### EC-095 — Release Branch Created From Wrong Commit
**Category:** CI/CD | **Severity:** Critical | **Env:** Prod

**Scenario:** Release branch `release/2.0` created from `develop`. But `develop` has an unfinished feature merged in. Release includes incomplete code. Discovered after QA sign-off.

> ⚠️ **DevOps Gotcha:** Use a "release freeze" process: stop merging features to develop, create release branch, run full regression. Or use feature flags so unfinished features are disabled in release.

---

## Recovery & Disaster

### EC-096 — `git reflog` Saves the Day After Accidental `reset --hard`
**Category:** Recovery | **Severity:** Critical | **Env:** Dev

**Scenario:** `git reset --hard HEAD~5` — deleted 5 commits. Panic. The commits are NOT gone — they're in the reflog for 30–90 days.

**Fix:**
```bash
# Find the lost commit:
git reflog   # shows every HEAD movement
# Output: abc1234 HEAD@{0}: reset: moving to HEAD~5
#         def5678 HEAD@{1}: commit: my important work  ← THIS ONE
git reset --hard def5678   # restore to that commit
```

```
  Git reflog: your personal undo history
  ┌──────────────────────────────────────────────────────┐
  │ Every time HEAD moves, reflog records it             │
  │ reset --hard? reflog has the old HEAD                │
  │ branch deleted? reflog has the last commit hash      │
  │ rebase gone wrong? reflog has the pre-rebase state   │
  └──────────────────────────────────────────────────────┘
  
  Reflog is LOCAL only — not shared with remote
  Expires after 90 days (configurable)
```

---

### EC-097 — Deleted Branch Recovered From Remote Tracking Ref
**Category:** Recovery | **Severity:** High | **Env:** Dev

**Scenario:** Developer deletes local branch `feature-important` without merging. Fortunately, it was pushed to remote. Remote tracking branch still exists locally.

**Fix:**
```bash
# Check if remote tracking branch exists:
git branch -r | grep feature-important
# Recreate local branch from remote:
git checkout -b feature-important origin/feature-important
# If remote branch also deleted:
git reflog | grep feature-important   # find last commit
```

---

### EC-098 — `git fsck --lost-found` Recovers Dangling Commits
**Category:** Recovery | **Severity:** High | **Env:** Dev

**Scenario:** Commits orphaned after aggressive history rewriting. `git log` doesn't show them. But `git fsck` can find dangling objects.

**Fix:**
```bash
git fsck --lost-found --no-reflogs
# Dangling commits written to .git/lost-found/commit/
# Browse them:
ls .git/lost-found/commit/
git show <hash>   # inspect each dangling commit
# Recover:
git branch recovered-branch <hash>
```

---

### EC-099 — Corrupted Index File — `git status` Fails
**Category:** Recovery | **Severity:** High | **Env:** Dev

**Scenario:** Power failure during `git add`. Index file (`.git/index`) is corrupt. `git status` fails with "fatal: index file corrupt".

**Fix:**
```bash
# Remove and rebuild the index:
rm .git/index
git reset   # rebuilds index from HEAD
# Your working directory files are untouched
# But you'll need to re-stage any previously staged changes
```

---

### EC-100 — Force Push Recovery — Using Remote Reflog (GitHub)
**Category:** Recovery | **Severity:** Critical | **Env:** Prod

**Scenario:** Accidental force push to `main` overwrites 50 commits. Those commits are not in YOUR reflog (they're from other developers).

**Fix:**
```bash
# GitHub/GitLab Events API can show the old commit:
# GitHub: check the push event in repository settings → events
# Or contact GitHub support — they maintain server-side reflogs for 90 days

# If any team member still has the old commits:
# On their machine:
git log origin/main --oneline   # should show old history
git push -f origin main         # restore old history (coordinate!)
```

> ⚠️ **DevOps Gotcha:** GitHub Support can recover force-pushed commits within 90 days. But the fastest recovery is from any team member who hasn't fetched the force push yet. Communicate in team chat IMMEDIATELY.

---

### EC-101 — Rebase Gone Wrong — Recovering Pre-Rebase State
**Category:** Recovery | **Severity:** High | **Env:** Dev

**Scenario:** `git rebase -i` went wrong — conflicts everywhere, lost track of what you've done. Need to abort and go back to pre-rebase state.

**Fix:**
```bash
# During rebase:
git rebase --abort   # returns to pre-rebase state (cleanest)

# After rebase completed (already committed):
git reflog   # find ORIG_HEAD or pre-rebase commit
git reset --hard ORIG_HEAD   # Git sets ORIG_HEAD before dangerous operations
```

---

### EC-102 — `git worktree` Left Behind After Branch Deletion
**Category:** Recovery | **Severity:** Low | **Env:** Dev

**Scenario:** Created a worktree for `hotfix-123`. Deleted the branch. Worktree directory still exists but references a non-existent branch. Git complains about lock files.

**Fix:**
```bash
git worktree list              # shows all worktrees
git worktree remove /path/to/worktree
git worktree prune             # clean up stale worktrees
```

---

### EC-103 — `.git/HEAD` Points to Non-Existent Branch
**Category:** Recovery | **Severity:** Critical | **Env:** Dev

**Scenario:** `.git/HEAD` contains `ref: refs/heads/feature-deleted`. Branch was deleted but HEAD still points to it. All git commands fail.

**Fix:**
```bash
# Manually fix HEAD:
echo "ref: refs/heads/main" > .git/HEAD
git checkout main
```

---

### EC-104 — Accidental `git push origin :main` Deletes Remote Main Branch
**Category:** Recovery | **Severity:** Critical | **Env:** Prod

**Scenario:** Typo or misunderstanding — `git push origin :main` (empty source = delete). Remote main branch deleted. CI/CD referencing main branch fails. Team can't clone or pull.

**Fix:**
```bash
# Immediately restore from any team member's local:
git push origin main   # push their local main to remote

# Prevention: protect the default branch
# GitHub: Settings → Branches → Add rule → "main" → check all protections
# This prevents deletion even by admins
```

---

### EC-105 — Recovering Stashed Changes After Accidental `git stash drop`
**Category:** Recovery | **Severity:** Medium | **Env:** Dev

**Scenario:** `git stash drop` accidentally removes a stash with important changes.

**Fix:**
```bash
# Stash entries are commits — findable via fsck:
git fsck --no-reflogs --unreachable | grep commit
# Find your stash commit:
git show <hash>   # check each until you find your changes
# Recover:
git stash apply <hash>
```

---

## Advanced / Internals

### EC-106 — `git revert` of a Merge Commit Requires Parent Specification
**Category:** Advanced | **Severity:** Medium | **Env:** Dev/Prod

**Scenario:** `git revert <merge-commit>` fails with "commit is a merge but no -m option was given". Merge commits have two parents — Git needs to know which parent to revert relative to.

**Fix:**
```bash
# Revert merge, keeping first parent (mainline) changes:
git revert -m 1 <merge-commit>
# -m 1 = keep parent 1 (usually main branch) and undo parent 2 (feature branch)
```

---

### EC-107 — Git Object Hash Collision (SHA-1) — Theoretical but Dangerous
**Category:** Advanced | **Severity:** Low | **Env:** Theoretical

**Scenario:** Two different objects produce the same SHA-1 hash. Git would confuse them — potentially serving wrong file content.

```
  SHA-1 collision: demonstrated by Google in 2017 (SHAttered)
  Git response: migrating to SHA-256 (object-format=sha256)
  Currently: SHA-1 with collision detection hardening
```

**Detection:**
```bash
# Check if your Git has collision detection:
git version   # 2.13+ has cr-sha1 with collision detection
```

---

### EC-108 — `git replace` Silently Alters History View
**Category:** Advanced | **Severity:** High | **Env:** Dev/Prod

**Scenario:** `git replace` substitutes one commit for another WITHOUT rewriting history. Local `git log` shows the replaced commit. But `git push` doesn't push replacements by default — other developers see original history.

**Detection:**
```bash
git replace -l   # list all active replacements
```

---

### EC-109 — Grafting Legacy History Breaks After `git gc`
**Category:** Advanced | **Severity:** Medium | **Env:** Dev

**Scenario:** `.git/info/grafts` used to stitch together two repositories' histories. After `git gc` or `git filter-repo`, graft points may be permanently baked in or lost.

> ⚠️ **DevOps Gotcha:** Grafts are deprecated. Use `git replace --graft` instead, which creates replace objects that can be shared (unlike grafts). Then use `git filter-repo` to permanently bake in the replacements.

---

### EC-110 — `git notes` Not Pushed With `git push` — Lost For Others
**Category:** Advanced | **Severity:** Low | **Env:** Dev

**Scenario:** Developer adds notes to commits with `git notes add -m "Reviewed by Alice" <commit>`. Notes exist locally only. Other developers never see them.

**Fix:**
```bash
# Push notes explicitly:
git push origin refs/notes/*
# Pull notes:
git fetch origin refs/notes/*:refs/notes/*
# Configure auto-fetch:
git config --add remote.origin.fetch '+refs/notes/*:refs/notes/*'
```

---

## Production FAQ

**Q1: Someone pushed credentials to the repo. What's the emergency procedure?**  
A: (1) Rotate the credential IMMEDIATELY — don't wait for Git cleanup. (2) Use `git filter-repo --invert-paths --path secrets.env` to remove from history. (3) Force push all branches. (4) ALL team members must re-clone. (5) Invalidate any cached CI artifacts that may contain the secret.

**Q2: Our monorepo is slow. `git status` takes 10+ seconds.**  
A: Enable FSMonitor: `git config core.fsmonitor true`. Enable untracked cache: `git config core.untrackedCache true`. Use sparse checkout if you only need part of the repo. Run `git gc` and `git commit-graph write --reachable`. Consider `git maintenance start` for background optimization.

**Q3: How do I undo a merge that's already been pushed?**  
A: `git revert -m 1 <merge-commit>` — creates a new commit that undoes the merge changes while preserving history. DON'T use `git reset --hard` on a shared branch. Remember: if you later want to re-merge that branch, you'll need to "revert the revert".

**Q4: CI pipeline says "diverged" and won't merge. What happened?**  
A: Branch has commits that aren't in main AND main has commits that aren't in the branch. Solution: `git fetch origin && git rebase origin/main` (or merge). If using GitHub PR, click "Update branch" button.

**Q5: A file is in `.gitignore` but still tracked. How to fix?**  
A: `git rm --cached <file>` removes from tracking but keeps the file on disk. Then commit. The `.gitignore` rule now applies. Note: the file still exists in git history — for secrets, use `git filter-repo` to purge.

**Q6: How to find which commit introduced a bug?**  
A: Use `git bisect`: `git bisect start`, `git bisect bad` (current HEAD is bad), `git bisect good v1.0.0` (known good version). Git binary searches the history. For automated bisect: `git bisect run ./test-script.sh`.

**Q7: Submodule shows `(modified content)` but I didn't change anything.**  
A: The submodule's checked-out commit differs from what the parent repo expects. Run `git submodule update --init` to checkout the pinned commit. If you intentionally want the newer version, `cd submodule && git pull && cd .. && git add submodule && git commit`.

**Q8: `git pull` creates ugly merge commits. How to keep history clean?**  
A: Use `git pull --rebase` to rebase your local commits on top of remote changes. Set as default: `git config --global pull.rebase true`. Or use `git pull --ff-only` which fails if fast-forward isn't possible.

**Q9: How to split a monorepo into separate repos while preserving history?**  
A: Use `git filter-repo --path src/service-a --path-rename src/service-a/:` — extracts only that directory's history and makes it the root. Push to new repo. Repeat for each service. Original monorepo remains intact.

**Q10: We need to change all commit author emails (company rebranding).**  
A: Use `git filter-repo --mailmap mailmap.txt` where mailmap.txt maps old to new: `New Name <new@email.com> Old Name <old@email.com>`. Force push all branches. ALL team members must re-clone. This rewrites every commit hash.

---

> **Next:** [Nginx Edge Cases](nginx_edge_cases.md) · [Apache Edge Cases](apache_edge_cases.md)  
> **Back:** [Cloud Edge Cases](cloud_edge_cases.md) · [Batch State](batch_state.md) · [🏠 Home](../README.md)
