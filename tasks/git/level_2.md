# Git — Level 2 (🟡 Medium)

## 1. Finding & Redacting Secrets from History

### Aim

Identify commits containing sensitive data and rewrite history safely to remove secrets from the repository.

### Why This Matters

If you accidentally commit an API key, password, or private key — just deleting it in the next commit is **NOT enough**. The secret is still in the Git history.

### Step-by-Step Implementation

#### Step 1 — Create a repo with "accidental" secrets

```bash
mkdir ~/git-secrets-lab && cd ~/git-secrets-lab
git init

echo "print('Hello World')" > app.py
git add app.py && git commit -m "Initial commit"

cat > config.py << 'EOF'
DATABASE_URL = "postgres://admin:SuperSecret123@db.example.com/mydb"
API_KEY = "sk-abc123def456ghi789"
AWS_SECRET_KEY = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
EOF
git add config.py && git commit -m "Add configuration"

echo "import config" >> app.py
git add app.py && git commit -m "Import config"

# "Fix" secrets (but they're still in history!)
cat > config.py << 'EOF'
import os
DATABASE_URL = os.environ.get("DATABASE_URL")
API_KEY = os.environ.get("API_KEY")
AWS_SECRET_KEY = os.environ.get("AWS_SECRET_KEY")
EOF
git add config.py && git commit -m "Remove hardcoded secrets"

echo "# My App" > README.md
git add README.md && git commit -m "Add README"
```

#### Step 2 — Find commits containing secrets

```bash
# Method 1: Pickaxe search
git log -S "sk-abc123def456ghi789" --oneline

# Method 2: Regex search
git log -G "API_KEY\s*=\s*\"[^\"]+\"" --oneline

# Method 3: Search all history
git grep "SuperSecret123" $(git rev-list --all)

# Method 4: Loop through all commits
for commit in $(git rev-list --all); do
    RESULT=$(git show "$commit" 2>/dev/null | grep -iE "(password|secret|api_key|token)\s*=\s*[\"'][^\"']+[\"']")
    if [ -n "$RESULT" ]; then
        echo "=== Commit: $(git log --oneline -1 $commit) ==="
        echo "$RESULT"
    fi
done
```

#### Step 3 — Remove secrets using git filter-repo

```bash
pip3 install git-filter-repo

cat > /tmp/replacements.txt << 'EOF'
SuperSecret123==>REDACTED
sk-abc123def456ghi789==>REDACTED
wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY==>REDACTED
EOF

git filter-repo --replace-text /tmp/replacements.txt

# Verify secrets are gone
git grep "SuperSecret123" $(git rev-list --all)
# Should return nothing!
```

#### Step 4 — Force push cleaned history

```bash
git push --force --all origin
git push --force --tags origin
```

#### Step 5 — Prevent this from happening again

```bash
echo "config.py" >> .gitignore
echo ".env" >> .gitignore
echo "*.key" >> .gitignore
echo "*.pem" >> .gitignore
git add .gitignore && git commit -m "Add gitignore to prevent secret leaks"
```

#### Cleanup

```bash
rm -rf ~/git-secrets-lab
```

---

---

## 2. Rearranging Commit History (Interactive Rebase)

### Aim

Use interactive rebase (`git rebase -i`) to squash, reorder, edit, or remove commits.

### Rebase Commands

| Command | Shortcut | What it does |
|---|---|---|
| `pick` | `p` | Keep the commit as-is |
| `reword` | `r` | Keep but change the message |
| `edit` | `e` | Pause to modify files |
| `squash` | `s` | Merge into previous (keep both messages) |
| `fixup` | `f` | Merge into previous (discard this message) |
| `drop` | `d` | Remove entirely |

### Step-by-Step Implementation

#### Step 1 — Create a repo with messy history

```bash
mkdir ~/git-rebase-lab && cd ~/git-rebase-lab
git init

echo "Line 1" > file.txt && git add file.txt && git commit -m "Add line 1"
echo "Line 2" >> file.txt && git add file.txt && git commit -m "Add line 2"
echo "Typo lien 3" >> file.txt && git add file.txt && git commit -m "Add lien 3 (typo)"
sed -i 's/lien 3/line 3/' file.txt && git add file.txt && git commit -m "Fix typo in line 3"
echo "Line 4" >> file.txt && git add file.txt && git commit -m "Add line 4"
echo "Debug: testing123" >> file.txt && git add file.txt && git commit -m "DEBUG - remove before merge"
echo "Line 5" >> file.txt && git add file.txt && git commit -m "Add line 5"
```

