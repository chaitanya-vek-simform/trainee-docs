# Linux — Level 1 (🟢 Easy)

## 1. User and Group Management

### Key Files

| File | Purpose |
|---|---|
| `/etc/passwd` | User account information (username, UID, GID, home dir, shell) |
| `/etc/shadow` | Encrypted user passwords and password aging info |
| `/etc/group` | Group definitions and membership |
| `/etc/sudoers` | Sudo access rules (always edit with `visudo`) |

### View Existing Users and Groups

```bash
# List all users (one per line in /etc/passwd)
cat /etc/passwd

# Show only usernames
cut -d: -f1 /etc/passwd

# List all groups
cat /etc/group

# Show only group names
cut -d: -f1 /etc/group

# See which groups the current user belongs to
groups

# See which groups a specific user belongs to
groups username

# Show detailed user info
id username
```

### Create a New User

```bash
# Basic user creation with home directory and default shell
sudo useradd -m -s /bin/bash johndoe

# Set password for the new user
sudo passwd johndoe
```

**Advanced user creation with specific options:**

```bash
# Create user with specific UID, comment, and additional groups
sudo useradd -m -s /bin/bash -u 1050 -c "John Doe - DevOps Trainee" -G sudo,docker johndoe
```

| Flag | Meaning |
|---|---|
| `-m` | Create home directory |
| `-s /bin/bash` | Set login shell |
| `-u 1050` | Set specific UID |
| `-c "..."` | Comment (usually full name / description) |
| `-G sudo,docker` | Add to supplementary groups |

**Create a system user (for running services):**

```bash
# System users typically have no home directory and no login shell
sudo useradd --system --no-create-home --shell /usr/sbin/nologin myappuser
```

### Modify an Existing User

```bash
# Change the login shell
sudo usermod -s /bin/zsh johndoe

# Add user to a group (IMPORTANT: always use -aG to APPEND)
sudo usermod -aG docker johndoe
# Without -a, the user will be REMOVED from all other supplementary groups!

# Lock a user account (prevents login)
sudo usermod -L johndoe

# Unlock a user account
sudo usermod -U johndoe

# Change the username
sudo usermod -l newname oldname

# Change the home directory (and move files)
sudo usermod -d /home/newname -m newname
```

### Create and Manage Groups

```bash
# Create a new group
sudo groupadd developers

# Create a group with a specific GID
sudo groupadd -g 2000 devops

# Add a user to a group
sudo usermod -aG developers johndoe

# Remove a user from a group
sudo gpasswd -d johndoe developers

# Rename a group
sudo groupmod -n newgroupname oldgroupname

# Delete a group
sudo groupdel developers
```

### Password Management

```bash
# Set/change password for a user
sudo passwd johndoe

# Force user to change password on next login
sudo passwd -e johndoe

# View password aging information
sudo chage -l johndoe

# Set password expiry to 90 days
sudo chage -M 90 johndoe

# Set minimum days between password changes
sudo chage -m 7 johndoe

# Set warning days before password expires
sudo chage -W 14 johndoe
```

### Configure Sudo Access

```bash
# ALWAYS edit sudoers with visudo (it checks syntax before saving)
sudo visudo
```

Add one of these lines at the end of the file:

```
# Give full sudo access to a user
johndoe ALL=(ALL:ALL) ALL

# Give sudo access without password prompt
johndoe ALL=(ALL) NOPASSWD: ALL

# Allow running only specific commands
johndoe ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx, /usr/bin/systemctl status nginx
```

**Using the sudoers.d directory (recommended):**

```bash
# Create a dedicated file for custom rules
sudo visudo -f /etc/sudoers.d/johndoe
```

Add:

```
johndoe ALL=(ALL) NOPASSWD: ALL
```

### Delete a User

```bash
# Delete user but keep home directory
sudo userdel johndoe

# Delete user AND home directory
sudo userdel -r johndoe

# Check for any remaining files owned by the deleted user
sudo find / -user 1050 -ls 2>/dev/null
```

### Practice Exercise

Create the following setup, then clean up:

```bash
# 1. Create a group
sudo groupadd trainees

# 2. Create a user in that group
sudo useradd -m -s /bin/bash -G trainees -c "Practice User" practiceuser
sudo passwd practiceuser

# 3. Verify
id practiceuser
groups practiceuser

# 4. Add to sudo group
sudo usermod -aG sudo practiceuser

# 5. Verify sudo access
su - practiceuser -c "sudo whoami"
```

