# Linux — Level 2 (🟡 Medium)

## 1. SSH Key-Based Authentication

### How SSH Key Authentication Works

```
┌──────────┐                              ┌──────────────┐
│  CLIENT   │                              │    SERVER     │
│           │  1. Client connects          │              │
│ Private   │ ─────────────────────────▶  │ Public Key    │
│ Key 🔑   │                              │ (authorized)  │
│           │  2. Server sends challenge   │              │
│           │ ◀─────────────────────────  │              │
│           │                              │              │
│           │  3. Client signs with        │              │
│           │     private key              │              │
│           │ ─────────────────────────▶  │  4. Server    │
│           │                              │  verifies     │
│           │  5. Access granted ✅        │  with public  │
│           │ ◀─────────────────────────  │  key          │
└──────────┘                              └──────────────┘
```

**Key Principle:** The private key never leaves your machine. The server only has the public key.

### Generate a Key Pair

```bash
# Generate an Ed25519 key (recommended — fast and secure)
ssh-keygen -t ed25519 -C "your_email@example.com"

# Or generate an RSA key (wider compatibility, use 4096 bits)
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

When prompted:
- **File location:** Press Enter for default (`~/.ssh/id_ed25519`)
- **Passphrase:** Enter a strong passphrase (adds a second layer of security)

### View Your Keys

```bash
# List key files
ls -la ~/.ssh/

# View public key (this is what you share)
cat ~/.ssh/id_ed25519.pub

# View key fingerprint
ssh-keygen -lf ~/.ssh/id_ed25519.pub
```

### Copy Public Key to a Server

**Method 1: Using `ssh-copy-id` (easiest):**

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server_ip
```

**Method 2: Manual copy:**

```bash
# On the client — copy the public key content
cat ~/.ssh/id_ed25519.pub

# On the server — paste into authorized_keys
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
# Paste the public key, save, and exit
chmod 600 ~/.ssh/authorized_keys
```

**Method 3: One-liner:**

```bash
cat ~/.ssh/id_ed25519.pub | ssh user@server_ip "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

### Test Key-Based Login

```bash
# Connect using key authentication
ssh user@server_ip

# Specify a key explicitly
ssh -i ~/.ssh/id_ed25519 user@server_ip

# Verify — should connect without asking for a password
```

### Disable Password Authentication

> ⚠️ **Only do this AFTER confirming key-based login works!**

```bash
sudo nano /etc/ssh/sshd_config
```

Find and set these values:

```
PubkeyAuthentication yes
PasswordAuthentication no
ChallengeResponseAuthentication no
UsePAM no
```

```bash
# Test config before restarting
sudo sshd -t

# Restart SSH service
sudo systemctl restart sshd
```

### SSH Config File

Create `~/.ssh/config` to simplify connections:

```bash
nano ~/.ssh/config
```

```
# Default settings for all hosts
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    AddKeysToAgent yes

# Development server
Host dev
    HostName 192.168.1.100
    User devuser
    Port 22
    IdentityFile ~/.ssh/id_ed25519

# Production server
Host prod
    HostName 10.0.1.50
    User admin
    Port 2222
    IdentityFile ~/.ssh/id_rsa_prod

# Jump host / bastion
Host bastion
    HostName bastion.example.com
    User jumpuser
    IdentityFile ~/.ssh/id_ed25519

# Connect through bastion to internal server
Host internal
    HostName 10.0.0.5
    User appuser
    ProxyJump bastion
```

```bash
chmod 600 ~/.ssh/config

# Now you can simply type:
ssh dev
ssh prod
ssh internal   # Automatically goes through bastion
```

### SSH Agent

```bash
# Start the SSH agent
eval "$(ssh-agent -s)"

# Add your key (you'll enter the passphrase once)
ssh-add ~/.ssh/id_ed25519

# List loaded keys
ssh-add -l

# Remove all keys from agent
ssh-add -D
```

### Troubleshooting

```bash
# Verbose connection (shows auth details)
ssh -v user@server_ip

# Very verbose (more details)
ssh -vv user@server_ip

# Check permissions (common cause of failure)
# On the server:
ls -la ~/.ssh/
# Expected:
#   ~/.ssh/             → 700 (drwx------)
#   ~/.ssh/authorized_keys → 600 (-rw-------)

