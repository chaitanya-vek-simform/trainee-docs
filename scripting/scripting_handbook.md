[🏠 Home](../README.md) · [Scripting](README.md)

# 📜 Scripting for DevOps — Comprehensive Handbook

> **Audience:** DevOps / Cloud Engineers & Trainees | **Focus:** Bash + Python, Automation Patterns, Production-Ready Scripts
> **Tip:** Sections 1–8 cover Bash, Sections 9–20 cover Python & advanced topics. Use the TOC to jump around.

---

## Table of Contents

1. [Why Scripting for DevOps](#1-why-scripting-for-devops)
2. [Bash Basics & Shell Fundamentals](#2-bash-basics--shell-fundamentals)
3. [Control Flow & Logic](#3-control-flow--logic)
4. [Functions, Arguments & Input](#4-functions-arguments--input)
5. [Text Processing in Scripts](#5-text-processing-in-scripts)
6. [File Operations & Process Management](#6-file-operations--process-management)
7. [Real-World Bash Patterns](#7-real-world-bash-patterns)
8. [Bash Debugging & Best Practices](#8-bash-debugging--best-practices)
9. [Python Fundamentals for DevOps](#9-python-fundamentals-for-devops)
10. [File I/O, JSON, YAML & APIs](#10-file-io-json-yaml--apis)
11. [OS Automation with Python](#11-os-automation-with-python)
12. [Error Handling & Logging](#12-error-handling--logging)
13. [Infrastructure Scripting](#13-infrastructure-scripting)
14. [CI/CD Pipeline Scripting](#14-cicd-pipeline-scripting)
15. [Container & Kubernetes Scripting](#15-container--kubernetes-scripting)
16. [Monitoring & Alerting Scripts](#16-monitoring--alerting-scripts)
17. [Security in Scripting](#17-security-in-scripting)
18. [Testing Scripts](#18-testing-scripts)
19. [Performance & Packaging](#19-performance--packaging)
20. [Production Scenario FAQ](#20-production-scenario-faq)

---

## 1. Why Scripting for DevOps

### 1.1 The Automation Mindset

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  THE DEVOPS AUTOMATION RULE                                      │
  │                                                                  │
  │  "If you do something twice, automate it."                      │
  │                                                                  │
  │  Manual Work                    Scripted Automation              │
  │  ┌──────────────────┐           ┌──────────────────────┐        │
  │  │ SSH into server  │           │ Run one command      │        │
  │  │ Remember steps   │           │ Reproducible output  │        │
  │  │ Type commands    │           │ Version controlled   │        │
  │  │ Hope nothing     │           │ Tested & reviewed    │        │
  │  │ was missed       │           │ Self-documenting     │        │
  │  │ Document later   │           │ Runs at 3 AM (cron)  │        │
  │  │ (you won't)      │           │ Scales to 1000       │        │
  │  └──────────────────┘           │ servers              │        │
  │                                 └──────────────────────┘        │
  │                                                                  │
  │  Time investment:                                                │
  │  ┌────────────────────────────────────────────────┐             │
  │  │ Manual:  5 min × 100 times = 500 min           │             │
  │  │ Script:  60 min once + 1 min × 100 = 160 min   │             │
  │  │ Savings: 340 min + zero human errors            │             │
  │  └────────────────────────────────────────────────┘             │
  └──────────────────────────────────────────────────────────────────┘
```

### 1.2 Bash vs Python vs Go — When to Use What

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  SCRIPTING LANGUAGE DECISION TREE                                │
  │                                                                  │
  │  "How complex is the task?"                                      │
  │                                                                  │
  │  Simple (< 50 lines)              Complex (> 50 lines)          │
  │  ├── Gluing CLI tools together?   ├── API integrations?         │
  │  ├── File operations?             ├── Data transformation?      │
  │  ├── System administration?       ├── Error handling needed?    │
  │  ├── Cron jobs?                   ├── Libraries needed?         │
  │  └── ⟹ USE BASH                  └── ⟹ USE PYTHON             │
  │                                                                  │
  │  Need a compiled binary?          Need extreme performance?     │
  │  ├── Cross-platform CLI tool?     ├── High-concurrency server?  │
  │  ├── Kubernetes operator?         ├── System-level tool?        │
  │  └── ⟹ USE GO                    └── ⟹ USE GO or RUST         │
  └──────────────────────────────────────────────────────────────────┘
```

| Criteria | Bash | Python | Go |
|----------|------|--------|----|
| **Learning curve** | Low | Medium | Medium-High |
| **Available everywhere** | ✅ Every Linux box | ✅ Most servers | ❌ Needs install |
| **CLI tool glue** | ⭐⭐⭐ Best | ⭐⭐ Good | ⭐ Overkill |
| **API/HTTP work** | ❌ Painful (curl) | ⭐⭐⭐ Best | ⭐⭐ Good |
| **Data parsing** | ❌ Limited | ⭐⭐⭐ Best | ⭐⭐ Good |
| **Error handling** | ❌ Fragile | ⭐⭐⭐ try/except | ⭐⭐ explicit |
| **Concurrency** | ❌ Hacky (&, wait) | ⭐⭐ threading/async | ⭐⭐⭐ goroutines |
| **Packaging** | N/A (just copy) | pip, Docker | Single binary |
| **DevOps use** | Glue, cron, init | Automation, tools | Operators, CLIs |

### 1.3 What DevOps Engineers Actually Script

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  COMMON DEVOPS SCRIPTING TASKS                                   │
  │                                                                  │
  │  Bash Scripts:                   Python Scripts:                 │
  │  ├── Server provisioning         ├── AWS/GCP/Azure automation   │
  │  ├── Log rotation & cleanup      ├── CI/CD pipeline helpers     │
  │  ├── Health check endpoints      ├── Kubernetes automation      │
  │  ├── Backup & restore            ├── Monitoring & alerting      │
  │  ├── Deployment wrappers         ├── Data migration scripts     │
  │  ├── Cron job scripts            ├── API integrations           │
  │  ├── Docker entrypoint scripts   ├── Secret rotation            │
  │  ├── Environment setup           ├── Infrastructure testing     │
  │  └── SSH key management          └── Custom CLI tools           │
  │                                                                  │
  │  Rule of Thumb:                                                  │
  │  ├── Bash = "Run commands on this machine"                      │
  │  └── Python = "Orchestrate across systems"                      │
  └──────────────────────────────────────────────────────────────────┘
```

> **DevOps Relevance:** As a DevOps engineer, scripting is NOT optional — it's your primary productivity multiplier. You'll write Bash daily (Dockerfiles, CI steps, cron jobs) and Python weekly (automation tools, API integrations, infrastructure scripts). Master both.

---

## 2. Bash Basics & Shell Fundamentals

### 2.1 The Shebang Line — Your Script's Identity

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  SHEBANG (#!) — FIRST LINE OF EVERY SCRIPT                     │
  │                                                                  │
  │  #!/bin/bash           ← "Use the Bash shell at /bin/bash"     │
  │  #!/usr/bin/env bash   ← "Find bash in $PATH" (more portable) │
  │  #!/usr/bin/env python3                                         │
  │  #!/bin/sh             ← POSIX shell (not Bash — no arrays,   │
  │                           no [[ ]], no process substitution)    │
  │                                                                  │
  │  Without shebang: script runs in whatever shell calls it       │
  │  With shebang:    kernel knows which interpreter to use        │
  │                                                                  │
  │  Best practice: Always use #!/usr/bin/env bash                 │
  │  (works on Linux, macOS, containers where bash path may vary)  │
  └──────────────────────────────────────────────────────────────────┘
```

```bash
#!/usr/bin/env bash
# My first script — always start with these 3 lines:

set -euo pipefail  # THE most important line (explained below)

echo "Hello, DevOps!"
```

```bash
# Make it executable and run it
chmod +x script.sh      # Add execute permission
./script.sh              # Run it
bash script.sh           # Alternative: explicitly call bash
```

### 2.2 The Holy Trinity: `set -euo pipefail`

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  set -euo pipefail — THE STRICT MODE FOR BASH                  │
  │                                                                  │
  │  set -e          Exit immediately if ANY command fails          │
  │                  Without this, script continues silently        │
  │                  after errors — DANGEROUS in production         │
  │                                                                  │
  │  set -u          Treat unset variables as errors               │
  │                  Without this, $UNDEFINED silently becomes ""  │
  │                  Can cause: rm -rf /$UNDEFINED_VAR/*           │
  │                  (This has deleted entire servers. Really.)    │
  │                                                                  │
  │  set -o pipefail Make pipelines fail if ANY command fails      │
  │                  Without this: `bad_cmd | good_cmd` = success  │
  │                  With this:    `bad_cmd | good_cmd` = failure   │
  │                                                                  │
  │  Combined: set -euo pipefail                                    │
  │  PUT THIS ON LINE 2 OF EVERY BASH SCRIPT. NO EXCEPTIONS.       │
  └──────────────────────────────────────────────────────────────────┘
```

```bash
#!/usr/bin/env bash
set -euo pipefail

# Without set -e: this continues silently after the error
cp /nonexistent/file /tmp/dest   # This FAILS
echo "This still prints!"        # Without -e, this runs. With -e, script exits.

# Without set -u:
echo "Deploying to $ENVIRONMENT"  # If $ENVIRONMENT is unset → empty string
# With -u: bash: ENVIRONMENT: unbound variable (script exits)

# Without pipefail:
cat /nonexistent | grep "pattern"  # Exit code = 0 (grep's code, not cat's)
# With pipefail: Exit code = 1 (cat failed → pipeline fails)
```

> **⚠️ Gotcha (set -e in conditionals):** `set -e` does NOT trigger inside `if` conditions, `||`, or `&&` chains. This is by design — the shell needs to "test" the command. But it means errors can be silently swallowed:
> ```bash
> if some_command; then echo "ok"; fi  # some_command failure won't exit
> some_command || true                  # Explicitly allowing failure
> ```

### 2.3 Variables & Quoting

```bash
#!/usr/bin/env bash
set -euo pipefail

# Variable assignment — NO SPACES around =
name="DevOps"                    # String
count=42                         # Number (still a string internally)
readonly VERSION="2.0.0"         # Constant — can't be changed

# Using variables — ALWAYS quote
echo "Hello, ${name}!"           # Curly braces for clarity
echo "Version: $VERSION"         # Works without braces too

# The QUOTING rules — this is where most bugs live
file="my file.txt"
cat $file         # BUG: bash sees TWO args: "my" and "file.txt"
cat "$file"       # CORRECT: bash sees ONE arg: "my file.txt"
```

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  QUOTING RULES — MEMORIZE THIS                                  │
  │                                                                  │
  │  Double quotes "..."                                             │
  │  ├── Variables ARE expanded:  "$USER" → "john"                  │
  │  ├── Command sub IS expanded: "$(date)" → "Mon Mar 15..."      │
  │  ├── Glob patterns NOT expanded: "*.txt" stays "*.txt"          │
  │  └── USE THIS BY DEFAULT for all variable usage                 │
  │                                                                  │
  │  Single quotes '...'                                             │
  │  ├── NOTHING is expanded: '$USER' stays '$USER'                 │
  │  ├── Completely literal string                                   │
  │  └── Use for: regex patterns, awk programs, fixed strings      │
  │                                                                  │
  │  No quotes (bare)                                                │
  │  ├── Variables expanded + word splitting + glob expansion       │
  │  ├── "my file.txt" becomes TWO words                            │
  │  └── ALMOST ALWAYS A BUG — avoid this                           │
  │                                                                  │
  │  RULE: "When in doubt, double-quote it."                       │
  └──────────────────────────────────────────────────────────────────┘
```

```bash
# Variable Operations
name="John"
echo "${name^^}"          # JOHN       (uppercase)
echo "${name,,}"          # john       (lowercase)
echo "${name:0:2}"        # Jo         (substring: offset:length)
echo "${#name}"           # 4          (length)

# Default values — critical for scripts with optional env vars
DB_HOST="${DB_HOST:-localhost}"       # Use localhost if DB_HOST is unset
DB_PORT="${DB_PORT:-5432}"           # Use 5432 if DB_PORT is unset
REQUIRED="${MUST_SET:?'Error: MUST_SET is required'}"  # Exit with error if unset

# Variable substitution (search/replace)
filepath="/var/log/nginx/access.log"
echo "${filepath%.log}"              # /var/log/nginx/access  (remove suffix)
echo "${filepath##*/}"               # access.log             (basename)
echo "${filepath%/*}"                # /var/log/nginx          (dirname)
echo "${filepath/nginx/apache}"      # /var/log/apache/access.log (replace first)
echo "${filepath//\//\\}"            # \var\log\nginx\access.log  (replace all)
```

### 2.4 Exit Codes — How Scripts Communicate

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  EXIT CODES — THE LANGUAGE OF UNIX PROCESSES                    │
  │                                                                  │
  │  Exit Code    Meaning                                            │
  │  ────────────────────────────────────────────                    │
  │  0            Success (ALL good)                                 │
  │  1            General error                                      │
  │  2            Misuse of shell built-in                           │
  │  126          Command found but NOT executable                  │
  │  127          Command NOT found                                  │
  │  128+N        Killed by signal N                                 │
  │  130          Killed by Ctrl+C (SIGINT = signal 2, 128+2=130)  │
  │  137          Killed by SIGKILL (128+9=137) — OOMKilled!       │
  │  143          Killed by SIGTERM (128+15=143) — graceful stop   │
  │                                                                  │
  │  RULE: 0 = success, anything else = failure                     │
  │  Check with: echo $?  (shows exit code of last command)        │
  └──────────────────────────────────────────────────────────────────┘
```

```bash
#!/usr/bin/env bash
set -euo pipefail

# Custom exit codes for your scripts
readonly EXIT_SUCCESS=0
readonly EXIT_INVALID_ARGS=1
readonly EXIT_FILE_NOT_FOUND=2
readonly EXIT_NETWORK_ERROR=3

# Use exit codes meaningfully
check_config() {
    local config_file="$1"
    if [[ ! -f "$config_file" ]]; then
        echo "ERROR: Config file not found: $config_file" >&2
        exit $EXIT_FILE_NOT_FOUND
    fi
}

# Check last command's exit code
if grep -q "error" /var/log/app.log; then
    echo "Errors found in log"
else
    echo "No errors"  # grep returns exit code 1 when no match
fi
```

### 2.5 Command Substitution & Arithmetic

```bash
#!/usr/bin/env bash
set -euo pipefail

# Command substitution — capture command output
today=$(date +%Y-%m-%d)           # Modern syntax (preferred)
today=`date +%Y-%m-%d`            # Legacy syntax (avoid — can't nest)
hostname=$(hostname -f)
ip_addr=$(hostname -I | awk '{print $1}')

# Arithmetic — use $(( ))
count=5
total=$((count * 3))               # 15
remaining=$((total - count))       # 10
((count++))                        # Increment
((count += 10))                    # Add 10

# DO NOT use expr (legacy):
# result=$(expr 5 + 3)            # Works but ugly and slow

# Floating point — bash can't do it! Use bc or awk
percentage=$(echo "scale=2; 85 / 100 * 100" | bc)
percentage=$(awk 'BEGIN {printf "%.2f", 85/100 * 100}')
```

> **⚠️ Gotcha (Arithmetic & leading zeros):** Bash treats numbers with leading zeros as OCTAL. `echo $((08 + 1))` gives an error because `08` isn't valid octal. Strip leading zeros: `echo $((10#08 + 1))` → `9`.

### 2.6 Arrays

```bash
#!/usr/bin/env bash
set -euo pipefail

# Indexed arrays
servers=("web01" "web02" "db01" "db02")
echo "${servers[0]}"               # web01 (first element)
echo "${servers[@]}"               # All elements
echo "${#servers[@]}"              # 4 (count)
echo "${servers[@]:1:2}"           # web02 db01 (slice: offset:count)

# Add to array
servers+=("cache01")

# Loop over array (ALWAYS quote "${array[@]}")
for server in "${servers[@]}"; do
    echo "Checking $server..."
    ssh "$server" uptime || echo "  FAILED: $server"
done

# Associative arrays (dictionaries) — Bash 4+
declare -A services
services[web]="nginx"
services[app]="gunicorn"
services[db]="postgresql"

for role in "${!services[@]}"; do
    echo "$role → ${services[$role]}"
done
```

> **⚠️ Gotcha (Array quoting):** `"${array[@]}"` (with quotes) preserves elements with spaces. `${array[@]}` (no quotes) splits on spaces and breaks. This is the #1 array bug:
> ```bash
> files=("my file.txt" "another file.txt")
> for f in ${files[@]}; do    # BUG: loops 4 times (word splitting)
> for f in "${files[@]}"; do  # CORRECT: loops 2 times
> ```

> **DevOps Relevance:** Bash is the first scripting language you'll use on any Linux server. Every Dockerfile contains shell commands, every CI pipeline runs shell steps, every cron job is a shell script. Master `set -euo pipefail`, quoting, and exit codes — these three concepts prevent 90% of script bugs in production.

---

## 3. Control Flow & Logic

### 3.1 The `test` Command & Bracket Syntax

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  TEST SYNTAX — THREE WAYS TO TEST CONDITIONS                    │
  │                                                                  │
  │  test expression        ← Original syntax (POSIX)              │
  │                                                                  │
  │  [ expression ]          ← Shorthand for test (POSIX)          │
  │                          Must have spaces inside brackets!     │
  │                          [ is actually a command, ] is its arg │
  │                                                                  │
  │  [[ expression ]]        ← Bash-specific (PREFERRED)           │
  │                          Better quoting, regex support,        │
  │                          pattern matching, no word splitting   │
  │                                                                  │
  │  RULE: Always use [[ ]] in Bash scripts                        │
  │  Only use [ ] if writing POSIX sh scripts (rare)              │
  └──────────────────────────────────────────────────────────────────┘
```

### 3.2 Comparison Operators

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  STRING COMPARISONS                                              │
  │  ─────────────────                                               │
  │  [[ "$a" == "$b" ]]    Strings are equal                        │
  │  [[ "$a" != "$b" ]]    Strings are not equal                    │
  │  [[ "$a" < "$b" ]]     String a sorts before b                 │
  │  [[ -z "$a" ]]         String is empty (Zero length)           │
  │  [[ -n "$a" ]]         String is NOT empty (Non-zero length)   │
  │  [[ "$a" == *.log ]]   Glob pattern match (in [[ ]] only)     │
  │  [[ "$a" =~ ^[0-9]+$ ]]  Regex match (in [[ ]] only)          │
  │                                                                  │
  │  NUMERIC COMPARISONS                                             │
  │  ───────────────────                                             │
  │  [[ $a -eq $b ]]      Equal                                    │
  │  [[ $a -ne $b ]]      Not equal                                │
  │  [[ $a -lt $b ]]      Less than                                │
  │  [[ $a -le $b ]]      Less than or equal                       │
  │  [[ $a -gt $b ]]      Greater than                             │
  │  [[ $a -ge $b ]]      Greater than or equal                    │
  │                                                                  │
  │  ⚠️ TRAP: Don't use == for numbers or -eq for strings!        │
  │  [[ "10" == "10" ]]     ← string comparison (works here)      │
  │  [[ "2" > "10" ]]      ← TRUE! (string sort: "2" > "1...")    │
  │  [[ 2 -gt 10 ]]        ← FALSE (numeric comparison: correct) │
  │                                                                  │
  │  FILE TESTS                                                      │
  │  ──────────                                                      │
  │  [[ -f "$path" ]]     Is a regular file                        │
  │  [[ -d "$path" ]]     Is a directory                           │
  │  [[ -e "$path" ]]     Exists (file, dir, or anything)         │
  │  [[ -r "$path" ]]     Is readable                              │
  │  [[ -w "$path" ]]     Is writable                              │
  │  [[ -x "$path" ]]     Is executable                            │
  │  [[ -s "$path" ]]     Exists and is NOT empty (Size > 0)      │
  │  [[ -L "$path" ]]     Is a symbolic link                      │
  │  [[ "$a" -nt "$b" ]]  File a is Newer Than file b            │
  │  [[ "$a" -ot "$b" ]]  File a is Older Than file b            │
  │                                                                  │
  │  LOGICAL OPERATORS (inside [[ ]])                               │
  │  ─────────────────────────────────                               │
  │  [[ cond1 && cond2 ]]    AND                                    │
  │  [[ cond1 || cond2 ]]    OR                                     │
  │  [[ ! condition ]]       NOT                                    │
  └──────────────────────────────────────────────────────────────────┘
```

### 3.3 If / Elif / Else

```bash
#!/usr/bin/env bash
set -euo pipefail

# Basic if/else
if [[ -f "/etc/nginx/nginx.conf" ]]; then
    echo "Nginx is installed"
elif [[ -f "/etc/apache2/apache2.conf" ]]; then
    echo "Apache is installed"
else
    echo "No web server found"
fi

# One-liner (for simple checks)
[[ -d "/backups" ]] || mkdir -p /backups

# Checking command success (no brackets needed)
if ping -c 1 -W 2 google.com &>/dev/null; then
    echo "Internet is up"
else
    echo "No internet connectivity" >&2
    exit 1
fi

# Combining conditions
if [[ -f "$config" && -r "$config" ]]; then
    source "$config"
fi

# Regex matching
email="admin@example.com"
if [[ "$email" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]; then
    echo "Valid email"
fi
```

### 3.4 Case Statement

```bash
#!/usr/bin/env bash
set -euo pipefail

# Case — cleaner than long if/elif chains
case "${1:-}" in
    start)
        echo "Starting service..."
        systemctl start myapp
        ;;
    stop)
        echo "Stopping service..."
        systemctl stop myapp
        ;;
    restart)
        echo "Restarting service..."
        systemctl restart myapp
        ;;
    status)
        systemctl status myapp
        ;;
    deploy)
        echo "Deploying..."
        ./deploy.sh
        ;;
    *)
        echo "Usage: $0 {start|stop|restart|status|deploy}" >&2
        exit 1
        ;;
esac
```

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  CASE PATTERN MATCHING                                           │
  │                                                                  │
  │  case "$var" in                                                  │
  │      pattern1)   ... ;;     ← exact match                      │
  │      pat1|pat2)  ... ;;     ← OR (multiple patterns)           │
  │      *.log)      ... ;;     ← glob match                       │
  │      [yY]*)      ... ;;     ← starts with y or Y              │
  │      ???)        ... ;;     ← exactly 3 characters             │
  │      *)          ... ;;     ← default / catch-all              │
  │  esac                                                            │
  │                                                                  │
  │  Use case for: service control scripts, argument parsing,      │
  │  menu selection, file type routing                              │
  └──────────────────────────────────────────────────────────────────┘
```

### 3.5 Loops

```bash
#!/usr/bin/env bash
set -euo pipefail

# FOR loop — iterating over a list
for server in web01 web02 web03 db01; do
    echo "Checking $server..."
    ssh "$server" uptime 2>/dev/null || echo "  $server is DOWN"
done

# FOR loop — iterating over files
for file in /var/log/*.log; do
    [[ -f "$file" ]] || continue   # Skip if glob didn't match
    size=$(stat --printf='%s' "$file" 2>/dev/null || echo 0)
    if [[ "$size" -gt 104857600 ]]; then  # > 100MB
        echo "WARNING: $file is $(( size / 1048576 )) MB"
    fi
done

# FOR loop — C-style
for ((i = 1; i <= 5; i++)); do
    echo "Attempt $i of 5..."
done

# FOR loop — reading from command output
for user in $(awk -F: '$3 >= 1000 {print $1}' /etc/passwd); do
    echo "Regular user: $user"
done

# WHILE loop — retry pattern (VERY common in DevOps)
max_retries=5
retry=0
until curl -sf http://localhost:8080/health &>/dev/null; do
    ((retry++))
    if [[ $retry -ge $max_retries ]]; then
        echo "ERROR: Service failed to start after $max_retries attempts" >&2
        exit 1
    fi
    echo "Waiting for service... (attempt $retry/$max_retries)"
    sleep 5
done
echo "Service is healthy!"

# WHILE loop — reading file line by line
while IFS= read -r line; do
    echo "Processing: $line"
done < /etc/hosts

# WHILE loop — reading with delimiter (CSV)
while IFS=',' read -r name role env; do
    echo "Deploying $name ($role) to $env"
done < servers.csv
```

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  LOOP CONTROL                                                    │
  │                                                                  │
  │  break       Exit the loop entirely                             │
  │  continue    Skip to the next iteration                         │
  │  break 2     Break out of 2 nested loops                        │
  │                                                                  │
  │  INFINITE LOOP PATTERN (with break):                            │
  │  while true; do                                                  │
  │      # check something                                           │
  │      [[ condition ]] && break                                    │
  │      sleep 1                                                     │
  │  done                                                            │
  └──────────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha (while read + pipe):** `echo "data" | while read -r line; do var="$line"; done` — the `var` is set inside a **subshell** (because of the pipe) and is LOST after the loop. Fix: use process substitution: `while read -r line; do ...; done < <(echo "data")`.

> **DevOps Relevance:** Control flow is the backbone of every automation script. The retry-with-backoff pattern (`until curl ... ; sleep`) is used in nearly every deployment script. Case statements power service control scripts. File-iteration loops handle log cleanup, config generation, and batch operations.

---

## 4. Functions, Arguments & Input

### 4.1 Functions — Modular Scripting

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  FUNCTIONS IN BASH                                               │
  │                                                                  │
  │  function name() {          ← Bash-specific syntax              │
  │      ...                                                         │
  │  }                                                               │
  │                                                                  │
  │  name() {                   ← POSIX-compatible (preferred)      │
  │      ...                                                         │
  │  }                                                               │
  │                                                                  │
  │  Key principles:                                                 │
  │  ├── Functions use the SAME $1, $2... as scripts               │
  │  ├── Variables are GLOBAL by default (use 'local'!)            │
  │  ├── Return value = exit code (0-255), NOT data                │
  │  ├── To return data: echo it and capture with $()              │
  │  └── Define functions BEFORE calling them                       │
  └──────────────────────────────────────────────────────────────────┘
```

```bash
#!/usr/bin/env bash
set -euo pipefail

# GOOD: Function with local variables
log() {
    local level="$1"
    local message="$2"
    local timestamp
    timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    echo "[$timestamp] [$level] $message"
}

# GOOD: Function that returns data via echo
get_ip() {
    local interface="${1:-eth0}"
    ip -4 addr show "$interface" | awk '/inet / {print $2}' | cut -d/ -f1
}

# GOOD: Function that validates and returns exit code
is_port_open() {
    local host="$1"
    local port="$2"
    timeout 2 bash -c "echo >/dev/tcp/$host/$port" 2>/dev/null
}

# Using the functions
log "INFO" "Deployment starting..."

my_ip=$(get_ip "eth0")
log "INFO" "Server IP: $my_ip"

if is_port_open "db01" 5432; then
    log "INFO" "Database is reachable"
else
    log "ERROR" "Cannot reach database on port 5432"
    exit 1
fi
```

> **⚠️ Gotcha (local variables):** Without `local`, variables in functions are GLOBAL and can overwrite variables in the calling scope. This is Bash's biggest trap for writing maintainable code:
> ```bash
> name="original"
> change_name() { name="changed"; }   # Overwrites global!
> change_name
> echo "$name"  # "changed" — surprise!
> ```
> Always use `local` for function variables.

### 4.2 Positional Parameters & Special Variables

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  SPECIAL VARIABLES                                               │
  │                                                                  │
  │  $0          Script name (full path as invoked)                 │
  │  $1..$9      Positional parameters (arguments)                 │
  │  ${10}       10th argument (braces required for > 9)           │
  │  $#          Number of arguments                                │
  │  $@          All arguments (as separate words)                  │
  │  "$@"        All arguments (preserving quoting) ← USE THIS    │
  │  $*          All arguments (as single string) ← AVOID         │
  │  $?          Exit code of last command                          │
  │  $$          PID of current script                              │
  │  $!          PID of last background process                     │
  │  $FUNCNAME   Current function name                              │
  │  $LINENO     Current line number                                │
  │  $_          Last argument of previous command                  │
  │  $BASH_SOURCE[0]  Path to current script (even when sourced)  │
  │                                                                  │
  │  "$@" vs "$*":                                                   │
  │  Script called with: ./script.sh "hello world" "foo bar"       │
  │  "$@" → "hello world" "foo bar"  (2 args — CORRECT)           │
  │  "$*" → "hello world foo bar"    (1 arg — WRONG)              │
  └──────────────────────────────────────────────────────────────────┘
```

```bash
#!/usr/bin/env bash
set -euo pipefail

# Argument validation pattern
usage() {
    cat <<EOF
Usage: $(basename "$0") <environment> <version> [--dry-run]

Deploy the application to the specified environment.

Arguments:
  environment   Target environment (dev|staging|production)
  version       Version tag to deploy (e.g., v2.1.0)

Options:
  --dry-run     Show what would be done without making changes

Examples:
  $(basename "$0") staging v2.1.0
  $(basename "$0") production v2.1.0 --dry-run
EOF
    exit 1
}

# Require minimum arguments
if [[ $# -lt 2 ]]; then
    echo "ERROR: Missing required arguments" >&2
    usage
fi

env="$1"
version="$2"
dry_run=false

# Check for optional flags
shift 2  # Remove first two args, remaining in "$@"
for arg in "$@"; do
    case "$arg" in
        --dry-run) dry_run=true ;;
        *) echo "Unknown argument: $arg" >&2; usage ;;
    esac
done

# Validate environment
case "$env" in
    dev|staging|production) ;;  # Valid
    *) echo "ERROR: Invalid environment: $env" >&2; usage ;;
esac

echo "Deploying $version to $env (dry_run=$dry_run)"
```

### 4.3 `getopts` — Professional Argument Parsing

```bash
#!/usr/bin/env bash
set -euo pipefail

# getopts — handles -f value, -v, -h style flags
usage() {
    echo "Usage: $(basename "$0") -e <env> -v <version> [-d] [-h]"
    echo "  -e    Environment (required)"
    echo "  -v    Version (required)"
    echo "  -d    Dry run mode"
    echo "  -h    Show this help"
    exit 1
}

environment=""
version=""
dry_run=false

while getopts ":e:v:dh" opt; do
    case $opt in
        e) environment="$OPTARG" ;;
        v) version="$OPTARG" ;;
        d) dry_run=true ;;
        h) usage ;;
        :) echo "ERROR: -$OPTARG requires an argument" >&2; usage ;;
        \?) echo "ERROR: Unknown option -$OPTARG" >&2; usage ;;
    esac
done

# Validate required args
if [[ -z "$environment" || -z "$version" ]]; then
    echo "ERROR: -e and -v are required" >&2
    usage
fi

echo "Deploying $version to $environment (dry_run=$dry_run)"
```

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  getopts SYNTAX EXPLAINED                                       │
  │                                                                  │
  │  getopts ":e:v:dh" opt                                          │
  │           │ │ │ ││                                               │
  │           │ │ │ │└── h = flag (no argument)                     │
  │           │ │ │ └─── d = flag (no argument)                     │
  │           │ │ └──── v: = option that TAKES an argument          │
  │           │ └────── e: = option that TAKES an argument          │
  │           └──────── : = silent error reporting mode             │
  │                       (we handle errors ourselves)               │
  │                                                                  │
  │  OPTARG = value passed to the option                             │
  │  OPTIND = index of next argument to process                     │
  │                                                                  │
  │  Limitation: getopts only handles SHORT options (-e, -v)       │
  │  For --long-options: use a manual while/case loop or getopt    │
  └──────────────────────────────────────────────────────────────────┘
```

### 4.4 User Input & Here Documents

```bash
#!/usr/bin/env bash
set -euo pipefail

# Read interactive input
read -rp "Enter environment (dev/staging/production): " env
read -rsp "Enter password: " password  # -s = silent (no echo)
echo  # newline after silent read

# Read with timeout
if read -rt 10 -p "Continue? (y/n) " answer; then
    [[ "$answer" == "y" ]] || exit 0
else
    echo "Timed out, aborting."
    exit 1
fi

# Read with default
read -rp "Port [8080]: " port
port="${port:-8080}"
```

```bash
# Here Document — multi-line input (variables ARE expanded)
cat <<EOF
Server: ${HOSTNAME}
Date:   $(date)
Status: Deployment complete
EOF

# Here Document to a file
cat > /etc/myapp/config.yaml <<EOF
database:
  host: ${DB_HOST}
  port: ${DB_PORT}
  name: ${DB_NAME}
EOF

# Here Document with NO variable expansion (quote the delimiter)
cat <<'EOF'
This $variable is NOT expanded
Neither is $(this command)
Use this for templates, awk scripts, etc.
EOF

# Here String — single line input to a command
grep "admin" <<< "$user_list"
bc <<< "scale=2; 100/3"
```

> **DevOps Relevance:** Functions are the building blocks of maintainable scripts. Use them to encapsulate actions (deploy, rollback, health-check) and compose them into workflow scripts. `getopts` makes your scripts production-ready with proper argument validation. Here documents are essential for generating config files in provisioning scripts and Dockerfiles.

---

## 5. Text Processing in Scripts

### 5.1 Regex in Bash — Pattern Matching Power

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  REGEX QUICK REFERENCE                                           │
  │                                                                  │
  │  Anchors:                                                        │
  │  ^          Start of line          $    End of line             │
  │  \b         Word boundary                                       │
  │                                                                  │
  │  Character Classes:                                              │
  │  .          Any character          [abc]  a, b, or c            │
  │  [a-z]      Lowercase letter      [^abc] NOT a, b, or c       │
  │  \d         Digit [0-9]            \w     Word char [a-zA-Z0-9_]│
  │  \s         Whitespace                                           │
  │                                                                  │
  │  Quantifiers:                                                    │
  │  *          0 or more              +      1 or more             │
  │  ?          0 or 1                 {n}    Exactly n             │
  │  {n,m}      n to m times          {n,}   n or more             │
  │                                                                  │
  │  Groups & Alternation:                                           │
  │  (abc)      Group                  a|b    a OR b                │
  │  (?:abc)    Non-capturing group                                 │
  │                                                                  │
  │  Common DevOps Patterns:                                         │
  │  ^[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$  IPv4     │
  │  ^[a-f0-9]{40}$                              Git SHA            │
  │  ^v[0-9]+\.[0-9]+\.[0-9]+$                  SemVer tag         │
  │  [0-9]{4}-[0-9]{2}-[0-9]{2}                 Date YYYY-MM-DD    │
  └──────────────────────────────────────────────────────────────────┘
```

### 5.2 grep/sed/awk in Scripts

```bash
#!/usr/bin/env bash
set -euo pipefail

# GREP in scripts — extract and filter
LOG="/var/log/nginx/access.log"

# Count 5xx errors in the last hour
error_count=$(grep "$(date '+%d/%b/%Y:%H')" "$LOG" | grep -c '" 5[0-9][0-9] ' || true)
if [[ "$error_count" -gt 100 ]]; then
    echo "ALERT: $error_count 5xx errors in the last hour" >&2
fi

# Extract unique IPs hitting a specific endpoint
grep "POST /api/login" "$LOG" \
    | awk '{print $1}' \
    | sort \
    | uniq -c \
    | sort -rn \
    | head -20

# SED in scripts — transform config files
configure_app() {
    local config_file="$1"
    local env="$2"

    # Replace values in config (in-place with backup)
    sed -i.bak \
        -e "s|^DB_HOST=.*|DB_HOST=${DB_HOST}|" \
        -e "s|^DB_PORT=.*|DB_PORT=${DB_PORT}|" \
        -e "s|^ENV=.*|ENV=${env}|" \
        "$config_file"

    echo "Config updated: $config_file"
}

# AWK in scripts — structured data processing
# Parse /proc to find memory-hungry processes
ps aux --sort=-%mem | awk '
NR==1 { print $0; next }
NR<=11 {
    printf "%-10s %5s%% MEM  %5s%% CPU  %s\n", $1, $4, $3, $11
}
'

# AWK — generate a report from CSV
awk -F',' '
BEGIN { print "=== Deployment Report ===" }
NR==1 { next }  # skip header
{
    total++
    if ($3 == "success") success++
    else failures++
}
END {
    printf "Total: %d | Success: %d | Failed: %d | Rate: %.1f%%\n",
        total, success, failures, (success/total)*100
}
' deployment_log.csv
```

### 5.3 String Manipulation in Bash

```bash
#!/usr/bin/env bash
set -euo pipefail

# String operations WITHOUT spawning external processes (faster)
url="https://api.example.com:8443/v2/users?active=true"

# Extract parts using parameter expansion
protocol="${url%%://*}"                  # https
host_and_path="${url#*://}"              # api.example.com:8443/v2/users?active=true
host="${host_and_path%%/*}"              # api.example.com:8443
path="/${host_and_path#*/}"             # /v2/users?active=true
host_only="${host%%:*}"                 # api.example.com
port="${host##*:}"                      # 8443

# String contains check
if [[ "$url" == *"example.com"* ]]; then
    echo "URL is for example.com"
fi

# String starts with / ends with
if [[ "$url" == https://* ]]; then echo "HTTPS URL"; fi
if [[ "$url" == *.com* ]]; then echo "Dot-com domain"; fi

# Replace in string
new_url="${url/http:/https:}"          # Replace first
cleaned="${url//[?&]/ }"                # Replace all special chars
```

> **⚠️ Gotcha (sed delimiter):** When your replacement string contains `/` (like file paths), use a different delimiter: `sed 's|/old/path|/new/path|g'` instead of `sed 's//old/path//new/path/g'` which breaks. The `|` or `#` separator is common.

> **DevOps Relevance:** Text processing is 50% of a DevOps engineer's scripting work. Log analysis, config templating, data extraction from API responses, CSV report generation — all rely on grep/sed/awk. Learn to chain them in pipelines for powerful one-liners that replace pages of Python code.

---

## 6. File Operations & Process Management

### 6.1 Secure Temporary Files

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  TEMPORARY FILES — DO IT RIGHT                                   │
  │                                                                  │
  │  BAD:                                                            │
  │  tmp_file="/tmp/mydata.txt"       ← Predictable name           │
  │                                    ← Race condition (TOCTOU)   │
  │                                    ← Security risk             │
  │                                                                  │
  │  GOOD:                                                           │
  │  tmp_file=$(mktemp)               ← Random name, created       │
  │  tmp_dir=$(mktemp -d)             ← Random temp directory      │
  │  mktemp /tmp/myapp.XXXXXX        ← Custom prefix + random     │
  │                                                                  │
  │  ALWAYS clean up with a trap (see 6.2)                          │
  └──────────────────────────────────────────────────────────────────┘
```

```bash
#!/usr/bin/env bash
set -euo pipefail

# Create temp file and ensure cleanup
tmp_file=$(mktemp)
tmp_dir=$(mktemp -d)

# Cleanup trap — runs on exit, error, or interrupt
cleanup() {
    rm -f "$tmp_file"
    rm -rf "$tmp_dir"
}
trap cleanup EXIT

# Now use tmp_file safely
curl -sf "https://api.example.com/data" > "$tmp_file"
jq '.results[]' "$tmp_file" | while read -r item; do
    echo "Processing: $item"
done

# tmp_file and tmp_dir are automatically cleaned up on exit
```

### 6.2 Traps & Signal Handling

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  SIGNALS & TRAPS                                                 │
  │                                                                  │
  │  Signal    Number  Default Action  Sent By                      │
  │  ─────────────────────────────────────────────                   │
  │  SIGHUP    1       Terminate       Terminal closed               │
  │  SIGINT    2       Terminate       Ctrl+C                       │
  │  SIGQUIT   3       Core dump       Ctrl+\                       │
  │  SIGKILL   9       Terminate       kill -9 (CANNOT be caught!) │
  │  SIGTERM   15      Terminate       kill (default), K8s shutdown │
  │  SIGUSR1   10      Terminate       User-defined                 │
  │  SIGUSR2   12      Terminate       User-defined                 │
  │                                                                  │
  │  Bash pseudo-signals:                                            │
  │  EXIT      —       —               Script exit (any reason)     │
  │  ERR       —       —               Command fails (with set -e) │
  │  DEBUG     —       —               Before each command          │
  │  RETURN    —       —               After function/source return │
  │                                                                  │
  │  trap 'commands' SIGNAL [SIGNAL...]                             │
  │  trap - SIGNAL                     ← Reset to default          │
  └──────────────────────────────────────────────────────────────────┘
```

```bash
#!/usr/bin/env bash
set -euo pipefail

# Comprehensive trap pattern for production scripts
LOCK_FILE="/var/run/myapp-deploy.lock"
LOG_FILE="/var/log/myapp-deploy.log"

cleanup() {
    local exit_code=$?
    rm -f "$LOCK_FILE"
    if [[ $exit_code -ne 0 ]]; then
        log "ERROR" "Script failed with exit code $exit_code"
        # Send alert to Slack/PagerDuty
        notify_failure "$exit_code"
    fi
    log "INFO" "Cleanup complete"
}

trap cleanup EXIT         # Always runs on exit
trap 'exit 130' INT       # Ctrl+C → triggers EXIT trap
trap 'exit 143' TERM      # kill → triggers EXIT trap

# Error handler with context
on_error() {
    local line="$1"
    local cmd="$2"
    log "ERROR" "Command '$cmd' failed at line $line"
}
trap 'on_error $LINENO "$BASH_COMMAND"' ERR
```

### 6.3 File Locking with `flock`

```bash
#!/usr/bin/env bash
set -euo pipefail

# Prevent concurrent script execution (critical for cron jobs)
LOCK_FILE="/var/run/backup.lock"

exec 200>"$LOCK_FILE"
if ! flock -n 200; then
    echo "Another instance is already running. Exiting." >&2
    exit 1
fi

# This code only runs if we got the lock
echo "Running backup..."
# ... backup logic ...

# Lock is automatically released when script exits (fd 200 closes)
```

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  WHY FILE LOCKING MATTERS                                       │
  │                                                                  │
  │  Without lock:                    With flock:                    │
  │  ┌──────────────────┐            ┌──────────────────┐           │
  │  │ Cron fires       │            │ Cron fires       │           │
  │  │ backup at 02:00  │            │ backup at 02:00  │           │
  │  │                  │            │ Gets lock ✅      │           │
  │  │ Backup takes     │            │                  │           │
  │  │ 90 minutes       │            │ 02:30 — still   │           │
  │  │                  │            │ running          │           │
  │  │ 03:00 — cron     │            │                  │           │
  │  │ fires AGAIN      │            │ 03:00 — cron    │           │
  │  │ TWO backups!     │            │ fires, can't    │           │
  │  │ Corrupted data 💥│            │ get lock, exits │           │
  │  └──────────────────┘            │ safely ✅        │           │
  │                                  └──────────────────┘           │
  └──────────────────────────────────────────────────────────────────┘
```

### 6.4 Background Processes & Job Control

```bash
#!/usr/bin/env bash
set -euo pipefail

# Run tasks in parallel and wait for all
pids=()
for server in web01 web02 web03 db01; do
    (
        echo "Deploying to $server..."
        ssh "$server" '/opt/deploy/update.sh' 2>&1 | sed "s/^/[$server] /"
    ) &
    pids+=($!)
done

# Wait for all background jobs
failed=0
for pid in "${pids[@]}"; do
    if ! wait "$pid"; then
        ((failed++))
    fi
done

if [[ $failed -gt 0 ]]; then
    echo "ERROR: $failed deployments failed" >&2
    exit 1
fi

echo "All deployments successful!"
```

> **⚠️ Gotcha (Background & set -e):** `set -e` does NOT apply to background processes (`cmd &`). A failing background job won't kill your script — you must explicitly `wait` for it and check the exit code.

> **DevOps Relevance:** File locking prevents duplicate cron jobs (a very common production bug). Trap handlers ensure cleanup happens even during failures — preventing stale lock files, temp file leaks, and incomplete deployments. Parallel execution via `&` and `wait` speeds up multi-server operations dramatically.

---

## 7. Real-World Bash Patterns

### 7.1 Health Check Script

```bash
#!/usr/bin/env bash
# health_check.sh — Check service health with alerting
set -euo pipefail

readonly SERVICES=(
    "http://web01:8080/health"
    "http://web02:8080/health"
    "http://api01:3000/health"
)
readonly SLACK_WEBHOOK="${SLACK_WEBHOOK:-}"
readonly TIMEOUT=5

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

send_alert() {
    local message="$1"
    if [[ -n "$SLACK_WEBHOOK" ]]; then
        curl -sf -X POST "$SLACK_WEBHOOK" \
            -H 'Content-Type: application/json' \
            -d "{\"text\": \"🚨 $message\"}" \
            > /dev/null || log "WARN: Failed to send Slack alert"
    fi
}

check_service() {
    local url="$1"
    local http_code
    http_code=$(curl -sf -o /dev/null -w '%{http_code}' \
        --max-time "$TIMEOUT" "$url" 2>/dev/null || echo "000")

    if [[ "$http_code" == "200" ]]; then
        log "OK:   $url (HTTP $http_code)"
        return 0
    else
        log "FAIL: $url (HTTP $http_code)"
        return 1
    fi
}

# Main
failed=0
for service in "${SERVICES[@]}"; do
    if ! check_service "$service"; then
        ((failed++))
        send_alert "Service DOWN: $service"
    fi
done

if [[ $failed -gt 0 ]]; then
    log "SUMMARY: $failed/${#SERVICES[@]} services unhealthy"
    exit 1
else
    log "SUMMARY: All ${#SERVICES[@]} services healthy ✅"
fi
```

### 7.2 Deployment Script with Rollback

```bash
#!/usr/bin/env bash
# deploy.sh — Blue-green style deployment with rollback
set -euo pipefail

readonly APP_DIR="/opt/myapp"
readonly RELEASES_DIR="${APP_DIR}/releases"
readonly CURRENT_LINK="${APP_DIR}/current"
readonly KEEP_RELEASES=5

usage() {
    echo "Usage: $(basename "$0") <version>"
    echo "  Example: $(basename "$0") v2.3.1"
    exit 1
}

log() { echo "[$(date '+%H:%M:%S')] $1"; }

[[ $# -eq 1 ]] || usage
readonly VERSION="$1"
readonly RELEASE_DIR="${RELEASES_DIR}/${VERSION}"
readonly PREVIOUS=$(readlink -f "$CURRENT_LINK" 2>/dev/null || echo "")

# Rollback function
rollback() {
    if [[ -n "$PREVIOUS" && -d "$PREVIOUS" ]]; then
        log "⚠️ ROLLING BACK to $PREVIOUS"
        ln -sfn "$PREVIOUS" "$CURRENT_LINK"
        systemctl restart myapp
        log "Rollback complete"
    else
        log "ERROR: No previous release to rollback to"
    fi
}
trap 'rollback' ERR

# Deploy
log "Deploying $VERSION..."

mkdir -p "$RELEASE_DIR"

log "1. Downloading artifact..."
curl -sfL "https://artifacts.example.com/myapp-${VERSION}.tar.gz" \
    | tar -xz -C "$RELEASE_DIR"

log "2. Installing dependencies..."
cd "$RELEASE_DIR" && npm ci --production 2>&1 | tail -1

log "3. Running migrations..."
"${RELEASE_DIR}/bin/migrate" up

log "4. Switching symlink..."
ln -sfn "$RELEASE_DIR" "$CURRENT_LINK"

log "5. Restarting service..."
systemctl restart myapp

log "6. Verifying health..."
sleep 5
if ! curl -sf http://localhost:8080/health > /dev/null; then
    log "Health check failed!"
    exit 1  # Triggers ERR trap → rollback
fi

# Cleanup old releases
log "7. Cleaning old releases (keeping $KEEP_RELEASES)..."
ls -dt "${RELEASES_DIR}"/*/ | tail -n +$((KEEP_RELEASES + 1)) \
    | xargs -r rm -rf

log "✅ Deployment of $VERSION complete!"
```

### 7.3 Log Rotation & Cleanup

```bash
#!/usr/bin/env bash
# log_cleanup.sh — Rotate and compress old logs
set -euo pipefail

readonly LOG_DIR="/var/log/myapp"
readonly MAX_AGE_DAYS=30
readonly COMPRESS_AFTER_DAYS=1

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"; }

# Compress logs older than 1 day (not already compressed)
compress_old_logs() {
    find "$LOG_DIR" -name "*.log" -mtime "+${COMPRESS_AFTER_DAYS}" -print0 \
        | while IFS= read -r -d '' file; do
            log "Compressing: $file"
            gzip "$file"
        done
}

# Delete compressed logs older than max age
delete_old_logs() {
    local count
    count=$(find "$LOG_DIR" -name "*.gz" -mtime "+${MAX_AGE_DAYS}" | wc -l)
    if [[ "$count" -gt 0 ]]; then
        log "Deleting $count files older than $MAX_AGE_DAYS days"
        find "$LOG_DIR" -name "*.gz" -mtime "+${MAX_AGE_DAYS}" -delete
    fi
}

# Report disk usage
report_usage() {
    local usage
    usage=$(du -sh "$LOG_DIR" | awk '{print $1}')
    log "Log directory size: $usage"
}

# Main
log "=== Log cleanup started ==="
compress_old_logs
delete_old_logs
report_usage
log "=== Log cleanup complete ==="
```

### 7.4 Backup Script with Retention

```bash
#!/usr/bin/env bash
# backup.sh — Database backup with retention policy
set -euo pipefail

readonly BACKUP_DIR="/backups/postgres"
readonly DB_NAME="${DB_NAME:-myapp}"
readonly DB_HOST="${DB_HOST:-localhost}"
readonly TIMESTAMP=$(date +%Y%m%d_%H%M%S)
readonly BACKUP_FILE="${BACKUP_DIR}/${DB_NAME}_${TIMESTAMP}.sql.gz"
readonly KEEP_DAILY=7
readonly KEEP_WEEKLY=4

LOCK_FILE="/var/run/backup-${DB_NAME}.lock"
exec 200>"$LOCK_FILE"
flock -n 200 || { echo "Backup already running"; exit 1; }

log() { echo "[$(date '+%H:%M:%S')] $1" | tee -a "${BACKUP_DIR}/backup.log"; }

cleanup() {
    local exit_code=$?
    if [[ $exit_code -ne 0 ]]; then
        rm -f "$BACKUP_FILE"  # Remove partial backup
        log "ERROR: Backup failed (exit code: $exit_code)"
    fi
}
trap cleanup EXIT

# Ensure directory exists
mkdir -p "$BACKUP_DIR"

# Create backup
log "Starting backup of $DB_NAME..."
pg_dump -h "$DB_HOST" -U postgres "$DB_NAME" \
    | gzip > "$BACKUP_FILE"

# Verify backup is not empty
if [[ ! -s "$BACKUP_FILE" ]]; then
    log "ERROR: Backup file is empty!"
    exit 1
fi

size=$(du -h "$BACKUP_FILE" | awk '{print $1}')
log "Backup created: $BACKUP_FILE ($size)"

# Retention — keep last N daily backups
log "Applying retention policy..."
ls -t "${BACKUP_DIR}/${DB_NAME}_"*.sql.gz 2>/dev/null \
    | tail -n +$((KEEP_DAILY + 1)) \
    | xargs -r rm -f

log "✅ Backup complete"
```

### 7.5 Retry with Exponential Backoff

```bash
#!/usr/bin/env bash
set -euo pipefail

# Generic retry function with exponential backoff
retry() {
    local max_attempts="${1}"
    local base_delay="${2}"
    shift 2
    local cmd=("$@")

    local attempt=1
    local delay="$base_delay"

    until "${cmd[@]}"; do
        if [[ $attempt -ge $max_attempts ]]; then
            echo "ERROR: '${cmd[*]}' failed after $attempt attempts" >&2
            return 1
        fi
        echo "Attempt $attempt failed. Retrying in ${delay}s..." >&2
        sleep "$delay"
        ((attempt++))
        delay=$((delay * 2))   # Exponential: 2, 4, 8, 16...
    done
    echo "Success on attempt $attempt"
}

# Usage
retry 5 2 curl -sf "https://api.example.com/data"
retry 3 1 docker push "registry.example.com/myapp:v2.0"
retry 4 5 ssh web01 "systemctl restart nginx"
```

> **DevOps Relevance:** These five patterns (health check, deployment, log rotation, backup, retry) cover 80% of what you'll write in production. Notice the common elements: logging, error handling, cleanup traps, lock files, and verification steps. Steal these templates and adapt them — don't start from scratch.

---

## 8. Bash Debugging & Best Practices

### 8.1 Debugging Techniques

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  BASH DEBUGGING TOOLKIT                                          │
  │                                                                  │
  │  set -x          Print every command before executing           │
  │                  (trace mode — shows exact expansion)           │
  │                                                                  │
  │  set +x          Turn off tracing                               │
  │                                                                  │
  │  bash -x script.sh    Run entire script in trace mode          │
  │                                                                  │
  │  PS4='+(${BASH_SOURCE}:${LINENO}): ${FUNCNAME[0]:+${FUNCNAME   │
  │  [0]}(): }'                                                      │
  │                  Better trace output with file, line, function │
  │                                                                  │
  │  trap 'echo "DEBUG: $BASH_COMMAND"' DEBUG                       │
  │                  Print each command (alternative to set -x)    │
  │                                                                  │
  │  echo "DEBUG: var=$var" >&2                                     │
  │                  Manual debug prints (to stderr)               │
  └──────────────────────────────────────────────────────────────────┘
```

```bash
#!/usr/bin/env bash
set -euo pipefail

# Targeted debugging — trace only specific sections
deploy() {
    set -x   # Enable tracing
    rsync -avz ./build/ "$server:/opt/app/"
    ssh "$server" "systemctl restart myapp"
    set +x   # Disable tracing
}

# Better PS4 for debugging (add to top of script)
export PS4='+(${BASH_SOURCE[0]}:${LINENO}): ${FUNCNAME[0]:+${FUNCNAME[0]}(): }'
# Output: +(deploy.sh:15): deploy(): rsync -avz ./build/ web01:/opt/app/
```

### 8.2 ShellCheck — The Linter You MUST Use

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  SHELLCHECK — STATIC ANALYSIS FOR SHELL SCRIPTS                 │
  │                                                                  │
  │  Install:                                                        │
  │  apt install shellcheck     # Debian/Ubuntu                     │
  │  brew install shellcheck    # macOS                              │
  │  yum install ShellCheck     # RHEL/CentOS                      │
  │                                                                  │
  │  Usage:                                                          │
  │  shellcheck myscript.sh     # Check a script                   │
  │  shellcheck -x myscript.sh  # Follow source'd files            │
  │  shellcheck -f json *.sh    # JSON output (for CI)             │
  │                                                                  │
  │  What it catches:                                                │
  │  ├── Unquoted variables (SC2086)                                │
  │  ├── Using [ ] instead of [[ ]] (SC2039)                       │
  │  ├── Useless cat: cat file | grep → grep file (SC2002)        │
  │  ├── cd failures not handled (SC2164)                           │
  │  ├── Globbing in quotes (SC2035)                                │
  │  └── Hundreds more...                                            │
  │                                                                  │
  │  CI Integration:                                                 │
  │  # GitHub Actions                                                │
  │  - name: ShellCheck                                              │
  │    uses: ludeeus/action-shellcheck@master                       │
  │                                                                  │
  │  RULE: Every shell script MUST pass shellcheck in CI            │
  └──────────────────────────────────────────────────────────────────┘
```

### 8.3 Error Handling Patterns

```bash
#!/usr/bin/env bash
set -euo pipefail

# Pattern 1: Die function
die() {
    echo "FATAL: $1" >&2
    exit "${2:-1}"
}

# Pattern 2: Require commands
require_cmd() {
    for cmd in "$@"; do
        command -v "$cmd" > /dev/null 2>&1 \
            || die "Required command not found: $cmd"
    done
}
require_cmd jq curl aws kubectl

# Pattern 3: Require environment variables
require_env() {
    for var in "$@"; do
        [[ -n "${!var:-}" ]] || die "Required environment variable not set: $var"
    done
}
require_env AWS_REGION CLUSTER_NAME DEPLOY_TOKEN

# Pattern 4: Conditional error handling (allow specific failures)
# With set -e, wrap allowed-to-fail commands:
count=$(grep -c "ERROR" /var/log/app.log 2>/dev/null || echo "0")

# Pattern 5: Validate before executing
pre_flight_checks() {
    log "Running pre-flight checks..."

    [[ -f "$CONFIG_FILE" ]]     || die "Config not found: $CONFIG_FILE"
    [[ -r "$CONFIG_FILE" ]]     || die "Config not readable: $CONFIG_FILE"
    is_port_open "$DB_HOST" 5432 || die "Database unreachable"
    [[ $(df --output=avail /opt | tail -1) -gt 1048576 ]] \
                                || die "Less than 1GB free on /opt"

    log "Pre-flight checks passed ✅"
}
```

### 8.4 Logging Framework

```bash
#!/usr/bin/env bash
set -euo pipefail

# Reusable logging framework — source this in all your scripts
readonly LOG_LEVEL="${LOG_LEVEL:-INFO}"
readonly LOG_FILE="${LOG_FILE:-/dev/null}"

# Color codes
readonly RED='\033[0;31m'
readonly YELLOW='\033[0;33m'
readonly GREEN='\033[0;32m'
readonly BLUE='\033[0;34m'
readonly NC='\033[0m'  # No Color

_log() {
    local level="$1"
    local color="$2"
    local message="$3"
    local timestamp
    timestamp=$(date '+%Y-%m-%d %H:%M:%S')

    # Write to stderr (so stdout is clean for data)
    echo -e "${color}[${timestamp}] [${level}] ${message}${NC}" >&2

    # Also write to log file (without colors)
    echo "[${timestamp}] [${level}] ${message}" >> "$LOG_FILE"
}

log_debug() { [[ "$LOG_LEVEL" == "DEBUG" ]] && _log "DEBUG" "$BLUE" "$1" || true; }
log_info()  { _log "INFO " "$GREEN"  "$1"; }
log_warn()  { _log "WARN " "$YELLOW" "$1"; }
log_error() { _log "ERROR" "$RED"    "$1"; }

# Usage
log_info "Starting deployment..."
log_warn "Disk space below 20%"
log_error "Connection refused"
log_debug "Variable state: count=$count"
```

### 8.5 Bash Style Guide & Conventions

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  BASH STYLE GUIDE (based on Google Shell Style Guide)           │
  │                                                                  │
  │  FILE:                                                           │
  │  ├── Shebang: #!/usr/bin/env bash                               │
  │  ├── set -euo pipefail on line 2                                │
  │  ├── Constants at top as readonly                                │
  │  ├── Functions before main logic                                 │
  │  ├── main() function for large scripts                          │
  │  └── Call main "$@" at the bottom                               │
  │                                                                  │
  │  NAMING:                                                         │
  │  ├── Variables:    lower_snake_case                              │
  │  ├── Constants:    UPPER_SNAKE_CASE                              │
  │  ├── Functions:    lower_snake_case (verbs: get_, check_, run_) │
  │  ├── Files:        lower_snake_case.sh                          │
  │  └── Env vars:     UPPER_SNAKE_CASE                              │
  │                                                                  │
  │  QUOTING:                                                        │
  │  ├── Always double-quote variables: "$var"                      │
  │  ├── Always double-quote arrays: "${arr[@]}"                    │
  │  └── Use single quotes for literal strings: 'no $expansion'    │
  │                                                                  │
  │  FORMATTING:                                                     │
  │  ├── Indent with 2 or 4 spaces (never tabs)                    │
  │  ├── Max line length: 80 characters                             │
  │  ├── Opening brace on same line: func() {                      │
  │  └── Use blank lines to separate logical sections              │
  │                                                                  │
  │  BEST PRACTICES:                                                 │
  │  ├── Use [[ ]] instead of [ ] for tests                        │
  │  ├── Use $() instead of backticks ``                            │
  │  ├── Use local in functions                                      │
  │  ├── Use arrays instead of space-separated strings             │
  │  ├── Send errors to stderr: echo "error" >&2                   │
  │  ├── Use meaningful exit codes                                   │
  │  ├── Clean up with trap EXIT                                     │
  │  ├── Run ShellCheck in CI                                        │
  │  └── Comment the "why", not the "what"                          │
  └──────────────────────────────────────────────────────────────────┘
```

```bash
#!/usr/bin/env bash
# deploy_service.sh — Deploy a service to the target environment.
# Usage: ./deploy_service.sh -e <env> -v <version> [-d]
set -euo pipefail

# --- Constants ---
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly APP_NAME="myapp"

# --- Source libraries ---
source "${SCRIPT_DIR}/lib/logging.sh"
source "${SCRIPT_DIR}/lib/utils.sh"

# --- Functions ---
usage() {
    cat <<EOF
Usage: $(basename "$0") -e <env> -v <version> [-d]
  -e    Target environment (dev|staging|production)
  -v    Version to deploy
  -d    Dry run
  -h    Help
EOF
    exit 1
}

main() {
    local environment=""
    local version=""
    local dry_run=false

    while getopts ":e:v:dh" opt; do
        case $opt in
            e) environment="$OPTARG" ;;
            v) version="$OPTARG" ;;
            d) dry_run=true ;;
            h) usage ;;
            *) usage ;;
        esac
    done

    [[ -n "$environment" && -n "$version" ]] || usage

    log_info "Deploying $APP_NAME $version to $environment"

    pre_flight_checks "$environment"
    download_artifact "$version"
    run_migrations "$environment"
    swap_symlink "$version"
    restart_service
    verify_health

    log_info "Deployment complete ✅"
}

main "$@"
```

> **⚠️ Gotcha (SCRIPT_DIR):** `$(dirname "$0")` can return a relative path. Use `$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)` to always get an absolute path. `BASH_SOURCE[0]` works even when the script is `source`d from another script, unlike `$0`.

> **DevOps Relevance:** Debugging and best practices separate hobby scripts from production-grade automation. ShellCheck in CI prevents entire classes of bugs. A consistent logging framework makes it possible to grep production script logs at 3 AM. The `main "$@"` pattern makes scripts testable and modular. Follow the Google Shell Style Guide — it exists because Google runs millions of shell scripts in production.

---

## 9. Python Fundamentals for DevOps

### 9.1 Why Python for DevOps

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  PYTHON IN THE DEVOPS ECOSYSTEM                                  │
  │                                                                  │
  │  Where Python Lives:                                             │
  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
  │  │ Ansible      │  │ AWS Lambda   │  │ Kubernetes   │          │
  │  │ (100% Python)│  │ (Python      │  │ client-python│          │
  │  │              │  │  runtime)    │  │              │          │
  │  ├──────────────┤  ├──────────────┤  ├──────────────┤          │
  │  │ SaltStack    │  │ Boto3 (AWS)  │  │ Prometheus   │          │
  │  │ OpenStack    │  │ Azure SDK    │  │ exporters    │          │
  │  │ Fabric       │  │ GCP client   │  │ Grafana API  │          │
  │  └──────────────┘  └──────────────┘  └──────────────┘          │
  │                                                                  │
  │  Python's DevOps Strengths:                                      │
  │  ├── Rich ecosystem: pip has 400K+ packages                    │
  │  ├── Cloud SDKs: first-class support from AWS/GCP/Azure        │
  │  ├── Readable: easy to review in PRs                            │
  │  ├── Error handling: try/except beats bash set -e               │
  │  ├── Data structures: dicts, lists, sets for complex logic     │
  │  └── Testing: pytest makes scripts testable                    │
  └──────────────────────────────────────────────────────────────────┘
```

### 9.2 Virtual Environments — Isolate Your Dependencies

```bash
# Create a virtual environment (ALWAYS do this for projects)
python3 -m venv .venv

# Activate it
source .venv/bin/activate         # Linux/macOS
# .venv\Scripts\activate          # Windows

# Install dependencies
pip install requests pyyaml boto3
pip freeze > requirements.txt     # Lock versions

# In another machine/container:
pip install -r requirements.txt   # Reproduce exact environment

# Deactivate
deactivate
```

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  VIRTUAL ENVIRONMENT MENTAL MODEL                                │
  │                                                                  │
  │  System Python (/usr/bin/python3)                               │
  │  └── System packages (DON'T install here with pip!)            │
  │                                                                  │
  │  Project A (.venv/)              Project B (.venv/)             │
  │  ├── requests==2.31.0           ├── requests==2.28.0           │
  │  ├── boto3==1.34.0              ├── flask==3.0.0               │
  │  └── pyyaml==6.0.1              └── pyyaml==5.4.1              │
  │                                                                  │
  │  Each project has its OWN dependencies → no conflicts          │
  │                                                                  │
  │  RULE: Never pip install as root / sudo pip install             │
  │  RULE: Always use a venv or Docker                              │
  └──────────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha (system pip):** Running `sudo pip install` on a server can BREAK the system's own Python packages (like `apt` on Ubuntu, which depends on Python). Always use a virtual environment or install in a container.

### 9.3 Data Types for DevOps

```python
#!/usr/bin/env python3
"""Essential Python data types for DevOps scripting."""

# --- Strings (f-strings are your best friend) ---
server = "web01"
port = 8080
env = "production"

# F-strings (Python 3.6+) — the ONLY string format you should use
url = f"http://{server}:{port}/health"
log_msg = f"[{env.upper()}] Deploying to {server}"

# Multi-line strings (for templates, configs)
nginx_config = f"""\
server {{
    listen {port};
    server_name {server}.example.com;
    location / {{
        proxy_pass http://127.0.0.1:3000;
    }}
}}
"""

# --- Lists (ordered, mutable) ---
servers = ["web01", "web02", "web03", "db01"]
servers.append("cache01")       # Add to end
servers.extend(["lb01", "lb02"])  # Add multiple
failed = servers.pop(0)          # Remove & return first
healthy = [s for s in servers if s.startswith("web")]  # Filter

# --- Dictionaries (key-value pairs — use EVERYWHERE) ---
deploy_config = {
    "app_name": "myapp",
    "version": "v2.3.1",
    "replicas": 3,
    "env_vars": {
        "DB_HOST": "db01.internal",
        "REDIS_URL": "redis://cache01:6379",
    },
    "regions": ["us-east-1", "eu-west-1"],
}

# Access with .get() to avoid KeyError
db_host = deploy_config.get("db_host", "localhost")  # Default if missing
version = deploy_config["version"]                     # Raises KeyError if missing

# --- Sets (unique values — great for dedup) ---
allowed_envs = {"dev", "staging", "production"}
requested_env = "staging"
if requested_env not in allowed_envs:
    raise ValueError(f"Invalid env: {requested_env}")

# Find common servers between two lists
team_a_servers = {"web01", "web02", "db01"}
team_b_servers = {"web02", "db01", "cache01"}
shared = team_a_servers & team_b_servers        # {'web02', 'db01'}
only_a = team_a_servers - team_b_servers        # {'web01'}
```

### 9.4 Comprehensions — Write Less, Do More

```python
#!/usr/bin/env python3
"""Comprehensions replace loops for data transformation."""

servers = ["web01", "web02", "db01", "cache01", "web03"]

# List comprehension — filter and transform
web_servers = [s for s in servers if s.startswith("web")]
# → ['web01', 'web02', 'web03']

upper_servers = [s.upper() for s in servers]
# → ['WEB01', 'WEB02', 'DB01', 'CACHE01', 'WEB03']

# Dict comprehension — build dicts from data
server_ports = {s: 8080 + i for i, s in enumerate(servers)}
# → {'web01': 8080, 'web02': 8081, 'db01': 8082, ...}

# Filter dict
healthy = {s: status for s, status in health_results.items() if status == "ok"}

# Set comprehension — unique values
regions = {server.split("-")[0] for server in ["us-east-web01", "eu-west-db01", "us-east-web02"]}
# → {'us-east', 'eu-west'}

# Conditional expression in comprehension
labels = [f"{s} ✅" if s in healthy else f"{s} ❌" for s in servers]
```

> **DevOps Relevance:** Python's data structures are why it's preferred over Bash for complex automation. When you need to parse JSON API responses, merge configuration dictionaries, deduplicate server lists, or transform deployment manifests — Python comprehensions and dicts make this trivial. If your Bash script grows past ~50 lines of data manipulation, rewrite it in Python.

---

## 10. File I/O, JSON, YAML & APIs

### 10.1 File Handling

```python
#!/usr/bin/env python3
"""File operations for DevOps scripts."""
from pathlib import Path

# --- pathlib (modern, preferred over os.path) ---
config_dir = Path("/etc/myapp")
config_file = config_dir / "config.yaml"      # Path joining with /

# Check existence
if config_file.exists() and config_file.is_file():
    content = config_file.read_text()
    print(f"Config size: {config_file.stat().st_size} bytes")

# Write to file
output = Path("/var/log/myapp/deploy.log")
output.parent.mkdir(parents=True, exist_ok=True)  # mkdir -p equivalent
output.write_text(f"Deployed at {datetime.now()}\n")

# Append to file
with open(output, "a") as f:
    f.write("Additional log entry\n")

# Read line by line (memory efficient for large files)
with open("/var/log/syslog") as f:
    for line in f:
        if "error" in line.lower():
            print(line.strip())

# Glob patterns
log_files = list(Path("/var/log").glob("*.log"))
all_yamls = list(Path("/etc").rglob("*.yaml"))  # Recursive

# Temp files (auto-cleanup)
import tempfile
with tempfile.NamedTemporaryFile(mode="w", suffix=".json", delete=True) as tmp:
    tmp.write('{"key": "value"}')
    tmp.flush()
    print(f"Temp file: {tmp.name}")
# File is automatically deleted when block exits
```

### 10.2 JSON — The DevOps Data Format

```python
#!/usr/bin/env python3
"""JSON parsing and generation for API automation."""
import json
from pathlib import Path

# --- Parse JSON ---
# From string
api_response = '{"status": "healthy", "version": "2.3.1", "uptime": 86400}'
data = json.loads(api_response)
print(data["status"])           # "healthy"
print(data.get("missing", "default"))  # "default"

# From file
config = json.loads(Path("config.json").read_text())

# With open (streaming - better for large files)
with open("large_data.json") as f:
    data = json.load(f)

# --- Generate JSON ---
deploy_info = {
    "app": "myapp",
    "version": "v2.3.1",
    "timestamp": "2024-01-15T10:30:00Z",
    "servers": ["web01", "web02"],
    "config": {"replicas": 3, "memory": "512Mi"},
}

# Pretty print (for config files, debugging)
json_str = json.dumps(deploy_info, indent=2, sort_keys=True)
print(json_str)

# Write to file
Path("deploy_manifest.json").write_text(
    json.dumps(deploy_info, indent=2)
)

# --- Nested JSON Navigation ---
# Terraform state, K8s API responses are deeply nested
k8s_response = {
    "items": [
        {
            "metadata": {"name": "myapp-abc123", "namespace": "production"},
            "status": {"phase": "Running", "containerStatuses": [
                {"name": "app", "ready": True, "restartCount": 0}
            ]}
        }
    ]
}

# Safe nested access pattern
for pod in k8s_response.get("items", []):
    name = pod.get("metadata", {}).get("name", "unknown")
    phase = pod.get("status", {}).get("phase", "Unknown")
    containers = pod.get("status", {}).get("containerStatuses", [])
    restarts = sum(c.get("restartCount", 0) for c in containers)
    print(f"Pod: {name} | Phase: {phase} | Restarts: {restarts}")
```

### 10.3 YAML — Configuration Language of DevOps

```python
#!/usr/bin/env python3
"""YAML handling — the language of K8s, Ansible, Docker Compose."""
import yaml  # pip install pyyaml
from pathlib import Path

# --- Parse YAML ---
yaml_content = """
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
    env: production
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: app
          image: myapp:v2.3.1
          ports:
            - containerPort: 8080
          env:
            - name: DB_HOST
              value: db01.internal
          resources:
            limits:
              memory: "512Mi"
              cpu: "500m"
"""

data = yaml.safe_load(yaml_content)  # ALWAYS use safe_load, never load()
print(data["metadata"]["name"])       # "myapp"
print(data["spec"]["replicas"])       # 3

# From file
with open("deployment.yaml") as f:
    manifest = yaml.safe_load(f)

# --- Multi-document YAML (common in K8s) ---
multi_doc = """
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
"""

for doc in yaml.safe_load_all(multi_doc):
    if doc:
        print(f"Kind: {doc['kind']}, Name: {doc['metadata']['name']}")

# --- Generate YAML ---
config = {
    "database": {
        "host": "db01.internal",
        "port": 5432,
        "name": "myapp_prod",
    },
    "redis": {
        "url": "redis://cache01:6379/0",
    },
    "features": ["auth", "caching", "monitoring"],
}

yaml_output = yaml.dump(config, default_flow_style=False, sort_keys=False)
Path("generated_config.yaml").write_text(yaml_output)

# --- Modify existing YAML ---
def update_image_tag(manifest_path: str, new_tag: str) -> None:
    """Update container image tag in a K8s deployment manifest."""
    manifest = yaml.safe_load(Path(manifest_path).read_text())

    containers = manifest["spec"]["template"]["spec"]["containers"]
    for container in containers:
        image_name = container["image"].split(":")[0]
        container["image"] = f"{image_name}:{new_tag}"

    Path(manifest_path).write_text(
        yaml.dump(manifest, default_flow_style=False)
    )
    print(f"Updated {manifest_path} to tag {new_tag}")

# Usage
update_image_tag("deployment.yaml", "v2.4.0")
```

> **⚠️ Gotcha (yaml.load vs yaml.safe_load):** NEVER use `yaml.load()` — it can execute arbitrary Python code from YAML input (remote code execution vulnerability). ALWAYS use `yaml.safe_load()`. This is a real security issue that has been exploited in the wild.

### 10.4 REST APIs with `requests`

```python
#!/usr/bin/env python3
"""HTTP/REST API automation with requests library."""
import requests  # pip install requests
import sys
from time import sleep

# --- GET request ---
def check_service_health(url: str, timeout: int = 5) -> bool:
    """Check if a service is healthy."""
    try:
        response = requests.get(f"{url}/health", timeout=timeout)
        response.raise_for_status()  # Raises for 4xx/5xx
        data = response.json()
        return data.get("status") == "healthy"
    except requests.ConnectionError:
        print(f"Cannot connect to {url}", file=sys.stderr)
        return False
    except requests.Timeout:
        print(f"Timeout connecting to {url}", file=sys.stderr)
        return False
    except requests.HTTPError as e:
        print(f"HTTP error from {url}: {e.response.status_code}", file=sys.stderr)
        return False

# --- POST request (create resource) ---
def create_dns_record(domain: str, ip: str, api_token: str) -> dict:
    """Create a DNS A record via API."""
    response = requests.post(
        "https://api.cloudflare.com/client/v4/zones/ZONE_ID/dns_records",
        headers={
            "Authorization": f"Bearer {api_token}",
            "Content-Type": "application/json",
        },
        json={
            "type": "A",
            "name": domain,
            "content": ip,
            "ttl": 300,
        },
        timeout=10,
    )
    response.raise_for_status()
    return response.json()

# --- Session for multiple requests (connection pooling) ---
def fetch_all_repos(org: str, token: str) -> list[dict]:
    """Fetch all repos from GitHub org with pagination."""
    session = requests.Session()
    session.headers.update({
        "Authorization": f"token {token}",
        "Accept": "application/vnd.github.v3+json",
    })

    repos = []
    url = f"https://api.github.com/orgs/{org}/repos"
    params = {"per_page": 100, "page": 1}

    while url:
        response = session.get(url, params=params, timeout=10)
        response.raise_for_status()
        repos.extend(response.json())

        # Follow pagination links
        url = response.links.get("next", {}).get("url")
        params = {}  # URL already contains params

    return repos

# --- Retry wrapper ---
def api_request_with_retry(method, url, max_retries=3, **kwargs):
    """Make an API request with retry and exponential backoff."""
    for attempt in range(1, max_retries + 1):
        try:
            response = requests.request(method, url, timeout=10, **kwargs)
            response.raise_for_status()
            return response
        except (requests.ConnectionError, requests.Timeout) as e:
            if attempt == max_retries:
                raise
            delay = 2 ** attempt
            print(f"Attempt {attempt} failed: {e}. Retrying in {delay}s...")
            sleep(delay)
    return None  # Unreachable, but satisfies type checker
```

> **DevOps Relevance:** APIs are how you automate everything in the cloud era — creating DNS records, managing cloud resources, triggering CI/CD pipelines, sending alerts, querying monitoring systems. The `requests` library is the Swiss Army knife. Always use `timeout`, always handle errors, always use sessions for repeated calls.

---

## 11. OS Automation with Python

### 11.1 File System Operations with `pathlib` and `shutil`

```python
#!/usr/bin/env python3
"""File system automation — replacing common bash commands."""
import shutil
from pathlib import Path
from datetime import datetime, timedelta

# --- pathlib replaces os.path ---
def cleanup_old_artifacts(artifacts_dir: str, max_age_days: int = 30) -> int:
    """Delete build artifacts older than max_age_days."""
    artifacts = Path(artifacts_dir)
    cutoff = datetime.now().timestamp() - (max_age_days * 86400)
    deleted = 0

    for item in artifacts.iterdir():
        if item.stat().st_mtime < cutoff:
            if item.is_dir():
                shutil.rmtree(item)
            else:
                item.unlink()
            deleted += 1
            print(f"Deleted: {item.name}")

    return deleted

# --- shutil replaces cp, mv, rm -rf ---
def create_release_backup(release_dir: str) -> Path:
    """Create a backup copy of the current release."""
    src = Path(release_dir)
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    dst = src.parent / f"{src.name}_backup_{timestamp}"

    shutil.copytree(src, dst)          # cp -r
    print(f"Backup created: {dst}")
    return dst

# shutil operations:
# shutil.copy2(src, dst)       # cp (preserves metadata)
# shutil.copytree(src, dst)    # cp -r
# shutil.rmtree(path)          # rm -rf
# shutil.move(src, dst)        # mv
# shutil.disk_usage("/")       # df /  (total, used, free in bytes)

# --- Disk usage check ---
def check_disk_space(path: str, min_gb: float = 5.0) -> bool:
    """Check if there's enough disk space."""
    usage = shutil.disk_usage(path)
    free_gb = usage.free / (1024 ** 3)
    total_gb = usage.total / (1024 ** 3)
    pct_used = (usage.used / usage.total) * 100

    print(f"Disk: {free_gb:.1f}GB free / {total_gb:.1f}GB total ({pct_used:.0f}% used)")

    if free_gb < min_gb:
        print(f"WARNING: Less than {min_gb}GB free!", flush=True)
        return False
    return True
```

### 11.2 Running Commands with `subprocess`

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  subprocess — RUNNING SHELL COMMANDS FROM PYTHON                │
  │                                                                  │
  │  DO use:                                                         │
  │  ├── subprocess.run()          ← Primary API (Python 3.5+)    │
  │  ├── subprocess.run(check=True)  ← Raise on failure           │
  │  ├── capture_output=True       ← Capture stdout/stderr        │
  │  └── text=True                 ← Return strings not bytes     │
  │                                                                  │
  │  DON'T use:                                                      │
  │  ├── os.system()              ← Insecure, no output capture   │
  │  ├── os.popen()               ← Deprecated                     │
  │  ├── subprocess.call()        ← Legacy API                     │
  │  └── shell=True               ← Security risk (injection!)    │
  └──────────────────────────────────────────────────────────────────┘
```

```python
#!/usr/bin/env python3
"""subprocess — the right way to run commands from Python."""
import subprocess
import sys

# --- Basic command (replaces os.system) ---
result = subprocess.run(
    ["kubectl", "get", "pods", "-n", "production", "-o", "json"],
    capture_output=True,
    text=True,
    check=True,  # Raises CalledProcessError if exit code != 0
)
pods_json = result.stdout
print(f"Exit code: {result.returncode}")

# --- Command with error handling ---
def run_command(cmd: list[str], cwd: str = None) -> subprocess.CompletedProcess:
    """Run a command with proper error handling and logging."""
    print(f"Running: {' '.join(cmd)}")
    try:
        result = subprocess.run(
            cmd,
            capture_output=True,
            text=True,
            check=True,
            cwd=cwd,
            timeout=300,  # 5 minute timeout
        )
        if result.stdout.strip():
            print(f"  stdout: {result.stdout.strip()[:200]}")
        return result
    except subprocess.CalledProcessError as e:
        print(f"  FAILED (exit code {e.returncode})", file=sys.stderr)
        print(f"  stderr: {e.stderr.strip()}", file=sys.stderr)
        raise
    except subprocess.TimeoutExpired:
        print(f"  TIMEOUT after 300s", file=sys.stderr)
        raise

# --- Pipeline (replaces bash pipes) ---
def get_top_memory_processes(n: int = 10) -> str:
    """Get top N memory-consuming processes."""
    ps = subprocess.run(
        ["ps", "aux", "--sort=-%mem"],
        capture_output=True, text=True, check=True,
    )
    lines = ps.stdout.strip().split("\n")
    return "\n".join(lines[:n + 1])  # Header + n lines

# --- Stream output in real-time (for long commands) ---
def stream_command(cmd: list[str]) -> int:
    """Run a command and stream output in real-time."""
    process = subprocess.Popen(
        cmd,
        stdout=subprocess.PIPE,
        stderr=subprocess.STDOUT,
        text=True,
        bufsize=1,  # Line buffered
    )
    for line in process.stdout:
        print(f"  | {line}", end="")

    process.wait()
    return process.returncode

# Usage
exit_code = stream_command(["docker", "build", "-t", "myapp:latest", "."])
```

> **⚠️ Gotcha (shell=True):** NEVER use `subprocess.run(f"rm -rf {user_input}", shell=True)` — this is a **shell injection** vulnerability. If `user_input = "; rm -rf /"`, the shell executes both commands. Always pass commands as a list: `subprocess.run(["rm", "-rf", user_input])`.

### 11.3 SSH Automation with `paramiko`

```python
#!/usr/bin/env python3
"""SSH automation — replacing bash ssh loops."""
import paramiko  # pip install paramiko

def ssh_execute(
    host: str,
    command: str,
    user: str = "deploy",
    key_path: str = "~/.ssh/id_rsa",
    timeout: int = 30,
) -> tuple[str, str, int]:
    """Execute a command on a remote host via SSH."""
    client = paramiko.SSHClient()
    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())

    try:
        client.connect(
            hostname=host,
            username=user,
            key_filename=str(Path(key_path).expanduser()),
            timeout=timeout,
        )

        stdin, stdout, stderr = client.exec_command(command, timeout=timeout)
        exit_code = stdout.channel.recv_exit_status()
        out = stdout.read().decode().strip()
        err = stderr.read().decode().strip()

        return out, err, exit_code
    finally:
        client.close()

# --- Deploy to multiple servers in parallel ---
from concurrent.futures import ThreadPoolExecutor, as_completed

def parallel_deploy(servers: list[str], version: str) -> dict:
    """Deploy to multiple servers concurrently."""
    results = {}

    def deploy_to_server(server: str) -> tuple[str, bool]:
        cmd = f"/opt/deploy/update.sh {version}"
        out, err, code = ssh_execute(server, cmd)
        success = code == 0
        if not success:
            print(f"  [{server}] FAILED: {err}")
        else:
            print(f"  [{server}] SUCCESS")
        return server, success

    with ThreadPoolExecutor(max_workers=5) as executor:
        futures = {
            executor.submit(deploy_to_server, s): s
            for s in servers
        }
        for future in as_completed(futures):
            server, success = future.result()
            results[server] = success

    return results

# Usage
servers = ["web01", "web02", "web03", "web04"]
results = parallel_deploy(servers, "v2.3.1")
failed = [s for s, ok in results.items() if not ok]
if failed:
    print(f"FAILED servers: {', '.join(failed)}")
    sys.exit(1)
```

### 11.4 Environment Variables

```python
#!/usr/bin/env python3
"""Environment variable handling for 12-factor apps."""
import os
import sys

# --- Read environment variables ---
db_host = os.environ.get("DB_HOST", "localhost")       # With default
db_port = int(os.environ.get("DB_PORT", "5432"))       # Cast to int
debug = os.environ.get("DEBUG", "false").lower() == "true"  # Bool

# Required env var (fail fast if missing)
def require_env(name: str) -> str:
    """Get required environment variable or exit."""
    value = os.environ.get(name)
    if not value:
        print(f"ERROR: Required environment variable {name} is not set", file=sys.stderr)
        sys.exit(1)
    return value

api_token = require_env("API_TOKEN")
cluster = require_env("CLUSTER_NAME")

# --- Set environment variables (for child processes) ---
os.environ["KUBECONFIG"] = "/etc/kubernetes/admin.conf"
os.environ["AWS_REGION"] = "us-east-1"

# --- Load .env files (python-dotenv) ---
# pip install python-dotenv
from dotenv import load_dotenv

load_dotenv()                          # Load from .env in current dir
load_dotenv("/etc/myapp/.env")         # Load from specific path
# Now os.environ contains values from .env file
```

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  12-FACTOR APP: CONFIGURATION VIA ENVIRONMENT                   │
  │                                                                  │
  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
  │  │ Development │  │ Staging     │  │ Production  │            │
  │  │             │  │             │  │             │            │
  │  │ DB_HOST=    │  │ DB_HOST=    │  │ DB_HOST=    │            │
  │  │ localhost   │  │ staging-    │  │ prod-db.    │            │
  │  │             │  │ db.internal │  │ internal    │            │
  │  │ DEBUG=true  │  │ DEBUG=false │  │ DEBUG=false │            │
  │  │ LOG=debug   │  │ LOG=info    │  │ LOG=warn    │            │
  │  └─────────────┘  └─────────────┘  └─────────────┘            │
  │                                                                  │
  │  Same code, different config via env vars                       │
  │  NEVER hardcode secrets, URLs, or config in source code        │
  └──────────────────────────────────────────────────────────────────┘
```

> **DevOps Relevance:** OS automation with Python replaces complex Bash scripts with readable, testable code. `subprocess` is the bridge between Python and CLI tools (kubectl, docker, terraform). `paramiko` replaces fragile SSH-in-a-loop Bash patterns with proper connection management and parallel execution. Environment variables are the 12-factor way to configure applications across environments.

---

## 12. Error Handling & Logging

### 12.1 Exception Handling — The Python Way

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  PYTHON EXCEPTION HIERARCHY (DevOps-relevant subset)            │
  │                                                                  │
  │  BaseException                                                   │
  │  └── Exception                                                   │
  │      ├── ValueError         Invalid value / argument            │
  │      ├── TypeError          Wrong type                           │
  │      ├── KeyError           Missing dict key                    │
  │      ├── FileNotFoundError  File doesn't exist                  │
  │      ├── PermissionError    No access rights                    │
  │      ├── TimeoutError       Operation timed out                 │
  │      ├── ConnectionError    Network connection failed           │
  │      ├── OSError            OS-level error (disk full, etc.)   │
  │      │   ├── FileExistsError                                    │
  │      │   └── IsADirectoryError                                  │
  │      ├── subprocess.CalledProcessError  Command failed          │
  │      ├── requests.HTTPError      HTTP 4xx/5xx                   │
  │      ├── json.JSONDecodeError    Invalid JSON                   │
  │      └── yaml.YAMLError          Invalid YAML                   │
  │                                                                  │
  │  RULE: Catch specific exceptions, not bare `except:`            │
  └──────────────────────────────────────────────────────────────────┘
```

```python
#!/usr/bin/env python3
"""Error handling patterns for DevOps scripts."""
import json
import subprocess
import sys

# --- Pattern 1: Specific exception handling ---
def read_config(path: str) -> dict:
    """Read and parse a JSON config file with proper error handling."""
    try:
        with open(path) as f:
            return json.load(f)
    except FileNotFoundError:
        print(f"Config file not found: {path}", file=sys.stderr)
        sys.exit(1)
    except PermissionError:
        print(f"No permission to read: {path}", file=sys.stderr)
        sys.exit(1)
    except json.JSONDecodeError as e:
        print(f"Invalid JSON in {path}: {e}", file=sys.stderr)
        sys.exit(1)

# --- Pattern 2: Retry with specific exceptions ---
def deploy_with_retry(version: str, max_attempts: int = 3) -> bool:
    """Deploy with retry on transient failures."""
    for attempt in range(1, max_attempts + 1):
        try:
            run_deployment(version)
            return True
        except ConnectionError:
            # Transient — retry
            print(f"Connection failed (attempt {attempt}/{max_attempts})")
            if attempt < max_attempts:
                sleep(2 ** attempt)
        except ValueError as e:
            # Permanent — don't retry
            print(f"Invalid configuration: {e}")
            return False
    return False

# --- Pattern 3: Context manager for cleanup ---
from contextlib import contextmanager

@contextmanager
def deployment_lock(lock_path: str):
    """Acquire a deployment lock, ensuring cleanup."""
    lock = Path(lock_path)
    if lock.exists():
        raise RuntimeError(f"Deployment already in progress (lock: {lock_path})")

    lock.write_text(str(os.getpid()))
    try:
        yield
    finally:
        lock.unlink(missing_ok=True)

# Usage
with deployment_lock("/var/run/deploy.lock"):
    run_deployment("v2.3.1")
# Lock is automatically removed, even on error
```

### 12.2 Custom Exceptions

```python
#!/usr/bin/env python3
"""Custom exceptions for clear error communication."""

class DeploymentError(Exception):
    """Base exception for deployment failures."""
    pass

class PreFlightError(DeploymentError):
    """Pre-flight checks failed."""
    pass

class RollbackError(DeploymentError):
    """Rollback failed — manual intervention needed."""
    pass

class HealthCheckError(DeploymentError):
    """Service health check failed after deployment."""
    pass

# Usage
def deploy(version: str, env: str):
    try:
        pre_flight_checks(env)
    except Exception as e:
        raise PreFlightError(f"Pre-flight failed for {env}: {e}") from e

    try:
        apply_deployment(version)
        verify_health(env)
    except HealthCheckError:
        print("Health check failed, rolling back...")
        try:
            rollback(env)
        except Exception as e:
            raise RollbackError(f"CRITICAL: Rollback failed: {e}") from e
        raise

# Caller
try:
    deploy("v2.3.1", "production")
except PreFlightError as e:
    print(f"Cannot deploy: {e}")
    sys.exit(1)
except RollbackError as e:
    print(f"🚨 CRITICAL: {e} — manual intervention required!")
    send_pagerduty_alert(str(e))
    sys.exit(2)
except HealthCheckError:
    print("Deployment rolled back successfully")
    sys.exit(1)
```

### 12.3 Python `logging` Module

```python
#!/usr/bin/env python3
"""Production-grade logging setup."""
import logging
import logging.handlers
from pathlib import Path

def setup_logging(
    name: str = "deploy",
    level: str = "INFO",
    log_file: str = None,
) -> logging.Logger:
    """Configure structured logging for a DevOps script."""
    logger = logging.getLogger(name)
    logger.setLevel(getattr(logging, level.upper()))

    # Console handler (stderr — keeps stdout clean for data)
    console = logging.StreamHandler()
    console.setFormatter(logging.Formatter(
        "%(asctime)s [%(levelname)-5s] %(message)s",
        datefmt="%Y-%m-%d %H:%M:%S",
    ))
    logger.addHandler(console)

    # File handler with rotation (optional)
    if log_file:
        Path(log_file).parent.mkdir(parents=True, exist_ok=True)
        file_handler = logging.handlers.RotatingFileHandler(
            log_file,
            maxBytes=10 * 1024 * 1024,  # 10 MB
            backupCount=5,
        )
        file_handler.setFormatter(logging.Formatter(
            "%(asctime)s [%(levelname)-5s] [%(name)s] %(message)s"
        ))
        logger.addHandler(file_handler)

    return logger

# Usage
log = setup_logging("deploy", level="INFO", log_file="/var/log/deploy.log")

log.info("Starting deployment of v2.3.1 to production")
log.warning("Disk space below 20%%")
log.error("Health check failed on web02")
log.debug("Response body: %s", response_body)  # Only printed if DEBUG

# Structured context
log.info("Deploy complete", extra={
    "version": "v2.3.1",
    "environment": "production",
    "duration_seconds": 45,
})
```

### 12.4 Structured Logging with `structlog`

```python
#!/usr/bin/env python3
"""Structured logging — JSON logs for log aggregation systems."""
import structlog  # pip install structlog

# Configure structlog for JSON output (production)
structlog.configure(
    processors=[
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.add_log_level,
        structlog.processors.JSONRenderer(),
    ],
)

log = structlog.get_logger()

# Structured log entries — machine-parseable
log.info("deployment_started", version="v2.3.1", env="production")
log.warning("disk_space_low", mount="/var", free_gb=2.3, threshold_gb=5.0)
log.error("health_check_failed", server="web02", http_code=503, attempt=3)

# Output (JSON — perfect for ELK/Loki/Datadog):
# {"event":"deployment_started","version":"v2.3.1","env":"production",
#  "timestamp":"2024-01-15T10:30:00Z","level":"info"}
```

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  LOGGING LEVELS — WHEN TO USE WHAT                              │
  │                                                                  │
  │  Level       When to Use                                         │
  │  ──────────────────────────────────────────────                  │
  │  DEBUG       Variable values, detailed flow (dev only)          │
  │  INFO        Normal operations (started, completed, counts)     │
  │  WARNING     Something unexpected but not broken (low disk)     │
  │  ERROR       Something failed (API error, timeout)              │
  │  CRITICAL    System is unusable (can't connect to DB)           │
  │                                                                  │
  │  Production default: INFO                                        │
  │  Debugging: DEBUG                                                │
  │  Noisy pipelines: WARNING                                        │
  │                                                                  │
  │  RULE: Log what happened, not what the code does               │
  │  BAD:  log.info("Entering function deploy()")                  │
  │  GOOD: log.info("Deploying v2.3.1 to production (3 replicas)")│
  └──────────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha (bare `except:`):** Never write `except:` or `except Exception:` without re-raising. This swallows ALL errors including keyboard interrupts and memory errors, making debugging impossible. Always catch **specific** exceptions.

> **DevOps Relevance:** Python's exception hierarchy maps cleanly to operational scenarios: `ConnectionError` = retry, `PermissionError` = check IAM, `TimeoutError` = increase limits. Custom exceptions make deployment scripts self-documenting. Structured logging (JSON) is essential for log aggregation tools like ELK, Loki, and Datadog — they can't parse unstructured text efficiently.

---

## 13. Infrastructure Scripting

### 13.1 AWS Automation with Boto3

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  CLOUD SDK LANDSCAPE                                             │
  │                                                                  │
  │  Cloud Provider    Python SDK        Auth Method                 │
  │  ──────────────────────────────────────────────                  │
  │  AWS               boto3             IAM roles, env vars,       │
  │                                      ~/.aws/credentials         │
  │  Azure             azure-*           Service principals,        │
  │                                      Managed Identity           │
  │  GCP               google-cloud-*    Service accounts,          │
  │                                      workload identity          │
  │                                                                  │
  │  AWS is the most common, so we'll focus on Boto3               │
  │  The patterns are the same across all cloud SDKs               │
  └──────────────────────────────────────────────────────────────────┘
```

```python
#!/usr/bin/env python3
"""AWS automation with boto3 — the most common DevOps Python task."""
import boto3  # pip install boto3
from datetime import datetime, timedelta

# --- EC2 Instance Management ---
def list_instances(region: str = "us-east-1", env: str = None) -> list[dict]:
    """List EC2 instances, optionally filtered by environment tag."""
    ec2 = boto3.client("ec2", region_name=region)

    filters = []
    if env:
        filters.append({"Name": "tag:Environment", "Values": [env]})

    response = ec2.describe_instances(Filters=filters)
    instances = []

    for reservation in response["Reservations"]:
        for instance in reservation["Instances"]:
            tags = {t["Key"]: t["Value"] for t in instance.get("Tags", [])}
            instances.append({
                "id": instance["InstanceId"],
                "type": instance["InstanceType"],
                "state": instance["State"]["Name"],
                "name": tags.get("Name", "unnamed"),
                "env": tags.get("Environment", "unknown"),
                "private_ip": instance.get("PrivateIpAddress", "N/A"),
            })

    return instances

# --- S3 Operations ---
def upload_artifact(bucket: str, local_path: str, s3_key: str) -> str:
    """Upload a build artifact to S3."""
    s3 = boto3.client("s3")
    s3.upload_file(
        local_path, bucket, s3_key,
        ExtraArgs={"ServerSideEncryption": "AES256"},
    )
    return f"s3://{bucket}/{s3_key}"

def cleanup_old_artifacts(bucket: str, prefix: str, max_age_days: int = 30):
    """Delete S3 objects older than max_age_days."""
    s3 = boto3.resource("s3")
    bucket_obj = s3.Bucket(bucket)
    cutoff = datetime.now(tz=None) - timedelta(days=max_age_days)
    deleted = 0

    for obj in bucket_obj.objects.filter(Prefix=prefix):
        if obj.last_modified.replace(tzinfo=None) < cutoff:
            obj.delete()
            deleted += 1

    print(f"Deleted {deleted} objects older than {max_age_days} days")

# --- IAM — Audit unused access keys ---
def find_stale_access_keys(max_age_days: int = 90) -> list[dict]:
    """Find IAM access keys not used in max_age_days."""
    iam = boto3.client("iam")
    stale_keys = []
    cutoff = datetime.now(tz=None) - timedelta(days=max_age_days)

    for user in iam.list_users()["Users"]:
        keys = iam.list_access_keys(UserName=user["UserName"])
        for key in keys["AccessKeyMetadata"]:
            if key["Status"] == "Active":
                last_used = iam.get_access_key_last_used(
                    AccessKeyId=key["AccessKeyId"]
                )
                last_date = last_used["AccessKeyLastUsed"].get("LastUsedDate")
                if not last_date or last_date.replace(tzinfo=None) < cutoff:
                    stale_keys.append({
                        "user": user["UserName"],
                        "key_id": key["AccessKeyId"],
                        "last_used": str(last_date or "Never"),
                    })

    return stale_keys
```

### 13.2 Terraform Wrapper Scripts

```python
#!/usr/bin/env python3
"""Terraform wrapper — standardize plan/apply across environments."""
import subprocess
import sys
import json
from pathlib import Path

class TerraformRunner:
    """Wrapper for Terraform CLI operations."""

    def __init__(self, working_dir: str, backend_config: dict = None):
        self.working_dir = working_dir
        self.backend_config = backend_config or {}

    def _run(self, args: list[str], capture: bool = False) -> subprocess.CompletedProcess:
        cmd = ["terraform"] + args
        print(f"  → terraform {' '.join(args)}")
        return subprocess.run(
            cmd,
            cwd=self.working_dir,
            capture_output=capture,
            text=True,
            check=True,
        )

    def init(self):
        """Initialize Terraform with backend config."""
        args = ["init", "-input=false"]
        for key, value in self.backend_config.items():
            args.append(f"-backend-config={key}={value}")
        self._run(args)

    def plan(self, var_file: str = None, out: str = "tfplan") -> str:
        """Create Terraform plan."""
        args = ["plan", "-input=false", f"-out={out}"]
        if var_file:
            args.append(f"-var-file={var_file}")
        self._run(args)
        return out

    def apply(self, plan_file: str = "tfplan"):
        """Apply a previously created plan."""
        self._run(["apply", "-input=false", plan_file])

    def output(self) -> dict:
        """Get Terraform outputs as dict."""
        result = self._run(["output", "-json"], capture=True)
        raw = json.loads(result.stdout)
        return {k: v["value"] for k, v in raw.items()}

    def destroy(self, var_file: str = None, auto_approve: bool = False):
        """Destroy infrastructure."""
        args = ["destroy", "-input=false"]
        if var_file:
            args.append(f"-var-file={var_file}")
        if auto_approve:
            args.append("-auto-approve")
        self._run(args)

# Usage
def deploy_infrastructure(env: str):
    tf = TerraformRunner(
        working_dir=f"./infra/environments/{env}",
        backend_config={
            "bucket": f"myorg-terraform-{env}",
            "key": "infra/terraform.tfstate",
            "region": "us-east-1",
        },
    )

    tf.init()
    tf.plan(var_file=f"vars/{env}.tfvars")

    if env == "production":
        confirm = input("Apply to PRODUCTION? (yes/no): ")
        if confirm != "yes":
            print("Aborted.")
            sys.exit(0)

    tf.apply()
    outputs = tf.output()
    print(f"Load Balancer DNS: {outputs.get('lb_dns_name')}")
```

> **⚠️ Gotcha (Terraform state):** Never run Terraform commands in parallel against the same state. Use DynamoDB state locking for AWS, or equivalent for other backends. Your wrapper script should check for lock status before proceeding.

> **DevOps Relevance:** Infrastructure scripting with cloud SDKs is the foundation of DevOps automation. Boto3 scripts handle everything from cleaning up old resources to auditing security. Terraform wrappers standardize the plan/apply workflow across teams and enforce guardrails like requiring approval for production changes.

---

## 14. CI/CD Pipeline Scripting

### 14.1 GitHub Actions Helper Scripts

```python
#!/usr/bin/env python3
"""Helper scripts for GitHub Actions workflows."""
import json
import os
import sys
import requests

# --- Set GitHub Actions output ---
def set_output(name: str, value: str):
    """Set a GitHub Actions output variable."""
    output_file = os.environ.get("GITHUB_OUTPUT", "")
    if output_file:
        with open(output_file, "a") as f:
            f.write(f"{name}={value}\n")
    else:
        # Fallback for local testing
        print(f"::set-output name={name}::{value}")

# --- Determine what changed ---
def get_changed_files() -> list[str]:
    """Get list of files changed in this PR/push."""
    event_path = os.environ.get("GITHUB_EVENT_PATH", "")
    if not event_path:
        return []

    with open(event_path) as f:
        event = json.load(f)

    # For pull requests
    if "pull_request" in event:
        base = event["pull_request"]["base"]["sha"]
        head = event["pull_request"]["head"]["sha"]
    else:
        # For push events
        base = event.get("before", "HEAD~1")
        head = event.get("after", "HEAD")

    import subprocess
    result = subprocess.run(
        ["git", "diff", "--name-only", base, head],
        capture_output=True, text=True, check=True,
    )
    return result.stdout.strip().split("\n")

def determine_affected_services(changed_files: list[str]) -> list[str]:
    """Map changed files to affected services for targeted deploys."""
    service_map = {
        "services/api/": "api",
        "services/web/": "web",
        "services/worker/": "worker",
        "libs/common/": "all",
        "infrastructure/": "infra",
    }

    affected = set()
    for file in changed_files:
        for prefix, service in service_map.items():
            if file.startswith(prefix):
                if service == "all":
                    return list(service_map.values())
                affected.add(service)

    return list(affected)

# Main — run in GitHub Actions
if __name__ == "__main__":
    changed = get_changed_files()
    services = determine_affected_services(changed)

    set_output("affected_services", json.dumps(services))
    set_output("has_infra_changes", str("infra" in services).lower())

    print(f"Changed files: {len(changed)}")
    print(f"Affected services: {services}")
```

### 14.2 Build & Release Automation

```bash
#!/usr/bin/env bash
# ci_build.sh — Build, tag, and push Docker images in CI
set -euo pipefail

# --- Determine version ---
GIT_SHA=$(git rev-parse --short HEAD)
GIT_BRANCH=$(git rev-parse --abbrev-ref HEAD)
TIMESTAMP=$(date +%Y%m%d%H%M%S)

case "$GIT_BRANCH" in
    main)
        # Production: use git tag if available
        VERSION=$(git describe --tags --exact-match 2>/dev/null || echo "main-${GIT_SHA}")
        ;;
    develop)
        VERSION="dev-${GIT_SHA}-${TIMESTAMP}"
        ;;
    *)
        # Feature branch
        SAFE_BRANCH=$(echo "$GIT_BRANCH" | sed 's/[^a-zA-Z0-9]/-/g' | cut -c1-50)
        VERSION="branch-${SAFE_BRANCH}-${GIT_SHA}"
        ;;
esac

REGISTRY="${REGISTRY:-ghcr.io/myorg}"
IMAGE="${REGISTRY}/myapp:${VERSION}"

echo "Building: $IMAGE"

# --- Build ---
docker build \
    --build-arg VERSION="$VERSION" \
    --build-arg BUILD_DATE="$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
    --build-arg GIT_SHA="$GIT_SHA" \
    --label "org.opencontainers.image.revision=${GIT_SHA}" \
    --tag "$IMAGE" \
    --tag "${REGISTRY}/myapp:latest" \
    .

# --- Push ---
docker push "$IMAGE"
if [[ "$GIT_BRANCH" == "main" ]]; then
    docker push "${REGISTRY}/myapp:latest"
fi

echo "✅ Published: $IMAGE"
```

### 14.3 Deployment Notification Script

```python
#!/usr/bin/env python3
"""Send deployment notifications to Slack and update status checks."""
import os
import sys
import requests
from datetime import datetime

def send_slack_notification(
    webhook_url: str,
    status: str,
    service: str,
    version: str,
    env: str,
    duration: int = None,
    error: str = None,
):
    """Send rich Slack notification for deployment events."""
    emoji = "✅" if status == "success" else "❌" if status == "failure" else "🔄"
    color = "#36a64f" if status == "success" else "#ff0000" if status == "failure" else "#ffaa00"

    fields = [
        {"title": "Service", "value": service, "short": True},
        {"title": "Version", "value": version, "short": True},
        {"title": "Environment", "value": env, "short": True},
        {"title": "Status", "value": f"{emoji} {status.upper()}", "short": True},
    ]

    if duration:
        fields.append({"title": "Duration", "value": f"{duration}s", "short": True})
    if error:
        fields.append({"title": "Error", "value": f"```{error[:500]}```", "short": False})

    payload = {
        "attachments": [{
            "color": color,
            "title": f"Deployment: {service} → {env}",
            "fields": fields,
            "footer": f"Deploy Bot | {datetime.utcnow().strftime('%Y-%m-%d %H:%M UTC')}",
        }]
    }

    response = requests.post(webhook_url, json=payload, timeout=10)
    response.raise_for_status()

# Usage in CI:
webhook = os.environ.get("SLACK_WEBHOOK_URL", "")
if webhook:
    send_slack_notification(
        webhook_url=webhook,
        status="success",
        service="api",
        version=os.environ.get("VERSION", "unknown"),
        env=os.environ.get("DEPLOY_ENV", "staging"),
        duration=45,
    )
```

> **DevOps Relevance:** CI/CD scripting bridges the gap between your pipeline YAML and business logic. Change-detection scripts enable monorepo workflows (only deploy what changed). Versioning strategies (semver for main, SHA for branches) ensure traceability. Notification scripts close the feedback loop — everyone should know when a deployment happens.

---

## 15. Container & Kubernetes Scripting

### 15.1 Docker Automation

```python
#!/usr/bin/env python3
"""Docker automation — image management and container operations."""
import subprocess
import json
from datetime import datetime, timedelta

def docker_image_cleanup(max_age_days: int = 7, keep_latest: int = 5):
    """Remove old Docker images to free disk space."""
    result = subprocess.run(
        ["docker", "images", "--format", "{{json .}}"],
        capture_output=True, text=True, check=True,
    )

    images = [json.loads(line) for line in result.stdout.strip().split("\n") if line]

    # Group by repository
    repos: dict[str, list] = {}
    for img in images:
        repo = img.get("Repository", "<none>")
        repos.setdefault(repo, []).append(img)

    removed = 0
    for repo, imgs in repos.items():
        if repo == "<none>":
            # Dangling images — always remove
            for img in imgs:
                subprocess.run(["docker", "rmi", img["ID"]], capture_output=True)
                removed += 1
            continue

        # Sort by creation date, keep latest N
        sorted_imgs = sorted(imgs, key=lambda x: x.get("CreatedAt", ""), reverse=True)
        for img in sorted_imgs[keep_latest:]:
            tag = f"{img['Repository']}:{img['Tag']}"
            print(f"Removing: {tag}")
            subprocess.run(["docker", "rmi", tag], capture_output=True)
            removed += 1

    print(f"Removed {removed} images")

def docker_compose_health_check(compose_file: str = "docker-compose.yml") -> bool:
    """Check health of all services in a Docker Compose stack."""
    result = subprocess.run(
        ["docker", "compose", "-f", compose_file, "ps", "--format", "json"],
        capture_output=True, text=True, check=True,
    )

    services = json.loads(result.stdout)
    all_healthy = True

    for svc in services:
        name = svc.get("Name", "unknown")
        state = svc.get("State", "unknown")
        health = svc.get("Health", "N/A")

        status = "✅" if state == "running" and health in ("healthy", "N/A") else "❌"
        print(f"  {status} {name}: state={state}, health={health}")

        if status == "❌":
            all_healthy = False

    return all_healthy
```

### 15.2 Kubernetes Automation with `client-python`

```python
#!/usr/bin/env python3
"""Kubernetes automation with the official Python client."""
from kubernetes import client, config  # pip install kubernetes
from kubernetes.client.rest import ApiException

# Load kubeconfig (from ~/.kube/config or in-cluster)
try:
    config.load_incluster_config()  # Running inside K8s
except config.ConfigException:
    config.load_kube_config()       # Running locally

v1 = client.CoreV1Api()
apps_v1 = client.AppsV1Api()

# --- List pods with filtering ---
def get_pods(namespace: str, label_selector: str = None) -> list[dict]:
    """Get pods in a namespace with optional label filter."""
    pods = v1.list_namespaced_pod(
        namespace=namespace,
        label_selector=label_selector,
    )

    return [{
        "name": pod.metadata.name,
        "phase": pod.status.phase,
        "node": pod.spec.node_name,
        "restarts": sum(
            cs.restart_count for cs in (pod.status.container_statuses or [])
        ),
        "ready": all(
            cs.ready for cs in (pod.status.container_statuses or [])
        ),
        "age": (datetime.now(tz=pod.metadata.creation_timestamp.tzinfo)
                - pod.metadata.creation_timestamp).days,
    } for pod in pods.items]

# --- Scale deployment ---
def scale_deployment(name: str, namespace: str, replicas: int):
    """Scale a deployment to the specified number of replicas."""
    body = {"spec": {"replicas": replicas}}
    apps_v1.patch_namespaced_deployment_scale(
        name=name,
        namespace=namespace,
        body=body,
    )
    print(f"Scaled {namespace}/{name} to {replicas} replicas")

# --- Restart deployment (rolling restart) ---
def restart_deployment(name: str, namespace: str):
    """Trigger a rolling restart by patching the template annotation."""
    now = datetime.utcnow().isoformat()
    body = {
        "spec": {
            "template": {
                "metadata": {
                    "annotations": {
                        "kubectl.kubernetes.io/restartedAt": now
                    }
                }
            }
        }
    }
    apps_v1.patch_namespaced_deployment(name, namespace, body)
    print(f"Rolling restart triggered for {namespace}/{name}")

# --- Find high-restart pods (potential crash loops) ---
def find_crashloop_pods(namespace: str = None, threshold: int = 5) -> list[dict]:
    """Find pods with restart count above threshold."""
    if namespace:
        pods = v1.list_namespaced_pod(namespace)
    else:
        pods = v1.list_pod_for_all_namespaces()

    problematic = []
    for pod in pods.items:
        for cs in (pod.status.container_statuses or []):
            if cs.restart_count >= threshold:
                problematic.append({
                    "namespace": pod.metadata.namespace,
                    "pod": pod.metadata.name,
                    "container": cs.name,
                    "restarts": cs.restart_count,
                    "state": str(cs.state),
                })

    return problematic
```

### 15.3 kubectl Wrapper Scripts

```bash
#!/usr/bin/env bash
# k8s_ops.sh — Common Kubernetes operations as reusable functions
set -euo pipefail

# --- Wait for deployment rollout ---
wait_for_rollout() {
    local name="$1"
    local namespace="${2:-default}"
    local timeout="${3:-300}"

    echo "Waiting for deployment/$name in $namespace..."
    if ! kubectl rollout status "deployment/$name" \
        -n "$namespace" \
        --timeout="${timeout}s"; then
        echo "ERROR: Rollout failed for $name" >&2
        kubectl get pods -n "$namespace" -l "app=$name" \
            --sort-by='.status.startTime' | tail -5
        return 1
    fi
    echo "✅ Rollout complete"
}

# --- Get resource usage ---
k8s_resource_report() {
    local namespace="${1:-default}"

    echo "=== Resource Report for namespace: $namespace ==="

    echo -e "\n--- Pod Resource Usage ---"
    kubectl top pods -n "$namespace" --sort-by=memory 2>/dev/null || \
        echo "  (metrics-server not available)"

    echo -e "\n--- Pod Status ---"
    kubectl get pods -n "$namespace" \
        -o custom-columns=\
'NAME:.metadata.name,STATUS:.status.phase,RESTARTS:.status.containerStatuses[0].restartCount,AGE:.metadata.creationTimestamp,NODE:.spec.nodeName' \
        --sort-by='.status.containerStatuses[0].restartCount'

    echo -e "\n--- PVC Usage ---"
    kubectl get pvc -n "$namespace" \
        -o custom-columns='NAME:.metadata.name,STATUS:.status.phase,SIZE:.spec.resources.requests.storage,STORAGECLASS:.spec.storageClassName'
}

# --- Quick debug pod ---
debug_pod() {
    local namespace="${1:-default}"
    local image="${2:-nicolaka/netshoot}"

    kubectl run debug-"$$" \
        --namespace="$namespace" \
        --image="$image" \
        --rm -it \
        --restart=Never \
        -- /bin/bash
}
```

> **⚠️ Gotcha (K8s client auth):** Python `kubernetes` library reads `~/.kube/config` for local dev but needs `load_incluster_config()` when running inside a pod. Always try in-cluster first with a fallback. For CI/CD, set `KUBECONFIG` environment variable.

> **DevOps Relevance:** Container and K8s scripting automates the parts of operations that kubectl can't do alone — finding crash-looping pods across all namespaces, automated scaling based on custom metrics, cleanup of old images and completed jobs. The Python K8s client gives you full API access for building custom operators and controllers.

---

## 16. Monitoring & Alerting Scripts

### 16.1 Custom Prometheus Exporter

```python
#!/usr/bin/env python3
"""Custom Prometheus exporter — expose app metrics for scraping."""
from prometheus_client import (     # pip install prometheus-client
    start_http_server,
    Gauge, Counter, Histogram, Info,
)
import time
import subprocess
import shutil

# --- Define metrics ---
DISK_USAGE = Gauge(
    "node_custom_disk_usage_percent",
    "Disk usage percentage",
    ["mountpoint"],
)
BACKUP_AGE = Gauge(
    "backup_age_seconds",
    "Age of the latest backup in seconds",
    ["database"],
)
DEPLOY_COUNT = Counter(
    "deployments_total",
    "Total number of deployments",
    ["environment", "status"],
)
REQUEST_DURATION = Histogram(
    "http_request_duration_seconds",
    "HTTP request duration",
    ["method", "endpoint"],
    buckets=[0.01, 0.05, 0.1, 0.5, 1.0, 5.0],
)
APP_INFO = Info("app", "Application information")
APP_INFO.info({"version": "2.3.1", "environment": "production"})

def collect_disk_metrics():
    """Collect disk usage metrics."""
    for mount in ["/", "/var", "/var/log", "/backups"]:
        try:
            usage = shutil.disk_usage(mount)
            pct = (usage.used / usage.total) * 100
            DISK_USAGE.labels(mountpoint=mount).set(round(pct, 1))
        except OSError:
            pass

def collect_backup_metrics():
    """Check age of latest backup."""
    from pathlib import Path
    backup_dir = Path("/backups/postgres")
    if backup_dir.exists():
        latest = max(backup_dir.glob("*.sql.gz"), key=lambda p: p.stat().st_mtime, default=None)
        if latest:
            age = time.time() - latest.stat().st_mtime
            BACKUP_AGE.labels(database="myapp").set(age)

if __name__ == "__main__":
    start_http_server(9100)  # Expose metrics on :9100/metrics
    print("Exporter running on :9100")

    while True:
        collect_disk_metrics()
        collect_backup_metrics()
        time.sleep(30)
```

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  PROMETHEUS METRIC TYPES                                         │
  │                                                                  │
  │  Counter     Only goes UP (total requests, errors, deploys)     │
  │              DEPLOY_COUNT.labels(env="prod", status="ok").inc() │
  │                                                                  │
  │  Gauge       Can go UP and DOWN (temperature, disk %, conns)    │
  │              DISK_USAGE.labels(mount="/").set(85.3)             │
  │                                                                  │
  │  Histogram   Distribution of values (latency, request size)    │
  │              REQUEST_DURATION.labels(method="GET").observe(0.3) │
  │                                                                  │
  │  Summary     Like Histogram but calculates quantiles client-    │
  │              side (less common)                                  │
  │                                                                  │
  │  Info        Static key-value metadata (version, env)           │
  │              APP_INFO.info({"version": "2.3.1"})               │
  │                                                                  │
  │  NAMING: namespace_subsystem_name_unit                          │
  │  Examples: http_requests_total, node_disk_usage_percent        │
  └──────────────────────────────────────────────────────────────────┘
```

### 16.2 Log Parsing & Alerting

```python
#!/usr/bin/env python3
"""Parse logs and alert on anomalies."""
import re
from collections import Counter
from pathlib import Path
from datetime import datetime

def parse_nginx_log(log_path: str, last_n_minutes: int = 5) -> dict:
    """Parse Nginx access log and generate a summary."""
    status_counts = Counter()
    endpoint_counts = Counter()
    error_ips = Counter()
    slow_requests = []

    log_pattern = re.compile(
        r'(?P<ip>[\d.]+) .+ \[(?P<time>[^\]]+)\] '
        r'"(?P<method>\w+) (?P<path>[^ ]+) [^"]*" '
        r'(?P<status>\d+) (?P<size>\d+) "[^"]*" "[^"]*" '
        r'(?P<duration>[\d.]+)'
    )

    with open(log_path) as f:
        for line in f:
            match = log_pattern.match(line)
            if not match:
                continue

            data = match.groupdict()
            status = int(data["status"])
            duration = float(data["duration"])

            status_counts[status] += 1
            endpoint_counts[data["path"]] += 1

            if status >= 500:
                error_ips[data["ip"]] += 1

            if duration > 2.0:
                slow_requests.append({
                    "path": data["path"],
                    "duration": duration,
                    "status": status,
                })

    return {
        "total_requests": sum(status_counts.values()),
        "status_breakdown": dict(status_counts.most_common()),
        "top_endpoints": dict(endpoint_counts.most_common(10)),
        "error_ips": dict(error_ips.most_common(10)),
        "slow_requests": sorted(slow_requests, key=lambda x: x["duration"], reverse=True)[:10],
        "error_rate": status_counts.get(500, 0) / max(sum(status_counts.values()), 1) * 100,
    }

# Alert on anomalies
report = parse_nginx_log("/var/log/nginx/access.log")
if report["error_rate"] > 5.0:
    send_alert(f"🚨 Error rate: {report['error_rate']:.1f}% (threshold: 5%)")
if len(report["slow_requests"]) > 20:
    send_alert(f"🐢 {len(report['slow_requests'])} slow requests (>2s)")
```

### 16.3 Webhook Integrations

```python
#!/usr/bin/env python3
"""Alerting integrations — Slack, PagerDuty, MS Teams."""
import requests
import json
from datetime import datetime

class AlertManager:
    """Unified alerting across multiple channels."""

    def __init__(self, slack_webhook: str = None, pagerduty_key: str = None):
        self.slack_webhook = slack_webhook
        self.pagerduty_key = pagerduty_key

    def send_slack(self, title: str, message: str, severity: str = "warning"):
        """Send alert to Slack."""
        if not self.slack_webhook:
            return

        color_map = {"info": "#36a64f", "warning": "#ffaa00", "critical": "#ff0000"}
        emoji_map = {"info": "ℹ️", "warning": "⚠️", "critical": "🚨"}

        payload = {
            "attachments": [{
                "color": color_map.get(severity, "#808080"),
                "title": f"{emoji_map.get(severity, '')} {title}",
                "text": message,
                "footer": f"Alert Bot | {datetime.utcnow().strftime('%H:%M UTC')}",
            }]
        }
        requests.post(self.slack_webhook, json=payload, timeout=10)

    def send_pagerduty(self, title: str, message: str, severity: str = "warning"):
        """Create PagerDuty incident."""
        if not self.pagerduty_key:
            return

        payload = {
            "routing_key": self.pagerduty_key,
            "event_action": "trigger",
            "payload": {
                "summary": title,
                "severity": severity,
                "source": "monitoring-script",
                "custom_details": {"message": message},
            },
        }
        requests.post(
            "https://events.pagerduty.com/v2/enqueue",
            json=payload, timeout=10,
        )

    def alert(self, title: str, message: str, severity: str = "warning"):
        """Send alert to all configured channels."""
        self.send_slack(title, message, severity)
        if severity == "critical":
            self.send_pagerduty(title, message, severity)

# Usage
alerts = AlertManager(
    slack_webhook=os.environ.get("SLACK_WEBHOOK"),
    pagerduty_key=os.environ.get("PAGERDUTY_KEY"),
)

alerts.alert(
    title="High Error Rate Detected",
    message="API error rate exceeded 5% threshold (current: 8.3%)",
    severity="critical",
)
```

> **DevOps Relevance:** Monitoring scripts are the nervous system of your infrastructure. Custom Prometheus exporters fill the gaps where standard exporters don't cover your business metrics. Log parsers detect anomalies faster than humans scanning dashboards. Webhook integrations ensure the right people get alerted at the right severity level — Slack for warnings, PagerDuty for critical incidents that need immediate response.

---

## 17. Security in Scripting

### 17.1 Secret Management

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  THE SECRETS HIERARCHY — FROM WORST TO BEST                     │
  │                                                                  │
  │  ❌ TERRIBLE:                                                    │
  │  password="hunter2"           # Hardcoded in source code        │
  │  curl -u admin:pass123        # Credentials in shell history    │
  │                                                                  │
  │  ❌ BAD:                                                         │
  │  .env file committed to git   # Secrets in version control     │
  │  echo "$SECRET" in logs       # Secrets in log output          │
  │                                                                  │
  │  ⚠️ ACCEPTABLE (dev only):                                      │
  │  .env file (gitignored)       # Local dev only                  │
  │  export in ~/.bashrc          # Single-user dev machine        │
  │                                                                  │
  │  ✅ GOOD:                                                        │
  │  CI/CD pipeline secrets       # GitHub Secrets, GitLab CI vars │
  │  Environment variables        # Injected at runtime            │
  │                                                                  │
  │  ✅ BEST:                                                        │
  │  HashiCorp Vault              # Dynamic secrets, auto-rotate   │
  │  AWS Secrets Manager          # Integrated with IAM            │
  │  AWS SSM Parameter Store      # Free tier, encrypted           │
  │  K8s External Secrets         # Synced from vault to K8s       │
  └──────────────────────────────────────────────────────────────────┘
```

```python
#!/usr/bin/env python3
"""Secure secret retrieval patterns."""
import os
import sys
import boto3  # pip install boto3

# --- Pattern 1: Environment variables (minimum viable) ---
def get_secret_from_env(name: str) -> str:
    """Get a secret from environment variables."""
    value = os.environ.get(name)
    if not value:
        print(f"ERROR: Secret {name} not set", file=sys.stderr)
        sys.exit(1)
    return value

db_password = get_secret_from_env("DB_PASSWORD")

# --- Pattern 2: AWS Secrets Manager ---
def get_aws_secret(secret_name: str, region: str = "us-east-1") -> dict:
    """Retrieve a secret from AWS Secrets Manager."""
    import json
    client = boto3.client("secretsmanager", region_name=region)
    response = client.get_secret_value(SecretId=secret_name)
    return json.loads(response["SecretString"])

# Usage
creds = get_aws_secret("prod/myapp/database")
db_url = f"postgresql://{creds['username']}:{creds['password']}@{creds['host']}/{creds['dbname']}"

# --- Pattern 3: AWS SSM Parameter Store ---
def get_ssm_parameter(name: str, region: str = "us-east-1") -> str:
    """Get an encrypted parameter from AWS SSM."""
    ssm = boto3.client("ssm", region_name=region)
    response = ssm.get_parameter(Name=name, WithDecryption=True)
    return response["Parameter"]["Value"]

api_key = get_ssm_parameter("/myapp/prod/api-key")
```

```bash
#!/usr/bin/env bash
set -euo pipefail

# NEVER do this:
# echo "Password is: $DB_PASSWORD"        # Leaks to logs
# curl -u "admin:$PASSWORD" https://...   # Shows in ps output

# DO this:
# Pass secrets via stdin or temp files
curl -sf https://api.example.com \
    -H "Authorization: Bearer $(cat /run/secrets/api_token)" \
    -d @/dev/stdin <<< "$request_body"

# Mask secrets in CI output
echo "::add-mask::${SECRET_VALUE}"  # GitHub Actions
```

### 17.2 Input Sanitization

```python
#!/usr/bin/env python3
"""Input validation — prevent injection attacks."""
import re
import subprocess
import shlex

# --- NEVER trust user input in shell commands ---
# BAD: Shell injection vulnerability
# subprocess.run(f"grep {user_input} /var/log/app.log", shell=True)

# GOOD: Pass as list (no shell interpretation)
def search_logs(pattern: str, log_file: str) -> str:
    """Safely search logs with user-provided pattern."""
    # Validate the log file path
    if not re.match(r'^/var/log/[\w.-]+\.log$', log_file):
        raise ValueError(f"Invalid log path: {log_file}")

    result = subprocess.run(
        ["grep", "-i", "--", pattern, log_file],
        capture_output=True, text=True,
    )
    return result.stdout

# --- Path traversal prevention ---
from pathlib import Path

def safe_file_read(base_dir: str, filename: str) -> str:
    """Read a file, preventing directory traversal attacks."""
    base = Path(base_dir).resolve()
    target = (base / filename).resolve()

    # Ensure the resolved path is still within base_dir
    if not str(target).startswith(str(base)):
        raise ValueError(f"Path traversal detected: {filename}")

    if not target.is_file():
        raise FileNotFoundError(f"File not found: {filename}")

    return target.read_text()

# --- Validate common inputs ---
def validate_hostname(hostname: str) -> bool:
    """Validate a hostname format."""
    pattern = r'^[a-zA-Z0-9]([a-zA-Z0-9-]*[a-zA-Z0-9])?(\.[a-zA-Z0-9]([a-zA-Z0-9-]*[a-zA-Z0-9])?)*$'
    return bool(re.match(pattern, hostname)) and len(hostname) <= 253

def validate_semver(version: str) -> bool:
    """Validate semantic version format."""
    return bool(re.match(r'^v?\d+\.\d+\.\d+(-[a-zA-Z0-9.]+)?$', version))
```

### 17.3 Security Checklist for Scripts

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  SCRIPT SECURITY CHECKLIST                                       │
  │                                                                  │
  │  Secrets:                                                        │
  │  ☐ No hardcoded passwords, tokens, or API keys                 │
  │  ☐ Secrets loaded from env vars or secret manager              │
  │  ☐ .env files are in .gitignore                                 │
  │  ☐ Secrets never printed to stdout/logs                        │
  │  ☐ AWS keys use IAM roles, not static credentials              │
  │                                                                  │
  │  Input Handling:                                                 │
  │  ☐ No shell=True with user input (Python subprocess)           │
  │  ☐ No unquoted variables in Bash (use "$var")                  │
  │  ☐ Path traversal checks for file operations                   │
  │  ☐ Input validation for hostnames, IPs, versions               │
  │                                                                  │
  │  File Operations:                                                │
  │  ☐ Temp files use mktemp (not predictable names)               │
  │  ☐ Cleanup via trap EXIT (Bash) or context managers (Python)   │
  │  ☐ Restrictive permissions: chmod 600 for secrets              │
  │  ☐ Don't write secrets to /tmp (world-readable)                │
  │                                                                  │
  │  Execution:                                                      │
  │  ☐ set -euo pipefail in all Bash scripts                       │
  │  ☐ Principle of least privilege (don't run as root)            │
  │  ☐ Pin dependencies (requirements.txt with versions)           │
  │  ☐ ShellCheck passes in CI                                      │
  │  ☐ No eval() or exec() with untrusted input                   │
  └──────────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha (git history):** If you accidentally commit a secret, deleting it in a new commit does NOT remove it — it's still in git history. You need to use `git filter-branch` or `BFG Repo-Cleaner` AND rotate the compromised credential immediately.

> **DevOps Relevance:** Security in scripting is non-negotiable. One leaked AWS key can cost thousands in minutes (crypto-mining). One shell injection can compromise your entire infrastructure. Use secret managers, validate inputs, and follow the principle of least privilege in every script.

---

## 18. Testing Scripts

### 18.1 Testing Bash with `bats`

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  BATS — BASH AUTOMATED TESTING SYSTEM                           │
  │                                                                  │
  │  Install:                                                        │
  │  npm install -g bats           # Via npm                        │
  │  brew install bats-core        # macOS                          │
  │  apt install bats              # Debian/Ubuntu                  │
  │                                                                  │
  │  File naming: test_*.bats or *_test.bats                       │
  │  Run: bats test/                                                │
  │  Run one: bats test/test_deploy.bats                            │
  └──────────────────────────────────────────────────────────────────┘
```

```bash
#!/usr/bin/env bats
# test_deploy.bats — Test deployment helper functions

# Source the script under test
setup() {
    source ./lib/deploy_helpers.sh
    export TEST_MODE=true
}

@test "validate_version accepts valid semver" {
    run validate_version "v2.3.1"
    [ "$status" -eq 0 ]
}

@test "validate_version rejects invalid format" {
    run validate_version "not-a-version"
    [ "$status" -eq 1 ]
    [[ "$output" =~ "Invalid version" ]]
}

@test "get_deploy_env returns correct env for main branch" {
    GIT_BRANCH="main"
    run get_deploy_env
    [ "$output" = "production" ]
}

@test "get_deploy_env returns staging for develop" {
    GIT_BRANCH="develop"
    run get_deploy_env
    [ "$output" = "staging" ]
}

@test "health_check returns success for healthy service" {
    # Mock curl to return 200
    curl() { echo '{"status":"healthy"}'; return 0; }
    export -f curl

    run health_check "http://localhost:8080"
    [ "$status" -eq 0 ]
}

@test "health_check returns failure for unhealthy service" {
    curl() { return 1; }
    export -f curl

    run health_check "http://localhost:8080"
    [ "$status" -eq 1 ]
}

@test "build_image_tag includes git SHA" {
    git() { echo "abc1234"; }
    export -f git

    run build_image_tag "main"
    [[ "$output" =~ "abc1234" ]]
}
```

### 18.2 Testing Python Scripts with `pytest`

```python
#!/usr/bin/env python3
"""test_deploy.py — Test deployment automation functions."""
import pytest
from unittest.mock import patch, MagicMock
from deploy import (
    validate_config,
    determine_version,
    check_health,
    DeploymentError,
    HealthCheckError,
)

# --- Basic tests ---
class TestValidateConfig:
    def test_valid_config(self):
        config = {"app_name": "myapp", "version": "v2.3.1", "replicas": 3}
        assert validate_config(config) is True

    def test_missing_required_field(self):
        config = {"app_name": "myapp"}  # missing version
        with pytest.raises(ValueError, match="version"):
            validate_config(config)

    def test_invalid_replicas(self):
        config = {"app_name": "myapp", "version": "v1.0", "replicas": -1}
        with pytest.raises(ValueError, match="replicas"):
            validate_config(config)

# --- Parametrized tests (test many cases concisely) ---
@pytest.mark.parametrize("branch,expected", [
    ("main", "v2.3.1"),
    ("develop", "dev-abc1234"),
    ("feature/login", "branch-feature-login-abc1234"),
])
def test_determine_version(branch, expected):
    with patch("deploy.get_git_sha", return_value="abc1234"):
        with patch("deploy.get_git_tag", return_value="v2.3.1"):
            version = determine_version(branch)
            assert version == expected

# --- Mocking external services ---
class TestHealthCheck:
    @patch("deploy.requests.get")
    def test_healthy_service(self, mock_get):
        mock_get.return_value = MagicMock(
            status_code=200,
            json=lambda: {"status": "healthy"},
        )
        assert check_health("http://localhost:8080") is True

    @patch("deploy.requests.get")
    def test_unhealthy_service(self, mock_get):
        mock_get.return_value = MagicMock(status_code=503)
        mock_get.return_value.raise_for_status.side_effect = Exception("503")
        with pytest.raises(HealthCheckError):
            check_health("http://localhost:8080")

    @patch("deploy.requests.get")
    def test_timeout(self, mock_get):
        import requests
        mock_get.side_effect = requests.Timeout()
        with pytest.raises(HealthCheckError, match="timeout"):
            check_health("http://localhost:8080")

# --- Fixtures for test setup/teardown ---
@pytest.fixture
def temp_config(tmp_path):
    """Create a temporary config file for testing."""
    config_file = tmp_path / "config.yaml"
    config_file.write_text("""
app_name: myapp
version: v2.3.1
replicas: 3
""")
    return str(config_file)

def test_load_config_from_file(temp_config):
    config = load_config(temp_config)
    assert config["app_name"] == "myapp"
    assert config["replicas"] == 3
```

### 18.3 CI Test Configuration

```yaml
# .github/workflows/test.yml — Run tests in CI
name: Test Scripts

on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  test-bash:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install bats
        run: npm install -g bats

      - name: ShellCheck
        uses: ludeeus/action-shellcheck@master
        with:
          scandir: './scripts'

      - name: Run bats tests
        run: bats test/bash/

  test-python:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install -r requirements-test.txt

      - name: Run pytest
        run: |
          pytest tests/ \
            --cov=src \
            --cov-report=term-missing \
            --cov-fail-under=80 \
            -v
```

> **DevOps Relevance:** Testing scripts is the difference between "automation" and "reliable automation." bats makes Bash scripts testable. pytest with mocking lets you test Python scripts without hitting real APIs or infrastructure. Setting `--cov-fail-under=80` in CI ensures test coverage never drops below 80%. If your script runs in production, it deserves tests.

---

## 19. Performance & Packaging

### 19.1 Parallel Execution

```bash
#!/usr/bin/env bash
set -euo pipefail

# --- xargs -P for simple parallelism ---
# Run health checks on 100 servers, 10 at a time
cat servers.txt | xargs -P 10 -I {} bash -c '
    if curl -sf --max-time 5 "http://{}:8080/health" > /dev/null 2>&1; then
        echo "OK:   {}"
    else
        echo "FAIL: {}"
    fi
'

# --- GNU Parallel (more features) ---
# Install: apt install parallel / brew install parallel
# Process 100 log files, 8 at a time
find /var/log -name "*.log" | parallel -j8 'gzip {}'

# Parallel rsync to multiple servers
parallel -j4 rsync -avz ./build/ {}:/opt/app/ ::: web01 web02 web03 web04
```

```python
#!/usr/bin/env python3
"""Parallel execution patterns in Python."""
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor, as_completed
import time

# --- ThreadPoolExecutor (I/O-bound: HTTP, SSH, file I/O) ---
def check_server(server: str) -> dict:
    """Check a single server's health."""
    import requests
    try:
        r = requests.get(f"http://{server}:8080/health", timeout=5)
        return {"server": server, "status": "ok", "code": r.status_code}
    except Exception as e:
        return {"server": server, "status": "fail", "error": str(e)}

servers = [f"web{i:02d}" for i in range(1, 51)]  # 50 servers

with ThreadPoolExecutor(max_workers=20) as executor:
    futures = {executor.submit(check_server, s): s for s in servers}
    for future in as_completed(futures):
        result = future.result()
        if result["status"] == "fail":
            print(f"  ❌ {result['server']}: {result.get('error', 'unknown')}")

# --- ProcessPoolExecutor (CPU-bound: parsing, compression) ---
def process_log_file(filepath: str) -> dict:
    """Parse a log file and count errors (CPU-bound)."""
    error_count = 0
    total = 0
    with open(filepath) as f:
        for line in f:
            total += 1
            if "ERROR" in line:
                error_count += 1
    return {"file": filepath, "total": total, "errors": error_count}

from pathlib import Path
log_files = [str(f) for f in Path("/var/log").glob("*.log")]

with ProcessPoolExecutor(max_workers=4) as executor:
    results = list(executor.map(process_log_file, log_files))

for r in sorted(results, key=lambda x: x["errors"], reverse=True)[:5]:
    print(f"  {r['errors']} errors in {r['file']}")
```

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  WHEN TO USE WHAT FOR PARALLELISM                               │
  │                                                                  │
  │  Task Type        Bash                     Python               │
  │  ──────────────────────────────────────────────────────         │
  │  I/O-bound        xargs -P, &/wait        ThreadPoolExecutor   │
  │  (HTTP, SSH,      GNU parallel             asyncio              │
  │   file I/O)                                                      │
  │                                                                  │
  │  CPU-bound        GNU parallel             ProcessPoolExecutor  │
  │  (parsing,                                  multiprocessing     │
  │   compression)                                                   │
  │                                                                  │
  │  Simple list      xargs -P 10              ThreadPool(10)       │
  │  Complex flow     for+& pattern            as_completed()       │
  │                                                                  │
  │  Rule of thumb: max_workers = CPU count for CPU tasks           │
  │                  max_workers = 10-50 for I/O tasks              │
  └──────────────────────────────────────────────────────────────────┘
```

### 19.2 Building CLI Tools with `argparse` and `click`

```python
#!/usr/bin/env python3
"""Professional CLI tool with argparse (stdlib — no dependencies)."""
import argparse
import sys

def main():
    parser = argparse.ArgumentParser(
        description="Deploy application to target environment",
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog="""
Examples:
  %(prog)s deploy staging v2.3.1
  %(prog)s deploy production v2.3.1 --dry-run
  %(prog)s status production
  %(prog)s rollback production
        """,
    )

    subparsers = parser.add_subparsers(dest="command", required=True)

    # Deploy command
    deploy_parser = subparsers.add_parser("deploy", help="Deploy a version")
    deploy_parser.add_argument("environment", choices=["dev", "staging", "production"])
    deploy_parser.add_argument("version", help="Version tag (e.g. v2.3.1)")
    deploy_parser.add_argument("--dry-run", action="store_true", help="Preview without applying")
    deploy_parser.add_argument("--skip-tests", action="store_true")

    # Status command
    status_parser = subparsers.add_parser("status", help="Check deployment status")
    status_parser.add_argument("environment", choices=["dev", "staging", "production"])

    # Rollback command
    rollback_parser = subparsers.add_parser("rollback", help="Rollback to previous version")
    rollback_parser.add_argument("environment", choices=["dev", "staging", "production"])
    rollback_parser.add_argument("--to-version", help="Specific version to rollback to")

    args = parser.parse_args()

    if args.command == "deploy":
        print(f"Deploying {args.version} to {args.environment}")
        if args.dry_run:
            print("  (dry-run mode)")
    elif args.command == "status":
        print(f"Checking status of {args.environment}")
    elif args.command == "rollback":
        print(f"Rolling back {args.environment}")

if __name__ == "__main__":
    main()
```

### 19.3 Packaging Python Scripts for Distribution

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  PACKAGING OPTIONS FOR DEVOPS SCRIPTS                           │
  │                                                                  │
  │  Script → pip Package → Docker Container → Helm Chart          │
  │                                                                  │
  │  Option 1: Just copy the script                                 │
  │  scp deploy.py server:~/bin/                                    │
  │  └── Simple, no dependencies, limited usefulness               │
  │                                                                  │
  │  Option 2: pip installable package                              │
  │  pip install git+https://github.com/org/devops-tools           │
  │  └── Versioned, dependency management, CLI entry points        │
  │                                                                  │
  │  Option 3: Docker container                                      │
  │  docker run ghcr.io/org/deploy-tool:v1.0 deploy staging        │
  │  └── Fully isolated, reproducible, CI-friendly                 │
  │                                                                  │
  │  Option 4: Self-contained binary (PyInstaller / Nuitka)         │
  │  ./deploy-tool deploy staging v2.3.1                            │
  │  └── Single file, no Python required, larger size              │
  └──────────────────────────────────────────────────────────────────┘
```

```toml
# pyproject.toml — Modern Python packaging
[build-system]
requires = ["setuptools>=68", "wheel"]
build-backend = "setuptools.backends._legacy:_Backend"

[project]
name = "devops-tools"
version = "1.0.0"
description = "Internal DevOps automation tools"
requires-python = ">=3.10"
dependencies = [
    "requests>=2.31",
    "pyyaml>=6.0",
    "boto3>=1.34",
    "click>=8.1",
]

[project.scripts]
deploy = "devops_tools.deploy:main"
infra = "devops_tools.infra:main"
monitor = "devops_tools.monitor:main"

[project.optional-dependencies]
dev = ["pytest>=8.0", "pytest-cov", "mypy", "ruff"]
```

```bash
# Install in editable mode (for development)
pip install -e ".[dev]"

# Now you can run:
deploy production v2.3.1
infra plan staging
monitor check-health
```

> **DevOps Relevance:** Performance matters when you're managing hundreds of servers or processing gigabytes of logs. Parallel execution (xargs -P / ThreadPoolExecutor) turns a 10-minute script into a 30-second one. Packaging your scripts as pip-installable CLI tools with `pyproject.toml` makes them versioned, dependency-managed, and shareable across your team.

---

## 20. Production Scenario FAQ

> **15 real-world scripting scenarios** you'll encounter as a DevOps engineer — with battle-tested solutions.

---

### Scenario 1: "My deployment script fails silently"

**Problem:** Script deploys but doesn't report errors. App is broken.

**Root Cause:** Missing `set -euo pipefail` or missing `check=True` in subprocess.

```bash
# FIX: Always use strict mode
#!/usr/bin/env bash
set -euo pipefail

# AND verify after deploying:
sleep 5
if ! curl -sf http://localhost:8080/health; then
    echo "DEPLOY FAILED — rolling back" >&2
    rollback
    exit 1
fi
```

---

### Scenario 2: "rm -rf deleted my entire server"

**Problem:** `rm -rf /$UNDEFINED_VAR/data` expanded to `rm -rf /data` because `$UNDEFINED_VAR` was unset.

**Fix:**
```bash
set -u                           # Catch unset variables
DEPLOY_DIR="${DEPLOY_DIR:?'DEPLOY_DIR must be set'}"
# Use explicit checks before destructive operations
[[ -n "$DEPLOY_DIR" && "$DEPLOY_DIR" != "/" ]] || exit 1
rm -rf "${DEPLOY_DIR}/old_releases"
```

---

### Scenario 3: "Cron job runs twice simultaneously"

**Problem:** Long-running backup cron starts again before previous run finishes.

**Fix:**
```bash
LOCK="/var/run/backup.lock"
exec 200>"$LOCK"
flock -n 200 || { echo "Already running"; exit 0; }
# Now run backup safely
```

---

### Scenario 4: "Script works locally but fails in CI"

**Problem:** Different PATH, shell, or tool versions in CI.

**Checklist:**
```bash
# 1. Use #!/usr/bin/env bash (not /bin/bash)
# 2. Check tool availability
require_cmd() {
    command -v "$1" >/dev/null || { echo "Missing: $1"; exit 1; }
}
require_cmd jq
require_cmd kubectl

# 3. Don't rely on ~/.bashrc (CI doesn't source it)
# 4. Pin tool versions in CI
# 5. Use absolute paths, not relative
```

---

### Scenario 5: "API calls fail intermittently"

**Problem:** Network glitches cause 10% of API calls to fail. Script exits on first failure.

**Fix:**
```python
def api_call_with_retry(url, max_retries=3):
    for attempt in range(1, max_retries + 1):
        try:
            r = requests.get(url, timeout=10)
            r.raise_for_status()
            return r.json()
        except (requests.ConnectionError, requests.Timeout):
            if attempt == max_retries:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

---

### Scenario 6: "Secret leaked in git history"

**Problem:** Database password was committed to `config.py` and pushed.

**Immediate Actions:**
```bash
# 1. ROTATE THE CREDENTIAL IMMEDIATELY (this is the priority)
# Change the DB password NOW, before cleanup

# 2. Remove from history
pip install git-filter-repo
git filter-repo --path config.py --invert-paths

# 3. Force push (coordinate with team)
git push --force --all

# 4. Add to .gitignore
echo "config.py" >> .gitignore

# 5. Post-mortem: set up pre-commit hooks
pip install pre-commit detect-secrets
```

---

### Scenario 7: "Log files filling up disk"

**Problem:** `/var/log` at 100%. Server unresponsive.

**Immediate Fix:**
```bash
# Quick triage — find the culprits
du -sh /var/log/* | sort -rh | head -10

# Truncate without deleting (keeps file handles valid)
truncate -s 0 /var/log/app/huge-log.log

# Compress old logs immediately
find /var/log -name "*.log" -size +100M -exec gzip {} \;

# LONG-TERM: Set up logrotate properly
cat > /etc/logrotate.d/myapp <<'EOF'
/var/log/myapp/*.log {
    daily
    rotate 7
    compress
    missingok
    notifempty
    copytruncate
}
EOF
```

---

### Scenario 8: "Python script runs out of memory on large files"

**Problem:** `data = open("10gb.log").read()` causes OOM.

**Fix:**
```python
# BAD: Loads entire file into memory
data = open("10gb.log").read()
errors = [line for line in data.split("\n") if "ERROR" in line]

# GOOD: Stream line by line (constant memory)
error_count = 0
with open("10gb.log") as f:
    for line in f:
        if "ERROR" in line:
            error_count += 1
print(f"Errors found: {error_count}")
```

---

### Scenario 9: "Need to run a database migration across environments"

**Problem:** Manual migration is error-prone and different per environment.

**Fix:**
```python
#!/usr/bin/env python3
"""Safe database migration runner."""
import subprocess
import sys

def run_migration(env: str, dry_run: bool = False):
    db_url = get_db_url(env)  # From secrets manager

    if dry_run:
        cmd = ["alembic", "upgrade", "head", "--sql"]
        result = subprocess.run(cmd, capture_output=True, text=True)
        print(f"SQL that would be executed:\n{result.stdout}")
        return

    # Create backup first
    backup_database(env)

    # Run migration with timeout
    try:
        subprocess.run(
            ["alembic", "upgrade", "head"],
            env={**os.environ, "DATABASE_URL": db_url},
            check=True, timeout=300,
        )
        print(f"✅ Migration complete for {env}")
    except subprocess.CalledProcessError:
        print(f"❌ Migration failed — restoring backup")
        restore_backup(env)
        sys.exit(1)
```

---

### Scenario 10: "SSH connection drops mid-deployment"

**Problem:** Long deployment over SSH fails because the connection times out.

**Fix:**
```bash
# Use tmux/screen for long-running deployments
ssh server "tmux new-session -d -s deploy './deploy.sh v2.3.1 2>&1 | tee deploy.log'"

# Check status later
ssh server "tmux capture-pane -t deploy -p"

# OR use nohup
ssh server "nohup ./deploy.sh v2.3.1 > deploy.log 2>&1 &"

# BETTER: Use a proper deployment tool (Ansible, etc.)
# that handles connection issues gracefully
```

---

### Scenario 11: "Need to find which pod is consuming the most memory"

**Problem:** K8s namespace is hitting memory limits, need to find the culprit.

**Fix:**
```bash
# Quick one-liner
kubectl top pods -n production --sort-by=memory | head -20

# More detail with custom columns
kubectl get pods -n production -o json | jq -r '
  .items[] |
  .metadata.name as $name |
  .spec.containers[] |
  [$name, .name, .resources.limits.memory // "no-limit",
   .resources.requests.memory // "no-request"] |
  @tsv
' | column -t | sort -k3 -rh
```

---

### Scenario 12: "Terraform state is locked"

**Problem:** Previous `terraform apply` crashed/was killed, leaving a stale lock.

**Fix:**
```bash
# 1. Check who holds the lock
terraform force-unlock LOCK_ID

# 2. If using DynamoDB for state locking:
aws dynamodb delete-item \
    --table-name terraform-locks \
    --key '{"LockID":{"S":"myorg-terraform-prod/infra/terraform.tfstate-md5"}}'

# 3. PREVENTION: Always run terraform in CI, not locally
# 4. Use a wrapper script that checks lock age
```

---

### Scenario 13: "Need to bulk-update Docker image tags across manifests"

**Problem:** 50 K8s manifests need their image tag updated from v2.3.0 to v2.4.0.

**Fix:**
```bash
# sed across multiple files
find k8s/ -name "*.yaml" -exec \
    sed -i 's|myapp:v2\.3\.0|myapp:v2.4.0|g' {} +

# Python for more precision
python3 -c "
import yaml
from pathlib import Path

for f in Path('k8s').rglob('*.yaml'):
    docs = list(yaml.safe_load_all(f.read_text()))
    modified = False
    for doc in docs:
        if doc and doc.get('kind') == 'Deployment':
            for c in doc['spec']['template']['spec']['containers']:
                if 'myapp' in c.get('image', ''):
                    c['image'] = 'myapp:v2.4.0'
                    modified = True
    if modified:
        f.write_text(yaml.dump_all(docs))
        print(f'Updated: {f}')
"
```

---

### Scenario 14: "On-call at 3 AM — service is returning 503s"

**Problem:** Production is down. Need quick diagnosis.

**Runbook script:**
```bash
#!/usr/bin/env bash
# incident_triage.sh — Quick diagnosis for production issues
set -euo pipefail

NS="production"

echo "=== 1. POD STATUS ==="
kubectl get pods -n "$NS" --sort-by='.status.containerStatuses[0].restartCount' | tail -20

echo -e "\n=== 2. RECENT EVENTS ==="
kubectl get events -n "$NS" --sort-by='.lastTimestamp' | tail -20

echo -e "\n=== 3. RECENT DEPLOYMENTS ==="
kubectl rollout history deployment/myapp -n "$NS" | tail -5

echo -e "\n=== 4. RESOURCE USAGE ==="
kubectl top pods -n "$NS" --sort-by=memory 2>/dev/null || echo "(metrics unavailable)"

echo -e "\n=== 5. CONTAINER LOGS (last 50 lines) ==="
kubectl logs -n "$NS" -l app=myapp --tail=50 --since=10m 2>/dev/null || echo "(no logs)"

echo -e "\n=== 6. INGRESS / SERVICE STATUS ==="
kubectl get svc,ingress -n "$NS"

echo -e "\n=== 7. NODE STATUS ==="
kubectl get nodes
kubectl top nodes 2>/dev/null || echo "(metrics unavailable)"
```

---

### Scenario 15: "Need to automate the entire release process"

**Problem:** Manual releases take 2 hours and are error-prone.

**Fix — end-to-end release script:**
```python
#!/usr/bin/env python3
"""release.py — Automated release pipeline."""
import sys
from deploy_lib import (
    pre_flight_checks, run_tests, build_image, push_image,
    deploy_to_staging, run_smoke_tests, deploy_to_production,
    send_notification, create_github_release,
)

def release(version: str, skip_staging: bool = False):
    steps = [
        ("Pre-flight checks", lambda: pre_flight_checks(version)),
        ("Run test suite", lambda: run_tests()),
        ("Build Docker image", lambda: build_image(version)),
        ("Push to registry", lambda: push_image(version)),
    ]

    if not skip_staging:
        steps.extend([
            ("Deploy to staging", lambda: deploy_to_staging(version)),
            ("Run smoke tests", lambda: run_smoke_tests("staging")),
        ])

    steps.extend([
        ("Deploy to production", lambda: deploy_to_production(version)),
        ("Verify production health", lambda: run_smoke_tests("production")),
        ("Create GitHub release", lambda: create_github_release(version)),
    ])

    for i, (name, fn) in enumerate(steps, 1):
        print(f"\n{'='*60}")
        print(f"  Step {i}/{len(steps)}: {name}")
        print(f"{'='*60}")
        try:
            fn()
            print(f"  ✅ {name} complete")
        except Exception as e:
            print(f"  ❌ {name} FAILED: {e}")
            send_notification(f"Release {version} failed at step: {name}", "critical")
            sys.exit(1)

    send_notification(f"Release {version} complete! 🎉", "info")

if __name__ == "__main__":
    import argparse
    parser = argparse.ArgumentParser()
    parser.add_argument("version", help="Release version (e.g. v2.4.0)")
    parser.add_argument("--skip-staging", action="store_true")
    args = parser.parse_args()
    release(args.version, args.skip_staging)
```

---

> **DevOps Relevance:** These 15 scenarios represent the real challenges you'll face in production. Bookmark this section. When you're on-call at 3 AM with a failing service, the incident triage script is worth its weight in gold. When a junior engineer asks "how do I handle X?", point them here. Scripts solve problems — but only if they handle errors, validate inputs, and are tested.

---

[🏠 Home](../README.md) · [Scripting](README.md)