**Cleanup:**

```bash
sudo userdel -r practiceuser
sudo groupdel trainees
```

---

---

## 2. File Permissions and Ownership

### Understanding the Permission System

When you run `ls -la`, here is what each part means:

```
-rwxr-xr-- 1 alice developers 4096 Mar  2 10:00 script.sh
│└┬┘└┬┘└┬┘ │  │      │          │         │         │
│ │   │   │  │  │      │          │         │         └── filename
│ │   │   │  │  │      │          │         └── modification date
│ │   │   │  │  │      │          └── file size (bytes)
│ │   │   │  │  │      └── group owner
│ │   │   │  │  └── user owner
│ │   │   │  └── number of hard links
│ │   │   └── other permissions (r--)
│ │   └── group permissions (r-x)
│ └── user/owner permissions (rwx)
└── file type (- = file, d = directory, l = symlink)
```

### Permission Values

| Symbol | Permission | On Files | On Directories | Octal |
|---|---|---|---|---|
| `r` | Read | View contents | List contents | 4 |
| `w` | Write | Modify contents | Create/delete files inside | 2 |
| `x` | Execute | Run as program | Enter the directory (`cd`) | 1 |
| `-` | No permission | — | — | 0 |

### Octal (Numeric) Notation

| Octal | Binary | Permission |
|---|---|---|
| 7 | 111 | rwx |
| 6 | 110 | rw- |
| 5 | 101 | r-x |
| 4 | 100 | r-- |
| 3 | 011 | -wx |
| 2 | 010 | -w- |
| 1 | 001 | --x |
| 0 | 000 | --- |

### Step-by-Step: Create Practice Files

```bash
# Create a practice directory
mkdir -p ~/permissions-lab && cd ~/permissions-lab

# Create some test files
touch file1.txt file2.txt script.sh
echo "#!/bin/bash" > script.sh
echo 'echo "Hello World"' >> script.sh
mkdir testdir

# View current permissions
ls -la
```

### Change Permissions with `chmod` (Symbolic)

```bash
# Add execute permission for the owner
chmod u+x script.sh

# Add read and write for group
chmod g+rw file1.txt

# Remove write for others
chmod o-w file2.txt

# Set exact permissions for user, group, and others
chmod u=rwx,g=rx,o=r script.sh

# Add execute for everyone
chmod a+x script.sh

# Remove all permissions for others
chmod o= file1.txt
```

**Symbolic notation reference:**

| Symbol | Meaning |
|---|---|
| `u` | User (owner) |
| `g` | Group |
| `o` | Others |
| `a` | All (u + g + o) |
| `+` | Add permission |
| `-` | Remove permission |
| `=` | Set exact permission |

### Change Permissions with `chmod` (Numeric)

```bash
# Owner: rwx, Group: r-x, Others: r-- (755)
chmod 755 script.sh

# Owner: rw-, Group: rw-, Others: --- (660)
chmod 660 file1.txt

# Owner: rw-, Group: r--, Others: r-- (644)
chmod 644 file2.txt

# Apply permissions recursively to a directory
chmod -R 755 testdir/
```

**Common permission patterns:**

| Octal | Meaning | Use Case |
|---|---|---|
| `755` | rwxr-xr-x | Executable scripts, directories |
| `644` | rw-r--r-- | Regular files, config files |
| `700` | rwx------ | Private scripts, private directories |
| `600` | rw------- | SSH keys, sensitive config files |
| `660` | rw-rw---- | Shared files within a group |
| `777` | rwxrwxrwx | ⚠️ **Avoid** — everyone can do everything |

### Change Ownership with `chown`

```bash
# Change the owner of a file
sudo chown alice file1.txt

# Change the owner and group
sudo chown alice:developers file1.txt

# Change only the group
sudo chown :developers file2.txt
# or use chgrp
sudo chgrp developers file2.txt

# Recursive ownership change
sudo chown -R alice:developers testdir/
```

### Special Permissions

| Permission | Octal | Effect on Files | Effect on Directories |
|---|---|---|---|
| **SetUID** | 4 | File runs as the file **owner** (e.g., `/usr/bin/passwd`) | No effect |
| **SetGID** | 2 | File runs as the file **group** | New files inherit the directory's group |
| **Sticky Bit** | 1 | No effect | Only file owner can delete their files (e.g., `/tmp`) |

