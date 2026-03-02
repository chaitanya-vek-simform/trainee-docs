# Linux — Level 3 (🔴 Hard)

## 1. Network Troubleshooting

### Check Network Interfaces

```bash
# Show all interfaces with IP addresses
ip addr show

# Short form
ip a

# Show only IPv4 addresses
ip -4 addr show

# Show a specific interface
ip addr show eth0

# Check interface status (up/down)
ip link show
```

### Ping — Basic Connectivity Test

```bash
# Ping a host (Ctrl+C to stop)
ping google.com

# Ping with a count limit
ping -c 4 google.com

# Ping with a specific interval
ping -i 0.5 -c 10 google.com

# Ping a local gateway
ip route | grep default
ping -c 4 192.168.1.1
```

### Traceroute — Trace the Path

```bash
# Install traceroute
sudo apt install traceroute -y

# Trace the route to a host
traceroute google.com

# Use tracepath (no root needed)
tracepath google.com

# Install and use mtr (combines ping + traceroute in real time)
sudo apt install mtr -y
mtr google.com

# mtr in report mode
mtr -r -c 10 google.com
```

### DNS Troubleshooting

```bash
# Simple DNS lookup
nslookup google.com

# Use a specific DNS server
nslookup google.com 8.8.8.8

# Detailed DNS lookup with dig
sudo apt install dnsutils -y

# Query A record (IPv4 address)
dig google.com

# Query specific record types
dig google.com A          # IPv4 address
dig google.com AAAA       # IPv6 address
dig google.com MX         # Mail servers
dig google.com NS         # Name servers
dig google.com TXT        # Text records (SPF, DKIM, etc.)
dig google.com CNAME      # Canonical name
dig google.com SOA        # Start of Authority

# Short answer
dig +short google.com

# Trace the full DNS resolution path
dig +trace google.com

# Reverse DNS lookup
dig -x 8.8.8.8

# Check current DNS configuration
cat /etc/resolv.conf

# Check systemd-resolved status
resolvectl status

# Flush DNS cache
sudo resolvectl flush-caches
# or
sudo systemd-resolve --flush-caches
```

### Check Open Ports

```bash
# Show listening TCP ports with process info
sudo ss -tlnp

# Show listening UDP ports
sudo ss -ulnp

# Show all connections (TCP)
ss -tan

# Filter by port
ss -tlnp | grep :80

# Show socket statistics
ss -s
```

### Test Remote Ports

```bash
# Test if a remote port is open with nc (netcat)
nc -zv google.com 443
nc -zv 192.168.1.100 22

# Test with curl (HTTP/HTTPS)
curl -v telnet://192.168.1.100:22
curl -I https://google.com

# Test with timeout
timeout 5 bash -c 'echo > /dev/tcp/google.com/443 && echo "Port open" || echo "Port closed"'

# Use telnet (if installed)
telnet google.com 80
```

### View and Manage Routing

```bash
# Show routing table
ip route show

# Show route to a specific destination
ip route get 8.8.8.8

# Add a static route
sudo ip route add 10.10.0.0/24 via 192.168.1.1 dev eth0

# Delete a route
sudo ip route del 10.10.0.0/24

# Add a default gateway
sudo ip route add default via 192.168.1.1

# Make routes permanent (Netplan on Ubuntu 22.04)
sudo nano /etc/netplan/01-netcfg.yaml
```

```yaml
network:
  version: 2
  ethernets:
    eth0:
      routes:
        - to: 10.10.0.0/24
          via: 192.168.1.1
```

```bash
sudo netplan apply
```

### Capture Network Traffic

```bash
# Install tcpdump
sudo apt install tcpdump -y

# Capture all traffic on an interface
sudo tcpdump -i eth0

# Capture only TCP traffic on port 80
sudo tcpdump -i eth0 tcp port 80

# Capture with readable output
sudo tcpdump -i eth0 -A tcp port 80

# Save capture to a file (for analysis in Wireshark)
sudo tcpdump -i eth0 -w /tmp/capture.pcap -c 100

# Read a capture file
sudo tcpdump -r /tmp/capture.pcap

# Filter by host
sudo tcpdump -i eth0 host 192.168.1.100

# Filter by source or destination
sudo tcpdump -i eth0 src 192.168.1.100
sudo tcpdump -i eth0 dst 192.168.1.100
```