#### Step 2 — Interactive rebase

```bash
git rebase -i HEAD~6
```

Change to:

```
pick f2f2f2f Add line 2
pick e3e3e3e Add lien 3 (typo)
fixup d4d4d4d Fix typo in line 3        # ← merge typo fix into original
pick c5c5c5c Add line 4
drop b6b6b6b DEBUG - remove before merge # ← remove debug commit
reword a7a7a7a Add line 5               # ← change this message
```

#### Step 3 — If something goes wrong

```bash
git rebase --abort    # Reset to before rebase started
```

#### Cleanup

```bash
rm -rf ~/git-rebase-lab
```

---

---

## 3. Signed Commits

### Aim

Sign commits using GPG to verify commit authenticity and ensure trusted authorship.

### Step-by-Step Implementation

#### Step 1 — Install GPG and generate a key

```bash
sudo apt install gnupg -y
gpg --full-generate-key
# Choose: RSA and RSA, 4096 bits, 1y expiration
```

#### Step 2 — Find your key ID

```bash
gpg --list-secret-keys --keyid-format=long
# Your key ID is after rsa4096/ (e.g., ABC123DEF456GHI7)
```

#### Step 3 — Configure Git

```bash
git config --global user.signingkey ABC123DEF456GHI7
git config --global commit.gpgsign true
git config --global tag.gpgsign true
```

#### Step 4 — Add the GPG key to GitHub

```bash
gpg --armor --export ABC123DEF456GHI7
# Copy output → GitHub → Settings → SSH and GPG keys → New GPG key
```

#### Step 5 — Make a signed commit

```bash
mkdir ~/git-signed-lab && cd ~/git-signed-lab
git init
echo "Signed content" > file.txt
git add file.txt && git commit -S -m "My first signed commit"

# Verify the signature
git log --show-signature -1
```

#### Troubleshooting

```bash
export GPG_TTY=$(tty)
echo 'export GPG_TTY=$(tty)' >> ~/.bashrc
```

#### Cleanup

```bash
rm -rf ~/git-signed-lab
```

---

---

## 4. Sparse Checkout

### Aim

Clone or checkout only specific directories from a large repository.

### Why Use Sparse Checkout?

In a monorepo with 50 services, you only need `services/auth/`. Sparse checkout lets you download only the folders you need.

### Step-by-Step Implementation

```bash
# Create a demo monorepo
mkdir ~/sparse-lab && cd ~/sparse-lab
git init
mkdir -p services/auth services/api services/frontend docs shared/utils
echo "Auth service code" > services/auth/main.py
echo "API service code" > services/api/main.py
echo "Frontend code" > services/frontend/index.js
echo "Documentation" > docs/README.md
echo "Shared utilities" > shared/utils/helpers.py
echo "Root config" > config.yml
git add . && git commit -m "Initial monorepo structure"
git clone --bare ~/sparse-lab /tmp/sparse-remote.git

# Clone with sparse checkout
cd /tmp
git clone --no-checkout /tmp/sparse-remote.git sparse-clone
cd sparse-clone
git sparse-checkout init --cone
git sparse-checkout set services/auth
git checkout main

# Only services/auth/ is checked out!
ls -R

# Add more directories
git sparse-checkout add shared/utils

# View current rules
git sparse-checkout list

# Disable sparse checkout (get everything back)
git sparse-checkout disable
```

#### Cleanup

```bash
rm -rf ~/sparse-lab /tmp/sparse-remote.git /tmp/sparse-clone
```

---

---

## 5. Large Binary Files with Git LFS

### Aim

Use Git LFS (Large File Storage) to manage large binaries efficiently.

### Why Git LFS?

Git stores the **full content** of every file in every commit. Git LFS replaces large files with small **pointer files** and stores the actual content on a separate server.

### Step-by-Step Implementation