```bash
# Set SetUID (notice the 4 prepended)
sudo chmod 4755 script.sh
ls -la script.sh
# -rwsr-xr-x  (note the 's' in owner execute position)

# Set SetGID on a directory
sudo chmod 2775 testdir/
ls -ld testdir/
# drwxrwsr-x  (note the 's' in group execute position)

# Set Sticky Bit
sudo chmod 1777 testdir/
ls -ld testdir/
# drwxrwxrwt  (note the 't' in others execute position)
```

### Understanding and Configuring `umask`

The `umask` determines the **default permissions** for newly created files and directories.

```
Default file permissions:    666 (rw-rw-rw-)
Default directory permissions: 777 (rwxrwxrwx)

If umask = 022:
  Files:      666 - 022 = 644 (rw-r--r--)
  Directories: 777 - 022 = 755 (rwxr-xr-x)
```

```bash
# View current umask
umask

# View in symbolic form
umask -S

# Set umask for current session
umask 027
# Files: 640 (rw-r-----), Directories: 750 (rwxr-x---)

# Make permanent — add to ~/.bashrc
echo "umask 027" >> ~/.bashrc
```

### Find Files with Specific Permissions

```bash
# Find all files with 777 permissions (potential security risk)
sudo find / -type f -perm 0777 -ls 2>/dev/null

# Find SetUID files
sudo find / -type f -perm -4000 -ls 2>/dev/null

# Find SetGID files
sudo find / -type f -perm -2000 -ls 2>/dev/null

# Find world-writable files
sudo find / -type f -perm -0002 -ls 2>/dev/null

# Find files NOT owned by any user
sudo find / -nouser -ls 2>/dev/null
```

### Cleanup

```bash
cd ~
rm -rf ~/permissions-lab
```

---

---

## 3. Package Management (apt)

### Update and Upgrade

```bash
# Update the package list (fetches latest info from repositories)
sudo apt update

# Upgrade all installed packages
sudo apt upgrade -y

# Full upgrade (handles dependencies that require removing packages)
sudo apt full-upgrade -y
```

### Search for Packages

```bash
# Search by name or description
apt search nginx

# Show detailed info about a package
apt show nginx

# List installed packages
apt list --installed

# Check if a specific package is installed
apt list --installed | grep nginx
dpkg -l | grep nginx
```

### Install Packages

```bash
# Install a single package
sudo apt install nginx -y

# Install multiple packages at once
sudo apt install git curl wget vim -y

# Install a specific version
sudo apt install nginx=1.18.0-0ubuntu1 -y

# List available versions
apt list -a nginx

# Install with minimal recommended packages
sudo apt install --no-install-recommends package_name -y

# Install a .deb file directly
sudo dpkg -i package_file.deb

# Fix any broken dependencies after dpkg
sudo apt install -f
```

### Remove Packages

```bash
# Remove a package (keeps config files)
sudo apt remove nginx -y

# Remove a package AND its config files
sudo apt purge nginx -y

# Remove unused dependency packages
sudo apt autoremove -y

# Clean the local package cache
sudo apt clean

# Remove only outdated cached packages
sudo apt autoclean
```

### Manage Repositories

**Add a PPA (Personal Package Archive):**

```bash
# Add a PPA
sudo add-apt-repository ppa:ondrej/php -y
sudo apt update

# Remove a PPA
sudo add-apt-repository --remove ppa:ondrej/php -y
sudo apt update
```

**Add a custom repository (example: Docker):**

```bash
# Install prerequisites
sudo apt install ca-certificates curl gnupg -y

# Add Docker's official GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update and install
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io -y
```

### Hold / Unhold Package Versions

```bash
# Prevent a package from being upgraded
sudo apt-mark hold nginx

# View held packages
apt-mark showhold

# Allow upgrades again
sudo apt-mark unhold nginx
```

### View Package History

```bash
# View apt history log
cat /var/log/apt/history.log

# View recent installs
grep " install " /var/log/apt/history.log

# View with timestamps
grep -E "^(Start-Date|Commandline)" /var/log/apt/history.log | tail -20
```

---

---

## 4. Shell Scripting — Practical Basics

### Your First Script

```bash
# Create a new script file
nano ~/hello.sh
```