### Network Troubleshooting Flowchart

```
Problem: Cannot reach a remote host
│
├── 1. Check local interface: ip addr show
│   └── Is the interface UP with an IP? → If NO → Fix network config
│
├── 2. Ping the local gateway: ping 192.168.1.1
│   └── Can you reach the gateway? → If NO → Check cables/WiFi/switch
│
├── 3. Ping an external IP: ping 8.8.8.8
│   └── Can you reach the internet? → If NO → Check gateway/ISP/firewall
│
├── 4. Ping a domain: ping google.com
│   └── Does DNS resolve? → If NO → Check /etc/resolv.conf, try dig
│
├── 5. Test specific port: nc -zv host port
│   └── Is the port open? → If NO → Check remote firewall/service
│
└── 6. Trace the route: traceroute host
    └── Where does it fail? → Identify the hop causing the problem
```

---

---

## 2. System Monitoring and Performance

### Quick Overview

```bash
# System uptime and load averages
uptime
# Output: 10:30:00 up 45 days, load average: 0.15, 0.10, 0.05
# Load averages: 1 min, 5 min, 15 min
# Guideline: load average should be < number of CPU cores

# Check number of CPU cores
nproc
```

### CPU Monitoring

```bash
# Real-time CPU usage with top
top
# Press '1' to show per-CPU stats
# Press 'P' to sort by CPU usage

# CPU stats per second (install sysstat)
sudo apt install sysstat -y
mpstat 1 5
# Shows CPU stats every 1 second, 5 times

# Detailed CPU info
lscpu
```

### Memory Monitoring

```bash
# Memory usage overview
free -h

# Detailed memory info
cat /proc/meminfo | head -20

# Watch memory usage in real time
watch -n 2 free -h
```

### Disk Monitoring

```bash
# Disk space usage
df -h

# Disk I/O statistics
iostat -x 1 5

# Real-time I/O monitoring
sudo apt install iotop -y
sudo iotop

# Disk health (SMART data)
sudo apt install smartmontools -y
sudo smartctl -a /dev/sda
sudo smartctl -H /dev/sda   # Quick health check
```

### Network Monitoring

```bash
# Interface statistics
ip -s link show

# Real-time bandwidth per interface
sudo apt install iftop -y
sudo iftop -i eth0

# Per-process bandwidth usage
sudo apt install nethogs -y
sudo nethogs eth0
```

### All-in-One Monitoring Tools

```bash
# htop — interactive process viewer
sudo apt install htop -y
htop

# glances — comprehensive monitoring dashboard
sudo apt install glances -y
glances

# nmon — performance monitoring
sudo apt install nmon -y
nmon
# Press: c=CPU, m=Memory, d=Disk, n=Network, t=Top processes
```

### System Health Check Script

