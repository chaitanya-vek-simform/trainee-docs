# Linux & DevOps Interview Q&A

---

## 🔹 Linux Fundamentals

### What happens internally in the Linux OS when you run a command?

Shell reads input → parses the command → searches `$PATH` for the binary → calls `fork()` to create a child process → calls `exec()` to replace child with the command's program → kernel executes it → process exits → shell collects exit status via `wait()`.

### What is the `$PATH` environment variable?

An environment variable containing a **colon-separated list of directories** where the shell searches for executable binaries (e.g., `/usr/local/bin:/usr/bin:/bin`).

### How does the system locate a command like `terraform` or `python` when you execute it?

The shell iterates through each directory listed in `$PATH` (left to right), looks for an executable file matching the command name, and runs the **first match** found.

### What happens if the `$PATH` variable is emptied?

Only shell **built-in** commands (like `cd`, `echo`, `export`) will work. External commands (`ls`, `grep`, `python`) will fail with `command not found` unless called with their **full absolute path** (e.g., `/usr/bin/ls`).

### What is the difference between a built-in shell command and an external executable?

| Built-in | External |
| --- | --- |
| Part of the shell itself (`cd`, `echo`, `export`) | Separate binary on disk (`/usr/bin/grep`, `/usr/bin/awk`) |
| No `fork()/exec()` needed | Requires `fork()/exec()` |
| Faster execution | Slightly slower (new process created) |
| Check with `type <cmd>` | Located via `$PATH` |

---

## 🔹 Processes & System Internals

### What is `fork()` and `exec()`?

- **`fork()`** — Creates a **copy** (child process) of the calling (parent) process.
- **`exec()`** — **Replaces** the current process image with a new program.

Used together: `fork()` creates the child, `exec()` loads the new command into it.

### What is a zombie process?

A process that has **finished execution** but still has an entry in the process table because the parent hasn't called `wait()` to read its exit status. It shows as `Z` or `defunct` in `ps`.

### What is an orphan process?

A running child process whose **parent has terminated**. The `init`/`systemd` (PID 1) adopts it and eventually reaps it.

### How do you check CPU usage of running processes?

```bash
top                          # interactive, real-time
htop                         # enhanced interactive view
ps aux --sort=-%cpu | head   # snapshot, sorted by CPU
```

### What command shows real-time process monitoring?

```bash
top
htop
```

### How can you view threads of a process?

```bash
ps -T -p <PID>
top -H -p <PID>
ls /proc/<PID>/task/
```

### How do you identify a zombie process?

```bash
ps aux | grep 'Z'
# or
ps -eo pid,ppid,stat,cmd | grep 'Z'
```

Look for state **Z** or **Z+** / `<defunct>`.

### How can you remove or handle a zombie process?

You **can't kill** a zombie (it's already dead). You must:

1. Send `SIGCHLD` to the parent: `kill -s SIGCHLD <parent_PID>`
2. If that doesn't work, **kill the parent process** — `init` will adopt and reap the zombie.

---

## 🔹 Memory Management

### What is swap memory?

Disk space used as **overflow** for RAM. When physical RAM is full, the kernel moves inactive pages to swap (on disk) to free up RAM.

### Why is swap slower than RAM?

RAM is **electronic/semiconductor** memory (nanosecond access). Swap is on **disk** (HDD/SSD) — even SSDs are orders of magnitude slower than RAM. Disk I/O is the bottleneck.

### If swap is getting used daily at peak load, should we increase swap or RAM? Why?

**Increase RAM.** Swap is an emergency fallback, not a permanent solution. Daily swap usage means the workload genuinely needs more physical memory. Adding more swap would just increase disk I/O and degrade performance further.

---

## 🔹 Filesystem Concepts

### What is an inode?

A data structure on a filesystem that stores **metadata** about a file (but NOT its name or content). Every file/directory has a unique inode number.

### What information does an inode store?