```bash
#!/bin/bash
# This is a comment — shebang line above tells the system to use bash

# Variables
NAME="World"

# Output
echo "Hello, $NAME!"
echo "Today is $(date)"
echo "You are logged in as: $(whoami)"
```

```bash
# Make it executable and run
chmod +x ~/hello.sh
~/hello.sh
```

### Variables and User Input

```bash
#!/bin/bash

# Variable assignment (NO spaces around =)
GREETING="Hello"
NAME="DevOps Trainee"

echo "$GREETING, $NAME!"

# Read user input
echo -n "Enter your name: "
read USER_NAME
echo "Welcome, $USER_NAME!"

# Command substitution
CURRENT_DATE=$(date +%Y-%m-%d)
HOSTNAME=$(hostname)
echo "Date: $CURRENT_DATE, Host: $HOSTNAME"

# Arithmetic
NUM1=10
NUM2=3
SUM=$((NUM1 + NUM2))
echo "$NUM1 + $NUM2 = $SUM"
```

**Special variables:**

| Variable | Meaning |
|---|---|
| `$0` | Name of the script |
| `$1`, `$2`, … | Positional parameters (arguments) |
| `$#` | Number of arguments |
| `$@` | All arguments (as separate words) |
| `$?` | Exit status of the last command (0 = success) |

```bash
#!/bin/bash
echo "Script name: $0"
echo "First arg: $1"
echo "Second arg: $2"
echo "Total args: $#"
echo "All args: $@"
```

### Conditionals

```bash
#!/bin/bash

# File tests
FILE="/etc/passwd"

if [ -f "$FILE" ]; then
    echo "$FILE exists and is a regular file."
fi

if [ -d "/tmp" ]; then
    echo "/tmp is a directory."
fi

# String comparison
read -p "Enter environment (dev/prod): " ENV

if [ "$ENV" = "prod" ]; then
    echo "⚠️  You are in PRODUCTION!"
elif [ "$ENV" = "dev" ]; then
    echo "You are in development."
else
    echo "Unknown environment: $ENV"
fi

# Numeric comparison
DISK_USAGE=85

if [ "$DISK_USAGE" -gt 90 ]; then
    echo "CRITICAL: Disk usage above 90%!"
elif [ "$DISK_USAGE" -gt 70 ]; then
    echo "WARNING: Disk usage above 70%."
else
    echo "OK: Disk usage is normal."
fi
```

**Common test operators:**

| Operator | Meaning |
|---|---|
| `-f file` | File exists and is regular |
| `-d dir` | Directory exists |
| `-e path` | Path exists (file or dir) |
| `-r file` | File is readable |
| `-w file` | File is writable |
| `-x file` | File is executable |
| `-s file` | File exists and is not empty |
| `-z string` | String is empty |
| `-n string` | String is not empty |
| `str1 = str2` | Strings are equal |
| `str1 != str2` | Strings are not equal |
| `-eq` | Numerically equal |
| `-ne` | Not equal |
| `-gt` | Greater than |
| `-ge` | Greater than or equal |
| `-lt` | Less than |
| `-le` | Less than or equal |

### Loops

```bash
#!/bin/bash

# For loop — iterate over a list
for FRUIT in apple banana cherry; do
    echo "I like $FRUIT"
done

# For loop — C-style
for ((i = 1; i <= 5; i++)); do
    echo "Count: $i"
done

# While loop — read a file line by line
while IFS= read -r LINE; do
    echo "Line: $LINE"
done < /etc/hostname

# While loop — wait for a condition
echo "Waiting for /tmp/ready file..."
while [ ! -f /tmp/ready ]; do
    echo "Still waiting..."
    sleep 2
done
echo "File found!"
```

### Functions

```bash
#!/bin/bash

# Function with local variables
greet() {
    local NAME="$1"
    local TIME
    TIME=$(date +%H)

    if [ "$TIME" -lt 12 ]; then
        echo "Good morning, $NAME!"
    elif [ "$TIME" -lt 18 ]; then
        echo "Good afternoon, $NAME!"
    else
        echo "Good evening, $NAME!"
    fi
}

# Call the function
greet "Alice"
greet "Bob"

# Function with return value
is_root() {
    if [ "$(id -u)" -eq 0 ]; then
        return 0  # success / true
    else
        return 1  # failure / false
    fi
}

if is_root; then
    echo "Running as root."
else
    echo "Not running as root."
fi
```