```bash
cat > ~/system-health.sh << 'SCRIPT'
#!/bin/bash
#===================================
# System Health Check Script
#===================================

echo "====================================="
echo "  System Health Report"
echo "  $(date)"
echo "====================================="

# Hostname and OS
echo ""
echo "--- System Info ---"
echo "Hostname: $(hostname)"
echo "OS: $(lsb_release -ds 2>/dev/null || cat /etc/os-release | grep PRETTY_NAME | cut -d= -f2)"
echo "Kernel: $(uname -r)"
echo "Uptime: $(uptime -p)"

# CPU
echo ""
echo "--- CPU ---"
echo "Cores: $(nproc)"
echo "Load Average: $(cat /proc/loadavg | awk '{print $1, $2, $3}')"
CPU_USAGE=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}')
echo "CPU Usage: ${CPU_USAGE}%"

# Memory
echo ""
echo "--- Memory ---"
free -h | awk '/Mem:/ {printf "Total: %s | Used: %s | Free: %s | Available: %s\n", $2, $3, $4, $7}'
MEM_PERCENT=$(free | awk '/Mem:/ {printf "%.1f", $3/$2 * 100}')
echo "Memory Usage: ${MEM_PERCENT}%"
if (( $(echo "$MEM_PERCENT > 90" | bc -l) )); then
    echo "⚠️  WARNING: Memory usage is above 90%!"
fi

# Swap
echo ""
echo "--- Swap ---"
free -h | awk '/Swap:/ {printf "Total: %s | Used: %s | Free: %s\n", $2, $3, $4}'

# Disk
echo ""
echo "--- Disk Usage ---"
df -h | awk 'NR==1 || /^\/dev/' | column -t
echo ""
DISK_ALERT=$(df -h | awk '/^\/dev/ {gsub(/%/,"",$5); if ($5 > 80) print $6 " is " $5 "% full"}')
if [ -n "$DISK_ALERT" ]; then
    echo "⚠️  WARNING: High disk usage detected:"
    echo "$DISK_ALERT"
fi

# Top Processes
echo ""
echo "--- Top 5 Processes by CPU ---"
ps aux --sort=-%cpu | head -6 | awk '{printf "%-10s %-8s %-6s %-6s %s\n", $1, $2, $3, $4, $11}'

echo ""
echo "--- Top 5 Processes by Memory ---"
ps aux --sort=-%mem | head -6 | awk '{printf "%-10s %-8s %-6s %-6s %s\n", $1, $2, $3, $4, $11}'

# Network
echo ""
echo "--- Listening Ports ---"
ss -tlnp 2>/dev/null | head -15

# Failed Services
echo ""
echo "--- Failed Services ---"
FAILED=$(systemctl --failed --no-legend 2>/dev/null)
if [ -z "$FAILED" ]; then
    echo "✅ No failed services"
else
    echo "⚠️  Failed services:"
    echo "$FAILED"
fi

echo ""
echo "====================================="
echo "  Health check complete"
echo "====================================="
SCRIPT

chmod +x ~/system-health.sh
```

```bash
# Run the health check
~/system-health.sh
```

---

---

## 3. Setting Up a LEMP Stack

> **LEMP** = **L**inux + **(E)Nginx** + **M**ySQL/MariaDB + **P**HP

### Step 1: Install Nginx

```bash
sudo apt update
sudo apt install nginx -y

# Start and enable
sudo systemctl start nginx
sudo systemctl enable nginx

# Verify
sudo systemctl status nginx
curl -I http://localhost
```

### Step 2: Install MySQL (MariaDB)

```bash
# Install MariaDB
sudo apt install mariadb-server mariadb-client -y

# Start and enable
sudo systemctl start mariadb
sudo systemctl enable mariadb

# Run security script
sudo mysql_secure_installation
```

When prompted:
- **Switch to unix_socket auth?** → N (if already set)
- **Set root password?** → Y → Enter a strong password
- **Remove anonymous users?** → Y
- **Disallow root login remotely?** → Y
- **Remove test database?** → Y
- **Reload privilege tables?** → Y

**Create a database and user:**

```bash
sudo mysql -u root -p
```

```sql
-- Create a database
CREATE DATABASE myapp_db;

-- Create a user with a password
CREATE USER 'myapp_user'@'localhost' IDENTIFIED BY 'StrongPassword123!';

-- Grant privileges
GRANT ALL PRIVILEGES ON myapp_db.* TO 'myapp_user'@'localhost';

-- Apply changes
FLUSH PRIVILEGES;

-- Verify
SHOW DATABASES;
SELECT User, Host FROM mysql.user;

EXIT;
```

### Step 3: Install PHP-FPM

```bash
# Install PHP and common extensions
sudo apt install php-fpm php-mysql php-cli php-curl php-gd php-mbstring php-xml php-zip -y

# Check PHP version
php --version

# Verify PHP-FPM is running
sudo systemctl status php*-fpm
```

### Step 4: Configure Nginx for PHP

```bash
sudo nano /etc/nginx/sites-available/myapp
```

```nginx
server {
    listen 80;
    server_name your_domain_or_ip;

    root /var/www/myapp;
    index index.php index.html index.htm;

    # Logs
    access_log /var/log/nginx/myapp_access.log;
    error_log /var/log/nginx/myapp_error.log;

    location / {
        try_files $uri $uri/ =404;
    }

    # PHP processing
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    # Deny access to .htaccess files
    location ~ /\.ht {
        deny all;
    }
}
```

