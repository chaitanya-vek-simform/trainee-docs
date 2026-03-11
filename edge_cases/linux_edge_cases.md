[🏠 Home](../README.md) · [Edge Cases](.)

# 🐧 Linux Admin — Edge Cases (110)

> **Audience:** DevOps engineers, SREs, and Linux admins dealing with production systems.  
> **Coverage:** Real-world surprises, timing-dependent bugs, silent failures, and configuration traps encountered in industry.

---

## Table of Contents

1. [File System & Disk (EC-001–015)](#file-system--disk)
2. [Process & Job Management (EC-016–025)](#process--job-management)
3. [Users, Permissions & Security (EC-026–035)](#users-permissions--security)
4. [Networking (EC-036–045)](#networking)
5. [SSH & Remote Access (EC-046–055)](#ssh--remote-access)
6. [systemd & Services (EC-056–065)](#systemd--services)
7. [Package Management (EC-066–075)](#package-management)
8. [Shell & Scripting (EC-076–085)](#shell--scripting)
9. [Performance & Resource Management (EC-086–095)](#performance--resource-management)
10. [LVM, RAID & Storage (EC-096–105)](#lvm-raid--storage)
11. [Kernel & Boot (EC-106–110)](#kernel--boot)
12. [Production FAQ](#production-faq)

---

## File System & Disk

### EC-001 — `df` Shows 100% But `du` Shows Free Space
**Category:** File System | **Severity:** Critical | **Env:** Prod

**Scenario:** Disk appears full (`df -h` shows 100%) but `du -sh /*` only accounts for 60% of disk space.

**What breaks:** Applications fail to write, logs stop, databases crash with "no space left on device".

```
  df -h          du -sh totals
  ┌──────────┐   ┌──────────┐
  │ 100% used│   │ ~60% used│ ← gap = deleted-but-open files
  └──────────┘   └──────────┘
         ↑
  Open file handles hold
  space even after unlink()
```

**Detection:**
```bash
lsof +L1 | grep -i deleted | awk '{print $7, $9}' | sort -rn | head -20
```

**Fix:**
```bash
# Restart the process holding the deleted file, or truncate the fd:
# Find the PID and fd number from lsof output:
> /proc/<PID>/fd/<FD>     # truncate without restart
# Or graceful restart of the service:
systemctl restart <service>
```

> ⚠️ **DevOps Gotcha:** Log rotation that uses `copytruncate` avoids this, but standard rotation (rename + HUP) needs the daemon to reopen its log FD. Always add `postrotate` signals to logrotate configs.

---

### EC-002 — Inode Exhaustion With Plenty of Block Space
**Category:** File System | **Severity:** Critical | **Env:** Prod

**Scenario:** `df -h` shows free space but writing any file gives "No space left on device".

**What breaks:** Cannot create new files, cron jobs fail, mail spools, CI artifact directories.

**Detection:**
```bash
df -i          # shows inode usage — look for 100% on IUse%
find / -xdev -printf '%h\n' | sort | uniq -c | sort -rn | head -20
```

**Fix:**
```bash
# Find the directory with millions of small files:
find /var/spool/postfix -type f | wc -l   # mail queue
find /tmp -name "sess_*" | wc -l           # PHP sessions

# Clean up or archive
find /tmp -name "sess_*" -mtime +1 -delete
```

> ⚠️ **DevOps Gotcha:** PHP session files, Postfix mail queues, and Kubernetes ephemeral containers are the top offenders. Set `inode_ratio` lower in `mkfs.ext4` for directories that hold millions of tiny files.

---

### EC-003 — `/tmp` Survives Reboot on systemd Systems
**Category:** File System | **Severity:** Medium | **Env:** Prod

**Scenario:** Expecting `/tmp` to be empty after reboot, but stale lock files and socket files remain.

**What breaks:** Applications check for lock files at startup and refuse to start, thinking another instance is running.

**Detection:**
```bash
systemctl status tmp.mount
# If not active, /tmp is NOT a tmpfs
cat /etc/systemd/system/tmp.mount 2>/dev/null || echo "Not using tmpfs"
```

**Fix:**
```bash
# Enable tmpfs for /tmp:
systemctl enable --now tmp.mount
# Or in /etc/fstab:
tmpfs  /tmp  tmpfs  defaults,noatime,nosuid,nodev,noexec,size=2G  0 0
```

> ⚠️ **DevOps Gotcha:** In Docker containers `/tmp` is NOT cleaned on container restart (only on full image rebuild). Always clean lock files in your entrypoint script.

---

### EC-004 — `rm -rf` on a Bind Mount Removes the Source
**Category:** File System | **Severity:** Critical | **Env:** Prod

**Scenario:** Running `rm -rf /mnt/backup/*` when `/mnt/backup` is a bind mount of `/data` deletes production data.

**What breaks:** Production data loss.

```
  Before rm:              After rm -rf /mnt/backup/*:
  /data  (source)         /data  ← EMPTY! Bind mount writes
     └─ file.txt             └─ (gone)   through to source
  /mnt/backup (bind)
     └─ file.txt  ← deleting here deletes /data/file.txt
```

**Detection:**
```bash
mount | grep /mnt/backup     # check if it's a bind mount
findmnt /mnt/backup
```

> ⚠️ **DevOps Gotcha:** Always check `mount` or `findmnt` before mass deletions. Kubernetes `hostPath` volumes are bind mounts — deleting inside the container deletes on the host.

---

### EC-005 — ext4 Journal Replay on Unclean Shutdown Takes Minutes
**Category:** File System | **Severity:** High | **Env:** Prod

**Scenario:** After power failure or kernel panic, system takes 5–20 minutes to mount large ext4 filesystem.

**What breaks:** Boot time SLA, RTO for DR recovery.

**Detection:**
```bash
dmesg | grep -E "EXT4|journal|replay"
# Look for: "EXT4-fs: recovering journal"
tune2fs -l /dev/sda1 | grep -E "state|mount count"
```

**Fix:**
```bash
# To force journal replay manually:
e2fsck -f /dev/sda1
# Consider switching to xfs for large filesystems (no journal replay needed)
```

> ⚠️ **DevOps Gotcha:** Cloud EBS volumes and vSphere VMDKs can take 10+ minutes to fsck after unclean shutdown. Plan your RTO accordingly and use `noatime` mount option to reduce journal writes.

---

### EC-006 — NFS `soft` Mount Hangs the Kernel Thread Forever
**Category:** File System | **Severity:** Critical | **Env:** Prod

**Scenario:** NFS server goes down. Application using `hard` NFS mount hangs forever — process becomes unkillable (D state).

**What breaks:** The entire process is stuck in uninterruptible sleep. `kill -9` doesn't work.

**Detection:**
```bash
ps aux | grep -E "^D"
cat /proc/<PID>/wchan     # shows "nfs_wait_on_..."
```

**Fix:**
```bash
# Short term: remount with soft + timeo options:
mount -o remount,soft,timeo=30,retrans=3 /mnt/nfs

# Long term: in /etc/fstab:
nfs-server:/share  /mnt/nfs  nfs  soft,timeo=30,retrans=3,intr  0 0
```

> ⚠️ **DevOps Gotcha:** `hard` NFS mounts are data-safe but availability-unsafe. `soft` mounts can return I/O errors (which apps can handle) instead of hanging forever. Choose based on your fault tolerance requirement.

---

### EC-007 — `chmod -R 777` Breaks SUID Binaries
**Category:** File System | **Severity:** High | **Env:** Prod

**Scenario:** Someone runs `chmod -R 777 /` or a wide path that includes `/usr/bin/passwd` or `sudo`.

**What breaks:** SUID bit is preserved by `777` but setuid programs may refuse to run if world-writable. `sudo` refuses to run if any parent directory is world-writable.

**Detection:**
```bash
sudo -l    # "sudo: /etc/sudoers is world writable" error
ls -la /usr/bin/sudo
```

**Fix:**
```bash
chmod 4755 /usr/bin/sudo
chmod 4755 /usr/bin/passwd
chmod 440 /etc/sudoers
```

> ⚠️ **DevOps Gotcha:** Ansible `file` module with `mode: '0777'` on `/var/www` can silently break PHP's `session.save_path` if that path is under `/var/lib/php/sessions`.

---

### EC-008 — Symlink Race Condition in `/tmp` (TOCTOU)
**Category:** File System | **Severity:** High | **Env:** Prod

**Scenario:** A root-running script checks `if [ -f /tmp/lockfile ]` then writes to it. An attacker pre-creates a symlink pointing to `/etc/passwd`.

**What breaks:** Root writes to `/etc/passwd` instead of lockfile. Security breach.

**Detection:**
```bash
ls -la /tmp/lockfile  # check if it's a symlink
```

**Fix:**
```bash
# Use mktemp for safe temp file creation:
TMPFILE=$(mktemp /tmp/myscript.XXXXXX)
# Or use O_EXCL flag in code: open("/tmp/lockfile", O_CREAT|O_EXCL|O_WRONLY, 0600)
# Use /run/<appname>/ instead of /tmp for lock files
```

> ⚠️ **DevOps Gotcha:** Many old init scripts use `/tmp` for lock files. Always use `/var/run/<appname>/` or `/run/<appname>/` which is on tmpfs and owned by root.

---

### EC-009 — `noexec` Mount Option Breaks Interpreted Scripts
**Category:** File System | **Severity:** Medium | **Env:** Prod

**Scenario:** `/tmp` or `/var` is mounted with `noexec`. Shell scripts stored there fail silently or with "Permission denied".

**What breaks:** Ansible temp files (it copies scripts to `/tmp` and executes them), some installers, CI runners.

**Detection:**
```bash
mount | grep noexec
findmnt -o TARGET,OPTIONS | grep noexec
```

**Fix:**
```bash
# For Ansible, set remote_tmp to an executable location:
# In ansible.cfg: remote_tmp = /home/{{ ansible_user }}/.ansible/tmp
# Or remount /tmp without noexec (less secure):
mount -o remount,exec /tmp
```

> ⚠️ **DevOps Gotcha:** CIS-hardened images mount `/tmp noexec`. Ansible will silently fail or throw cryptic errors. Always set `remote_tmp` in your Ansible config when targeting hardened hosts.

---

### EC-010 — `df` Shows Wrong Filesystem Size After LVM Resize
**Category:** File System | **Severity:** High | **Env:** Prod

**Scenario:** `lvextend` runs successfully but `df -h` still shows the old size.

**What breaks:** Application thinks disk is still full.

**Detection:**
```bash
lvdisplay /dev/mapper/vg-data   # shows new LV size
df -h /data                      # still shows old size
```

**Fix:**
```bash
# For ext4:
resize2fs /dev/mapper/vg-data
# For xfs (must be mounted):
xfs_growfs /data
# For btrfs:
btrfs filesystem resize max /data
```

> ⚠️ **DevOps Gotcha:** `lvextend -r` does both steps atomically (extend + resize filesystem). Without `-r`, you must manually run `resize2fs` or `xfs_growfs`. The filesystem has NO idea the LV changed.

---

### EC-011 — Deleted File Recreated by Another Process Immediately
**Category:** File System | **Severity:** Medium | **Env:** Dev/Prod

**Scenario:** You delete a config file, expecting the service to fail. Instead, another process (watchdog, config manager, Kubernetes ConfigMap controller) recreates it immediately.

**Detection:**
```bash
inotifywait -m /path/to/file    # watch for CREATE events
auditctl -w /etc/app/config.yml -p wra -k config_change
ausearch -k config_change
```

> ⚠️ **DevOps Gotcha:** Kubernetes ConfigMap-mounted files are recreated by kubelet within seconds. Deleting them in the container does nothing to the actual mount.

---

### EC-012 — `find -mtime` Off-by-One Due to Partial Day Counting
**Category:** File System | **Severity:** Medium | **Env:** Prod

**Scenario:** `find /logs -mtime +7 -delete` deletes files you didn't expect because `mtime` counts 24-hour blocks, not calendar days.

**What breaks:** Log retention policies that keep less than expected.

**Detection:**
```bash
find /logs -mtime +7 -ls | head -10   # verify before deleting
# Use -mmin for minute-based precision:
find /logs -mmin +$((7*24*60)) -ls
```

> ⚠️ **DevOps Gotcha:** A file created 7 days and 1 second ago is `-mtime +6` not `-mtime +7`. Use `+7` to mean "strictly more than 7 complete 24-hour periods".

---

### EC-013 — Filesystem Mounted Read-Only After Kernel Error
**Category:** File System | **Severity:** Critical | **Env:** Prod

**Scenario:** Kernel detects I/O error on disk and automatically remounts the filesystem read-only to prevent corruption. Applications start failing with "Read-only file system".

**Detection:**
```bash
dmesg | grep -E "remount|read-only|I/O error|EXT4-fs error"
mount | grep "ro,"
cat /proc/mounts | grep " ro "
```

**Fix:**
```bash
# After fixing the underlying disk issue:
mount -o remount,rw /
# Or if ext4 journal errors set ERRORS_REMOUNT:
tune2fs -e continue /dev/sda1   # change error behavior
```

> ⚠️ **DevOps Gotcha:** AWS EBS volumes can go into a bad state due to hypervisor issues. CloudWatch disk metrics will look normal until the filesystem auto-remounts read-only. Monitor for `fs.mount.readonly` in your monitoring.

---

### EC-014 — Hardlinks Fool Disk Usage Accounting
**Category:** File System | **Severity:** Medium | **Env:** Prod

**Scenario:** `du -sh /var/lib/docker` shows 50GB but `df -h` shows only 20GB used. Docker uses hardlinks between image layers.

**Detection:**
```bash
du -sh --count-links /path    # counts each hardlink separately
du -sh /path                   # counts each inode once
```

> ⚠️ **DevOps Gotcha:** `rsync` with `--hard-links` preserves hardlinks. Without it, backup size doubles. `cp -a` preserves hardlinks; `cp -r` breaks them. Docker's overlay2 storage driver uses this extensively.

---

### EC-015 — `/proc` and `/sys` Files Are Not Regular Files
**Category:** File System | **Severity:** Medium | **Env:** Prod

**Scenario:** `cp /proc/sys/net/ipv4/ip_forward /backup/` creates a 0-byte file. `scp` of `/proc` files always copies 0 bytes.

**Detection:**
```bash
ls -la /proc/cpuinfo     # shows 0 bytes but cat shows content
file /proc/cpuinfo       # reports it as empty
```

> ⚠️ **DevOps Gotcha:** Never back up `/proc` or `/sys` in your rsync/backup scripts. Use `--exclude=/proc --exclude=/sys`. Tar archives of `/proc` will cause `tar` to hang waiting for reads.

---

## Process & Job Management

### EC-016 — Zombie Processes Accumulate Until PID Namespace Exhausts
**Category:** Process | **Severity:** High | **Env:** Prod

**Scenario:** Long-running daemon forks children but never calls `wait()`. Children exit but become zombies (Z state). After thousands accumulate, `fork()` fails with EAGAIN.

**What breaks:** Application cannot spawn new workers. Web server stops accepting connections.

**Detection:**
```bash
ps aux | awk '$8 == "Z"' | wc -l
cat /proc/sys/kernel/pid_max     # default 32768
```

**Fix:**
```bash
# Short term: restart the parent process (kills all zombies)
# Long term: fix the code to call waitpid() or install SIGCHLD handler
# Or set SIGCHLD to SIG_IGN to auto-reap:
# signal(SIGCHLD, SIG_IGN);
```

> ⚠️ **DevOps Gotcha:** In Kubernetes, PID 1 inside a container must reap zombies. Use `tini` or `dumb-init` as your container entrypoint. Without it, zombie accumulation is guaranteed for long-running pods.

---

### EC-017 — `kill -9` Doesn't Kill a Process in D State
**Category:** Process | **Severity:** Critical | **Env:** Prod

**Scenario:** Process is stuck waiting for I/O (D = uninterruptible sleep). SIGKILL is ignored while in this state.

```
  Process States:
  R (running) → S (sleeping, interruptible) → can be killed
  R (running) → D (sleeping, uninterruptible) → CANNOT be killed
                  ↑
           Waiting for I/O: NFS, hung disk, device driver
```

**Detection:**
```bash
ps aux | awk '$8 == "D"'
cat /proc/<PID>/wchan    # what kernel function it's stuck in
```

**Fix:**
```bash
# Only options:
# 1. Fix the underlying I/O issue (unmount hung NFS, fix disk)
# 2. Reboot
# 3. If it's NFS: umount -f -l /mnt/nfs (lazy unmount)
```

> ⚠️ **DevOps Gotcha:** A single stuck D-state process can cascade — anything trying to wait for it (shell waiting for exit, systemd waiting to stop service) also blocks. This is why `systemctl stop` can hang indefinitely.

---

### EC-018 — Cron Job Silent Failure Due to Missing PATH
**Category:** Process | **Severity:** High | **Env:** Prod

**Scenario:** A cron job runs a script that works perfectly when executed manually but silently fails via cron. No output, no error.

**What breaks:** Backups, reports, cleanup jobs.

**Detection:**
```bash
# Cron's environment is minimal:
env -i bash -c 'printenv'    # simulate cron environment
# Cron only has: HOME, LOGNAME, PATH=/usr/bin:/bin, SHELL=/bin/sh
```

**Fix:**
```bash
# In crontab, always set PATH explicitly:
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
* * * * * /full/path/to/script.sh

# Or source profile in the script:
#!/bin/bash
source /etc/profile
source ~/.bash_profile
```

> ⚠️ **DevOps Gotcha:** Cron doesn't load `.bashrc` or `.bash_profile`. Commands like `docker`, `kubectl`, `aws` that are in `/usr/local/bin` are NOT in cron's default PATH.

---

### EC-019 — `nohup` Process Dies When Terminal Session Ends Anyway
**Category:** Process | **Severity:** Medium | **Env:** Dev

**Scenario:** `nohup ./script.sh &` seems to work, but when SSH session closes the process still dies.

**What breaks:** Long-running jobs that should outlive SSH sessions.

**Detection:**
```bash
ps aux | grep script.sh
# After logout — check if it's still running
```

**Fix:**
```bash
# nohup redirects SIGHUP but the process can still be killed if terminal group
# closes. Use screen or tmux instead:
screen -S myjob ./script.sh
tmux new -d -s myjob './script.sh'

# Or use disown to remove from shell's job table:
./script.sh &
disown %1
```

> ⚠️ **DevOps Gotcha:** `nohup` only protects against SIGHUP. If `huponexit` shell option is set, the shell sends SIGHUP to all jobs when exiting. `disown` removes the job from the shell job table so it's never sent SIGHUP.

---

### EC-020 — `ulimit` Settings Don't Persist Across Reboots
**Category:** Process | **Severity:** High | **Env:** Prod

**Scenario:** You set `ulimit -n 65536` for open file descriptors in the shell. After reboot, the limit is back to 1024. Nginx/Redis/Postgres immediately hit fd limits.

**Detection:**
```bash
cat /proc/<nginx-pid>/limits | grep "open files"
ulimit -n    # current shell only
```

**Fix:**
```bash
# /etc/security/limits.conf:
nginx   soft    nofile  65536
nginx   hard    nofile  65536
*       soft    nofile  65536

# For systemd services, in the unit file:
[Service]
LimitNOFILE=65536

# System-wide:
echo "fs.file-max = 2097152" >> /etc/sysctl.conf
sysctl -p
```

> ⚠️ **DevOps Gotcha:** `/etc/security/limits.conf` only applies to PAM login sessions. systemd services need `LimitNOFILE` in their unit file, as they don't go through PAM.

---

### EC-021 — OOM Killer Picks the Wrong Process
**Category:** Process | **Severity:** Critical | **Env:** Prod

**Scenario:** System runs out of memory. OOM killer selects and kills `sshd` or `systemd` instead of the runaway process.

**What breaks:** You lose remote access to the server.

**Detection:**
```bash
dmesg | grep -i "oom killer"
dmesg | grep -E "Killed process|Out of memory"
grep -i "oom" /var/log/syslog
```

**Fix:**
```bash
# Protect critical processes with oom_score_adj:
echo -1000 > /proc/$(pgrep sshd)/oom_score_adj   # -1000 = never kill

# In systemd unit file:
[Service]
OOMScoreAdjust=-1000

# Increase score of culprit to make it preferred victim:
echo 500 > /proc/<PID>/oom_score_adj
```

> ⚠️ **DevOps Gotcha:** Set `OOMScoreAdjust=-1000` for SSH daemon, systemd, and monitoring agents. Without this, a misbehaving app can kill your only lifeline to the server.

---

### EC-022 — `at` and `batch` Jobs Blocked by `atd` Not Running
**Category:** Process | **Severity:** Low | **Env:** Prod

**Scenario:** `at now + 5 minutes` queues a job but it never runs. No error message.

**Detection:**
```bash
systemctl status atd
atq    # check the queue
```

---

### EC-023 — Fork Bomb Brings Down Server Despite ulimit
**Category:** Process | **Severity:** Critical | **Env:** Prod

**Scenario:** `:(){:|:&};:` or a runaway script forks infinitely. Even with `ulimit -u` set, existing processes fork before the limit is reached.

**Detection:**
```bash
watch -n 0.5 'ps aux | wc -l'   # see process count explode
cat /proc/sys/kernel/pid_max
```

**Fix:**
```bash
# Set in /etc/security/limits.conf:
* hard nproc 1000

# Use cgroups (systemd slice) to limit fork per user:
systemctl set-property user-1000.slice TasksMax=500
```

> ⚠️ **DevOps Gotcha:** Container environments without PID limits are vulnerable. Always set `--pids-limit` in Docker and `resources.limits.pids` in Kubernetes.

---

### EC-024 — Signal Sent to Wrong Process After PID Reuse
**Category:** Process | **Severity:** High | **Env:** Prod

**Scenario:** Script stores PID, waits, then sends SIGTERM. By that time, original process died and PID was reused by a different process (e.g., `/usr/bin/cron`).

**Detection:**
```bash
# Use pidfd (Linux 5.2+) for safe PID handling
# Or verify process identity before signaling:
kill -0 $PID && [ "$(cat /proc/$PID/comm)" = "myapp" ] && kill $PID
```

> ⚠️ **DevOps Gotcha:** PID files in `/var/run` can become stale. Always check that the process name matches before killing. systemd handles this correctly via cgroups tracking.

---

### EC-025 — `nice` Value Set But CPU Priority Unchanged
**Category:** Process | **Severity:** Medium | **Env:** Prod

**Scenario:** `nice -n 19 ./heavy-job.sh` runs but still consumes all CPU. On a lightly loaded system, nice has no effect.

**Detection:**
```bash
ps -l -p <PID>   # NI column shows nice value
# nice only affects priority when CPU is contended
```

> ⚠️ **DevOps Gotcha:** Use `cgroups` (via systemd CPUQuota) instead of nice for hard CPU limits. `nice` is only a hint to the scheduler and has no effect when there's spare CPU capacity.

---

## Users, Permissions & Security

### EC-026 — `sudo` Works for User But Breaks in Cron
**Category:** Permissions | **Severity:** High | **Env:** Prod

**Scenario:** User can run `sudo systemctl restart app` interactively. Same command in a cron job fails with "sudo: no tty present and no askpass program specified".

**Fix:**
```bash
# In /etc/sudoers, add NOPASSWD and NOREQUIRETTY:
Defaults:username !requiretty
username ALL=(ALL) NOPASSWD: /bin/systemctl restart app
```

> ⚠️ **DevOps Gotcha:** `requiretty` is the default sudoers policy on RHEL/CentOS. Always use `NOPASSWD` for automation and explicitly whitelist only the specific commands needed.

---

### EC-027 — ACL Permissions Override Standard Unix Permissions
**Category:** Permissions | **Severity:** Medium | **Env:** Prod

**Scenario:** `ls -la` shows `-rw-r--r--` but a particular user cannot read the file. Or vice versa — a user who shouldn't have access can read it.

**Detection:**
```bash
getfacl /path/to/file   # shows ACL entries
ls -la /path/to/file    # if you see a "+" at the end of perms, ACLs exist
```

**Fix:**
```bash
# Remove all ACLs:
setfacl -b /path/to/file
# Or set specific ACL:
setfacl -m u:username:r-- /path/to/file
```

> ⚠️ **DevOps Gotcha:** `rsync` does NOT copy ACLs by default. Use `rsync -AX` to preserve ACLs and extended attributes. Ansible `copy` module also ignores ACLs.

---

### EC-028 — `umask` Different Between SSH and Console Login
**Category:** Permissions | **Severity:** Medium | **Env:** Prod

**Scenario:** Files created via SSH session have `644`, but files created by the same user via su/sudo have `022`. Causes permission inconsistencies.

**Detection:**
```bash
ssh user@host 'umask'
su -c 'umask' user
```

**Fix:**
```bash
# Set umask consistently in /etc/profile, /etc/bashrc, and ~/.bashrc:
umask 022
# For more restrictive: umask 027 (group can't write, others get nothing)
```

---

### EC-029 — SELinux/AppArmor Silently Blocking a Service
**Category:** Security | **Severity:** High | **Env:** Prod

**Scenario:** Service fails to start or behaves incorrectly. `journalctl -u service` shows no useful errors. The real error is in SELinux audit log.

**Detection:**
```bash
# SELinux:
ausearch -m AVC -ts recent | grep denied
sealert -a /var/log/audit/audit.log
getenforce    # Enforcing / Permissive / Disabled

# AppArmor:
dmesg | grep apparmor
aa-status
```

**Fix:**
```bash
# Quick (don't do in prod permanently):
setenforce 0   # set permissive mode

# Proper fix — generate and apply a policy:
ausearch -m AVC | audit2allow -M mypolicy
semodule -i mypolicy.pp
```

> ⚠️ **DevOps Gotcha:** NEVER permanently disable SELinux in production. Use `audit2allow` to generate proper policies. Many CIS benchmarks require SELinux enforcing.

---

### EC-030 — Group Membership Change Requires New Login Session
**Category:** Permissions | **Severity:** Medium | **Env:** Prod

**Scenario:** Add user to `docker` group with `usermod -aG docker username`. User tries `docker ps` immediately and gets "permission denied". 

**Detection:**
```bash
groups           # shows current session's groups
id               # shows current session's GID list
# The new group won't appear until next login
```

**Fix:**
```bash
# Without logout:
newgrp docker    # opens new shell with docker group active
# Or:
su - $USER       # creates new login session
```

> ⚠️ **DevOps Gotcha:** Ansible `user` module adds groups correctly but the *running Ansible connection* to that host won't see the new group. Restarting services managed by that user may require a full session restart.

---

### EC-031 — Sticky Bit Misunderstood on Files vs Directories
**Category:** Permissions | **Severity:** Medium | **Env:** Prod

**Scenario:** Someone sets sticky bit on a file thinking it prevents deletion. Sticky bit on a file is mostly obsolete and ignored by modern kernels.

**What sticky bit actually does:**
```
  On a DIRECTORY (the useful case):
  /tmp (drwxrwxrwt)
     Only the file owner (or root) can delete their own files.
     Other users cannot delete each other's files even though
     directory is world-writable.

  On a FILE (legacy/ignored):
     Historically kept executable in swap. Modern kernels ignore it.
```

---

### EC-032 — SUID Bit Set on Shell Script (Does Nothing)
**Category:** Security | **Severity:** Medium | **Env:** Prod

**Scenario:** Admin sets SUID on a bash script thinking it will run as root. Linux kernels ignore SUID on interpreted scripts (security feature).

**Detection:**
```bash
ls -la /usr/local/bin/myscript.sh   # -rwsr-xr-x (SUID set but ineffective)
```

> ⚠️ **DevOps Gotcha:** SUID works on compiled ELF binaries only. For privilege escalation in scripts, use `sudo` with NOPASSWD for specific commands, or write a small SUID C wrapper.

---

### EC-033 — `/etc/shadow` Permissions Break PAM Authentication
**Category:** Security | **Severity:** Critical | **Env:** Prod

**Scenario:** Someone accidentally runs `chmod 644 /etc/shadow`. All PAM authentication breaks — no one can log in.

**Detection:**
```bash
ls -la /etc/shadow    # should be ---------- (000) or -r-------- (400)
```

**Fix:**
```bash
chmod 000 /etc/shadow    # or:
chmod 640 /etc/shadow    # some distros use 640 for shadow group
chown root:shadow /etc/shadow
```

---

### EC-034 — `visudo` Syntax Error Locks Out All sudo Access
**Category:** Security | **Severity:** Critical | **Env:** Prod

**Scenario:** Edit `/etc/sudoers` directly with `vi` (not `visudo`), introduce a syntax error, save. Now `sudo` is completely broken — no one can gain root.

**Fix:**
```bash
# If you still have a root shell open:
pkexec visudo    # graphical sudo alternative
# Or boot into recovery mode and fix from there
# Always use: visudo -f /etc/sudoers
```

> ⚠️ **DevOps Gotcha:** `visudo` validates syntax before saving. Direct editing with `vi`/`nano` does not. This is one of the most common self-lockout scenarios in production.

---

### EC-035 — World-Writable `/etc/cron.d` Scripts Run as Root
**Category:** Security | **Severity:** Critical | **Env:** Prod

**Scenario:** Package install creates a cron file in `/etc/cron.d/` owned by app user with 666 permissions. Any user can overwrite it and execute code as root.

**Detection:**
```bash
find /etc/cron* /var/spool/cron -not -user root -ls
find /etc/cron.d -perm -o+w -ls
```

**Fix:**
```bash
chmod 644 /etc/cron.d/*
chown root:root /etc/cron.d/*
```

> ⚠️ **DevOps Gotcha:** crond refuses to run cron files in `/etc/cron.d` if they are writable by anyone other than root. This is a security AND reliability issue.

---

## Networking

### EC-036 — TIME_WAIT Exhaustion Blocks New Connections
**Category:** Networking | **Severity:** Critical | **Env:** Prod

**Scenario:** High-traffic server exhausts local port range due to thousands of sockets in TIME_WAIT state. New outbound connections fail with "Cannot assign requested address".

```
  Client port range: 32768–60999 (28,231 ports)
  Each connection uses 1 port for TIME_WAIT duration (default: 2 * MSL = 120s)
  At 235 req/sec you exhaust all ports!
```

**Detection:**
```bash
ss -s | grep TIME-WAIT
cat /proc/net/sockstat | grep TCP
ss -tan state time-wait | wc -l
```

**Fix:**
```bash
# Enable port reuse:
sysctl -w net.ipv4.tcp_tw_reuse=1
# Reduce local port range:
sysctl -w net.ipv4.ip_local_port_range="1024 65535"
# Do NOT use tcp_tw_recycle — it's removed in Linux 4.12+ and breaks NAT
```

> ⚠️ **DevOps Gotcha:** `tcp_tw_recycle` was removed in kernel 4.12. Setting it on modern kernels does nothing. `tcp_tw_reuse=1` is the safe option for client-side sockets.

---

### EC-037 — DNS Search Domain Causes Wrong Service Resolution
**Category:** Networking | **Severity:** High | **Env:** Prod

**Scenario:** Application trying to reach `db` resolves to `db.internal.company.com` instead of the intended `db.production.svc.cluster.local` in Kubernetes.

**Detection:**
```bash
cat /etc/resolv.conf
# search internal.company.com svc.cluster.local cluster.local
# The order of search domains matters!
```

> ⚠️ **DevOps Gotcha:** In Kubernetes, `resolv.conf` has `ndots:5`. Any hostname with fewer than 5 dots gets search domains appended first. `curl http://db/api` will try `db.default.svc.cluster.local` before `db` as a bare name.

---

### EC-038 — Asymmetric Routing Breaks Stateful Firewall
**Category:** Networking | **Severity:** High | **Env:** Prod

**Scenario:** Traffic enters on eth0 but returns on eth1 (two ISPs, or VPC with multiple routes). Stateful iptables drops the return packets because conntrack didn't see the outbound.

```
  Client → eth0 → Server (conntrack tracks this)
  Server → eth1 → Client (conntrack: UNKNOWN connection → DROP)
```

**Detection:**
```bash
ip route show table main
ip rule show    # policy routing rules
conntrack -L | grep UNREPLIED
```

**Fix:**
```bash
# Use policy routing — force return traffic through same interface:
ip rule add fwmark 1 table 100
ip route add default via <gw1> table 100
iptables -t mangle -A PREROUTING -i eth0 -j MARK --set-mark 1
```

---

### EC-039 — `iptables -F` Drops All Traffic (Default Policy = DROP)
**Category:** Networking | **Severity:** Critical | **Env:** Prod

**Scenario:** Running `iptables -F` to clear rules when default policy is DROP. All traffic immediately blocked including SSH — you lock yourself out.

```
  Before -F:          After -F:
  Policy: DROP        Policy: DROP  ← still DROP!
  Rule 1: ACCEPT SSH  (no rules)
  Rule 2: ACCEPT HTTP ← -F removes rules but NOT policy
```

**Fix:**
```bash
# Always set permissive policy BEFORE flushing:
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT
iptables -F
iptables -X
```

> ⚠️ **DevOps Gotcha:** This is the #1 cause of self-lockout via iptables. In AWS, Security Groups are stateful and separate from iptables — you can recover via the console. On bare metal, you may need physical access.

---

### EC-040 — MTU Mismatch Causes Intermittent Large Packet Drops
**Category:** Networking | **Severity:** High | **Env:** Prod

**Scenario:** Small HTTP requests work fine. Large file uploads, TLS handshakes, or SSH sessions hang. The issue is PMTUD (Path MTU Discovery) being blocked by a firewall.

```
  Client (MTU 1500) → Router (MTU 1400) → Server
                          ↑
                    Sends ICMP "frag needed" back to client
                    But firewall blocks ICMP → client never learns MTU
                    → Large packets get dropped silently
```

**Detection:**
```bash
ping -M do -s 1400 <host>    # test with Don't Fragment bit
tcpdump -i eth0 icmp
```

**Fix:**
```bash
# Set MSS clamping on the router/firewall:
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -j TCPMSS --clamp-mss-to-pmtu

# Or explicitly set interface MTU:
ip link set eth0 mtu 1400
```

> ⚠️ **DevOps Gotcha:** VPN tunnels (WireGuard, OpenVPN, IPIP) add headers that reduce effective MTU. Always set MTU 60-80 bytes lower on tunnel interfaces.

---

### EC-041 — `localhost` Resolves Differently in IPv6-Enabled Systems
**Category:** Networking | **Severity:** Medium | **Env:** Dev/Prod

**Scenario:** Application listens on `0.0.0.0:8080`. `curl localhost:8080` fails because `localhost` resolves to `::1` (IPv6) but the app only listens on IPv4.

**Detection:**
```bash
getent hosts localhost   # shows both ::1 and 127.0.0.1
curl -v localhost:8080   # shows which IP it tries
```

**Fix:**
```bash
# Explicitly specify IPv4:
curl http://127.0.0.1:8080
# Or make the app listen on both:
app --listen 0.0.0.0:8080  # IPv4
app --listen [::]:8080     # IPv4+IPv6 (dual-stack)
```

---

### EC-042 — ARP Cache Poisoning by Duplicate IPs
**Category:** Networking | **Severity:** High | **Env:** Prod

**Scenario:** Two machines on the same subnet have the same IP (misconfiguration or DHCP conflict). ARP cache flaps between the two MACs. Both machines intermittently lose connectivity.

**Detection:**
```bash
arp -n | sort              # look for duplicate IPs with different MACs
arping -I eth0 <suspect-ip>    # check if multiple MACs respond
journalctl | grep "duplicate address"
```

---

### EC-043 — `ss`/`netstat` Shows Port Listening But Service Unreachable
**Category:** Networking | **Severity:** High | **Env:** Prod

**Scenario:** `ss -tlnp | grep 8080` shows service listening on `127.0.0.1:8080`. External connections are rejected because it's bound to loopback only.

**Detection:**
```bash
ss -tlnp | grep 8080
# 127.0.0.1:8080 vs 0.0.0.0:8080 — critical difference!
```

> ⚠️ **DevOps Gotcha:** Many services default to `127.0.0.1` for security. Prometheus, Grafana, Jenkins all do this. In Docker, you must map `0.0.0.0:9090:9090` not `127.0.0.1:9090:9090` if you want external access.

---

### EC-044 — `sysctl` Network Settings Reset on Interface Bounce
**Category:** Networking | **Severity:** Medium | **Env:** Prod

**Scenario:** `sysctl -w net.ipv4.conf.eth0.rp_filter=0` set at runtime. After `ip link set eth0 down/up`, the setting resets to default.

**Fix:**
```bash
# Persist in /etc/sysctl.d/99-network.conf:
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0
sysctl -p /etc/sysctl.d/99-network.conf
```

---

### EC-045 — Conntrack Table Full Causes All New Connections to Fail
**Category:** Networking | **Severity:** Critical | **Env:** Prod

**Scenario:** Under high load, the netfilter conntrack table fills up. `dmesg` shows "nf_conntrack: table full, dropping packet". All new connections are rejected.

**Detection:**
```bash
cat /proc/net/nf_conntrack | wc -l
sysctl net.netfilter.nf_conntrack_max
sysctl net.netfilter.nf_conntrack_count
```

**Fix:**
```bash
sysctl -w net.netfilter.nf_conntrack_max=524288
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=300
# Add to /etc/sysctl.d/ to persist
```

> ⚠️ **DevOps Gotcha:** Every Kubernetes node running kube-proxy (iptables mode) uses conntrack. High-traffic nodes exhaust the default conntrack table quickly. This is a common cause of intermittent connection failures in K8s clusters.

---

## SSH & Remote Access

### EC-046 — SSH Hangs on `debug1: expecting SSH2_MSG_KEX_ECDH_REPLY`
**Category:** SSH | **Severity:** High | **Env:** Prod

**Scenario:** SSH connection hangs for 30–60 seconds during key exchange. Eventually connects or times out.

**What causes it:** DNS reverse lookup on the client IP taking too long on the server side.

**Detection:**
```bash
time ssh -vvv user@host 2>&1 | head -30   # look for where it hangs
# On server: /etc/ssh/sshd_config
grep UseDNS /etc/ssh/sshd_config
```

**Fix:**
```bash
# In /etc/ssh/sshd_config:
UseDNS no
GSSAPIAuthentication no   # also causes delays if Kerberos not configured
```

---

### EC-047 — SSH `ControlMaster` Shared Socket Left Stale
**Category:** SSH | **Severity:** Medium | **Env:** Dev/Prod

**Scenario:** With `ControlMaster auto` in `~/.ssh/config`, a stale socket file blocks all new SSH connections to that host after the master connection dies abnormally.

**Detection:**
```bash
ls ~/.ssh/control/    # stale socket files
ssh -vvv user@host    # shows "Control socket connect(...): Connection refused"
```

**Fix:**
```bash
rm ~/.ssh/control/user@host*
# Or use the -S flag to specify a different socket
ssh -S none user@host
```

---

### EC-048 — Public Key Works Locally But Not via Ansible/CI
**Category:** SSH | **Severity:** High | **Env:** Prod

**Scenario:** `ssh user@host` works manually. Ansible/CI runner fails with "Permission denied (publickey)".

**Common Causes:**
```
  1. ~/.ssh/authorized_keys permissions wrong (should be 600)
  2. ~/.ssh/ directory permissions wrong (should be 700)
  3. Home directory world-writable (sshd refuses it)
  4. SELinux context wrong on authorized_keys
  5. CI runner uses different key than manually tested
```

**Detection:**
```bash
ssh -vvv user@host    # verbose output shows exactly which key was tried
# On server: tail -f /var/log/auth.log  (or /var/log/secure)
```

---

### EC-049 — Port Forwarding Fails With "bind: Address already in use"
**Category:** SSH | **Severity:** Medium | **Env:** Dev

**Scenario:** `ssh -L 8080:localhost:80 user@host` fails because a previous SSH tunnel is still holding port 8080.

**Detection:**
```bash
ss -tlnp | grep 8080
lsof -i :8080
```

**Fix:**
```bash
# Kill the old tunnel:
pkill -f "ssh.*8080"
# Or use a different local port:
ssh -L 18080:localhost:80 user@host
```

---

### EC-050 — SSH Agent Forwarding Leaks Private Key Access
**Category:** SSH | **Severity:** High | **Env:** Prod

**Scenario:** `ssh -A user@jumphost` forwards your SSH agent. Anyone with root on `jumphost` can use your agent socket to authenticate as you to other servers.

```
  Your laptop → jumphost (compromised root) → production
      ↑                    ↓
  SSH_AUTH_SOCK        root reads your
  forwarded             agent socket
                        → impersonates you to prod!
```

> ⚠️ **DevOps Gotcha:** Never use `-A` to hosts you don't fully trust. Use `ProxyJump` (`-J`) instead — it creates a direct encrypted tunnel without exposing the agent socket to the jump host.

---

### EC-051 — `MaxSessions` Exhausted — New SSH Connections Queued Forever
**Category:** SSH | **Severity:** High | **Env:** Prod

**Scenario:** `MaxSessions 10` in sshd_config. Ansible runs with 10+ forks all SSHing to same host. Extra connections queue but timeout.

**Fix:**
```bash
# In /etc/ssh/sshd_config:
MaxSessions 100
MaxStartups 100:30:200   # allow more unauthenticated connections
```

---

### EC-052 — SSH Host Key Changed Warning Is Ignored (MITM Risk)
**Category:** SSH | **Severity:** Critical | **Env:** Prod

**Scenario:** Server reinstalled, new host key generated. Automation scripts use `StrictHostKeyChecking no` to avoid "WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED". This silently opens you to MITM attacks.

> ⚠️ **DevOps Gotcha:** Never use `StrictHostKeyChecking no` in production automation. Instead, pre-populate `known_hosts` via a trusted channel (use `ssh-keyscan` during provisioning and commit to your infra repo).

---

### EC-053 — SSH Multiplexing Sends Commands to Wrong Session
**Category:** SSH | **Severity:** Medium | **Env:** Dev

**Scenario:** With ControlMaster, running `ssh user@host command1` while another session is open reuses the existing connection. Environment variables from the old session may persist.

---

### EC-054 — `sshd` Fails to Start Because of `/etc/ssh/sshd_config` Syntax Error
**Category:** SSH | **Severity:** Critical | **Env:** Prod

**Scenario:** You edit `/etc/ssh/sshd_config` and add a typo. `systemctl restart sshd` fails. Your existing SSH session is still alive, but when it drops no new connections will work.

**Prevention:**
```bash
# Always validate sshd config before restarting:
sshd -t    # dry-run syntax check
# Or:
sshd -T    # print effective configuration
```

> ⚠️ **DevOps Gotcha:** Always keep an existing SSH session open while testing sshd config changes. Only restart after validating with `sshd -t`. Do this BEFORE closing your existing session.

---

### EC-055 — `authorized_keys` with `command=` Option Restricts to Single Command
**Category:** SSH | **Severity:** Medium | **Env:** Prod

**Scenario:** Key in `authorized_keys` has `command="backup.sh"` option. SSH login from that key ALWAYS runs `backup.sh` regardless of what command the client requests.

```
# /root/.ssh/authorized_keys
command="/usr/local/bin/backup.sh",no-pty,no-x11-forwarding ssh-rsa AAAA...
```

> This is actually a security feature — use it to restrict deployment keys to specific commands (e.g., git push only).

---

## systemd & Services

### EC-056 — `systemctl restart` Doesn't Reload Config (vs `reload`)
**Category:** systemd | **Severity:** Medium | **Env:** Prod

**Scenario:** Running `systemctl restart nginx` causes brief downtime (stops then starts). Using `systemctl reload nginx` sends SIGHUP and reloads config with zero downtime.

```
  restart:  STOP (all connections dropped) → START
  reload:   SIGHUP → graceful config reload (connections maintained)
```

> ⚠️ **DevOps Gotcha:** Not all services support `reload`. Check with `systemctl show nginx | grep ExecReload`. If empty, reload is not supported and restart is the only option.

---

### EC-057 — Systemd Unit `After=` Doesn't Mean `Requires=`
**Category:** systemd | **Severity:** High | **Env:** Prod

**Scenario:** Unit file has `After=network.target` but not `Requires=network.target`. If the network is never available, the service starts anyway and fails. `After` only means ordering, not dependency.

**Fix:**
```ini
[Unit]
After=network-online.target
Wants=network-online.target     # Wants = start it, but don't fail if unavailable
# or:
Requires=network-online.target  # Requires = fail if dependency fails
```

> ⚠️ **DevOps Gotcha:** `network.target` means "network configuration has been applied". `network-online.target` means "at least one interface has an IP". For services that need connectivity, use `network-online.target`.

---

### EC-058 — Service Silently Exits Due to Missing `Type=notify` or Wrong PIDFile
**Category:** systemd | **Severity:** High | **Env:** Prod

**Scenario:** systemd starts a daemon, considers it running (Active: active (running)), but the daemon never completed initialization. Type mismatch causes this.

```
  Type=simple    → systemd assumes started as soon as exec() is called
  Type=forking   → systemd waits for parent to exit (parent forks child)
  Type=notify    → daemon sends sd_notify() when ready
  Type=oneshot   → process must exit with 0 for "success"
```

> ⚠️ **DevOps Gotcha:** Using `Type=simple` for a forking daemon means systemd tracks the wrong PID. The main process is the fork parent which exits immediately, leaving the child as an orphan outside systemd's supervision.

---

### EC-059 — `ExecStop` Not Running Because Service Crashed
**Category:** systemd | **Severity:** Medium | **Env:** Prod

**Scenario:** `ExecStop` cleanup script never runs because the service process was killed (SIGKILL) rather than stopping gracefully. `ExecStop` only runs when the service is explicitly stopped.

**Fix:**
```ini
[Service]
ExecStopPost=/usr/local/bin/cleanup.sh   # runs ALWAYS, even after crash
```

---

### EC-060 — `Restart=always` Creates a Restart Loop That Consumes Resources
**Category:** systemd | **Severity:** High | **Env:** Prod

**Scenario:** Misconfigured service with `Restart=always` enters a rapid restart loop, consuming CPU and flooding logs. systemd rate-limits after 5 restarts in 10 seconds but the service still restarts continuously.

**Detection:**
```bash
systemctl status myservice   # shows "start request repeated too quickly"
journalctl -u myservice --since "5 min ago" | grep -c "Started"
```

**Fix:**
```ini
[Service]
Restart=on-failure
RestartSec=5s
StartLimitIntervalSec=60s
StartLimitBurst=3     # only try 3 times in 60 seconds, then give up
```

---

### EC-061 — `journalctl` Log Rotation Deletes Logs Before Analysis
**Category:** systemd | **Severity:** Medium | **Env:** Prod

**Scenario:** `journalctl --vacuum-size=100M` runs automatically and deletes old logs right when you need them for incident analysis.

**Fix:**
```bash
# In /etc/systemd/journald.conf:
Storage=persistent     # write to /var/log/journal (survives reboots)
SystemMaxUse=2G
SystemKeepFree=500M
MaxRetentionSec=30day
```

---

### EC-062 — Service Starts Before Filesystem Is Mounted
**Category:** systemd | **Severity:** Critical | **Env:** Prod

**Scenario:** Service starts, writes to `/data/logs/`, but `/data` hasn't been mounted yet (NFS or LVM). Writes go to the root filesystem instead.

**Fix:**
```ini
[Unit]
RequiresMountsFor=/data   # wait until /data is mounted
```

> ⚠️ **DevOps Gotcha:** `RequiresMountsFor=` is one of the most underused systemd directives. Any service writing to a non-root mount point should use it.

---

### EC-063 — `EnvironmentFile` With Wrong Syntax Silently Ignores Variables
**Category:** systemd | **Severity:** Medium | **Env:** Prod

**Scenario:** `EnvironmentFile=/etc/app/env` has variables like `export DB_HOST=prod-db`. systemd ignores lines starting with `export`. Variables are never set.

**Fix:**
```bash
# Correct format (no 'export', no quotes unless needed):
DB_HOST=prod-db
DB_PORT=5432
DB_PASS="p@ssw0rd with spaces"
```

---

### EC-064 — `systemctl daemon-reload` Required After Unit File Change
**Category:** systemd | **Severity:** Medium | **Env:** Prod

**Scenario:** Edit a `.service` file in `/etc/systemd/system/`. Run `systemctl restart service`. systemd uses the old unit definition.

**Fix:**
```bash
systemctl daemon-reload
systemctl restart service
```

> ⚠️ **DevOps Gotcha:** Ansible `systemd` module handles this automatically if you set `daemon_reload: true`. Without it, your config changes won't take effect.

---

### EC-065 — `WantedBy=multi-user.target` But Service Never Auto-Starts
**Category:** systemd | **Severity:** Medium | **Env:** Prod

**Scenario:** Unit file has `WantedBy=multi-user.target` but service doesn't auto-start on boot.

**What's missing:** `systemctl enable myservice` must be run to create the symlink in `multi-user.target.wants/`.

```bash
# Enable (creates symlink):
systemctl enable myservice
# Verify:
ls /etc/systemd/system/multi-user.target.wants/
```

---

## Package Management

### EC-066 — Partial Upgrade Leaves Package Dependencies Broken
**Category:** Package Management | **Severity:** High | **Env:** Prod

**Scenario:** `apt upgrade` is interrupted (network loss, disk full). dpkg database is left in a broken state. Subsequent installs fail.

**Detection:**
```bash
dpkg --audit
apt-get -f install   # attempt to fix broken deps
dpkg --configure -a
```

**Fix:**
```bash
dpkg --configure -a
apt-get -f install
apt-get autoremove
apt-get update && apt-get dist-upgrade
```

---

### EC-067 — `yum/dnf update` Kernel Update Doesn't Apply Until Reboot
**Category:** Package Management | **Severity:** Medium | **Env:** Prod

**Scenario:** `yum update kernel` completes successfully. Running kernel version unchanged. Team assumes the system is patched.

**Detection:**
```bash
uname -r                                        # running kernel
rpm -q --last kernel | head -3                  # installed kernels
needs-restarting -r                             # check if reboot needed (RHEL/CentOS)
```

> ⚠️ **DevOps Gotcha:** In your CI/CD health checks, verify BOTH the installed AND running kernel versions. Consider using `kpatch` or `livepatch` for zero-downtime kernel patching on critical systems.

---

### EC-068 — Pinned Package Version Blocks Security Updates
**Category:** Package Management | **Severity:** High | **Env:** Prod

**Scenario:** `/etc/apt/preferences.d/` has a pin for `nginx=1.18.0`. A critical CVE requires upgrading to 1.20.x but `apt upgrade` silently skips nginx.

**Detection:**
```bash
apt-cache policy nginx       # shows pin priority and available versions
apt-mark showhold            # shows held packages
```

**Fix:**
```bash
apt-mark unhold nginx
apt-get install nginx=1.20.0-1
```

---

### EC-069 — Third-Party Repo GPG Key Expiry Breaks Package Install
**Category:** Package Management | **Severity:** Medium | **Env:** Prod

**Scenario:** `apt update` fails with "The following signatures were invalid: EXPKEYSIG". Usually happens for Nginx, Docker, or HashiCorp repos.

**Fix:**
```bash
apt-key list | grep -A1 "expired"
# Fetch updated key:
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor | apt-key add -
apt update
```

---

### EC-070 — `snap` Package Uses Different Binary Path Than Apt
**Category:** Package Management | **Severity:** Medium | **Env:** Dev/Prod

**Scenario:** `apt install kubectl` installs to `/usr/bin/kubectl`. `snap install kubectl` installs to `/snap/bin/kubectl`. Scripts using `which kubectl` or hardcoded paths break.

**Detection:**
```bash
which kubectl
type -a kubectl    # shows all locations
```

---

### EC-071 — Conflicting Package Versions Between System and pip/npm
**Category:** Package Management | **Severity:** High | **Env:** Dev/Prod

**Scenario:** System Python has `requests==2.18`. `pip install` upgrades to 2.28. Now `yum`/`apt` operations fail because they import system Python packages.

**Fix:**
```bash
# Never use pip without a virtualenv in Python:
python3 -m venv /opt/myapp/venv
source /opt/myapp/venv/bin/activate
pip install requests==2.28
```

> ⚠️ **DevOps Gotcha:** Never run `pip install` as root without a virtualenv on production systems. It can corrupt the system Python that package managers depend on.

---

### EC-072 — `dpkg -l` Shows `rc` Status — Package "Removed" But Config Remains
**Category:** Package Management | **Severity:** Low | **Env:** Prod

**Scenario:** `dpkg -l nginx` shows `rc` (removed but config left). Old config files conflict with fresh install.

**Fix:**
```bash
dpkg -l | grep "^rc"          # list all removed-with-config packages
dpkg --purge $(dpkg -l | grep "^rc" | awk '{print $2}')   # purge configs
# Or: apt-get purge nginx
```

---

### EC-073 — `rpm -e` Removes Shared Library Breaking Multiple Apps
**Category:** Package Management | **Severity:** Critical | **Env:** Prod

**Scenario:** `rpm -e libssl` to remove old OpenSSL. Multiple apps (`nginx`, `openssh`, `curl`) silently start failing when next invoked.

**Detection:**
```bash
rpm -q --whatrequires libssl   # check dependents before removing
ldd /usr/sbin/nginx | grep libssl   # check binary dependencies
```

> ⚠️ **DevOps Gotcha:** Always use `rpm -e --test` before actual removal to see what would break. `yum remove` is safer as it resolves the dependency tree.

---

### EC-074 — Package Cache Causes Stale Version Install in Docker
**Category:** Package Management | **Severity:** High | **Env:** Prod

**Scenario:** Dockerfile caches `RUN apt-get update` separately from `apt-get install`. On a later build, the install layer runs with a stale cache and installs old packages.

```dockerfile
# WRONG — cache busting issue:
RUN apt-get update            # cached layer from months ago
RUN apt-get install nginx     # uses stale package list!

# CORRECT — always combine:
RUN apt-get update && apt-get install -y nginx && rm -rf /var/lib/apt/lists/*
```

---

### EC-075 — `ansible` Uses Wrong Python Interpreter
**Category:** Package Management | **Severity:** Medium | **Env:** Prod

**Scenario:** Target host has Python 2 at `/usr/bin/python` and Python 3 at `/usr/bin/python3`. Ansible uses Python 2 by default, and a module requires Python 3 features.

**Fix:**
```yaml
# In inventory or playbook:
ansible_python_interpreter: /usr/bin/python3
# Or auto-detect:
ansible_python_interpreter: auto
```

---

## Shell & Scripting

### EC-076 — Unquoted Variable With Spaces Causes Word Splitting
**Category:** Shell | **Severity:** High | **Env:** Prod

**Scenario:** `rm $FILE` where `FILE="/path/to/my file.txt"`. Shell splits on space and runs `rm /path/to/my` and `file.txt` separately.

```bash
FILE="important file.txt"
rm $FILE        # WRONG — splits into 2 arguments
rm "$FILE"      # CORRECT — treats as single argument
```

> ⚠️ **DevOps Gotcha:** Always quote variables in shell scripts. Use `shellcheck` in your CI pipeline to catch unquoted variables automatically.

---

### EC-077 — `set -e` Doesn't Exit on Pipe Failures
**Category:** Shell | **Severity:** High | **Env:** Prod

**Scenario:** `set -e` in script. `false | true` — the pipe returns 0 (exit code of last command). Script doesn't exit even though `false` failed.

**Fix:**
```bash
set -e
set -o pipefail    # exit if ANY command in a pipe fails
set -u             # exit on unset variable reference
# Together: set -euo pipefail (recommended for all scripts)
```

---

### EC-078 — `[` vs `[[` Behavioral Differences
**Category:** Shell | **Severity:** Medium | **Env:** Dev/Prod

**Scenario:** `[ $var = "" ]` fails with "too many arguments" when `$var` contains spaces. `[[ $var = "" ]]` handles it correctly.

```bash
VAR="hello world"
[ $VAR = "hello" ]    # Error: too many arguments (word splits "hello world")
[[ $VAR = "hello" ]]  # Works: no word splitting inside [[
```

---

### EC-079 — Bash Array vs String Confusion With Spaces
**Category:** Shell | **Severity:** Medium | **Env:** Dev/Prod

**Scenario:** Building a command with arguments that may contain spaces. String concatenation breaks argument boundaries.

```bash
# WRONG:
ARGS="--config /path/with spaces/config.yml --debug"
myapp $ARGS    # splits incorrectly

# CORRECT:
ARGS=(--config "/path/with spaces/config.yml" --debug)
myapp "${ARGS[@]}"    # preserves argument boundaries
```

---

### EC-080 — `read` Without `-r` Mangles Backslashes
**Category:** Shell | **Severity:** Low | **Env:** Dev/Prod

**Scenario:** `read line` from a file that contains backslashes. Backslashes are consumed as escape characters.

```bash
# WRONG:
while read line; do echo "$line"; done < file.txt

# CORRECT:
while IFS= read -r line; do echo "$line"; done < file.txt
# IFS= preserves leading/trailing whitespace
# -r preserves backslashes
```

---

### EC-081 — Heredoc Indentation Fails With Tabs vs Spaces
**Category:** Shell | **Severity:** Low | **Env:** Dev/Prod

**Scenario:** `<<-EOF` strips leading TABS but not SPACES. Editor that converts tabs to spaces breaks heredoc indentation stripping.

```bash
# <<-EOF only strips literal tab characters:
cat <<-EOF
	This line starts with a tab — stripped
    This line starts with spaces — NOT stripped
EOF
```

---

### EC-082 — `exit` in a Subshell Doesn't Exit the Parent
**Category:** Shell | **Severity:** Medium | **Env:** Dev/Prod

**Scenario:** `(exit 1)` or `$(command_that_calls_exit)` — `exit` inside a subshell only exits the subshell, not the parent.

```bash
set -e
(exit 1)     # exits subshell, parent continues!
# Must check $?:
(exit 1); ret=$?; [ $ret -eq 0 ] || exit $ret
```

---

### EC-083 — Globbing Fails When No Files Match (nullglob not set)
**Category:** Shell | **Severity:** Medium | **Env:** Dev/Prod

**Scenario:** `for f in /tmp/app-*.log; do process "$f"; done` — if no files match, `$f` is literally the string `/tmp/app-*.log`.

```bash
# Enable nullglob so glob expands to nothing if no match:
shopt -s nullglob
for f in /tmp/app-*.log; do
    process "$f"
done
```

---

### EC-084 — `trap` Not Inherited by Functions or Subshells
**Category:** Shell | **Severity:** Medium | **Env:** Dev/Prod

**Scenario:** `trap cleanup EXIT` set in main script. Cleanup function calls a function that uses `( )` subshell. Subshell doesn't run the trap.

> ⚠️ **DevOps Gotcha:** For reliable cleanup in complex scripts, use `trap` at every level or use a cleanup function called at each exit point. `trap` is inherited by shell functions but NOT by subshells created with `( )`.

---

### EC-085 — `grep` Returns Exit Code 1 With `set -e` When No Match
**Category:** Shell | **Severity:** High | **Env:** Dev/Prod

**Scenario:** `set -e` script. `grep "pattern" file` finds no match, returns exit code 1. Script aborts unexpectedly.

```bash
# Solutions:
grep "pattern" file || true          # ignore grep's exit code
grep -q "pattern" file && do_thing   # conditional without abort
count=$(grep -c "pattern" file || true)   # get count, never fail
```

---

## Performance & Resource Management

### EC-086 — High CPU But Low `top` Utilization (IRQ Softirq)
**Category:** Performance | **Severity:** High | **Env:** Prod

**Scenario:** System feels sluggish, `top` shows CPU mostly idle. The real CPU usage is in `si` (softirq) from network interrupt processing.

**Detection:**
```bash
top    # watch 'si' and 'hi' columns (softirq and hardware interrupt)
cat /proc/interrupts | sort -t' ' -k2 -rn | head -10
mpstat -P ALL 1 5   # per-CPU breakdown
```

**Fix:**
```bash
# Distribute network interrupts across CPUs:
ethtool -L eth0 combined 8    # set number of queues = number of CPUs
# Enable IRQ balancing:
systemctl start irqbalance
```

---

### EC-087 — Memory Pressure Causes Swap Thrashing Despite Free RAM
**Category:** Performance | **Severity:** High | **Env:** Prod

**Scenario:** System has 4GB free RAM but is heavily swapping. `vm.swappiness=60` causes the kernel to swap pages even when memory is available.

**Detection:**
```bash
vmstat 1 10 | awk '{print $7, $8}'   # si=swap in, so=swap out
free -m
sysctl vm.swappiness
```

**Fix:**
```bash
sysctl -w vm.swappiness=10    # 10 is good for most servers; 0 for databases
echo "vm.swappiness = 10" >> /etc/sysctl.d/99-performance.conf
```

> ⚠️ **DevOps Gotcha:** Set `vm.swappiness=1` for Redis, Elasticsearch, and other memory-intensive services. Swapped pages cause latency spikes that are hard to diagnose.

---

### EC-088 — iowait High But Not All Disks Busy
**Category:** Performance | **Severity:** High | **Env:** Prod

**Scenario:** `iostat` shows high iowait on one disk. Other disks are idle. Application performance degraded even though only one component uses that disk.

**Detection:**
```bash
iostat -xz 1 5    # per-device utilization, %util and await columns
iotop -ob         # per-process I/O usage
```

---

### EC-089 — `transparent_hugepages=always` Causes Latency Spikes
**Category:** Performance | **Severity:** High | **Env:** Prod

**Scenario:** Database (MongoDB, Redis, PostgreSQL) experiences random latency spikes every few minutes. Root cause: THP defragmentation pauses.

**Detection:**
```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
# [always] madvise never  ← "always" is the problem
```

**Fix:**
```bash
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
# Persist via systemd:
cat > /etc/systemd/system/disable-thp.service << EOF
[Unit]
Description=Disable Transparent Huge Pages
[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/enabled'
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/defrag'
[Install]
WantedBy=multi-user.target
EOF
systemctl enable --now disable-thp
```

> ⚠️ **DevOps Gotcha:** AWS, Google Cloud, and MongoDB all document this as a required configuration. THP is the #1 cause of unexplained latency spikes in database workloads.

---

### EC-090 — Too Many Open Files Throttles Performance Before Hitting Limit
**Category:** Performance | **Severity:** Medium | **Env:** Prod

**Scenario:** Service slows down significantly at ~800 connections even though the file descriptor limit is 1024. Linux reserves some FDs for system operations.

**Detection:**
```bash
ls /proc/<PID>/fd | wc -l
cat /proc/<PID>/limits | grep "open files"
```

---

### EC-091 — CPU Throttling in Container Due to CFS Bandwidth Quotas
**Category:** Performance | **Severity:** High | **Env:** Prod (K8s)

**Scenario:** Kubernetes pod has `cpu: "500m"` limit. Application is NOT using 500m CPU but is still getting throttled. CFS quota period causes burst throttling.

```
  CFS period: 100ms
  cpu limit: 500m → 50ms quota per period
  If app bursts for 50ms, it's throttled for remaining 50ms
  Even if average is well below 500m!
```

**Detection:**
```bash
cat /sys/fs/cgroup/cpu/cpu.stat | grep throttled
kubectl top pods
# Prometheus: container_cpu_cfs_throttled_seconds_total
```

**Fix:**
```bash
# Increase cpu limit or use request/limit ratio > 1:
resources:
  requests:
    cpu: "250m"
  limits:
    cpu: "2000m"   # allow burst
```

---

### EC-092 — `top` Load Average Misleading on Multi-CPU Systems
**Category:** Performance | **Severity:** Medium | **Env:** Dev/Prod

**Scenario:** Load average shows 4.0 on a 4-CPU system. System feels fine. But on a 2-CPU system, load average of 4.0 means it's saturated.

```
  Load Average meaning:
  Load 1.0 on 1 CPU = 100% utilized
  Load 4.0 on 4 CPUs = 100% utilized  
  Load 4.0 on 8 CPUs = 50% utilized — still room to grow
```

**Detection:**
```bash
nproc          # number of CPUs
cat /proc/loadavg
# Rule: load average / nproc > 1.0 means saturation
```

---

### EC-093 — Memory Leak Detection via RSS Growth Over Time
**Category:** Performance | **Severity:** High | **Env:** Prod

**Scenario:** Service runs fine at first but memory grows slowly over days/weeks until OOM kill occurs.

**Detection:**
```bash
# Watch RSS growth:
watch -n 60 "ps aux --sort -rss | head -5"
# Or track over time:
while true; do
    echo "$(date) $(cat /proc/<PID>/status | grep VmRSS)"
    sleep 300
done >> /tmp/rss_monitor.log
```

---

### EC-094 — Disk I/O Saturated by Journal Syncs
**Category:** Performance | **Severity:** High | **Env:** Prod

**Scenario:** Disk I/O bottleneck traced to `journald` syncing hundreds of MB/s of logs from a verbose application.

**Detection:**
```bash
iotop -ob | grep systemd-journal
journalctl --disk-usage
```

**Fix:**
```bash
# Reduce journal sync frequency:
# In /etc/systemd/journald.conf:
Storage=volatile     # don't persist to disk
RateLimitIntervalSec=30s
RateLimitBurst=10000
```

---

### EC-095 — `strace` on Production Process Slows It Down 10-100x
**Category:** Performance | **Severity:** Medium | **Env:** Prod

**Scenario:** You attach `strace -p <PID>` to diagnose a production issue. The tracing overhead causes the process to slow down dramatically, making the original issue worse.

> ⚠️ **DevOps Gotcha:** Use `perf`, `bpftrace`, or `eBPF` tools for production tracing — they have minimal overhead. `strace` is safe only for non-latency-sensitive diagnostics in controlled windows.

---

## LVM, RAID & Storage

### EC-096 — LVM Snapshot Gets Full and Becomes Invalid
**Category:** Storage | **Severity:** Critical | **Env:** Prod

**Scenario:** LVM snapshot taken for backup. Original volume has high write activity. Snapshot COW (copy-on-write) space fills up before backup completes. Snapshot becomes invalid — backup is corrupted.

**Detection:**
```bash
lvdisplay /dev/vg/snap | grep -E "LV Size|COW-table"
lvs -o +snap_percent   # shows snapshot usage percentage
```

**Fix:**
```bash
# Create snapshot with adequate COW space (20-30% of original):
lvcreate -L 20G -s -n snap /dev/vg/data

# Extend a snapshot before it fills up:
lvextend -L +10G /dev/vg/snap
```

> ⚠️ **DevOps Gotcha:** An LVM snapshot that exceeds its COW space is silently marked invalid. `lvs` will show `OI---s` with 100% used. Any mount attempt will fail. Monitor snapshot usage continuously during backups.

---

### EC-097 — RAID Degraded Array Silently Running for Months
**Category:** Storage | **Severity:** Critical | **Env:** Prod

**Scenario:** RAID-1 array had one disk fail. No alert was configured. System runs degraded for months. Second disk fails → complete data loss.

**Detection:**
```bash
cat /proc/mdstat
mdadm --detail /dev/md0 | grep -E "State|Failed"
# Should be: active, clean — NOT: degraded
```

**Fix:**
```bash
# Set up email alerts:
echo "MAILADDR admin@company.com" >> /etc/mdadm/mdadm.conf
systemctl enable --now mdmonitor
# Or monitor via SMART:
smartctl -a /dev/sda | grep -E "Reallocated|Uncorrectable"
```

---

### EC-098 — `mdadm --stop` Loses Array Config on Reboot
**Category:** Storage | **Severity:** High | **Env:** Prod

**Scenario:** `mdadm --stop /dev/md0` to do maintenance. After reboot, array comes up as `/dev/md127` with a different name, breaking `/etc/fstab`.

**Fix:**
```bash
# Always update mdadm.conf after configuration changes:
mdadm --detail --scan >> /etc/mdadm/mdadm.conf
update-initramfs -u    # rebuild initramfs to include array config
```

---

### EC-099 — `pvmove` Running During High I/O Causes Data Loss Risk
**Category:** Storage | **Severity:** High | **Env:** Prod

**Scenario:** `pvmove /dev/sdb` to migrate LVM data off a failing disk. If the system crashes mid-move, the LV may be split across both PVs and become inconsistent.

> ⚠️ **DevOps Gotcha:** Always take a snapshot backup before running `pvmove`. Run `pvmove` during low I/O maintenance windows. Monitor with `pvdisplay` and `lvs`.

---

### EC-100 — Thin-Provisioned LVM Volume Overwrites Itself on Full
**Category:** Storage | **Severity:** Critical | **Env:** Prod

**Scenario:** Thin-provisioned LV fills its thin pool. Write operations start failing but the metadata becomes inconsistent. Data corruption risk.

**Detection:**
```bash
lvs -o+thin_id,data_percent,metadata_percent | grep thin
```

**Fix:**
```bash
# Extend the thin pool before it hits 100%:
lvextend -L +50G /dev/vg/pool
# Set auto-extend threshold:
lvmconfig | grep thin
# In /etc/lvm/lvm.conf:
# thin_pool_autoextend_threshold = 70
# thin_pool_autoextend_percent = 20
```

---

### EC-101 — XFS Filesystem Won't Shrink (Only Grows)
**Category:** Storage | **Severity:** Medium | **Env:** Prod

**Scenario:** Trying to reduce an XFS filesystem size. `xfs_growfs -D` only grows. XFS cannot be shrunk.

> ⚠️ **DevOps Gotcha:** If you need to shrink an XFS filesystem, the only option is: backup data → recreate smaller filesystem → restore data. Choose filesystem size carefully for XFS, or use ext4 which supports offline shrink.

---

### EC-102 — `dd` Progress Not Visible by Default
**Category:** Storage | **Severity:** Low | **Env:** Dev/Prod

**Scenario:** Running `dd if=/dev/sda of=/backup/disk.img` — no progress indication. Appears to hang.

**Detection:**
```bash
# Modern dd (coreutils 8.24+):
dd if=/dev/sda of=/dev/sdb status=progress
# Or send SIGUSR1:
kill -USR1 $(pgrep ^dd)
# Or use pv:
pv /dev/sda > /dev/sdb
```

---

### EC-103 — LUKS Encryption Key Slot Exhausted
**Category:** Storage | **Severity:** High | **Env:** Prod

**Scenario:** LUKS encrypted disk supports only 8 key slots. After adding many keys (recovery keys, team keys), slots are exhausted. Can't add new key without removing old one.

**Detection:**
```bash
cryptsetup luksDump /dev/sda | grep "Key Slot"
```

---

### EC-104 — Filesystem Corruption After Forced Unmount
**Category:** Storage | **Severity:** Critical | **Env:** Prod

**Scenario:** `umount -f /data` with open file handles. Subsequent mount shows journal errors. Some data in buffer not flushed to disk.

**Fix:**
```bash
# Check filesystem health:
e2fsck -f /dev/mapper/data    # run on unmounted fs
# Use barrier=1 mount option for data integrity:
mount -o barrier=1 /dev/sda /data
```

---

### EC-105 — SSD TRIM Not Running Causes Performance Degradation
**Category:** Storage | **Severity:** Medium | **Env:** Prod

**Scenario:** SSD performance degrades over time on Linux servers that never run `fstrim`.

**Fix:**
```bash
# Enable weekly TRIM:
systemctl enable --now fstrim.timer
systemctl status fstrim.timer
# Manual TRIM:
fstrim -v /
```

---

## Kernel & Boot

### EC-106 — GRUB2 Config Not Updated After Kernel Parameter Change
**Category:** Kernel | **Severity:** High | **Env:** Prod

**Scenario:** Edit `/etc/default/grub` to add `quiet splash=0`. Reboot — changes don't apply because `update-grub` wasn't run.

**Fix:**
```bash
# Debian/Ubuntu:
update-grub
# RHEL/CentOS:
grub2-mkconfig -o /boot/grub2/grub.cfg
# UEFI:
grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg
```

---

### EC-107 — `sysctl` Settings Not Applied at Boot (initrd Runs Before sysctl)
**Category:** Kernel | **Severity:** Medium | **Env:** Prod

**Scenario:** `/etc/sysctl.conf` settings applied too late. Initrd loads, network comes up with default settings before sysctl runs.

**Fix:**
```bash
# For early boot parameters, use kernel command line in GRUB:
# GRUB_CMDLINE_LINUX="net.ifnames=0 biosdevname=0"
# Some settings also need to be in initrd:
echo 'add_drivers+=" dm_thin_pool "' > /etc/dracut.conf.d/thin.conf
dracut --force
```

---

### EC-108 — Kernel Panic on Boot Due to Missing initrd Module
**Category:** Kernel | **Severity:** Critical | **Env:** Prod

**Scenario:** After kernel upgrade, system panics on boot: "VFS: Unable to mount root fs on unknown block". Root filesystem driver (e.g., ext4, dm-crypt) not included in initrd.

**Fix:**
```bash
# Boot from rescue ISO, chroot, regenerate initrd:
dracut --force /boot/initramfs-$(uname -r).img $(uname -r)
# Or for Debian:
update-initramfs -u -k $(uname -r)
```

---

### EC-109 — `dmesg` Timestamp Confusion (Uptime vs Wall Clock)
**Category:** Kernel | **Severity:** Low | **Env:** Dev/Prod

**Scenario:** `dmesg` shows timestamps in seconds since boot (e.g., `[1234567.890]`). Correlating with application logs using wall-clock times is difficult.

**Fix:**
```bash
dmesg -T    # convert to human-readable timestamps
dmesg --ctime  # same, older syntax
# Or:
journalctl -k   # kernel log with proper timestamps
```

---

### EC-110 — Soft Lockup / Hard Lockup Detected Doesn't Always Mean Hardware Failure
**Category:** Kernel | **Severity:** High | **Env:** Prod

**Scenario:** `dmesg` shows "BUG: soft lockup - CPU#0 stuck for 22s!" causing panic. May be caused by a runaway kernel module, VM steal time, or virtualization overhead rather than actual CPU failure.

**Detection:**
```bash
dmesg | grep -E "lockup|stuck|hung_task"
# In VM environments, check steal time:
top    # %st column (steal time)
```

> ⚠️ **DevOps Gotcha:** High steal time (>5%) in VMs causes soft lockups that look like hardware failures. Check your cloud provider's health events. Noisy neighbors on the hypervisor are often the root cause.

---

## Production FAQ

**Q1: Disk is full but I can't delete anything — process keeps writing faster than I delete.**  
A: Use `lsof +L1` to find deleted-but-open files (immediate space recovery without deletion). Then rotate/truncate the offending log file without killing the service.

**Q2: My cron job emails me errors but the exit code is 0.**  
A: Cron mails any stdout/stderr output regardless of exit code. Redirect output: `>/dev/null 2>&1` or redirect stderr to stdout: `2>&1 | logger -t mycron`.

**Q3: Server load is high but CPU and I/O look normal.**  
A: Check for D-state processes waiting on I/O (`ps aux | awk '$8=="D"'`). Load average includes uninterruptible sleep — even 5 processes waiting on a slow NFS mount can spike load to 5.0.

**Q4: A service "works" but is actually starting in a degraded state.**  
A: Check exit codes of `ExecStartPre` commands in the unit file. systemd may consider the service "active" even if pre-start checks failed, depending on `Type=`.

**Q5: `ansible-playbook` reports changed/ok but the service is still broken.**  
A: Ansible's idempotency means it reports what state it SET, not whether the service is working correctly. Always add `post_tasks` with actual verification tests (e.g., `uri` module to test HTTP endpoints).

**Q6: I ran `chmod -R 755 /var/www` and PHP sessions broke.**  
A: `/var/lib/php/sessions` may be under `/var` or nearby and also got its permissions changed. PHP sessions directory needs `chmod 1733` (sticky bit, world-writable) to work with multiple users.

**Q7: System time drifts after running in a container/VM.**  
A: Containers share the host kernel clock and cannot set time independently. Use `chronyd` on the HOST. In VMs, enable VMware Tools time synchronization or AWS Time Sync Service (`169.254.169.123`).

**Q8: `top` shows 0% CPU for a process I know is busy.**  
A: If the process is running in kernel space (system calls), CPU time is attributed to the kernel, not the user-space process. Use `perf top` or `strace -c` to see kernel time breakdown.

**Q9: SSH connection drops exactly every 60 seconds.**  
A: A network appliance (load balancer, NAT gateway, AWS SG) is timing out idle TCP connections. Set `ServerAliveInterval 30` in `~/.ssh/config` to send keepalives.

**Q10: Log file shows application is running but service is unresponsive.**  
A: Application may be in a deadlock or infinite loop consuming CPU. Use `strace -p <PID>` to see what system calls it's making (or if it's stuck in one). A process can write "heartbeat" logs and still be deadlocked on a different thread.

---

> **Next:** [Networking Edge Cases](networking_edge_cases.md) · [Cloud Edge Cases](cloud_edge_cases.md)  
> **Back:** [Batch State](batch_state.md) · [🏠 Home](../README.md)
