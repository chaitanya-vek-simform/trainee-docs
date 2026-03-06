[🏠 Home](../README.md) · [Git](README.md)

# 🔀 Git — Comprehensive DevOps/Cloud Handbook

> **Audience:** DevOps / Cloud Engineers & Trainees | **Focus:** Internals, Mental Models, Production Workflows  
> **Covers:** Git Internals, Branching, Merge/Rebase, Workflows, Disaster Recovery, Security, CI/CD Integration, Large Repos  
> **Tip:** Sections build on each other — read sequentially first, then use TOC for reference.

---

## Table of Contents

1. [What Git Actually Is](#1-what-git-actually-is)
2. [Git Internals — Inside .git/](#2-git-internals--inside-git)
3. [Branch Mechanics](#3-branch-mechanics)
4. [Merge vs Rebase — Deep Understanding](#4-merge-vs-rebase--deep-understanding)
5. [Remote Management](#5-remote-management)
6. [Pull Strategies](#6-pull-strategies)
7. [Git Workflows — Enterprise Level](#7-git-workflows--enterprise-level)
8. [Interactive Rebase](#8-interactive-rebase)
9. [Reset vs Restore vs Revert](#9-reset-vs-restore-vs-revert)
10. [Disaster Recovery — Reflog & Beyond](#10-disaster-recovery--reflog--beyond)
11. [Handling Secrets & Security](#11-handling-secrets--security)
12. [Submodules, Subtrees & Monorepos](#12-submodules-subtrees--monorepos)
13. [Hooks & Automation — DevOps Territory](#13-hooks--automation--devops-territory)
14. [Performance & Large Repos](#14-performance--large-repos)
15. [Edge Cases That Break Juniors](#15-edge-cases-that-break-juniors)
16. [Git in Production Environments](#16-git-in-production-environments)
17. [Git Internals for Elite Level](#17-git-internals-for-elite-level)
18. [Production Scenario Based FAQ](#18-production-scenario-based-faq)

---

## 1. What Git Actually Is

### 1.1 Why Understanding Git Deeply Matters for DevOps

As a DevOps/Cloud engineer, Git is **not just a tool — it's infrastructure**:
- Every CI/CD pipeline starts with a `git` event (push, tag, PR)
- Infrastructure as Code (Terraform, Ansible, Helm) lives in Git
- GitOps (ArgoCD, Flux) treats Git as the **single source of truth** for cluster state
- Incident response often requires precise history navigation (`bisect`, `reflog`, `blame`)
- Secret leaks in Git history can compromise entire production environments

> **Bottom line:** A DevOps engineer who doesn't understand Git deeply is building on sand.

### 1.2 Distributed VCS — Mental Model

Git is a **Distributed Version Control System (DVCS)**. Every clone is a full repository with complete history.

```
  ┌─────────────────────────────────────────────────────────────────┐
  │            CENTRALIZED (SVN)  vs  DISTRIBUTED (Git)             │
  │                                                                 │
  │  Centralized (Subversion)         Distributed (Git)             │
  │  ┌──────────────────┐            ┌──────────────────┐           │
  │  │  Central Server  │            │  Remote (GitHub)  │           │
  │  │  (single point   │            │  (one of many     │           │
  │  │   of failure)    │            │   possible remotes)│          │
  │  └──────┬───────────┘            └──────┬───────────┘           │
  │         │                               │                       │
  │    ┌────┼────┐                   ┌──────┼──────┐                │
  │    │    │    │                   │      │      │                │
  │    ▼    ▼    ▼                   ▼      ▼      ▼                │
  │  ┌───┐┌───┐┌───┐             ┌─────┐┌─────┐┌─────┐             │
  │  │WC ││WC ││WC │             │FULL ││FULL ││FULL │             │
  │  │   ││   ││   │             │REPO ││REPO ││REPO │             │
  │  │no ││no ││no │             │+hist││+hist││+hist│             │
  │  │hist││hist││hist│            │+brnch│+brnch│+brnch│            │
  │  └───┘└───┘└───┘             └─────┘└─────┘└─────┘             │
  │                                                                 │
  │  • Offline = dead             • Offline = full functionality    │
  │  • Server down = no work      • Server down = still commit,    │
  │  • Single point of failure      branch, merge, log, diff       │
  │                               • Every clone is a backup        │
  └─────────────────────────────────────────────────────────────────┘
```

### 1.3 Snapshots, Not Diffs

Most VCS systems store **diffs** (changes between versions). Git stores **snapshots** of the entire project state at each commit.

```
  DIFF-BASED (SVN, CVS)                 SNAPSHOT-BASED (Git)
  ┌─────────────────────────┐            ┌──────────────────────────┐
  │                         │            │                          │
  │  V1:  FileA (base)      │            │  Commit 1:               │
  │  V2:  FileA (Δ1)        │            │   snapshot → A1, B1, C1  │
  │  V3:  FileA (Δ2)        │            │                          │
  │  V4:  FileA (Δ3)        │            │  Commit 2:               │
  │                         │            │   snapshot → A1, B2, C1  │
  │  To get V4:             │            │   (A1, C1 are pointers   │
  │  Apply base + Δ1 +      │            │    to same blob — dedup!)│
  │  Δ2 + Δ3               │            │                          │
  │  = Slow for old files   │            │  Commit 3:               │
  │                         │            │   snapshot → A2, B2, C2  │
  │                         │            │                          │
  │                         │            │  Any commit = instant    │
  │                         │            │  access (no reconstruction)│
  └─────────────────────────┘            └──────────────────────────┘
```

> **DevOps Relevance:** Snapshot-based design is why `git checkout` to any commit in a 10-year-old repo is instant. CI/CD pipelines depend on this speed.

### 1.4 Content-Addressable Storage — SHA-1 Hashes

Everything in Git is identified by a **SHA-1 hash** (40-character hex string). Git is essentially a **content-addressable filesystem** — a key-value store where the key is the hash of the content.

```
  Content → SHA-1 Hash → Stored as Object
  ┌──────────────────────────────────────────────────┐
  │                                                  │
  │  "Hello World\n"  ──SHA-1──►  557db03de997c86a   │
  │                                                  │
  │  If content changes, hash changes.               │
  │  If content is same, hash is same (dedup).       │
  │  This guarantees:                                │
  │   • Integrity (any corruption = hash mismatch)   │
  │   • Deduplication (same file = stored once)      │
  │   • Immutability (objects never change)          │
  │                                                  │
  └──────────────────────────────────────────────────┘
```

> **⚠️ Gotcha:** Git is migrating from SHA-1 to SHA-256 (since Git 2.29+). SHA-1 has known collision attacks (SHAttered, 2017). For most repos this isn't a practical risk, but high-security environments should track this migration.

### 1.5 The Four Areas — Working Directory, Staging, Local Repo, Remote

This is the most important mental model in Git:

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │                    GIT's FOUR AREAS                                  │
  │                                                                      │
  │  ┌───────────┐    ┌───────────┐    ┌───────────┐    ┌─────────────┐  │
  │  │  Working  │    │  Staging  │    │   Local   │    │   Remote    │  │
  │  │ Directory │───►│   Area    │───►│   Repo    │───►│   Repo      │  │
  │  │           │add │  (Index)  │commit│ (.git/) │push│  (GitHub/   │  │
  │  │ Your files│    │ Snapshot  │    │  History  │    │  GitLab)    │  │
  │  │ on disk   │    │ preview   │    │  DAG      │    │             │  │
  │  └───────────┘    └───────────┘    └───────────┘    └─────────────┘  │
  │       ▲                                  │               │           │
  │       │              checkout            │    fetch      │           │
  │       └──────────────────────────────────┘◄──────────────┘           │
  │                                                                      │
  │  Lifecycle:                                                          │
  │  ┌─────────┐  git add  ┌────────┐ git commit ┌──────────┐ git push  │
  │  │Modified │──────────►│Staged  │───────────►│Committed │──────────►│
  │  └─────────┘           └────────┘            └──────────┘   Remote  │
  │                                                                      │
  └──────────────────────────────────────────────────────────────────────┘
```

#### Each Area Explained

| Area | Location | Purpose | Key Commands |
|------|----------|---------|-------------|
| **Working Directory** | Your filesystem | Files you edit directly | `vim`, `code`, any editor |
| **Staging Area (Index)** | `.git/index` | Pre-commit snapshot — cherry-pick what goes into next commit | `git add`, `git reset HEAD` |
| **Local Repository** | `.git/objects/` | Full commit history stored as DAG of objects | `git commit`, `git log` |
| **Remote Repository** | Server (GitHub, GitLab, Bitbucket) | Shared repository for collaboration | `git push`, `git fetch`, `git pull` |

> **Why Staging Exists:** It lets you craft precise commits. You can modify 10 files but only stage 3 for one commit and 7 for another. **Atomic, meaningful commits** are critical for debugging with `git bisect` and readable `git log` in production.

### 1.6 File Status Lifecycle

```
  ┌─────────────────────────────────────────────────────────────┐
  │              GIT FILE STATUS LIFECYCLE                       │
  │                                                             │
  │  ┌───────────┐  git add  ┌───────────┐  git commit         │
  │  │ Untracked │──────────►│  Staged   │───────────┐         │
  │  └───────────┘           └───────────┘           │         │
  │       ▲                       ▲                  ▼         │
  │       │                       │            ┌───────────┐   │
  │  new file                  git add         │ Committed │   │
  │  created                   (modified)      │(Unmodified)│  │
  │       │                       │            └─────┬─────┘   │
  │       │                       │                  │         │
  │       │                  ┌────┴──────┐      edit file      │
  │       │                  │  Modified │◄──────────┘         │
  │       │                  └───────────┘                     │
  │       │                       │                            │
  │       │  git rm --cached      │                            │
  │       │◄──────────────────────┘                            │
  │       │                                                    │
  └───────┴────────────────────────────────────────────────────┘
```

```bash
# Check file status
git status                    # Full status
git status -s                 # Short format: M=modified, A=added, ?=untracked, D=deleted

# Stage files
git add file.txt              # Stage specific file
git add .                     # Stage all changes in current dir (recursively)
git add -p                    # Interactive staging — stage HUNKS, not whole files

# Unstage
git reset HEAD file.txt       # Unstage (keep changes in working directory)
git restore --staged file.txt # Modern way (Git 2.23+)
```

> **⚠️ Gotcha:** `git add .` stages everything including files you may not want (build artifacts, secrets). Always have a `.gitignore` in place BEFORE your first commit.

> **DevOps Pro Tip:** `git add -p` (patch mode) is invaluable for creating clean, atomic commits. It lets you stage individual hunks within a file — critical for code review hygiene.

---

## 2. Git Internals — Inside .git/

### 2.1 Why Learn Git Internals?

Most Git problems that engineers "can't figure out" are instantly solvable when you understand the internals. This knowledge separates someone who uses Git from someone who **understands** Git.

### 2.2 The .git/ Directory — Anatomy

When you run `git init`, Git creates a `.git/` directory. This IS your repository. Everything else is the working directory.

```
  .git/                                    ← THE repository
  ├── HEAD                                 ← Pointer to current branch (e.g., ref: refs/heads/main)
  ├── config                               ← Repo-level configuration
  ├── description                          ← Used by GitWeb (rarely relevant)
  ├── index                                ← THE staging area (binary file)
  ├── objects/                             ← ALL content stored here (blobs, trees, commits, tags)
  │   ├── pack/                            ← Packed objects (compressed, for efficiency)
  │   ├── info/
  │   └── ab/                              ← Objects stored in subdirs by first 2 chars of hash
  │       └── cdef1234567890...            ← Object file (remaining 38 chars)
  ├── refs/                                ← Branch and tag pointers
  │   ├── heads/                           ← Local branch pointers
  │   │   ├── main                         ← File containing commit hash that 'main' points to
  │   │   └── feature-x                    ← File containing commit hash
  │   ├── tags/                            ← Tag pointers
  │   │   └── v1.0.0                       ← File containing tag object hash
  │   └── remotes/                         ← Remote tracking branches
  │       └── origin/
  │           ├── main
  │           └── feature-x
  ├── hooks/                               ← Client-side hook scripts
  │   ├── pre-commit.sample
  │   ├── pre-push.sample
  │   └── commit-msg.sample
  ├── info/
  │   └── exclude                          ← Repo-local ignore rules (not shared)
  └── logs/                                ← Reflog data (history of ref changes)
      ├── HEAD
      └── refs/
          └── heads/
              └── main
```

> **DevOps Relevance:** Understanding `.git/` structure is critical when:
> - Debugging CI/CD clone failures (shallow clones missing refs)
> - Investigating "detached HEAD" in automated pipelines
> - Setting up Git hooks for policy enforcement
> - Troubleshooting `.git/config` for remote URLs (SSH vs HTTPS)

### 2.3 Git Object Model — The Four Object Types

Everything Git stores is one of four object types. This is the core data model.

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                  GIT OBJECT MODEL                                │
  │                                                                  │
  │  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────────┐  │
  │  │   BLOB   │   │   TREE   │   │  COMMIT  │   │     TAG      │  │
  │  │          │   │          │   │          │   │  (annotated)  │  │
  │  │ Raw file │   │ Directory│   │ Snapshot │   │              │  │
  │  │ content  │   │ listing  │   │ metadata │   │ Named pointer│  │
  │  │          │   │          │   │          │   │ + message    │  │
  │  │ No name  │   │ Names +  │   │ Points   │   │ + GPG sig   │  │
  │  │ No meta  │   │ blob/tree│   │ to tree  │   │              │  │
  │  │ Just data│   │ refs     │   │ + parent │   │ Points to    │  │
  │  │          │   │          │   │ + author │   │ commit       │  │
  │  │          │   │          │   │ + message│   │              │  │
  │  └──────────┘   └──────────┘   └──────────┘   └──────────────┘  │
  │                                                                  │
  │  Relationship Chain:                                             │
  │                                                                  │
  │  TAG ──► COMMIT ──► TREE ──► BLOB                                │
  │                       │                                          │
  │                       ├──► TREE (subdirectory) ──► BLOB          │
  │                       │                                          │
  │                       └──► BLOB                                  │
  │                                                                  │
  └──────────────────────────────────────────────────────────────────┘
```

#### Object Type Details

| Object | Stores | Identified By | Example |
|--------|--------|---------------|---------|
| **Blob** | Raw file content (no filename, no permissions) | SHA-1 of content | The contents of `README.md` |
| **Tree** | Directory listing: filenames + permissions + blob/tree refs | SHA-1 of tree structure | A directory with `README.md` (blob) and `src/` (tree) |
| **Commit** | Tree ref + parent commit(s) + author + committer + message | SHA-1 of commit metadata | Points to root tree + previous commit |
| **Tag** | Annotated tag: tag name + tagger + message + optional GPG sig | SHA-1 of tag object | `v1.0.0` with release notes |

### 2.4 How a Commit Is Structured — Visual Walkthrough

```
  $ git cat-file -p HEAD       (inspect current commit)

  ┌─────────────────────────────────────────────────┐
  │  Commit Object (e.g. a1b2c3d...)                │
  │                                                 │
  │  tree    f4e5d6a7...    ──────────┐             │
  │  parent  b8c9d0e1...    ──┐       │             │
  │  author  Jane <j@x.com>  │       │             │
  │  committer Jane <j@x.com>│       │             │
  │  date    2026-03-03      │       │             │
  │                           │       │             │
  │  "Fix: resolve null ptr"  │       │             │
  └───────────────────────────┘       │             │
                                      │             │
  ┌──────────────────────┐            │             │
  │  Parent Commit       │◄───────────┘             │
  │  (previous commit)   │                          │
  └──────────────────────┘                          │
                                                    │
  ┌─────────────────────────────────────────┐       │
  │  Tree Object (root directory)           │◄──────┘
  │                                         │
  │  100644 blob abc123... README.md        │──► Blob "# My Project\n..."
  │  100644 blob def456... Makefile         │──► Blob "all:\n\tgo build..."
  │  040000 tree 789abc... src/             │──┐
  └─────────────────────────────────────────┘  │
                                               │
  ┌─────────────────────────────────────────┐  │
  │  Tree Object (src/ subdirectory)        │◄─┘
  │                                         │
  │  100644 blob aaa111... main.go          │──► Blob "package main\n..."
  │  100644 blob bbb222... utils.go         │──► Blob "package main\n..."
  └─────────────────────────────────────────┘
```

### 2.5 Exploring Git Objects — Commands

```bash
# Hash an object (see what Git would store)
echo "Hello World" | git hash-object --stdin
# Output: 557db03de997c86a4a028e1ebd3a1ceb225be238

# Store an object in the database
echo "Hello World" | git hash-object -w --stdin

# Read an object's content
git cat-file -p <hash>             # Pretty-print content
git cat-file -t <hash>             # Show object type (blob, tree, commit, tag)
git cat-file -s <hash>             # Show object size

# List tree contents
git ls-tree HEAD                   # Show root tree of current commit
git ls-tree -r HEAD                # Recursive — all files in commit

# Show what's in the index (staging area)
git ls-files --stage               # Show staged files with hashes and modes

# Inspect pack files
git verify-pack -v .git/objects/pack/pack-*.idx
```

> **DevOps Pro Tip:** When CI pipelines do shallow clones (`git clone --depth 1`), many objects are missing. This breaks `git log`, `git blame`, and `git describe`. Understanding the object model helps you debug these failures.

### 2.6 How Git Stores Data — The DAG (Directed Acyclic Graph)

Git's commit history forms a DAG. Each commit points back to its parent(s). This is what makes branching and merging possible.

```
  ┌──────────────────────────────────────────────────────────────┐
  │             GIT COMMIT HISTORY = DAG                         │
  │                                                              │
  │  Simple linear history:                                      │
  │                                                              │
  │  A ◄─── B ◄─── C ◄─── D       (each points to parent)      │
  │                          ▲                                   │
  │                          │                                   │
  │                         HEAD                                 │
  │                         main                                 │
  │                                                              │
  │  With branches:                                              │
  │                                                              │
  │  A ◄─── B ◄─── C ◄─── D       main                         │
  │                  ▲                                           │
  │                  └──── E ◄─── F       feature                │
  │                                                              │
  │  After merge:                                                │
  │                                                              │
  │  A ◄─── B ◄─── C ◄─── D ◄─── G       main (merge commit)  │
  │                  ▲              ▲                             │
  │                  └──── E ◄─── F┘       G has TWO parents    │
  │                                                              │
  │  Key properties of DAG:                                      │
  │  • Directed: commits point to parents (backward in time)     │
  │  • Acyclic: no loops possible                                │
  │  • Enables parallel branches that rejoin                     │
  │  • Allows traversal for log, blame, bisect                   │
  └──────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha:** Git arrows point **backward** (child → parent). This confuses people who think "time flows left to right." In Git, the latest commit points BACK to its parent. `HEAD` is the tip — you walk backward through history.

---

## 3. Branch Mechanics

### 3.1 What Is a Branch, Really?

A branch in Git is nothing more than a **pointer (ref) to a commit**. It's a 41-byte file (40-char hash + newline) stored in `.git/refs/heads/`.

```
  ┌──────────────────────────────────────────────────────────────┐
  │  A branch is NOT a copy of files.                            │
  │  A branch is NOT a directory.                                │
  │  A branch IS a movable pointer to a commit.                  │
  │                                                              │
  │  .git/refs/heads/main:                                       │
  │  ┌──────────────────────────────────────┐                    │
  │  │  a1b2c3d4e5f6... (40 char SHA-1)    │ ← just a text file │
  │  └──────────────────────────────────────┘                    │
  │                                                              │
  │  Creating a branch = creating a 41-byte file                 │
  │  Switching a branch = changing what HEAD points to           │
  │  Deleting a branch = deleting a 41-byte file                 │
  │  (commits are NOT deleted — they become unreachable)         │
  │                                                              │
  │  That's why Git branches are "cheap" — they're just pointers │
  └──────────────────────────────────────────────────────────────┘
```

```bash
# Prove it:
cat .git/refs/heads/main          # Shows the commit hash main points to
cat .git/HEAD                     # Shows: ref: refs/heads/main

# Branch operations
git branch feature-x              # Create branch (doesn't switch to it)
git checkout feature-x            # Switch to branch (old way)
git switch feature-x              # Switch to branch (modern, Git 2.23+)
git checkout -b feature-x         # Create + switch (old way)
git switch -c feature-x           # Create + switch (modern)
git branch -d feature-x           # Delete (only if merged)
git branch -D feature-x           # Force delete (even if unmerged)
git branch -a                     # List all branches (local + remote)
git branch -vv                    # Verbose: show tracking info + last commit
```

### 3.2 HEAD — The Special Pointer

HEAD is a pointer to **the current branch** (which in turn points to a commit). It's how Git knows "where you are."

```
  Normal state (HEAD → branch → commit):
  ┌──────────────────────────────────────────────────┐
  │                                                  │
  │  HEAD ──► main ──► commit C                      │
  │                                                  │
  │  .git/HEAD contains:                             │
  │  "ref: refs/heads/main"                          │
  │                                                  │
  │  When you commit, main moves forward.            │
  │  HEAD still points to main.                      │
  │                                                  │
  │  A ◄── B ◄── C      (main, HEAD)                │
  │                                                  │
  │  After new commit:                               │
  │  A ◄── B ◄── C ◄── D      (main, HEAD)          │
  │                                                  │
  └──────────────────────────────────────────────────┘
```

### 3.3 Detached HEAD — What, Why, and When

Detached HEAD means HEAD points **directly to a commit** instead of to a branch.

```
  Normal HEAD:                      Detached HEAD:
  ┌─────────────────────┐           ┌──────────────────────┐
  │                     │           │                      │
  │  HEAD → main → C    │           │  HEAD → C (directly) │
  │                     │           │  main → C            │
  │  .git/HEAD:         │           │                      │
  │  ref: refs/heads/   │           │  .git/HEAD:          │
  │       main          │           │  a1b2c3d4e5f6...     │
  │                     │           │  (raw commit hash)   │
  └─────────────────────┘           └──────────────────────┘
```

**When does detached HEAD happen?**

```bash
git checkout <commit-hash>          # Checking out a specific commit
git checkout v1.0.0                 # Checking out a tag
git checkout origin/main            # Checking out a remote tracking branch
git rebase                          # Temporarily during rebase
git bisect                          # During bisect operations
```

**Danger of detached HEAD:**
```
  ┌─────────────────────────────────────────────────────────────┐
  │  If you commit in detached HEAD:                            │
  │                                                             │
  │  A ◄── B ◄── C      (main)                                 │
  │               ▲                                             │
  │               └── D ◄── E      (HEAD — no branch!)          │
  │                                                             │
  │  If you switch to main now:                                 │
  │  D and E become UNREACHABLE (orphaned).                     │
  │  They will be garbage collected after ~30 days.             │
  │                                                             │
  │  RECOVERY:                                                  │
  │  git branch rescue-branch HEAD    ← before switching away   │
  │  OR                                                         │
  │  git reflog → find the hash → git branch rescue <hash>     │
  └─────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha (CI/CD):** Many CI systems clone in detached HEAD state (Jenkins, GitLab CI). If your pipeline script does `git branch --show-current`, it returns empty. Use `git rev-parse HEAD` instead for the commit hash. This trips up DevOps engineers writing pipeline scripts.

### 3.4 Branch Edge Cases

| Scenario | What Happens | Fix |
|----------|-------------|-----|
| Commit in detached HEAD | Commits are orphaned when you switch away | `git branch <name>` before switching |
| Delete the currently checked-out branch | Git refuses — you can't saw off the branch you're sitting on | Switch to another branch first |
| Force push a rewritten branch | Remote branch pointer moves; anyone who pulled the old history gets diverged | Coordinate with team, use `--force-with-lease` |
| Two branches pointing to same commit | Perfectly normal — branches are just pointers | No fix needed |
| Branch name with `/` (e.g., `feature/login`) | Works fine — creates nested dirs in `.git/refs/heads/` | Avoid conflicts like having both `feature` branch AND `feature/login` |

---

## 4. Merge vs Rebase — Deep Understanding

### 4.1 Why This Matters

This is the single most debated topic in Git, and misunderstanding it causes:
- Corrupted team histories
- Duplicated commits after rebase of public branches
- Lost work during conflict resolution
- CI/CD pipeline confusion from non-linear history

### 4.2 Merge — How It Works

A merge combines two branches by creating a **merge commit** with two parents.

```
  Before merge:
  ┌──────────────────────────────────────────┐
  │                                          │
  │  A ◄── B ◄── C ◄── D      (main)        │
  │              ▲                           │
  │              └── E ◄── F   (feature)     │
  │                                          │
  └──────────────────────────────────────────┘

  $ git checkout main
  $ git merge feature

  After merge:
  ┌──────────────────────────────────────────┐
  │                                          │
  │  A ◄── B ◄── C ◄── D ◄── G  (main)      │
  │              ▲              │             │
  │              └── E ◄── F ◄─┘             │
  │                     (merge commit G      │
  │                      has TWO parents:    │
  │                      D and F)            │
  └──────────────────────────────────────────┘
```

#### Types of Merge

```
  ┌─────────────────────────────────────────────────────────────────────┐
  │                     MERGE TYPES                                     │
  │                                                                     │
  │  1. FAST-FORWARD MERGE                                              │
  │     (when no divergence — main hasn't moved)                        │
  │                                                                     │
  │     Before:   A ◄── B ◄── C      (main)                            │
  │                           ▲                                         │
  │                           └── D ◄── E   (feature)                   │
  │                                                                     │
  │     After:    A ◄── B ◄── C ◄── D ◄── E   (main, feature)          │
  │                                                                     │
  │     → No merge commit! main just moves forward.                     │
  │     → Linear history (clean)                                        │
  │                                                                     │
  │  2. THREE-WAY MERGE                                                 │
  │     (when both branches have new commits)                           │
  │                                                                     │
  │     Before:   A ◄── B ◄── C ◄── D      (main)                      │
  │                           ▲                                         │
  │                           └── E ◄── F   (feature)                   │
  │                                                                     │
  │     After:    A ◄── B ◄── C ◄── D ◄── G   (main, merge commit)     │
  │                           ▲              │                          │
  │                           └── E ◄── F ◄──┘                          │
  │                                                                     │
  │     → Creates merge commit with 2 parents                           │
  │     → Non-linear history                                            │
  │     → The "three ways": common ancestor (C), main tip (D),          │
  │       feature tip (F)                                               │
  │                                                                     │
  └─────────────────────────────────────────────────────────────────────┘
```

```bash
# Standard merge
git checkout main
git merge feature                    # Fast-forward if possible, else 3-way

# Force merge commit even when fast-forward is possible
git merge --no-ff feature            # Always create merge commit

# Abort a merge in progress (conflict)
git merge --abort

# Squash merge — combine all feature commits into one (no merge commit)
git merge --squash feature
git commit -m "feat: add login"      # You must commit manually after squash
```

> **DevOps Pro Tip:** Many teams enforce `--no-ff` via branch protection rules (GitHub/GitLab). This preserves the "feature branch topology" in history, making it easy to see which commits belonged to which feature — critical for rollbacks.

### 4.3 Rebase — How It Works

Rebase **replays your commits on top of another branch**, creating NEW commits with new hashes.

```
  Before rebase:
  ┌──────────────────────────────────────────────┐
  │                                              │
  │  A ◄── B ◄── C ◄── D      (main)            │
  │              ▲                               │
  │              └── E ◄── F   (feature, HEAD)   │
  │                                              │
  └──────────────────────────────────────────────┘

  $ git checkout feature
  $ git rebase main

  During rebase (internally):
  ┌──────────────────────────────────────────────────────────┐
  │  1. Find common ancestor: C                              │
  │  2. Detach feature commits: E, F                         │
  │  3. Move feature base to main tip: D                     │
  │  4. Replay E as E' on top of D                           │
  │  5. Replay F as F' on top of E'                          │
  │                                                          │
  │  Note: E' and F' have DIFFERENT hashes than E and F      │
  │  because their parent changed → content hash changes     │
  └──────────────────────────────────────────────────────────┘

  After rebase:
  ┌──────────────────────────────────────────────────────────┐
  │                                                          │
  │  A ◄── B ◄── C ◄── D ◄── E' ◄── F'   (feature, HEAD)   │
  │                      ▲                                   │
  │                      main                                │
  │                                                          │
  │  Old E and F still exist but are unreachable             │
  │  (garbage collected after ~30 days)                      │
  │                                                          │
  │  Now: git checkout main && git merge feature             │
  │  → Fast-forward! Clean linear history.                   │
  └──────────────────────────────────────────────────────────┘
```

### 4.4 Merge vs Rebase — Comparison

```
  ┌───────────────────────────────┬───────────────────────────────┐
  │         MERGE                 │          REBASE               │
  ├───────────────────────────────┼───────────────────────────────┤
  │ Preserves exact history       │ Rewrites history              │
  │ Creates merge commit          │ No merge commit (linear)      │
  │ Non-destructive               │ Destructive (new hashes)      │
  │ Safe for public branches      │ DANGEROUS for public branches │
  │ Graph shows branch topology   │ Graph is clean, linear        │
  │ Easy to revert entire feature │ Hard to undo a rebase         │
  │ Can be messy with many merges │ Clean history for review      │
  │                               │                               │
  │ Use when:                     │ Use when:                     │
  │ • Merging to main/production  │ • Updating feature branch     │
  │ • You need clear audit trail  │   with latest main            │
  │ • Branch already pushed       │ • Cleaning up before PR       │
  │                               │ • Local branches only         │
  └───────────────────────────────┴───────────────────────────────┘
```

### 4.5 The Golden Rule of Rebase

```
  ╔═══════════════════════════════════════════════════════════════╗
  ║                                                               ║
  ║   🚨  NEVER REBASE COMMITS THAT HAVE BEEN PUSHED AND         ║
  ║       SHARED WITH OTHERS                                      ║
  ║                                                               ║
  ║   Rebase rewrites history (new commit hashes).                ║
  ║   If others have pulled the old commits, they'll have         ║
  ║   DIVERGENT history. The result:                              ║
  ║                                                               ║
  ║   • Duplicate commits in git log                              ║
  ║   • Merge conflicts on every pull                             ║
  ║   • Confused teammates                                        ║
  ║   • Potential data loss                                       ║
  ║                                                               ║
  ║   SAFE:  Rebase local, unpushed commits                       ║
  ║   SAFE:  Rebase YOUR feature branch (if solo)                 ║
  ║   DANGER: Rebase main, develop, or any shared branch          ║
  ║                                                               ║
  ╚═══════════════════════════════════════════════════════════════╝
```

### 4.6 Rebase Variants

```bash
# Standard rebase
git checkout feature
git rebase main                      # Replay feature commits on top of main

# Interactive rebase (rewrite history)
git rebase -i HEAD~5                 # Edit last 5 commits (pick, squash, reword, etc.)

# Rebase --onto (advanced: transplant commits)
# Move feature commits from old-base to new-base
git rebase --onto new-base old-base feature

# Example: feature was branched from develop, move it to main
git rebase --onto main develop feature

# Rebase conflict resolution
git rebase main
# ... conflict occurs ...
# 1. Fix conflicts in files
# 2. git add <resolved-files>
# 3. git rebase --continue         ← Resume rebase
# OR
# git rebase --abort               ← Cancel and return to pre-rebase state
# OR
# git rebase --skip                ← Skip this commit entirely
```

#### Rebase --onto — Visual

```
  Before:
  A ◄── B ◄── C ◄── D                 (main)
              ▲
              └── E ◄── F             (develop)
                        ▲
                        └── G ◄── H   (feature, based on develop)

  Goal: Move feature to be based on main (not develop)

  $ git rebase --onto main develop feature

  After:
  A ◄── B ◄── C ◄── D ◄── G' ◄── H'  (feature — now on main)
              ▲
              └── E ◄── F             (develop — unchanged)

  Only G and H were replayed (commits unique to feature, after develop)
```

### 4.7 Rebase Edge Cases & Gotchas

| Scenario | Problem | Solution |
|----------|---------|----------|
| Rebase conflicts mid-chain | Each replayed commit can conflict separately | Resolve each, `git rebase --continue` for each |
| Rebase after push | Remote has old hashes, you have new → diverged | `git push --force-with-lease` (safer than `--force`) |
| Rebase public/shared branch | Everyone else's history diverges | **Don't do it.** Use merge instead. |
| Rebase with merge commits | Merge commits are dropped by default | Use `git rebase --rebase-merges` to preserve them |
| Rebase loses commits | Usually because you rebased onto wrong base | `git reflog` to find old branch tip, `git reset --hard <hash>` |

> **⚠️ Gotcha (`--force` vs `--force-with-lease`):** Always use `git push --force-with-lease` instead of `--force`. It fails if someone else pushed to the branch after your last fetch — preventing you from overwriting their work. This should be a team-wide policy.

> **DevOps Relevance:** In CI/CD, when PR branches are rebased before merge, the pipeline must re-run because commit hashes changed. Some CI systems (Jenkins) cache by commit hash — a rebased branch re-triggers the full build. Budget for this in your pipeline design.

---

## 5. Remote Management

### 5.1 What Are Remotes?

A remote is a **bookmark** pointing to another copy of the repository (usually on a server). It's a short name (like `origin`) that maps to a URL.

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                    REMOTE ARCHITECTURE                           │
  │                                                                  │
  │   Your Machine                        Servers                    │
  │  ┌──────────────────┐       ┌────────────────────────┐           │
  │  │  Local Repo      │       │  origin (your fork)    │           │
  │  │  .git/           │◄─────►│  github.com/you/repo   │           │
  │  │                  │ push  │                        │           │
  │  │  Working Dir     │ fetch └────────────────────────┘           │
  │  │  Staging Area    │                                            │
  │  │                  │       ┌────────────────────────┐           │
  │  │  refs/remotes/   │◄──────│  upstream (original)   │           │
  │  │    origin/main   │ fetch │  github.com/org/repo   │           │
  │  │    upstream/main │ only  │                        │           │
  │  └──────────────────┘       └────────────────────────┘           │
  │                                                                  │
  │  Remote tracking branches (refs/remotes/origin/main) are        │
  │  READ-ONLY snapshots of where remote branches were at           │
  │  last fetch/pull. You CANNOT commit to them directly.           │
  └──────────────────────────────────────────────────────────────────┘
```

### 5.2 origin vs upstream — The Fork Model

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │                     FORK WORKFLOW MODEL                              │
  │                                                                      │
  │  ┌───────────────────────┐                                           │
  │  │  upstream             │  ← Original project repo (org/repo)       │
  │  │  (source of truth)    │     You typically have READ-ONLY access   │
  │  └───────────┬───────────┘                                           │
  │              │ Fork (GitHub UI)                                       │
  │              ▼                                                       │
  │  ┌───────────────────────┐                                           │
  │  │  origin               │  ← Your fork (you/repo)                   │
  │  │  (your copy on server)│     You have FULL access (push/pull)      │
  │  └───────────┬───────────┘                                           │
  │              │ git clone                                             │
  │              ▼                                                       │
  │  ┌───────────────────────┐                                           │
  │  │  local                │  ← Your machine                           │
  │  │  (working copy)       │                                           │
  │  └───────────────────────┘                                           │
  │                                                                      │
  │  Typical Flow:                                                       │
  │  1. git fetch upstream          ← Get latest from original           │
  │  2. git merge upstream/main     ← Update local main                  │
  │  3. git checkout -b feature     ← Branch off updated main            │
  │  4. (work, commit)                                                   │
  │  5. git push origin feature     ← Push to YOUR fork                  │
  │  6. Open Pull Request           ← upstream ← origin/feature          │
  └──────────────────────────────────────────────────────────────────────┘
```

### 5.3 Remote Commands

```bash
# List remotes
git remote -v                          # Show remotes with URLs (fetch + push)

# Add a remote
git remote add upstream https://github.com/org/repo.git
git remote add origin git@github.com:you/repo.git

# Remove / rename
git remote remove upstream
git remote rename origin backup

# Change URL (e.g., SSH → HTTPS or vice versa)
git remote set-url origin git@github.com:you/repo.git

# Show detailed remote info
git remote show origin                 # Branches, tracking, stale refs

# Prune stale remote-tracking branches
git remote prune origin                # Remove refs to branches deleted on remote
git fetch --prune                      # Fetch + prune in one command
```

### 5.4 Protected Branches

Protected branches are a **server-side** feature (GitHub, GitLab, Bitbucket) — not a Git feature.

```
  ┌──────────────────────────────────────────────────────────────┐
  │              BRANCH PROTECTION RULES                         │
  │                                                              │
  │  Typical rules for main/production:                          │
  │                                                              │
  │  ┌────────────────────────────────────────────────────┐      │
  │  │ ✅ Require pull request reviews (≥1 approval)      │      │
  │  │ ✅ Require status checks to pass (CI must be green)│      │
  │  │ ✅ Require branches to be up to date before merge  │      │
  │  │ ✅ Require signed commits (GPG)                    │      │
  │  │ ✅ Require linear history (no merge commits)       │      │
  │  │ ❌ Block force pushes                              │      │
  │  │ ❌ Block branch deletion                           │      │
  │  │ ❌ Restrict who can push (admin only)              │      │
  │  └────────────────────────────────────────────────────┘      │
  │                                                              │
  │  DevOps enforces these via:                                  │
  │  • GitHub: Settings → Branches → Branch protection rules     │
  │  • GitLab: Settings → Repository → Protected branches        │
  │  • Terraform: github_branch_protection resource              │
  │  • Policy-as-code: OPA/Gatekeeper for Git policies           │
  └──────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha:** Branch protection only works on the server. Anyone with direct access to a bare repo (e.g., self-hosted Git on a VM) can bypass them. In enterprise, always use a Git hosting platform with RBAC.

> **DevOps Pro Tip:** Define branch protection as code using Terraform's `github_branch_protection` resource. This way, protection rules are version-controlled, auditable, and reproducible across repos.

---

## 6. Pull Strategies

### 6.1 fetch vs pull — Critical Distinction

This is where most engineers get confused. Understanding this prevents accidental history corruption.

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │                  fetch vs pull                                       │
  │                                                                      │
  │  git fetch:                                                          │
  │  ┌───────────────────┐          ┌────────────────────┐               │
  │  │  Remote (origin)  │ ────────►│ Remote-tracking    │               │
  │  │  origin/main: D   │  fetch   │ refs/remotes/      │               │
  │  └───────────────────┘          │ origin/main: D     │               │
  │                                 └────────────────────┘               │
  │                                                                      │
  │  Your local main stays at C. Working directory UNTOUCHED.            │
  │  You decide what to do next (merge? rebase? inspect?)                │
  │                                                                      │
  │  ─────────────────────────────────────────────────────               │
  │                                                                      │
  │  git pull = git fetch + git merge (by default)                       │
  │  ┌───────────────────┐  fetch   ┌────────────────────┐              │
  │  │  Remote (origin)  │ ────────►│ Remote-tracking    │              │
  │  │  origin/main: D   │          │ origin/main: D     │              │
  │  └───────────────────┘          └────────┬───────────┘              │
  │                                          │ merge                    │
  │                                          ▼                          │
  │                                 ┌────────────────────┐              │
  │                                 │  Local main: merge │              │
  │                                 │  commit (C + D → M)│              │
  │                                 └────────────────────┘              │
  │                                                                      │
  │  ⚠️ This creates a merge commit you may not want!                    │
  └──────────────────────────────────────────────────────────────────────┘
```

### 6.2 Pull Variants

```bash
# Default pull (fetch + merge)
git pull                              # Creates merge commit if diverged

# Pull with rebase (fetch + rebase)
git pull --rebase                     # Replays your local commits on top of remote
                                      # Result: linear history, no merge commit

# Fetch only (safest — inspect before integrating)
git fetch origin                      # Download changes, don't touch working dir
git log main..origin/main             # See what's new on remote
git diff main origin/main             # Diff your main vs remote main
git merge origin/main                 # Explicit merge after inspection
# OR
git rebase origin/main                # Explicit rebase after inspection

# Configure default pull strategy
git config --global pull.rebase true          # Always rebase on pull
git config --global pull.ff only              # Only fast-forward, fail otherwise
```

### 6.3 Which Strategy Should DevOps Engineers Use?

```
  ╔═══════════════════════════════════════════════════════════════════╗
  ║                                                                   ║
  ║   RECOMMENDED FOR DEVOPS:                                         ║
  ║                                                                   ║
  ║   git fetch + explicit merge/rebase                               ║
  ║                                                                   ║
  ║   WHY:                                                            ║
  ║   • Implicit operations are dangerous in production               ║
  ║   • fetch lets you INSPECT before integrating                     ║
  ║   • You control whether to merge or rebase                        ║
  ║   • No surprise merge commits in your history                     ║
  ║   • CI/CD scripts should be explicit, never implicit              ║
  ║                                                                   ║
  ║   In CI/CD pipelines:                                             ║
  ║   • Use: git fetch origin && git checkout origin/main             ║
  ║   • NOT: git pull (can fail on divergence, or create merges)      ║
  ║                                                                   ║
  ╚═══════════════════════════════════════════════════════════════════╝
```

### 6.4 Pull Strategy Comparison

| Strategy | Command | Creates Merge Commit? | Rewrites History? | Best For |
|----------|---------|----------------------|-------------------|----------|
| **Merge (default)** | `git pull` | Yes (if diverged) | No | Teams wanting full history |
| **Rebase** | `git pull --rebase` | No | Yes (local only) | Clean linear history |
| **Fast-forward only** | `git pull --ff-only` | No (fails if diverged) | No | Strict linear workflow |
| **Fetch + manual** | `git fetch` + decide | You choose | You choose | DevOps/CI pipelines |

> **⚠️ Gotcha:** If you have uncommitted changes and run `git pull`, it may fail or stash automatically (depending on config). Always commit or stash before pulling. Better yet: `git fetch` first.

---

## 7. Git Workflows — Enterprise Level

### 7.1 Why Workflows Matter

A Git workflow defines **how a team uses branches, merges, and releases**. Wrong workflow = merge hell, broken builds, and slow deployments. The right workflow depends on team size, release cadence, and CI/CD maturity.

### 7.2 Git Flow

The most structured workflow. Best for teams with scheduled releases.

```
  ┌──────────────────────────────────────────────────────────────────────────┐
  │                           GIT FLOW                                       │
  │                                                                          │
  │  main ─────●──────────────────────●──────────────────●────── (releases)  │
  │            │                      ▲                  ▲                   │
  │            │                      │ merge            │ merge             │
  │            ▼                      │                  │                   │
  │  develop ──●──●──●──●──●──●──●───●──●──●──●──●──●───●────── (integration)│
  │               │     ▲  │     ▲                                           │
  │               │     │  │     │                                           │
  │               ▼     │  ▼     │                                           │
  │  feature/  ───●──●──┘  ●──●──┘                                           │
  │  login        (merge)  (merge)                                           │
  │                                                                          │
  │  hotfix/  ─────────────────────────────────●──┐                          │
  │  fix-crash                                     │ merge to BOTH           │
  │                                                ├──► main                 │
  │                                                └──► develop              │
  │                                                                          │
  │  release/  ────────────────●──●──●──┐                                    │
  │  v2.0                     (bugfixes) │ merge to BOTH                     │
  │                                      ├──► main (tag v2.0)               │
  │                                      └──► develop                        │
  │                                                                          │
  │  Branches:                                                               │
  │  ┌──────────┬────────────────────────────────────────────────────┐       │
  │  │ main     │ Production-ready code. Every merge = a release.    │       │
  │  │ develop  │ Integration branch. Features merge here first.     │       │
  │  │ feature/*│ New features. Branch from & merge to develop.      │       │
  │  │ release/*│ Release prep. Branch from develop, merge to both.  │       │
  │  │ hotfix/* │ Emergency prod fix. Branch from main, merge both.  │       │
  │  └──────────┴────────────────────────────────────────────────────┘       │
  └──────────────────────────────────────────────────────────────────────────┘
```

**Pros:**
- Clear separation of concerns
- Parallel development of features
- Hotfix path doesn't disrupt feature work

**Cons:**
- Complex — many branches to manage
- Slow for CI/CD (feature → develop → release → main = many merges)
- Overkill for small teams or continuous deployment

### 7.3 Trunk-Based Development

The preferred workflow for high-performing CI/CD teams.

```
  ┌──────────────────────────────────────────────────────────────────────────┐
  │                    TRUNK-BASED DEVELOPMENT                               │
  │                                                                          │
  │  main (trunk) ──●──●──●──●──●──●──●──●──●──●──●──●──●──── (always       │
  │                  ▲  ▲     ▲     ▲  ▲                        deployable)  │
  │                  │  │     │     │  │                                     │
  │  short-lived  ───●──┘  ───●─────┘  │                                     │
  │  branches         │       │        │                                     │
  │  (< 1-2 days)  ──●──●────┘   ─────●──┘                                  │
  │                                                                          │
  │  Key Rules:                                                              │
  │  ┌─────────────────────────────────────────────────────────────────┐     │
  │  │ 1. main is ALWAYS deployable                                    │     │
  │  │ 2. Branches are SHORT-LIVED (hours to 1-2 days max)            │     │
  │  │ 3. Merge to main FREQUENTLY (multiple times per day)           │     │
  │  │ 4. Feature flags control incomplete features in production      │     │
  │  │ 5. CI runs on EVERY commit to main                             │     │
  │  │ 6. CD deploys main automatically (or with one-click)           │     │
  │  └─────────────────────────────────────────────────────────────────┘     │
  │                                                                          │
  │  Release mechanism:                                                      │
  │  • Tags on main: v1.2.3, v1.2.4                                         │
  │  • OR: Release branches cut from main (for hotfixes to old versions)     │
  │  • Feature flags: features merged but hidden behind flags               │
  └──────────────────────────────────────────────────────────────────────────┘
```

**Pros:**
- Fastest feedback loop (CI/CD on every merge)
- Minimal merge conflicts (short-lived branches)
- Aligns with DevOps/SRE culture
- Google, Meta, Netflix use this

**Cons:**
- Requires strong CI/CD infrastructure
- Requires feature flags for incomplete work
- Team must be disciplined about small, frequent merges

### 7.4 Forking Workflow

Used by open-source projects and organizations with external contributors.

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │                      FORKING WORKFLOW                                │
  │                                                                      │
  │  ┌────────────────────────┐                                          │
  │  │  upstream (org/repo)   │  ← Blessed repo (maintainers only)       │
  │  │  main: A─B─C─D─E      │                                          │
  │  └───────────┬────────────┘                                          │
  │              │ Fork                                                  │
  │  ┌───────────▼────────────┐                                          │
  │  │  origin (you/repo)     │  ← Your fork (full push access)          │
  │  │  main: A─B─C─D─E      │                                          │
  │  │  feature: A─B─C─X─Y   │                                          │
  │  └───────────┬────────────┘                                          │
  │              │ clone                                                 │
  │  ┌───────────▼────────────┐                                          │
  │  │  local (your machine)  │                                          │
  │  │  main: A─B─C─D─E      │                                          │
  │  │  feature: A─B─C─X─Y   │                                          │
  │  └────────────────────────┘                                          │
  │                                                                      │
  │  Flow:                                                               │
  │  1. Fork upstream → origin                                           │
  │  2. Clone origin → local                                             │
  │  3. Add upstream remote: git remote add upstream <url>               │
  │  4. Create feature branch, commit, push to origin                    │
  │  5. Open PR: upstream/main ← origin/feature                          │
  │  6. Maintainers review & merge                                       │
  │  7. Sync: git fetch upstream && git merge upstream/main              │
  └──────────────────────────────────────────────────────────────────────┘
```

### 7.5 Monorepo Strategy

Multiple projects in a single repo (Google, Meta, Microsoft approach).

```
  ┌──────────────────────────────────────────────────────────────┐
  │                    MONOREPO LAYOUT                            │
  │                                                              │
  │  repo/                                                       │
  │  ├── services/                                               │
  │  │   ├── api-gateway/        ← Microservice 1                │
  │  │   ├── auth-service/       ← Microservice 2                │
  │  │   └── payment-service/    ← Microservice 3                │
  │  ├── libs/                                                   │
  │  │   ├── common-utils/       ← Shared library                │
  │  │   └── proto-definitions/  ← Shared protobuf              │
  │  ├── infra/                                                  │
  │  │   ├── terraform/          ← Infrastructure code           │
  │  │   └── k8s-manifests/      ← Kubernetes configs            │
  │  ├── .github/workflows/      ← CI/CD (path-based triggers)  │
  │  └── nx.json / turbo.json    ← Build orchestrator config     │
  │                                                              │
  │  Key tooling:                                                │
  │  • Nx, Turborepo, Bazel, Pants — build only what changed    │
  │  • Path-based CI triggers — only build affected services     │
  │  • CODEOWNERS — per-directory ownership                      │
  │  • Sparse checkout — clone only what you need                │
  └──────────────────────────────────────────────────────────────┘
```

### 7.6 Workflow Comparison — Trade-offs

| Aspect | Git Flow | Trunk-Based | Forking | Monorepo |
|--------|----------|-------------|---------|----------|
| **Complexity** | High | Low | Medium | High (tooling) |
| **Release cadence** | Scheduled | Continuous | PR-based | Continuous |
| **Branch lifespan** | Long (weeks) | Short (hours-days) | Medium (days) | Short |
| **Merge conflicts** | Frequent | Rare | Moderate | Depends on ownership |
| **CI/CD fit** | Moderate | Excellent | Good | Excellent (with tooling) |
| **Team size** | Medium-Large | Any | Open source / large | Large (with discipline) |
| **Best for** | Enterprise with release cycles | DevOps-heavy, CD | Open source, contractors | Platform teams, shared libs |

> **DevOps Pro Tip:** Don't blindly adopt Git Flow because "everyone uses it." If you're deploying multiple times per day (CD), trunk-based is almost always better. Git Flow was designed for scheduled releases — a pattern that's increasingly rare in cloud-native environments.

> **⚠️ Gotcha (Monorepo):** Without proper tooling (Nx/Turborepo/Bazel), a monorepo becomes a "monolith in disguise." CI builds everything on every commit → pipeline takes hours. Path-based triggers and build caching are mandatory.

---

## 8. Interactive Rebase

### 8.1 Why Interactive Rebase Is Essential

Interactive rebase (`git rebase -i`) is the most powerful history editing tool in Git. It lets you rewrite, reorder, combine, split, and delete commits **before sharing them**.

> **DevOps Rule:** Clean commits → clean `git log` → easier debugging → faster incident response. Interactive rebase is how you maintain commit hygiene.

### 8.2 The Commands

```bash
# Rebase the last N commits interactively
git rebase -i HEAD~5                  # Edit the last 5 commits

# Rebase from a specific commit (exclusive — that commit is not edited)
git rebase -i abc1234                 # Edit all commits after abc1234

# Rebase onto another branch interactively
git rebase -i main                    # Edit commits from branch point to HEAD
```

When you run `git rebase -i`, Git opens an editor with a "todo list":

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  Interactive Rebase TODO File                                    │
  │                                                                  │
  │  pick   a1b2c3d  feat: add user authentication                   │
  │  pick   e4f5g6h  fix: typo in auth module                        │
  │  pick   i7j8k9l  feat: add password hashing                      │
  │  pick   m0n1o2p  wip: debugging stuff                            │
  │  pick   q3r4s5t  feat: add session management                    │
  │                                                                  │
  │  # Commands:                                                     │
  │  # p, pick   = use commit as is                                  │
  │  # r, reword = use commit, but edit the commit message           │
  │  # e, edit   = use commit, but pause for amending                │
  │  # s, squash = meld into previous commit (keep message)          │
  │  # f, fixup  = like squash, but discard this commit's message    │
  │  # d, drop   = remove commit entirely                            │
  │  # x, exec   = run a shell command                               │
  │  # b, break  = stop here (continue with 'git rebase --continue') │
  └──────────────────────────────────────────────────────────────────┘
```

### 8.3 Interactive Rebase Actions — Deep Dive

| Action | Keyword | What It Does | Use Case |
|--------|---------|-------------|----------|
| **Keep as-is** | `pick` | Use commit unchanged | Default — no change |
| **Edit message** | `reword` | Pause to edit commit message | Fix typo in message, add ticket ref |
| **Pause & amend** | `edit` | Stop at this commit, let you amend files + message | Split a commit, add forgotten file |
| **Merge up** | `squash` | Combine with previous commit, keep both messages | Combine related commits |
| **Merge up silently** | `fixup` | Combine with previous commit, discard this message | Merge "fix typo" into parent commit |
| **Remove** | `drop` | Delete commit entirely | Remove WIP commits, debug code |
| **Run command** | `exec` | Execute shell command between commits | Run tests between each commit |
| **Pause** | `break` | Pause rebase, resume with `--continue` | Manual inspection mid-rebase |

### 8.4 Common Interactive Rebase Recipes

#### Recipe 1: Squash "Fix" Commits Into Their Parent

```
  Before (messy):                       After (clean):
  ┌────────────────────────┐            ┌────────────────────────┐
  │ pick  feat: add login  │            │ pick  feat: add login  │
  │ pick  fix: typo        │  ────►     │       (includes typo   │
  │ pick  fix: forgot file │            │        fix + file)     │
  │ pick  feat: add logout │            │ pick  feat: add logout │
  └────────────────────────┘            └────────────────────────┘

  TODO file:
  pick   a1b2c3d  feat: add login
  fixup  e4f5g6h  fix: typo                ← absorbed into above
  fixup  i7j8k9l  fix: forgot file         ← absorbed into above
  pick   m0n1o2p  feat: add logout
```

#### Recipe 2: Reorder Commits

```
  Before:                               After:
  pick  feat: add login                  pick  feat: add logout    ← moved up
  pick  feat: add dashboard              pick  feat: add login
  pick  feat: add logout                 pick  feat: add dashboard

  Simply reorder the lines in the TODO file.
```

#### Recipe 3: Split a Commit

```bash
# 1. Start interactive rebase
git rebase -i HEAD~3

# 2. Change 'pick' to 'edit' on the commit to split
# edit  a1b2c3d  feat: add auth + logging (too big!)

# 3. Git pauses at that commit. Reset it:
git reset HEAD~1                 # Undo the commit, keep changes in working dir

# 4. Stage and commit separately:
git add auth.py
git commit -m "feat: add authentication"
git add logging.py
git commit -m "feat: add logging"

# 5. Continue rebase:
git rebase --continue
```

#### Recipe 4: Remove Sensitive Data From History

```bash
# 1. Interactive rebase to the commit BEFORE the sensitive one
git rebase -i HEAD~10

# 2. Change 'pick' to 'edit' on the commit with secrets

# 3. Git pauses. Remove the secret:
git rm --cached secrets.env
echo "secrets.env" >> .gitignore
git add .gitignore
git commit --amend                   # Amend the commit without the secret

# 4. Continue:
git rebase --continue

# 5. Force push (history rewritten):
git push --force-with-lease

# ⚠️ This only works if the secret hasn't been pushed to a public branch.
# For already-pushed secrets, use git filter-repo (Section 11).
```

### 8.5 Autosquash — The Professional Workflow

```bash
# When you commit a fix intended to be squashed later:
git commit --fixup=a1b2c3d         # Creates: "fixup! feat: add login"
# OR
git commit --squash=a1b2c3d        # Creates: "squash! feat: add login"

# Later, when cleaning up:
git rebase -i --autosquash main    # fixup!/squash! commits auto-placed correctly

# Make it default:
git config --global rebase.autoSquash true
```

> **DevOps Pro Tip:** Set `rebase.autoSquash = true` globally. When engineers use `--fixup` commits during development, `git rebase -i --autosquash` before merging creates clean history automatically. Enforce this pattern in your team's contributing guide.

---

## 9. Reset vs Restore vs Revert

### 9.1 Why This Section Exists

These three commands confuse 90% of engineers because they sound similar but do fundamentally different things. Getting them wrong can destroy work or corrupt shared history.

### 9.2 Mental Model — The Three R's

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │                THE THREE R's — QUICK MENTAL MODEL                    │
  │                                                                      │
  │  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────────┐   │
  │  │   RESET      │  │   RESTORE    │  │        REVERT             │   │
  │  │              │  │              │  │                           │   │
  │  │  Moves branch│  │  Restores    │  │  Creates NEW commit      │   │
  │  │  pointer     │  │  files       │  │  that undoes an old      │   │
  │  │  backward    │  │  (working    │  │  commit's changes        │   │
  │  │              │  │   dir or     │  │                           │   │
  │  │  Can destroy │  │   staging)   │  │  SAFE for shared         │   │
  │  │  history     │  │              │  │  branches (preserves     │   │
  │  │              │  │  SAFE —      │  │  history)                │   │
  │  │  DANGEROUS   │  │  never loses │  │                           │   │
  │  │  on shared   │  │  commits     │  │  Think: "undo commit     │   │
  │  │  branches    │  │              │  │   by applying inverse"   │   │
  │  └──────────────┘  └──────────────┘  └───────────────────────────┘   │
  │                                                                      │
  │  Decision tree:                                                      │
  │  ┌──────────────────────────────────────────────────────────────┐    │
  │  │  Want to undo a commit on a SHARED branch?  → git revert    │    │
  │  │  Want to undo a commit on a LOCAL branch?   → git reset     │    │
  │  │  Want to restore a FILE to a prior state?   → git restore   │    │
  │  │  Want to unstage a file?                    → git restore   │    │
  │  └──────────────────────────────────────────────────────────────┘    │
  └──────────────────────────────────────────────────────────────────────┘
```

### 9.3 git reset — Move the Branch Pointer

`git reset` moves the current branch pointer to a different commit. It has three modes that differ in what happens to the staging area and working directory.

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │              git reset — THREE MODES                                  │
  │                                                                      │
  │  Starting state:                                                     │
  │  A ◄── B ◄── C ◄── D    (HEAD, main)                                │
  │                                                                      │
  │  $ git reset --soft B                                                │
  │  ┌───────────────────────────────────────────────────────┐           │
  │  │  Branch pointer: moves to B                           │           │
  │  │  Staging area:   KEEPS changes from C and D (staged)  │           │
  │  │  Working dir:    KEEPS changes from C and D           │           │
  │  │  Use case:       Re-commit with different message     │           │
  │  │                  or combine C+D into one commit       │           │
  │  └───────────────────────────────────────────────────────┘           │
  │                                                                      │
  │  $ git reset --mixed B    (DEFAULT — no flag needed)                 │
  │  ┌───────────────────────────────────────────────────────┐           │
  │  │  Branch pointer: moves to B                           │           │
  │  │  Staging area:   CLEARED (changes from C,D unstaged)  │           │
  │  │  Working dir:    KEEPS changes from C and D           │           │
  │  │  Use case:       Re-stage selectively (git add -p)    │           │
  │  └───────────────────────────────────────────────────────┘           │
  │                                                                      │
  │  $ git reset --hard B                                                │
  │  ┌───────────────────────────────────────────────────────┐           │
  │  │  Branch pointer: moves to B                           │           │
  │  │  Staging area:   CLEARED                              │           │
  │  │  Working dir:    REVERTED to B (changes GONE) ⚠️       │           │
  │  │  Use case:       Complete undo — discard everything   │           │
  │  │  DANGER:         Uncommitted changes are LOST FOREVER │           │
  │  └───────────────────────────────────────────────────────┘           │
  │                                                                      │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# Reset examples
git reset --soft HEAD~1              # Undo last commit, keep changes staged
git reset --mixed HEAD~1             # Undo last commit, keep changes unstaged
git reset --hard HEAD~1              # Undo last commit, DESTROY all changes
git reset --hard origin/main         # Reset local main to match remote exactly

# Reset a single file (unstage)
git reset HEAD file.txt              # Unstage file (old way)
```

### 9.4 git restore — Restore Files (Git 2.23+)

`git restore` was introduced to **separate file restoration from branch manipulation** (which `git checkout` used to do for both).

```bash
# Restore working directory file to last committed state
git restore file.txt                         # Discard uncommitted changes in file

# Restore file from staging area (unstage)
git restore --staged file.txt                # Unstage, keep changes in working dir

# Restore file from a specific commit
git restore --source=HEAD~3 file.txt         # Get file as it was 3 commits ago

# Restore all files
git restore .                                # Discard ALL uncommitted changes
git restore --staged .                       # Unstage ALL files
```

```
  ┌──────────────────────────────────────────────────────────────┐
  │         git restore — What It Affects                        │
  │                                                              │
  │  git restore file.txt                                        │
  │    Staging: untouched                                        │
  │    Working: restored from staging (or HEAD if not staged)    │
  │                                                              │
  │  git restore --staged file.txt                               │
  │    Staging: restored from HEAD (unstaged)                    │
  │    Working: untouched                                        │
  │                                                              │
  │  git restore --staged --worktree file.txt                    │
  │    Staging: restored from HEAD                               │
  │    Working: restored from HEAD                               │
  │    (equivalent to: "completely undo all changes")            │
  │                                                              │
  │  KEY: git restore NEVER moves branch pointers.               │
  │  It ONLY affects files. This is why it's safer than reset.   │
  └──────────────────────────────────────────────────────────────┘
```

### 9.5 git revert — Safe Undo for Shared History

`git revert` creates a **new commit** that is the inverse of the target commit. History is preserved — nothing is rewritten.

```
  Before revert:
  A ◄── B ◄── C ◄── D      (main, HEAD)

  $ git revert C             # Undo the changes introduced by C

  After revert:
  A ◄── B ◄── C ◄── D ◄── C'     (main, HEAD)
                              │
                              └── C' is the INVERSE of C
                                  (if C added a line, C' removes it)
                                  (if C deleted a file, C' restores it)

  History is preserved — C still exists.
  C' explicitly shows "we undid C" — auditable!
```

```bash
# Revert a single commit
git revert abc1234                   # Creates inverse commit

# Revert without auto-committing (stage changes, commit manually)
git revert --no-commit abc1234       # Stage inverse changes, you commit

# Revert a merge commit (must specify parent)
git revert -m 1 <merge-commit>      # -m 1 = keep mainline, undo feature
                                     # -m 2 = keep feature, undo mainline

# Revert a range of commits
git revert HEAD~3..HEAD              # Revert last 3 commits (3 new revert commits)
```

> **⚠️ Gotcha (Reverting Merge Commits):** Reverting a merge commit with `git revert -m 1` undoes the feature BUT Git considers the merge "already done." If you try to re-merge that feature branch later, Git says "already up to date." You must **revert the revert** (`git revert <revert-commit>`) before re-merging. This catches teams off guard in production hotfix scenarios.

### 9.6 Comparison Table — Reset vs Restore vs Revert

| Aspect | `git reset` | `git restore` | `git revert` |
|--------|-------------|---------------|---------------|
| **What it does** | Moves branch pointer | Restores file content | Creates inverse commit |
| **Affects history?** | Yes (removes commits from branch) | No | No (adds new commit) |
| **Safe for shared branches?** | ❌ No (rewrites history) | ✅ Yes | ✅ Yes |
| **Can lose data?** | `--hard` loses uncommitted work | Discards working dir changes | No |
| **Scope** | Whole branch / commits | Individual files | Individual commits |
| **When to use** | Local cleanup, undo local commits | Restore files, unstage | Undo commits on shared branches |

### 9.7 Production Rules

```
  ╔════════════════════════════════════════════════════════════════════╗
  ║                    PRODUCTION SAFETY RULES                        ║
  ║                                                                    ║
  ║  1. NEVER use git reset --hard on shared/pushed branches           ║
  ║     → Use git revert instead                                       ║
  ║                                                                    ║
  ║  2. NEVER use git reset --hard if you have uncommitted work        ║
  ║     → Stash first: git stash                                       ║
  ║                                                                    ║
  ║  3. Undo a pushed commit?                                          ║
  ║     → git revert (always)                                          ║
  ║                                                                    ║
  ║  4. Undo a local unpushed commit?                                  ║
  ║     → git reset --soft HEAD~1 (keep changes)                       ║
  ║     → git reset --hard HEAD~1 (discard changes)                    ║
  ║                                                                    ║
  ║  5. Accidentally ran reset --hard?                                 ║
  ║     → git reflog → find the commit → git reset --hard <hash>       ║
  ║     → Uncommitted changes are gone forever (no recovery)           ║
  ║                                                                    ║
  ║  6. In CI/CD pipelines:                                            ║
  ║     → NEVER use reset or revert automatically                      ║
  ║     → Rollback = redeploy previous tagged release                  ║
  ╚════════════════════════════════════════════════════════════════════╝
```

> **DevOps Relevance:** In GitOps workflows (ArgoCD, Flux), `git revert` is the standard rollback mechanism. Revert the bad commit → ArgoCD detects the change → automatically deploys the previous state. This is why clean, atomic commits matter — you need to be able to revert a single commit without breaking everything.

---

## 10. Disaster Recovery — Reflog & Beyond

### 10.1 Why This Is Critical for DevOps

Production incidents caused by Git mistakes are more common than you think:
- Someone ran `git reset --hard` on a shared branch
- A force push wiped out a colleague's commits
- A branch was accidentally deleted
- An interactive rebase went wrong and "lost" commits

If you know the reflog, **you can recover from almost anything**.

### 10.2 What Is the Reflog?

The reflog (reference log) is a **local, per-repo journal** of every time HEAD or a branch ref changes. It records every checkout, commit, reset, rebase, merge, pull — anything that moves a pointer.

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │                    THE REFLOG — YOUR TIME MACHINE                    │
  │                                                                      │
  │  Regular git log:                                                    │
  │  Shows commits reachable from current branch.                        │
  │  If you reset --hard, those commits DISAPPEAR from git log.          │
  │                                                                      │
  │  Reflog:                                                             │
  │  Shows EVERY position HEAD has ever been in.                         │
  │  Even after reset --hard, the old commits are in the reflog.         │
  │                                                                      │
  │  ┌─────────────────────────────────────────────────────────────┐     │
  │  │  git log (after reset --hard to B):                         │     │
  │  │                                                             │     │
  │  │  A ◄── B        (main, HEAD)                                │     │
  │  │                                                             │     │
  │  │  C, D are GONE from log — unreachable!                      │     │
  │  └─────────────────────────────────────────────────────────────┘     │
  │                                                                      │
  │  ┌─────────────────────────────────────────────────────────────┐     │
  │  │  git reflog:                                                │     │
  │  │                                                             │     │
  │  │  b2c3d4e HEAD@{0}: reset: moving to B                      │     │
  │  │  d4e5f6g HEAD@{1}: commit: feat: add session (D)           │     │
  │  │  c3d4e5f HEAD@{2}: commit: feat: add auth (C)              │     │
  │  │  b2c3d4e HEAD@{3}: commit: initial setup (B)               │     │
  │  │  a1b2c3d HEAD@{4}: commit (initial): init (A)              │     │
  │  │                                                             │     │
  │  │  D is still at HEAD@{1} — recoverable!                      │     │
  │  └─────────────────────────────────────────────────────────────┘     │
  │                                                                      │
  │  ⚠️ IMPORTANT LIMITS:                                                │
  │  • Reflog is LOCAL ONLY — not pushed to remote                       │
  │  • Entries expire: 90 days (reachable), 30 days (unreachable)        │
  │  • git gc can clean up unreachable objects                           │
  │  • Reflog does NOT survive re-cloning                                │
  └──────────────────────────────────────────────────────────────────────┘
```

### 10.3 Reflog Commands

```bash
# View reflog for HEAD
git reflog                            # Short format
git reflog show HEAD                  # Same thing
git reflog show main                  # Reflog for a specific branch

# Verbose output with dates
git reflog --date=iso                 # Show ISO timestamps
git reflog --date=relative            # "2 hours ago", "3 days ago"

# Search reflog for specific actions
git reflog | grep "reset"             # Find all resets
git reflog | grep "rebase"            # Find all rebases
git reflog | grep "checkout"          # Find all branch switches

# Show diff between reflog entries
git diff HEAD@{0} HEAD@{3}           # What changed between two reflog points
git show HEAD@{2}                     # Show the commit at that reflog entry
```

### 10.4 Recovery Scenarios — Step by Step

#### Scenario 1: Recover After `git reset --hard`

```
  Problem: You ran git reset --hard HEAD~3 and lost 3 commits

  ┌────────────────────────────────────────────────────────┐
  │  Step 1: Find the lost commit in reflog                │
  │                                                        │
  │  $ git reflog                                          │
  │  a1b2c3d HEAD@{0}: reset: moving to HEAD~3             │
  │  f4e5d6c HEAD@{1}: commit: feat: the commit you want   │  ← HERE
  │  ...                                                   │
  │                                                        │
  │  Step 2: Reset back to the lost commit                 │
  │                                                        │
  │  $ git reset --hard f4e5d6c                            │
  │  # OR                                                  │
  │  $ git reset --hard HEAD@{1}                           │
  │                                                        │
  │  ✅ All 3 commits are restored!                         │
  └────────────────────────────────────────────────────────┘
```

#### Scenario 2: Recover a Deleted Branch

```
  Problem: You deleted a branch with important commits

  $ git branch -D feature-critical    # Oops!

  ┌────────────────────────────────────────────────────────┐
  │  Step 1: Find the branch tip in reflog                 │
  │                                                        │
  │  $ git reflog | grep "feature-critical"                │
  │  abc1234 HEAD@{5}: checkout: moving from feature-      │
  │          critical to main                              │
  │                                                        │
  │  OR: look for the last commit on that branch           │
  │  $ git reflog                                          │
  │  ...find the commit hash...                            │
  │                                                        │
  │  Step 2: Recreate the branch at that commit            │
  │                                                        │
  │  $ git branch feature-critical abc1234                 │
  │                                                        │
  │  ✅ Branch is back with all its commits!                │
  └────────────────────────────────────────────────────────┘
```

#### Scenario 3: Recover After a Bad Rebase

```
  Problem: Interactive rebase went wrong, commits are mangled

  ┌────────────────────────────────────────────────────────┐
  │  Step 1: Find the pre-rebase state                     │
  │                                                        │
  │  $ git reflog                                          │
  │  e5f6g7h HEAD@{0}: rebase (finish): ...                │
  │  d4e5f6g HEAD@{1}: rebase (pick): ...                  │
  │  c3d4e5f HEAD@{2}: rebase (start): ...                 │
  │  b2c3d4e HEAD@{3}: commit: your last good commit  ← HERE
  │                                                        │
  │  Step 2: Reset to pre-rebase state                     │
  │                                                        │
  │  $ git reset --hard HEAD@{3}                           │
  │  # OR use ORIG_HEAD (Git sets this before rebase):     │
  │  $ git reset --hard ORIG_HEAD                          │
  │                                                        │
  │  ✅ Back to before the rebase started!                  │
  └────────────────────────────────────────────────────────┘
```

#### Scenario 4: Recover Detached HEAD Commits

```
  Problem: You committed in detached HEAD, then switched to main.
  Those commits have no branch — they're orphaned.

  ┌────────────────────────────────────────────────────────┐
  │  Step 1: Find the orphaned commits                     │
  │                                                        │
  │  $ git reflog                                          │
  │  a1b2c3d HEAD@{0}: checkout: moving from f4e5d6c       │
  │          to main                                       │
  │  f4e5d6c HEAD@{1}: commit: work done in detached  ← HERE
  │                                                        │
  │  Step 2: Create a branch to rescue them                │
  │                                                        │
  │  $ git branch rescue-work f4e5d6c                      │
  │                                                        │
  │  Step 3: Merge or cherry-pick into your branch         │
  │                                                        │
  │  $ git checkout main                                   │
  │  $ git merge rescue-work                               │
  │                                                        │
  │  ✅ Orphaned commits are now on a proper branch!        │
  └────────────────────────────────────────────────────────┘
```

### 10.5 Nuclear Option: git fsck

When the reflog isn't enough (e.g., reflog expired), `git fsck` finds ALL unreachable objects (dangling commits, blobs).

```bash
# Find dangling (unreachable) commits
git fsck --unreachable --no-reflogs

# Find dangling commits specifically
git fsck --lost-found                 # Writes to .git/lost-found/

# Inspect a dangling commit
git show <dangling-commit-hash>

# Cherry-pick or branch from it
git cherry-pick <dangling-commit-hash>
git branch recovered <dangling-commit-hash>
```

### 10.6 Reflog Retention & Garbage Collection

```
  ┌──────────────────────────────────────────────────────────────────┐
  │              REFLOG RETENTION RULES                               │
  │                                                                  │
  │  Reachable entries (still on a branch):                          │
  │    Default: 90 days                                              │
  │    Config:  gc.reflogExpire                                      │
  │                                                                  │
  │  Unreachable entries (orphaned commits):                         │
  │    Default: 30 days                                              │
  │    Config:  gc.reflogExpireUnreachable                           │
  │                                                                  │
  │  After expiry, git gc removes the reflog entries AND             │
  │  the unreachable objects they pointed to.                        │
  │                                                                  │
  │  ⚠️ In CI/CD environments:                                      │
  │  • CI runners often do fresh clones → NO reflog history          │
  │  • Shared build agents may run git gc → reduces recovery window  │
  │  • For critical repos: set longer retention on the server        │
  │                                                                  │
  │  Extend retention:                                               │
  │  git config gc.reflogExpire "180 days"                           │
  │  git config gc.reflogExpireUnreachable "90 days"                 │
  └──────────────────────────────────────────────────────────────────┘
```

> **DevOps Pro Tip:** On self-hosted Git servers (Gitea, GitLab self-managed), configure reflog retention and gc schedules carefully. The defaults are fine for developer machines, but shared repos may need longer retention for audit and recovery. Document the gc policy in your runbook.

---

## 11. Handling Secrets & Security

### 11.1 Why This Is CRITICAL for DevOps

This is not optional — it's a **security responsibility**:
- API keys, database credentials, cloud access keys pushed to Git → **instant compromise**
- Git history is permanent — even if you delete the file, the blob lives in history
- Secrets in Git = secrets in every clone, every fork, every CI runner cache
- Regulatory frameworks (SOC2, HIPAA, PCI-DSS) require secret management controls

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │              THE SECRET LEAK TIMELINE                                 │
  │                                                                      │
  │  Developer commits AWS key ──► Push to GitHub ──► GitHub secret      │
  │                                                    scanning alert    │
  │                                                    (if enabled)      │
  │                                                         │            │
  │  Meanwhile:                                             │            │
  │  ┌──────────────────────────────────────────┐           │            │
  │  │  Bot scrapes public repos in < 5 min     │           │            │
  │  │  → Spins up crypto miners on your AWS    │           │            │
  │  │  → $50K bill by morning                  │           │            │
  │  └──────────────────────────────────────────┘           │            │
  │                                                         │            │
  │  Even after you delete the file:                        ▼            │
  │  ┌──────────────────────────────────────────┐  ┌──────────────┐     │
  │  │  The blob is STILL in Git history        │  │  Every fork  │     │
  │  │  Every clone has the secret              │  │  has it too  │     │
  │  │  git log -p shows it                     │  └──────────────┘     │
  │  │  Deleting the commit is NOT enough       │                       │
  │  └──────────────────────────────────────────┘                       │
  │                                                                      │
  │  CORRECT RESPONSE:                                                   │
  │  1. Rotate/invalidate the credential IMMEDIATELY                     │
  │  2. Remove from history (git filter-repo or BFG)                     │
  │  3. Force push the rewritten history                                 │
  │  4. Notify the team to re-clone                                      │
  │  5. Post-mortem: add pre-commit hook to prevent recurrence           │
  └──────────────────────────────────────────────────────────────────────┘
```

### 11.2 Prevention — Stop Secrets Before They Enter

#### .gitignore — The First Line of Defense

```bash
# .gitignore — essential patterns for DevOps repos
# ─────────────────────────────────────────────────

# Environment files
.env
.env.*
*.env

# Credentials & keys
*.pem
*.key
*.p12
*.pfx
*.jks
credentials.json
service-account*.json

# Terraform state (contains secrets!)
*.tfstate
*.tfstate.backup
.terraform/

# Ansible vault files (if unencrypted)
*vault*.yml
!*vault*.yml.example

# IDE & OS artifacts
.idea/
.vscode/settings.json
.DS_Store
Thumbs.db

# Build artifacts
node_modules/
__pycache__/
*.pyc
dist/
build/
```

#### .git/info/exclude — Repo-Local Ignores (Not Shared)

```bash
# For personal ignores that shouldn't be in .gitignore
# (e.g., your local test scripts, debug configs)
echo "my-local-test.sh" >> .git/info/exclude
```

> **⚠️ Gotcha:** `.gitignore` only prevents **untracked** files from being staged. If a file is already tracked (was committed before being added to `.gitignore`), it will continue to be tracked. You must explicitly remove it:
> ```bash
> git rm --cached secrets.env          # Remove from tracking, keep local file
> echo "secrets.env" >> .gitignore     # Prevent future tracking
> git commit -m "chore: remove tracked secret file"
> ```

#### Pre-commit Hooks — Automated Secret Detection

```bash
# Install detect-secrets (Yelp's tool)
pip install detect-secrets

# Create baseline
detect-secrets scan > .secrets.baseline

# Pre-commit hook integration (.pre-commit-config.yaml)
repos:
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']

# Other tools:
# • gitleaks — fast, Go-based, CI-friendly
# • truffleHog — deep history scanning
# • git-secrets (AWS) — prevents AWS key commits
```

```
  ┌──────────────────────────────────────────────────────────────┐
  │          PRE-COMMIT SECRET DETECTION FLOW                    │
  │                                                              │
  │  Developer runs git commit                                   │
  │         │                                                    │
  │         ▼                                                    │
  │  ┌──────────────────────┐                                    │
  │  │  pre-commit hook     │                                    │
  │  │  runs detect-secrets │                                    │
  │  │  / gitleaks          │                                    │
  │  └──────────┬───────────┘                                    │
  │             │                                                │
  │     ┌───────┴───────┐                                        │
  │     │               │                                        │
  │  No secrets      Secrets found                               │
  │     │               │                                        │
  │     ▼               ▼                                        │
  │  ✅ Commit       ❌ Commit BLOCKED                            │
  │  proceeds        "Possible secret detected in config.py:12"  │
  │                  Developer must remove secret or add          │
  │                  to allowlist (.secrets.baseline)             │
  └──────────────────────────────────────────────────────────────┘
```

### 11.3 Removal — Purging Secrets From History

When prevention fails, you must rewrite history to remove the secret.

#### git filter-repo (Recommended — Modern)

```bash
# Install
pip install git-filter-repo

# Remove a specific file from ALL history
git filter-repo --invert-paths --path secrets.env

# Remove a specific string/pattern from ALL files in ALL history
git filter-repo --replace-text expressions.txt
# expressions.txt contains:
# AKIAIOSFODNN7EXAMPLE==>***REDACTED***
# regex:password\s*=\s*['"][^'"]+[']==>password=***REDACTED***

# Remove a directory from ALL history
git filter-repo --invert-paths --path-glob '*.pem'

# After filter-repo:
git push --force --all                # Force push ALL branches
git push --force --tags               # Force push ALL tags
```

#### BFG Repo Cleaner (Fast, Simple — Legacy Alternative)

```bash
# Install (requires Java)
# Download from: https://rtyley.github.io/bfg-repo-cleaner/

# Remove files by name
java -jar bfg.jar --delete-files secrets.env repo.git

# Replace text in all history
java -jar bfg.jar --replace-text passwords.txt repo.git
# passwords.txt contains:
# MyS3cretP@ss123

# Remove files larger than 100M
java -jar bfg.jar --strip-blobs-bigger-than 100M repo.git

# Clean up after BFG
cd repo.git
git reflog expire --expire=now --all
git gc --prune=now --aggressive
git push --force --all
git push --force --tags
```

#### git filter-branch (Legacy — Avoid)

```bash
# ⚠️ Deprecated in favor of git filter-repo
# Extremely slow on large repos
# Only shown for legacy reference — do NOT use in new workflows

git filter-branch --force --index-filter \
  'git rm --cached --ignore-unmatch secrets.env' \
  --prune-empty --tag-name-filter cat -- --all
```

### 11.4 History Rewrite Implications

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │            AFTER HISTORY REWRITE — CRITICAL STEPS                    │
  │                                                                      │
  │  1. Force push ALL branches and tags                                 │
  │     git push --force --all && git push --force --tags                │
  │                                                                      │
  │  2. ALL team members must re-clone or:                               │
  │     git fetch --all                                                  │
  │     git reset --hard origin/main                                     │
  │     (for each branch they had checked out)                           │
  │                                                                      │
  │  3. ALL forks must be updated or deleted                             │
  │     → Forks retain the OLD history with the secret                   │
  │     → You cannot force-update someone else's fork                    │
  │                                                                      │
  │  4. GitHub/GitLab cache:                                             │
  │     → Cached views, PR diffs may still show the secret               │
  │     → Contact support to purge server-side cache                     │
  │     → GitHub: support ticket → server-side gc                        │
  │                                                                      │
  │  5. CI/CD runner caches:                                             │
  │     → Clear all build caches that may contain the old repo           │
  │     → Docker layer caches, npm caches, etc.                          │
  │                                                                      │
  │  6. ALWAYS rotate the credential                                     │
  │     → Removing from history is NOT enough                            │
  │     → Assume the secret is compromised the moment it was pushed      │
  └──────────────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha:** GitHub's "delete commit" or "edit file" UI does NOT remove the blob from history. The old commit is still accessible via its SHA hash directly (`github.com/org/repo/commit/<hash>`). You MUST do a full history rewrite + contact GitHub support for server-side gc.

### 11.5 GPG Signing Commits

Signed commits prove that the commit was actually made by the claimed author — prevents impersonation.

```bash
# Generate a GPG key
gpg --full-generate-key               # Choose RSA, 4096 bits

# List keys
gpg --list-secret-keys --keyid-format=long

# Configure Git to sign
git config --global user.signingkey <KEY-ID>
git config --global commit.gpgsign true         # Auto-sign all commits
git config --global tag.gpgsign true            # Auto-sign all tags

# Sign a specific commit
git commit -S -m "feat: signed commit"

# Sign a tag
git tag -s v1.0.0 -m "Signed release"

# Verify
git log --show-signature                        # Show GPG status in log
git verify-commit HEAD                          # Verify specific commit
git verify-tag v1.0.0                           # Verify specific tag
```

```
  ┌──────────────────────────────────────────────────────────────┐
  │           GPG SIGNING FLOW                                   │
  │                                                              │
  │  Developer                        Verifier                   │
  │  ┌──────────────┐                ┌──────────────┐            │
  │  │ Private Key  │                │ Public Key   │            │
  │  │ (kept secret)│                │ (on GitHub/  │            │
  │  └──────┬───────┘                │  keyserver)  │            │
  │         │                        └──────┬───────┘            │
  │         │ Sign                          │ Verify             │
  │         ▼                               ▼                    │
  │  ┌──────────────┐                ┌──────────────┐            │
  │  │ git commit   │ ──── push ───► │ git verify-  │            │
  │  │ -S           │                │ commit       │            │
  │  │ (signature   │                │ → "Good      │            │
  │  │  embedded)   │                │    signature" │            │
  │  └──────────────┘                └──────────────┘            │
  │                                                              │
  │  GitHub shows "Verified" ✅ badge on signed commits          │
  │  Unsigned commits show no badge (or "Unverified" ❌)         │
  │                                                              │
  │  Branch protection can REQUIRE signed commits:               │
  │  Settings → Branches → "Require signed commits"              │
  └──────────────────────────────────────────────────────────────┘
```

> **DevOps Relevance:** In supply-chain security (SLSA framework, Sigstore), signed commits are a building block. Requiring signed commits ensures no one can push malicious code by spoofing `user.email` in `.gitconfig`. Enterprise compliance (SOC2, FedRAMP) increasingly requires commit signing.

### 11.6 Secret Management — The Right Way

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         WHERE SECRETS SHOULD LIVE (NOT in Git)                       │
  │                                                                      │
  │  ┌──────────────────────────────────────────────────────────────┐    │
  │  │  Dedicated Secret Managers:                                  │    │
  │  │  • HashiCorp Vault         — industry standard               │    │
  │  │  • AWS Secrets Manager     — AWS native                      │    │
  │  │  • Azure Key Vault         — Azure native                    │    │
  │  │  • GCP Secret Manager      — GCP native                     │    │
  │  │  • Doppler, Infisical      — developer-friendly SaaS         │    │
  │  └──────────────────────────────────────────────────────────────┘    │
  │                                                                      │
  │  ┌──────────────────────────────────────────────────────────────┐    │
  │  │  CI/CD Platform Secrets:                                     │    │
  │  │  • GitHub Actions → Repository / Environment Secrets         │    │
  │  │  • GitLab CI → CI/CD Variables (masked, protected)           │    │
  │  │  • Jenkins → Credentials store                               │    │
  │  └──────────────────────────────────────────────────────────────┘    │
  │                                                                      │
  │  ┌──────────────────────────────────────────────────────────────┐    │
  │  │  Encrypted in Git (acceptable for some use cases):           │    │
  │  │  • SOPS (Mozilla)         — encrypt specific YAML values     │    │
  │  │  • Ansible Vault          — encrypt playbook vars            │    │
  │  │  • git-crypt              — transparent file encryption      │    │
  │  │  • Sealed Secrets (K8s)   — encrypt for cluster only         │    │
  │  └──────────────────────────────────────────────────────────────┘    │
  │                                                                      │
  │  ❌ NEVER acceptable:                                                │
  │  • Plaintext in .env committed to repo                               │
  │  • Hardcoded in source code                                          │
  │  • In Terraform .tfstate committed to repo                           │
  │  • In Docker build args without multi-stage                          │
  └──────────────────────────────────────────────────────────────────────┘
```

### 11.7 Security Checklist for DevOps Teams

| Control | Tool/Method | Priority |
|---------|------------|----------|
| `.gitignore` with secret patterns | Manual + templates | 🔴 Critical |
| Pre-commit secret scanning | gitleaks, detect-secrets | 🔴 Critical |
| CI pipeline secret scanning | gitleaks in CI, GitHub Advanced Security | 🔴 Critical |
| GPG commit signing | `commit.gpgsign = true` | 🟡 High |
| Branch protection (require reviews) | GitHub/GitLab settings | 🔴 Critical |
| Secret rotation policy | Vault, AWS Secrets Manager | 🟡 High |
| History scanning (periodic) | truffleHog, gitleaks `--all` | 🟡 High |
| Incident response runbook | Document: detect → rotate → purge → notify | 🔴 Critical |

> **⚠️ Gotcha (Terraform):** `terraform.tfstate` contains every secret your infrastructure uses — database passwords, API keys, private IPs. NEVER commit tfstate to Git. Use remote backends (S3 + DynamoDB, Terraform Cloud, Azure Blob) with encryption at rest.

---

## 12. Submodules, Subtrees & Monorepos

### 12.1 Why This Matters for DevOps

Real-world infrastructure repos rarely exist in isolation. You'll encounter:
- A Terraform modules repo consumed by 10 infrastructure repos
- A shared Helm chart library used across microservice repos
- A vendor SDK pinned to a specific version inside your project
- A monorepo containing services, libraries, and infrastructure together

Getting submodule/subtree management wrong causes **production outages** — builds that silently use stale dependencies, CI pipelines that clone the wrong version, and engineers who can't reproduce issues locally.

### 12.2 Git Submodules

A submodule is a **Git repo embedded inside another Git repo** at a specific commit. The parent repo tracks a pointer (commit hash) to the child repo — not the child's files.

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │                   GIT SUBMODULE MODEL                                │
  │                                                                      │
  │  Parent Repo (my-infra/)                                             │
  │  ┌──────────────────────────────────────────────┐                    │
  │  │  .gitmodules          ← Config: URL + path   │                    │
  │  │  terraform/                                   │                    │
  │  │  k8s-manifests/                               │                    │
  │  │  modules/shared-vpc/  ← SUBMODULE             │                    │
  │  │    │                                          │                    │
  │  │    └─► Points to commit abc1234               │                    │
  │  │        in repo: github.com/org/shared-vpc     │                    │
  │  │                                               │                    │
  │  │  The parent stores:                           │                    │
  │  │  • Path: modules/shared-vpc                   │                    │
  │  │  • URL: git@github.com:org/shared-vpc.git     │                    │
  │  │  • Commit: abc1234 (pinned version)           │                    │
  │  │                                               │                    │
  │  │  The parent does NOT store the submodule's     │                    │
  │  │  actual files — only a pointer.               │                    │
  │  └──────────────────────────────────────────────┘                    │
  │                                                                      │
  │  .gitmodules file:                                                   │
  │  ┌────────────────────────────────────────────┐                      │
  │  │  [submodule "modules/shared-vpc"]          │                      │
  │  │    path = modules/shared-vpc               │                      │
  │  │    url = git@github.com:org/shared-vpc.git │                      │
  │  └────────────────────────────────────────────┘                      │
  └──────────────────────────────────────────────────────────────────────┘
```

#### Submodule Commands

```bash
# Add a submodule
git submodule add https://github.com/org/shared-vpc.git modules/shared-vpc

# Clone a repo WITH submodules
git clone --recurse-submodules https://github.com/org/my-infra.git
# OR after a regular clone:
git submodule init                     # Register submodules from .gitmodules
git submodule update                   # Checkout the pinned commits
# OR combined:
git submodule update --init --recursive

# Update submodule to latest commit on its remote branch
cd modules/shared-vpc
git checkout main
git pull
cd ../..
git add modules/shared-vpc            # Stage the new commit pointer
git commit -m "chore: bump shared-vpc to latest"

# Update ALL submodules to latest
git submodule update --remote --merge

# Check submodule status
git submodule status                   # Shows commit hash + path + status
git submodule foreach 'git status'     # Run command in each submodule

# Remove a submodule (multi-step!)
git submodule deinit -f modules/shared-vpc
rm -rf .git/modules/modules/shared-vpc
git rm -f modules/shared-vpc
git commit -m "chore: remove shared-vpc submodule"
```

#### Submodule Architecture — How It Connects

```
  ┌────────────────────────────────────────────────────────────────┐
  │           SUBMODULE COMMIT RELATIONSHIP                        │
  │                                                                │
  │  Parent repo commits:                                          │
  │  P1 ◄── P2 ◄── P3 ◄── P4                                     │
  │              │           │                                     │
  │              │ points to │ points to                            │
  │              ▼           ▼                                     │
  │  Submodule commits:                                            │
  │  S1 ◄── S2 ◄── S3 ◄── S4 ◄── S5                               │
  │          ▲           ▲                                         │
  │          │           │                                         │
  │       P2 pins     P4 pins                                      │
  │       this         this                                        │
  │                                                                │
  │  P3 still points to S2 — submodule didn't change.             │
  │  The parent "pins" the submodule at a specific commit.         │
  │  This is VERSION PINNING — like a lockfile for repos.          │
  └────────────────────────────────────────────────────────────────┘
```

### 12.3 Submodule Gotchas — The Pain Points

| Gotcha | Problem | Solution |
|--------|---------|----------|
| **Detached HEAD inside submodule** | After `git submodule update`, you're in detached HEAD (pinned commit, no branch) | `cd submodule && git checkout main` before making changes |
| **Forgot `--recurse-submodules`** | Clone/pull without it → empty submodule directories | Run `git submodule update --init --recursive` |
| **Stale submodule pointer** | Parent still points to old commit even after submodule has new commits | Must explicitly `cd submodule && git pull`, then stage + commit in parent |
| **CI/CD fails on submodule** | Pipeline clone doesn't fetch submodules | Add `--recurse-submodules` to clone, or `git submodule update --init --recursive` step |
| **Recursive submodules** | Submodule contains its own submodules | Always use `--recursive` flag |
| **Removing a submodule is 4 steps** | `deinit`, `rm .git/modules/`, `git rm`, commit | No single command — document in your runbook |

> **⚠️ Gotcha (CI/CD):** GitHub Actions, GitLab CI, and Jenkins all handle submodules differently:
> - **GitHub Actions:** `actions/checkout` needs `submodules: recursive`
> - **GitLab CI:** Set `GIT_SUBMODULE_STRATEGY: recursive` in `.gitlab-ci.yml`
> - **Jenkins:** Check "Advanced clone behaviors" → "Recursively update submodules"
>
> Missing this = your build uses empty directories = cryptic failures.

### 12.4 Git Subtrees — The Alternative

A subtree merges another repo's files **directly into your repo** — no special pointer, no `.gitmodules`, no detached HEAD. The files are regular files in your repo.

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │               SUBMODULE vs SUBTREE — MENTAL MODEL                    │
  │                                                                      │
  │  Submodule:                            Subtree:                      │
  │  ┌──────────────────────┐              ┌──────────────────────┐      │
  │  │  Parent repo         │              │  Parent repo         │      │
  │  │  ┌────────────────┐  │              │  ┌────────────────┐  │      │
  │  │  │ submodule/     │  │              │  │ vendor/lib/    │  │      │
  │  │  │ (empty pointer │  │              │  │ (REAL files    │  │      │
  │  │  │  to external   │  │              │  │  merged in,    │  │      │
  │  │  │  repo commit)  │  │              │  │  part of THIS  │  │      │
  │  │  └────────────────┘  │              │  │  repo)         │  │      │
  │  │                      │              │  └────────────────┘  │      │
  │  │  Requires separate   │              │  No extra setup      │      │
  │  │  clone/init/update   │              │  for consumers       │      │
  │  └──────────────────────┘              └──────────────────────┘      │
  │                                                                      │
  │  Submodule: pointer → external repo                                  │
  │  Subtree:   files copied INTO your repo                              │
  └──────────────────────────────────────────────────────────────────────┘
```

#### Subtree Commands

```bash
# Add a subtree (pull external repo into a directory)
git subtree add --prefix=vendor/shared-lib \
  https://github.com/org/shared-lib.git main --squash

# Update subtree from upstream
git subtree pull --prefix=vendor/shared-lib \
  https://github.com/org/shared-lib.git main --squash

# Push changes BACK to the upstream repo (contributing back)
git subtree push --prefix=vendor/shared-lib \
  https://github.com/org/shared-lib.git main

# Split subtree into its own branch (for extraction)
git subtree split --prefix=vendor/shared-lib -b shared-lib-branch
```

> **Note:** `--squash` collapses the subtree's history into a single commit in your repo. Without it, you get the full history of the external repo merged into yours.

### 12.5 Submodule vs Subtree — Comparison

| Aspect | Submodule | Subtree |
|--------|-----------|---------|
| **Storage** | Pointer to external commit | Files merged into repo |
| **Clone complexity** | Need `--recurse-submodules` | Normal `git clone` works |
| **CI/CD friendliness** | Requires extra config | Works out of the box ✅ |
| **Version pinning** | Explicit (commit hash) | Implicit (merged state) |
| **Updating from upstream** | `cd submodule && git pull` + commit in parent | `git subtree pull` |
| **Contributing back** | Direct push from submodule dir | `git subtree push` |
| **Detached HEAD issue** | Yes — constant annoyance | No — normal files |
| **Repo size** | Small (just pointers) | Larger (files duplicated) |
| **History clarity** | Clean separation | Mixed history (unless `--squash`) |
| **Best for** | Large shared libraries with independent release cycles | Vendored dependencies, small shared code |

> **DevOps Pro Tip:** If your team constantly forgets `--recurse-submodules` and CI breaks, consider switching to subtrees. The simplicity wins. For large, independently versioned libraries (like a shared Terraform module repo), submodules make more sense because you want explicit version pinning.

### 12.6 Monorepo Considerations

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │            MONOREPO vs POLYREPO — DECISION FRAMEWORK                 │
  │                                                                      │
  │  Monorepo (single repo):                Polyrepo (many repos):       │
  │  ┌────────────────────────┐             ┌────────────────────────┐   │
  │  │ + Atomic cross-service │             │ + Independent releases │   │
  │  │   changes              │             │ + Isolated permissions │   │
  │  │ + Shared tooling &     │             │ + Smaller clone size   │   │
  │  │   CI config            │             │ + Team autonomy        │   │
  │  │ + Easier refactoring   │             │                        │   │
  │  │ + Single source of     │             │ - Diamond dependency   │   │
  │  │   truth                │             │   problem              │   │
  │  │                        │             │ - Cross-repo changes   │   │
  │  │ - Needs build tooling  │             │   are painful          │   │
  │  │   (Nx, Bazel, Turbo)   │             │ - Version matrix hell  │   │
  │  │ - Large clone size     │             │ - CI config duplication│   │
  │  │ - CI must be smart     │             │                        │   │
  │  │   (path-based triggers)│             │                        │   │
  │  └────────────────────────┘             └────────────────────────┘   │
  │                                                                      │
  │  When to use monorepo:                                               │
  │  • Tightly coupled services with shared libraries                    │
  │  • Platform teams managing infra + apps together                     │
  │  • Strong CI/CD with build caching (Nx, Turborepo, Bazel)           │
  │                                                                      │
  │  When to use polyrepo:                                               │
  │  • Independent teams with different release cycles                   │
  │  • Open-source projects with external contributors                   │
  │  • Strict access control per service                                 │
  └──────────────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha:** Many production outages happen because someone updated a submodule pointer in a monorepo without testing the downstream services. In monorepos, use tools like Nx's `affected` command or Bazel's dependency graph to automatically test everything that depends on a changed library.

---

## 13. Hooks & Automation — DevOps Territory

### 13.1 What Are Git Hooks?

Git hooks are **scripts that run automatically** at specific points in the Git workflow. They live in `.git/hooks/` and are triggered by Git events (commit, push, merge, etc.).

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │                    GIT HOOKS LIFECYCLE                                │
  │                                                                      │
  │  Developer workflow:                                                 │
  │                                                                      │
  │  Edit files                                                          │
  │      │                                                               │
  │      ▼                                                               │
  │  git add .                                                           │
  │      │                                                               │
  │      ▼                                                               │
  │  git commit ─────────────┐                                           │
  │      │                   │                                           │
  │      │         ┌─────────▼──────────┐                                │
  │      │         │   pre-commit       │  ← Lint, format, scan secrets  │
  │      │         │   (can BLOCK)      │                                │
  │      │         └─────────┬──────────┘                                │
  │      │                   │ pass                                      │
  │      │         ┌─────────▼──────────┐                                │
  │      │         │   prepare-commit-  │  ← Auto-add ticket number     │
  │      │         │   msg              │                                │
  │      │         └─────────┬──────────┘                                │
  │      │                   │                                           │
  │      │         ┌─────────▼──────────┐                                │
  │      │         │   commit-msg       │  ← Validate commit message    │
  │      │         │   (can BLOCK)      │    format (Conventional       │
  │      │         └─────────┬──────────┘    Commits)                    │
  │      │                   │ pass                                      │
  │      │         ┌─────────▼──────────┐                                │
  │      │         │   post-commit      │  ← Notify, log (can't block)  │
  │      │         └─────────┬──────────┘                                │
  │      │                   │                                           │
  │      ▼                   ▼                                           │
  │  git push ───────────────┐                                           │
  │                ┌─────────▼──────────┐                                │
  │                │   pre-push         │  ← Run tests, check CI status  │
  │                │   (can BLOCK)      │                                │
  │                └─────────┬──────────┘                                │
  │                          │ pass                                      │
  │                          ▼                                           │
  │                   Push to remote                                     │
  │                          │                                           │
  │                ┌─────────▼──────────┐                                │
  │                │   post-receive     │  ← (Server-side) Trigger       │
  │                │   (server hook)    │    CI/CD pipeline              │
  │                └────────────────────┘                                │
  │                                                                      │
  └──────────────────────────────────────────────────────────────────────┘
```

### 13.2 Hook Types — Client-Side vs Server-Side

| Hook | Side | Trigger | Can Block? | Common Use |
|------|------|---------|------------|-----------|
| **pre-commit** | Client | Before commit is created | ✅ Yes | Lint, format, secret scan |
| **prepare-commit-msg** | Client | After default msg generated, before editor opens | ✅ Yes | Auto-insert ticket number, template |
| **commit-msg** | Client | After user writes message | ✅ Yes | Validate Conventional Commits format |
| **post-commit** | Client | After commit is created | ❌ No | Notifications, logging |
| **pre-push** | Client | Before push to remote | ✅ Yes | Run tests, check branch name |
| **post-merge** | Client | After a merge completes | ❌ No | Run `npm install`, update deps |
| **pre-rebase** | Client | Before rebase starts | ✅ Yes | Prevent rebase of protected branches |
| **post-checkout** | Client | After checkout/switch | ❌ No | Auto-setup environment |
| **pre-receive** | Server | Before refs are updated | ✅ Yes | Enforce policies (signed commits, branch naming) |
| **update** | Server | Per-ref update | ✅ Yes | Per-branch policy enforcement |
| **post-receive** | Server | After refs are updated | ❌ No | Trigger CI/CD, send notifications |

### 13.3 Writing Hooks — Practical Examples

#### Example 1: pre-commit — Lint & Secret Scan

```bash
#!/bin/bash
# .git/hooks/pre-commit (must be chmod +x)

set -e

echo "🔍 Running pre-commit checks..."

# 1. Check for secrets
if command -v gitleaks &> /dev/null; then
    gitleaks protect --staged --verbose
    echo "✅ No secrets detected"
fi

# 2. Lint staged files
STAGED_GO=$(git diff --cached --name-only --diff-filter=ACM | grep '\.go$' || true)
if [ -n "$STAGED_GO" ]; then
    echo "🔧 Running gofmt..."
    UNFORMATTED=$(gofmt -l $STAGED_GO)
    if [ -n "$UNFORMATTED" ]; then
        echo "❌ Unformatted Go files:"
        echo "$UNFORMATTED"
        exit 1                        # Block the commit
    fi
fi

# 3. Check for large files
LARGE_FILES=$(git diff --cached --name-only --diff-filter=ACM | while read f; do
    size=$(wc -c < "$f" 2>/dev/null || echo 0)
    if [ "$size" -gt 5242880 ]; then  # 5MB
        echo "$f ($size bytes)"
    fi
done)
if [ -n "$LARGE_FILES" ]; then
    echo "❌ Files too large (>5MB). Use Git LFS:"
    echo "$LARGE_FILES"
    exit 1
fi

echo "✅ All pre-commit checks passed"
```

#### Example 2: commit-msg — Enforce Conventional Commits

```bash
#!/bin/bash
# .git/hooks/commit-msg

COMMIT_MSG_FILE=$1
COMMIT_MSG=$(cat "$COMMIT_MSG_FILE")

# Conventional Commits pattern: type(scope): description
PATTERN="^(feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert)(\([a-z0-9-]+\))?: .{1,72}$"

if ! echo "$COMMIT_MSG" | head -1 | grep -qE "$PATTERN"; then
    echo "❌ Commit message does not follow Conventional Commits format."
    echo ""
    echo "Expected: type(scope): description"
    echo "Examples:"
    echo "  feat(auth): add OAuth2 login"
    echo "  fix(api): resolve null pointer in handler"
    echo "  docs: update README with deploy steps"
    echo "  ci(pipeline): add staging deploy stage"
    echo ""
    echo "Types: feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert"
    exit 1
fi
```

#### Example 3: pre-push — Prevent Push to Protected Branches

```bash
#!/bin/bash
# .git/hooks/pre-push

PROTECTED_BRANCHES="^(main|master|production|release/.*)$"

while read local_ref local_sha remote_ref remote_sha; do
    remote_branch=$(echo "$remote_ref" | sed 's|refs/heads/||')

    if echo "$remote_branch" | grep -qE "$PROTECTED_BRANCHES"; then
        echo "❌ Direct push to '$remote_branch' is blocked."
        echo "   Use a pull request instead."
        exit 1
    fi
done

exit 0
```

### 13.4 Sharing Hooks — The pre-commit Framework

Hooks live in `.git/hooks/` which is **NOT committed** to the repo. To share hooks across a team, use the **pre-commit framework** (or Husky for Node.js).

```bash
# Install pre-commit (Python-based)
pip install pre-commit

# Create .pre-commit-config.yaml in repo root
cat > .pre-commit-config.yaml << 'EOF'
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: check-added-large-files
        args: ['--maxkb=5000']
      - id: detect-private-key

  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks

  - repo: https://github.com/compilerla/conventional-pre-commit
    rev: v3.0.0
    hooks:
      - id: conventional-pre-commit
        stages: [commit-msg]
EOF

# Install hooks into .git/hooks/
pre-commit install                     # pre-commit hook
pre-commit install --hook-type commit-msg   # commit-msg hook

# Run on all files (first time)
pre-commit run --all-files

# Auto-update hook versions
pre-commit autoupdate
```

```
  ┌──────────────────────────────────────────────────────────────┐
  │          pre-commit FRAMEWORK FLOW                           │
  │                                                              │
  │  .pre-commit-config.yaml (committed to repo)                 │
  │         │                                                    │
  │         │ pre-commit install                                 │
  │         ▼                                                    │
  │  .git/hooks/pre-commit (auto-generated, NOT committed)       │
  │         │                                                    │
  │         │ git commit                                         │
  │         ▼                                                    │
  │  pre-commit framework runs each hook:                        │
  │  ┌────────────────────────────────────────┐                  │
  │  │ 1. trailing-whitespace    → ✅ Passed   │                  │
  │  │ 2. check-yaml             → ✅ Passed   │                  │
  │  │ 3. detect-private-key     → ✅ Passed   │                  │
  │  │ 4. gitleaks               → ❌ FAILED   │                  │
  │  │    "AWS key found in config.py:23"      │                  │
  │  └────────────────────────────────────────┘                  │
  │         │                                                    │
  │         ▼                                                    │
  │  ❌ Commit BLOCKED — developer must fix                      │
  │                                                              │
  │  New team member clones repo:                                │
  │  git clone ... && cd repo && pre-commit install              │
  │  → Hooks are active. Enforced for everyone.                  │
  └──────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha:** `pre-commit install` must be run by each developer after cloning. To enforce this, add it to your Makefile, npm postinstall script, or onboarding docs. Some teams add a CI check that verifies the `.pre-commit-config.yaml` hooks pass.

### 13.5 Git + CI/CD Pipeline Integration

Git events are the **trigger mechanism** for CI/CD pipelines. Understanding how Git integrates with CI is core DevOps knowledge.

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │            GIT → CI/CD TRIGGER ARCHITECTURE                          │
  │                                                                      │
  │  Developer                                                           │
  │     │                                                                │
  │     │ git push / PR / tag                                            │
  │     ▼                                                                │
  │  ┌──────────────────┐                                                │
  │  │  Git Server       │                                                │
  │  │  (GitHub/GitLab)  │                                                │
  │  └────────┬─────────┘                                                │
  │           │                                                          │
  │           │ Webhook (HTTP POST to CI server)                         │
  │           │ Contains: repo, branch, commit SHA,                      │
  │           │   event type, author, changed files                      │
  │           ▼                                                          │
  │  ┌──────────────────────────────────────┐                            │
  │  │         CI/CD Platform               │                            │
  │  │  (Jenkins / GitHub Actions /         │                            │
  │  │   GitLab CI / ArgoCD)                │                            │
  │  └────────┬─────────────────────────────┘                            │
  │           │                                                          │
  │           │ Evaluates trigger rules:                                 │
  │           │ ┌──────────────────────────────────┐                     │
  │           │ │ • Branch pattern: main, release/* │                     │
  │           │ │ • Path filter: src/**, infra/**   │                     │
  │           │ │ • Event type: push, PR, tag       │                     │
  │           │ │ • Tag pattern: v*                  │                     │
  │           │ └──────────────────────────────────┘                     │
  │           │                                                          │
  │           ▼                                                          │
  │  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐   │
  │  │  Build           │  │  Test            │  │  Deploy          │   │
  │  │  • git checkout  │─►│  • Unit tests    │─►│  • Staging       │   │
  │  │  • compile       │  │  • Integration   │  │  • Production    │   │
  │  │  • docker build  │  │  • Security scan │  │  • Rollback      │   │
  │  └──────────────────┘  └──────────────────┘  └──────────────────┘   │
  └──────────────────────────────────────────────────────────────────────┘
```

### 13.6 CI Platform Trigger Examples

#### GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI Pipeline
on:
  push:
    branches: [main, 'release/**']     # Branch pattern
    paths: ['src/**', 'Dockerfile']     # Path filter
    tags: ['v*']                        # Tag pattern
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0                # Full history (for git describe, blame)
          submodules: recursive         # Clone submodules too
      - run: make build
      - run: make test
```

#### GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - deploy

build:
  stage: build
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/'      # Semantic version tags
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
  variables:
    GIT_SUBMODULE_STRATEGY: recursive    # Handle submodules
  script:
    - make build

deploy-prod:
  stage: deploy
  rules:
    - if: '$CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/'
      when: manual                       # Require manual approval for prod
  environment: production
  script:
    - make deploy
```

#### Jenkins (Jenkinsfile)

```groovy
// Jenkinsfile
pipeline {
    agent any
    triggers {
        // Webhook trigger (configured in GitHub/GitLab)
        githubPush()
    }
    stages {
        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    extensions: [
                        [$class: 'SubmoduleOption', recursiveSubmodules: true],
                        [$class: 'CloneOption', depth: 0, shallow: false]
                    ]
                ])
            }
        }
        stage('Build') {
            steps { sh 'make build' }
        }
        stage('Deploy') {
            when {
                tag pattern: "v\\d+\\.\\d+\\.\\d+", comparator: "REGEXP"
            }
            steps { sh 'make deploy' }
        }
    }
}
```

### 13.7 Webhook Architecture

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │                    WEBHOOK DEEP DIVE                                  │
  │                                                                      │
  │  What is a webhook?                                                  │
  │  An HTTP POST request sent by the Git server to a configured URL     │
  │  whenever a specific event occurs.                                   │
  │                                                                      │
  │  ┌──────────┐   git push   ┌──────────┐  HTTP POST  ┌────────────┐  │
  │  │Developer │─────────────►│ GitHub   │────────────►│ CI Server  │  │
  │  │          │              │          │             │ (Jenkins/  │  │
  │  │          │              │ Events:  │  Payload:   │  ArgoCD)   │  │
  │  └──────────┘              │ • push   │  {          │            │  │
  │                            │ • PR     │   ref,      │ Parses     │  │
  │                            │ • tag    │   commits,  │ payload →  │  │
  │                            │ • release│   repo,     │ triggers   │  │
  │                            │ • fork   │   sender    │ pipeline   │  │
  │                            └──────────┘  }          └────────────┘  │
  │                                                                      │
  │  Security:                                                           │
  │  • Webhooks use a shared secret for HMAC signature verification      │
  │  • CI server validates X-Hub-Signature-256 header                    │
  │  • Always use HTTPS endpoints                                        │
  │  • Restrict webhook source IPs (GitHub publishes their IP ranges)    │
  └──────────────────────────────────────────────────────────────────────┘
```

### 13.8 Git + CI/CD — Gotchas & Best Practices

| Gotcha | Problem | Solution |
|--------|---------|----------|
| **Shallow clone in CI** | `git clone --depth 1` breaks `git describe`, `git log`, `git blame` | Use `fetch-depth: 0` or fetch enough history |
| **Detached HEAD in CI** | CI checks out a commit, not a branch → `git branch --show-current` is empty | Use `$GITHUB_SHA`, `$CI_COMMIT_SHA` env vars |
| **Submodule not cloned** | Empty dirs, build fails with missing files | Add `submodules: recursive` / `GIT_SUBMODULE_STRATEGY` |
| **Tag trigger but no checkout of tag** | Pipeline runs but code isn't at the tag | Explicitly `git checkout $TAG_NAME` |
| **Webhook secret leaked** | Attacker can trigger pipelines by sending fake POST | Store webhook secret in CI secrets, validate HMAC |
| **Concurrent pushes** | Two pushes trigger two pipelines → race condition on deploy | Use pipeline concurrency controls / locking |
| **Large repo slow clone** | CI spends minutes cloning before build starts | Use `--filter=blob:none` (partial clone) or Git LFS |

> **DevOps Pro Tip:** In GitOps (ArgoCD, Flux), the Git repo IS the deployment manifest. ArgoCD watches for changes to a branch/path and automatically syncs the cluster state. The webhook triggers ArgoCD sync — no pipeline needed for deployment. Understand this pattern: Git → Webhook → ArgoCD → K8s.

---

## 14. Performance & Large Repos

### 14.1 Why This Matters

Enterprise repos grow large — years of history, binary assets, thousands of contributors. Without performance tuning:
- `git clone` takes 20+ minutes in CI → slow pipelines → wasted compute cost
- `git status` lags on repos with 100K+ files → developer frustration
- Storage costs balloon when binaries are committed (Docker images, ML models, videos)
- Garbage collection pauses can stall Git operations on large servers

### 14.2 Shallow Clone

A shallow clone fetches only the last N commits of history — not the full DAG. Dramatically faster for CI.

```
  Full Clone:                         Shallow Clone (--depth 1):
  ┌──────────────────────────┐        ┌──────────────────────────┐
  │  A ◄── B ◄── C ◄── D    │        │                    D     │
  │  │                       │        │                    ▲     │
  │  All 4 commits + all     │        │                    │     │
  │  blobs for every version │        │               (grafted)  │
  │  of every file           │        │                          │
  │                          │        │  Only commit D + its     │
  │  Size: could be GBs      │        │  tree of blobs           │
  │  Time: minutes            │        │                          │
  │                          │        │  Size: much smaller       │
  │                          │        │  Time: seconds            │
  └──────────────────────────┘        └──────────────────────────┘
```

```bash
# Shallow clone — only latest commit
git clone --depth 1 https://github.com/org/repo.git

# Shallow clone — last 10 commits
git clone --depth 10 https://github.com/org/repo.git

# Shallow clone of a specific branch
git clone --depth 1 --branch release/v2.0 https://github.com/org/repo.git

# "Unshallow" later if you need full history
git fetch --unshallow

# Fetch only enough history to include a specific commit
git fetch --shallow-since="2025-01-01"
git fetch --shallow-exclude=old-branch
```

> **⚠️ Gotcha:** Shallow clones break many Git operations:
> - `git log` — incomplete history
> - `git blame` — can't trace beyond shallow depth
> - `git describe` — can't find tags beyond shallow depth
> - `git merge-base` — may fail to find common ancestor
> - `git submodule update` — may fail for submodules pinned to old commits
>
> **Rule:** Use shallow clones in CI for build speed, but use full clones for operations that need history (semantic versioning, changelogs, bisect).

### 14.3 Partial Clone (Blobless / Treeless)

Introduced in Git 2.19+ — a more surgical approach than shallow clones. Fetches all commits and trees but delays downloading blob (file) content until needed.

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │              CLONE STRATEGIES COMPARED                               │
  │                                                                      │
  │  Full clone:                                                         │
  │  ┌──────────────────────────────────────────────┐                    │
  │  │  All commits + all trees + all blobs          │                    │
  │  │  = complete history + all file versions        │                    │
  │  │  Size: 100% | Speed: slowest                   │                    │
  │  └──────────────────────────────────────────────┘                    │
  │                                                                      │
  │  Blobless clone (--filter=blob:none):                                │
  │  ┌──────────────────────────────────────────────┐                    │
  │  │  All commits + all trees + NO blobs           │                    │
  │  │  Blobs fetched on-demand (git checkout, diff)  │                    │
  │  │  Size: ~30-60% | Speed: much faster            │                    │
  │  │  ✅ git log works  ✅ git blame works            │                    │
  │  │  ✅ git merge-base works  ✅ git describe works  │                    │
  │  └──────────────────────────────────────────────┘                    │
  │                                                                      │
  │  Treeless clone (--filter=tree:0):                                   │
  │  ┌──────────────────────────────────────────────┐                    │
  │  │  All commits + NO trees + NO blobs            │                    │
  │  │  Trees + blobs fetched on-demand               │                    │
  │  │  Size: ~10-20% | Speed: fastest                │                    │
  │  │  ⚠️ Many operations trigger network fetches     │                    │
  │  └──────────────────────────────────────────────┘                    │
  │                                                                      │
  │  Shallow clone (--depth N):                                          │
  │  ┌──────────────────────────────────────────────┐                    │
  │  │  Last N commits + their trees + their blobs   │                    │
  │  │  NO history beyond depth N                     │                    │
  │  │  Size: smallest | Speed: fastest               │                    │
  │  │  ❌ git log incomplete  ❌ git blame limited     │                    │
  │  │  ❌ git describe may fail                       │                    │
  │  └──────────────────────────────────────────────┘                    │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# Blobless clone (recommended for CI)
git clone --filter=blob:none https://github.com/org/repo.git

# Treeless clone (fastest, but triggers many fetches during operations)
git clone --filter=tree:0 https://github.com/org/repo.git

# Combine with single-branch for maximum speed
git clone --filter=blob:none --single-branch --branch main https://github.com/org/repo.git
```

> **DevOps Pro Tip:** For CI pipelines that need `git describe` or `git log` (e.g., semantic-release, changelog generation), use `--filter=blob:none` instead of `--depth 1`. You get the full commit graph (so describe/log work) but skip downloading every version of every file.

### 14.4 Sparse Checkout

Clone the full repo but only check out a subset of files/directories into your working tree. Essential for monorepos.

```
  ┌──────────────────────────────────────────────────────────────┐
  │              SPARSE CHECKOUT — MONOREPO USE CASE             │
  │                                                              │
  │  Full monorepo:                   Sparse checkout:           │
  │  ┌──────────────────┐            ┌──────────────────┐        │
  │  │ services/        │            │ services/        │        │
  │  │   api-gateway/   │            │   api-gateway/ ✅│        │
  │  │   auth-service/  │            │                  │        │
  │  │   payment/       │            │ libs/            │        │
  │  │ libs/            │            │   common/ ✅     │        │
  │  │   common/        │            │                  │        │
  │  │   proto/         │            │ (everything else │        │
  │  │ infra/           │            │  NOT on disk)    │        │
  │  │   terraform/     │            └──────────────────┘        │
  │  │   k8s/           │                                        │
  │  └──────────────────┘            Only files you need!        │
  │                                                              │
  │  .git/ still has full history — just working tree is sparse  │
  └──────────────────────────────────────────────────────────────┘
```

```bash
# Enable sparse checkout (cone mode — directory-based, faster)
git clone --filter=blob:none --sparse https://github.com/org/monorepo.git
cd monorepo

# Add directories you want checked out
git sparse-checkout set services/api-gateway libs/common

# Add more directories later
git sparse-checkout add infra/terraform

# List what's checked out
git sparse-checkout list

# Disable sparse checkout (check out everything)
git sparse-checkout disable
```

> **⚠️ Gotcha:** Sparse checkout with `--filter=blob:none` is the optimal combo for monorepos. Without the filter, Git still downloads all blobs even if they aren't checked out. Combine both for minimum download + minimum disk usage.

### 14.5 Git LFS (Large File Storage)

Git LFS replaces large binary files with **text pointers** in the repo, storing the actual file content on a separate LFS server.

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │                   GIT LFS ARCHITECTURE                               │
  │                                                                      │
  │  Without LFS:                        With LFS:                       │
  │  ┌──────────────────────┐            ┌──────────────────────┐        │
  │  │  .git/objects/       │            │  .git/objects/       │        │
  │  │  ├── blob (code) 5KB │            │  ├── blob (code) 5KB │        │
  │  │  ├── blob (code) 3KB │            │  ├── pointer 130B    │        │
  │  │  ├── blob (model)    │            │  │   "oid sha256:abc"│        │
  │  │  │   500MB ❌         │            │  └── pointer 130B    │        │
  │  │  └── blob (video)    │            │      "oid sha256:def"│        │
  │  │      2GB ❌           │            └──────────┬───────────┘        │
  │  └──────────────────────┘                       │                    │
  │                                                 │ On demand          │
  │  Every clone downloads                          ▼                    │
  │  ALL versions of ALL                 ┌──────────────────────┐        │
  │  large files = HUGE                  │  LFS Server          │        │
  │                                      │  ├── model 500MB     │        │
  │                                      │  └── video 2GB       │        │
  │                                      │                      │        │
  │                                      │  Only downloads when │        │
  │                                      │  you checkout that   │        │
  │                                      │  file version        │        │
  │                                      └──────────────────────┘        │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# Install Git LFS
sudo apt install git-lfs          # Debian/Ubuntu
brew install git-lfs              # macOS

# Initialize LFS in a repo
git lfs install

# Track file patterns with LFS
git lfs track "*.psd"
git lfs track "*.zip"
git lfs track "*.tar.gz"
git lfs track "models/**"
git lfs track "*.pb"               # ML model files

# This creates/updates .gitattributes:
cat .gitattributes
# *.psd filter=lfs diff=lfs merge=lfs -text
# *.zip filter=lfs diff=lfs merge=lfs -text

# IMPORTANT: commit .gitattributes first!
git add .gitattributes
git commit -m "chore: configure LFS tracking"

# Then add and commit large files normally
git add model.pb
git commit -m "feat: add trained model"
git push                           # LFS files uploaded to LFS server

# Check what's tracked by LFS
git lfs ls-files                   # List LFS files in current checkout
git lfs status                     # Show LFS file status

# Migrate existing files to LFS (rewrite history)
git lfs migrate import --include="*.psd" --everything
```

> **⚠️ Gotcha (LFS + CI):** CI runners need `git lfs install` and `git lfs pull` to download LFS files. Without it, your build gets the pointer files (130-byte text) instead of the actual content. GitHub Actions: `lfs: true` in checkout action. GitLab CI: `GIT_LFS_SKIP_SMUDGE=1` for speed, then `git lfs pull` only for files you need.

> **⚠️ Gotcha (LFS Costs):** GitHub LFS has bandwidth and storage limits. Free tier: 1GB storage + 1GB/month bandwidth. Every clone/CI run that downloads LFS files counts against bandwidth. Enterprise repos with large ML models can hit limits fast — consider self-hosted LFS (e.g., on S3 with lfs-s3).

### 14.6 Packfiles & Garbage Collection

Git periodically packs loose objects into **packfiles** for efficiency (delta compression).

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │              LOOSE OBJECTS vs PACKFILES                               │
  │                                                                      │
  │  Loose objects (initial):             Packfiles (after gc):          │
  │  ┌──────────────────────┐            ┌──────────────────────┐        │
  │  │  .git/objects/       │            │  .git/objects/pack/  │        │
  │  │  ab/cdef...  (blob)  │   git gc   │  pack-abc123.pack   │        │
  │  │  cd/efgh...  (blob)  │ ────────►  │  pack-abc123.idx    │        │
  │  │  ef/ghij...  (tree)  │            │                      │        │
  │  │  gh/ijkl...  (commit)│            │  Multiple objects    │        │
  │  │                      │            │  packed together     │        │
  │  │  One file per object │            │  with delta          │        │
  │  │  Many small files    │            │  compression         │        │
  │  │  Slower lookups      │            │  Much fewer files    │        │
  │  └──────────────────────┘            │  Faster lookups      │        │
  │                                      └──────────────────────┘        │
  │                                                                      │
  │  Delta compression:                                                  │
  │  If file V2 differs from V1 by 10 lines,                            │
  │  packfile stores V2 as "V1 + delta" — huge space savings.           │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# Manual garbage collection
git gc                             # Pack loose objects, remove unreachable ones
git gc --aggressive                # More thorough (slower), better compression
git gc --auto                      # Only run if thresholds are met

# Prune unreachable objects
git prune                          # Remove objects not reachable from any ref
git prune --expire=now             # Remove ALL unreachable objects immediately

# Check repository integrity
git fsck                           # Full integrity check
git fsck --unreachable             # List unreachable objects
git fsck --lost-found              # Save dangling objects to .git/lost-found/

# Inspect packfiles
git count-objects -v               # Show object count and size stats
git verify-pack -v .git/objects/pack/pack-*.idx | head -20

# Repack for optimal performance
git repack -a -d --depth=250 --window=250    # Aggressive repack
```

### 14.7 Performance Tuning Cheat Sheet

| Problem | Solution | Command/Config |
|---------|----------|---------------|
| Slow clone | Shallow/partial clone | `--depth 1` or `--filter=blob:none` |
| Huge repo on disk | Git LFS for binaries | `git lfs track "*.bin"` |
| Slow `git status` | Enable filesystem monitor | `git config core.fsmonitor true` |
| Slow `git status` (many files) | Enable untracked cache | `git config core.untrackedCache true` |
| Monorepo — too many files | Sparse checkout | `git sparse-checkout set <dirs>` |
| Many loose objects | Garbage collection | `git gc --auto` |
| Slow network fetches | Commit graph | `git commit-graph write --reachable` |
| Large number of refs | Packed refs | `git pack-refs --all` |
| Slow `git log --graph` | Commit graph file | `git config fetch.writeCommitGraph true` |

> **DevOps Pro Tip:** For self-hosted Git servers (Gitea, GitLab), schedule `git gc` during off-peak hours. On GitHub/GitLab SaaS, this is handled automatically. For CI: set `GIT_CONFIG_COUNT`, `GIT_CONFIG_KEY_0=gc.auto`, `GIT_CONFIG_VALUE_0=0` to disable auto-gc in pipelines (it wastes CI time).

---

## 15. Edge Cases That Break Juniors

### 15.1 Stash — Deep Dive

`git stash` temporarily shelves uncommitted changes so you can switch context.

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │                    GIT STASH — MENTAL MODEL                          │
  │                                                                      │
  │  Working Dir          Stash Stack              Working Dir           │
  │  (dirty)              (LIFO — last in,         (clean)               │
  │                        first out)                                    │
  │  ┌──────────┐         ┌──────────────┐         ┌──────────┐         │
  │  │ Modified │  stash  │ stash@{0}    │  pop    │ Clean    │         │
  │  │ files    │────────►│ stash@{1}    │────────►│ state    │         │
  │  │          │  push   │ stash@{2}    │         │ + stashed│         │
  │  └──────────┘         └──────────────┘         │ changes  │         │
  │                                                └──────────┘         │
  │                                                                      │
  │  Stash stores:                                                       │
  │  • Staged changes (index)                                            │
  │  • Unstaged changes (working tree)                                   │
  │  • Optionally: untracked files (with -u)                             │
  │  • Optionally: ignored files too (with -a)                           │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# Basic stash
git stash                          # Stash tracked modified + staged files
git stash push -m "WIP: login page" # Stash with descriptive message

# Include untracked files
git stash -u                       # Stash including untracked files
git stash --include-untracked      # Same thing

# Include everything (even ignored files)
git stash -a                       # Stash ALL including .gitignore'd files

# Stash specific files (Git 2.13+)
git stash push -m "partial" -- src/auth.py tests/

# List stashes
git stash list                     # Show all stashes

# Inspect a stash
git stash show                     # Summary of stash@{0} changes
git stash show -p                  # Full diff of stash@{0}
git stash show stash@{2} -p       # Full diff of specific stash

# Apply stash
git stash pop                      # Apply stash@{0} AND remove from stack
git stash apply                    # Apply stash@{0} but KEEP in stack
git stash apply stash@{2}         # Apply specific stash

# Drop / clear stash
git stash drop stash@{1}          # Remove specific stash
git stash clear                    # Remove ALL stashes ⚠️
```

#### Stash Pop vs Apply — Critical Difference

```
  ┌──────────────────────────────────────────────────────────────┐
  │                                                              │
  │  git stash pop:                                              │
  │  • Applies the stash                                         │
  │  • If successful: REMOVES from stash stack                   │
  │  • If conflict: stash is NOT removed (still in stack)        │
  │    → Resolve conflict, then manually git stash drop          │
  │                                                              │
  │  git stash apply:                                            │
  │  • Applies the stash                                         │
  │  • KEEPS stash in stack regardless of success/conflict       │
  │  • Safer — you can apply the same stash to multiple branches │
  │                                                              │
  │  RECOMMENDATION: Use `apply` when unsure,                    │
  │  `pop` when you're confident there won't be conflicts.       │
  └──────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha (Stash with Conflicts):** When `git stash pop` encounters a conflict, the stash remains in the stack (not removed). After resolving the conflict and committing, you must manually run `git stash drop` to clean up. Many engineers forget this and accumulate stale stashes.

### 15.2 Cherry-pick — Surgical Commit Transplant

Cherry-pick copies a specific commit from one branch to another, creating a new commit with the same changes but a different hash.

```
  ┌──────────────────────────────────────────────────────────────┐
  │                 CHERRY-PICK — VISUAL                         │
  │                                                              │
  │  Before:                                                     │
  │  main:     A ◄── B ◄── C ◄── D                              │
  │  feature:  A ◄── B ◄── E ◄── F ◄── G                        │
  │                                                              │
  │  We want commit F on main (but not E or G):                  │
  │                                                              │
  │  $ git checkout main                                         │
  │  $ git cherry-pick <hash-of-F>                               │
  │                                                              │
  │  After:                                                      │
  │  main:     A ◄── B ◄── C ◄── D ◄── F'                       │
  │  feature:  A ◄── B ◄── E ◄── F ◄── G                        │
  │                                                              │
  │  F' has the SAME changes as F but a DIFFERENT hash           │
  │  (different parent → different SHA)                          │
  └──────────────────────────────────────────────────────────────┘
```

```bash
# Cherry-pick a single commit
git cherry-pick abc1234

# Cherry-pick without auto-committing (stage changes only)
git cherry-pick --no-commit abc1234

# Cherry-pick a range of commits (exclusive start, inclusive end)
git cherry-pick A..D                # Picks B, C, D (NOT A)
git cherry-pick A^..D               # Picks A, B, C, D (inclusive)

# Cherry-pick from another branch
git cherry-pick feature~3           # 3rd commit from tip of feature

# Handle conflicts
git cherry-pick abc1234
# ... conflict ...
# Fix conflicts, then:
git add .
git cherry-pick --continue
# OR abort:
git cherry-pick --abort
```

> **⚠️ Gotcha (Cherry-pick Duplicates):** If you cherry-pick commit F to main, then later merge the feature branch into main, commit F effectively appears twice (F on feature, F' on main). Git usually handles this gracefully, but it can cause confusion in `git log`. Use `git log --cherry-mark` to identify cherry-picked duplicates.

> **DevOps Relevance:** Cherry-pick is the standard **hotfix strategy**: pick the fix commit from develop/feature, apply it directly to the release/production branch. This avoids merging unfinished features.

### 15.3 Orphan Branches

An orphan branch has **no parent commit** — it starts with a completely empty history. Useful for GitHub Pages, documentation, or storing unrelated content in the same repo.

```bash
# Create an orphan branch
git checkout --orphan gh-pages

# Working dir still has files from previous branch — remove them
git rm -rf .

# Create new content
echo "<h1>Docs</h1>" > index.html
git add index.html
git commit -m "Initial GitHub Pages commit"

# Push
git push origin gh-pages
```

```
  ┌──────────────────────────────────────────────────────────────┐
  │                 ORPHAN BRANCH STRUCTURE                      │
  │                                                              │
  │  main:      A ◄── B ◄── C ◄── D                             │
  │                                                              │
  │  gh-pages:  X ◄── Y ◄── Z     (completely separate history) │
  │                                                              │
  │  No common ancestor between main and gh-pages.               │
  │  They exist in the same .git/ but are independent DAGs.      │
  └──────────────────────────────────────────────────────────────┘
```

### 15.4 Rewriting Author History

Sometimes you need to fix the author on commits (wrong email, company policy, etc.).

```bash
# Fix author on the LAST commit
git commit --amend --author="Jane Doe <jane@company.com>" --no-edit

# Fix author on multiple commits (interactive rebase)
git rebase -i HEAD~5
# Change 'pick' to 'edit' on each commit to fix
# For each paused commit:
git commit --amend --author="Jane Doe <jane@company.com>" --no-edit
git rebase --continue

# Fix ALL commits by a specific author (filter-repo)
git filter-repo --commit-callback '
    if commit.author_email == b"jane@personal.com":
        commit.author_email = b"jane@company.com"
        commit.author_name = b"Jane Doe"
        commit.committer_email = b"jane@company.com"
        commit.committer_name = b"Jane Doe"
'
```

> **⚠️ Gotcha:** Changing author rewrites commit hashes (author is part of the commit object). This requires force push and team re-clone — same implications as any history rewrite.

### 15.5 CRLF vs LF — Line Ending Issues

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │              LINE ENDING CHAOS                                       │
  │                                                                      │
  │  Windows:  Lines end with CR+LF  (\r\n)  — 2 bytes                  │
  │  Linux/Mac: Lines end with LF    (\n)    — 1 byte                   │
  │                                                                      │
  │  Problem:                                                            │
  │  ┌────────────────────────────────────────────────────────────┐      │
  │  │  Windows dev commits files with CRLF                       │      │
  │  │  Linux CI checks out → sees CRLF → tests may fail          │      │
  │  │  git diff shows EVERY line changed (just line endings)     │      │
  │  │  Merge conflicts on files that haven't "really" changed   │      │
  │  └────────────────────────────────────────────────────────────┘      │
  │                                                                      │
  │  Solution: .gitattributes (committed to repo — consistent for all)  │
  │  ┌────────────────────────────────────────────────────────────┐      │
  │  │  # .gitattributes                                          │      │
  │  │  * text=auto                  # Auto-detect text files,    │      │
  │  │                               # normalize to LF in repo    │      │
  │  │  *.sh text eol=lf            # Force LF for shell scripts │      │
  │  │  *.bat text eol=crlf         # Force CRLF for batch files │      │
  │  │  *.png binary                # Treat as binary (no conv)  │      │
  │  │  *.jpg binary                                              │      │
  │  │  *.exe binary                                              │      │
  │  └────────────────────────────────────────────────────────────┘      │
  │                                                                      │
  │  Git config (per-developer fallback):                                │
  │  git config --global core.autocrlf input    # Linux/Mac              │
  │  git config --global core.autocrlf true     # Windows                │
  │                                                                      │
  │  "input" = convert CRLF→LF on commit, no conversion on checkout     │
  │  "true"  = convert CRLF→LF on commit, LF→CRLF on checkout           │
  │  "false" = no conversion (dangerous on mixed teams)                  │
  └──────────────────────────────────────────────────────────────────────┘
```

> **DevOps Pro Tip:** ALWAYS commit a `.gitattributes` file in every repo with at least `* text=auto`. This normalizes line endings in the Git database (LF) regardless of developer OS. This prevents the "every line changed" diffs and cross-platform build failures.

### 15.6 Resolving Binary Conflicts

Git can't merge binary files (images, compiled files, protobuf). When a binary conflicts:

```bash
# During a merge conflict on a binary file:
# Git shows:
#   CONFLICT (content): Merge conflict in logo.png

# Option 1: Keep ours (current branch version)
git checkout --ours logo.png
git add logo.png

# Option 2: Keep theirs (incoming branch version)
git checkout --theirs logo.png
git add logo.png

# Option 3: Use a custom merge driver (for specific binary formats)
# .gitattributes:
# *.psd merge=keepTheirs

# In Git config:
# git config merge.keepTheirs.driver "cp %B %A"
```

### 15.7 Fixing a Corrupted Repo

Rare, but it happens — usually from disk issues, interrupted operations, or NFS storage.

```bash
# Step 1: Check for corruption
git fsck --full

# Step 2: If objects are missing, try to recover from remote
git fetch origin

# Step 3: If pack files are corrupted
mv .git/objects/pack .git/objects/pack.bak
git fetch origin
git repack -a -d

# Step 4: Nuclear option — re-clone with local changes preserved
cd ..
mv repo repo-broken
git clone https://github.com/org/repo.git
cp -r repo-broken/uncommitted-files repo/
# Apply uncommitted changes from broken repo

# Step 5: Recover from reflog if available
git reflog                         # Check if reflog entries exist
git fsck --lost-found              # Check .git/lost-found/ for dangling objects
```

### 15.8 Edge Cases Quick Reference

| Scenario | Command / Fix | Notes |
|----------|--------------|-------|
| Stash pop with conflict | Resolve conflicts → `git add` → `git stash drop` | Stash stays in stack on conflict |
| Cherry-pick range | `git cherry-pick A^..D` | `A..D` excludes A; `A^..D` includes A |
| Create orphan branch | `git checkout --orphan <name>` | No parent commit — new root |
| Fix wrong author | `git commit --amend --author="..."` | Rewrites hash — needs force push |
| CRLF issues | `.gitattributes` with `* text=auto` | Commit this file to repo root |
| Binary merge conflict | `git checkout --ours/--theirs <file>` | No auto-merge for binaries |
| Corrupted repo | `git fsck --full` → `git fetch` → `git repack` | Try remote recovery first |
| Accidental commit to wrong branch | `git stash` → switch → `git stash pop` | Or: reset + cherry-pick |
| Committed to main instead of feature | `git branch feature` → `git reset --hard HEAD~1` | Creates feature at current commit, resets main back |
| Want to undo `git add` | `git restore --staged <file>` | Modern way (Git 2.23+) |

> **⚠️ Gotcha (NFS/Network Filesystems):** Running Git on NFS or CIFS-mounted drives causes corruption, lock file issues, and performance problems. Git relies on POSIX atomic rename and file locking — NFS doesn't guarantee these. Always clone to local disk. CI runners should never use network-mounted workspace directories.

---

## 16. Git in Production Environments

### 16.1 Why This Section Is DevOps-Critical

In production, Git isn't just version control — it's the **release mechanism**. Tags trigger deployments, branches map to environments, and rollback means reverting to a prior Git state. Misconfigured tagging, broken hotfix workflows, or weak branch protection can cause outages.

### 16.2 Tagging Releases — Semantic Versioning

Tags mark specific commits as release points. They are **immutable pointers** (unlike branches which move).

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │                    SEMANTIC VERSIONING (SemVer)                       │
  │                                                                      │
  │                    v MAJOR . MINOR . PATCH                           │
  │                      │       │       │                               │
  │                      │       │       └── Bug fixes, no API change    │
  │                      │       └────────── New features, backward      │
  │                      │                   compatible                  │
  │                      └────────────────── Breaking changes            │
  │                                                                      │
  │  Examples:                                                           │
  │  v1.0.0 → v1.0.1   (patch: bug fix)                                 │
  │  v1.0.1 → v1.1.0   (minor: new feature, backward compatible)        │
  │  v1.1.0 → v2.0.0   (major: breaking change)                         │
  │                                                                      │
  │  Pre-release:                                                        │
  │  v2.0.0-alpha.1                                                      │
  │  v2.0.0-beta.1                                                       │
  │  v2.0.0-rc.1        (release candidate)                              │
  │                                                                      │
  │  Build metadata:                                                     │
  │  v1.2.3+build.456                                                    │
  │  v1.2.3+20260303                                                     │
  └──────────────────────────────────────────────────────────────────────┘
```

#### Lightweight vs Annotated Tags

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │              TAG TYPES                                               │
  │                                                                      │
  │  Lightweight Tag:                    Annotated Tag:                   │
  │  ┌──────────────────────┐            ┌──────────────────────┐        │
  │  │  Just a pointer to   │            │  A full Git OBJECT   │        │
  │  │  a commit (like a    │            │  (stored in .git/    │        │
  │  │  branch that doesn't │            │   objects/)          │        │
  │  │  move)               │            │                      │        │
  │  │                      │            │  Contains:           │        │
  │  │  No metadata         │            │  • Tagger name/email │        │
  │  │  No message          │            │  • Date              │        │
  │  │  No GPG signature    │            │  • Message           │        │
  │  │                      │            │  • Optional GPG sig  │        │
  │  │  git tag v1.0.0      │            │                      │        │
  │  │                      │            │  git tag -a v1.0.0   │        │
  │  │                      │            │    -m "Release 1.0"  │        │
  │  └──────────────────────┘            └──────────────────────┘        │
  │                                                                      │
  │  RULE: Always use ANNOTATED tags for releases.                       │
  │  Lightweight tags are for personal/temporary use.                    │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# Create annotated tag (recommended for releases)
git tag -a v1.2.0 -m "Release v1.2.0: add OAuth2, fix payment bug"

# Create signed tag (annotated + GPG signature)
git tag -s v1.2.0 -m "Signed release v1.2.0"

# Create lightweight tag (quick bookmark — NOT for releases)
git tag v1.2.0-dev

# Tag a specific commit (not HEAD)
git tag -a v1.1.5 abc1234 -m "Hotfix: patch security vuln"

# List tags
git tag                            # All tags
git tag -l "v1.*"                  # Tags matching pattern
git tag -l --sort=-version:refname # Sort by version (newest first)

# Show tag details
git show v1.2.0                    # Shows tag metadata + commit

# Push tags (tags are NOT pushed by default!)
git push origin v1.2.0             # Push specific tag
git push origin --tags             # Push ALL tags
git push --follow-tags             # Push commits + their annotated tags

# Delete tags
git tag -d v1.2.0                  # Delete locally
git push origin --delete v1.2.0    # Delete from remote
git push origin :refs/tags/v1.2.0  # Alternative delete syntax

# Describe current position relative to nearest tag
git describe --tags                # e.g., "v1.2.0-14-gabcdef1"
                                   # = 14 commits after v1.2.0, at commit abcdef1
```

> **⚠️ Gotcha:** `git push` does NOT push tags by default. You must explicitly `git push --tags` or `git push origin <tag>`. Many engineers create a tag locally and wonder why CI didn't trigger — the tag was never pushed. Add `git push --follow-tags` to your workflow or configure `push.followTags = true`.

### 16.3 Tag-Based Deployment Flow

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │              TAG-BASED DEPLOYMENT ARCHITECTURE                       │
  │                                                                      │
  │  Developer                                                           │
  │     │                                                                │
  │     │  1. Merge PR to main                                           │
  │     │  2. git tag -a v1.2.0 -m "Release v1.2.0"                     │
  │     │  3. git push origin v1.2.0                                     │
  │     ▼                                                                │
  │  ┌──────────────────┐                                                │
  │  │  GitHub/GitLab   │                                                │
  │  │  detects tag     │                                                │
  │  │  push event      │                                                │
  │  └────────┬─────────┘                                                │
  │           │ webhook                                                  │
  │           ▼                                                          │
  │  ┌──────────────────────────────────────────────┐                    │
  │  │  CI/CD Pipeline (triggered by tag pattern)    │                    │
  │  │                                               │                    │
  │  │  if: tag matches v*                           │                    │
  │  │                                               │                    │
  │  │  ┌──────┐  ┌──────┐  ┌──────────┐  ┌──────┐  │                    │
  │  │  │Build │─►│Test  │─►│ Security │─►│Deploy│  │                    │
  │  │  │      │  │      │  │ Scan     │  │ Prod │  │                    │
  │  │  └──────┘  └──────┘  └──────────┘  └──────┘  │                    │
  │  └──────────────────────────────────────────────┘                    │
  │                                                                      │
  │  Rollback = deploy a previous tag:                                   │
  │  git checkout v1.1.0 && deploy                                       │
  │  OR: re-trigger pipeline with v1.1.0 tag                             │
  └──────────────────────────────────────────────────────────────────────┘
```

### 16.4 Hotfix Strategy

When production is broken, you need to fix it without deploying unfinished features.

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │              HOTFIX WORKFLOW                                          │
  │                                                                      │
  │  Scenario: v1.2.0 is in production, bug found.                       │
  │  main has unreleased features (v1.3.0-dev).                          │
  │                                                                      │
  │  main:     A ◄── B ◄── C(v1.2.0) ◄── D ◄── E   (unreleased)        │
  │                                                                      │
  │  OPTION 1: Hotfix branch from tag (Git Flow style)                   │
  │  ──────────────────────────────────────────────                      │
  │  $ git checkout v1.2.0                                               │
  │  $ git checkout -b hotfix/fix-payment                                │
  │  $ (fix the bug, commit)                                             │
  │  $ git tag -a v1.2.1 -m "Hotfix: fix payment processing"            │
  │  $ git push origin hotfix/fix-payment --tags                         │
  │  → Deploy v1.2.1                                                     │
  │  → Merge hotfix back to main:                                        │
  │    git checkout main && git merge hotfix/fix-payment                  │
  │                                                                      │
  │  Result:                                                             │
  │  main:     A─B─C(v1.2.0)─D─E─────M   (merge commit)                │
  │                    \             /                                    │
  │  hotfix:            └──F(v1.2.1)┘                                    │
  │                                                                      │
  │  OPTION 2: Cherry-pick to release branch (Trunk-based style)         │
  │  ──────────────────────────────────────────────                      │
  │  $ git checkout main                                                 │
  │  $ (fix bug on main, commit G)                                       │
  │  $ git checkout -b release/v1.2 v1.2.0    ← branch from tag         │
  │  $ git cherry-pick G                       ← pick the fix            │
  │  $ git tag -a v1.2.1 -m "Hotfix"                                    │
  │  $ git push origin release/v1.2 --tags                               │
  │  → Deploy v1.2.1                                                     │
  │                                                                      │
  │  OPTION 3: Revert on main (GitOps style)                             │
  │  ──────────────────────────────────────────                          │
  │  $ git revert <bad-commit>                                           │
  │  $ git push origin main                                              │
  │  → ArgoCD auto-deploys the reverted state                            │
  │  → Fix properly later                                                │
  └──────────────────────────────────────────────────────────────────────┘
```

### 16.5 Branch Protection Rules — Production Enforcement

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         RECOMMENDED BRANCH PROTECTION (production repos)             │
  │                                                                      │
  │  ┌──────────────────────────────────────────────────────────────┐    │
  │  │  main / production branch:                                   │    │
  │  │                                                              │    │
  │  │  ✅ Require PR with at least 1 approval                     │    │
  │  │  ✅ Dismiss stale PR approvals when new commits pushed      │    │
  │  │  ✅ Require status checks (CI must pass)                    │    │
  │  │  ✅ Require branches to be up to date before merge          │    │
  │  │  ✅ Require signed commits (GPG/SSH)                        │    │
  │  │  ✅ Require linear history (rebase-only, no merge commits)  │    │
  │  │  ❌ Block force pushes (NEVER allow on main)                │    │
  │  │  ❌ Block branch deletion                                   │    │
  │  │  ✅ Require CODEOWNERS review for critical paths            │    │
  │  │  ✅ Restrict pushes to specific teams/bots only             │    │
  │  └──────────────────────────────────────────────────────────────┘    │
  │                                                                      │
  │  ┌──────────────────────────────────────────────────────────────┐    │
  │  │  develop / staging branch:                                   │    │
  │  │                                                              │    │
  │  │  ✅ Require PR with at least 1 approval                     │    │
  │  │  ✅ Require status checks                                   │    │
  │  │  ⚠️ Allow force push (for rebase workflows)                 │    │
  │  │  ❌ Block branch deletion                                   │    │
  │  └──────────────────────────────────────────────────────────────┘    │
  │                                                                      │
  │  ┌──────────────────────────────────────────────────────────────┐    │
  │  │  feature/* branches:                                         │    │
  │  │                                                              │    │
  │  │  ⚠️ Minimal protection — developers need freedom             │    │
  │  │  ✅ Status checks (lint, tests)                              │    │
  │  │  ✅ Allow force push (for interactive rebase)                │    │
  │  └──────────────────────────────────────────────────────────────┘    │
  │                                                                      │
  │  Enforce as Code (Terraform):                                        │
  │  resource "github_branch_protection" "main" {                        │
  │    repository_id  = github_repository.repo.node_id                   │
  │    pattern        = "main"                                           │
  │    enforce_admins = true                                             │
  │    required_pull_request_reviews {                                    │
  │      required_approving_review_count = 1                             │
  │      dismiss_stale_reviews           = true                          │
  │      require_code_owner_reviews      = true                          │
  │    }                                                                 │
  │    required_status_checks {                                          │
  │      strict = true                                                   │
  │      contexts = ["ci/build", "ci/test", "security/scan"]             │
  │    }                                                                 │
  │  }                                                                   │
  └──────────────────────────────────────────────────────────────────────┘
```

### 16.6 CODEOWNERS — Per-Path Review Requirements

```bash
# .github/CODEOWNERS (GitHub) or CODEOWNERS (GitLab)
# Format: <pattern>  <owners>

# Default owner for everything
*                       @devops-team

# Infrastructure — requires SRE review
infra/terraform/        @sre-team
infra/k8s/              @sre-team @platform-team

# Security-sensitive files
.github/workflows/      @security-team @devops-team
Dockerfile              @security-team
*.pem                   @security-team

# Service-specific ownership
services/auth/          @auth-team
services/payment/       @payment-team @security-team

# Critical configs
.env.example            @devops-team @security-team
```

> **DevOps Relevance:** CODEOWNERS + branch protection = **mandatory expert review** for critical paths. An intern can't accidentally merge a Terraform change without SRE approval. This is a core guardrail in production Git workflows.

### 16.7 CI Enforcement Policies

| Policy | Implementation | Why |
|--------|---------------|-----|
| **All tests pass** | Required status check | No broken code reaches main |
| **Security scan clean** | Required status check (Snyk, Trivy, gitleaks) | No vulns in production |
| **Lint/format clean** | Required status check | Consistent code quality |
| **Minimum coverage** | Required status check | Prevent regression gaps |
| **CODEOWNERS approved** | Branch protection rule | Expert review for critical paths |
| **No direct push** | Branch protection rule | Everything through PR |
| **Signed commits** | Branch protection rule | Identity verification |
| **Linear history** | Branch protection rule | Clean rollback path |
| **Auto-delete merged branches** | Repo setting | Keep ref list clean |
| **Semantic PR title** | GitHub Action / bot | Auto-generate changelog |

---

## 17. Git Internals for Elite Level

### 17.1 Why Go This Deep?

Very few DevOps engineers understand Git at the storage layer. This knowledge helps you:
- Debug obscure Git failures that no StackOverflow answer covers
- Optimize large repo performance at the fundamental level
- Understand why certain operations are fast/slow
- Build custom Git tooling for your organization
- Pass elite-level interviews at companies that care about fundamentals

### 17.2 How Git Stores Objects — Content-Addressable Filesystem

Every object in Git is stored as a file named by its SHA-1 hash. The storage format is:

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │              GIT OBJECT STORAGE FORMAT                               │
  │                                                                      │
  │  Raw content:  "Hello World\n"                                       │
  │                                                                      │
  │  Git wraps it:                                                       │
  │  ┌──────────────────────────────────────────────┐                    │
  │  │  "blob 12\0Hello World\n"                    │                    │
  │  │   │    │  │                                  │                    │
  │  │   │    │  └── NULL byte separator            │                    │
  │  │   │    └───── Content length (12 bytes)      │                    │
  │  │   └────────── Object type                    │                    │
  │  └──────────────────────────────────────────────┘                    │
  │                                                                      │
  │  SHA-1 of wrapped content:                                           │
  │  557db03de997c86a4a028e1ebd3a1ceb225be238                            │
  │                                                                      │
  │  Stored at:                                                          │
  │  .git/objects/55/7db03de997c86a4a028e1ebd3a1ceb225be238              │
  │               ^^                                                     │
  │               First 2 chars = directory name                         │
  │               Remaining 38 chars = filename                          │
  │                                                                      │
  │  The file is zlib-compressed.                                        │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# Prove it yourself:
echo -n "Hello World" | git hash-object --stdin
# Output: 557db03de997c86a...

# Manual SHA-1 calculation (what Git does internally):
echo -ne "blob 11\0Hello World" | sha1sum
# Same hash!

# Write object and inspect:
echo "Hello World" | git hash-object -w --stdin
git cat-file -t 557db03    # "blob"
git cat-file -s 557db03    # "12" (size)
git cat-file -p 557db03    # "Hello World\n" (content)
```

### 17.3 Tree Objects — Directory Snapshots

A tree object represents a **directory listing** — mapping filenames to blob/tree hashes with file mode permissions.

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │              TREE OBJECT FORMAT                                       │
  │                                                                      │
  │  $ git ls-tree HEAD                                                  │
  │                                                                      │
  │  100644 blob a1b2c3d... README.md                                    │
  │  100644 blob d4e5f6g... Makefile                                     │
  │  040000 tree h7i8j9k... src/                                         │
  │  100755 blob l0m1n2o... deploy.sh                                    │
  │  120000 blob p3q4r5s... link-to-file                                 │
  │                                                                      │
  │  Mode bits:                                                          │
  │  ┌──────────┬────────────────────────────────────┐                   │
  │  │  100644  │  Regular file (non-executable)     │                   │
  │  │  100755  │  Executable file                   │                   │
  │  │  040000  │  Directory (tree object)           │                   │
  │  │  120000  │  Symbolic link                     │                   │
  │  │  160000  │  Git submodule (gitlink)           │                   │
  │  └──────────┴────────────────────────────────────┘                   │
  │                                                                      │
  │  Note: Git only tracks these modes — NOT full Unix permissions.      │
  │  No uid/gid, no sticky bit, no setuid. Just executable or not.       │
  └──────────────────────────────────────────────────────────────────────┘
```

### 17.4 Commit Object — The Metadata Layer

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │              COMMIT OBJECT — RAW FORMAT                              │
  │                                                                      │
  │  $ git cat-file -p HEAD                                              │
  │                                                                      │
  │  tree d8329fc1cc938780ffdd9f94e0d364e0ea74f579                       │
  │  parent 7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b                   │
  │  author Jane Doe <jane@company.com> 1709500800 +0530                 │
  │  committer Jane Doe <jane@company.com> 1709500800 +0530              │
  │                                                                      │
  │  feat(auth): add OAuth2 login flow                                   │
  │                                                                      │
  │  ┌──────────────────────────────────────────────────────┐            │
  │  │  tree:      points to root tree (project snapshot)   │            │
  │  │  parent:    points to previous commit (0, 1, or 2+)  │            │
  │  │             0 parents = root commit                   │            │
  │  │             1 parent  = normal commit                 │            │
  │  │             2 parents = merge commit                  │            │
  │  │  author:    who wrote the code + timestamp            │            │
  │  │  committer: who committed (may differ from author)    │            │
  │  │  message:   commit description                        │            │
  │  └──────────────────────────────────────────────────────┘            │
  │                                                                      │
  │  Author vs Committer:                                                │
  │  • Author: original writer (e.g., someone who emailed a patch)       │
  │  • Committer: who applied it to the repo (e.g., the maintainer)      │
  │  • In most cases, they're the same person.                           │
  │  • They differ after: cherry-pick, rebase, git am (apply patch)      │
  └──────────────────────────────────────────────────────────────────────┘
```

### 17.5 Packfiles & Delta Compression — Deep Dive

When Git runs `git gc`, it packs loose objects into a single **packfile** with an **index**. Inside, it uses **delta compression** — storing similar objects as deltas from a base object.

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │              PACKFILE INTERNALS                                       │
  │                                                                      │
  │  .git/objects/pack/                                                  │
  │  ├── pack-abc123.pack              ← Binary file with all objects    │
  │  └── pack-abc123.idx              ← Index for fast lookups           │
  │                                                                      │
  │  Pack file structure:                                                │
  │  ┌──────────────────────────────────────────────────┐                │
  │  │  HEADER: "PACK" + version + object count         │                │
  │  │                                                  │                │
  │  │  Object 1: [type][size][compressed data]         │                │
  │  │  Object 2: [type][size][compressed data]         │                │
  │  │  Object 3: [OFS_DELTA][base offset][delta data]  │ ← delta!      │
  │  │  Object 4: [REF_DELTA][base SHA-1][delta data]   │ ← delta!      │
  │  │  ...                                             │                │
  │  │                                                  │                │
  │  │  TRAILER: SHA-1 checksum of entire pack          │                │
  │  └──────────────────────────────────────────────────┘                │
  │                                                                      │
  │  Delta compression:                                                  │
  │  ┌──────────────────────────────────────────────────┐                │
  │  │  File V1 (1000 lines)  ─── stored as full object │                │
  │  │  File V2 (1003 lines)  ─── stored as:            │                │
  │  │    "base = V1, add 3 lines at line 500"          │                │
  │  │    Size: ~100 bytes instead of full file          │                │
  │  │                                                  │                │
  │  │  Git chooses the best "base" object              │                │
  │  │  automatically — not necessarily the              │                │
  │  │  chronological predecessor!                       │                │
  │  └──────────────────────────────────────────────────┘                │
  │                                                                      │
  │  Index (.idx) enables O(1) lookup:                                   │
  │  SHA-1 → offset in .pack → read object directly                      │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# Inspect packfile contents
git verify-pack -v .git/objects/pack/pack-*.idx | head -20

# Output format:
# SHA-1  type  size  size-in-pack  offset  [depth  base-SHA-1]
# abc123 blob  15234  4521         1234
# def456 blob  15280  124          5678     1  abc123    ← DELTA (124 bytes vs 15KB!)

# Count objects and size
git count-objects -v
# count: 0           ← loose objects
# packs: 1           ← number of packfiles
# size-pack: 12345   ← total packfile size in KB

# Force repack with aggressive delta compression
git repack -a -d --depth=250 --window=250
# depth: max delta chain length (deeper = smaller, slower)
# window: objects considered for delta base (wider = better compression, slower)
```

### 17.6 The Commit Graph — Speeding Up History Traversal

The commit-graph file (`.git/objects/info/commit-graph`) is an **acceleration structure** that caches commit metadata for fast traversal. Without it, Git must read and decompress each commit object to walk history.

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │              COMMIT GRAPH FILE                                       │
  │                                                                      │
  │  Without commit-graph:                With commit-graph:             │
  │  ┌──────────────────────┐            ┌──────────────────────┐        │
  │  │  git log = read each │            │  git log = read      │        │
  │  │  commit object from  │            │  pre-computed graph  │        │
  │  │  packfile, decompress│            │  file (single seek)  │        │
  │  │  zlib, parse text    │            │                      │        │
  │  │                      │            │  Fixed-width records:│        │
  │  │  O(n) disk reads     │            │  • OID               │        │
  │  │  per commit in log   │            │  • Tree OID           │        │
  │  │                      │            │  • Parent indices     │        │
  │  │  SLOW on large repos │            │  • Generation number  │        │
  │  │  (100K+ commits)     │            │  • Commit timestamp   │        │
  │  │                      │            │                      │        │
  │  │                      │            │  O(1) per commit      │        │
  │  │                      │            │  FAST even on huge    │        │
  │  │                      │            │  repos                │        │
  │  └──────────────────────┘            └──────────────────────┘        │
  │                                                                      │
  │  Generation numbers enable pruning:                                  │
  │  "Is commit A an ancestor of commit B?"                              │
  │  Without graph: walk entire history = O(n)                           │
  │  With graph: compare generation numbers = O(1) to prune,             │
  │              then short walk = much faster                           │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# Generate commit graph
git commit-graph write --reachable

# Enable auto-generation on fetch
git config fetch.writeCommitGraph true

# Verify commit graph
git commit-graph verify

# Speed test (try on a large repo):
time git log --oneline | wc -l           # Without graph
git commit-graph write --reachable
time git log --oneline | wc -l           # With graph — noticeably faster
```

### 17.7 Refs & Reflogs — The Reference System

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │              GIT REFERENCE HIERARCHY                                  │
  │                                                                      │
  │  .git/                                                               │
  │  ├── HEAD                    ← "ref: refs/heads/main"               │
  │  │                              (symbolic ref → current branch)       │
  │  ├── ORIG_HEAD               ← Backup of HEAD before risky ops       │
  │  │                              (merge, rebase, reset)               │
  │  ├── FETCH_HEAD              ← Result of last git fetch              │
  │  ├── MERGE_HEAD              ← The commit being merged (during merge)│
  │  ├── CHERRY_PICK_HEAD        ← The commit being cherry-picked        │
  │  │                                                                   │
  │  ├── refs/                                                           │
  │  │   ├── heads/              ← Local branches                        │
  │  │   │   ├── main            ← contains: abc1234...                  │
  │  │   │   └── feature-x       ← contains: def5678...                  │
  │  │   ├── tags/               ← Tag references                        │
  │  │   │   ├── v1.0.0          ← annotated: points to tag object       │
  │  │   │   └── v1.0.1          ← lightweight: points to commit         │
  │  │   ├── remotes/            ← Remote tracking branches              │
  │  │   │   └── origin/                                                 │
  │  │   │       ├── main        ← Last known position of origin/main    │
  │  │   │       └── feature-x                                           │
  │  │   └── stash               ← Stash reference                       │
  │  │                                                                   │
  │  ├── packed-refs             ← Packed refs (optimization)             │
  │  │                              All refs in one file instead of       │
  │  │                              many small files                      │
  │  │                                                                   │
  │  └── logs/                   ← Reflog entries                         │
  │      ├── HEAD                ← History of HEAD movements             │
  │      └── refs/heads/                                                 │
  │          ├── main            ← History of main movements             │
  │          └── feature-x                                               │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# Inspect refs directly
cat .git/HEAD                         # "ref: refs/heads/main"
cat .git/refs/heads/main              # commit hash
cat .git/packed-refs                  # All packed refs

# List all refs
git for-each-ref                      # All refs with type and hash
git for-each-ref --format='%(refname:short) %(objecttype) %(objectname:short)' refs/heads/

# Symbolic refs
git symbolic-ref HEAD                 # "refs/heads/main"
git symbolic-ref HEAD refs/heads/feature  # Switch HEAD (low-level)

# Pack refs for performance
git pack-refs --all                   # Merge loose ref files into packed-refs
```

### 17.8 The Index (Staging Area) — Binary Format

The index (`.git/index`) is a binary file that represents the **staging area**. It's a flat, sorted list of all tracked files with their metadata.

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │              THE INDEX FILE                                          │
  │                                                                      │
  │  .git/index (binary format):                                         │
  │                                                                      │
  │  ┌──────────────────────────────────────────────────┐                │
  │  │  HEADER: "DIRC" + version (2/3/4) + entry count  │                │
  │  │                                                  │                │
  │  │  Entry 1:                                        │                │
  │  │  ├── ctime (creation time)                       │                │
  │  │  ├── mtime (modification time)                   │                │
  │  │  ├── dev, ino (device, inode — for filesystem)   │                │
  │  │  ├── mode (100644 or 100755)                     │                │
  │  │  ├── uid, gid                                    │                │
  │  │  ├── file size                                   │                │
  │  │  ├── SHA-1 of the blob                           │                │
  │  │  ├── flags (name length, stages for merge)       │                │
  │  │  └── file path (variable length, NUL terminated) │                │
  │  │                                                  │                │
  │  │  Entry 2: ...                                    │                │
  │  │  Entry N: ...                                    │                │
  │  │                                                  │                │
  │  │  EXTENSIONS: tree cache, resolve-undo, etc.      │                │
  │  │  TRAILER: SHA-1 checksum                         │                │
  │  └──────────────────────────────────────────────────┘                │
  │                                                                      │
  │  Why ctime/mtime/inode?                                              │
  │  Git uses these to quickly detect if a file changed WITHOUT          │
  │  reading its content — the "stat cache." This makes git status       │
  │  fast even with thousands of files.                                  │
  │                                                                      │
  │  Why this matters for DevOps:                                        │
  │  • Docker containers reset mtimes → git status may be confused       │
  │  • NFS doesn't preserve inode numbers → broken stat cache            │
  │  • CI runners with shared workspace → stale index                    │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# Inspect the index
git ls-files --stage                  # Show all entries with mode, hash, stage
git ls-files -v                       # Show assume-unchanged flags

# Update index (useful for ignoring local changes temporarily)
git update-index --assume-unchanged file.txt    # Ignore local changes
git update-index --no-assume-unchanged file.txt # Track again
git update-index --skip-worktree file.txt       # Stronger ignore (survives checkout)
```

### 17.9 SHA-1 to SHA-256 Transition

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │              SHA-1 → SHA-256 MIGRATION                               │
  │                                                                      │
  │  SHA-1 (current default):            SHA-256 (future):               │
  │  ┌──────────────────────┐            ┌──────────────────────┐        │
  │  │  160 bits (40 hex)   │            │  256 bits (64 hex)   │        │
  │  │  Collision found     │            │  No known collisions │        │
  │  │  (SHAttered, 2017)   │            │  Quantum-resistant   │        │
  │  │                      │            │  (more than SHA-1)   │        │
  │  │  Still used because: │            │                      │        │
  │  │  • Practical attacks │            │  Enabled via:        │        │
  │  │    require huge      │            │  git init            │        │
  │  │    compute           │            │    --object-format=  │        │
  │  │  • Git hardens       │            │    sha256            │        │
  │  │    against collision │            │                      │        │
  │  │    attacks           │            │  Status (2026):      │        │
  │  │                      │            │  Experimental but    │        │
  │  │                      │            │  functional.         │        │
  │  │                      │            │  Not yet default.    │        │
  │  │                      │            │  Interop with SHA-1  │        │
  │  │                      │            │  repos is complex.   │        │
  │  └──────────────────────┘            └──────────────────────┘        │
  │                                                                      │
  │  For DevOps: monitor this transition. When SHA-256 becomes           │
  │  default, repo migrations will be needed. GitHub/GitLab will         │
  │  handle most of this, but self-hosted instances need planning.       │
  └──────────────────────────────────────────────────────────────────────┘
```

### 17.10 Internals Exploration Commands — Cheat Sheet

| Command | What It Shows | When to Use |
|---------|--------------|------------|
| `git cat-file -p <hash>` | Pretty-print any object | Inspect commits, trees, blobs, tags |
| `git cat-file -t <hash>` | Object type | Identify unknown hash |
| `git cat-file -s <hash>` | Object size in bytes | Debug storage issues |
| `git ls-tree HEAD` | Root tree of current commit | See directory snapshot |
| `git ls-tree -r HEAD` | All files recursively | Full file listing with hashes |
| `git ls-files --stage` | Staging area contents | Debug merge conflicts (stages 1,2,3) |
| `git rev-parse HEAD` | Resolve ref to hash | Get commit hash in scripts |
| `git rev-parse --git-dir` | Path to .git directory | Locate repo in scripts |
| `git for-each-ref` | All refs with metadata | Audit branches and tags |
| `git verify-pack -v *.idx` | Packfile contents | Analyze delta chains, sizes |
| `git count-objects -v` | Object count and storage | Monitor repo size |
| `git fsck --full` | Repository integrity check | Detect corruption |
| `git reflog show HEAD` | HEAD movement history | Disaster recovery |
| `git commit-graph verify` | Commit graph integrity | Performance debugging |

> **DevOps Pro Tip:** Build a Git health-check script that runs `git count-objects -v`, `git fsck`, and `git commit-graph verify` on your self-hosted repos. Schedule it as a cron job. Catch corruption and bloat before they become outages.

---

## 18. Production Scenario Based FAQ

Real-world Git scenarios that DevOps engineers face in production — not textbook examples, but messy, high-pressure situations with actual solutions.

---

### FAQ 1: "A developer force-pushed to main and overwrote the last 10 commits. Deployment is broken."

**Severity:** 🔴 Critical — production code lost

**What happened:**
```
  Before force push:    main:  A ◄── B ◄── C ◄── D ◄── E (deployed)
  After force push:     main:  A ◄── B ◄── X ◄── Y       (dev's branch)
  Commits C, D, E are now unreachable from main.
```

**Immediate fix (within 90 days of reflog retention):**
```bash
# 1. Find the lost commit on the server (if you have SSH/admin access)
#    On the remote (GitHub/GitLab), check the "push events" audit log
#    for the SHA of commit E.

# 2. On ANY machine that had the original main:
git reflog show origin/main
# Look for the entry BEFORE the force push:
# abc1234 origin/main@{1}: fetch: forced-update

# 3. Restore main to the correct commit:
git push --force-with-lease origin abc1234:main

# 4. If no one has the old reflog, check CI artifacts:
#    CI runners may have cloned the repo before the force push.
#    The .git directory in that clone still has the old commits.
```

**Prevention (MUST implement after incident):**
```bash
# On GitHub/GitLab: Enable branch protection on main
# ✅ Block force pushes
# ✅ Require PR with approvals
# ✅ Require status checks

# Server-side hook (self-hosted GitLab/Gitea):
#!/bin/bash
# pre-receive hook — block force pushes to main
while read oldrev newrev refname; do
  if [[ "$refname" == "refs/heads/main" ]]; then
    if ! git merge-base --is-ancestor "$oldrev" "$newrev" 2>/dev/null; then
      echo "ERROR: Force push to main is BLOCKED."
      echo "Contact DevOps team for exceptions."
      exit 1
    fi
  fi
done
```

> **⚠️ Gotcha:** `--force-with-lease` is NOT a silver bullet. It only checks if your local remote-tracking ref matches the actual remote. If you ran `git fetch` recently, the lease is "refreshed" and the force push succeeds even if someone else pushed. Use branch protection rules — don't rely on developer discipline.

---

### FAQ 2: "Our repo is 8 GB and git clone takes 20 minutes. CI builds are timing out."

**Severity:** 🟡 High — slowing down entire team + CI

**Diagnosis:**
```bash
# Check what's taking space
git count-objects -v
# Look at: size-pack (in KB)

# Find the largest objects
git rev-list --objects --all \
  | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' \
  | awk '/^blob/ {print $3, $4}' \
  | sort -rn \
  | head -20

# Common culprits:
# • .zip/.tar archives committed to repo
# • Compiled binaries / .jar / .dll
# • ML model weights (.h5, .pt, .onnx)
# • Database dumps (.sql)
# • Vendor directories (node_modules committed accidentally)
# • Large images / videos in docs
```

**Fix — multi-pronged approach:**
```bash
# STEP 1: For CI — use shallow clone (immediate relief)
git clone --depth 1 https://github.com/org/repo.git
# Clone time: 20 min → 2 min

# STEP 2: For CI that needs some history
git clone --filter=blob:none https://github.com/org/repo.git
# Treeless clone — downloads tree/commit objects, blobs on demand
# Clone time: 20 min → 3 min

# STEP 3: Permanently remove large files from history
# WARNING: This rewrites history — coordinate with ALL developers
pip install git-filter-repo
git filter-repo --strip-blobs-bigger-than 10M
# OR remove specific paths:
git filter-repo --invert-paths --path data/huge_dump.sql --path models/

# STEP 4: Move large files to Git LFS going forward
git lfs install
git lfs track "*.zip" "*.tar.gz" "*.h5" "*.pt" "*.onnx"
git add .gitattributes
git commit -m "chore: track large files with LFS"

# STEP 5: After history rewrite, all developers must re-clone
# Send team-wide notice:
# "REPO HISTORY REWRITTEN — delete your local clone and re-clone."
# Old clones will have diverged history and cannot push.
```

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │              LARGE REPO — DECISION TREE                              │
  │                                                                      │
  │  Repo is too large?                                                  │
  │     │                                                                │
  │     ├── CI/CD only (no history needed)?                              │
  │     │   └── YES → git clone --depth 1                               │
  │     │                                                                │
  │     ├── Need some files but not all?                                 │
  │     │   └── YES → Sparse checkout + partial clone                   │
  │     │             git clone --filter=blob:none --sparse              │
  │     │             git sparse-checkout set src/ tests/                │
  │     │                                                                │
  │     ├── Large files still being added?                               │
  │     │   └── YES → Set up Git LFS for those file types               │
  │     │                                                                │
  │     ├── Large files already in history?                              │
  │     │   └── YES → git filter-repo to purge + LFS for future         │
  │     │                                                                │
  │     └── Monorepo with many services?                                 │
  │         └── Consider sparse checkout per team/service                │
  │             OR evaluate splitting into multiple repos                │
  └──────────────────────────────────────────────────────────────────────┘
```

---

### FAQ 3: "A secret (AWS key, database password) was committed to the repo. How do we scrub it completely?"

**Severity:** 🔴 Critical — security breach

**Immediate response (FIRST — within minutes):**
```bash
# 1. ROTATE THE SECRET IMMEDIATELY
#    Do this BEFORE scrubbing Git history.
#    Even if you delete the commit, anyone who cloned the repo already has it.
#    Bots scrape GitHub for exposed keys within seconds.

# 2. Revoke access:
#    • AWS: Deactivate the access key in IAM console
#    • Database: Change the password
#    • API: Revoke the token/key
#    • Generate new credentials

# 3. Check for unauthorized access:
#    • AWS: Check CloudTrail for API calls using the exposed key
#    • DB: Check query logs for suspicious activity
#    • Review access logs for the compromised service
```

**Scrub from Git history (AFTER rotation):**
```bash
# Option A: git filter-repo (recommended)
pip install git-filter-repo

# Remove a specific file that contained the secret:
git filter-repo --invert-paths --path config/secrets.yml

# Replace the secret string everywhere in history:
git filter-repo --replace-text <(echo 'AKIAIOSFODNN7EXAMPLE==>***REDACTED***')

# Option B: BFG Repo Cleaner (simpler for common cases)
java -jar bfg.jar --replace-text passwords.txt repo.git
# passwords.txt contains: AKIAIOSFODNN7EXAMPLE

# After scrubbing:
git reflog expire --expire=now --all
git gc --prune=now --aggressive

# Force push ALL branches (history rewritten):
git push --force --all
git push --force --tags

# ALL developers must re-clone
# ALL forks must be re-forked or force-updated
```

**Prevention checklist:**
```bash
# 1. Pre-commit hook (gitleaks)
brew install gitleaks   # or download binary
gitleaks detect --source . --verbose

# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks

# 2. CI/CD scan (every push)
# GitHub Actions:
- name: Run gitleaks
  uses: gitleaks/gitleaks-action@v2
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

# 3. .gitignore MUST include:
.env
.env.*
*.pem
*.key
*credentials*
*secret*
terraform.tfvars

# 4. Use secret managers, NEVER commit secrets:
# • AWS Secrets Manager / SSM Parameter Store
# • HashiCorp Vault
# • GitHub Actions secrets
# • GitLab CI/CD variables (masked)
```

> **⚠️ Gotcha:** Even after scrubbing history, the secret may exist in: (1) GitHub cached PR diffs, (2) forks, (3) CI build logs, (4) developer local clones, (5) backup systems, (6) cached pages in search engines. Treat the secret as **permanently compromised** — always rotate first.

---

### FAQ 4: "Two teams merged conflicting database migrations. Production schema is broken."

**Severity:** 🔴 Critical — data integrity at risk

**What happened:**
```
  Team A:  migration_003_add_users_table.sql
  Team B:  migration_003_add_orders_table.sql (same sequence number!)

  Both merged to main. ORM sees two migration_003 files.
  Depending on which runs first, the schema is inconsistent.
```

**Immediate fix:**
```bash
# 1. Check which migration ran in production
SELECT * FROM schema_migrations ORDER BY version DESC LIMIT 10;

# 2. If neither ran yet — just fix the numbering:
git mv migration_003_add_orders_table.sql migration_004_add_orders_table.sql
git commit -m "fix: renumber migration to resolve conflict"

# 3. If one already ran — manually apply the other:
#    Apply the missing migration manually in production
#    Then fix the migration files in Git for consistency
```

**Prevention — Git workflow for migrations:**
```
  ┌──────────────────────────────────────────────────────────────────────┐
  │        MIGRATION CONFLICT PREVENTION                                 │
  │                                                                      │
  │  Strategy 1: Timestamp-based naming (recommended)                    │
  │  ─────────────────────────────────────────────────                   │
  │  20260303143000_add_users_table.sql                                  │
  │  20260303150000_add_orders_table.sql                                 │
  │  Timestamps rarely collide (unlike sequential numbers)               │
  │                                                                      │
  │  Strategy 2: CI check for duplicate migration numbers                │
  │  ─────────────────────────────────────────────────                   │
  │  # In CI pipeline:                                                   │
  │  DUPES=$(ls migrations/ | cut -d'_' -f1 | sort | uniq -d)           │
  │  if [ -n "$DUPES" ]; then                                           │
  │    echo "ERROR: Duplicate migration numbers: $DUPES"                 │
  │    exit 1                                                            │
  │  fi                                                                  │
  │                                                                      │
  │  Strategy 3: Lock file for migration sequence                        │
  │  ─────────────────────────────────────────────────                   │
  │  migrations/SEQUENCE (contains next number)                          │
  │  PRs that add migrations must increment SEQUENCE                     │
  │  Git merge conflict on SEQUENCE = signal to renumber                 │
  │                                                                      │
  │  Strategy 4: CODEOWNERS on migrations/                               │
  │  ─────────────────────────────────────────────────                   │
  │  migrations/   @database-team                                        │
  │  One team reviews all migrations — catches conflicts in PR review    │
  └──────────────────────────────────────────────────────────────────────┘
```

---

### FAQ 5: "git pull says 'divergent branches' and refuses to merge. Half the team is stuck."

**Severity:** 🟡 Medium — blocks development workflow

**What happened (Git 2.27+ default behavior):**
```bash
# Git 2.27+ refuses to pull without explicit merge strategy:
hint: You have divergent branches and need to specify how to reconcile them.
hint: You can do so by running one of the following commands sometime before
hint: your next pull:
hint:
hint:   git config pull.rebase false  # merge
hint:   git config pull.rebase true   # rebase
hint:   git config pull.ff only       # fast-forward only
```

**Fix — team-wide configuration:**
```bash
# OPTION A: Rebase on pull (recommended for most teams)
# Clean linear history, no merge bubbles
git config --global pull.rebase true

# OPTION B: Fast-forward only (strictest — for trunk-based)
# Fails if local and remote diverged — forces you to rebase manually
git config --global pull.ff only

# OPTION C: Merge on pull (traditional, creates merge commits)
git config --global pull.rebase false

# Set team-wide default via repo config:
# .gitconfig in repo root (shared via commit):
[pull]
    rebase = true
```

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │        WHICH pull STRATEGY? — DECISION GUIDE                         │
  │                                                                      │
  │  pull.rebase = true (rebase)                                         │
  │  ├── ✅ Linear history                                               │
  │  ├── ✅ Clean git log                                                │
  │  ├── ⚠️ Rewrites local commits (changes SHAs)                       │
  │  └── Best for: feature branches, small teams                         │
  │                                                                      │
  │  pull.ff = only (fast-forward only)                                  │
  │  ├── ✅ Cleanest history                                             │
  │  ├── ✅ No surprise merge commits                                    │
  │  ├── ❌ Fails when diverged (must rebase manually)                   │
  │  └── Best for: trunk-based, disciplined teams                        │
  │                                                                      │
  │  pull.rebase = false (merge)                                         │
  │  ├── ✅ Preserves exact branch history                               │
  │  ├── ❌ Creates merge commits ("merge bubbles")                      │
  │  ├── ❌ Noisy git log                                                │
  │  └── Best for: open source with many contributors                    │
  └──────────────────────────────────────────────────────────────────────┘
```

---

### FAQ 6: "Our CI pipeline triggered on a docs-only change and deployed to production."

**Severity:** 🟡 Medium — wasted CI resources, unnecessary risk

**Root cause:** CI pipeline doesn't filter paths — every push to main triggers full build+deploy.

**Fix — path-based CI triggers:**
```yaml
# GitHub Actions — only trigger on code changes:
name: Deploy
on:
  push:
    branches: [main]
    paths:
      - 'src/**'
      - 'Dockerfile'
      - 'package.json'
      - 'package-lock.json'
    paths-ignore:
      - 'docs/**'
      - '*.md'
      - '.github/ISSUE_TEMPLATE/**'
      - 'LICENSE'

# GitLab CI — rules with changes:
deploy:
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      changes:
        - src/**/*
        - Dockerfile
        - package*.json
      when: always
    - when: never

# Jenkins — changeset condition:
pipeline {
  stages {
    stage('Deploy') {
      when {
        changeset "src/**"
      }
      steps {
        sh 'make deploy'
      }
    }
  }
}
```

> **DevOps Pro Tip:** Combine path filtering with **labels** or **commit message conventions**. Convention: prefix docs-only commits with `docs:` and skip CI with `[skip ci]` in the commit message. But don't rely solely on commit messages — path filters are more reliable.

---

### FAQ 7: "A merge to main passed all tests but broke production. git bisect points to a commit 3 weeks old."

**Severity:** 🔴 Critical — production outage

**How this happens:**
```
  The bug was introduced 3 weeks ago in commit X.
  Tests didn't catch it because:
  • Missing test coverage for that edge case
  • The bug only manifests under production load/data
  • Feature flags masked the behavior until they were toggled

  Feature branch was rebased onto main (tests pass in isolation)
  but the combination of commit X + new code = production failure.
```

**Using git bisect effectively:**
```bash
# 1. Start bisect with known good and bad commits
git bisect start
git bisect bad HEAD                    # Current main is broken
git bisect good v1.1.0                 # Last known working release

# 2. Git checks out a midpoint — test it
# Run your test (manually or automated):
./run_production_test.sh
# If broken:
git bisect bad
# If working:
git bisect good

# 3. Repeat until Git finds the exact commit:
# "abc1234 is the first bad commit"

# 4. AUTOMATE with a test script:
git bisect start HEAD v1.1.0
git bisect run ./test_payment_flow.sh
# Script must exit 0 for good, 1-124 for bad, 125 for skip
# Git does binary search automatically — finds culprit in O(log n) steps

# 5. For 1000 commits, bisect needs only ~10 tests to find the bug

# 6. After finding the culprit:
git bisect reset                       # Return to original HEAD
git show abc1234                       # Inspect the bad commit
```

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │        git bisect — BINARY SEARCH VISUALIZATION                      │
  │                                                                      │
  │  1000 commits between good and bad:                                  │
  │                                                                      │
  │  Step 1: Test commit 500  → BAD                                      │
  │  Step 2: Test commit 250  → GOOD                                     │
  │  Step 3: Test commit 375  → BAD                                      │
  │  Step 4: Test commit 312  → GOOD                                     │
  │  Step 5: Test commit 343  → BAD                                      │
  │  Step 6: Test commit 327  → GOOD                                     │
  │  Step 7: Test commit 335  → BAD                                      │
  │  Step 8: Test commit 331  → GOOD                                     │
  │  Step 9: Test commit 333  → BAD                                      │
  │  Step 10: Test commit 332 → THIS IS THE FIRST BAD COMMIT            │
  │                                                                      │
  │  10 tests to search 1000 commits = O(log₂ n)                        │
  └──────────────────────────────────────────────────────────────────────┘
```

**Post-mortem actions:**
1. Write a test that catches this specific bug
2. Add it to the CI pipeline
3. If the bug was data-dependent, add production-like data to test fixtures
4. Consider canary deployments to catch issues before full rollout

---

### FAQ 8: "We need to split a monorepo into separate repos without losing Git history."

**Severity:** 🟡 Medium — organizational restructure

**Scenario:**
```
  monorepo/
  ├── services/
  │   ├── auth/          → wants to become: github.com/org/auth-service
  │   ├── payment/       → wants to become: github.com/org/payment-service
  │   └── notification/  → wants to become: github.com/org/notification-service
  ├── libs/
  │   └── common/        → wants to become: github.com/org/common-lib
  └── infra/             → stays in monorepo
```

**Step-by-step extraction (preserving history):**
```bash
# 1. Clone the monorepo (fresh clone to avoid local artifacts)
git clone https://github.com/org/monorepo.git monorepo-extract
cd monorepo-extract

# 2. Extract one service with FULL history
pip install git-filter-repo

# Keep only services/auth/ and rewrite it to be the root:
git filter-repo --subdirectory-filter services/auth/
# Result: All files from services/auth/ are now at repo root
# History: Only commits that touched services/auth/ are kept

# 3. If you need shared library history too:
# More complex — use path filtering instead:
git filter-repo --path services/auth/ --path libs/common/

# 4. Create the new repo and push
git remote add origin https://github.com/org/auth-service.git
git push -u origin main --tags

# 5. Repeat for each service (start from fresh clone each time!)
cd ..
git clone https://github.com/org/monorepo.git payment-extract
cd payment-extract
git filter-repo --subdirectory-filter services/payment/
git remote add origin https://github.com/org/payment-service.git
git push -u origin main --tags
```

> **⚠️ Gotcha:** `--subdirectory-filter` rewrites history — commit hashes change. Cross-references (e.g., "fixed in commit abc123") in old issues/PRs will break. Document the old-to-new hash mapping. Also, shared libraries (`libs/common/`) referenced by multiple services require careful planning — consider publishing as a package rather than duplicating history.

---

### FAQ 9: "Developer says 'git push' is rejected with 'pre-receive hook declined.' They can't push any branch."

**Severity:** 🟡 Medium — blocks individual developer

**Common causes and fixes:**

| Cause | Diagnosis | Fix |
|-------|-----------|-----|
| **Commit not signed** | Branch protection requires signed commits | `git config commit.gpgsign true` + set up GPG/SSH key |
| **Email mismatch** | Commit email doesn't match GitHub verified email | `git config user.email "correct@company.com"` + `git commit --amend --reset-author` |
| **Large file rejected** | Pre-receive hook blocks files > size limit | Remove large file: `git rm --cached bigfile.zip && git commit --amend` |
| **Secret detected** | gitleaks/trufflehog hook found a secret | Remove secret, use env var or secret manager instead |
| **Branch name invalid** | Hook enforces naming convention | Rename: `git branch -m old-name feature/JIRA-123-description` |
| **Commit message format** | Hook requires conventional commits | `git commit --amend -m "feat(auth): add login"` |
| **Protected branch** | Pushing directly to main/release | Create a PR instead — direct push is blocked |
| **LFS objects missing** | LFS pointer committed but LFS storage quota exceeded | Check LFS storage: `git lfs env`, purchase more storage, or clean old LFS objects |

```bash
# Debug: see which hook is failing (self-hosted repos)
# Check server-side hook output in the push rejection message

# For GitHub: check repository Settings → Branches → Branch protection rules
# For GitLab: check Settings → Repository → Protected branches
# Also check: Settings → Push rules (GitLab)

# Test commit signing:
git log --show-signature -1

# Test email:
git config user.email
git log --format='%ae' -1  # Check last commit's email
```

---

### FAQ 10: "We're migrating from SVN/Mercurial/TFS to Git. How do we preserve history?"

**Severity:** 🟡 Medium — one-time migration

**SVN → Git:**
```bash
# 1. Create author mapping file (SVN usernames → Git authors)
# authors.txt:
jdoe = Jane Doe <jane@company.com>
bsmith = Bob Smith <bob@company.com>
(no author) = Unknown <unknown@company.com>

# 2. Clone SVN repo into Git (preserving branches/tags)
git svn clone \
  --stdlayout \
  --authors-file=authors.txt \
  --no-metadata \
  https://svn.company.com/repos/project \
  project-git

# --stdlayout: assumes trunk/branches/tags SVN structure
# --no-metadata: don't include git-svn-id in commit messages

# 3. Convert SVN tags to Git tags
cd project-git
for tag in $(git branch -r | grep 'tags/' | sed 's/tags\///'); do
  git tag "$tag" "refs/remotes/tags/$tag"
  git branch -r -d "tags/$tag"
done

# 4. Convert SVN branches to Git branches
for branch in $(git branch -r | grep -v 'tags/' | grep -v 'trunk'); do
  git branch "$(basename $branch)" "refs/remotes/$branch"
  git branch -r -d "$branch"
done

# 5. Set trunk as main
git branch -m trunk main

# 6. Push to new Git remote
git remote add origin https://github.com/org/project.git
git push -u origin --all
git push origin --tags

# For large SVN repos, consider: https://github.com/nirvdrum/svn2git
```

**Mercurial → Git:**
```bash
# Using hg-fast-export:
git init project-git
cd project-git
hg-fast-export.sh -r /path/to/hg-repo -A authors.txt
git checkout HEAD

# OR using GitHub's importer:
# GitHub → New Repository → Import → Paste Mercurial URL
```

**TFS/TFVC → Git:**
```bash
# Using git-tfs:
git tfs clone https://tfs.company.com/collection $/Project/Main project-git

# OR use Azure DevOps built-in migration:
# Azure Repos → Import Repository → TFVC → select path
```

> **DevOps Pro Tip:** After any VCS migration, run `git fsck --full` and `git log --all --oneline | wc -l` to verify commit count matches the source. Set up the new Git repo with proper branch protection, CI/CD, and CODEOWNERS before announcing the migration to the team.

---

### FAQ 11: "Merge conflicts in package-lock.json / yarn.lock are impossible to resolve manually."

**Severity:** 🟢 Low — annoying but solvable

**Why it happens:** Lock files are auto-generated. Two branches install different packages → merge conflict in a file with thousands of lines of JSON.

**Fix — never manually resolve lock files:**
```bash
# Strategy: Accept EITHER version, then regenerate

# Step 1: Accept "ours" (current branch version)
git checkout --ours package-lock.json
git add package-lock.json

# Step 2: Regenerate the lock file
npm install
# This merges both sets of dependencies correctly

# Step 3: Commit the resolved lock file
git add package-lock.json
git commit -m "fix: resolve package-lock.json merge conflict"
```

**Automate with merge driver:**
```bash
# .gitattributes — tell Git how to handle lock file conflicts:
package-lock.json merge=npm-merge
yarn.lock merge=yarn-merge

# .git/config (or global gitconfig):
[merge "npm-merge"]
  name = "Regenerate package-lock.json on conflict"
  driver = "cp %A %A.bak && npm install && git add package-lock.json && true"

# Alternatively, accept theirs and regenerate:
[merge "npm-merge"]
  name = "npm lock merge"
  driver = "npm install --package-lock-only"
```

---

### FAQ 12: "We need to enforce Conventional Commits across 15 repos. How?"

**Severity:** 🟢 Low — quality enforcement

**What are Conventional Commits?**
```
  ┌──────────────────────────────────────────────────────────────────────┐
  │        CONVENTIONAL COMMIT FORMAT                                    │
  │                                                                      │
  │  <type>(<scope>): <description>                                      │
  │                                                                      │
  │  [optional body]                                                     │
  │                                                                      │
  │  [optional footer(s)]                                                │
  │                                                                      │
  │  Types:                                                              │
  │  ┌──────────┬──────────────────────────────────────────┐             │
  │  │  feat     │  New feature (triggers MINOR bump)      │             │
  │  │  fix      │  Bug fix (triggers PATCH bump)          │             │
  │  │  docs     │  Documentation only                     │             │
  │  │  style    │  Formatting (no code change)            │             │
  │  │  refactor │  Code restructuring                     │             │
  │  │  perf     │  Performance improvement                │             │
  │  │  test     │  Adding/fixing tests                    │             │
  │  │  build    │  Build system changes                   │             │
  │  │  ci       │  CI configuration                       │             │
  │  │  chore    │  Maintenance tasks                      │             │
  │  │  revert   │  Revert a commit                        │             │
  │  └──────────┴──────────────────────────────────────────┘             │
  │                                                                      │
  │  BREAKING CHANGE:                                                    │
  │  feat(auth)!: change login API response format                       │
  │       ↑ The ! indicates a BREAKING CHANGE (triggers MAJOR bump)      │
  │                                                                      │
  │  Examples:                                                           │
  │  feat(cart): add coupon code validation                              │
  │  fix(payment): handle timeout in Stripe webhook                      │
  │  docs: update API reference for v2 endpoints                         │
  │  chore(deps): bump lodash from 4.17.20 to 4.17.21                   │
  └──────────────────────────────────────────────────────────────────────┘
```

**Enforcement stack (3 layers):**
```bash
# LAYER 1: Pre-commit hook (local — catches before push)
# commitlint + husky
npm install --save-dev @commitlint/cli @commitlint/config-conventional husky

# commitlint.config.js
module.exports = { extends: ['@commitlint/config-conventional'] };

# .husky/commit-msg
npx --no -- commitlint --edit "$1"

# LAYER 2: CI check (server — catches if hook bypassed)
# GitHub Actions:
- name: Validate PR title (Conventional Commits)
  uses: amannn/action-semantic-pull-request@v5
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

# LAYER 3: Branch protection (enforce PR title check)
# Settings → Branches → Require status checks → "Semantic Pull Request"
```

**Benefits for DevOps:**
- Auto-generate CHANGELOG from commit history
- Auto-determine semantic version bumps (feat→minor, fix→patch, breaking→major)
- Tools: `semantic-release`, `release-please`, `standard-version`

---

### FAQ 13: "A critical release needs to go out but one feature in main is broken. How do we release without it?"

**Severity:** 🔴 Critical — release blocked

**Strategy: Revert the broken feature, release, then re-apply:**
```bash
# 1. Identify the merge commit that brought in the broken feature
git log --oneline --merges -10
# e.g., abc1234 Merge PR #456: feat(payments): new checkout flow

# 2. Revert the merge commit
git revert -m 1 abc1234
# -m 1 = keep the mainline (main's history), undo the branch's changes

# 3. Run tests — verify the broken feature is removed
npm test

# 4. Tag and release
git tag -a v1.3.1 -m "Release: revert broken checkout, stable release"
git push origin main --tags

# 5. LATER — re-apply the feature (after fixing)
# You CANNOT just merge the feature branch again!
# Git thinks it's already merged. You must:
git revert <revert-commit-hash>  # "Revert the revert"
# Then merge the fix on top
git merge feature/checkout-fix

# The timeline looks like:
# main: ...─M(merge)─R(revert, v1.3.1 released)─RR(revert-revert)─F(fix)
```

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │        REVERT-A-MERGE WORKFLOW                                       │
  │                                                                      │
  │  main:  A ── B ── M ── R(revert) ── tag:v1.3.1 ── RR ── F          │
  │                  /                                  │                │
  │  feature:  X ── Y (broken)                   revert the revert      │
  │                                              + apply fix             │
  │                                                                      │
  │  Key insight: reverting a merge does NOT delete the branch.          │
  │  It applies the inverse diff. To re-merge, you must first           │
  │  "revert the revert" so Git sees the changes as new again.          │
  └──────────────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha:** The "revert a merge, then revert the revert" pattern confuses many engineers. Document this pattern in your team's runbook. The alternative is cherry-picking only the good commits to a release branch — but that's more error-prone with many commits.

---

### FAQ 14: "Our Git hosting (GitHub/GitLab) went down. Can we still work?"

**Severity:** 🟡 Medium — workflow disrupted, not data loss

**Why Git handles this gracefully:**
```
  ┌──────────────────────────────────────────────────────────────────────┐
  │        DISTRIBUTED = RESILIENT                                       │
  │                                                                      │
  │  Every clone is a FULL backup of the repository.                     │
  │                                                                      │
  │  GitHub is down:                                                     │
  │  ┌──────────────┐     ╳     ┌──────────────┐                        │
  │  │  Developer A  │ ────╳───►│   GitHub      │  UNREACHABLE           │
  │  │  (full repo)  │     ╳    │   (remote)    │                        │
  │  └──────────────┘           └──────────────┘                        │
  │                                                                      │
  │  What STILL works:          What DOESN'T work:                       │
  │  ✅ git commit              ❌ git push                              │
  │  ✅ git branch              ❌ git pull (from GitHub)                │
  │  ✅ git log                 ❌ Pull Requests                         │
  │  ✅ git diff                ❌ Issues / Wiki                         │
  │  ✅ git stash               ❌ GitHub Actions                        │
  │  ✅ git merge               ❌ GitHub Pages                          │
  │  ✅ git rebase              ❌ Code review                           │
  │  ✅ git bisect                                                       │
  │  ✅ git blame                                                        │
  │  ✅ Full history access                                              │
  └──────────────────────────────────────────────────────────────────────┘
```

**Emergency workflow during outage:**
```bash
# 1. Keep working locally — commit as usual
git add . && git commit -m "feat: continue work during outage"

# 2. If you need to share code with a teammate:
# Option A: Push to a secondary remote
git remote add backup https://gitlab.com/org/repo.git   # pre-configured
git push backup main

# Option B: Create a bundle file and share via Slack/email
git bundle create emergency.bundle main
# Teammate receives the file:
git clone emergency.bundle repo-from-bundle
# OR merge into existing repo:
git pull emergency.bundle main

# Option C: Push peer-to-peer (if on same network)
# On your machine:
git daemon --reuseaddr --base-path=. --export-all .
# Teammate:
git pull git://your-ip/ main

# 3. When GitHub comes back:
git push origin main   # Push all accumulated commits
```

> **DevOps Pro Tip:** For critical infrastructure repos, maintain a **mirror** on a separate Git hosting service. Set up automatic mirroring: `git push --mirror backup-remote` as a cron job or post-receive hook. If your primary hosting goes down, switch CI/CD to the mirror.

---

### FAQ 15: "How do we implement zero-downtime deployments using Git?"

**Severity:** 🟢 Informational — architecture question

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │        ZERO-DOWNTIME DEPLOYMENT STRATEGIES WITH GIT                  │
  │                                                                      │
  │  Strategy 1: Blue-Green Deployment                                   │
  │  ─────────────────────────────────                                   │
  │                                                                      │
  │  main:  A ── B ── C(v1.2.0)── D ── E(v1.3.0)                        │
  │                    │                 │                                │
  │         ┌──────────┘                 └──────────┐                    │
  │         ▼                                       ▼                    │
  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │
  │  │  Blue (live)  │  │ Green (idle) │  │ Green (live) │               │
  │  │  running      │  │ deploy v1.3  │  │ switch       │               │
  │  │  v1.2.0       │  │ test         │  │ traffic      │               │
  │  └──────────────┘  └──────────────┘  └──────────────┘               │
  │       BEFORE            DURING            AFTER                      │
  │  Rollback: switch traffic back to Blue (still running v1.2.0)        │
  │                                                                      │
  │  Strategy 2: Canary Deployment                                       │
  │  ─────────────────────────────                                       │
  │                                                                      │
  │  Tag v1.3.0 pushed → CI deploys to canary (5% traffic)              │
  │                                                                      │
  │  ┌────────────────────────────────────────────┐                      │
  │  │  Traffic split:                             │                      │
  │  │  ██████████████████████████████████████ 95% │ v1.2.0 (stable)     │
  │  │  ██                                     5% │ v1.3.0 (canary)     │
  │  └────────────────────────────────────────────┘                      │
  │                                                                      │
  │  Monitor error rates, latency for 30 min                             │
  │  If OK → gradually increase: 25% → 50% → 100%                       │
  │  If BAD → route 100% back to v1.2.0 (instant rollback)              │
  │                                                                      │
  │  Strategy 3: GitOps (ArgoCD / Flux)                                  │
  │  ──────────────────────────────                                      │
  │                                                                      │
  │  Git repo IS the source of truth for cluster state:                  │
  │                                                                      │
  │  k8s-manifests repo:                                                 │
  │  └── apps/                                                           │
  │      └── auth-service/                                               │
  │          └── deployment.yaml  (image: auth:v1.2.0)                   │
  │                                                                      │
  │  Developer changes image tag → PR → merge → ArgoCD detects           │
  │  → Auto-applies to cluster → Rolling update (zero downtime)          │
  │                                                                      │
  │  Rollback: git revert the manifest change                            │
  │  → ArgoCD auto-reverts the cluster                                   │
  │  → Full audit trail in Git history                                   │
  └──────────────────────────────────────────────────────────────────────┘
```

---

### Quick Reference — Production Emergency Runbook

| Emergency | First Command | Full Fix |
|-----------|--------------|----------|
| Force push overwrote main | `git reflog show origin/main` | Restore with `git push --force-with-lease origin <old-sha>:main` |
| Secret committed | **Rotate credential immediately** | `git filter-repo` + re-clone all |
| Repo corrupted | `git fsck --full` | `git clone` from remote, or `git unpack-objects` from backup |
| Wrong commit deployed | `git revert <sha>` + push | Tag correct commit, redeploy |
| Merge broke production | `git revert -m 1 <merge-sha>` | Revert merge, release fix, then revert-the-revert |
| CI can't clone (timeout) | `git clone --depth 1` | Move to partial/sparse clone in CI config |
| Developer can't push | Check push rejection message | Fix signing/email/branch-name per hook requirements |
| Need to release without broken feature | `git revert -m 1 <feature-merge>` | Release, then revert-revert + fix later |
| Host down (GitHub/GitLab) | Work locally, `git bundle` to share | Push when service restored, consider secondary mirror |
| Migration from SVN | `git svn clone --stdlayout` | Author mapping + tag/branch conversion |

---

> **🎓 Final Note:** This handbook covered Git from first principles to production-grade operations. The best way to internalize these concepts is to **break things in a test repo** — force push, corrupt objects, create merge disasters, then recover. Muscle memory in disaster recovery is what separates a junior from a senior DevOps engineer. Keep this handbook as a reference, and revisit sections as you encounter real-world scenarios.