### Step 5: Enable the Site

```bash
# Create the document root
sudo mkdir -p /var/www/myapp
sudo chown -R www-data:www-data /var/www/myapp

# Enable the site
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/

# Remove default site (optional)
sudo rm -f /etc/nginx/sites-enabled/default

# Test and reload
sudo nginx -t
sudo systemctl reload nginx
```

### Step 6: Create a PHP Test Page

```bash
sudo nano /var/www/myapp/index.php
```

```php
<!DOCTYPE html>
<html>
<head><title>LEMP Stack Test</title></head>
<body>
    <h1>LEMP Stack is Working!</h1>

    <h2>PHP Info</h2>
    <p>PHP Version: <?php echo phpversion(); ?></p>
    <p>Server Software: <?php echo $_SERVER['SERVER_SOFTWARE']; ?></p>

    <h2>MySQL Connection Test</h2>
    <?php
    $host = 'localhost';
    $db   = 'myapp_db';
    $user = 'myapp_user';
    $pass = 'StrongPassword123!';

    try {
        $pdo = new PDO("mysql:host=$host;dbname=$db", $user, $pass);
        $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
        echo "<p style='color: green;'>✅ Connected to MySQL successfully!</p>";

        // Show MySQL version
        $stmt = $pdo->query("SELECT VERSION() as version");
        $row = $stmt->fetch(PDO::FETCH_ASSOC);
        echo "<p>MySQL Version: " . $row['version'] . "</p>";
    } catch (PDOException $e) {
        echo "<p style='color: red;'>❌ Connection failed: " . $e->getMessage() . "</p>";
    }
    ?>
</body>
</html>
```

### Step 7: Test

```bash
# Test in the browser or with curl
curl http://localhost

# Check PHP-FPM processing
curl http://localhost/index.php
```

### Security Cleanup

```bash
# Remove the test file after verification (it exposes sensitive info!)
sudo rm /var/www/myapp/index.php

# Create a proper index file
echo "<h1>Welcome to MyApp</h1>" | sudo tee /var/www/myapp/index.html
```

---

---

## 4. Systemd Timers (Modern Alternative to Cron)

### Why Timers Over Cron?

| Feature | Cron | Systemd Timers |
|---|---|---|
| Logging | Manual (redirect output) | Automatic (journalctl) |
| Dependencies | None | Can depend on other services |
| Missed runs | Skipped | `Persistent=true` catches up |
| Resource control | None | Full systemd resource limits (CPU, Memory) |
| Accuracy | Minute-level | Microsecond-level |
| Calendar syntax | `* * * * *` | Human-readable (`daily`, `Mon *-*-* 09:00`) |
| Management | `crontab -e` | `systemctl`, `systemd-analyze` |
| Randomized delay | Not supported | `RandomizedDelaySec=` |

### Step 1: Create the Service Unit

```bash
sudo nano /etc/systemd/system/my-backup.service
```

```ini
[Unit]
Description=Run daily backup script

[Service]
Type=oneshot
User=root
ExecStart=/usr/local/bin/backup.sh
StandardOutput=journal
StandardError=journal
```

> **`Type=oneshot`** means the service runs, completes, and exits (not a long-running daemon).

Create the backup script:

```bash
sudo nano /usr/local/bin/backup.sh
```

```bash
#!/bin/bash
set -euo pipefail

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/var/backups/myapp"
SOURCE="/var/www/myapp"

mkdir -p "$BACKUP_DIR"

tar -czf "$BACKUP_DIR/myapp_$TIMESTAMP.tar.gz" "$SOURCE" 2>/dev/null

# Keep only last 7 backups
ls -t "$BACKUP_DIR"/myapp_*.tar.gz | tail -n +8 | xargs -r rm

echo "Backup completed: myapp_$TIMESTAMP.tar.gz"
```

```bash
sudo chmod +x /usr/local/bin/backup.sh
```

### Step 2: Create the Timer Unit

```bash
sudo nano /etc/systemd/system/my-backup.timer
```