### Error Handling

```bash
#!/bin/bash

# Strict mode
set -euo pipefail
# -e  : Exit on any error
# -u  : Treat unset variables as errors
# -o pipefail : Catch errors in piped commands

# Trap — run cleanup on exit or error
cleanup() {
    echo "Cleaning up temporary files..."
    rm -f /tmp/my_temp_file
}
trap cleanup EXIT

# Log function
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"
}

log "Script started"

# Check if running as root
if [ "$(id -u)" -ne 0 ]; then
    log "ERROR: This script must be run as root."
    exit 1
fi

log "Running as root — proceeding..."
log "Script completed successfully."
```

### Cleanup

```bash
rm -f ~/hello.sh
```

---

---

## 5. Cron Jobs and Scheduled Tasks

### Cron Syntax

```
┌───────────── minute (0–59)
│ ┌───────────── hour (0–23)
│ │ ┌───────────── day of month (1–31)
│ │ │ ┌───────────── month (1–12)
│ │ │ │ ┌───────────── day of week (0–7, 0 and 7 = Sunday)
│ │ │ │ │
* * * * *  command_to_run
```

### Common Schedule Examples

| Schedule | Expression | Description |
|---|---|---|
| Every minute | `* * * * *` | Runs every single minute |
| Every 5 minutes | `*/5 * * * *` | Runs every 5 minutes |
| Every hour | `0 * * * *` | At minute 0 of every hour |
| Every day at midnight | `0 0 * * *` | At 00:00 every day |
| Every day at 2:30 AM | `30 2 * * *` | At 02:30 every day |
| Every Monday at 9 AM | `0 9 * * 1` | At 09:00 every Monday |
| First day of month | `0 0 1 * *` | At midnight on the 1st |
| Every weekday at 8 AM | `0 8 * * 1-5` | Mon–Fri at 08:00 |
| Every 15 min, business hours | `*/15 8-17 * * 1-5` | Every 15 min, 8 AM–5 PM, Mon–Fri |

### View and Edit Crontab

```bash
# View your current crontab
crontab -l

# Edit your crontab
crontab -e

# View another user's crontab (requires root)
sudo crontab -u username -l
```

### Create a Simple Cron Job

```bash
# Open the crontab editor
crontab -e
```

Add the following line to run a script every day at 3 AM:

```
0 3 * * * /home/johndoe/scripts/daily-task.sh >> /home/johndoe/logs/daily-task.log 2>&1
```

### Practical: Backup Cron Job

Create a backup script:

```bash
mkdir -p ~/scripts ~/logs ~/backups

cat > ~/scripts/backup.sh << 'EOF'
#!/bin/bash
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="$HOME/backups"
SOURCE_DIR="$HOME/Documents"

tar -czf "$BACKUP_DIR/documents_$TIMESTAMP.tar.gz" "$SOURCE_DIR" 2>/dev/null

# Keep only last 7 backups
ls -t "$BACKUP_DIR"/documents_*.tar.gz | tail -n +8 | xargs -r rm

echo "$(date): Backup completed" >> "$HOME/logs/backup.log"
EOF

chmod +x ~/scripts/backup.sh
```

Schedule it:

```bash
crontab -e
```

```
# Daily backup at 2 AM
0 2 * * * /home/johndoe/scripts/backup.sh
```

### System-Wide Cron Jobs

```bash
# System crontab (has an extra field for the user)
sudo cat /etc/crontab

# Drop-in directory for system cron jobs
ls /etc/cron.d/

# Pre-defined schedule directories
ls /etc/cron.daily/
ls /etc/cron.hourly/
ls /etc/cron.weekly/
ls /etc/cron.monthly/
```

To add a system-wide daily job:

```bash
sudo nano /etc/cron.d/my-daily-cleanup
```

```
# Run cleanup as root every day at 4 AM
0 4 * * * root /usr/local/bin/cleanup.sh >> /var/log/cleanup.log 2>&1
```

### Environment Gotchas

Cron runs with a minimal environment. **Always use full paths:**

```
# ❌ Bad — cron may not find the command
0 * * * * node /opt/app/script.js

# ✅ Good — use full paths
0 * * * * /usr/bin/node /opt/app/script.js

# Find the full path of a command
which node      # /usr/bin/node
which python3   # /usr/bin/python3
```

### Redirect Cron Output