```bash
sudo apt install git-lfs -y

mkdir ~/git-lfs-lab && cd ~/git-lfs-lab
git init
git lfs install

# Track file patterns with LFS
git lfs track "*.zip"
git lfs track "*.mp4"
git lfs track "*.psd"

# Commit tracking rules FIRST
git add .gitattributes && git commit -m "Configure Git LFS tracking rules"

# Add a large file
dd if=/dev/zero of=demo-video.mp4 bs=1M count=10
git add demo-video.mp4 && git commit -m "Add demo video via LFS"

# Verify LFS is working
git lfs ls-files
git show HEAD:demo-video.mp4    # Shows a pointer, not binary data

# Clone without LFS files (useful for CI/CD)
GIT_LFS_SKIP_SMUDGE=1 git clone <repo-url> /tmp/lfs-clone-skip
cd /tmp/lfs-clone-skip
git lfs pull    # Download LFS files when needed

# Migrate existing large files to LFS
git lfs migrate import --include="*.mp4" --everything
git push --force --all origin
```

#### Cleanup

```bash
rm -rf ~/git-lfs-lab
```

---

---

## 6. Accidental Branch Deletion & Recovery

### Aim

Recover deleted branches using `git reflog`, commit hashes, or remote references.

### Step-by-Step Implementation

```bash
mkdir ~/git-recovery-lab && cd ~/git-recovery-lab
git init
echo "Main code" > main.py && git add main.py && git commit -m "Initial commit"

# Create a feature branch with work
git checkout -b feature/important-work
echo "Important feature" > feature.py && git add . && git commit -m "Add important feature"
echo "More work" >> feature.py && git add . && git commit -m "Extend important feature"
echo "Critical fix" >> feature.py && git add . && git commit -m "Critical fix in feature"

# Switch to main and "accidentally" delete the branch
git checkout main
git branch -D feature/important-work
# Output: Deleted branch feature/important-work (was abc1234).
```

#### Recovery Methods

**Method 1: Use the hash from the delete message**

```bash
git checkout -b feature/important-work abc1234
```

**Method 2: Use `git reflog`**

```bash
git reflog
# Find the last commit on the deleted branch
git checkout -b feature/important-work <hash-from-reflog>
```

**Method 3: Use `git fsck` (last resort)**

```bash
git fsck --lost-found
git log --oneline <dangling-commit-hash>
git checkout -b feature/important-work <dangling-commit-hash>
```

#### Preventive measure

```bash
git config --global alias.safe-delete '!f() { echo "About to delete: $1"; echo "Last commit: $(git log --oneline -1 $1)"; read -p "Sure? (y/n) " c; if [ "$c" = "y" ]; then git branch -D $1; fi; }; f'
```

#### Cleanup

```bash
rm -rf ~/git-recovery-lab
```

---

---

## 7. Git Submodules

### Aim

Include one Git repository inside another, useful for shared libraries or components.

### Step-by-Step Implementation

#### Step 1 — Create a "library" repo

```bash
mkdir ~/submodule-library && cd ~/submodule-library
git init
echo "def shared_func(): return 'from library'" > lib.py
git add lib.py && git commit -m "Initial library code"
git clone --bare ~/submodule-library /tmp/library-remote.git
cd ~/submodule-library
git remote add origin /tmp/library-remote.git
git push -u origin main
```

#### Step 2 — Add submodule to main project

```bash
mkdir ~/submodule-project && cd ~/submodule-project
git init
echo "Main project" > main.py && git add main.py && git commit -m "Initial project"

git submodule add /tmp/library-remote.git libs/shared-library
git add . && git commit -m "Add shared library as submodule"
```

#### Step 3 — Clone a project with submodules

```bash
# Method 1: Clone then init
git clone ~/submodule-project /tmp/project-clone
cd /tmp/project-clone
git submodule init && git submodule update

# Method 2: All at once (recommended)
git clone --recurse-submodules ~/submodule-project /tmp/project-clone-2
```

#### Step 4 — Update a submodule

```bash
cd ~/submodule-project/libs/shared-library
git fetch origin && git checkout main && git pull
cd ~/submodule-project
git add libs/shared-library && git commit -m "Update shared library"
```

#### Step 5 — Remove a submodule

```bash
git submodule deinit libs/shared-library
rm -rf .git/modules/libs/shared-library
git rm libs/shared-library
git commit -m "Remove shared library submodule"
```

#### Cleanup

```bash
rm -rf ~/submodule-library ~/submodule-project /tmp/library-remote.git /tmp/project-clone /tmp/project-clone-2
```