```ini
[Unit]
Description=Daily backup timer

[Timer]
# Run daily at 2:30 AM
OnCalendar=*-*-* 02:30:00

# Add random delay (up to 10 min) to avoid all systems running at once
RandomizedDelaySec=600

# If the system was off during scheduled time, run on next boot
Persistent=true

[Install]
WantedBy=timers.target
```

**OnCalendar syntax:**

| Expression | Meaning |
|---|---|
| `*-*-* 02:30:00` | Every day at 2:30 AM |
| `Mon *-*-* 09:00:00` | Every Monday at 9 AM |
| `*-*-01 00:00:00` | First day of every month at midnight |
| `*-01,07-01 00:00:00` | January 1st and July 1st |
| `hourly` | Every hour |
| `daily` | Every day at midnight |
| `weekly` | Every Monday at midnight |
| `monthly` | First day of each month at midnight |
| `*-*-* *:00/15:00` | Every 15 minutes |

```bash
# Verify your calendar expression
systemd-analyze calendar "*-*-* 02:30:00"
systemd-analyze calendar "Mon *-*-* 09:00:00"
```

### Step 3: Enable and Start the Timer

```bash
# Reload systemd
sudo systemctl daemon-reload

# Enable the timer (starts at boot)
sudo systemctl enable my-backup.timer

# Start the timer now
sudo systemctl start my-backup.timer

# Verify
sudo systemctl status my-backup.timer
sudo systemctl list-timers --all | grep my-backup
```

### Step 4: View Logs

```bash
# View timer execution logs
sudo journalctl -u my-backup.service

# View recent runs
sudo journalctl -u my-backup.service --since "24 hours ago"

# Follow in real time
sudo journalctl -u my-backup.service -f
```

### Step 5: Test Manually

```bash
# Trigger the service manually (without waiting for the timer)
sudo systemctl start my-backup.service

# Check the result
sudo systemctl status my-backup.service
sudo journalctl -u my-backup.service -n 10
ls -la /var/backups/myapp/
```

### Cleanup

```bash
sudo systemctl stop my-backup.timer
sudo systemctl disable my-backup.timer
sudo rm /etc/systemd/system/my-backup.service
sudo rm /etc/systemd/system/my-backup.timer
sudo rm /usr/local/bin/backup.sh
sudo systemctl daemon-reload
```

---

---

## 5. LVM — Logical Volume Management

### What is LVM?

LVM adds a flexible abstraction layer between physical disks and filesystems.

```
┌──────────────────────────────────────────────────────┐
│                   Filesystem (ext4/xfs)               │
│                   Mounted at /data                    │
├──────────────────────────────────────────────────────┤
│              Logical Volume (LV)                      │
│              lv_data — 50GB                           │
├──────────────────────────────────────────────────────┤
│              Volume Group (VG)                        │
│              vg_storage — 100GB total                 │
├─────────────────────┬────────────────────────────────┤
│   Physical Volume    │   Physical Volume              │
│   /dev/sdb (50GB)    │   /dev/sdc (50GB)             │
└─────────────────────┴────────────────────────────────┘
```

| Layer | Purpose | Commands |
|---|---|---|
| **PV** (Physical Volume) | Mark disks for LVM use | `pvcreate`, `pvdisplay`, `pvs` |
| **VG** (Volume Group) | Pool PVs together | `vgcreate`, `vgdisplay`, `vgs` |
| **LV** (Logical Volume) | Carve usable volumes from VG | `lvcreate`, `lvdisplay`, `lvs` |

### Step 1: Install LVM

```bash
sudo apt install lvm2 -y
```

### Step 2: Identify Available Disks

```bash
# List block devices
lsblk

# Example: /dev/sdb and /dev/sdc are new unpartitioned disks
sudo fdisk -l /dev/sdb /dev/sdc
```

### Step 3: Create Physical Volumes (PV)

```bash
# Initialize disks for LVM
sudo pvcreate /dev/sdb /dev/sdc

# Verify
sudo pvs
sudo pvdisplay
```

### Step 4: Create a Volume Group (VG)

```bash
# Create a volume group from the two PVs
sudo vgcreate vg_storage /dev/sdb /dev/sdc

# Verify
sudo vgs
sudo vgdisplay vg_storage
```

### Step 5: Create Logical Volumes (LV)