```bash
# Send output to a log file
* * * * * /path/to/script.sh >> /var/log/myjob.log 2>&1

# Discard all output (silent)
* * * * * /path/to/script.sh > /dev/null 2>&1

# Send output via email (requires mail configured)
MAILTO="admin@example.com"
0 6 * * * /path/to/report.sh
```

### Monitor Cron

```bash
# Check if cron service is running
sudo systemctl status cron

# View cron logs
grep CRON /var/log/syslog | tail -20

# Or with journalctl
sudo journalctl -u cron --since "1 hour ago"
```

---

---

## 6. Environment Variables and Profile Configuration

### View Environment Variables

```bash
# Show all environment variables
env

# Show all variables (including local shell variables)
set

# Show a specific variable
echo $HOME
echo $PATH
echo $USER
echo $SHELL

# Print all environment variables in a formatted way
printenv | sort
```

### Set Temporary Variables

```bash
# Set a variable for the current session only
export MY_VAR="hello"
echo $MY_VAR

# Set a variable for a single command
MY_VAR="world" bash -c 'echo $MY_VAR'

# Unset a variable
unset MY_VAR
```

### Profile Files Explained

**Login shell** (when you SSH in or log in at a terminal):

```
/etc/profile → /etc/profile.d/*.sh → ~/.bash_profile → ~/.bash_login → ~/.profile
```

**Non-login interactive shell** (when you open a new terminal window):

```
~/.bashrc
```

**Which file runs when?**

| Scenario | Files Sourced |
|---|---|
| SSH login | `/etc/profile` → `~/.profile` → `~/.bashrc` (if called from `.profile`) |
| Open terminal in GUI | `~/.bashrc` |
| Run a script | None (non-interactive) — unless script sources them |
| `su - username` | Login shell: `/etc/profile` → `~/.profile` |
| `su username` | Non-login shell: `~/.bashrc` |

### Set Permanent Variables

**For the current user — `~/.bashrc`:**

```bash
echo 'export EDITOR="vim"' >> ~/.bashrc
echo 'export MY_APP_ENV="development"' >> ~/.bashrc

# Apply changes immediately
source ~/.bashrc
```

**For ALL users — `/etc/environment`:**

```bash
sudo nano /etc/environment
```

```
# Add key=value pairs (no 'export' keyword needed)
JAVA_HOME="/usr/lib/jvm/java-17-openjdk-amd64"
MY_GLOBAL_VAR="shared_value"
```

**For ALL users with a script — `/etc/profile.d/`:**

```bash
sudo nano /etc/profile.d/custom-env.sh
```

```bash
#!/bin/bash
export COMPANY_NAME="MyCompany"
export LOG_LEVEL="info"
```

```bash
sudo chmod +x /etc/profile.d/custom-env.sh
```

### Customize PATH

```bash
# Add a directory to PATH (for current user)
echo 'export PATH="$HOME/bin:$HOME/.local/bin:$PATH"' >> ~/.bashrc

# Add for all users
echo 'export PATH="/opt/myapp/bin:$PATH"' | sudo tee /etc/profile.d/myapp-path.sh
sudo chmod +x /etc/profile.d/myapp-path.sh

# Verify
source ~/.bashrc
echo $PATH
```

### Customize PS1 Prompt

```bash
# Simple custom prompt: username@host:directory$
export PS1="\u@\h:\w\$ "

# Colored prompt
export PS1="\[\033[32m\]\u@\h\[\033[00m\]:\[\033[34m\]\w\[\033[00m\]\$ "

# Prompt with git branch (add to ~/.bashrc)
parse_git_branch() {
    git branch 2>/dev/null | sed -e '/^[^*]/d' -e 's/* \(.*\)/ (\1)/'
}
export PS1="\u@\h:\w\$(parse_git_branch)\$ "
```

| Escape | Meaning |
|---|---|
| `\u` | Username |
| `\h` | Hostname (short) |
| `\H` | Hostname (full) |
| `\w` | Current working directory |
| `\W` | Basename of current directory |
| `\d` | Date |
| `\t` | Time (24hr) |
| `\$` | `$` for normal user, `#` for root |

### Useful Aliases

Add these to `~/.bashrc`:

```bash
# Navigation
alias ..="cd .."
alias ...="cd ../.."
alias ll="ls -alh"
alias la="ls -A"

# Safety
alias rm="rm -i"
alias cp="cp -i"
alias mv="mv -i"

# Shortcuts
alias update="sudo apt update && sudo apt upgrade -y"
alias ports="ss -tlnp"
alias myip="curl -s ifconfig.me"
alias meminfo="free -h"
alias diskinfo="df -h"

# Git
alias gs="git status"
alias gl="git log --oneline -10"
alias gp="git pull"
```

```bash
# Apply changes
source ~/.bashrc
```

---

---

## 7. Archiving, Compression, and File Transfer

### tar (Tape Archive)

```bash
# Create an archive
tar -cvf archive.tar /path/to/directory/
# c = create, v = verbose, f = filename

# Create a compressed archive (gzip)
tar -czvf archive.tar.gz /path/to/directory/
# z = gzip compression

# Create a compressed archive (bzip2 — slower but better compression)
tar -cjvf archive.tar.bz2 /path/to/directory/

# Extract an archive
tar -xvf archive.tar
# x = extract

# Extract a gzip archive
tar -xzvf archive.tar.gz

# Extract to a specific directory
tar -xzvf archive.tar.gz -C /path/to/destination/

# List contents without extracting
tar -tzvf archive.tar.gz

# Extract a single file from an archive
tar -xzvf archive.tar.gz path/to/specific/file.txt
```

### gzip / gunzip

```bash
# Compress a file (replaces original with .gz)
gzip file.txt

# Decompress
gunzip file.txt.gz

# Keep the original file
gzip -k file.txt

# Compress with best compression level (1–9, default 6)
gzip -9 file.txt

# View compressed file without decompressing
zcat file.txt.gz
zless file.txt.gz
```

### zip / unzip

```bash
# Install if needed
sudo apt install zip unzip -y

# Create a zip file
zip archive.zip file1.txt file2.txt

# Zip a directory recursively
zip -r archive.zip /path/to/directory/

# Extract a zip file
unzip archive.zip

# Extract to a specific directory
unzip archive.zip -d /path/to/destination/

# List contents
unzip -l archive.zip
```

### scp (Secure Copy)

```bash
# Copy file TO a remote server
scp file.txt user@remote_host:/path/to/destination/

# Copy file FROM a remote server
scp user@remote_host:/path/to/file.txt /local/destination/

# Copy a directory recursively
scp -r /local/directory/ user@remote_host:/remote/destination/

# Use a specific port
scp -P 2222 file.txt user@remote_host:/path/to/destination/

# Use a specific SSH key
scp -i ~/.ssh/mykey file.txt user@remote_host:/path/to/destination/
```

### rsync (Remote Sync)

```bash
# Sync a local directory to another local directory
rsync -av /source/directory/ /destination/directory/
# a = archive mode (preserves permissions, timestamps, etc.)
# v = verbose

# Sync to a remote server
rsync -av /local/directory/ user@remote_host:/remote/directory/

# Sync from a remote server
rsync -av user@remote_host:/remote/directory/ /local/directory/

# Dry run (show what WOULD happen without making changes)
rsync -av --dry-run /source/ /destination/

# Delete files in destination that don't exist in source (mirror)
rsync -av --delete /source/ /destination/

# Exclude files or directories
rsync -av --exclude='*.log' --exclude='node_modules/' /source/ /destination/

# Show progress
rsync -av --progress /source/ /destination/

# Resume a partially transferred file
rsync -av --partial --progress /source/largefile.iso user@remote:/destination/
```

### rsync vs scp — Comparison

| Feature | `rsync` | `scp` |
|---|---|---|
| Transfers only differences | ✅ Yes | ❌ No (copies everything) |
| Resume interrupted transfer | ✅ Yes (`--partial`) | ❌ No |
| Delete extra files at destination | ✅ Yes (`--delete`) | ❌ No |
| Exclude patterns | ✅ Yes (`--exclude`) | ❌ No |
| Preserves permissions/timestamps | ✅ Yes (`-a`) | ✅ Yes (`-p`) |
| Compression during transfer | ✅ Yes (`-z`) | ✅ Yes (`-C`) |
| Simple one-time copy | ✅ Works | ✅ Simpler syntax |
| Ideal for | Backups, syncing, large transfers | Quick one-off copies |

> **Rule of thumb:** Use `scp` for quick, one-time copies. Use `rsync` for everything else — especially backups, syncing, and large or recurring transfers.

---

*End of Easy Scenarios*