# Fix permissions
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chown -R $USER:$USER ~/.ssh

# Check SSH server logs
sudo tail -f /var/log/auth.log
sudo journalctl -u sshd -f
```

---

---

## 2. Firewall Configuration with UFW & iptables

### UFW vs iptables

| Feature | UFW | iptables |
|---|---|---|
| Complexity | Simple (beginner-friendly) | Complex (powerful) |
| Syntax | Human-readable | Technical |
| What it is | Frontend for iptables | Actual firewall tool |
| Use case | Most servers | Fine-grained control, legacy systems |
| Persistence | Built-in | Requires `iptables-persistent` |
| Logging | Easy to enable | Manual configuration |

---

### Part A: UFW (Uncomplicated Firewall)

#### Install and Initial Setup

```bash
# Install UFW (usually pre-installed on Ubuntu)
sudo apt install ufw -y

# Check status
sudo ufw status verbose
```

#### Set Default Policies

```bash
# Deny all incoming traffic by default
sudo ufw default deny incoming

# Allow all outgoing traffic by default
sudo ufw default allow outgoing
```

#### Allow Services

> ⚠️ **ALWAYS allow SSH before enabling UFW, or you'll lock yourself out!**

```bash
# Allow SSH (CRITICAL — do this first!)
sudo ufw allow ssh
# or
sudo ufw allow 22/tcp

# Allow HTTP and HTTPS
sudo ufw allow http
sudo ufw allow https
# or
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Allow a custom port
sudo ufw allow 3000/tcp

# Allow a port range
sudo ufw allow 8000:8100/tcp
```

#### Enable the Firewall

```bash
# Enable UFW
sudo ufw enable

# Verify status
sudo ufw status numbered
```

#### Advanced Rules

```bash
# Allow from a specific IP
sudo ufw allow from 192.168.1.100

# Allow from a specific IP to a specific port
sudo ufw allow from 192.168.1.100 to any port 22

# Allow from a subnet
sudo ufw allow from 10.0.0.0/24 to any port 3306

# Deny from a specific IP
sudo ufw deny from 203.0.113.50

# Rate limiting for SSH (blocks IPs with 6+ connections in 30 seconds)
sudo ufw limit ssh
```

#### Delete Rules

```bash
# View rules with numbers
sudo ufw status numbered

# Delete by number
sudo ufw delete 3

# Delete by rule specification
sudo ufw delete allow 8080/tcp
```

#### Application Profiles

```bash
# List available application profiles
sudo ufw app list

# View details of a profile
sudo ufw app info "Nginx Full"

# Allow an application profile
sudo ufw allow "Nginx Full"
```

```bash
# Disable UFW
sudo ufw disable

# Reset all rules
sudo ufw reset
```

---

### Part B: iptables

#### View Current Rules

```bash
# List all rules with line numbers
sudo iptables -L -n -v --line-numbers

# List NAT table
sudo iptables -t nat -L -n -v
```

#### Basic Rules

```bash
# Allow established and related connections
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Allow loopback (localhost) traffic
sudo iptables -A INPUT -i lo -j ACCEPT

# Allow SSH
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow HTTP and HTTPS
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Drop all other incoming traffic
sudo iptables -A INPUT -j DROP
```

**iptables syntax explained:**

| Component | Meaning |
|---|---|
| `-A INPUT` | Append to INPUT chain (incoming traffic) |
| `-A OUTPUT` | Append to OUTPUT chain (outgoing traffic) |
| `-A FORWARD` | Append to FORWARD chain (routed traffic) |
| `-p tcp` | Protocol: TCP |
| `-p udp` | Protocol: UDP |
| `--dport 22` | Destination port 22 |
| `--sport 80` | Source port 80 |
| `-s 192.168.1.0/24` | Source IP/subnet |
| `-d 10.0.0.5` | Destination IP |
| `-i eth0` | Input interface |
| `-o eth0` | Output interface |
| `-j ACCEPT` | Target: Accept the packet |
| `-j DROP` | Target: Silently drop the packet |
| `-j REJECT` | Target: Drop and send error back |
| `-m conntrack --ctstate` | Match connection state |

#### Save and Persist Rules

```bash
# Install iptables-persistent
sudo apt install iptables-persistent -y