```bash
# Create a 30GB logical volume
sudo lvcreate -L 30G -n lv_data vg_storage

# Create another LV using a percentage of remaining space
sudo lvcreate -l 50%FREE -n lv_logs vg_storage

# Verify
sudo lvs
sudo lvdisplay
```

### Step 6: Create Filesystems and Mount

```bash
# Create ext4 filesystems
sudo mkfs.ext4 /dev/vg_storage/lv_data
sudo mkfs.ext4 /dev/vg_storage/lv_logs

# Create mount points
sudo mkdir -p /mnt/data /mnt/logs

# Mount
sudo mount /dev/vg_storage/lv_data /mnt/data
sudo mount /dev/vg_storage/lv_logs /mnt/logs

# Verify
df -h /mnt/data /mnt/logs
```

### Step 7: Make Mounts Permanent

```bash
# Add entries to /etc/fstab
echo '/dev/vg_storage/lv_data  /mnt/data  ext4  defaults  0  2' | sudo tee -a /etc/fstab
echo '/dev/vg_storage/lv_logs  /mnt/logs  ext4  defaults  0  2' | sudo tee -a /etc/fstab

# Verify fstab
sudo mount -a
df -h /mnt/data /mnt/logs
```

### Step 8: Extend a Logical Volume

**Method 1: Add a specific amount:**

```bash
# Add 10GB to lv_data
sudo lvextend -L +10G /dev/vg_storage/lv_data

# Resize the filesystem to use the new space
sudo resize2fs /dev/vg_storage/lv_data
```

**Method 2: Extend to a specific size:**

```bash
sudo lvextend -L 50G /dev/vg_storage/lv_data
sudo resize2fs /dev/vg_storage/lv_data
```

**Method 3: Use all remaining free space:**

```bash
sudo lvextend -l +100%FREE /dev/vg_storage/lv_data
sudo resize2fs /dev/vg_storage/lv_data
```

```bash
# Verify
df -h /mnt/data
```

### Step 9: Add a New Disk to the Volume Group

```bash
# A new disk /dev/sdd has been attached
sudo pvcreate /dev/sdd

# Add it to the existing volume group
sudo vgextend vg_storage /dev/sdd

# Verify — the VG now has more free space
sudo vgs
```

### Step 10: Create an LVM Snapshot

```bash
# Create a snapshot of lv_data (allocate 5GB for changes)
sudo lvcreate -L 5G -s -n lv_data_snap /dev/vg_storage/lv_data

# Verify
sudo lvs

# Mount the snapshot as read-only
sudo mkdir -p /mnt/data_snap
sudo mount -o ro /dev/vg_storage/lv_data_snap /mnt/data_snap

# Browse the snapshot (point-in-time copy)
ls /mnt/data_snap/
```

**Restore from a snapshot:**

```bash
# Unmount both
sudo umount /mnt/data
sudo umount /mnt/data_snap

# Restore (merge snapshot back into original)
sudo lvconvert --merge /dev/vg_storage/lv_data_snap

# Remount
sudo mount /dev/vg_storage/lv_data /mnt/data
```

**Remove a snapshot (without restoring):**

```bash
sudo umount /mnt/data_snap
sudo lvremove /dev/vg_storage/lv_data_snap
```

### Cleanup

```bash
# Unmount
sudo umount /mnt/data
sudo umount /mnt/logs

# Remove fstab entries
sudo sed -i '/vg_storage/d' /etc/fstab

# Remove logical volumes
sudo lvremove /dev/vg_storage/lv_data -y
sudo lvremove /dev/vg_storage/lv_logs -y

# Remove volume group
sudo vgremove vg_storage -y

# Remove physical volumes
sudo pvremove /dev/sdb /dev/sdc

# Remove mount points
sudo rmdir /mnt/data /mnt/logs /mnt/data_snap 2>/dev/null
```

---

---

## 6. Hardening a Linux Server — Security Checklist

### 1. Keep the System Updated

```bash
# Update everything
sudo apt update && sudo apt upgrade -y

# Enable automatic security updates
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure -plow unattended-upgrades

# Verify configuration
cat /etc/apt/apt.conf.d/20auto-upgrades
```