File type, permissions, owner (UID/GID), size, timestamps (atime, mtime, ctime), number of hard links, pointers to data blocks on disk. **Not stored:** filename (that's in the directory entry).

### What is the difference between a hard link and a soft link?

| Hard Link | Soft (Symbolic) Link |
| --- | --- |
| Points to the **same inode** | Points to the **file path/name** |
| Cannot cross filesystems | Can cross filesystems |
| Cannot link directories | Can link directories |
| Original deleted → data still accessible | Original deleted → **broken link** |

### Does a hard link create a new inode?

**No.** A hard link shares the **same inode** as the original file. It just creates a new directory entry pointing to the existing inode (link count increments).

### What happens to a soft link if the original file is deleted?

It becomes a **dangling/broken link**. The symlink still exists but points to a non-existent path. Accessing it gives `No such file or directory`.

---

## 🔹 Permissions

### What is `umask`?

A mask that **subtracts** permissions from the default when new files/directories are created. It defines which permissions to **remove** by default.

### What is the difference between `umask` and `chmod`?

| umask | chmod |
| --- | --- |
| Sets **default** permissions for new files | Changes permissions on **existing** files |
| Subtractive (removes permissions) | Explicit (sets permissions directly) |
| Applies automatically at creation | Must be run manually |

### What is the default permission for a newly created file?

Base: `666` (files) / `777` (directories). Then `umask` is subtracted.
With default umask `022`: files → `644`, directories → `755`.

### If `umask` is set to `027`, what will be the file permission?

`666 - 027 = 640` → owner: `rw-`, group: `r--`, others: `---`

---

## 🔹 Disk Usage & Storage

### What is the difference between `du` and `df`?

| `du` | `df` |
| --- | --- |
| Shows disk usage **per file/directory** | Shows disk usage **per filesystem/mount** |
| Granular, file-level | High-level overview |

### What does `du` stand for?

**Disk Usage**

### What does `df` stand for?

**Disk Free**

### How do you find the largest files in your system?

```bash
du -ah / | sort -rh | head -20
# or
find / -type f -exec du -h {} + | sort -rh | head -20
```

### How do you check disk usage of a specific directory?

```bash
du -sh /path/to/directory       # summary
du -ah /path/to/directory       # all files listed
```

---

## 🔹 File & Log Management

### How do you view logs of a process?

```bash
journalctl -u <service-name>       # systemd service logs
tail -f /var/log/<logfile>         # follow log file in real-time
```

### Where are Nginx or Apache logs usually stored?

- **Nginx:** `/var/log/nginx/access.log` and `/var/log/nginx/error.log`
- **Apache:** `/var/log/apache2/access.log` and `/var/log/apache2/error.log` (Debian/Ubuntu) or `/var/log/httpd/` (RHEL/CentOS)

### What is the difference between `>` and `>>`?

- `>` — **Overwrites** the file (truncates then writes).
- `>>` — **Appends** to the file (adds to the end).

### How do you delete files older than 30 days?

```bash
find /path/to/dir -type f -mtime +30 -delete
```

---

## 🔹 Scheduling & Automation

### How do you create a cron job?

```bash
crontab -e    # opens the user's crontab for editing
```

Add a line in the format: `MIN HOUR DOM MON DOW command`

### Write a cron job to print "hello" into a file every minute.

```
* * * * * echo "hello" >> /tmp/hello.txt
```

### How do you edit cron jobs?

```bash
crontab -e              # edit current user's cron
crontab -l              # list current cron jobs
sudo crontab -e -u <user>   # edit another user's cron
```

---

## 🔹 Windows Server Concepts

### What is a Domain Controller?

A Windows Server that runs **Active Directory Domain Services (AD DS)**. It authenticates users, enforces security policies, and manages all objects (users, computers, groups) in the domain.

### What is the difference between a local user and a domain user?

| Local User | Domain User |
| --- | --- |
| Exists on **one machine** only | Exists in **Active Directory**, accessible across the domain |
| Managed via local SAM database | Managed via Domain Controller |
| Can log into that machine only | Can log into **any domain-joined** machine |

---

## 🔹 Networking

### What is DHCP?

**Dynamic Host Configuration Protocol** — automatically assigns IP addresses, subnet masks, gateways, and DNS servers to devices on a network.

### How does DHCP lease process work?

4-step **DORA** process:

1. **Discover** — Client broadcasts "I need an IP"
2. **Offer** — DHCP server offers an available IP
3. **Request** — Client requests the offered IP
4. **Acknowledge** — Server confirms and leases the IP for a set duration

### What does Nginx `proxy_pass` do?

Forwards incoming client requests to a **backend/upstream server**. Nginx acts as a **reverse proxy**.

### What is the difference between `proxy_pass http://localhost:3000;` and `proxy_pass http://localhost:3000/;`?

| Without trailing slash | With trailing slash `/` |
| --- | --- |
| The **location path is appended** to the upstream URL | The location path is **stripped/replaced** |

Example with `location /api/`:

- **Without `/`** → request `/api/users` → proxied to `http://localhost:3000/api/users`
- **With `/`** → request `/api/users` → proxied to `http://localhost:3000/users` (`/api/` is stripped)

---

## 🔹 Debugging Mindset

### Your root disk is 98% full. How will you identify the largest files?

```bash
df -h                                          # confirm which mount is full
du -ah / | sort -rh | head -20                 # find largest files/dirs
find / -type f -size +100M -exec ls -lh {} +   # files > 100MB
journalctl --disk-usage                        # check journal log size
du -sh /var/log/*                              # check log sizes
```

### Your application on a VM is slow. What commands will you run first?

```bash
top / htop          # check CPU and memory usage
free -h             # check RAM and swap usage
df -h               # check disk space
iostat              # check disk I/O
vmstat 1 5          # check system performance (CPU, memory, I/O)
netstat -tulnp      # check network connections
dmesg | tail        # check kernel messages for errors
journalctl -xe      # check recent system logs
```

### How do you check running services?

```bash
systemctl list-units --type=service --state=running
# or
service --status-all
```

### How do you check system logs in Linux?

```bash
journalctl -xe                 # systemd journal (recent + errors)
tail -f /var/log/syslog        # Debian/Ubuntu
tail -f /var/log/messages      # RHEL/CentOS
dmesg                          # kernel ring buffer
```