# Save current rules
sudo netfilter-persistent save

# Rules are saved to:
#   /etc/iptables/rules.v4
#   /etc/iptables/rules.v6

# Reload rules
sudo netfilter-persistent reload
```

#### Cleanup

```bash
# Flush all rules
sudo iptables -F

# Delete all custom chains
sudo iptables -X

# Reset default policies to ACCEPT
sudo iptables -P INPUT ACCEPT
sudo iptables -P FORWARD ACCEPT
sudo iptables -P OUTPUT ACCEPT
```

---

---

## 3. Disk Management and Monitoring

### Check Disk Space

```bash
# Show disk usage for all mounted filesystems
df -h

# Show only specific filesystem types
df -hT -x tmpfs -x devtmpfs

# Show inode usage
df -i
```

### Find What's Using Space

```bash
# Check directory sizes (top-level summary)
du -sh /var/*

# Find the biggest directories
du -h --max-depth=1 / 2>/dev/null | sort -rh | head -20

# Find large files (>100MB)
sudo find / -type f -size +100M -exec ls -lh {} \; 2>/dev/null | sort -k5 -rh | head -20

# Find files modified in the last 7 days
find /var -type f -mtime -7 -size +10M -ls 2>/dev/null
```

### Clean Up Disk Space

```bash
# Clean apt cache
sudo apt clean
sudo apt autoremove -y

# Vacuum old journal logs (keep last 7 days)
sudo journalctl --vacuum-time=7d

# Or limit journal size to 500MB
sudo journalctl --vacuum-size=500M

# Find and remove old log files
sudo find /var/log -name "*.gz" -mtime +30 -delete
sudo find /var/log -name "*.log.*" -mtime +30 -delete

# Check for old kernels
dpkg --list | grep linux-image
# Remove old kernels (keep current)
sudo apt autoremove --purge -y
```

### List Block Devices

```bash
# Show all block devices in a tree
lsblk

# Show with filesystem info
lsblk -f

# Show detailed info
sudo fdisk -l
```

### Partition a New Disk

> **Scenario:** A new disk `/dev/sdb` has been attached and needs to be partitioned and mounted.

```bash
# 1. Identify the new disk
lsblk
sudo fdisk -l /dev/sdb

# 2. Create a partition
sudo fdisk /dev/sdb
```

In the `fdisk` interactive prompt:

```
n        # New partition
p        # Primary partition
1        # Partition number 1
[Enter]  # Default first sector
[Enter]  # Default last sector (use entire disk)
w        # Write changes and exit
```

```bash
# 3. Create a filesystem
sudo mkfs.ext4 /dev/sdb1

# 4. Create a mount point
sudo mkdir -p /mnt/data

# 5. Mount the partition
sudo mount /dev/sdb1 /mnt/data

# 6. Verify
df -h /mnt/data
lsblk
```

### Make the Mount Permanent

```bash
# Get the UUID of the partition
sudo blkid /dev/sdb1
# Example output: /dev/sdb1: UUID="a1b2c3d4-..." TYPE="ext4"

# Add to /etc/fstab
sudo nano /etc/fstab
```

Add this line:

```
UUID=a1b2c3d4-xxxx-xxxx-xxxx-xxxxxxxxxxxx  /mnt/data  ext4  defaults  0  2
```

```bash
# Test the fstab entry (mount all entries)
sudo mount -a

# Verify
df -h /mnt/data
```

### Monitor Disk I/O

```bash
# Install monitoring tools
sudo apt install iotop sysstat -y

# Real-time I/O monitoring (requires root)
sudo iotop

# Disk I/O statistics
iostat -x 1 5
# Shows stats every 1 second, 5 times

# Check filesystem health (unmount first!)
sudo umount /dev/sdb1
sudo fsck /dev/sdb1
sudo mount /dev/sdb1 /mnt/data
```

---

---

## 4. Process Management

### View Processes

```bash
# Show all processes (snapshot)
ps aux

# Explanation of columns:
# USER  PID  %CPU  %MEM  VSZ  RSS  TTY  STAT  START  TIME  COMMAND

# Show process tree
ps auxf

# Find a specific process
ps aux | grep nginx
pgrep -a nginx

# Show processes for a specific user
ps -u www-data
```

### Process States

| State | Code | Description |
|---|---|---|
| Running | `R` | Currently running or in the run queue |
| Sleeping | `S` | Waiting for an event (interruptible) |
| Disk Sleep | `D` | Waiting for I/O (uninterruptible) |
| Stopped | `T` | Stopped by a signal (e.g., Ctrl+Z) |
| Zombie | `Z` | Terminated but not reaped by parent |

### Foreground and Background Jobs

```bash
# Run a command in the background
sleep 300 &

# List background jobs
jobs

# Bring a job to the foreground
fg %1

# Suspend a foreground job (Ctrl+Z)
# Then resume it in the background
bg %1

# Run a command that survives logout
nohup long_running_command &
# Output goes to nohup.out

# Or use disown
long_running_command &
disown %1
```

### Signals

```bash
# List all available signals
kill -l

# Send SIGTERM (graceful shutdown — default)
kill PID
kill -15 PID
kill -SIGTERM PID

# Send SIGKILL (force kill — cannot be caught)
kill -9 PID
kill -SIGKILL PID

# Send SIGHUP (reload configuration)
kill -1 PID
kill -SIGHUP PID

# Send SIGSTOP (pause process)
kill -SIGSTOP PID

# Send SIGCONT (resume paused process)
kill -SIGCONT PID

# Kill all processes by name
killall nginx

# Kill processes matching a pattern
pkill -f "python3 app.py"
```

| Signal | Number | Default Action | Can Be Caught? |
|---|---|---|---|
| `SIGTERM` | 15 | Graceful termination | ✅ Yes |
| `SIGKILL` | 9 | Immediate termination | ❌ No |
| `SIGHUP` | 1 | Hangup / Reload config | ✅ Yes |
| `SIGSTOP` | 19 | Pause process | ❌ No |
| `SIGCONT` | 18 | Resume paused process | ✅ Yes |
| `SIGINT` | 2 | Interrupt (Ctrl+C) | ✅ Yes |

### Process Priority (nice / renice)

```bash
# Run a command with lower priority (higher nice value = lower priority)
# Nice values range from -20 (highest priority) to 19 (lowest)
nice -n 10 long_running_command

# Change the priority of a running process
renice -n 5 -p PID

# Set highest priority (requires root)
sudo renice -n -20 -p PID

# View nice values
ps -eo pid,ni,comm | head -20
```

### Interactive Tools

```bash
# top — built-in process viewer
top
# Inside top:
#   M  = sort by memory
#   P  = sort by CPU
#   k  = kill a process (enter PID)
#   q  = quit

# htop — interactive process viewer (recommended)
sudo apt install htop -y
htop
# Inside htop:
#   F3 = search
#   F5 = tree view
#   F6 = sort by column
#   F9 = kill process
#   F10 = quit
```

### Find Resource-Heavy Processes

```bash
# Top 10 by CPU usage
ps aux --sort=-%cpu | head -11

# Top 10 by memory usage
ps aux --sort=-%mem | head -11

# Find processes using the most memory (alternative)
ps -eo pid,ppid,%mem,%cpu,comm --sort=-%mem | head -11

# Watch in real time
watch -n 2 "ps aux --sort=-%cpu | head -11"
```

### Zombie Processes

```bash
# Find zombie processes
ps aux | awk '$8 ~ /Z/ {print}'

# Or
ps -eo pid,ppid,stat,comm | grep -w Z

# Find the parent of a zombie
ps -eo pid,ppid,stat,comm | grep -w Z
# Note the PPID (parent PID)
ps -p PPID -o pid,comm

# The proper fix is to fix the parent process
# As a last resort, kill the parent:
kill -SIGCHLD PPID
# or
kill PPID
```

---

---

## 5. Log Management with journalctl and Logrotate

### Part A: journalctl

#### Basic Log Viewing

```bash
# View all logs (oldest first)
sudo journalctl

# View logs in reverse (newest first)
sudo journalctl -r

# Follow logs in real time (like tail -f)
sudo journalctl -f

# Show last 50 lines
sudo journalctl -n 50

# Show logs from current boot only
sudo journalctl -b
```

#### Filter by Service

```bash
# View logs for a specific service
sudo journalctl -u nginx
sudo journalctl -u ssh

# Follow a specific service in real time
sudo journalctl -u nginx -f

# Combine multiple services
sudo journalctl -u nginx -u php-fpm
```

#### Filter by Time

```bash
# Logs since a specific time
sudo journalctl --since "2026-03-01 00:00:00"

# Logs in a time range
sudo journalctl --since "2026-03-01" --until "2026-03-02"

# Relative time
sudo journalctl --since "1 hour ago"
sudo journalctl --since "30 min ago"
sudo journalctl --since today
sudo journalctl --since yesterday --until today
```

#### Filter by Priority

```bash
# Show only errors and above
sudo journalctl -p err

# Show only warnings and above
sudo journalctl -p warning

# Priority levels:
#   0 = emerg
#   1 = alert
#   2 = crit
#   3 = err
#   4 = warning
#   5 = notice
#   6 = info
#   7 = debug
```

#### Output Formats

```bash
# JSON output (for parsing)
sudo journalctl -u nginx -o json-pretty -n 5

# Short format with timestamps
sudo journalctl -o short-iso

# Verbose (all fields)
sudo journalctl -u nginx -o verbose -n 1
```

#### Manage Disk Usage

```bash
# Check how much disk space journals use
sudo journalctl --disk-usage

# Remove old journals (keep last 7 days)
sudo journalctl --vacuum-time=7d

# Limit journal size to 500MB
sudo journalctl --vacuum-size=500M

# Set permanent limits
sudo nano /etc/systemd/journald.conf
```

```ini
[Journal]
SystemMaxUse=500M
SystemMaxFileSize=50M
MaxRetentionSec=1month
```

```bash
sudo systemctl restart systemd-journald
```

---

### Part B: Important Log Locations

| Log File | Purpose |
|---|---|
| `/var/log/syslog` | General system activity log |
| `/var/log/auth.log` | Authentication and authorization logs |
| `/var/log/kern.log` | Kernel messages |
| `/var/log/dpkg.log` | Package manager (dpkg) logs |
| `/var/log/apt/history.log` | Apt install/remove history |
| `/var/log/nginx/access.log` | Nginx access logs |
| `/var/log/nginx/error.log` | Nginx error logs |
| `/var/log/mysql/error.log` | MySQL error log |

```bash
# Quick way to watch multiple log files
sudo tail -f /var/log/syslog /var/log/auth.log
```

---

### Part C: Logrotate

#### Configure Rotation for a Custom Application

```bash
sudo nano /etc/logrotate.d/myapp
```

```
/var/log/myapp/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0640 www-data www-data
    sharedscripts
    postrotate
        systemctl reload myapp > /dev/null 2>&1 || true
    endscript
}
```

| Directive | Meaning |
|---|---|
| `daily` | Rotate logs every day |
| `rotate 14` | Keep 14 rotated log files |
| `compress` | Compress rotated files with gzip |
| `delaycompress` | Don't compress the most recent rotated file |
| `missingok` | Don't error if log file is missing |
| `notifempty` | Don't rotate if the file is empty |
| `create 0640 www-data www-data` | Create new file with these permissions and ownership |
| `sharedscripts` | Run postrotate only once (not per file) |
| `postrotate ... endscript` | Commands to run after rotation |

```bash
# Test the configuration (dry run)
sudo logrotate -d /etc/logrotate.d/myapp

# Force rotation (useful for testing)
sudo logrotate -f /etc/logrotate.d/myapp

# View logrotate status
cat /var/lib/logrotate/status
```

---

---

## 6. Swap Space Management

### What is Swap?

Swap is disk space used as "emergency" memory when RAM is full. It is much slower than RAM but prevents out-of-memory crashes.

| Scenario | Swap Helpful? |
|---|---|
| RAM is full, system starts killing processes | ✅ Yes — prevents OOM killer |
| Hibernation support | ✅ Yes — saves RAM state to disk |
| Running memory-heavy applications temporarily | ✅ Yes — provides breathing room |
| Consistently using more memory than available RAM | ⚠️ Band-aid — add more RAM |
| High-performance databases (Redis, etc.) | ❌ No — use RAM, disable swap |

### Check Current Swap

```bash
# View swap usage
free -h

# Detailed swap info
swapon --show

# Check if swap is in fstab
grep swap /etc/fstab
```

### Create a Swap File

```bash
# 1. Create a 2GB swap file
sudo fallocate -l 2G /swapfile

# 2. Set correct permissions (only root should access)
sudo chmod 600 /swapfile

# 3. Set up the swap area
sudo mkswap /swapfile

# 4. Enable the swap file
sudo swapon /swapfile

# 5. Verify
swapon --show
free -h
```

### Make Swap Permanent

```bash
# Add to /etc/fstab
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Verify
grep swap /etc/fstab
```

### Tune Swappiness

Swappiness controls how aggressively the kernel uses swap (0–100).

- **0:** Only swap when absolutely necessary (minimal swap usage)
- **60:** Default Ubuntu value
- **100:** Aggressively swap to disk

```bash
# Check current swappiness
cat /proc/sys/vm/swappiness

# Set temporarily (until reboot)
sudo sysctl vm.swappiness=10

# Set permanently
echo "vm.swappiness=10" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

> **Recommended:** For servers, set swappiness to **10**. For desktops, **60** (default) is fine.

### Resize Swap

```bash
# 1. Disable current swap
sudo swapoff /swapfile

# 2. Resize (e.g., to 4GB)
sudo fallocate -l 4G /swapfile

# 3. Recreate swap area
sudo mkswap /swapfile

# 4. Re-enable
sudo swapon /swapfile

# 5. Verify
free -h
```

### Remove Swap

```bash
# 1. Disable swap
sudo swapoff /swapfile

# 2. Remove from fstab
sudo sed -i '/\/swapfile/d' /etc/fstab

# 3. Delete the file
sudo rm /swapfile

# 4. Verify
free -h
swapon --show
```

---

---

## 7. Systemd Service + Nginx Reverse Proxy for Node.js

### What is systemd?

Systemd is the init system for modern Linux. It manages **services** (daemons) — starting them at boot, restarting on failure, and handling dependencies.

### What is a Reverse Proxy?

A reverse proxy sits between clients and your application, forwarding requests.

```
┌─────────┐        ┌────────────────┐        ┌──────────────┐
│  Client  │──:80──▶│  Nginx (proxy) │──:3000─▶│  Node.js App │
│ (browser)│◀───────│  Port 80/443   │◀────────│  Port 3000   │
└─────────┘        └────────────────┘        └──────────────┘
```

**Why use one?**
- SSL termination (HTTPS)
- Load balancing
- Static file serving
- Security (hide your app server)
- Caching

### Step 1: Install Node.js

```bash
# Install Node.js via NodeSource
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install nodejs -y

# Verify
node --version
npm --version
```

### Step 2: Create a Node.js App

```bash
# Create app directory
sudo mkdir -p /opt/nodeapp
sudo chown $USER:$USER /opt/nodeapp
cd /opt/nodeapp

# Initialize and create the app
npm init -y
```

Create the application file:

```bash
sudo nano /opt/nodeapp/app.js
```

```javascript
const http = require('http');
const os = require('os');

const PORT = process.env.PORT || 3000;

const server = http.createServer((req, res) => {
    const info = {
        message: 'Hello from Node.js!',
        hostname: os.hostname(),
        pid: process.pid,
        uptime: Math.floor(process.uptime()) + ' seconds',
        timestamp: new Date().toISOString(),
        url: req.url,
        method: req.method
    };

    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify(info, null, 2));
});

server.listen(PORT, () => {
    console.log(`Server running on port ${PORT} (PID: ${process.pid})`);
});
```

### Step 3: Test Manually

```bash
node /opt/nodeapp/app.js &
curl http://localhost:3000
kill %1
```

### Step 4: Create a Dedicated System User

```bash
# Create a system user for the app (no login, no home directory)
sudo useradd --system --no-create-home --shell /usr/sbin/nologin nodeapp

# Give ownership
sudo chown -R nodeapp:nodeapp /opt/nodeapp
```

### Step 5: Create the Systemd Service File

```bash
sudo nano /etc/systemd/system/nodeapp.service
```

```ini
[Unit]
# Description shown in systemctl status
Description=Node.js Application
# Documentation link
Documentation=https://example.com/docs

# Start after network is up
After=network.target

[Service]
# Type of service
Type=simple

# Run as dedicated user/group
User=nodeapp
Group=nodeapp

# Working directory
WorkingDirectory=/opt/nodeapp

# Environment variables
Environment=NODE_ENV=production
Environment=PORT=3000

# The command to start the service
ExecStart=/usr/bin/node /opt/nodeapp/app.js

# Restart policy
Restart=on-failure
RestartSec=5

# Limit restarts (5 attempts in 60 seconds)
StartLimitIntervalSec=60
StartLimitBurst=5

# Logging (stdout/stderr go to journald)
StandardOutput=journal
StandardError=journal
SyslogIdentifier=nodeapp

[Install]
# Start at boot when multi-user.target is reached
WantedBy=multi-user.target
```

**Restart policy options:**

| Value | Behavior |
|---|---|
| `no` | Never restart (default) |
| `on-failure` | Restart only on non-zero exit code |
| `on-abnormal` | Restart on signal, timeout, or watchdog |
| `on-success` | Restart only on clean exit (code 0) |
| `always` | Always restart regardless of exit status |

### Step 6: Enable and Start the Service

```bash
# Reload systemd to pick up the new service file
sudo systemctl daemon-reload

# Enable the service to start at boot
sudo systemctl enable nodeapp

# Start the service now
sudo systemctl start nodeapp

# Check status
sudo systemctl status nodeapp

# View logs
sudo journalctl -u nodeapp -f
```

### Step 7: Test Auto-Restart

```bash
# Find the current PID
sudo systemctl status nodeapp | grep "Main PID"

# Kill the process
sudo kill $(pgrep -f "node /opt/nodeapp/app.js")

# Wait a few seconds, then check — it should have a NEW PID
sleep 6
sudo systemctl status nodeapp | grep "Main PID"

# The service restarted automatically!
```

### Step 8: Install and Configure Nginx as Reverse Proxy

```bash
# Install Nginx
sudo apt install nginx -y

# Create the Nginx server block
sudo nano /etc/nginx/sites-available/nodeapp
```

```nginx
server {
    listen 80;
    server_name your_domain_or_ip;

    # Proxy all requests to the Node.js app
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;

        # WebSocket support
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';

        # Pass client info to the app
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;

        # Don't cache
        proxy_cache_bypass $http_upgrade;
    }

    # Serve static files directly (optional — if you have a public/ dir)
    location /static/ {
        alias /opt/nodeapp/public/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # Logs
    access_log /var/log/nginx/nodeapp_access.log;
    error_log /var/log/nginx/nodeapp_error.log;
}
```

### Step 9: Enable the Site and Test

```bash
# Enable the site
sudo ln -s /etc/nginx/sites-available/nodeapp /etc/nginx/sites-enabled/

# Remove default site (optional)
sudo rm -f /etc/nginx/sites-enabled/default

# Test Nginx configuration
sudo nginx -t

# Reload Nginx
sudo systemctl reload nginx

# Test!
curl http://localhost
```

### Useful Management Commands

```bash
# Service management
sudo systemctl start nodeapp
sudo systemctl stop nodeapp
sudo systemctl restart nodeapp
sudo systemctl reload nodeapp   # Only if the app supports it

# Status and logs
sudo systemctl status nodeapp
sudo journalctl -u nodeapp --since "10 min ago"
sudo journalctl -u nodeapp -f

# Nginx management
sudo systemctl reload nginx
sudo nginx -t
sudo tail -f /var/log/nginx/nodeapp_access.log
sudo tail -f /var/log/nginx/nodeapp_error.log
```

### Cleanup

```bash
# Stop and disable the service
sudo systemctl stop nodeapp
sudo systemctl disable nodeapp

# Remove files
sudo rm /etc/systemd/system/nodeapp.service
sudo systemctl daemon-reload

# Remove Nginx config
sudo rm /etc/nginx/sites-enabled/nodeapp
sudo rm /etc/nginx/sites-available/nodeapp
sudo systemctl reload nginx

# Remove app and user
sudo rm -rf /opt/nodeapp
sudo userdel nodeapp
```

---

*End of Medium Scenarios*