Expected content:

```
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

### 2. Secure SSH

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
sudo nano /etc/ssh/sshd_config
```

Apply these settings:

```
# Change default port (obscurity, not security — but reduces noise)
Port 2222

# Disable root login
PermitRootLogin no

# Disable password authentication (use keys only)
PasswordAuthentication no

# Allow only specific users
AllowUsers deployer admin

# Limit authentication attempts
MaxAuthTries 3

# Set idle timeout (disconnect after 5 min of inactivity)
ClientAliveInterval 300
ClientAliveCountMax 0

# Disable empty passwords
PermitEmptyPasswords no

# Disable X11 forwarding (unless needed)
X11Forwarding no
```

```bash
# Test configuration
sudo sshd -t

# Restart SSH
sudo systemctl restart sshd
```

> ⚠️ **Keep your current SSH session open** and test in a **new terminal** before closing!

### 3. Configure Firewall (UFW)

```bash
# Set defaults
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH on custom port
sudo ufw allow 2222/tcp

# Allow web traffic
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Enable
sudo ufw enable
sudo ufw status verbose
```

### 4. Install and Configure Fail2Ban

```bash
sudo apt install fail2ban -y

# Create local config (never edit jail.conf directly)
sudo nano /etc/fail2ban/jail.local
```

```ini
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 3
banaction = ufw

[sshd]
enabled = true
port = 2222
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600
```

```bash
# Start and enable
sudo systemctl enable fail2ban
sudo systemctl start fail2ban

# Check status
sudo fail2ban-client status
sudo fail2ban-client status sshd

# View banned IPs
sudo fail2ban-client get sshd banned

# Unban an IP
sudo fail2ban-client set sshd unbanip 192.168.1.100
```

### 5. Disable Unused Services

```bash
# List all running services
sudo systemctl list-units --type=service --state=running

# Disable services you don't need (examples)
sudo systemctl stop cups
sudo systemctl disable cups

sudo systemctl stop avahi-daemon
sudo systemctl disable avahi-daemon

sudo systemctl stop bluetooth
sudo systemctl disable bluetooth

# List enabled services
sudo systemctl list-unit-files --type=service --state=enabled
```

### 6. Secure Shared Memory

```bash
# Prevent programs from using shared memory for exploits
echo "tmpfs /run/shm tmpfs defaults,noexec,nosuid 0 0" | sudo tee -a /etc/fstab

# Apply immediately
sudo mount -o remount /run/shm
```

### 7. Log Monitoring (Logwatch)

```bash
# Install logwatch
sudo apt install logwatch -y

# Run a report
sudo logwatch --detail High --range today --output stdout

# Configure daily email reports
sudo nano /etc/logwatch/conf/logwatch.conf
```

```
Output = mail
MailTo = admin@example.com
Detail = Med
Range = yesterday
```

### 8. Audit User Accounts

```bash
# Find users with empty passwords
sudo awk -F: '($2 == "" ) {print $1}' /etc/shadow

# Find all users with UID 0 (should only be root)
awk -F: '($3 == 0) {print $1}' /etc/passwd

# List users who can log in
grep -v '/nologin\|/false' /etc/passwd

# Lock unused accounts
sudo usermod -L username

# Set password aging policy
sudo nano /etc/login.defs
```

```
PASS_MAX_DAYS   90
PASS_MIN_DAYS   7
PASS_WARN_AGE   14
```

### 9. File Integrity Monitoring (AIDE)

```bash
# Install AIDE (Advanced Intrusion Detection Environment)
sudo apt install aide -y

# Initialize the database (takes a few minutes)
sudo aideinit

# The database is created at /var/lib/aide/aide.db.new
sudo cp /var/lib/aide/aide.db.new /var/lib/aide/aide.db

# Run a check (compare current system against the database)
sudo aide --check

# After legitimate changes, update the database
sudo aide --update
sudo cp /var/lib/aide/aide.db.new /var/lib/aide/aide.db
```

### Security Checklist Summary

| # | Task | Status |
|---|---|---|
| 1 | System fully updated | ☐ |
| 2 | Automatic security updates enabled | ☐ |
| 3 | SSH: non-default port | ☐ |
| 4 | SSH: root login disabled | ☐ |
| 5 | SSH: password authentication disabled | ☐ |
| 6 | SSH: key-based authentication configured | ☐ |
| 7 | Firewall enabled with minimal open ports | ☐ |
| 8 | Fail2Ban installed and configured | ☐ |
| 9 | Unused services disabled | ☐ |
| 10 | Shared memory secured | ☐ |
| 11 | Log monitoring configured | ☐ |
| 12 | No accounts with empty passwords | ☐ |
| 13 | Only root has UID 0 | ☐ |
| 14 | Password aging policy set | ☐ |
| 15 | File integrity monitoring (AIDE) initialized | ☐ |
| 16 | Unused user accounts locked | ☐ |

---

---

## 7. Quick Reference — Essential Linux Commands

### Navigation

| Command | Description |
|---|---|
| `pwd` | Print working directory |
| `cd /path` | Change directory |
| `cd ..` | Go up one directory |
| `cd ~` | Go to home directory |
| `cd -` | Go to previous directory |
| `ls -la` | List all files (long format, hidden) |
| `tree -L 2` | Show directory tree (2 levels deep) |

### File Operations

| Command | Description |
|---|---|
| `cp src dest` | Copy file |
| `cp -r src/ dest/` | Copy directory recursively |
| `mv src dest` | Move/rename file |
| `rm file` | Remove file |
| `rm -rf dir/` | Remove directory recursively |
| `mkdir -p path/to/dir` | Create directories (with parents) |
| `touch file` | Create empty file / update timestamp |
| `ln -s target link` | Create symbolic link |
| `find / -name "*.log"` | Find files by name |
| `locate filename` | Quick file search (uses database) |
| `which command` | Show full path of a command |

### Text Processing

| Command | Description |
|---|---|
| `cat file` | Display file contents |
| `less file` | Page through a file |
| `head -n 20 file` | First 20 lines |
| `tail -n 20 file` | Last 20 lines |
| `tail -f file` | Follow file in real time |
| `grep "pattern" file` | Search for pattern in file |
| `grep -r "pattern" dir/` | Recursive search |
| `awk '{print $1}' file` | Print first column |
| `sed 's/old/new/g' file` | Find and replace |
| `sort file` | Sort lines |
| `uniq` | Remove duplicate adjacent lines |
| `wc -l file` | Count lines |
| `cut -d: -f1 file` | Cut fields by delimiter |
| `diff file1 file2` | Compare two files |

### System Information

| Command | Description |
|---|---|
| `uname -a` | System information |
| `hostnamectl` | Hostname and OS info |
| `uptime` | System uptime and load |
| `whoami` | Current username |
| `id` | User and group IDs |
| `date` | Current date and time |
| `cal` | Calendar |
| `free -h` | Memory usage |
| `df -h` | Disk usage |
| `du -sh dir/` | Directory size |
| `top` / `htop` | Process monitor |
| `ps aux` | List all processes |
| `lsblk` | List block devices |
| `lscpu` | CPU information |
| `dmidecode` | Hardware information |

### Network

| Command | Description |
|---|---|
| `ip addr show` | Show IP addresses |
| `ip route show` | Show routing table |
| `ping -c 4 host` | Test connectivity |
| `traceroute host` | Trace network path |
| `dig domain` | DNS lookup |
| `nslookup domain` | DNS lookup (simpler) |
| `ss -tlnp` | Show listening ports |
| `curl -I url` | Fetch HTTP headers |
| `wget url` | Download a file |
| `scp file user@host:/path` | Secure copy |
| `rsync -av src/ dest/` | Sync directories |
| `netstat -tlnp` | Show listening ports (legacy) |

### Permissions & Ownership

| Command | Description |
|---|---|
| `chmod 755 file` | Set permissions (numeric) |
| `chmod u+x file` | Add execute for owner (symbolic) |
| `chown user:group file` | Change owner and group |
| `chgrp group file` | Change group |
| `umask 022` | Set default permission mask |
| `sudo command` | Run as superuser |
| `su - user` | Switch user (login shell) |
| `visudo` | Edit sudoers safely |

---

*End of Hard Scenarios*
