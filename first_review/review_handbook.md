[🏠 Home](../README.md) · [1st Review](README.md)

# 📝 1st Review — Comprehensive Interview Prep Handbook

> **Audience:** DevOps / Cloud Engineers & Trainees | **Focus:** Deep Internals, Mental Models, Interview Readiness
> **Source:** 1st Technical Review Topics + Extended Coverage
> **Tip:** Every section answers "what happens internally" — the interviewer's favorite question. Use the TOC to jump around.

---

## Table of Contents

1. [Command Execution Internals — What Happens When You Run a Command](#1-command-execution-internals--what-happens-when-you-run-a-command)
2. [How Linux Decides If a Command Is Correct or Incorrect](#2-how-linux-decides-if-a-command-is-correct-or-incorrect)
3. [Redirection Operators & Stream Mechanics](#3-redirection-operators--stream-mechanics)
4. [STDIN, STDOUT, STDERR — The Three Streams](#4-stdin-stdout-stderr--the-three-streams)
5. [Linux Boot Process — From Power Button to Login Prompt](#5-linux-boot-process--from-power-button-to-login-prompt)
6. [Inodes — The Filesystem's Hidden Engine](#6-inodes--the-filesystems-hidden-engine)
7. [rm * — What Really Happens (Hidden Files, ARG_MAX, Edge Cases)](#7-rm---what-really-happens-hidden-files-arg_max-edge-cases)
8. [Process Management — Signals, kill -9 vs kill -15, ps vs top](#8-process-management--signals-kill--9-vs-kill--15-ps-vs-top)
9. [Text Processing — grep vs egrep, Regular Expressions Internals](#9-text-processing--grep-vs-egrep-regular-expressions-internals)
10. [Umask & Disk Usage — Permissions Default + du vs df](#10-umask--disk-usage--permissions-default--du-vs-df)
11. [Shell Scripting & Automation — Scripts, Log Rotation, Systemd Services, Cron](#11-shell-scripting--automation--scripts-log-rotation-systemd-services-cron)
12. [Service Management & Debugging — systemctl, journalctl, Log Locations](#12-service-management--debugging--systemctl-journalctl-log-locations)
13. [Networking Commands — ifconfig, ip, netstat, ss, ping, traceroute, DNS](#13-networking-commands--ifconfig-ip-netstat-ss-ping-traceroute-dns)
14. [Nginx Production Scenarios — Config Deletion, SED, Troubleshooting](#14-nginx-production-scenarios--config-deletion-sed-troubleshooting)
15. [Windows Server & Active Directory — DC, OUs, GPOs](#15-windows-server--active-directory--dc-ous-gpos)
16. [Additional Deep-Dive Topics — Containers, Advanced Linux, Interview Curveballs](#16-additional-deep-dive-topics--containers-advanced-linux-interview-curveballs)
17. [Production Scenario FAQ & Interview Q&A Bank](#17-production-scenario-faq--interview-qa-bank)
18. [Interview Prep — Rapid-Fire Q&A Bank](#18-interview-prep--rapid-fire-qa-bank)
19. [Linux User Login Troubleshooting — "Users Can't Log In"](#19-linux-user-login-troubleshooting--users-cant-log-in)
20. [Physical Disk Mounting — End-to-End Walkthrough](#20-physical-disk-mounting--end-to-end-walkthrough)
21. [systemctl stop vs disable — The Deep Comparison](#21-systemctl-stop-vs-disable--the-deep-comparison)

---

## 1. Command Execution Internals — What Happens When You Run a Command

### 1.1 Why This Question Matters in Interviews

This is the **#1 most-asked Linux internals question**. The interviewer wants to know you understand the kernel, shell, processes, and syscalls — not just "it runs the program." Every DevOps debugging scenario (why a command hangs, why it's not found, why permissions fail) traces back to this flow.

### 1.2 The Complete Lifecycle — Mental Model

When you type `ls -la /tmp` and press Enter, here's **exactly** what happens:

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         WHAT HAPPENS WHEN YOU TYPE A COMMAND                         │
  │                                                                      │
  │  YOU TYPE: ls -la /tmp [ENTER]                                       │
  │     │                                                                │
  │     ▼                                                                │
  │  ┌──────────────────────────────────────────────────┐                │
  │  │  STEP 1: SHELL READS INPUT                       │                │
  │  │  • Terminal sends keystrokes to shell (bash)      │                │
  │  │  • Shell reads the line from STDIN (fd 0)         │                │
  │  │  • readline library handles editing, history      │                │
  │  └─────────────────────┬────────────────────────────┘                │
  │                        ▼                                             │
  │  ┌──────────────────────────────────────────────────┐                │
  │  │  STEP 2: SHELL PARSES THE LINE                   │                │
  │  │  • Tokenization: ["ls", "-la", "/tmp"]            │                │
  │  │  • Checks for: aliases, shell builtins,           │                │
  │  │    functions, special characters (|, >, &, ;)     │                │
  │  │  • Variable expansion: $HOME → /home/user         │                │
  │  │  • Glob expansion: *.txt → file1.txt file2.txt    │                │
  │  └─────────────────────┬────────────────────────────┘                │
  │                        ▼                                             │
  │  ┌──────────────────────────────────────────────────┐                │
  │  │  STEP 3: COMMAND LOOKUP (Resolution Order)       │                │
  │  │                                                  │                │
  │  │  1. Aliases     → alias ls='ls --color=auto'     │                │
  │  │  2. Functions   → my_ls() { ... }                │                │
  │  │  3. Builtins    → cd, echo, export, source       │                │
  │  │  4. Hash table  → cached path from previous run  │                │
  │  │  5. $PATH search → /usr/bin/ls (left to right)   │                │
  │  │                                                  │                │
  │  │  If NONE found → "command not found" error       │                │
  │  └─────────────────────┬────────────────────────────┘                │
  │                        ▼                                             │
  │  ┌──────────────────────────────────────────────────┐                │
  │  │  STEP 4: FORK — Create a child process           │                │
  │  │                                                  │                │
  │  │  Shell calls fork() syscall                      │                │
  │  │  Kernel creates an EXACT COPY of the shell:      │                │
  │  │  • Same memory (copy-on-write)                   │                │
  │  │  • Same open file descriptors                    │                │
  │  │  • Same environment variables                    │                │
  │  │  • NEW PID assigned                              │                │
  │  │                                                  │                │
  │  │  Parent (shell): PID 1000 → waits                │                │
  │  │  Child (copy):   PID 1001 → continues to exec    │                │
  │  └─────────────────────┬────────────────────────────┘                │
  │                        ▼                                             │
  │  ┌──────────────────────────────────────────────────┐                │
  │  │  STEP 5: EXEC — Replace child with new program   │                │
  │  │                                                  │                │
  │  │  Child calls execve("/usr/bin/ls", args, env)    │                │
  │  │  Kernel:                                         │                │
  │  │  • Checks file permissions (execute bit)         │                │
  │  │  • Reads ELF header (binary format)              │                │
  │  │  • Loads program into memory                     │                │
  │  │  • Sets up new stack, heap                       │                │
  │  │  • Replaces child's memory COMPLETELY            │                │
  │  │  • Starts execution at entry point               │                │
  │  │                                                  │                │
  │  │  The child process IS NOW "ls" — bash is gone    │                │
  │  │  from this process. PID stays the same (1001).   │                │
  │  └─────────────────────┬────────────────────────────┘                │
  │                        ▼                                             │
  │  ┌──────────────────────────────────────────────────┐                │
  │  │  STEP 6: PROGRAM EXECUTES                        │                │
  │  │                                                  │                │
  │  │  ls reads /tmp directory (opendir, readdir)      │                │
  │  │  ls calls stat() on each file → gets metadata    │                │
  │  │  ls formats output                               │                │
  │  │  ls writes to STDOUT (fd 1)                      │                │
  │  │  ls calls exit(0) → success                      │                │
  │  └─────────────────────┬────────────────────────────┘                │
  │                        ▼                                             │
  │  ┌──────────────────────────────────────────────────┐                │
  │  │  STEP 7: WAIT & REAP — Shell gets control back   │                │
  │  │                                                  │                │
  │  │  Kernel sends SIGCHLD to parent (shell)          │                │
  │  │  Shell calls wait()/waitpid()                    │                │
  │  │  Kernel returns exit status (0 = success)        │                │
  │  │  Shell stores it in $? variable                  │                │
  │  │  Shell prints next prompt: $                     │                │
  │  └──────────────────────────────────────────────────┘                │
  └──────────────────────────────────────────────────────────────────────┘
```

### 1.3 The fork() + exec() Model — Why Two Steps?

This is a follow-up interviewers love. Why not just "run the program" in one step?

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         WHY fork() + exec() — NOT A SINGLE "run()" CALL             │
  │                                                                      │
  │  The GAP between fork() and exec() is where the magic happens:      │
  │                                                                      │
  │  Shell (parent)                  Child (after fork, before exec)     │
  │  ┌──────────────┐               ┌──────────────────────────────┐    │
  │  │              │  fork()        │  In this gap, the child can: │    │
  │  │  bash        │──────────────►│                              │    │
  │  │  PID 1000    │               │  • Redirect STDOUT to file   │    │
  │  │              │               │    (for > output.txt)        │    │
  │  │  waits...    │               │  • Set up pipes              │    │
  │  │              │               │    (for cmd1 | cmd2)         │    │
  │  │              │               │  • Close file descriptors    │    │
  │  │              │               │  • Change UID/GID (sudo)     │    │
  │  │              │               │  • Set env variables         │    │
  │  │              │               │                              │    │
  │  │              │               │  THEN call exec() to become  │    │
  │  │              │               │  the target program.         │    │
  │  └──────────────┘               └──────────────────────────────┘    │
  │                                                                      │
  │  This is HOW REDIRECTION WORKS:                                      │
  │  $ ls > output.txt                                                   │
  │  1. Shell forks                                                      │
  │  2. Child opens "output.txt", dups fd to STDOUT                      │
  │  3. Child execs "ls" — ls writes to STDOUT → goes to file           │
  │  4. ls has NO IDEA it's writing to a file — it just writes to fd 1  │
  └──────────────────────────────────────────────────────────────────────┘
```

### 1.4 $PATH Search — How Linux Finds the Binary

```bash
# View your PATH
echo $PATH
# /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# PATH is searched LEFT TO RIGHT — first match wins
# That's why /usr/local/bin comes before /usr/bin
# (local installs override system packages)
```

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         $PATH SEARCH ORDER                                           │
  │                                                                      │
  │  Command: "ls"                                                       │
  │                                                                      │
  │  /usr/local/sbin/ls  → exists? NO → next                            │
  │  /usr/local/bin/ls   → exists? NO → next                            │
  │  /usr/sbin/ls        → exists? NO → next                            │
  │  /usr/bin/ls         → exists? YES → check execute permission        │
  │                         │                                            │
  │                         ├── Permission OK? → USE THIS ONE            │
  │                         └── Permission denied? → error               │
  │                                                                      │
  │  If NONE found in any PATH directory:                                │
  │  "bash: ls: command not found"                                       │
  │                                                                      │
  │  Hash table optimization:                                            │
  │  After first lookup, bash CACHES the path:                           │
  │  $ hash                                                              │
  │  hits  command                                                       │
  │     3  /usr/bin/ls                                                   │
  │     1  /usr/bin/grep                                                 │
  │                                                                      │
  │  $ hash -r            # Clear cache (needed after installing new     │
  │                       # binary in a PATH directory)                  │
  └──────────────────────────────────────────────────────────────────────┘
```

### 1.5 Command Resolution Order — Full Priority

```bash
# Check what type a command is:
type ls         # ls is aliased to 'ls --color=auto'
type cd         # cd is a shell builtin
type my_func    # my_func is a function
type python3    # python3 is /usr/bin/python3

# Check ALL possible interpretations:
type -a echo
# echo is a shell builtin        ← builtin wins
# echo is /usr/bin/echo           ← external binary also exists

# The full resolution order:
# 1. Aliases        (alias ls='ls --color')
# 2. Functions      (defined in .bashrc or scripts)
# 3. Shell builtins (cd, echo, export, source, test, etc.)
# 4. Hash table     (cached paths from previous executions)
# 5. $PATH search   (left to right through directories)
```

| Priority | Type | Example | Where Defined |
|----------|------|---------|--------------|
| 1st | Alias | `alias ll='ls -la'` | `~/.bashrc`, `~/.bash_aliases` |
| 2nd | Function | `my_func() { echo hi; }` | `~/.bashrc`, scripts |
| 3rd | Builtin | `cd`, `echo`, `export` | Compiled into bash |
| 4th | Hash | Cached `/usr/bin/ls` | Bash internal hash table |
| 5th | $PATH binary | `/usr/bin/ls` | Filesystem |

> **⚠️ Gotcha:** If you install a new version of a program (say, a newer `python3` in `/usr/local/bin/`), bash may still use the OLD path from its hash table. Run `hash -r` to clear the cache, or `hash -d python3` to clear just that entry.

### 1.6 What Happens with Pipes (`|`)

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         PIPE INTERNALS: cat file.txt | grep "error" | wc -l         │
  │                                                                      │
  │  Shell creates TWO pipes (kernel buffers) BEFORE forking:            │
  │                                                                      │
  │  ┌────────┐  pipe1   ┌────────┐  pipe2   ┌────────┐                │
  │  │  cat   │─────────►│  grep  │─────────►│  wc    │                │
  │  │PID 101 │  STDOUT  │PID 102 │  STDOUT  │PID 103 │                │
  │  │        │  → pipe  │ STDIN  │  → pipe  │ STDIN  │                │
  │  │        │  write   │  pipe  │  write   │  pipe  │                │
  │  │        │  end     │  read  │  end     │  read  │                │
  │  └────────┘          └────────┘          └────────┘                │
  │                                                                      │
  │  Each │ creates:                                                     │
  │  1. pipe() syscall → kernel buffer (64KB default on Linux)           │
  │  2. fork() for each command                                          │
  │  3. Child rewires STDOUT/STDIN to pipe ends                          │
  │  4. exec() to run the actual program                                 │
  │                                                                      │
  │  ALL THREE processes run SIMULTANEOUSLY (parallel).                  │
  │  grep doesn't wait for cat to finish — it reads as data arrives.    │
  │                                                                      │
  │  Important: only the EXIT CODE of the LAST command in the pipe       │
  │  is stored in $? (here: wc's exit code).                             │
  │  Use ${PIPESTATUS[@]} to get all exit codes.                         │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# Check exit codes of all pipe stages:
cat /etc/passwd | grep root | wc -l
echo "${PIPESTATUS[@]}"
# 0 0 0   (all succeeded)

# With a failure in the middle:
cat /nonexistent | grep root | wc -l
echo "${PIPESTATUS[@]}"
# 1 0 0   (cat failed, grep and wc succeeded)

# To catch pipe failures: set -o pipefail
set -o pipefail
cat /nonexistent | grep root | wc -l
echo $?
# 1  (returns first non-zero exit code in pipe)
```

> **DevOps Pro Tip:** In shell scripts, ALWAYS use `set -o pipefail`. Without it, `curl https://api.example.com | jq '.data'` returns success ($?=0) even if `curl` failed — because `jq` ran "successfully" on empty input. This causes silent failures in CI/CD pipelines.

---

## 2. How Linux Decides If a Command Is Correct or Incorrect

### 2.1 The Decision Tree

"Correct" vs "incorrect" happens at **multiple layers** — not just "is the command spelled right."

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         COMMAND VALIDATION — LAYER BY LAYER                          │
  │                                                                      │
  │  You type: systemctl restartt nginx                                  │
  │     │                                                                │
  │     ▼                                                                │
  │  LAYER 1: SHELL PARSING                                              │
  │  ├── Syntax errors?                                                  │
  │  │   • Unmatched quotes: echo "hello                                 │
  │  │   • Bad redirections: > > file                                    │
  │  │   • Incomplete pipes: ls |                                        │
  │  │   → Shell itself prints error, command NEVER runs                 │
  │  │                                                                   │
  │  ▼                                                                   │
  │  LAYER 2: COMMAND LOOKUP                                             │
  │  ├── Not found in aliases, functions, builtins, or $PATH?           │
  │  │   → "command not found" (exit code 127)                          │
  │  │                                                                   │
  │  ▼                                                                   │
  │  LAYER 3: PERMISSION CHECK                                           │
  │  ├── File exists but no execute permission?                          │
  │  │   → "Permission denied" (exit code 126)                          │
  │  │                                                                   │
  │  ▼                                                                   │
  │  LAYER 4: KERNEL EXECUTION                                           │
  │  ├── ELF binary invalid? Wrong architecture?                         │
  │  │   → "Exec format error" or "cannot execute binary"               │
  │  │                                                                   │
  │  ▼                                                                   │
  │  LAYER 5: PROGRAM RUNS — ARGUMENT VALIDATION                         │
  │  ├── The PROGRAM itself checks arguments                             │
  │  │   • systemctl sees "restartt" → not a valid verb                 │
  │  │   • The program prints its own error message                     │
  │  │   • Returns non-zero exit code                                   │
  │  │                                                                   │
  │  ▼                                                                   │
  │  LAYER 6: RUNTIME ERRORS                                             │
  │  ├── Program starts but fails during execution                       │
  │  │   • File not found, network timeout, segfault                    │
  │  │   • Returns non-zero exit code + error message                   │
  │  └──────────────────────────────────────────────────────────────────│
  └──────────────────────────────────────────────────────────────────────┘
```

### 2.2 Exit Codes — The Universal Signal

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         EXIT CODE REFERENCE                                          │
  │                                                                      │
  │  Code  │  Meaning                                                    │
  │  ──────┼────────────────────────────────────────────                 │
  │    0   │  Success ✅                                                 │
  │    1   │  General error (catch-all)                                  │
  │    2   │  Misuse of shell command (bad syntax/arguments)             │
  │  126   │  Command found but NOT EXECUTABLE (permission denied)       │
  │  127   │  Command NOT FOUND in $PATH                                 │
  │  128   │  Invalid exit argument                                      │
  │  128+N │  Killed by signal N (e.g., 128+9=137 = killed by SIGKILL)  │
  │  130   │  Terminated by Ctrl+C (128 + 2 = SIGINT)                   │
  │  137   │  Killed by SIGKILL (128 + 9) — OOM killer or kill -9       │
  │  143   │  Terminated by SIGTERM (128 + 15) — graceful shutdown       │
  │  255   │  Exit status out of range                                   │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# Check exit code of last command:
ls /tmp
echo $?    # 0 (success)

ls /nonexistent
echo $?    # 2 (error — no such file)

not_a_command
echo $?    # 127 (command not found)

touch /root/test    # as non-root user
echo $?    # 1 (permission denied — touch's error)

# Exit code 137 is critical in DevOps:
# Docker containers killed by OOM (Out Of Memory) return 137
# Kubernetes pods showing "OOMKilled" = process got SIGKILL (exit 137)
```

> **⚠️ Gotcha (Interview Curveball):** "What does exit code 137 mean?" — This is a Docker/Kubernetes classic. 137 = 128 + 9 = killed by SIGKILL. Usually means the container exceeded its memory limit. Check `docker inspect` or `kubectl describe pod` for OOMKilled status. Fix: increase memory limits or fix the memory leak.

### 2.3 The Shebang Line — How the Kernel Decides What Runs a Script

```bash
#!/bin/bash          ← "shebang" or "hashbang"
echo "Hello"
```

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         SHEBANG PROCESSING                                           │
  │                                                                      │
  │  When the kernel's execve() reads the file:                          │
  │                                                                      │
  │  First 2 bytes = "#!" ?                                              │
  │  ├── YES → It's a script                                            │
  │  │   • Read the rest of the first line: /bin/bash                   │
  │  │   • Kernel runs: /bin/bash ./script.sh                           │
  │  │   • The interpreter (bash) gets the script as argument           │
  │  │                                                                   │
  │  ├── NO → Check if it's an ELF binary                               │
  │  │   • First 4 bytes = 0x7f 'E' 'L' 'F'?                          │
  │  │   • YES → Load and execute the binary directly                   │
  │  │   • NO → "Exec format error"                                     │
  │  │                                                                   │
  │  Common shebangs:                                                    │
  │  #!/bin/bash           → Bash shell                                 │
  │  #!/bin/sh             → POSIX shell (may be dash on Ubuntu)        │
  │  #!/usr/bin/python3    → Python 3                                   │
  │  #!/usr/bin/env python3 → Find python3 in $PATH (portable!)        │
  │  #!/usr/bin/env node   → Node.js                                    │
  └──────────────────────────────────────────────────────────────────────┘
```

> **DevOps Relevance:** Always use `#!/usr/bin/env bash` instead of `#!/bin/bash` in scripts that run across different systems. On some systems (FreeBSD, macOS Homebrew), `bash` may not be at `/bin/bash`. `env` searches `$PATH` — making your scripts portable.

---

## 3. Redirection Operators & Stream Mechanics

### 3.1 The Three File Descriptors — Foundation

Every process on Linux is born with three open file descriptors:

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         FILE DESCRIPTORS — THE THREE STREAMS                         │
  │                                                                      │
  │  ┌─────────────┐                                                     │
  │  │   PROCESS    │                                                     │
  │  │  (e.g., ls)  │                                                     │
  │  │              │                                                     │
  │  │  fd 0 (STDIN)  ◄──── Keyboard / pipe / file                      │
  │  │              │        (where input comes FROM)                     │
  │  │              │                                                     │
  │  │  fd 1 (STDOUT) ────► Terminal / pipe / file                      │
  │  │              │        (normal output goes HERE)                    │
  │  │              │                                                     │
  │  │  fd 2 (STDERR) ────► Terminal / pipe / file                      │
  │  │              │        (error messages go HERE)                     │
  │  └─────────────┘                                                     │
  │                                                                      │
  │  By DEFAULT, all three point to the terminal.                        │
  │  Redirection CHANGES where they point.                               │
  │                                                                      │
  │  fd 0 = STDIN   = standard input                                     │
  │  fd 1 = STDOUT  = standard output                                    │
  │  fd 2 = STDERR  = standard error                                     │
  │  fd 3+ = additional files opened by the process                      │
  └──────────────────────────────────────────────────────────────────────┘
```

### 3.2 Redirection Operators — Complete Reference

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         REDIRECTION OPERATORS                                        │
  │                                                                      │
  │  OPERATOR        │  WHAT IT DOES                                     │
  │  ────────────────┼─────────────────────────────────────────────      │
  │  >  file         │  Redirect STDOUT to file (CREATE/OVERWRITE)       │
  │  >> file         │  Redirect STDOUT to file (APPEND)                 │
  │  2> file         │  Redirect STDERR to file (CREATE/OVERWRITE)       │
  │  2>> file        │  Redirect STDERR to file (APPEND)                 │
  │  &> file         │  Redirect BOTH STDOUT+STDERR to file              │
  │  &>> file        │  APPEND BOTH STDOUT+STDERR to file                │
  │  2>&1            │  Redirect STDERR to wherever STDOUT goes          │
  │  1>&2            │  Redirect STDOUT to wherever STDERR goes          │
  │  < file          │  Use file as STDIN (input)                        │
  │  << WORD         │  Here-document (inline multi-line input)          │
  │  <<< "string"    │  Here-string (single-line input)                  │
  │  > /dev/null     │  Discard output (black hole)                      │
  └──────────────────────────────────────────────────────────────────────┘
```

### 3.3 `>` vs `>>` — The Critical Difference

```bash
# > CREATES the file if it doesn't exist, TRUNCATES (empties) if it does
echo "line 1" > output.txt
cat output.txt
# line 1

echo "line 2" > output.txt     # OVERWRITES!
cat output.txt
# line 2                        ← "line 1" is GONE

# >> CREATES the file if it doesn't exist, APPENDS if it does
echo "line 1" >> output.txt
echo "line 2" >> output.txt
cat output.txt
# line 2                        ← from the > above
# line 1
# line 2
```

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         >  (OVERWRITE)                  >>  (APPEND)                 │
  │                                                                      │
  │  File before:                  File before:                          │
  │  ┌────────────────┐            ┌────────────────┐                    │
  │  │ existing data  │            │ existing data  │                    │
  │  │ old content    │            │ old content    │                    │
  │  └────────────────┘            └────────────────┘                    │
  │          │                              │                            │
  │          ▼                              ▼                            │
  │  echo "new" > file             echo "new" >> file                    │
  │          │                              │                            │
  │          ▼                              ▼                            │
  │  File after:                   File after:                           │
  │  ┌────────────────┐            ┌────────────────┐                    │
  │  │ new            │            │ existing data  │                    │
  │  │                │            │ old content    │                    │
  │  │  (old data     │            │ new            │                    │
  │  │   DESTROYED)   │            │  (appended)    │                    │
  │  └────────────────┘            └────────────────┘                    │
  └──────────────────────────────────────────────────────────────────────┘
```

### 3.4 What Happens If the File Is Empty?

```bash
# Both > and >> behave the same when the target file is EMPTY:

# File exists but is empty (0 bytes):
> empty_file.txt         # Creates/truncates — result: still empty
echo "data" > empty_file.txt    # Writes "data\n" — file now has content
echo "data" >> empty_file.txt   # Appends "data\n" — same result if file was empty

# File doesn't exist at all:
echo "data" > new_file.txt      # CREATES file, writes data
echo "data" >> new_file.txt     # ALSO CREATES file, writes data
# Both operators create the file — no difference for non-existent files

# THE GOTCHA — redirecting nothing:
> file.txt                       # This TRUNCATES (empties) the file!
                                 # It's a valid command: redirect nothing to file
                                 # DevOps use: quickly empty a log file without
                                 # deleting it (preserves inode, open file handles)

# Equivalent to:
truncate -s 0 file.txt
# or:
: > file.txt                    # : is the null command (no-op) — same effect
```

> **⚠️ Gotcha (Production Disaster):** Running `> /var/log/nginx/access.log` empties the log file. But if Nginx has the file open, it KEEPS WRITING because the file descriptor is still valid (inode didn't change). This is actually how you safely truncate a log file without restarting Nginx. But `rm /var/log/nginx/access.log` and creating a new file would make Nginx write to the DELETED file (old inode) — the new file gets nothing until you `systemctl reload nginx`.

### 3.5 `2>&1` — Explained Properly

This is the **most-asked redirection question** in interviews.

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         2>&1 — REDIRECT STDERR TO STDOUT                             │
  │                                                                      │
  │  Read it as: "file descriptor 2 → point to where fd 1 goes"         │
  │                                                                      │
  │  BEFORE:                           AFTER:                            │
  │  ┌──────────┐                      ┌──────────┐                      │
  │  │ STDOUT(1)│──► Terminal          │ STDOUT(1)│──► Terminal          │
  │  │ STDERR(2)│──► Terminal          │ STDERR(2)│──► Terminal (via 1)  │
  │  └──────────┘                      └──────────┘                      │
  │  (both go to terminal,             (both merged, but functionally    │
  │   but they're separate)             identical if destination same)    │
  │                                                                      │
  │  WHERE IT MATTERS — combined with file redirect:                     │
  │                                                                      │
  │  command > output.txt 2>&1                                           │
  │                                                                      │
  │  Step 1: > output.txt  → STDOUT(1) now points to output.txt         │
  │  Step 2: 2>&1          → STDERR(2) now points to where 1 goes       │
  │                           = STDERR also goes to output.txt           │
  │                                                                      │
  │  ┌──────────┐                                                        │
  │  │ STDOUT(1)│──► output.txt                                          │
  │  │ STDERR(2)│──► output.txt  (via the 2>&1 redirect)                │
  │  └──────────┘                                                        │
  │                                                                      │
  │  ⚠️ ORDER MATTERS!                                                   │
  │                                                                      │
  │  command 2>&1 > output.txt     ← WRONG ORDER!                       │
  │  Step 1: 2>&1          → STDERR(2) points to where 1 is NOW         │
  │                           = terminal (STDOUT hasn't been redirected) │
  │  Step 2: > output.txt  → STDOUT(1) points to output.txt             │
  │  Result: STDOUT → file, STDERR → terminal (NOT what you wanted!)    │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# CORRECT — both to file:
command > output.txt 2>&1

# SHORTHAND (bash 4+) — same thing, easier:
command &> output.txt

# Send STDOUT to one file, STDERR to another:
command > stdout.log 2> stderr.log

# Discard all output:
command > /dev/null 2>&1
# or:
command &> /dev/null

# APPEND both to file:
command >> output.txt 2>&1
# or:
command &>> output.txt
```

### 3.6 Practical DevOps Redirection Patterns

```bash
# Pattern 1: Cron job — log everything, silently
*/5 * * * * /opt/scripts/health-check.sh >> /var/log/health.log 2>&1

# Pattern 2: Background process with logging
nohup java -jar app.jar > /var/log/app.log 2>&1 &

# Pattern 3: Separate success and error logs
./deploy.sh > deploy-success.log 2> deploy-error.log

# Pattern 4: Pipe STDERR through grep (needs process substitution)
command 2>&1 | grep "error"
# This merges stderr into stdout FIRST, then pipes both to grep

# Pattern 5: Pipe ONLY STDERR (not stdout):
command 2>&1 1>/dev/null | grep "error"
# Step 1: stderr → stdout (merged)
# Step 2: original stdout → /dev/null (discarded)
# Step 3: only stderr (now on stdout) goes to grep

# Pattern 6: Tee — write to file AND show on screen:
command 2>&1 | tee output.log
# tee reads from stdin, writes to BOTH file AND stdout

# Pattern 7: Safely empty a log file without stopping the service:
> /var/log/nginx/access.log
# OR
truncate -s 0 /var/log/nginx/access.log
```

> **DevOps Pro Tip:** In production, NEVER use `> file` in scripts where the file might be important. Use `>> file` (append) by default. A typo or re-run of a script with `>` silently destroys existing log data. The only exception is when you intentionally want to create/overwrite (like generating reports).

---

## 4. STDIN, STDOUT, STDERR — The Three Streams

### 4.1 Deep Mental Model

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         THE UNIX STREAM MODEL                                        │
  │                                                                      │
  │  "Everything is a file" — even keyboard input and screen output      │
  │                                                                      │
  │                    ┌──────────────────────────┐                      │
  │  KEYBOARD ────────►│  STDIN  (fd 0)           │                      │
  │  (or file/pipe)    │                          │                      │
  │                    │       ┌──────────┐       │                      │
  │                    │       │ PROCESS  │       │                      │
  │                    │       │ (ls, grep│       │                      │
  │                    │       │  cat...) │       │                      │
  │                    │       └──────────┘       │                      │
  │                    │                          │                      │
  │                    │  STDOUT (fd 1) ──────────┼────► TERMINAL        │
  │                    │  (normal output)         │     (or file/pipe)   │
  │                    │                          │                      │
  │                    │  STDERR (fd 2) ──────────┼────► TERMINAL        │
  │                    │  (error messages)        │     (or file/pipe)   │
  │                    └──────────────────────────┘                      │
  │                                                                      │
  │  KEY INSIGHT: STDOUT and STDERR are SEPARATE channels.               │
  │  They both go to the terminal by default, but can be redirected      │
  │  independently. This is why you see error messages even when you     │
  │  redirect output to a file.                                          │
  └──────────────────────────────────────────────────────────────────────┘
```

### 4.2 Why Separate STDOUT and STDERR?

```bash
# Scenario: You want only the successful results, not errors
find / -name "*.conf" > results.txt 2>/dev/null
# STDOUT (found files) → results.txt
# STDERR (permission denied) → discarded
# Without 2>/dev/null, your terminal fills with "Permission denied" errors

# Scenario: Debug failures in CI
./build.sh > build-output.log 2> build-errors.log
# If build fails, check build-errors.log — clean separation

# Scenario: Pipe only stdout (stderr bypasses the pipe)
ls /tmp /nonexistent | wc -l
# STDOUT: listing of /tmp → piped to wc → counted
# STDERR: "No such file" → goes directly to terminal (NOT piped)
```

### 4.3 File Descriptors Under the Hood

```bash
# Every process has a file descriptor table in /proc:
ls -la /proc/$$/fd
# lrwx------ 1 user user 0 Mar  3 10:00 0 -> /dev/pts/0    ← STDIN
# lrwx------ 1 user user 0 Mar  3 10:00 1 -> /dev/pts/0    ← STDOUT
# lrwx------ 1 user user 0 Mar  3 10:00 2 -> /dev/pts/0    ← STDERR
# ($$ is the PID of the current shell)

# /dev/pts/0 is your terminal (pseudo-terminal)
# When you redirect, the symlink changes:
ls > output.txt &
ls -la /proc/$!/fd
# 0 -> /dev/pts/0         ← STDIN still terminal
# 1 -> /home/user/output.txt  ← STDOUT now points to file
# 2 -> /dev/pts/0         ← STDERR still terminal
```

### 4.4 Advanced: Custom File Descriptors

```bash
# Open fd 3 for writing to a file:
exec 3> custom.log
echo "This goes to fd 3" >&3
echo "This goes to terminal (fd 1)"
exec 3>&-    # Close fd 3

# Open fd 4 for reading from a file:
exec 4< input.txt
read -u 4 LINE    # Read one line from fd 4
echo "$LINE"
exec 4<&-    # Close fd 4

# Practical: swap stdout and stderr
exec 3>&1 1>&2 2>&3 3>&-
# Now STDOUT goes to stderr and vice versa
# Useful for sending "normal" output to error log
```

> **DevOps Relevance:** Understanding file descriptors is essential for debugging "why is my log empty?" scenarios. If a service opens a log file, gets fd 5, then someone deletes and recreates the file, the service STILL WRITES TO THE OLD fd 5 (deleted file). You must signal the service to reopen its files — this is exactly what `logrotate` does with its `postrotate` script.

---

## 5. Linux Boot Process — From Power Button to Login Prompt

### 5.1 Why This Matters for DevOps

This is asked in **every** Linux interview. But beyond interviews, you need this knowledge to:
- Debug VMs that won't boot (stuck at GRUB? kernel panic? systemd dependency loop?)
- Build custom AMIs/Docker images (what gets skipped in containers?)
- Understand recovery mode, single-user mode, root password reset
- Diagnose slow boot times (`systemd-analyze blame`)

### 5.2 The Complete Boot Chain — Mental Model

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         LINUX BOOT PROCESS — COMPLETE CHAIN                          │
  │                                                                      │
  │  ┌─────────────┐                                                     │
  │  │  POWER ON   │                                                     │
  │  └──────┬──────┘                                                     │
  │         ▼                                                            │
  │  ┌─────────────────────────────────────────────────────────────┐     │
  │  │  STAGE 1: FIRMWARE (BIOS / UEFI)                            │     │
  │  │                                                             │     │
  │  │  • POST (Power-On Self Test)                                │     │
  │  │    - Tests CPU, RAM, basic hardware                         │     │
  │  │    - If fails → beep codes, no boot                         │     │
  │  │                                                             │     │
  │  │  • Finds boot device (disk, USB, network)                   │     │
  │  │    - BIOS: reads MBR (first 512 bytes of disk)              │     │
  │  │    - UEFI: reads ESP (EFI System Partition), GPT table      │     │
  │  │                                                             │     │
  │  │  • Loads bootloader into memory and jumps to it             │     │
  │  └──────────────────────────┬──────────────────────────────────┘     │
  │                             ▼                                        │
  │  ┌─────────────────────────────────────────────────────────────┐     │
  │  │  STAGE 2: BOOTLOADER (GRUB2)                                │     │
  │  │                                                             │     │
  │  │  • Displays boot menu (kernel versions, recovery options)   │     │
  │  │  • Reads /boot/grub/grub.cfg                                │     │
  │  │  • Loads the KERNEL image (/boot/vmlinuz-*)                 │     │
  │  │  • Loads the INITRAMFS (/boot/initrd.img-*)                 │     │
  │  │  • Passes kernel parameters (root=, quiet, etc.)            │     │
  │  │  • Hands control to the kernel                              │     │
  │  └──────────────────────────┬──────────────────────────────────┘     │
  │                             ▼                                        │
  │  ┌─────────────────────────────────────────────────────────────┐     │
  │  │  STAGE 3: KERNEL INITIALIZATION                             │     │
  │  │                                                             │     │
  │  │  • Decompresses itself (kernel is compressed on disk)       │     │
  │  │  • Initializes memory management (virtual memory, paging)   │     │
  │  │  • Detects CPU features, sets up interrupt handlers         │     │
  │  │  • Mounts initramfs as temporary root filesystem            │     │
  │  │  • Loads essential drivers from initramfs                   │     │
  │  │    (disk drivers, filesystem drivers, RAID, LVM, LUKS)      │     │
  │  │  • Mounts the REAL root filesystem (/)                      │     │
  │  │  • Starts PID 1 (the init system)                           │     │
  │  └──────────────────────────┬──────────────────────────────────┘     │
  │                             ▼                                        │
  │  ┌─────────────────────────────────────────────────────────────┐     │
  │  │  STAGE 4: INIT SYSTEM (systemd — PID 1)                    │     │
  │  │                                                             │     │
  │  │  • systemd is the FIRST userspace process (PID 1)           │     │
  │  │  • Reads unit files from /etc/systemd/system/               │     │
  │  │  • Processes dependencies, starts services IN PARALLEL      │     │
  │  │  • Mounts filesystems (/etc/fstab)                          │     │
  │  │  • Starts networking, logging (journald), udev              │     │
  │  │  • Reaches the DEFAULT TARGET:                              │     │
  │  │    - multi-user.target (servers — no GUI)                   │     │
  │  │    - graphical.target (desktops — with GUI)                 │     │
  │  └──────────────────────────┬──────────────────────────────────┘     │
  │                             ▼                                        │
  │  ┌─────────────────────────────────────────────────────────────┐     │
  │  │  STAGE 5: LOGIN PROMPT                                      │     │
  │  │                                                             │     │
  │  │  • getty / agetty spawns on TTYs (Ctrl+Alt+F1-F6)          │     │
  │  │  • SSH daemon (sshd) ready for remote login                 │     │
  │  │  • System fully operational                                 │     │
  │  └─────────────────────────────────────────────────────────────┘     │
  └──────────────────────────────────────────────────────────────────────┘
```

### 5.3 BIOS vs UEFI — The Firmware Layer

| Feature | BIOS (Legacy) | UEFI (Modern) |
|---------|--------------|---------------|
| **Age** | 1980s design | 2005+ standard |
| **Partition table** | MBR (max 2TB disks, 4 primary partitions) | GPT (9.4ZB disks, 128+ partitions) |
| **Boot code location** | First 512 bytes of disk (MBR) | EFI System Partition (FAT32, /boot/efi/) |
| **Architecture** | 16-bit real mode | 32/64-bit protected mode |
| **Secure Boot** | ❌ Not supported | ✅ Verifies bootloader signature |
| **Boot speed** | Slower (sequential init) | Faster (parallel init) |
| **UI** | Text-only, keyboard | Mouse-capable, graphical |

```bash
# Check if you're running BIOS or UEFI:
[ -d /sys/firmware/efi ] && echo "UEFI" || echo "BIOS"

# View EFI boot entries:
efibootmgr -v

# View partition table type:
sudo fdisk -l /dev/sda | head -5
# "Disklabel type: gpt" = UEFI
# "Disklabel type: dos" = BIOS/MBR
```

### 5.4 GRUB2 — The Bootloader

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         GRUB2 — KEY FILES AND FLOW                                   │
  │                                                                      │
  │  Configuration:                                                      │
  │  /etc/default/grub          ← Your settings (timeout, kernel params) │
  │  /etc/grub.d/*              ← Scripts that GENERATE grub.cfg         │
  │  /boot/grub/grub.cfg        ← AUTO-GENERATED (never edit directly!) │
  │                                                                      │
  │  After editing /etc/default/grub:                                    │
  │  $ sudo update-grub         ← Regenerates grub.cfg                  │
  │  (or: sudo grub-mkconfig -o /boot/grub/grub.cfg)                    │
  │                                                                      │
  │  /boot/ contents:                                                    │
  │  ├── vmlinuz-5.15.0-91     ← Compressed kernel image                │
  │  ├── initrd.img-5.15.0-91  ← Initial RAM disk (drivers, tools)      │
  │  ├── config-5.15.0-91      ← Kernel build configuration             │
  │  ├── System.map-5.15.0-91  ← Kernel symbol table (debugging)        │
  │  └── grub/                                                           │
  │      └── grub.cfg          ← GRUB menu configuration                │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# Common GRUB settings (/etc/default/grub):
GRUB_TIMEOUT=5              # Seconds to show menu (0 = skip, -1 = wait forever)
GRUB_DEFAULT=0              # Boot first entry by default
GRUB_CMDLINE_LINUX=""       # Extra kernel parameters for ALL entries
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"  # Params for DEFAULT entry only

# Kernel parameters (passed via GRUB → kernel):
# quiet       = suppress most boot messages
# splash      = show boot splash screen
# single      = boot to single-user mode (recovery)
# init=/bin/bash  = bypass systemd, drop to root shell (emergency!)
# nomodeset   = disable GPU drivers (fix display issues)

# Apply changes:
sudo update-grub
```

> **⚠️ Gotcha (Interview Classic):** "How do you reset the root password on a Linux server?" — At GRUB menu, edit the boot entry (press `e`), add `init=/bin/bash` to the kernel line, boot (Ctrl+X). You get a root shell without a password. Mount filesystem read-write (`mount -o remount,rw /`), change password (`passwd`), reboot. **This is why physical security matters** — anyone with console access can reset root.

### 5.5 initramfs — Why a Temporary Root Filesystem?

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         WHY initramfs EXISTS                                         │
  │                                                                      │
  │  PROBLEM:                                                            │
  │  The kernel needs to mount the root filesystem (/) to load drivers.  │
  │  But it needs drivers to READ the disk that has the root filesystem. │
  │  Chicken-and-egg problem!                                            │
  │                                                                      │
  │  SOLUTION: initramfs                                                 │
  │  A tiny, compressed filesystem loaded into RAM by GRUB.              │
  │  Contains just enough drivers and tools to mount the real root.      │
  │                                                                      │
  │  ┌──────────────┐     ┌─────────────────────┐     ┌────────────┐    │
  │  │   GRUB loads  │────►│  initramfs unpacked │────►│ Real root  │    │
  │  │   kernel +    │     │  into RAM           │     │ mounted    │    │
  │  │   initramfs   │     │                     │     │ at /       │    │
  │  │   into RAM    │     │  Contains:          │     │            │    │
  │  └──────────────┘     │  • Disk drivers     │     │  initramfs │    │
  │                        │    (SATA, NVMe,     │     │  discarded │    │
  │                        │     SCSI, virtio)   │     │  from RAM  │    │
  │                        │  • Filesystem       │     │            │    │
  │                        │    drivers (ext4,   │     │  pivot_root│    │
  │                        │     xfs, btrfs)     │     │  switches  │    │
  │                        │  • RAID/LVM/LUKS    │     │  to real / │    │
  │                        │    tools            │     └────────────┘    │
  │                        │  • udev (device     │                       │
  │                        │    detection)       │                       │
  │                        └─────────────────────┘                       │
  │                                                                      │
  │  In containers (Docker): NO initramfs, NO GRUB, NO kernel —         │
  │  the container shares the HOST kernel. Boot stages 1-3 are skipped. │
  │  Container "boot" = start PID 1 (your app) directly.                │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# View initramfs contents:
lsinitramfs /boot/initrd.img-$(uname -r) | head -30

# Regenerate initramfs (e.g., after adding new driver):
sudo update-initramfs -u
# or on RHEL/CentOS:
sudo dracut --force
```

### 5.6 systemd — The Init System (PID 1)

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         SYSTEMD — THE MODERN INIT SYSTEM                             │
  │                                                                      │
  │  systemd replaces the old SysVinit (/etc/init.d/ scripts).          │
  │  It is ALWAYS PID 1 on modern Linux (Ubuntu 16+, RHEL 7+, etc.)     │
  │                                                                      │
  │  KEY CONCEPT: UNITS & TARGETS                                        │
  │                                                                      │
  │  Unit Types:                                                         │
  │  ┌──────────────┬──────────────────────────────────────────┐         │
  │  │  .service    │  A daemon/service (nginx, sshd, docker)  │         │
  │  │  .target     │  A group of units (like runlevels)       │         │
  │  │  .mount      │  A filesystem mount point                │         │
  │  │  .timer      │  A scheduled task (cron replacement)     │         │
  │  │  .socket     │  A network socket (socket activation)    │         │
  │  │  .path       │  Watch a file path for changes           │         │
  │  │  .device     │  A device exposed by udev                │         │
  │  │  .slice      │  A cgroup resource control group         │         │
  │  └──────────────┴──────────────────────────────────────────┘         │
  │                                                                      │
  │  Target = Runlevel Mapping:                                          │
  │  ┌────────────────────┬─────────────────┬─────────────────────┐      │
  │  │  SysVinit Runlevel │  systemd Target │  Description        │      │
  │  ├────────────────────┼─────────────────┼─────────────────────┤      │
  │  │  0                 │  poweroff       │  Shutdown            │      │
  │  │  1 (single-user)   │  rescue         │  Single-user/rescue  │      │
  │  │  2, 3, 4           │  multi-user     │  Multi-user, no GUI  │      │
  │  │  5                 │  graphical      │  Multi-user + GUI    │      │
  │  │  6                 │  reboot         │  Reboot              │      │
  │  └────────────────────┴─────────────────┴─────────────────────┘      │
  │                                                                      │
  │  Boot sequence with systemd:                                         │
  │                                                                      │
  │  default.target (e.g., multi-user.target)                            │
  │       │                                                              │
  │       ├── Requires: basic.target                                     │
  │       │       ├── sysinit.target (udev, tmpfiles, sysctl)           │
  │       │       ├── local-fs.target (mount filesystems from fstab)    │
  │       │       └── swap.target (enable swap)                          │
  │       │                                                              │
  │       ├── Wants: network-online.target                               │
  │       │       └── NetworkManager.service                             │
  │       │                                                              │
  │       ├── Wants: sshd.service                                        │
  │       ├── Wants: nginx.service                                       │
  │       ├── Wants: docker.service                                      │
  │       └── ... (all enabled services)                                 │
  │                                                                      │
  │  PARALLEL startup — systemd starts independent services              │
  │  simultaneously, dramatically speeding up boot.                      │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# Check default target (what system boots into):
systemctl get-default
# multi-user.target  (servers)
# graphical.target   (desktops)

# Change default target:
sudo systemctl set-default multi-user.target

# Analyze boot time:
systemd-analyze                    # Total boot time
systemd-analyze blame              # Time per service (slowest first)
systemd-analyze critical-chain     # Dependency chain (what blocked what)

# Check boot log:
journalctl -b                      # Current boot
journalctl -b -1                   # Previous boot
journalctl -b -p err               # Only errors from current boot
```

### 5.7 Boot Process in Containers vs VMs — Interview Comparison

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         BOOT PROCESS: BARE METAL vs VM vs CONTAINER                  │
  │                                                                      │
  │  Bare Metal / VM:                Container (Docker):                 │
  │  ┌────────────────────┐          ┌────────────────────┐              │
  │  │ 1. BIOS/UEFI       │          │                    │              │
  │  │ 2. GRUB             │          │   ALL SKIPPED      │              │
  │  │ 3. Kernel           │          │   (shares host     │              │
  │  │ 4. initramfs        │          │    kernel)         │              │
  │  │ 5. systemd (PID 1)  │          │                    │              │
  │  │ 6. Services start   │          │ 1. PID 1 = your   │              │
  │  │ 7. Login prompt     │          │    CMD/ENTRYPOINT  │              │
  │  └────────────────────┘          │    (e.g., nginx,   │              │
  │                                   │     node, python)  │              │
  │  Boot time: 30-120 seconds       │                    │              │
  │                                   │ Boot time: <1 sec  │              │
  │  Full kernel, full OS             │ No kernel, no init │              │
  │  Full isolation                   │ Process isolation  │              │
  │                                   └────────────────────┘              │
  │                                                                      │
  │  This is WHY containers start so fast — they skip stages 1-5.       │
  │  And why "systemctl" doesn't work inside most containers —           │
  │  there IS no systemd (PID 1 is your app, not systemd).              │
  └──────────────────────────────────────────────────────────────────────┘
```

> **DevOps Pro Tip:** When debugging "my service doesn't start at boot," trace the dependency chain: `systemctl list-dependencies multi-user.target` shows everything that must start. If your service isn't `enabled` (`systemctl enable myapp`), it won't be in the dependency tree and won't start at boot.

---

## 6. Inodes — The Filesystem's Hidden Engine

### 6.1 Why Interviewers Love This Question

"What is an inode?" is a gatekeeper question. If you answer "it's like metadata for files," that's surface-level. The deep answer explains how the filesystem **actually works**, why `df -i` matters, why hard links share inodes, and why you can delete a file that's still being written to.

### 6.2 What Is an Inode?

An inode (index node) is a **data structure on disk** that stores all metadata about a file EXCEPT its name and content.

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         INODE — THE FILE'S IDENTITY CARD                             │
  │                                                                      │
  │  Every file/directory on a Linux filesystem has an INODE.            │
  │  The inode number is the TRUE identity of a file — NOT its name.    │
  │                                                                      │
  │  INODE CONTAINS:                         INODE DOES NOT CONTAIN:     │
  │  ┌──────────────────────────┐            ┌──────────────────────┐    │
  │  │ • File type (regular,    │            │ • Filename ❌        │    │
  │  │   directory, symlink,    │            │ • File content ❌    │    │
  │  │   socket, pipe, device)  │            │                      │    │
  │  │ • Permissions (rwxr-xr-x)│            │ The FILENAME is      │    │
  │  │ • Owner UID              │            │ stored in the        │    │
  │  │ • Group GID              │            │ DIRECTORY that       │    │
  │  │ • File size (bytes)      │            │ contains the file.   │    │
  │  │ • Timestamps:            │            │                      │    │
  │  │   - atime (last access)  │            │ A directory is just  │    │
  │  │   - mtime (last modify)  │            │ a table of:          │    │
  │  │   - ctime (last change)  │            │ name → inode number  │    │
  │  │ • Hard link count        │            │                      │    │
  │  │ • Pointers to data blocks│            │ (This is why renaming│    │
  │  │   (where content lives   │            │  is instant — just   │    │
  │  │    on disk)              │            │  update the dir      │    │
  │  └──────────────────────────┘            │  entry, not the file)│    │
  │                                          └──────────────────────┘    │
  └──────────────────────────────────────────────────────────────────────┘
```

### 6.3 The Relationship: Directory → Filename → Inode → Data Blocks

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         HOW FILES ARE ACTUALLY STORED                                 │
  │                                                                      │
  │  DIRECTORY /home/user/                                               │
  │  (A directory is a special file: a table of name→inode mappings)     │
  │                                                                      │
  │  ┌────────────────────────────────┐                                  │
  │  │  Directory Entry Table:        │                                  │
  │  │  ┌──────────────┬──────────┐   │                                  │
  │  │  │ Filename     │ Inode #  │   │                                  │
  │  │  ├──────────────┼──────────┤   │                                  │
  │  │  │ .            │  1048577 │   │  (self)                          │
  │  │  │ ..           │  1048576 │   │  (parent)                        │
  │  │  │ notes.txt    │  1048590 │───┼───┐                              │
  │  │  │ script.sh    │  1048591 │   │   │                              │
  │  │  │ data/        │  1048592 │   │   │                              │
  │  │  └──────────────┴──────────┘   │   │                              │
  │  └────────────────────────────────┘   │                              │
  │                                       │                              │
  │          ┌────────────────────────────┘                              │
  │          ▼                                                           │
  │  ┌────────────────────────────────┐                                  │
  │  │  INODE #1048590                │                                  │
  │  │  type: regular file            │                                  │
  │  │  perms: -rw-r--r--            │                                  │
  │  │  owner: 1000 (user)           │                                  │
  │  │  group: 1000 (user)           │                                  │
  │  │  size: 4096 bytes             │                                  │
  │  │  links: 1                     │                                  │
  │  │  atime: 2026-03-03 10:00:00   │                                  │
  │  │  mtime: 2026-03-02 14:30:00   │                                  │
  │  │  ctime: 2026-03-02 14:30:00   │                                  │
  │  │  data block pointers:          │                                  │
  │  │  [block 50012] [block 50013]   │─────┐                           │
  │  └────────────────────────────────┘     │                           │
  │                                         ▼                           │
  │  ┌────────────────────────────────┐                                  │
  │  │  DATA BLOCKS ON DISK           │                                  │
  │  │  Block 50012: "Hello worl"     │                                  │
  │  │  Block 50013: "d!\nThis i..."  │                                  │
  │  │  (Actual file content)         │                                  │
  │  └────────────────────────────────┘                                  │
  └──────────────────────────────────────────────────────────────────────┘
```

### 6.4 Does the Inode Store the Filename? — NO

This is the specific interview question. The answer is **NO** — and here's why:

```bash
# Proof: Same inode, different names (hard links)
echo "hello" > original.txt
ln original.txt hardlink.txt      # Create hard link

ls -li original.txt hardlink.txt
# 1048590 -rw-r--r-- 2 user user 6 Mar  3 10:00 hardlink.txt
# 1048590 -rw-r--r-- 2 user user 6 Mar  3 10:00 original.txt
#     ↑                ↑
#  SAME inode       link count = 2 (two names point to same inode)

# Both names point to the SAME inode (1048590).
# If the filename were stored in the inode, it could only have ONE name.
# The filename is stored in the DIRECTORY ENTRY, not the inode.
```

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         WHY FILENAME IS NOT IN THE INODE                             │
  │                                                                      │
  │  Directory A:                    Directory B:                        │
  │  ┌──────────────────────┐        ┌──────────────────────┐            │
  │  │ "original.txt" → 590 │        │ "hardlink.txt" → 590 │            │
  │  └──────────┬───────────┘        └──────────┬───────────┘            │
  │             │                               │                        │
  │             └───────────┬───────────────────┘                        │
  │                         ▼                                            │
  │              ┌───────────────────┐                                   │
  │              │  INODE #590       │  ← Only ONE inode                 │
  │              │  (metadata + data │     for MULTIPLE names            │
  │              │   block pointers) │     This is a HARD LINK           │
  │              └───────────────────┘                                   │
  │                                                                      │
  │  File is deleted ONLY when link count reaches 0                      │
  │  AND no process has the file open.                                   │
  └──────────────────────────────────────────────────────────────────────┘
```

### 6.5 Inode Exhaustion — The Hidden Disk-Full Scenario

```bash
# Check inode usage:
df -i
# Filesystem       Inodes  IUsed    IFree IUse% Mounted on
# /dev/sda1       6553600 1234567  5319033   19% /

# You can have FREE DISK SPACE but NO FREE INODES:
df -h     # Shows: 50% disk used (space available)
df -i     # Shows: 100% inodes used (NO inodes free!)

# Result: "No space left on device" — even though disk has free space!
# This happens when you have MILLIONS of tiny files.

# Common causes:
# • Mail server with millions of small queue files
# • /tmp filled with session files
# • Container image layers building up
# • Package manager cache

# Fix:
find /tmp -type f -atime +30 -delete   # Delete old temp files
find /var/spool -type f -delete         # Clean mail queue
```

> **⚠️ Gotcha (Production Classic):** A monitoring alert says "disk full" but `df -h` shows 40% used. The real problem is inode exhaustion — `df -i` shows 100%. This commonly hits mail servers, session-heavy web apps, and container hosts. Always check BOTH `df -h` (space) and `df -i` (inodes).

### 6.6 Hard Links vs Symbolic Links

| Feature | Hard Link | Symbolic (Soft) Link |
|---------|-----------|---------------------|
| **Points to** | Same inode number | A filepath (string) |
| **Cross filesystem** | ❌ No (same filesystem only) | ✅ Yes |
| **Link to directories** | ❌ No (prevents cycles) | ✅ Yes |
| **Original deleted** | File still accessible (link count > 0) | **BROKEN** (dangling symlink) |
| **Inode** | Same as original | Different inode (it's its own file) |
| **Created with** | `ln original link` | `ln -s original link` |
| **`ls -l` shows** | Same as regular file | `lrwxrwxrwx ... link -> original` |
| **Size** | Same as original (they ARE the same) | Size of the path string |

```bash
# Hard link:
ln file.txt hard_link.txt
ls -li file.txt hard_link.txt
# Same inode, link count = 2

# Symbolic link:
ln -s file.txt sym_link.txt
ls -li file.txt sym_link.txt
# Different inodes! sym_link has its own inode.
# sym_link.txt -> file.txt

# Delete original:
rm file.txt
cat hard_link.txt    # ✅ WORKS — inode still exists (link count was 2, now 1)
cat sym_link.txt     # ❌ FAILS — "No such file" (path it points to is gone)
```

### 6.7 The Three Timestamps — atime, mtime, ctime

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         INODE TIMESTAMPS                                             │
  │                                                                      │
  │  Timestamp  │  Full Name           │  Updated When                   │
  │  ───────────┼──────────────────────┼────────────────────────────     │
  │  atime      │  Access Time         │  File content READ              │
  │             │                      │  (cat, less, head, read())      │
  │  ───────────┼──────────────────────┼────────────────────────────     │
  │  mtime      │  Modification Time   │  File CONTENT changed           │
  │             │                      │  (echo >> file, vim save)       │
  │  ───────────┼──────────────────────┼────────────────────────────     │
  │  ctime      │  Change Time         │  INODE METADATA changed         │
  │             │                      │  (chmod, chown, rename, or      │
  │             │                      │   mtime changes → ctime too)    │
  └──────────────────────────────────────────────────────────────────────┘

  NOTE: There is NO "creation time" (birth time) in traditional Unix.
  Some modern filesystems (ext4, btrfs) track it, but it's not standard.
  
  Performance: atime updates on every read are expensive on busy servers.
  Modern Linux uses "relatime" by default — only updates atime if it's
  older than mtime (reduces disk writes dramatically).
```

```bash
# View all timestamps:
stat file.txt
# Access: 2026-03-03 10:00:00.000000000 +0530
# Modify: 2026-03-02 14:30:00.000000000 +0530
# Change: 2026-03-02 14:30:00.000000000 +0530
#  Birth: 2026-03-01 09:00:00.000000000 +0530  (ext4 only)

# Find files by timestamp:
find /var/log -mtime -1     # Modified in last 24 hours
find /tmp -atime +30        # Not accessed in 30+ days
find /etc -ctime -1         # Metadata changed in last 24 hours
```

---

## 7. rm * — What Really Happens (Hidden Files, ARG_MAX, Edge Cases)

### 7.1 What `rm *` Actually Does — Step by Step

`rm *` is **NOT** "delete everything." It has very specific behavior that trips up both interviewers and engineers.

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         WHAT HAPPENS WHEN YOU RUN: rm *                              │
  │                                                                      │
  │  Step 1: SHELL (bash) expands * (glob expansion)                     │
  │  ────────────────────────────────────────────────                    │
  │  * matches ALL non-hidden files/dirs in current directory.           │
  │                                                                      │
  │  If directory contains:                                              │
  │    file1.txt  file2.txt  script.sh  .hidden  .bashrc  subdir/       │
  │                                                                      │
  │  Shell expands * to:                                                 │
  │    file1.txt file2.txt script.sh subdir                              │
  │                                                                      │
  │  NOT included: .hidden, .bashrc (hidden files starting with .)       │
  │  NOT included: . and .. (never matched by *)                         │
  │                                                                      │
  │  Step 2: SHELL passes expanded list to rm                            │
  │  ────────────────────────────────────────────                        │
  │  The actual command becomes:                                         │
  │    rm file1.txt file2.txt script.sh subdir                           │
  │                                                                      │
  │  Step 3: rm processes each argument                                  │
  │  ────────────────────────────────────────────                        │
  │  • file1.txt → deleted ✅                                           │
  │  • file2.txt → deleted ✅                                           │
  │  • script.sh → deleted ✅                                           │
  │  • subdir → ERROR: "cannot remove 'subdir': Is a directory"         │
  │    (rm without -r won't delete directories)                          │
  │                                                                      │
  │  RESULT: Regular files deleted. Directories untouched.               │
  │          Hidden files untouched. . and .. untouched.                  │
  └──────────────────────────────────────────────────────────────────────┘
```

### 7.2 Does `rm *` Delete Hidden Files? — NO

```bash
# Hidden files (dotfiles) are NOT matched by *
mkdir /tmp/test && cd /tmp/test
touch file1.txt file2.txt .hidden .secret
ls -la
# .  ..  .hidden  .secret  file1.txt  file2.txt

rm *
ls -la
# .  ..  .hidden  .secret    ← hidden files STILL EXIST

# To delete hidden files too:
rm .*                  # ⚠️ DANGEROUS — includes ".." which means parent dir
                       # bash prevents this: "refusing to remove '.' or '..' "
                       # but some shells don't!

# SAFE ways to delete everything including hidden:
rm -rf /tmp/test/{*,.*} 2>/dev/null   # Both patterns, suppress . and .. errors
# OR:
find /tmp/test -mindepth 1 -delete     # safest — deletes everything INSIDE dir
# OR:
shopt -s dotglob                       # Make * also match dotfiles
rm *                                   # Now deletes hidden files too
shopt -u dotglob                       # Turn it back off
```

### 7.3 What If There Are Too Many Files? — The ARG_MAX Problem

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         ARG_MAX — ARGUMENT LIST TOO LONG                             │
  │                                                                      │
  │  When * expands to THOUSANDS of files, the total size of all         │
  │  filenames may exceed the kernel's ARG_MAX limit.                    │
  │                                                                      │
  │  $ rm *                                                              │
  │  bash: /usr/bin/rm: Argument list too long                           │
  │                                                                      │
  │  WHY: The shell builds the full command line in memory:              │
  │  rm file1.txt file2.txt file3.txt ... file999999.txt                 │
  │  If this string exceeds ARG_MAX (typically 2 MB), exec() fails.     │
  │                                                                      │
  │  Check your limit:                                                   │
  │  $ getconf ARG_MAX                                                   │
  │  2097152  (bytes — about 2 MB on modern Linux)                       │
  │                                                                      │
  │  SOLUTIONS:                                                          │
  │                                                                      │
  │  1. find + -delete (BEST — doesn't use shell expansion):             │
  │     find . -maxdepth 1 -type f -delete                               │
  │                                                                      │
  │  2. find + xargs (batches the arguments):                            │
  │     find . -maxdepth 1 -type f -print0 | xargs -0 rm               │
  │     (-print0 and -0 handle filenames with spaces/special chars)      │
  │                                                                      │
  │  3. find + -exec (runs rm once per file — slower but safe):          │
  │     find . -maxdepth 1 -type f -exec rm {} \;                       │
  │                                                                      │
  │  4. ls + xargs (works but find is better):                           │
  │     ls | xargs rm                                                    │
  │                                                                      │
  │  5. For loop (slow but works):                                       │
  │     for f in *; do rm "$f"; done                                     │
  └──────────────────────────────────────────────────────────────────────┘
```

### 7.4 The Deletion Mechanism — What rm Actually Does to the Filesystem

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         WHAT rm DOES INTERNALLY                                      │
  │                                                                      │
  │  rm calls the unlink() syscall — it does NOT "erase" data.          │
  │                                                                      │
  │  BEFORE rm:                                                          │
  │  Directory entry: "notes.txt" → inode #590                           │
  │  Inode #590: link_count=1, data blocks=[50012, 50013]                │
  │  Blocks 50012-50013: actual file content on disk                     │
  │                                                                      │
  │  AFTER rm notes.txt:                                                 │
  │  1. unlink() removes directory entry "notes.txt → 590"              │
  │  2. Inode #590: link_count decremented (1 → 0)                      │
  │  3. Since link_count = 0:                                            │
  │     • Inode marked as free in inode bitmap                           │
  │     • Data blocks marked as free in block bitmap                     │
  │     • DATA IS STILL ON DISK (just marked "available")               │
  │                                                                      │
  │  KEY INSIGHT: rm does NOT overwrite data!                            │
  │  The data persists until new files overwrite those blocks.           │
  │  This is why data recovery tools (photorec, testdisk) work.         │
  │                                                                      │
  │  EXCEPTION: If a process still has the file OPEN when rm runs:       │
  │  • Directory entry removed (file disappears from ls)                 │
  │  • Inode link_count drops to 0                                       │
  │  • BUT the process still has an open fd → file NOT freed             │
  │  • Data blocks remain allocated until the process closes the fd      │
  │  • "Deleted but still in use" — visible with: lsof +L1              │
  │                                                                      │
  │  This is the classic "disk is full but nothing is using it" bug:     │
  │  A log file was deleted but the writing process still has it open.   │
  │  The space won't be freed until you restart the process.             │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# Find deleted files still held open by processes:
sudo lsof +L1
# Shows files with link count < 1 (deleted but still open)
# COMMAND   PID  USER   FD   TYPE DEVICE SIZE/OFF NLINK NODE NAME
# nginx    1234  root   5w   REG  253,1  50000000  0    590 /var/log/nginx/access.log (deleted)

# Fix: restart the process to release the file
sudo systemctl restart nginx
# OR: truncate via fd (advanced — no restart needed):
# > /proc/1234/fd/5
```

> **⚠️ Gotcha (Production):** After a `logrotate` run, if the process (nginx, apache) doesn't get signaled to reopen its log files, it keeps writing to the OLD (now deleted) file via its open fd. Disk space never gets freed. This is why logrotate configs have `postrotate` blocks that send `kill -USR1` or `systemctl reload` to the service.

### 7.5 Dangerous rm Variations — What to NEVER Run

| Command | What Happens | Danger Level |
|---------|-------------|-------------|
| `rm *` | Deletes non-hidden files in current dir | 🟡 Moderate (directories survive) |
| `rm -r *` | Deletes EVERYTHING (files + dirs) in current dir | 🔴 High |
| `rm -rf *` | Same as above, no confirmation prompts | 🔴 Very High |
| `rm -rf /` | Deletes entire filesystem | ☠️ Catastrophic (modern rm blocks this) |
| `rm -rf /*` | Bypasses the `/` protection — deletes everything | ☠️ Catastrophic |
| `rm -rf $VAR/` | If $VAR is empty → becomes `rm -rf /` | ☠️ Catastrophic |
| `rm -rf ~` | Deletes entire home directory | 🔴 Very High |

```bash
# PROTECTION: Modern rm requires --no-preserve-root for /
rm -rf /
# rm: it is dangerous to operate recursively on '/'
# rm: use --no-preserve-root to override this failsafe

# BUT rm -rf /* BYPASSES THIS (/ isn't the direct argument, /* is):
rm -rf /*    # ☠️ This WILL destroy everything — no protection

# SAFE SCRIPTING PRACTICES:
# 1. Always quote variables:
rm -rf "$DEPLOY_DIR/"        # If DEPLOY_DIR is empty → rm -rf "/"
                              # Quoted empty string → error (good!)
# vs:
rm -rf $DEPLOY_DIR/          # If DEPLOY_DIR is empty → rm -rf /
                              # Unquoted expands to literally "/"  ← DISASTER

# 2. Use set -u (treat unset vars as errors):
set -euo pipefail
rm -rf "$DEPLOY_DIR/"        # If DEPLOY_DIR unset → script errors out (safe!)

# 3. Verify before deleting:
if [ -n "$DEPLOY_DIR" ] && [ -d "$DEPLOY_DIR" ]; then
    rm -rf "$DEPLOY_DIR"
fi
```

> **DevOps Pro Tip:** The `rm -rf $VAR/` disaster is one of the most common production incidents. In 2015, a Valve/Steam update script had `rm -rf "$STEAMROOT/"` where `$STEAMROOT` was unset — it deleted user home directories. Always use `set -u` in scripts, and add guard checks before any `rm -rf` with variables.

---

*Batch 2 complete — Phases 3 & 4 (Linux Boot Process + Filesystem Internals — Inodes, rm *, Hidden Files)*

---

## 8. Process Management — Signals, kill -9 vs kill -15, ps vs top

### 8.1 What Is a Process?

A process is a **running instance of a program**. Every command you run creates at least one process. Understanding processes is fundamental to Linux administration — you'll debug hangs, kill stuck services, analyze resource usage, and manage background jobs daily.

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         PROGRAM vs PROCESS vs THREAD                                 │
  │                                                                      │
  │  PROGRAM          PROCESS              THREAD                        │
  │  ┌──────────┐     ┌──────────────┐     ┌─────────────────────┐      │
  │  │ Binary   │     │ Running      │     │ Lightweight         │      │
  │  │ file on  │────►│ instance of  │────►│ execution unit      │      │
  │  │ disk     │     │ a program    │     │ WITHIN a process    │      │
  │  │          │     │              │     │                     │      │
  │  │ /usr/bin │     │ Has its own: │     │ Shares process's:   │      │
  │  │ /nginx   │     │ • PID        │     │ • Memory space      │      │
  │  │          │     │ • Memory     │     │ • File descriptors  │      │
  │  │ Passive  │     │ • File descs │     │ • PID (same parent) │      │
  │  │ (just    │     │ • UID/GID    │     │                     │      │
  │  │  code)   │     │ • Env vars   │     │ Has its own:        │      │
  │  │          │     │ • Exit code  │     │ • Stack             │      │
  │  │          │     │              │     │ • Thread ID (TID)   │      │
  │  └──────────┘     │ Active       │     │ • CPU registers     │      │
  │                    │ (running)    │     └─────────────────────┘      │
  │  One program can   └──────────────┘                                  │
  │  spawn MANY                          One process can have            │
  │  processes (e.g.,                    MANY threads (e.g., nginx       │
  │  10 nginx workers)                   worker uses thread pool)        │
  └──────────────────────────────────────────────────────────────────────┘
```

### 8.2 Process Lifecycle — From fork() to exit()

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         PROCESS LIFECYCLE                                            │
  │                                                                      │
  │  ┌─────────┐  fork()   ┌─────────┐  exec()   ┌─────────┐           │
  │  │ Parent  │──────────►│ Child   │──────────►│ Child   │           │
  │  │ Process │           │ (clone  │           │ (new    │           │
  │  │ (bash)  │           │  of     │           │  program│           │
  │  │         │           │  parent)│           │  loaded)│           │
  │  └─────────┘           └─────────┘           └────┬────┘           │
  │       │                                           │                 │
  │       │ waitpid()                                 │ runs...         │
  │       │ (parent waits                             │                 │
  │       │  for child)                               ▼                 │
  │       │                                    ┌─────────────┐          │
  │       │                                    │ exit(code)  │          │
  │       │                                    │ Process     │          │
  │       │                                    │ terminates  │          │
  │       │                                    └──────┬──────┘          │
  │       │                                           │                 │
  │       │                                           ▼                 │
  │       │                                    ┌─────────────┐          │
  │       │                                    │ ZOMBIE      │          │
  │       │                                    │ (waiting for│          │
  │       │◄───────────────────────────────────│  parent to  │          │
  │       │  parent reads exit code            │  read exit  │          │
  │       │  → zombie cleaned up ("reaped")    │  code)      │          │
  │       │                                    └─────────────┘          │
  │       ▼                                                             │
  │  Parent continues                                                   │
  │                                                                      │
  │  PROCESS STATES:                                                     │
  │  ┌────────┬───────────────────────────────────────────────────────┐  │
  │  │ State  │ Meaning                                               │  │
  │  ├────────┼───────────────────────────────────────────────────────┤  │
  │  │ R      │ Running / Runnable (on CPU or ready to run)           │  │
  │  │ S      │ Sleeping (interruptible — waiting for I/O, signal)    │  │
  │  │ D      │ Disk sleep (uninterruptible — waiting for disk I/O)   │  │
  │  │        │ ⚠️ CANNOT be killed, even with kill -9!               │  │
  │  │ T      │ Stopped (paused by SIGSTOP / Ctrl+Z)                  │  │
  │  │ Z      │ Zombie (exited, waiting for parent to read status)    │  │
  │  │ X      │ Dead (should never be seen in ps output)              │  │
  │  └────────┴───────────────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────────────────────────┘
```

### 8.3 Zombie Processes — The Interview Favourite

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         ZOMBIE PROCESSES                                             │
  │                                                                      │
  │  A zombie is a process that has EXITED but its parent hasn't         │
  │  called waitpid() to read its exit code yet.                         │
  │                                                                      │
  │  WHY DO ZOMBIES EXIST?                                               │
  │  The kernel keeps the process table entry so the parent can          │
  │  retrieve the child's exit status. Once parent reads it → gone.      │
  │                                                                      │
  │  ARE ZOMBIES DANGEROUS?                                              │
  │  • They use NO memory, NO CPU (they're already dead)                 │
  │  • They use ONE slot in the process table (PID)                      │
  │  • A FEW zombies are normal and harmless                             │
  │  • THOUSANDS of zombies = process table exhaustion = no new          │
  │    processes can start → system appears "frozen"                     │
  │                                                                      │
  │  CAN YOU kill -9 A ZOMBIE?                                           │
  │  NO! The zombie is already dead. kill -9 sends SIGKILL to a         │
  │  living process. You can't kill what's already dead.                  │
  │                                                                      │
  │  HOW TO FIX ZOMBIES:                                                 │
  │  1. Kill the PARENT process → zombies get reparented to PID 1       │
  │     (systemd/init), which automatically reaps them.                  │
  │  2. Send SIGCHLD to parent: kill -SIGCHLD <parent_pid>              │
  │     (reminds parent to call waitpid())                               │
  │  3. Fix the parent program to properly wait() for children.          │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# Find zombie processes:
ps aux | awk '$8 == "Z"'
# or:
ps -eo pid,ppid,stat,comm | grep -w Z

# Count zombies:
ps aux | awk '$8 == "Z"' | wc -l

# Find zombie's parent:
ps -eo pid,ppid,stat,comm | grep -w Z
#  PID  PPID STAT COMMAND
# 5678  1234 Z+   <defunct>
# → Parent PID is 1234. Kill 1234 to reap zombie 5678.

kill -SIGCHLD 1234     # Ask parent nicely to reap children
kill 1234              # If that doesn't work, terminate parent
```

> **⚠️ Gotcha:** A process in state `D` (uninterruptible sleep) is **NOT** a zombie — it's alive but stuck waiting for I/O (often a dead NFS mount or failing disk). You CANNOT kill a `D` state process with `kill -9`. The only fix is resolving the underlying I/O issue (unmount NFS, replace disk) or rebooting.

### 8.4 Signals — The Process Communication System

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         UNIX SIGNALS — COMPLETE REFERENCE                            │
  │                                                                      │
  │  Signals are software interrupts sent to a process.                  │
  │  Every signal has a default action, but processes can CATCH most     │
  │  signals and define custom handlers (except SIGKILL and SIGSTOP).    │
  │                                                                      │
  │  CRITICAL SIGNALS FOR INTERVIEWS:                                    │
  │  ┌─────────┬────────┬──────────┬──────────────────────────────────┐  │
  │  │ Signal  │ Number │ Catchable│ Action / Meaning                 │  │
  │  ├─────────┼────────┼──────────┼──────────────────────────────────┤  │
  │  │ SIGHUP  │ 1      │ ✅ Yes   │ Hangup — terminal closed.       │  │
  │  │         │        │          │ Daemons: reload configuration.   │  │
  │  │         │        │          │ (nginx, apache re-read configs)  │  │
  │  ├─────────┼────────┼──────────┼──────────────────────────────────┤  │
  │  │ SIGINT  │ 2      │ ✅ Yes   │ Interrupt — Ctrl+C sends this.  │  │
  │  │         │        │          │ Polite "stop please."            │  │
  │  ├─────────┼────────┼──────────┼──────────────────────────────────┤  │
  │  │ SIGQUIT │ 3      │ ✅ Yes   │ Quit — Ctrl+\ sends this.       │  │
  │  │         │        │          │ Like SIGINT but dumps core.      │  │
  │  ├─────────┼────────┼──────────┼──────────────────────────────────┤  │
  │  │ SIGKILL │ 9      │ ❌ NO    │ Kill immediately. Cannot be      │  │
  │  │         │        │          │ caught, blocked, or ignored.     │  │
  │  │         │        │          │ Kernel terminates the process.   │  │
  │  ├─────────┼────────┼──────────┼──────────────────────────────────┤  │
  │  │ SIGTERM │ 15     │ ✅ Yes   │ Terminate gracefully. DEFAULT    │  │
  │  │         │        │          │ signal sent by kill command.     │  │
  │  │         │        │          │ Process can clean up & exit.     │  │
  │  ├─────────┼────────┼──────────┼──────────────────────────────────┤  │
  │  │ SIGSTOP │ 19     │ ❌ NO    │ Pause process (Ctrl+Z sends     │  │
  │  │         │        │          │ SIGTSTP which IS catchable).     │  │
  │  ├─────────┼────────┼──────────┼──────────────────────────────────┤  │
  │  │ SIGCONT │ 18     │ ✅ Yes   │ Resume a stopped process.        │  │
  │  ├─────────┼────────┼──────────┼──────────────────────────────────┤  │
  │  │ SIGUSR1 │ 10     │ ✅ Yes   │ User-defined. nginx: reopen     │  │
  │  │         │        │          │ log files. Custom app actions.   │  │
  │  ├─────────┼────────┼──────────┼──────────────────────────────────┤  │
  │  │ SIGUSR2 │ 12     │ ✅ Yes   │ User-defined. nginx: graceful   │  │
  │  │         │        │          │ upgrade of binary.               │  │
  │  ├─────────┼────────┼──────────┼──────────────────────────────────┤  │
  │  │ SIGCHLD │ 17     │ ✅ Yes   │ Child process stopped/exited.    │  │
  │  │         │        │          │ Parent uses this to reap.        │  │
  │  ├─────────┼────────┼──────────┼──────────────────────────────────┤  │
  │  │ SIGPIPE │ 13     │ ✅ Yes   │ Write to pipe with no reader.   │  │
  │  │         │        │          │ Default: terminate process.      │  │
  │  ├─────────┼────────┼──────────┼──────────────────────────────────┤  │
  │  │ SIGALRM │ 14     │ ✅ Yes   │ Alarm clock timer expired.       │  │
  │  ├─────────┼────────┼──────────┼──────────────────────────────────┤  │
  │  │ SIGSEGV │ 11     │ ✅ Yes*  │ Segmentation fault — invalid     │  │
  │  │         │        │          │ memory access. Usually fatal.    │  │
  │  └─────────┴────────┴──────────┴──────────────────────────────────┘  │
  │                                                                      │
  │  * SIGSEGV CAN be caught but almost never should be.                 │
  │    If you catch it and try to continue, you're in undefined          │
  │    behavior territory.                                               │
  │                                                                      │
  │  ONLY TWO SIGNALS CANNOT BE CAUGHT:                                  │
  │  SIGKILL (9) — always kills    SIGSTOP (19) — always pauses         │
  └──────────────────────────────────────────────────────────────────────┘
```

### 8.5 kill -9 vs kill -15 — The Most Important Interview Question

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         kill -15 (SIGTERM) vs kill -9 (SIGKILL)                      │
  │                                                                      │
  │  kill -15 (SIGTERM) — "Please shut down gracefully"                  │
  │  ┌─────────────────────────────────────────────────────────────────┐ │
  │  │ • DEFAULT signal sent by `kill` (kill PID = kill -15 PID)      │ │
  │  │ • Process RECEIVES the signal and can HANDLE it                │ │
  │  │ • Process can:                                                 │ │
  │  │   - Finish current requests                                    │ │
  │  │   - Close database connections cleanly                         │ │
  │  │   - Flush buffers to disk                                      │ │
  │  │   - Remove temp/lock/PID files                                 │ │
  │  │   - Send "goodbye" to connected clients                        │ │
  │  │   - Log "shutting down gracefully" message                     │ │
  │  │   - Exit with code 0                                           │ │
  │  │ • Process can ALSO ignore SIGTERM entirely (misbehaving app)   │ │
  │  └─────────────────────────────────────────────────────────────────┘ │
  │                                                                      │
  │  kill -9 (SIGKILL) — "Die immediately, no questions"                 │
  │  ┌─────────────────────────────────────────────────────────────────┐ │
  │  │ • Process NEVER receives this signal                           │ │
  │  │ • The KERNEL terminates the process directly                   │ │
  │  │ • No cleanup happens:                                          │ │
  │  │   - Open files NOT flushed (data loss possible)                │ │
  │  │   - Database connections NOT closed (locks may persist)        │ │
  │  │   - Temp/lock/PID files NOT removed (stale locks!)            │ │
  │  │   - Child processes become ORPHANS (reparented to PID 1)      │ │
  │  │   - Shared memory segments may leak                            │ │
  │  │ • Use ONLY when SIGTERM doesn't work (after waiting)          │ │
  │  └─────────────────────────────────────────────────────────────────┘ │
  │                                                                      │
  │  THE CORRECT SEQUENCE:                                               │
  │                                                                      │
  │   kill <PID>          ← Step 1: Send SIGTERM (graceful)              │
  │       │                                                              │
  │       ▼ wait 5-30 seconds                                           │
  │                                                                      │
  │   kill -0 <PID>       ← Step 2: Check if still alive                 │
  │       │                  (kill -0 sends NO signal, just checks PID)  │
  │       │                                                              │
  │   ┌───┴───┐                                                          │
  │   │Still  │ YES → kill -9 <PID>  ← Step 3: Force kill as last resort│
  │   │alive? │                                                          │
  │   │       │ NO  → Done ✅ (process exited gracefully)               │
  │   └───────┘                                                          │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# The professional way to stop a process:
PID=1234

kill "$PID"                        # Step 1: SIGTERM
sleep 5                            # Step 2: Wait
if kill -0 "$PID" 2>/dev/null; then
    echo "Process still alive, sending SIGKILL..."
    kill -9 "$PID"                 # Step 3: SIGKILL (last resort)
fi

# How systemd does it (systemctl stop myservice):
# 1. Sends SIGTERM to main process
# 2. Waits TimeoutStopSec (default: 90s)
# 3. If still alive → sends SIGKILL
# You can configure this in the unit file:
# [Service]
# TimeoutStopSec=30
# KillSignal=SIGTERM
# FinalKillSignal=SIGKILL

# Send signal by name:
kill -SIGHUP 1234      # Reload config (nginx, apache)
kill -SIGUSR1 1234     # Reopen logs (nginx)

# Kill all processes by name:
killall nginx          # Send SIGTERM to all nginx processes
pkill -f "python app"  # Kill processes matching pattern

# Kill entire process group:
kill -- -$(ps -o pgid= -p $PID | tr -d ' ')
```

> **DevOps Pro Tip:** `systemctl stop` follows the SIGTERM→wait→SIGKILL pattern automatically. But if you see `A stop job is running for <service>`, it means the process ignored SIGTERM and systemd is waiting its `TimeoutStopSec` before SIGKILL. If your app takes too long, tune `TimeoutStopSec` in the unit file — don't just `kill -9` from the terminal.

### 8.6 Orphan Processes — When Parent Dies First

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         ORPHAN vs ZOMBIE — DON'T CONFUSE THEM                        │
  │                                                                      │
  │  ZOMBIE:                           ORPHAN:                           │
  │  Child is DEAD,                    Parent is DEAD,                   │
  │  parent hasn't reaped it.          child is STILL RUNNING.           │
  │                                                                      │
  │  ┌────────┐    ┌────────┐          ┌────────┐    ┌────────┐         │
  │  │ Parent │    │ Child  │          │ Parent │    │ Child  │         │
  │  │ ALIVE  │    │ ZOMBIE │          │ DEAD   │    │ ALIVE  │         │
  │  │ (not   │    │ (Z)    │          │        │    │ (still │         │
  │  │  wait) │    │        │          │        │    │  runs) │         │
  │  └────────┘    └────────┘          └────────┘    └────┬───┘         │
  │                                                       │              │
  │  Fix: kill parent                   Kernel reparents  │              │
  │  → PID 1 reaps zombie              child to PID 1 ───┘              │
  │                                     (systemd adopts orphans         │
  │                                      and reaps them when done)      │
  │                                                                      │
  │  Dangerous: can exhaust PID table   Safe: PID 1 handles cleanup     │
  └──────────────────────────────────────────────────────────────────────┘
```

### 8.7 ps vs top vs htop — Process Viewing Tools

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         PROCESS MONITORING TOOLS COMPARISON                          │
  │                                                                      │
  │  ┌──────┬─────────────┬──────────────────────────────────────────┐   │
  │  │ Tool │ Type        │ Use Case                                 │   │
  │  ├──────┼─────────────┼──────────────────────────────────────────┤   │
  │  │ ps   │ Snapshot    │ List processes at this instant. Good for │   │
  │  │      │ (one-shot)  │ scripting, piping to grep/awk.           │   │
  │  ├──────┼─────────────┼──────────────────────────────────────────┤   │
  │  │ top  │ Real-time   │ Live, updating view. CPU, memory, load. │   │
  │  │      │ (built-in)  │ Available on EVERY Linux system.         │   │
  │  ├──────┼─────────────┼──────────────────────────────────────────┤   │
  │  │ htop │ Real-time   │ Better top: color, mouse, tree view,    │   │
  │  │      │ (install)   │ kill from UI. Needs installation.        │   │
  │  └──────┴─────────────┴──────────────────────────────────────────┘   │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# ──────────────────── PS ESSENTIALS ────────────────────
# BSD-style (most common):
ps aux
# a = all users, u = user-oriented format, x = include no-terminal processes
#
# USER       PID %CPU %MEM    VSZ   RSS TTY  STAT START   TIME COMMAND
# root         1  0.0  0.1 225816  9556 ?    Ss   Mar02   0:03 /lib/systemd/systemd
# nginx     1234  0.2  0.5  50324 25000 ?    S    10:00   0:45 nginx: worker process

# Unix-style (System V):
ps -ef
# UID        PID  PPID  C STIME TTY          TIME CMD
# root         1     0  0 Mar02 ?        00:00:03 /lib/systemd/systemd

# COLUMNS DECODED:
# PID   = Process ID
# PPID  = Parent Process ID
# %CPU  = CPU usage percentage
# %MEM  = Memory usage percentage
# VSZ   = Virtual memory size (total addressable memory — often huge, not real usage)
# RSS   = Resident Set Size (actual physical RAM used — THIS is what matters)
# STAT  = Process state (R/S/D/Z/T + modifiers)
# TTY   = Terminal attached (? = no terminal = daemon)
# TIME  = Cumulative CPU time

# Custom format — show exactly what you need:
ps -eo pid,ppid,user,%cpu,%mem,stat,comm --sort=-%mem | head -20
# Top 20 processes by memory usage

# Find specific processes:
ps aux | grep nginx          # ⚠️ also matches "grep nginx" itself
ps aux | grep [n]ginx        # Trick: [n] doesn't match the grep process
pgrep -a nginx               # Better — built for this purpose

# Process tree — who spawned whom:
ps auxf                      # Forest view (tree)
pstree -p                    # Full process tree with PIDs
pstree -p 1234               # Tree for specific PID

# ──────────────────── TOP ESSENTIALS ────────────────────
# Launch:
top

# HEADER EXPLAINED:
# top - 10:30:00 up 5 days,  3:15,  2 users,  load average: 0.52, 0.41, 0.38
#                                               ↑      ↑      ↑
#                                             1 min  5 min  15 min
# Load Average: number of processes waiting for CPU
# Rule of thumb: if load avg > number of CPU cores → system is overloaded
# Check cores: nproc    or    lscpu | grep "^CPU(s)"

# Interactive keys in top:
# P = sort by CPU (default)
# M = sort by memory
# T = sort by time
# k = kill a process (enter PID)
# r = renice a process (change priority)
# 1 = show per-CPU usage (toggle)
# c = show full command line
# H = show threads
# q = quit
```

### 8.8 Background & Foreground Job Control

```bash
# Run command in background:
./long_task.sh &             # & sends to background
# [1] 5678                   # Job number [1], PID 5678

# List background jobs:
jobs
# [1]+  Running    ./long_task.sh &

# Bring to foreground:
fg %1                        # Bring job 1 to foreground

# Pause (stop) running process:
# Press Ctrl+Z               # Sends SIGTSTP (like SIGSTOP but catchable)
# [1]+  Stopped    ./long_task.sh

# Resume in background:
bg %1                        # Resume job 1 in background

# Disconnect process from terminal:
nohup ./long_task.sh &       # Survives terminal close (redirects to nohup.out)
disown %1                    # Remove job from shell's job table (survives logout)

# Modern alternative — use screen or tmux:
tmux new -s mysession        # Create named session
# run commands...
# Ctrl+B, D                  # Detach (keeps running)
tmux attach -t mysession     # Re-attach later (even from different SSH!)
```

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         JOB CONTROL FLOW                                             │
  │                                                                      │
  │                  ┌────────────────────┐                              │
  │                  │ FOREGROUND         │                              │
  │        ┌────────►│ (has terminal)     │◄────────┐                   │
  │        │         │ Ctrl+C = SIGINT    │         │                   │
  │        │         └────────┬───────────┘         │                   │
  │        │                  │ Ctrl+Z              │ fg %N             │
  │        │                  │ (SIGTSTP)           │                   │
  │        │                  ▼                     │                   │
  │        │         ┌────────────────────┐         │                   │
  │   fg %N│         │ STOPPED            │         │                   │
  │        │         │ (paused)           │─────────┘                   │
  │        │         └────────┬───────────┘                             │
  │        │                  │ bg %N                                    │
  │        │                  ▼                                          │
  │        │         ┌────────────────────┐                              │
  │        └─────────│ BACKGROUND         │                              │
  │                  │ (runs, no terminal)│                              │
  │                  │ & at end of command│                              │
  │                  └────────────────────┘                              │
  └──────────────────────────────────────────────────────────────────────┘
```

> **DevOps Pro Tip:** Never use `&` or `nohup` to run production services. Use systemd unit files instead — they provide automatic restart, logging via `journald`, resource limits via cgroups, dependency management, and `systemctl status` visibility. A `nohup ./app &` process is invisible to systemd, won't restart on crash, and its logs may be lost.

### 8.9 Process Priority — nice and renice

```bash
# Nice value: -20 (highest priority) to +19 (lowest priority)
# Default: 0
# Only root can set negative nice values (higher priority)

# Start process with low priority:
nice -n 10 ./heavy_task.sh    # Lower priority (nice to others)
nice -n -5 ./urgent_task.sh   # Higher priority (needs root)

# Change priority of running process:
renice 15 -p 1234             # Set PID 1234 to nice value 15
renice -5 -p 1234             # Needs root for negative values
renice 10 -u www-data         # All processes of user www-data

# View nice values:
ps -eo pid,ni,comm | head
# PID  NI COMMAND
#   1   0 systemd
# 1234 10 backup.sh

# In top: NI column shows nice value, PR shows kernel priority
```

### 8.10 /proc Filesystem — Process Internals Exposed

```bash
# /proc/<PID>/ is a virtual filesystem exposing process internals:

ls /proc/1234/
# cmdline    = full command line that started the process
# cwd        = symlink to current working directory
# environ    = environment variables (null-separated)
# exe        = symlink to the actual binary
# fd/        = directory of open file descriptors
# maps       = memory mappings
# status     = human-readable process status
# stat       = machine-readable process stats
# limits     = resource limits (ulimit)
# io         = I/O statistics (bytes read/written)
# net/       = network info for this process's namespace

# Useful debugging commands:
cat /proc/1234/cmdline | tr '\0' ' '     # See full command line
ls -la /proc/1234/fd/                     # See all open files
cat /proc/1234/status | grep -i mem       # Memory usage
readlink /proc/1234/exe                   # What binary is running
cat /proc/1234/limits                     # Resource limits

# Count open files:
ls /proc/1234/fd | wc -l

# System-wide info:
cat /proc/loadavg           # Load average
cat /proc/meminfo           # Detailed memory info
cat /proc/cpuinfo           # CPU info
cat /proc/uptime            # System uptime in seconds
```

> **⚠️ Gotcha:** `/proc` is NOT on disk — it's a virtual filesystem generated by the kernel on-the-fly. Files in `/proc` have zero size (`stat` shows 0 bytes) but produce content when read. This is why `ls -l /proc/meminfo` shows 0 bytes but `cat /proc/meminfo` shows data. Don't try to `cp -r /proc/` — it won't work as expected.

---

## 9. Text Processing — grep vs egrep, Regular Expressions Internals

### 9.1 Why Text Processing Matters in DevOps

Linux is text-based. Config files, logs, command output — everything is text. Your ability to search, filter, transform, and extract data from text determines how fast you debug production issues.

### 9.2 grep — The Fundamental Search Tool

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         grep — GLOBAL REGULAR EXPRESSION PRINT                       │
  │                                                                      │
  │  Origin: from the ed command g/re/p                                  │
  │  (globally search for a regular expression and print matching lines) │
  │                                                                      │
  │  VARIANTS:                                                           │
  │  ┌────────────┬─────────────────────────────────────────────────┐    │
  │  │ Command    │ Regex Flavor                                    │    │
  │  ├────────────┼─────────────────────────────────────────────────┤    │
  │  │ grep       │ Basic Regular Expressions (BRE)                 │    │
  │  │            │ Metacharacters: . * ^ $ [ ] \( \) \{ \}         │    │
  │  │            │ + ? | ( ) must be escaped: \+ \? \| \( \)       │    │
  │  ├────────────┼─────────────────────────────────────────────────┤    │
  │  │ egrep      │ Extended Regular Expressions (ERE)              │    │
  │  │ = grep -E  │ Same as grep but: + ? | ( ) work WITHOUT \     │    │
  │  │            │ More natural syntax for complex patterns        │    │
  │  ├────────────┼─────────────────────────────────────────────────┤    │
  │  │ fgrep      │ Fixed strings (NO regex at all)                 │    │
  │  │ = grep -F  │ Treats pattern as literal text. FASTEST.        │    │
  │  │            │ Use when searching for exact strings with . * $ │    │
  │  ├────────────┼─────────────────────────────────────────────────┤    │
  │  │ grep -P    │ Perl-Compatible Regex (PCRE)                    │    │
  │  │            │ Most powerful: lookahead, lookbehind, \d, \w    │    │
  │  │            │ Not available on all systems (needs libpcre)    │    │
  │  └────────────┴─────────────────────────────────────────────────┘    │
  │                                                                      │
  │  RULE OF THUMB:                                                      │
  │  Use grep -E (egrep) for patterns with + ? | ( )                    │
  │  Use grep -F (fgrep) for literal strings (e.g., searching for ".")  │
  │  Use grep -P for advanced patterns (\d, lookahead, etc.)            │
  └──────────────────────────────────────────────────────────────────────┘
```

### 9.3 grep vs egrep — Side-by-Side Comparison

```bash
# SAME search, different syntax:

# Match "error" or "warning" in a log:
grep 'error\|warning' /var/log/syslog          # BRE: must escape |
grep -E 'error|warning' /var/log/syslog        # ERE: | works directly
egrep 'error|warning' /var/log/syslog          # Same as grep -E

# Match one or more digits:
grep '[0-9]\+' file.txt                         # BRE: must escape +
grep -E '[0-9]+' file.txt                       # ERE: + works directly

# Grouping:
grep '\(error\|warn\).*timeout' file.txt        # BRE: escape ( ) |
grep -E '(error|warn).*timeout' file.txt        # ERE: natural syntax

# WHEN IT MATTERS — searching for literal special characters:
# Search for "192.168.1.1" in a file:
grep '192.168.1.1' file.txt        # ⚠️ . matches ANY char! Matches "192x168y1z1"
grep -F '192.168.1.1' file.txt     # ✅ Literal search, dots are just dots
grep '192\.168\.1\.1' file.txt     # ✅ Escaped dots in regex
```

> **⚠️ Gotcha:** `egrep` and `fgrep` are deprecated (you'll see warnings on newer systems). Use `grep -E` and `grep -F` instead. They're functionally identical but `grep -E` / `grep -F` are the POSIX-standard forms.

### 9.4 Essential grep Flags — Interview Must-Knows

```bash
# ──────────────────── MATCHING FLAGS ────────────────────
grep -i 'error' file.txt       # Case insensitive
grep -w 'error' file.txt       # Whole word (won't match "errors" or "myerror")
grep -x 'exact line' file.txt  # Entire line must match

# ──────────────────── OUTPUT FLAGS ────────────────────
grep -n 'error' file.txt       # Show line NUMBERS
grep -c 'error' file.txt       # COUNT of matching lines (not occurrences!)
grep -l 'error' *.log          # Only show FILENAMES that contain match
grep -L 'error' *.log          # Only show filenames that DON'T contain match
grep -o 'error' file.txt       # Only show the MATCHING PART (not whole line)
grep -m 5 'error' file.txt     # Stop after first 5 matches

# ──────────────────── CONTEXT FLAGS ────────────────────
grep -A 3 'error' file.txt     # Show 3 lines AFTER match
grep -B 3 'error' file.txt     # Show 3 lines BEFORE match
grep -C 3 'error' file.txt     # Show 3 lines BEFORE + AFTER (context)

# ──────────────────── INVERSION & RECURSION ────────────────────
grep -v 'debug' file.txt       # INVERT — show lines that DON'T match
grep -r 'TODO' /project/       # Recursive search in directory
grep -rn 'TODO' /project/      # Recursive + line numbers
grep -rn --include='*.py' 'import' /project/  # Only search Python files

# ──────────────────── PRODUCTION PATTERNS ────────────────────
# Find errors in last hour of logs:
grep "$(date '+%b %e %H')" /var/log/syslog | grep -i error

# Count HTTP status codes in access log:
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# Find IPs with most 5xx errors:
grep ' 5[0-9][0-9] ' /var/log/nginx/access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head

# Watch log in real-time for errors:
tail -f /var/log/syslog | grep --line-buffered -i error

# Multi-pattern search (any of these):
grep -E 'error|critical|fatal|panic' /var/log/syslog

# Exclude patterns:
grep -E 'error|warning' /var/log/syslog | grep -v 'known_issue'
```

### 9.5 Regular Expressions — The Complete Mental Model

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         REGEX BUILDING BLOCKS                                        │
  │                                                                      │
  │  ANCHORS — WHERE to match                                            │
  │  ┌────┬──────────────────────────────────────────────────────────┐   │
  │  │ ^  │ Start of line          ^Error = line starts with "Error" │   │
  │  │ $  │ End of line            \.log$ = line ends with ".log"    │   │
  │  │ \b │ Word boundary (PCRE)   \berror\b = whole word "error"   │   │
  │  └────┴──────────────────────────────────────────────────────────┘   │
  │                                                                      │
  │  CHARACTER CLASSES — WHAT to match                                   │
  │  ┌─────────────┬────────────────────────────────────────────────┐    │
  │  │ .           │ Any single character (except newline)          │    │
  │  │ [abc]       │ Any ONE of: a, b, or c                         │    │
  │  │ [a-z]       │ Any lowercase letter                           │    │
  │  │ [0-9]       │ Any digit                                      │    │
  │  │ [^abc]      │ Any character EXCEPT a, b, c                   │    │
  │  │ [a-zA-Z0-9] │ Any alphanumeric character                    │    │
  │  │ \d          │ Any digit (PCRE only, = [0-9])                 │    │
  │  │ \w          │ Word char (PCRE only, = [a-zA-Z0-9_])         │    │
  │  │ \s          │ Whitespace (PCRE only, = [ \t\n\r\f])         │    │
  │  └─────────────┴────────────────────────────────────────────────┘    │
  │                                                                      │
  │  QUANTIFIERS — HOW MANY to match                                     │
  │  ┌─────────┬───────────────────────────────────────────────────┐     │
  │  │ *       │ Zero or more          a* = "", "a", "aaa"        │     │
  │  │ +       │ One or more           a+ = "a", "aaa" (not "")   │     │
  │  │ ?       │ Zero or one           colou?r = "color", "colour" │     │
  │  │ {n}     │ Exactly n             a{3} = "aaa"               │     │
  │  │ {n,}    │ n or more             a{2,} = "aa", "aaa", ...   │     │
  │  │ {n,m}   │ Between n and m       a{2,4} = "aa","aaa","aaaa" │     │
  │  └─────────┴───────────────────────────────────────────────────┘     │
  │                                                                      │
  │  GROUPING & ALTERNATION                                              │
  │  ┌─────────┬───────────────────────────────────────────────────┐     │
  │  │ (...)   │ Group: (error|warn) matches "error" or "warn"    │     │
  │  │ |       │ OR: cat|dog matches "cat" or "dog"               │     │
  │  │ \1      │ Backreference: matches same text as group 1      │     │
  │  └─────────┴───────────────────────────────────────────────────┘     │
  │                                                                      │
  │  REMEMBER: In BRE (plain grep), you must escape: + ? | ( ) { }      │
  │           In ERE (grep -E), these work directly without escaping.    │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# REGEX EXAMPLES — from simple to complex:

# Match IP address (approximate):
grep -E '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' /var/log/auth.log

# Match email address (simple):
grep -E '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}' contacts.txt

# Match date format YYYY-MM-DD:
grep -E '[0-9]{4}-[0-9]{2}-[0-9]{2}' logfile.txt

# Match lines that start with # (comments):
grep '^#' /etc/nginx/nginx.conf

# Match empty lines:
grep '^$' file.txt

# Match lines that are NOT empty and NOT comments:
grep -v -E '^$|^#|^;' /etc/config.ini

# Find duplicate consecutive words:
grep -P '\b(\w+)\s+\1\b' document.txt
# Matches: "the the", "is is", etc.

# Extract quoted strings:
grep -oP '"[^"]*"' file.txt
```

### 9.6 Beyond grep — The Text Processing Toolkit

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         LINUX TEXT PROCESSING PIPELINE                                │
  │                                                                      │
  │  ┌──────┬────────────────────────────────────────────────────────┐   │
  │  │ Tool │ Purpose                                                │   │
  │  ├──────┼────────────────────────────────────────────────────────┤   │
  │  │ grep │ FILTER lines matching a pattern                        │   │
  │  │ sed  │ TRANSFORM text (find/replace, delete lines)            │   │
  │  │ awk  │ PROCESS columnar data (split, calculate, format)       │   │
  │  │ cut  │ EXTRACT specific columns/fields                        │   │
  │  │ sort │ SORT lines                                             │   │
  │  │ uniq │ DEDUPLICATE consecutive identical lines                │   │
  │  │ wc   │ COUNT lines, words, characters                         │   │
  │  │ tr   │ TRANSLATE/delete characters                             │   │
  │  │ head │ First N lines                                          │   │
  │  │ tail │ Last N lines (tail -f for live following)               │   │
  │  │ tee  │ Split output to file AND stdout                        │   │
  │  └──────┴────────────────────────────────────────────────────────┘   │
  │                                                                      │
  │  PIPING THEM TOGETHER — THE UNIX PHILOSOPHY:                         │
  │                                                                      │
  │  cat access.log                        "Read the file"               │
  │    | grep ' 500 '                      "Filter 500 errors"           │
  │    | awk '{print $1}'                  "Extract IP column"           │
  │    | sort                              "Sort IPs"                    │
  │    | uniq -c                           "Count unique IPs"            │
  │    | sort -rn                          "Sort by count desc"          │
  │    | head -10                          "Top 10 offenders"            │
  │                                                                      │
  │  This is the STANDARD pattern for log analysis in production.        │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# ──────────────────── sed ESSENTIALS ────────────────────
# sed = Stream EDitor — transforms text line by line

# Find and replace (first occurrence per line):
sed 's/old/new/' file.txt

# Replace ALL occurrences per line (global):
sed 's/old/new/g' file.txt

# Edit file IN PLACE:
sed -i 's/old/new/g' file.txt           # Linux
sed -i '' 's/old/new/g' file.txt        # macOS (needs empty '' for backup)

# Delete lines:
sed '/^#/d' file.txt                     # Delete comment lines
sed '/^$/d' file.txt                     # Delete empty lines
sed '1,5d' file.txt                      # Delete lines 1-5

# Print specific lines:
sed -n '10,20p' file.txt                 # Print lines 10-20

# ──────────────────── awk ESSENTIALS ────────────────────
# awk = pattern scanning and text processing

# Print specific columns (default separator: whitespace):
awk '{print $1}' file.txt               # First column
awk '{print $1, $3}' file.txt           # First and third columns
awk '{print $NF}' file.txt              # Last column
awk '{print $(NF-1)}' file.txt          # Second to last column

# Custom separator:
awk -F: '{print $1, $7}' /etc/passwd    # Username and shell
awk -F, '{print $2}' data.csv           # Second column of CSV

# Filter + process:
awk '$3 > 100 {print $1, $3}' data.txt  # Lines where col 3 > 100
awk '/error/ {count++} END {print count}' log.txt  # Count error lines

# Sum a column:
awk '{sum += $5} END {print sum}' data.txt

# ──────────────────── cut, sort, uniq ────────────────────
# cut — extract columns by delimiter:
cut -d: -f1 /etc/passwd                 # First field, colon-delimited
cut -d' ' -f1,3 file.txt               # Fields 1 and 3, space-delimited
cut -c1-10 file.txt                     # Characters 1-10

# sort:
sort file.txt                           # Alphabetical
sort -n file.txt                        # Numeric
sort -rn file.txt                       # Reverse numeric (largest first)
sort -t: -k3 -n /etc/passwd            # Sort by 3rd field (UID), numeric
sort -u file.txt                        # Sort + deduplicate

# uniq (MUST sort first — only removes CONSECUTIVE duplicates):
sort file.txt | uniq                    # Remove duplicates
sort file.txt | uniq -c                 # Count occurrences
sort file.txt | uniq -d                 # Show ONLY duplicates

# wc:
wc -l file.txt                         # Line count
wc -w file.txt                         # Word count
wc -c file.txt                         # Byte count
```

> **DevOps Pro Tip:** The `| sort | uniq -c | sort -rn | head` pipeline is the single most useful log analysis pattern. It answers "What are the most common X?" for any column — most common IPs, most common HTTP status codes, most common error messages, most frequent URLs. Memorize this pipeline.

---

## 10. Umask & Disk Usage — Permission Defaults + du vs df

### 10.1 Umask — Why New Files Get Specific Permissions

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         UMASK — PERMISSION DEFAULT MASK                              │
  │                                                                      │
  │  When you create a file or directory, what permissions does it get?  │
  │  The answer: base permissions MINUS umask.                           │
  │                                                                      │
  │  BASE PERMISSIONS (hardcoded in the system):                         │
  │  • Files:       666 (rw-rw-rw-)  — never get execute by default    │
  │  • Directories: 777 (rwxrwxrwx)  — need execute to cd/list         │
  │                                                                      │
  │  UMASK CALCULATION:                                                  │
  │  Final permissions = Base - Umask (conceptually)                     │
  │  Technical: Final = Base AND (NOT Umask)  (bitwise)                  │
  │                                                                      │
  │  EXAMPLE with umask 022:                                             │
  │                                                                      │
  │  File:                                                               │
  │  ┌──────────┬──────────┬──────────────────┬───────────────┐          │
  │  │ Base     │ Umask    │ Calculation      │ Result        │          │
  │  │ 666      │ 022      │ 666 - 022        │ 644           │          │
  │  │ rw-rw-rw-│ ----w--w-│                  │ rw-r--r--     │          │
  │  └──────────┴──────────┴──────────────────┴───────────────┘          │
  │  → Owner: read+write, Group: read, Others: read                     │
  │                                                                      │
  │  Directory:                                                          │
  │  ┌──────────┬──────────┬──────────────────┬───────────────┐          │
  │  │ Base     │ Umask    │ Calculation      │ Result        │          │
  │  │ 777      │ 022      │ 777 - 022        │ 755           │          │
  │  │ rwxrwxrwx│ ----w--w-│                  │ rwxr-xr-x     │          │
  │  └──────────┴──────────┴──────────────────┴───────────────┘          │
  │  → Owner: all, Group: read+execute, Others: read+execute            │
  │                                                                      │
  │  COMMON UMASK VALUES:                                                │
  │  ┌───────┬───────────────┬───────────────┬──────────────────────┐    │
  │  │ Umask │ File result   │ Dir result    │ Use case             │    │
  │  ├───────┼───────────────┼───────────────┼──────────────────────┤    │
  │  │ 022   │ 644 (rw-r--r-)│ 755 (rwxr-xr-x)│ Default (most systems)│ │
  │  │ 002   │ 664 (rw-rw-r-)│ 775 (rwxrwxr-x)│ Group-collaborative   │ │
  │  │ 027   │ 640 (rw-r----)│ 750 (rwxr-x---)│ Restrictive           │ │
  │  │ 077   │ 600 (rw-------)│ 700 (rwx------)│ Private (secure)     │ │
  │  │ 000   │ 666 (rw-rw-rw-)│ 777 (rwxrwxrwx)│ Wide open (danger!) │ │
  │  └───────┴───────────────┴───────────────┴──────────────────────┘    │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# Check current umask:
umask            # Octal: 0022
umask -S         # Symbolic: u=rwx,g=rx,o=rx

# Set umask for current session:
umask 027        # Files: 640, Directories: 750

# Verify:
touch testfile && mkdir testdir
ls -la testfile testdir
# -rw-r----- 1 user user    0 Mar  3 10:00 testfile   (640)
# drwxr-x--- 2 user user 4096 Mar  3 10:00 testdir    (750)

# Set umask permanently:
echo "umask 027" >> ~/.bashrc      # For one user
# Or: edit /etc/login.defs          # For all users (UMASK setting)
# Or: edit /etc/profile             # System-wide for login shells
```

> **⚠️ Gotcha (Interview Trap):** "Umask 022 — new file permissions?" Many people say 755. WRONG. Files get 644 (base is 666 for files, NOT 777). Only directories get 755 (base 777). Files NEVER get execute permission from default creation — you must `chmod +x` explicitly. This is a security feature.

### 10.2 Linux Permission System — Quick Recap

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         PERMISSION BITS DECODED                                      │
  │                                                                      │
  │  -rwxr-xr-- 1 user group 4096 Mar 3 10:00 file.txt                 │
  │  │└┬┘└┬┘└┬┘                                                         │
  │  │ │  │  └── Others: r-- (4) read only                              │
  │  │ │  └───── Group:  r-x (5) read + execute                        │
  │  │ └──────── Owner:  rwx (7) read + write + execute                 │
  │  └────────── Type:   - = file, d = dir, l = symlink                 │
  │                                                                      │
  │  NUMERIC:  r=4  w=2  x=1                                            │
  │  rwx = 4+2+1 = 7    r-x = 4+0+1 = 5    r-- = 4+0+0 = 4            │
  │  So: rwxr-xr-- = 754                                                │
  │                                                                      │
  │  SPECIAL BITS (beyond rwx):                                          │
  │  ┌──────────┬─────────┬──────────────────────────────────────────┐   │
  │  │ Bit      │ Octal   │ Effect                                   │   │
  │  ├──────────┼─────────┼──────────────────────────────────────────┤   │
  │  │ SetUID   │ 4xxx    │ File runs as file OWNER (not runner)     │   │
  │  │ (s on u) │         │ e.g., /usr/bin/passwd runs as root       │   │
  │  ├──────────┼─────────┼──────────────────────────────────────────┤   │
  │  │ SetGID   │ 2xxx    │ File runs as file GROUP                  │   │
  │  │ (s on g) │         │ Dir: new files inherit dir's group       │   │
  │  ├──────────┼─────────┼──────────────────────────────────────────┤   │
  │  │ Sticky   │ 1xxx    │ Dir: only file OWNER can delete files    │   │
  │  │ (t on o) │         │ e.g., /tmp has sticky bit (drwxrwxrwt)  │   │
  │  └──────────┴─────────┴──────────────────────────────────────────┘   │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# Set permissions:
chmod 755 script.sh           # Owner: rwx, Group: r-x, Others: r-x
chmod u+x script.sh           # Add execute for owner
chmod go-w file.txt           # Remove write from group and others
chmod -R 750 /app/            # Recursive

# Set special bits:
chmod 4755 program            # SetUID + rwxr-xr-x
chmod 2775 /shared/           # SetGID on directory
chmod 1777 /tmp/              # Sticky bit

# Change ownership:
chown user:group file.txt     # Change owner AND group
chown -R www-data:www-data /var/www/   # Recursive

# Find files with SetUID (security audit):
find / -perm -4000 -type f 2>/dev/null
# Lists all SetUID binaries — potential privilege escalation vectors
```

### 10.3 du vs df — The Two Disk Usage Tools (and Why They Disagree)

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         du vs df — DIFFERENT QUESTIONS, DIFFERENT ANSWERS             │
  │                                                                      │
  │  df (disk free)                    du (disk usage)                   │
  │  ┌──────────────────────┐          ┌──────────────────────┐         │
  │  │ "How much space is   │          │ "How much space do   │         │
  │  │  available on the    │          │  these files/dirs    │         │
  │  │  FILESYSTEM?"        │          │  actually use?"      │         │
  │  │                      │          │                      │         │
  │  │ Reads SUPERBLOCK     │          │ Walks the DIRECTORY  │         │
  │  │ metadata (instant)   │          │ tree, stats each     │         │
  │  │                      │          │ file (can be slow)   │         │
  │  │ Shows: total, used,  │          │                      │         │
  │  │ available, mount pt  │          │ Shows: size of each  │         │
  │  │                      │          │ dir/file subtree     │         │
  │  │ Counts ALLOCATED     │          │                      │         │
  │  │ blocks (including    │          │ Counts blocks used   │         │
  │  │ deleted-but-open     │          │ by VISIBLE files     │         │
  │  │ files!)              │          │ only                 │         │
  │  └──────────────────────┘          └──────────────────────┘         │
  │                                                                      │
  │  WHY df AND du CAN DISAGREE:                                         │
  │                                                                      │
  │  Scenario: A 10GB log file is deleted (rm) but nginx still has      │
  │  it open. The blocks are still ALLOCATED (df counts them) but        │
  │  no directory entry exists (du can't see it).                        │
  │                                                                      │
  │  df says: 90% used (10GB still allocated)                            │
  │  du says: only 80% accounted for (can't see deleted-but-open file)  │
  │                                                                      │
  │  The "missing" 10GB = deleted files with open file descriptors.      │
  │  Fix: find them with lsof +L1, restart the holding process.          │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# ──────────────────── df ESSENTIALS ────────────────────
df -h                           # Human-readable (GB, MB)
df -h /                         # Specific filesystem
df -i                           # Inode usage (see §6.5)
df -Th                          # Show filesystem TYPE (ext4, xfs, tmpfs)

# OUTPUT:
# Filesystem     Type  Size  Used Avail Use% Mounted on
# /dev/sda1      ext4   50G   32G   16G  67% /
# tmpfs          tmpfs  3.9G  280M  3.6G   8% /dev/shm
# /dev/sdb1      xfs   100G   45G   55G  45% /data

# ──────────────────── du ESSENTIALS ────────────────────
du -sh /var/log                 # Total size of /var/log (summary, human)
du -sh /var/log/*               # Size of each item in /var/log
du -sh /* 2>/dev/null           # Size of each top-level directory

# Top 10 largest directories:
du -sh /* 2>/dev/null | sort -rh | head -10

# Top 10 largest files in a directory tree:
find /var -type f -exec du -h {} + 2>/dev/null | sort -rh | head -10

# Specific depth:
du -h --max-depth=1 /var/      # One level deep

# Exclude patterns:
du -sh --exclude='*.log' /var/

# ──────────────────── FINDING WHAT'S USING SPACE ────────────────────
# Step 1: Which filesystem is full?
df -h

# Step 2: What's using space there?
du -sh /var/* | sort -rh | head
du -sh /var/log/* | sort -rh | head
# Keep drilling down...

# Step 3: Check for deleted-but-open files (df/du mismatch):
sudo lsof +L1 | grep deleted
# Restart offending process to reclaim space

# Step 4: Check for large individual files:
find / -xdev -type f -size +100M -exec ls -lh {} \; 2>/dev/null | sort -k5 -rh

# ncdu — interactive disk usage viewer (install if available):
sudo apt install ncdu
ncdu /var                       # Interactive, navigate with arrows, delete with 'd'
```

### 10.4 The 5% Reserved Block Mystery

```bash
# On ext4 filesystems, 5% of space is reserved for root by default.
# This is why df might show 95% used but say "no space" for regular users.

# View reserved blocks:
sudo tune2fs -l /dev/sda1 | grep -i reserved
# Reserved block count:     655360
# Reserved blocks uid:      0 (user root)

# Reduce reserved space (production servers often set to 1%):
sudo tune2fs -m 1 /dev/sda1    # Set to 1% reserved
# or:
sudo tune2fs -m 0 /dev/sda1    # No reserved space (not recommended for root FS)

# WHY reserved space exists:
# 1. Prevents regular users from filling disk 100%
# 2. Ensures root can still log in and fix issues when disk is "full"
# 3. Reduces filesystem fragmentation
# 4. Critical system processes (logging, cron) can still write

# On data-only partitions (/data, /backup), you can safely set -m 0
# On root filesystem (/), keep at least 1-2% reserved
```

> **DevOps Pro Tip:** When `df` shows a filesystem at 100% but users report they can't write files, check three things in order: (1) `df -h` — is it actually full? (2) `df -i` — is it inode exhaustion? (3) `lsof +L1` — are deleted files holding space? These three checks resolve 99% of "disk full" tickets.

---

*Batch 3 complete — Phases 5 & 6 (Process Management + Text Processing & Permissions)*

---

## 11. Shell Scripting & Automation — Scripts, Log Rotation, Systemd Services, Cron

### 11.1 Why Shell Scripting Matters for DevOps

Shell scripts are the **glue** of DevOps. You'll write them for deployment, monitoring, backups, log rotation, health checks, and CI/CD pipelines. Interviewers expect you to read, write, and debug shell scripts fluently.

### 11.2 The Anatomy of a Shell Script

```bash
#!/bin/bash
# ↑ SHEBANG — tells the kernel which interpreter to use
#   /bin/bash = Bash shell
#   /bin/sh   = POSIX shell (more portable, fewer features)
#   /usr/bin/env bash = find bash in $PATH (most portable)

set -euo pipefail
# ↑ THE SAFETY NET — always use this in production scripts
#
# set -e          = Exit immediately if any command fails (non-zero exit)
# set -u          = Treat unset variables as errors (prevents rm -rf $UNSET/)
# set -o pipefail = Pipe fails if ANY command in pipe fails
#                   (not just the last one)

# Without set -euo pipefail:
# grep "pattern" file | wc -l
# If grep fails (no match, exit 1), wc still runs and returns 0.
# Script continues as if nothing went wrong. DANGEROUS.

# With set -o pipefail:
# Pipe exit code = exit code of rightmost FAILED command.
# Script exits immediately on the grep failure. SAFE.
```

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         SHELL SCRIPT STRUCTURE — PRODUCTION TEMPLATE                 │
  │                                                                      │
  │  #!/bin/bash                                                         │
  │  set -euo pipefail                    ← Safety net                   │
  │                                                                      │
  │  #─────────── CONSTANTS ───────────                                  │
  │  readonly SCRIPT_NAME="$(basename "$0")"                             │
  │  readonly LOG_DIR="/var/log/myapp"                                   │
  │  readonly BACKUP_DIR="/backup/$(date +%Y%m%d)"                      │
  │                                                                      │
  │  #─────────── FUNCTIONS ───────────                                  │
  │  log() {                                                             │
  │      echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$SCRIPT_NAME] $*"       │
  │  }                                                                   │
  │                                                                      │
  │  die() {                                                             │
  │      log "ERROR: $*" >&2                                             │
  │      exit 1                                                          │
  │  }                                                                   │
  │                                                                      │
  │  cleanup() {                                                         │
  │      log "Cleaning up temporary files..."                            │
  │      rm -rf "$TEMP_DIR"                                              │
  │  }                                                                   │
  │  trap cleanup EXIT          ← Runs cleanup on exit (normal or error) │
  │                                                                      │
  │  #─────────── VALIDATION ───────────                                 │
  │  [[ $EUID -eq 0 ]] || die "Must run as root"                        │
  │  [[ -d "$LOG_DIR" ]] || mkdir -p "$LOG_DIR"                          │
  │  command -v curl &>/dev/null || die "curl not found"                 │
  │                                                                      │
  │  #─────────── MAIN LOGIC ───────────                                 │
  │  log "Starting backup..."                                            │
  │  # ... actual work here ...                                          │
  │  log "Backup complete."                                              │
  └──────────────────────────────────────────────────────────────────────┘
```

### 11.3 Variables, Quoting, and Parameter Expansion

```bash
# ──────────────────── VARIABLE BASICS ────────────────────
name="world"                   # No spaces around =
echo "Hello $name"             # Double quotes: variables expanded
echo 'Hello $name'             # Single quotes: literal (no expansion)
echo "Hello ${name}!"          # Braces for clarity/concatenation

# ──────────────────── QUOTING RULES ────────────────────
# ALWAYS double-quote variables. ALWAYS.
file="my document.txt"
rm $file                       # ❌ Bash sees: rm my document.txt (2 args!)
rm "$file"                     # ✅ Bash sees: rm "my document.txt" (1 arg)

# ──────────────────── PARAMETER EXPANSION ────────────────────
# These are INTERVIEW FAVOURITES:
${var:-default}      # Use "default" if var is unset or empty
${var:=default}      # Set var to "default" if unset, then use it
${var:+alternate}    # Use "alternate" if var IS set (opposite of :-)
${var:?error msg}    # Print error and exit if var is unset

# String manipulation:
filename="backup_2026-03-03.tar.gz"
${filename%.tar.gz}            # Remove shortest match from end → "backup_2026-03-03"
${filename%.*}                 # Remove shortest suffix → "backup_2026-03-03.tar"
${filename%%.*}                # Remove longest suffix  → "backup_2026-03-03"
${filename#*.}                 # Remove shortest prefix → "tar.gz"
${filename##*.}                # Remove longest prefix  → "gz" (get extension)

# Length:
${#filename}                   # String length → 27

# Substring:
${filename:0:6}                # First 6 chars → "backup"
${filename:7:10}               # 10 chars starting at pos 7 → "2026-03-03"

# Replacement:
${filename/backup/restore}     # Replace first match → "restore_2026-03-03.tar.gz"
${filename//0/X}               # Replace ALL matches → "backup_2X26-X3-X3.tar.gz"
```

### 11.4 Conditionals and Loops

```bash
# ──────────────────── IF STATEMENTS ────────────────────
# Modern bash: use [[ ]] (not [ ])
# [[ ]] is a bash keyword — handles spaces, regex, &&/||

if [[ -f "/etc/nginx/nginx.conf" ]]; then
    echo "Nginx config exists"
elif [[ -d "/etc/apache2" ]]; then
    echo "Apache directory exists"
else
    echo "No web server config found"
fi

# FILE TEST OPERATORS:
# -f FILE    = true if FILE exists and is a regular file
# -d DIR     = true if DIR exists and is a directory
# -e PATH    = true if PATH exists (any type)
# -r FILE    = true if FILE is readable
# -w FILE    = true if FILE is writable
# -x FILE    = true if FILE is executable
# -s FILE    = true if FILE exists and is NOT empty
# -L FILE    = true if FILE is a symbolic link
# ! -f FILE  = true if FILE does NOT exist

# STRING COMPARISON (use [[ ]] not [ ]):
[[ "$str" == "hello" ]]     # Equal
[[ "$str" != "hello" ]]     # Not equal
[[ -z "$str" ]]             # True if empty/unset
[[ -n "$str" ]]             # True if not empty
[[ "$str" =~ ^[0-9]+$ ]]   # Regex match (numbers only)

# NUMERIC COMPARISON:
[[ "$num" -eq 5 ]]          # Equal
[[ "$num" -ne 5 ]]          # Not equal
[[ "$num" -gt 5 ]]          # Greater than
[[ "$num" -lt 5 ]]          # Less than
[[ "$num" -ge 5 ]]          # Greater or equal
[[ "$num" -le 5 ]]          # Less or equal
# Or use (( )) for arithmetic:
(( num > 5 )) && echo "Greater"

# ──────────────────── LOOPS ────────────────────
# For loop — iterate over list:
for server in web1 web2 web3 db1; do
    echo "Checking $server..."
    ssh "$server" uptime
done

# For loop — iterate over files:
for f in /var/log/*.log; do
    [[ -f "$f" ]] || continue    # Skip if glob didn't match
    echo "Processing $f..."
    gzip "$f"
done

# For loop — C-style:
for ((i=1; i<=10; i++)); do
    echo "Iteration $i"
done

# While loop — read lines from file:
while IFS= read -r line; do
    echo "Line: $line"
done < /etc/hosts
# IFS=   = don't trim whitespace
# -r     = don't interpret backslashes

# While loop — infinite (daemon-style):
while true; do
    check_health || alert
    sleep 60
done

# Until loop — run until condition is true:
until ping -c1 google.com &>/dev/null; do
    echo "Waiting for network..."
    sleep 5
done
echo "Network is up!"
```

### 11.5 Functions, Exit Codes, and Error Handling

```bash
# ──────────────────── FUNCTIONS ────────────────────
# Define:
check_disk() {
    local threshold="${1:-90}"    # First arg, default 90
    local usage
    usage=$(df / | awk 'NR==2 {print $5}' | tr -d '%')

    if (( usage > threshold )); then
        echo "ALERT: Disk usage at ${usage}% (threshold: ${threshold}%)"
        return 1
    fi
    echo "OK: Disk usage at ${usage}%"
    return 0
}

# Call:
check_disk 80     # Pass 80 as threshold
echo $?           # Exit code: 0=OK, 1=over threshold

# ──────────────────── EXIT CODES ────────────────────
# 0   = success
# 1   = general error
# 2   = misuse of shell command
# 126 = command found but not executable
# 127 = command not found
# 128+N = killed by signal N (e.g., 137 = 128+9 = SIGKILL)

# ──────────────────── ERROR HANDLING PATTERNS ────────────────────
# Pattern 1: || operator (do something on failure)
mkdir /backup/today || { echo "Failed to create backup dir"; exit 1; }

# Pattern 2: trap for cleanup
trap 'echo "Error on line $LINENO"; exit 1' ERR

# Pattern 3: Custom error handler
handle_error() {
    local line=$1
    local command=$2
    echo "ERROR: Command '$command' failed at line $line" >&2
    # Send alert, cleanup, etc.
}
trap 'handle_error $LINENO "$BASH_COMMAND"' ERR
```

> **⚠️ Gotcha:** `set -e` has surprising behavior with conditionals. `if mycommand; then ...` will NOT trigger `set -e` if `mycommand` fails (because it's being tested). Same with `mycommand || handle_error` — the `||` suppresses `set -e`. This means you can accidentally swallow errors. Always test critical commands explicitly with `$?` when using complex error handling.

### 11.6 Log Rotation — logrotate

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         LOG ROTATION — WHY AND HOW                                   │
  │                                                                      │
  │  WITHOUT log rotation:                                               │
  │  /var/log/nginx/access.log grows FOREVER until disk is full.        │
  │  A 50GB log file is impossible to grep, open, or transfer.          │
  │                                                                      │
  │  WITH log rotation:                                                  │
  │  access.log → access.log.1 → access.log.2.gz → ... → deleted       │
  │  Each log file stays small. Old logs compressed. Ancient logs gone.  │
  │                                                                      │
  │  HOW logrotate WORKS:                                                │
  │                                                                      │
  │  1. cron runs logrotate daily (/etc/cron.daily/logrotate)           │
  │  2. logrotate reads /etc/logrotate.conf + /etc/logrotate.d/*        │
  │  3. For each configured log:                                         │
  │     a. Check if rotation needed (size/time threshold)               │
  │     b. Rename current log → .1, .1 → .2, etc.                      │
  │     c. Create new empty log file                                     │
  │     d. Compress old logs (.gz)                                       │
  │     e. Delete logs beyond "rotate N" count                           │
  │     f. Run postrotate commands (signal the service!)                 │
  │                                                                      │
  │  CRITICAL: Step (f) — After rotation, the service still has the     │
  │  OLD file open via its fd. You MUST signal it to reopen logs!       │
  │  Otherwise: service writes to deleted old file, new file stays       │
  │  empty, disk space never freed. (See §7.4 — rm and open fds)       │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# ──────────────────── logrotate CONFIG ────────────────────
# /etc/logrotate.d/nginx  (one file per application)

/var/log/nginx/*.log {
    daily                    # Rotate daily (or: weekly, monthly)
    missingok                # Don't error if log file is missing
    rotate 14                # Keep 14 rotated files (2 weeks)
    compress                 # Compress rotated files with gzip
    delaycompress            # Don't compress yesterday's (still being read)
    notifempty               # Don't rotate if file is empty
    create 0640 www-data adm # Create new log with these permissions
    sharedscripts            # Run postrotate ONCE (not per file)
    postrotate
        # Signal nginx to reopen log files:
        if [ -f /var/run/nginx.pid ]; then
            kill -USR1 $(cat /var/run/nginx.pid)
        fi
    endscript
}

# ──────────────────── logrotate COMMANDS ────────────────────
# Test configuration (dry run — shows what WOULD happen):
sudo logrotate -d /etc/logrotate.d/nginx

# Force rotation now (useful for testing):
sudo logrotate -f /etc/logrotate.d/nginx

# Check status (when was each log last rotated):
cat /var/lib/logrotate/status

# ──────────────────── CUSTOM LOG ROTATION ────────────────────
# For your own application logs:
# Create /etc/logrotate.d/myapp:
/var/log/myapp/*.log {
    daily
    rotate 7
    compress
    missingok
    notifempty
    copytruncate            # ← ALTERNATIVE to postrotate signal
                            # Copies log, then truncates original
                            # Process keeps writing to same fd
                            # ⚠️ Risk: lines written during copy are lost
    # OR use postrotate:
    # postrotate
    #     systemctl reload myapp
    # endscript
}
```

> **DevOps Pro Tip:** `copytruncate` vs `postrotate` signal — use `copytruncate` for apps that can't reopen logs (no signal handler). Use `postrotate` + signal for apps that support it (nginx, apache, syslog). `copytruncate` has a tiny data-loss window (lines written during the copy), while the signal approach is lossless.

### 11.7 Systemd Service Files — Making Your App a Proper Service

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         SYSTEMD UNIT FILE ANATOMY                                    │
  │                                                                      │
  │  Location: /etc/systemd/system/myapp.service                         │
  │  (Custom services go here. /lib/systemd/system/ is for packages.)   │
  │                                                                      │
  │  ┌──────────────────────────────────────────────────────────┐        │
  │  │  [Unit]             ← Metadata & dependencies           │        │
  │  │  Description=       ← Human-readable description        │        │
  │  │  After=             ← Start AFTER these units           │        │
  │  │  Wants=             ← Start these too (soft dependency) │        │
  │  │  Requires=          ← MUST have these (hard dependency) │        │
  │  ├──────────────────────────────────────────────────────────┤        │
  │  │  [Service]          ← How to run the service            │        │
  │  │  Type=              ← simple/forking/oneshot/notify     │        │
  │  │  User=              ← Run as this user                  │        │
  │  │  ExecStart=         ← The command to start              │        │
  │  │  ExecStop=          ← How to stop (optional)            │        │
  │  │  Restart=           ← When to auto-restart              │        │
  │  │  RestartSec=        ← Delay between restarts            │        │
  │  │  Environment=       ← Environment variables             │        │
  │  │  WorkingDirectory=  ← cd here before starting           │        │
  │  ├──────────────────────────────────────────────────────────┤        │
  │  │  [Install]          ← When/how to enable                │        │
  │  │  WantedBy=          ← Which target pulls this in        │        │
  │  └──────────────────────────────────────────────────────────┘        │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# ──────────────────── EXAMPLE: Node.js App Service ────────────────────
cat > /etc/systemd/system/myapp.service << 'EOF'
[Unit]
Description=My Node.js Application
Documentation=https://github.com/myteam/myapp
After=network.target
Wants=network-online.target

[Service]
Type=simple
User=appuser
Group=appuser
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/node /opt/myapp/server.js
ExecStop=/bin/kill -SIGTERM $MAINPID

# Restart policy:
Restart=on-failure          # Restart only on non-zero exit
# Restart=always            # Restart even on clean exit (0)
RestartSec=5                # Wait 5 seconds before restart
StartLimitIntervalSec=300   # Reset restart counter every 5 min
StartLimitBurst=5           # Max 5 restarts in that interval

# Security hardening:
NoNewPrivileges=yes         # Can't gain new privileges (e.g., setuid)
ProtectSystem=strict        # Read-only filesystem except allowed paths
ReadWritePaths=/var/log/myapp /var/lib/myapp
ProtectHome=yes             # Can't access /home
PrivateTmp=yes              # Isolated /tmp

# Environment:
Environment=NODE_ENV=production
Environment=PORT=3000
# Or use file:
EnvironmentFile=/etc/myapp/env

# Resource limits:
LimitNOFILE=65535           # Max open files
MemoryMax=512M              # Kill if exceeds 512MB RAM
CPUQuota=80%                # Max 80% of one CPU core

# Logging:
StandardOutput=journal      # stdout → journald
StandardError=journal       # stderr → journald
SyslogIdentifier=myapp      # Tag in journal

[Install]
WantedBy=multi-user.target  # Start at multi-user (server) boot stage
EOF

# ──────────────────── REGISTER & MANAGE ────────────────────
sudo systemctl daemon-reload           # Re-read unit files (MUST after edit)
sudo systemctl enable myapp            # Start at boot
sudo systemctl start myapp             # Start now
sudo systemctl status myapp            # Check status
sudo systemctl restart myapp           # Stop + Start
sudo systemctl reload myapp            # Reload config (if app supports)
sudo systemctl stop myapp              # Stop
sudo systemctl disable myapp           # Don't start at boot
```

### 11.8 Service Type — Understanding the `Type=` Directive

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         SYSTEMD SERVICE TYPES                                        │
  │                                                                      │
  │  ┌──────────┬────────────────────────────────────────────────────┐   │
  │  │ Type     │ Behavior                                           │   │
  │  ├──────────┼────────────────────────────────────────────────────┤   │
  │  │ simple   │ ExecStart IS the main process. systemd considers   │   │
  │  │ (default)│ service "started" immediately. Most apps use this. │   │
  │  │          │ e.g., Node.js, Python, Go apps                     │   │
  │  ├──────────┼────────────────────────────────────────────────────┤   │
  │  │ forking  │ ExecStart forks a child and exits. The CHILD is    │   │
  │  │          │ the main process. Traditional daemons (apache,     │   │
  │  │          │ older MySQL). Needs PIDFile= to track child.       │   │
  │  ├──────────┼────────────────────────────────────────────────────┤   │
  │  │ oneshot  │ Runs, does its work, exits. systemd waits for      │   │
  │  │          │ completion. Good for init scripts, setup tasks.     │   │
  │  │          │ Often paired with RemainAfterExit=yes.             │   │
  │  ├──────────┼────────────────────────────────────────────────────┤   │
  │  │ notify   │ Like simple, but service sends sd_notify() when    │   │
  │  │          │ ready. More accurate startup detection.             │   │
  │  │          │ e.g., nginx (with notify support)                   │   │
  │  ├──────────┼────────────────────────────────────────────────────┤   │
  │  │ idle     │ Like simple, but delays start until all other      │   │
  │  │          │ jobs finish. Rarely used.                           │   │
  │  └──────────┴────────────────────────────────────────────────────┘   │
  │                                                                      │
  │  CHOOSING THE RIGHT TYPE:                                            │
  │  • App stays in foreground (doesn't daemonize)? → simple             │
  │  • App forks and parent exits?                  → forking + PIDFile  │
  │  • App is a one-time task?                      → oneshot            │
  │  • App signals readiness via sd_notify?         → notify             │
  └──────────────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha:** Using `Type=simple` for a `forking` daemon means systemd tracks the WRONG process (the parent that exits). It thinks the service crashed immediately. Using `Type=forking` for a non-forking app means systemd waits forever for a fork that never happens — the service times out on start. **Wrong Type= is the #1 cause of "service keeps restarting" or "service fails to start."**

### 11.9 Cron — Scheduled Tasks

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         CRONTAB FORMAT                                               │
  │                                                                      │
  │  ┌───────────── minute (0-59)                                        │
  │  │ ┌───────────── hour (0-23)                                        │
  │  │ │ ┌───────────── day of month (1-31)                              │
  │  │ │ │ ┌───────────── month (1-12 or jan-dec)                        │
  │  │ │ │ │ ┌───────────── day of week (0-7, 0 and 7 = Sunday)         │
  │  │ │ │ │ │                                                           │
  │  * * * * * command_to_run                                            │
  │                                                                      │
  │  SPECIAL CHARACTERS:                                                 │
  │  *       = every value                                               │
  │  ,       = list (1,3,5 = 1st, 3rd, 5th)                            │
  │  -       = range (1-5 = 1 through 5)                                │
  │  /       = step (*/15 = every 15 units)                             │
  │                                                                      │
  │  EXAMPLES:                                                           │
  │  ┌──────────────────────┬──────────────────────────────────────┐     │
  │  │ Expression           │ Meaning                              │     │
  │  ├──────────────────────┼──────────────────────────────────────┤     │
  │  │ * * * * *            │ Every minute                         │     │
  │  │ 0 * * * *            │ Every hour (at :00)                  │     │
  │  │ 0 0 * * *            │ Daily at midnight                    │     │
  │  │ 0 0 * * 0            │ Weekly on Sunday at midnight         │     │
  │  │ 0 0 1 * *            │ Monthly on 1st at midnight           │     │
  │  │ */15 * * * *         │ Every 15 minutes                     │     │
  │  │ 0 9-17 * * 1-5       │ Hourly, 9am-5pm, Mon-Fri            │     │
  │  │ 30 2 * * *           │ Daily at 2:30 AM                     │     │
  │  │ 0 0 */2 * *          │ Every other day at midnight          │     │
  │  │ 0 3 1,15 * *         │ 1st and 15th of month at 3 AM       │     │
  │  └──────────────────────┴──────────────────────────────────────┘     │
  │                                                                      │
  │  SHORTHAND (some systems):                                           │
  │  @reboot    = run once at startup                                    │
  │  @hourly    = 0 * * * *                                              │
  │  @daily     = 0 0 * * *                                              │
  │  @weekly    = 0 0 * * 0                                              │
  │  @monthly   = 0 0 1 * *                                              │
  │  @yearly    = 0 0 1 1 *                                              │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# ──────────────────── MANAGING CRONTAB ────────────────────
crontab -e                   # Edit YOUR crontab
crontab -l                   # List YOUR crontab
crontab -r                   # Remove YOUR crontab (⚠️ no confirmation!)
sudo crontab -e -u www-data  # Edit another user's crontab

# System-wide cron files (require username field):
# /etc/crontab              ← System crontab
# /etc/cron.d/*             ← Drop-in cron files
# /etc/cron.daily/*         ← Scripts run daily (by anacron)
# /etc/cron.hourly/*        ← Scripts run hourly
# /etc/cron.weekly/*        ← Scripts run weekly
# /etc/cron.monthly/*       ← Scripts run monthly

# ──────────────────── PRODUCTION CRON PATTERNS ────────────────────
# Backup database nightly at 2 AM:
0 2 * * * /usr/local/bin/backup_db.sh >> /var/log/backup.log 2>&1

# Health check every 5 minutes:
*/5 * * * * /usr/local/bin/health_check.sh >> /var/log/health.log 2>&1

# Clean temp files weekly on Sunday at 3 AM:
0 3 * * 0 find /tmp -type f -atime +7 -delete >> /var/log/cleanup.log 2>&1

# Certificate renewal check (Let's Encrypt):
0 0,12 * * * certbot renew --quiet >> /var/log/certbot.log 2>&1
```

### 11.10 Cron Gotchas — Why Your Cron Job Doesn't Work

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         TOP CRON DEBUGGING ISSUES                                    │
  │                                                                      │
  │  1. ENVIRONMENT IS MINIMAL                                           │
  │     Cron runs with a stripped-down environment.                      │
  │     PATH is minimal (just /usr/bin:/bin).                            │
  │     $HOME, $USER set, but most env vars are MISSING.                │
  │                                                                      │
  │     FIX: Use full paths in cron commands:                            │
  │     ❌ 0 * * * * node /app/server.js                                │
  │     ✅ 0 * * * * /usr/bin/node /app/server.js                       │
  │     Or set PATH at top of crontab:                                   │
  │     PATH=/usr/local/bin:/usr/bin:/bin                                │
  │                                                                      │
  │  2. OUTPUT IS SILENTLY MAILED (or discarded)                         │
  │     Cron tries to EMAIL stdout/stderr to the user.                  │
  │     If no mail system → output is silently lost.                     │
  │                                                                      │
  │     FIX: Always redirect output:                                     │
  │     0 2 * * * /script.sh >> /var/log/script.log 2>&1                │
  │                                                                      │
  │  3. SCRIPT NOT EXECUTABLE                                            │
  │     ❌ Script doesn't have +x permission                            │
  │     ✅ chmod +x /usr/local/bin/backup.sh                            │
  │     Or call interpreter explicitly:                                  │
  │     0 2 * * * /bin/bash /usr/local/bin/backup.sh                    │
  │                                                                      │
  │  4. PERCENT SIGNS (%) ARE SPECIAL                                    │
  │     In crontab, % = newline. Use \% to escape.                      │
  │     ❌ 0 0 * * * echo "$(date +%Y-%m-%d)" >> /log                  │
  │     ✅ 0 0 * * * echo "$(date +\%Y-\%m-\%d)" >> /log              │
  │     Or put the command in a script (better approach).                │
  │                                                                      │
  │  5. TIMEZONE CONFUSION                                               │
  │     Cron uses system timezone. If server is UTC but you              │
  │     expect IST → jobs run 5:30 hours off.                           │
  │     Check: timedatectl, or set TZ in crontab.                       │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# Debug cron issues:
# Check if cron is running:
systemctl status cron           # Ubuntu/Debian
systemctl status crond          # RHEL/CentOS

# Check cron log:
grep CRON /var/log/syslog       # Ubuntu/Debian
grep CRON /var/log/cron         # RHEL/CentOS

# Test your cron command manually — simulate cron's environment:
env -i HOME=$HOME SHELL=/bin/bash PATH=/usr/bin:/bin /your/script.sh
# env -i = start with empty environment (like cron)
```

> **DevOps Pro Tip:** Modern alternative to cron: **systemd timers**. They offer better logging (journalctl), dependency management, resource controls (cgroups), and can run missed tasks after downtime (Persistent=true). For new setups, prefer systemd timers over cron.

### 11.11 systemd Timers — The Modern Cron Replacement

```bash
# ──────────────────── TIMER + SERVICE PAIR ────────────────────
# Timer unit: /etc/systemd/system/backup.timer
cat > /etc/systemd/system/backup.timer << 'EOF'
[Unit]
Description=Run backup daily

[Timer]
OnCalendar=*-*-* 02:00:00       # Daily at 2 AM (YYYY-MM-DD HH:MM:SS)
Persistent=true                  # Run if missed (e.g., server was off)
RandomizedDelaySec=300           # Random 0-5min delay (prevents thundering herd)

[Install]
WantedBy=timers.target
EOF

# Service unit: /etc/systemd/system/backup.service
cat > /etc/systemd/system/backup.service << 'EOF'
[Unit]
Description=Database backup

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup_db.sh
User=backup
EOF

# Enable and start:
sudo systemctl daemon-reload
sudo systemctl enable --now backup.timer

# Check timer status:
systemctl list-timers                   # All timers, next run time
systemctl list-timers --all             # Include inactive
systemctl status backup.timer           # Specific timer
journalctl -u backup.service           # Logs from the service

# OnCalendar syntax examples:
# *-*-* 02:00:00         = daily at 2 AM
# Mon *-*-* 03:00:00     = every Monday at 3 AM
# *-*-01 00:00:00        = first of each month at midnight
# *-*-* *:00/15:00       = every 15 minutes
# *-*-* 09..17:00:00     = every hour from 9 AM to 5 PM

# Test calendar expression:
systemd-analyze calendar "*-*-* 02:00:00"
```

---

## 12. Service Management & Debugging — systemctl, journalctl, Log Locations

### 12.1 systemctl — The Service Control Command

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         systemctl — COMPLETE COMMAND REFERENCE                       │
  │                                                                      │
  │  SERVICE LIFECYCLE:                                                  │
  │  ┌─────────────────────────────────────────────────────────────┐     │
  │  │  systemctl start   <svc>   ← Start the service NOW         │     │
  │  │  systemctl stop    <svc>   ← Stop the service NOW          │     │
  │  │  systemctl restart <svc>   ← Stop + Start                  │     │
  │  │  systemctl reload  <svc>   ← Reload config (no downtime)   │     │
  │  │  systemctl enable  <svc>   ← Start at boot (creates symlink)│    │
  │  │  systemctl disable <svc>   ← Don't start at boot           │     │
  │  │  systemctl status  <svc>   ← Show status, PID, recent logs │     │
  │  └─────────────────────────────────────────────────────────────┘     │
  │                                                                      │
  │  IMPORTANT DISTINCTION:                                              │
  │  start/stop   = affect CURRENT state only                            │
  │  enable/disable = affect BOOT behavior only                          │
  │  They are INDEPENDENT:                                               │
  │  • start + enable = running NOW and at boot ✅                      │
  │  • start + disable = running NOW but NOT at boot                    │
  │  • stop + enable = stopped NOW but WILL start at boot               │
  │  • Shortcut: systemctl enable --now <svc> = enable + start          │
  │                                                                      │
  │  QUERYING:                                                           │
  │  ┌─────────────────────────────────────────────────────────────┐     │
  │  │  systemctl is-active  <svc>     ← "active" or "inactive"   │     │
  │  │  systemctl is-enabled <svc>     ← "enabled" or "disabled"  │     │
  │  │  systemctl is-failed  <svc>     ← "failed" or "active"     │     │
  │  │  systemctl list-units --failed  ← All failed services       │     │
  │  │  systemctl list-units --type=service  ← All services        │     │
  │  │  systemctl list-unit-files      ← All unit files + state    │     │
  │  │  systemctl show <svc>           ← All properties (machine)  │     │
  │  │  systemctl cat <svc>            ← Show unit file contents   │     │
  │  │  systemctl edit <svc>           ← Override unit settings    │     │
  │  │  systemctl list-dependencies <svc> ← Dependency tree        │     │
  │  └─────────────────────────────────────────────────────────────┘     │
  └──────────────────────────────────────────────────────────────────────┘
```

### 12.2 systemctl status — Reading the Output

```bash
$ sudo systemctl status nginx
# ● nginx.service - A high performance web server
#      Loaded: loaded (/lib/systemd/system/nginx.service; enabled; preset: enabled)
#      Active: active (running) since Mon 2026-03-02 10:00:00 IST; 1 day ago
#        Docs: man:nginx(8)
#     Process: 1230 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
#    Main PID: 1234 (nginx)
#       Tasks: 5 (limit: 9354)
#      Memory: 12.5M
#         CPU: 1.234s
#      CGroup: /system.slice/nginx.service
#              ├─1234 "nginx: master process /usr/sbin/nginx"
#              ├─1235 "nginx: worker process"
#              ├─1236 "nginx: worker process"
#              ├─1237 "nginx: worker process"
#              └─1238 "nginx: worker process"
#
# Mar 02 10:00:00 server01 systemd[1]: Starting A high performance web server...
# Mar 02 10:00:00 server01 nginx[1230]: nginx: configuration file /etc/nginx/nginx.conf test is successful
# Mar 02 10:00:00 server01 systemd[1]: Started A high performance web server.

# WHAT TO LOOK FOR:
# ● = active (green dot)    ○ = inactive    ● = failed (red)
# Loaded: enabled = starts at boot
# Active: since when, how long ago
# Main PID: the main process ID
# CGroup: all child processes (workers)
# Bottom lines: recent log entries
```

### 12.3 systemctl edit — Overriding Service Configuration

```bash
# NEVER edit /lib/systemd/system/*.service directly!
# Package updates will overwrite your changes.

# Use drop-in overrides instead:
sudo systemctl edit nginx
# Opens editor to create /etc/systemd/system/nginx.service.d/override.conf

# Example override — increase timeout and add environment:
# [Service]
# TimeoutStartSec=120
# Environment=NGINX_WORKER_CONNECTIONS=4096

# View effective configuration (merged):
systemctl cat nginx

# To override the entire unit file (not just parts):
sudo systemctl edit --full nginx
# Copies to /etc/systemd/system/nginx.service (overrides /lib version)

# After any edit:
sudo systemctl daemon-reload
sudo systemctl restart nginx
```

### 12.4 journalctl — The System Log Viewer

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         journalctl — THE UNIFIED LOG SYSTEM                          │
  │                                                                      │
  │  journald is systemd's logging daemon. It captures:                  │
  │  • stdout/stderr from ALL systemd services                           │
  │  • Kernel messages (like dmesg)                                      │
  │  • Syslog messages                                                   │
  │  • Audit logs                                                        │
  │                                                                      │
  │  All stored in a binary journal (/var/log/journal/).                 │
  │  You READ it with journalctl.                                        │
  │                                                                      │
  │  ADVANTAGE over text logs:                                           │
  │  • Structured data (metadata per entry: PID, unit, priority)        │
  │  • Indexed (fast filtering by time, unit, priority)                 │
  │  • Automatic rotation (based on disk/time limits)                   │
  │  • Cannot be easily tampered with (sealed binary format)            │
  │                                                                      │
  │  DISADVANTAGE:                                                       │
  │  • Binary format (can't grep raw files — must use journalctl)       │
  │  • Takes more CPU than simple text logging                          │
  │  • Not persistent by default on some distros (fix: mkdir below)     │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# ──────────────────── BASIC USAGE ────────────────────
journalctl                         # All logs (HUGE — use filters!)
journalctl -f                      # Follow (like tail -f) — live log stream
journalctl --no-pager              # Don't paginate (for piping)

# ──────────────────── FILTER BY SERVICE ────────────────────
journalctl -u nginx                # All nginx logs
journalctl -u nginx -f             # Follow nginx logs (live)
journalctl -u nginx --since today  # Today's nginx logs
journalctl -u nginx -n 50          # Last 50 lines

# Multiple services:
journalctl -u nginx -u php-fpm    # nginx AND php-fpm logs together

# ──────────────────── FILTER BY TIME ────────────────────
journalctl --since "2026-03-03 00:00:00"
journalctl --since "2 hours ago"
journalctl --since "1 hour ago" --until "30 minutes ago"
journalctl --since today           # Since midnight today
journalctl --since yesterday --until today

# ──────────────────── FILTER BY PRIORITY ────────────────────
# Priorities: emerg(0) alert(1) crit(2) err(3) warning(4) notice(5) info(6) debug(7)
journalctl -p err                  # Only errors and above (emerg, alert, crit, err)
journalctl -p warning              # Warnings and above
journalctl -u nginx -p err         # Only nginx errors

# ──────────────────── FILTER BY BOOT ────────────────────
journalctl -b                      # Current boot
journalctl -b -1                   # Previous boot
journalctl --list-boots             # List all boots (when persistent)

# ──────────────────── OUTPUT FORMATS ────────────────────
journalctl -u nginx -o json-pretty  # JSON format (for parsing)
journalctl -u nginx -o verbose      # All metadata fields
journalctl -u nginx -o short-iso    # ISO timestamps

# ──────────────────── DISK USAGE ────────────────────
journalctl --disk-usage             # How much space journals use
sudo journalctl --vacuum-size=500M  # Shrink to 500MB
sudo journalctl --vacuum-time=7d    # Keep only last 7 days

# Configure in /etc/systemd/journald.conf:
# SystemMaxUse=500M          # Max disk usage
# MaxRetentionSec=1month     # Max age
# Storage=persistent         # Persist across reboots (needs /var/log/journal/)

# Make journal persistent (some distros default to volatile):
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal
sudo systemctl restart systemd-journald
```

### 12.5 Traditional Log Files — Where to Look

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         LINUX LOG FILE LOCATIONS                                      │
  │                                                                      │
  │  ┌────────────────────────────┬───────────────────────────────────┐  │
  │  │ Log File                   │ What It Contains                  │  │
  │  ├────────────────────────────┼───────────────────────────────────┤  │
  │  │ /var/log/syslog            │ General system log (Ubuntu/Debian)│  │
  │  │ /var/log/messages          │ General system log (RHEL/CentOS) │  │
  │  ├────────────────────────────┼───────────────────────────────────┤  │
  │  │ /var/log/auth.log          │ Authentication (SSH, sudo, login)│  │
  │  │ /var/log/secure            │ Same but on RHEL/CentOS          │  │
  │  ├────────────────────────────┼───────────────────────────────────┤  │
  │  │ /var/log/kern.log          │ Kernel messages (hardware, OOM)  │  │
  │  │ /var/log/dmesg             │ Boot-time kernel messages        │  │
  │  ├────────────────────────────┼───────────────────────────────────┤  │
  │  │ /var/log/nginx/            │ Nginx access + error logs        │  │
  │  │   access.log               │   HTTP requests                  │  │
  │  │   error.log                │   Errors and warnings            │  │
  │  ├────────────────────────────┼───────────────────────────────────┤  │
  │  │ /var/log/apache2/          │ Apache logs (Ubuntu)             │  │
  │  │ /var/log/httpd/            │ Apache logs (RHEL)               │  │
  │  ├────────────────────────────┼───────────────────────────────────┤  │
  │  │ /var/log/mysql/            │ MySQL/MariaDB logs               │  │
  │  │ /var/log/postgresql/       │ PostgreSQL logs                  │  │
  │  ├────────────────────────────┼───────────────────────────────────┤  │
  │  │ /var/log/cron              │ Cron job execution log           │  │
  │  │ /var/log/mail.log          │ Mail server logs                 │  │
  │  │ /var/log/boot.log          │ Boot process log                 │  │
  │  │ /var/log/faillog           │ Failed login attempts            │  │
  │  │ /var/log/wtmp              │ Login records (read with last)   │  │
  │  │ /var/log/btmp              │ Bad login records (lastb)        │  │
  │  └────────────────────────────┴───────────────────────────────────┘  │
  │                                                                      │
  │  UBUNTU/DEBIAN vs RHEL/CENTOS — KEY DIFFERENCES:                     │
  │  syslog ↔ messages    auth.log ↔ secure    apt ↔ yum               │
  └──────────────────────────────────────────────────────────────────────┘
```

### 12.6 The Debugging Flowchart — Service Won't Start

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │  SERVICE WON'T START — SYSTEMATIC DEBUGGING FLOW                     │
  │                                                                      │
  │  1. CHECK STATUS                                                     │
  │     $ systemctl status myapp                                         │
  │     Look at: Active line (why it failed), last log lines             │
  │         │                                                            │
  │         ▼                                                            │
  │  2. READ FULL LOGS                                                   │
  │     $ journalctl -u myapp -n 100 --no-pager                         │
  │     $ journalctl -u myapp -b -p err                                  │
  │     Look for: error messages, stack traces, permission denied        │
  │         │                                                            │
  │         ▼                                                            │
  │  3. CHECK CONFIGURATION                                              │
  │     $ systemctl cat myapp                 (is the unit file correct?)│
  │     $ myapp --test-config                 (e.g., nginx -t)          │
  │     $ /path/to/app --check-config         (app-specific syntax test)│
  │         │                                                            │
  │         ▼                                                            │
  │  4. TRY RUNNING MANUALLY                                             │
  │     $ sudo -u appuser /path/to/app        (run as the service user) │
  │     This reveals errors hidden by systemd: missing deps,             │
  │     permission errors, wrong paths, missing env vars.                │
  │         │                                                            │
  │         ▼                                                            │
  │  5. CHECK PERMISSIONS                                                │
  │     $ ls -la /path/to/app                 (binary executable?)      │
  │     $ ls -la /var/log/myapp/              (log dir writable?)       │
  │     $ ls -la /etc/myapp/config.yml        (config readable?)        │
  │     $ namei -l /path/to/app               (check every dir in path)│
  │         │                                                            │
  │         ▼                                                            │
  │  6. CHECK PORT CONFLICTS                                             │
  │     $ ss -tlnp | grep :80                 (port already in use?)    │
  │     $ sudo lsof -i :80                    (what's using port 80?)   │
  │         │                                                            │
  │         ▼                                                            │
  │  7. CHECK RESOURCE LIMITS                                            │
  │     $ cat /proc/$(pgrep myapp)/limits     (file descriptor limits)  │
  │     $ systemctl show myapp | grep Limit   (systemd-imposed limits)  │
  │     $ df -h                               (disk full?)              │
  │     $ free -h                             (memory available?)       │
  │         │                                                            │
  │         ▼                                                            │
  │  8. CHECK SELINUX / APPARMOR                                         │
  │     $ getenforce                          (SELinux enforcing?)      │
  │     $ ausearch -m avc -ts recent          (SELinux denials)         │
  │     $ aa-status                           (AppArmor profiles)       │
  │                                                                      │
  │  COMMON CULPRITS (90% of cases):                                     │
  │  • Wrong ExecStart path or missing binary                            │
  │  • Permission denied (wrong User= or file perms)                    │
  │  • Port already in use (another service on same port)               │
  │  • Config syntax error (always test config before restart)          │
  │  • Missing dependency (After= not set, database not ready)          │
  │  • Wrong Type= in unit file (see §11.8)                             │
  └──────────────────────────────────────────────────────────────────────┘
```

### 12.7 Production Debugging — Real Scenarios

```bash
# ──────────── SCENARIO 1: Service keeps restarting ────────────
systemctl status myapp
# Active: activating (auto-restart) ...
# (service crashes, systemd restarts it, crashes again — loop)

# Diagnosis:
journalctl -u myapp -b --no-pager | tail -50
# Look for the error BEFORE each restart

# Common causes:
# • App crashes on startup (missing config, DB unreachable)
# • Wrong Type= (see §11.8) — systemd thinks it crashed
# • Resource limit hit (OOM killer → exit 137)
journalctl -u myapp | grep -i "killed\|oom\|signal"

# ──────────── SCENARIO 2: "Address already in use" ────────────
journalctl -u nginx -b | tail
# [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)

# Find what's on port 80:
sudo ss -tlnp | grep :80
# LISTEN  0  511  0.0.0.0:80  *  users:(("apache2",pid=5678,...))
# → Apache is running on port 80! Stop it first.
sudo systemctl stop apache2
sudo systemctl start nginx

# ──────────── SCENARIO 3: Permission denied ────────────
journalctl -u myapp -b | tail
# Error: EACCES: permission denied, open '/var/log/myapp/app.log'

# Fix:
sudo mkdir -p /var/log/myapp
sudo chown appuser:appuser /var/log/myapp
sudo systemctl restart myapp

# ──────────── SCENARIO 4: Config syntax error ────────────
sudo systemctl restart nginx
# Job for nginx.service failed...

# Always test config BEFORE restarting:
sudo nginx -t
# nginx: [emerg] unexpected "}" in /etc/nginx/sites-enabled/mysite:42
# Fix the config error, then restart.

# ──────────── SCENARIO 5: Why did it stop? ────────────
# Check exit code:
systemctl show myapp -p ExecMainStatus
# ExecMainStatus=137    → 128+9 → killed by SIGKILL (OOM killer!)

# Check OOM killer:
dmesg | grep -i oom
journalctl -k | grep -i oom
# [  123.456] Out of memory: Killed process 1234 (myapp) score 850
```

### 12.8 dmesg — Kernel Messages

```bash
# dmesg shows kernel ring buffer messages:
# hardware detection, driver loading, OOM kills, disk errors

dmesg                              # All kernel messages
dmesg -T                           # Human-readable timestamps (not boot offset)
dmesg --level=err,warn             # Only errors and warnings
dmesg -T | tail -50                # Recent messages
dmesg -T | grep -i error           # Search for errors
dmesg -w                           # Watch (follow) kernel messages live

# What to look for:
dmesg -T | grep -i "oom"           # Out Of Memory kills
dmesg -T | grep -i "error"         # Hardware/driver errors
dmesg -T | grep -i "fail"          # Failed operations
dmesg -T | grep -iE "sda|nvme"    # Disk messages
dmesg -T | grep -i "usb"           # USB device events
dmesg -T | grep -i "link"          # Network link up/down
```

> **DevOps Pro Tip:** The most critical production debugging skill is systematic elimination: **status → logs → config → manual run → permissions → ports → resources → security**. Never jump to "restart it" before checking logs. Restarting hides the evidence — the crash logs might be gone after a successful restart, especially with `journalctl -b` (current boot only). Save the output first: `journalctl -u myapp -b > /tmp/myapp_crash.log`.

---

*Batch 4 complete — Phases 7 & 8 (Shell Scripting & Automation + Service Management & Debugging)*

---

## 13. Networking Commands — ifconfig, ip, netstat, ss, ping, traceroute, DNS

### 13.1 Why Networking is a Core DevOps Skill

Every production issue is either "the server is down" or "the network is broken." You need to quickly determine: Is the service listening? Can I reach the port? Where does the packet die? Is DNS resolving? These commands answer those questions.

### 13.2 The Network Troubleshooting Mental Model

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         NETWORK DEBUGGING — THE LAYER APPROACH                       │
  │                                                                      │
  │  When "I can't connect" — work BOTTOM UP:                           │
  │                                                                      │
  │  Layer 1: PHYSICAL / LINK                                            │
  │  ┌───────────────────────────────────────────────┐                   │
  │  │ "Is the interface up? Do I have an IP?"       │                   │
  │  │  ip link show    ip addr show                 │                   │
  │  │  ethtool eth0    (link speed, duplex, state)  │                   │
  │  └─────────────────────┬─────────────────────────┘                   │
  │                        ▼                                             │
  │  Layer 2: NETWORK / IP                                               │
  │  ┌───────────────────────────────────────────────┐                   │
  │  │ "Can I reach the gateway? Other hosts?"       │                   │
  │  │  ip route show     ping <gateway>             │                   │
  │  │  ping <target>     traceroute <target>        │                   │
  │  └─────────────────────┬─────────────────────────┘                   │
  │                        ▼                                             │
  │  Layer 3: DNS                                                        │
  │  ┌───────────────────────────────────────────────┐                   │
  │  │ "Does the hostname resolve?"                  │                   │
  │  │  nslookup <host>   dig <host>                 │                   │
  │  │  cat /etc/resolv.conf                         │                   │
  │  └─────────────────────┬─────────────────────────┘                   │
  │                        ▼                                             │
  │  Layer 4: TRANSPORT / PORT                                           │
  │  ┌───────────────────────────────────────────────┐                   │
  │  │ "Is the port open? Is the service listening?" │                   │
  │  │  ss -tlnp     netstat -tlnp                   │                   │
  │  │  telnet <host> <port>   curl <url>            │                   │
  │  └─────────────────────┬─────────────────────────┘                   │
  │                        ▼                                             │
  │  Layer 5: APPLICATION                                                │
  │  ┌───────────────────────────────────────────────┐                   │
  │  │ "Is the app responding correctly?"            │                   │
  │  │  curl -v <url>    wget <url>                  │                   │
  │  │  Check app logs: journalctl -u myapp          │                   │
  │  └───────────────────────────────────────────────┘                   │
  └──────────────────────────────────────────────────────────────────────┘
```

### 13.3 ifconfig vs ip — Interface Configuration

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         ifconfig vs ip — OLD vs NEW                                  │
  │                                                                      │
  │  ifconfig is DEPRECATED (part of net-tools, not installed by         │
  │  default on newer systems). ip is the modern replacement.            │
  │                                                                      │
  │  ┌─────────────────────────────┬───────────────────────────────┐     │
  │  │ ifconfig (legacy)           │ ip (modern)                   │     │
  │  ├─────────────────────────────┼───────────────────────────────┤     │
  │  │ ifconfig                    │ ip addr show (or: ip a)       │     │
  │  │ ifconfig eth0               │ ip addr show dev eth0         │     │
  │  │ ifconfig eth0 up            │ ip link set eth0 up           │     │
  │  │ ifconfig eth0 down          │ ip link set eth0 down         │     │
  │  │ ifconfig eth0 192.168.1.10  │ ip addr add 192.168.1.10/24  │     │
  │  │   netmask 255.255.255.0    │   dev eth0                    │     │
  │  │ route -n                    │ ip route show (or: ip r)      │     │
  │  │ route add default gw        │ ip route add default via      │     │
  │  │   192.168.1.1               │   192.168.1.1                 │     │
  │  │ arp -a                      │ ip neigh show (or: ip n)      │     │
  │  └─────────────────────────────┴───────────────────────────────┘     │
  │                                                                      │
  │  ALWAYS use ip in interviews and scripts. Use ifconfig only          │
  │  if asked about it specifically.                                     │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# ──────────────────── ip COMMAND ESSENTIALS ────────────────────
# Show all interfaces and their IPs:
ip addr show
# or short form:
ip a

# OUTPUT DECODED:
# 2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP
#     link/ether 52:54:00:ab:cd:ef brd ff:ff:ff:ff:ff:ff
#     inet 192.168.1.100/24 brd 192.168.1.255 scope global dynamic eth0
#     inet6 fe80::5054:ff:feab:cdef/64 scope link
#
# Key fields:
# state UP      = interface is active
# LOWER_UP      = physical link detected (cable plugged in)
# inet 192.168.1.100/24 = IPv4 address and subnet mask (/24 = 255.255.255.0)
# link/ether    = MAC address

# Show only IPv4 addresses:
ip -4 addr show

# Show routing table:
ip route show
# default via 192.168.1.1 dev eth0 proto dhcp metric 100
# 192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.100
#
# "default via 192.168.1.1" = default gateway (where packets go if no specific route)

# Add a temporary IP:
sudo ip addr add 10.0.0.50/24 dev eth0

# Add a temporary route:
sudo ip route add 10.10.0.0/16 via 192.168.1.1

# Show link status (up/down):
ip link show

# Show ARP table (IP → MAC mappings):
ip neigh show
```

### 13.4 netstat vs ss — Socket and Connection Monitoring

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         netstat vs ss — OLD vs NEW                                   │
  │                                                                      │
  │  netstat is DEPRECATED (net-tools). ss is the modern replacement.    │
  │  ss is FASTER (reads directly from kernel via netlink).              │
  │                                                                      │
  │  ┌──────────────────────────────┬──────────────────────────────┐     │
  │  │ netstat (legacy)             │ ss (modern)                  │     │
  │  ├──────────────────────────────┼──────────────────────────────┤     │
  │  │ netstat -tlnp                │ ss -tlnp                     │     │
  │  │ netstat -ulnp                │ ss -ulnp                     │     │
  │  │ netstat -an                  │ ss -an                       │     │
  │  │ netstat -rn                  │ ip route show                │     │
  │  │ netstat -i                   │ ip -s link show              │     │
  │  └──────────────────────────────┴──────────────────────────────┘     │
  │                                                                      │
  │  THE FLAGS (same for both):                                          │
  │  -t = TCP connections                                                │
  │  -u = UDP connections                                                │
  │  -l = LISTENING sockets only (waiting for connections)               │
  │  -n = Numeric (don't resolve hostnames — much faster)                │
  │  -p = Show PROCESS using the socket (needs root)                     │
  │  -a = ALL sockets (listening + established)                          │
  │                                                                      │
  │  MOST USED: ss -tlnp = "What TCP ports are services LISTENING on?"  │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# ──────────────────── ss IN ACTION ────────────────────
# What's listening? (THE most common network debugging command)
sudo ss -tlnp
# State  Recv-Q  Send-Q  Local Address:Port  Peer Address:Port  Process
# LISTEN 0       511     0.0.0.0:80          0.0.0.0:*          users:(("nginx",pid=1234,...))
# LISTEN 0       128     0.0.0.0:22          0.0.0.0:*          users:(("sshd",pid=567,...))
# LISTEN 0       511     0.0.0.0:443         0.0.0.0:*          users:(("nginx",pid=1234,...))
# LISTEN 0       128     127.0.0.1:3306      0.0.0.0:*          users:(("mysqld",pid=890,...))
#                        ↑                                       ↑
#                    Address:Port                              What process

# KEY INSIGHT: 0.0.0.0:80 = listening on ALL interfaces
#              127.0.0.1:3306 = listening ONLY on localhost (not reachable from outside!)

# Is a specific port in use?
ss -tlnp | grep :80
ss -tlnp | grep :443

# Show all established connections:
ss -tn                          # All TCP connections (not just listening)
ss -tn state established        # Only established connections

# Count connections per state:
ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn
# Useful for detecting connection buildup (too many TIME_WAIT, CLOSE_WAIT)

# Show connections to a specific port:
ss -tn dst :443                 # Connections TO port 443
ss -tn src :80                  # Connections FROM port 80

# Show per-socket memory usage:
ss -tnm

# Find who's connected to your SSH:
ss -tn state established '( dport = :22 )'
```

### 13.5 ping — Connectivity Testing

```bash
# Basic ping (runs forever — Ctrl+C to stop):
ping 192.168.1.1
ping google.com

# Send specific number of pings:
ping -c 4 google.com          # 4 pings then stop

# OUTPUT:
# PING google.com (142.250.182.46) 56(84) bytes of data.
# 64 bytes from 142.250.182.46: icmp_seq=1 ttl=117 time=12.3 ms
# 64 bytes from 142.250.182.46: icmp_seq=2 ttl=117 time=11.8 ms
#
# --- google.com ping statistics ---
# 4 packets transmitted, 4 received, 0% packet loss, time 3004ms
# rtt min/avg/max/mdev = 11.8/12.1/12.5/0.3 ms

# KEY FIELDS:
# ttl=117     = Time To Live (hops remaining — started at 128 or 64)
# time=12.3ms = Round-trip time (latency)
# 0% loss     = All packets returned (100% loss = host unreachable or blocking ICMP)

# Quick connectivity check (useful in scripts):
ping -c 1 -W 2 192.168.1.1 &>/dev/null && echo "UP" || echo "DOWN"
# -W 2 = 2 second timeout

# Flood ping (stress test — needs root):
sudo ping -f -c 1000 192.168.1.1
# Sends as fast as possible. Shows packet loss under load.
```

> **⚠️ Gotcha:** "ping fails but website works" — this is normal! Many servers and firewalls **block ICMP** (ping). A failed ping does NOT mean the host is down. Always test the actual service: `curl http://host:port` or `telnet host port`. Ping only tests ICMP, not TCP/UDP.

### 13.6 traceroute / tracepath — Path Discovery

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         TRACEROUTE — HOW PACKETS TRAVEL                              │
  │                                                                      │
  │  traceroute shows EVERY router (hop) between you and the target.    │
  │  It works by sending packets with incrementing TTL values:           │
  │                                                                      │
  │  TTL=1 → first router responds "TTL expired" (reveals its IP)       │
  │  TTL=2 → second router responds                                     │
  │  TTL=3 → third router responds                                      │
  │  ...until the packet reaches the destination.                        │
  │                                                                      │
  │  YOUR HOST           HOP 1          HOP 2          DESTINATION      │
  │  ┌────────┐     ┌──────────┐   ┌──────────┐     ┌──────────┐       │
  │  │ You    │────►│ Router 1 │──►│ Router 2 │────►│ Target   │       │
  │  │        │     │ 10.0.0.1 │   │ 72.14.x  │     │ 142.250  │       │
  │  └────────┘     │ 1.2ms    │   │ 15.4ms   │     │ 22.1ms   │       │
  │                  └──────────┘   └──────────┘     └──────────┘       │
  │                                                                      │
  │  * * * = router didn't respond (ICMP blocked, not necessarily down) │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# Standard traceroute (uses UDP by default on Linux):
traceroute google.com

# Use ICMP (like Windows tracert):
sudo traceroute -I google.com

# Use TCP (best for testing — firewalls rarely block TCP):
sudo traceroute -T -p 443 google.com     # TCP to port 443

# OUTPUT:
# traceroute to google.com (142.250.182.46), 30 hops max, 60 byte packets
#  1  gateway (192.168.1.1)     1.234 ms  0.987 ms  0.876 ms
#  2  isp-router (10.200.1.1)  5.432 ms  5.321 ms  5.210 ms
#  3  * * *                                                    ← ICMP blocked
#  4  72.14.236.110             12.345 ms  12.234 ms  12.123 ms
#  5  142.250.182.46            15.678 ms  15.567 ms  15.456 ms
#
# Three times per hop = three probes (showing latency variance)

# tracepath (doesn't need root, discovers MTU):
tracepath google.com

# mtr — combines ping + traceroute (live, continuous):
mtr google.com                  # Interactive display
mtr -rw -c 100 google.com     # Report mode: 100 cycles, wide output
# Shows packet loss PER HOP — pinpoints exactly where packets die.
```

### 13.7 DNS Resolution — How Names Become IPs

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         DNS RESOLUTION ORDER ON LINUX                                │
  │                                                                      │
  │  When an application calls getaddrinfo("google.com"):               │
  │                                                                      │
  │  Step 1: Check /etc/nsswitch.conf (name resolution order)           │
  │          Usually: "hosts: files dns"                                 │
  │          → Check files first, then DNS                               │
  │                                                                      │
  │  Step 2: Check /etc/hosts (local override)                          │
  │          127.0.0.1  localhost                                        │
  │          192.168.1.50  myapp.local                                   │
  │          If found → return immediately, no DNS query.                │
  │                                                                      │
  │  Step 3: Query DNS resolver (from /etc/resolv.conf)                 │
  │          nameserver 8.8.8.8                                          │
  │          nameserver 8.8.4.4                                          │
  │          search example.com                                          │
  │                                                                      │
  │  Step 4: DNS server resolves (recursive query)                       │
  │          Root (.) → .com → google.com → A record: 142.250.182.46   │
  │                                                                      │
  │  IMPORTANT: /etc/hosts OVERRIDES DNS (always checked first).        │
  │  This is how you test a new server before changing DNS.              │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# ──────────────────── DNS LOOKUP TOOLS ────────────────────

# nslookup — simple lookup (available everywhere):
nslookup google.com
# Server:    8.8.8.8
# Address:   8.8.8.8#53
#
# Non-authoritative answer:
# Name:      google.com
# Address:   142.250.182.46

# Query specific DNS server:
nslookup google.com 1.1.1.1        # Ask Cloudflare DNS

# Reverse lookup (IP → hostname):
nslookup 142.250.182.46

# ──────────────────── dig — THE POWER TOOL ────────────────────
# dig provides MUCH more detail than nslookup:

dig google.com
# ;; ANSWER SECTION:
# google.com.    228    IN    A    142.250.182.46
#                 ↑           ↑         ↑
#              TTL(sec)    Record    IP Address
#                          Type

# Specific record types:
dig google.com A               # IPv4 address
dig google.com AAAA            # IPv6 address
dig google.com MX              # Mail servers
dig google.com NS              # Name servers
dig google.com TXT             # TXT records (SPF, DKIM, verification)
dig google.com CNAME           # Canonical name (alias)
dig google.com SOA             # Start of Authority (zone info)
dig google.com ANY             # All records (some servers block this)

# Short output (just the answer):
dig +short google.com
# 142.250.182.46

# Trace the full DNS resolution path:
dig +trace google.com
# Shows: root servers → .com servers → google.com authoritative servers

# Query specific DNS server:
dig @8.8.8.8 google.com       # Ask Google DNS
dig @1.1.1.1 google.com       # Ask Cloudflare DNS

# Check TTL (cache duration):
dig +nocmd +noall +answer google.com
# google.com.  228  IN  A  142.250.182.46
#              ↑ TTL: 228 seconds until cache expires

# ──────────────────── DNS CONFIG FILES ────────────────────
cat /etc/resolv.conf
# nameserver 8.8.8.8           ← Primary DNS
# nameserver 8.8.4.4           ← Secondary DNS
# search example.com           ← Append this domain for short names
#                                 (so "web1" resolves as "web1.example.com")

cat /etc/hosts
# 127.0.0.1   localhost
# 192.168.1.50 myapp.local    ← Local override (no DNS needed)

# On systems with systemd-resolved:
resolvectl status              # Show actual DNS config
resolvectl query google.com    # Resolve using systemd-resolved
```

### 13.8 Port Connectivity Testing

```bash
# ──────────────────── TEST IF A PORT IS OPEN ────────────────────

# telnet (classic — installed on most systems):
telnet 192.168.1.50 80
# Connected to 192.168.1.50.  ← PORT IS OPEN ✅
# Connection refused.         ← PORT IS CLOSED (no service)
# Trying... timeout           ← FIREWALL BLOCKING (packet dropped)

# nc (netcat) — more versatile:
nc -zv 192.168.1.50 80        # -z = scan only, -v = verbose
# Connection to 192.168.1.50 80 port [tcp/http] succeeded!

# Scan a range of ports:
nc -zv 192.168.1.50 20-100    # Ports 20 through 100

# curl — test HTTP specifically:
curl -v http://192.168.1.50:80/
curl -sI http://192.168.1.50/  # Headers only, silent mode
curl -o /dev/null -s -w "%{http_code}" http://192.168.1.50/  # Just status code

# /dev/tcp — bash built-in (no extra tools needed!):
(echo >/dev/tcp/192.168.1.50/80) 2>/dev/null && echo "OPEN" || echo "CLOSED"
# Works even in minimal environments with no nc/telnet/curl

# timeout for scripts:
timeout 3 bash -c 'echo > /dev/tcp/192.168.1.50/80' 2>/dev/null
echo $?   # 0 = open, 1 = closed/timeout
```

### 13.9 Network Configuration Files

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         KEY NETWORK CONFIG FILES                                      │
  │                                                                      │
  │  ┌──────────────────────────┬──────────────────────────────────────┐ │
  │  │ File                     │ Purpose                              │ │
  │  ├──────────────────────────┼──────────────────────────────────────┤ │
  │  │ /etc/hostname            │ This machine's hostname              │ │
  │  │ /etc/hosts               │ Static hostname → IP mappings        │ │
  │  │ /etc/resolv.conf         │ DNS server configuration             │ │
  │  │ /etc/nsswitch.conf       │ Name resolution order                │ │
  │  │ /etc/network/interfaces  │ Interface config (Debian old)        │ │
  │  │ /etc/netplan/*.yaml      │ Interface config (Ubuntu 18+)        │ │
  │  │ /etc/sysconfig/network-  │ Interface config (RHEL/CentOS)       │ │
  │  │   scripts/ifcfg-eth0     │                                      │ │
  │  │ /etc/services            │ Port → service name mappings         │ │
  │  │ /etc/protocols           │ Protocol number mappings             │ │
  │  └──────────────────────────┴──────────────────────────────────────┘ │
  │                                                                      │
  │  IMPORTANT: On modern Ubuntu, network config uses Netplan:           │
  │                                                                      │
  │  /etc/netplan/01-netcfg.yaml:                                        │
  │  network:                                                            │
  │    version: 2                                                        │
  │    ethernets:                                                        │
  │      eth0:                                                           │
  │        dhcp4: no                                                     │
  │        addresses: [192.168.1.100/24]                                 │
  │        gateway4: 192.168.1.1                                         │
  │        nameservers:                                                  │
  │          addresses: [8.8.8.8, 8.8.4.4]                              │
  │                                                                      │
  │  Apply: sudo netplan apply                                           │
  └──────────────────────────────────────────────────────────────────────┘
```

### 13.10 Firewall Basics — iptables and ufw

```bash
# ──────────────────── ufw (Ubuntu — simplified firewall) ────────────────────
sudo ufw status                    # Check status
sudo ufw status verbose            # Detailed status
sudo ufw enable                    # Turn on firewall
sudo ufw disable                   # Turn off firewall

# Allow/deny:
sudo ufw allow 80/tcp             # Allow HTTP
sudo ufw allow 443/tcp            # Allow HTTPS
sudo ufw allow 22/tcp             # Allow SSH (DO THIS BEFORE ENABLING!)
sudo ufw allow from 10.0.0.0/8   # Allow entire subnet
sudo ufw deny 3306/tcp            # Block MySQL from outside

# Delete a rule:
sudo ufw delete allow 80/tcp

# ──────────────────── iptables (low-level — all distros) ────────────────────
# View rules:
sudo iptables -L -n -v            # List all rules (numeric, verbose)
sudo iptables -L -n --line-numbers # With line numbers (for deletion)

# Common rules:
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT    # Allow HTTP
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT    # Allow SSH
sudo iptables -A INPUT -j DROP                         # Drop everything else
# ⚠️ Order matters! Rules are processed top-to-bottom, first match wins.

# Save rules (they're lost on reboot otherwise!):
sudo iptables-save > /etc/iptables/rules.v4    # Debian/Ubuntu
sudo service iptables save                       # RHEL/CentOS
```

> **⚠️ Gotcha (Career-Ending):** When enabling a firewall remotely via SSH, ALWAYS add an SSH allow rule FIRST: `ufw allow 22/tcp` THEN `ufw enable`. If you enable the firewall without allowing SSH, you lock yourself out. On cloud VMs, you'd need console access (which you might not have) to fix this.

### 13.11 Common Network Debugging Scenarios

```bash
# ──────────── SCENARIO 1: "Website is down" ────────────
# Step 1: Can you resolve the name?
dig mysite.com +short
# No answer? → DNS issue. Check /etc/resolv.conf, try dig @8.8.8.8

# Step 2: Can you reach the IP?
ping -c 2 142.250.182.46
# No response? → Network issue or ICMP blocked. Try:
traceroute 142.250.182.46      # Where does it die?

# Step 3: Is the port open?
nc -zv 142.250.182.46 443
# Connection refused? → Service not running.
# Timeout? → Firewall blocking.

# Step 4: Is the service responding?
curl -sI https://mysite.com
# Check HTTP status code, headers.

# ──────────── SCENARIO 2: "App can't connect to database" ────────────
# Is the DB listening?
ss -tlnp | grep 3306
# LISTEN 0 128 127.0.0.1:3306   ← Only on localhost!
# Fix: change bind-address in MySQL config to 0.0.0.0 or specific IP.

# Can the app server reach the DB port?
nc -zv db-server 3306
# Connection refused → DB not listening or firewall blocking.

# ──────────── SCENARIO 3: "Too many connections" ────────────
# Count connections per state:
ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn
#  5000 TIME-WAIT       ← Normal for busy servers (connections closing)
#  2000 ESTABLISHED     ← Active connections
#   500 CLOSE-WAIT      ← ⚠️ App not closing connections! Bug in app.

# TIME-WAIT: Normal, kernel is waiting to ensure all packets arrived.
# CLOSE-WAIT: The REMOTE side closed, but YOUR app hasn't closed its end.
#             This is always a bug in your application code.

# Count connections per remote IP:
ss -tn state established | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn | head
# Shows which IPs have the most connections (DDoS detection).
```

> **DevOps Pro Tip:** The single most important networking command for server debugging is `ss -tlnp` — "What services are listening on what ports?" If the service isn't in this list, it's not running (or listening on the wrong interface). This is always step 1 when debugging connectivity issues.

---

## 14. Nginx Production Scenarios — Config Deletion, SED, Troubleshooting

### 14.1 Nginx Architecture — Quick Mental Model

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         NGINX ARCHITECTURE                                           │
  │                                                                      │
  │  ┌──────────────────────────────────────────────────────┐            │
  │  │  MASTER PROCESS (runs as root)                       │            │
  │  │  PID 1234                                            │            │
  │  │  • Reads configuration                               │            │
  │  │  • Binds to privileged ports (80, 443)               │            │
  │  │  • Spawns worker processes                           │            │
  │  │  • Handles signals (reload, graceful shutdown)       │            │
  │  │                                                      │            │
  │  │  ┌──────────────────────────────────────────────┐    │            │
  │  │  │  WORKER PROCESSES (run as www-data / nginx)  │    │            │
  │  │  │  PIDs: 1235, 1236, 1237, 1238                │    │            │
  │  │  │  • Handle client connections                  │    │            │
  │  │  │  • Process requests                           │    │            │
  │  │  │  • Serve static files                         │    │            │
  │  │  │  • Proxy to upstream (backend) servers        │    │            │
  │  │  │  • Event-driven, non-blocking I/O             │    │            │
  │  │  │  • Each worker handles THOUSANDS of           │    │            │
  │  │  │    connections simultaneously                 │    │            │
  │  │  └──────────────────────────────────────────────┘    │            │
  │  └──────────────────────────────────────────────────────┘            │
  │                                                                      │
  │  worker_processes auto;   ← Usually = number of CPU cores            │
  │  worker_connections 1024; ← Max connections per worker               │
  │  Max total connections = workers × worker_connections                 │
  │  e.g., 4 workers × 1024 = 4096 simultaneous connections             │
  └──────────────────────────────────────────────────────────────────────┘
```

### 14.2 Nginx Configuration Structure

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         NGINX CONFIG FILE HIERARCHY                                  │
  │                                                                      │
  │  /etc/nginx/                                                         │
  │  ├── nginx.conf               ← MAIN config (global settings)       │
  │  ├── mime.types               ← MIME type mappings                   │
  │  ├── conf.d/                  ← Additional config files (*.conf)     │
  │  │   └── default.conf         ← Default server block                 │
  │  ├── sites-available/         ← All virtual host configs             │
  │  │   ├── default              ← Default site                         │
  │  │   └── mysite.conf          ← Your site config                     │
  │  ├── sites-enabled/           ← ACTIVE sites (symlinks to available) │
  │  │   └── mysite.conf → ../sites-available/mysite.conf               │
  │  ├── snippets/                ← Reusable config fragments            │
  │  │   └── ssl-params.conf      ← SSL settings snippet                │
  │  └── modules-enabled/         ← Loaded dynamic modules               │
  │                                                                      │
  │  ACTIVATION PATTERN:                                                 │
  │  1. Create config in sites-available/                                │
  │  2. Symlink to sites-enabled/:                                       │
  │     ln -s /etc/nginx/sites-available/mysite.conf \                  │
  │           /etc/nginx/sites-enabled/                                  │
  │  3. Test: nginx -t                                                   │
  │  4. Reload: systemctl reload nginx                                   │
  │                                                                      │
  │  To disable: rm /etc/nginx/sites-enabled/mysite.conf                │
  │  (removes symlink, original in sites-available/ is preserved)       │
  └──────────────────────────────────────────────────────────────────────┘
```

### 14.3 Nginx Config — The Context Hierarchy

```bash
# /etc/nginx/nginx.conf — STRUCTURE:

# ─── MAIN CONTEXT (global) ───
user www-data;                          # Worker process user
worker_processes auto;                  # Number of workers (auto = CPU cores)
pid /run/nginx.pid;                     # PID file location
error_log /var/log/nginx/error.log;     # Global error log

# ─── EVENTS CONTEXT ───
events {
    worker_connections 1024;            # Max connections per worker
    multi_accept on;                    # Accept multiple connections at once
    use epoll;                          # Linux event mechanism (best performance)
}

# ─── HTTP CONTEXT ───
http {
    # MIME types:
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Logging:
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    # Performance:
    sendfile on;                        # Efficient file serving (kernel-level)
    tcp_nopush on;                      # Optimize TCP packet sending
    tcp_nodelay on;                     # Disable Nagle's algorithm
    keepalive_timeout 65;               # Keep connections open (seconds)
    gzip on;                            # Compress responses

    # Include site configs:
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;

    # ─── SERVER CONTEXT (virtual host) ───
    server {
        listen 80;                      # Listen on port 80
        server_name example.com www.example.com;
        root /var/www/example;          # Document root

        # ─── LOCATION CONTEXT (URL matching) ───
        location / {
            try_files $uri $uri/ =404;  # Try file, then dir, then 404
        }

        location /api/ {
            proxy_pass http://localhost:3000;  # Reverse proxy
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        location ~* \.(jpg|jpeg|png|gif|css|js)$ {
            expires 30d;                # Cache static files for 30 days
            add_header Cache-Control "public, immutable";
        }
    }
}
```

### 14.4 SCENARIO: Nginx Config File Deleted — Recovery

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │  PRODUCTION SCENARIO: Someone deleted the nginx config!              │
  │                                                                      │
  │  "Oh no, I accidentally ran: rm /etc/nginx/nginx.conf"             │
  │                                                                      │
  │  DON'T PANIC. Here's the systematic recovery:                       │
  │                                                                      │
  │  STEP 1: IS NGINX STILL RUNNING?                                    │
  │  $ systemctl status nginx                                            │
  │  If YES → nginx is using the CONFIG IN MEMORY.                      │
  │  DO NOT restart/reload — that would try to read the deleted file!   │
  │                                                                      │
  │  STEP 2: RECOVER CONFIG FROM RUNNING PROCESS                        │
  │  Option A: Check if process still has the file open:                 │
  │  $ sudo ls -la /proc/$(pgrep -o nginx)/fd/ | grep nginx.conf       │
  │  $ sudo cat /proc/$(pgrep -o nginx)/fd/<fd_number>                  │
  │  (May work if the fd is still open — not always the case for        │
  │   config files that are read-then-closed)                            │
  │                                                                      │
  │  Option B: Recover from package default:                             │
  │  $ sudo apt install --reinstall nginx-common    # Debian/Ubuntu     │
  │  $ sudo yum reinstall nginx                      # RHEL/CentOS      │
  │  This restores the DEFAULT config (not your customizations).         │
  │                                                                      │
  │  Option C: Check backups / version control:                          │
  │  $ ls /etc/nginx/nginx.conf.bak                  # Manual backup?   │
  │  $ sudo find / -name "nginx.conf*" 2>/dev/null   # Find copies     │
  │  $ etckeeper log                                  # If using etckeeper│
  │  $ git log                                        # If /etc is in git│
  │                                                                      │
  │  Option D: Reconstruct from nginx -T:                                │
  │  $ sudo nginx -T                                                     │
  │  This DUMPS the entire running configuration (all included files)!  │
  │  ⚠️ Only works while nginx is still running with old config.        │
  │  $ sudo nginx -T > /tmp/nginx_full_config.txt                       │
  │                                                                      │
  │  STEP 3: RESTORE AND TEST                                           │
  │  $ sudo cp /tmp/recovered_nginx.conf /etc/nginx/nginx.conf          │
  │  $ sudo nginx -t                      # Test config syntax           │
  │  $ sudo systemctl reload nginx        # Reload (not restart!)       │
  │                                                                      │
  │  PREVENTION:                                                         │
  │  • Use etckeeper or git for /etc/                                    │
  │  • Automate backups of /etc/nginx/                                   │
  │  • Use Ansible/Puppet/Chef for config management                    │
  │  • Never edit production configs directly (use staging → deploy)    │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# CRITICAL COMMAND: nginx -T (capital T)
# Dumps FULL effective configuration (all includes merged)
sudo nginx -T
# Shows the complete config that nginx is CURRENTLY using.
# Pipe to a file for recovery:
sudo nginx -T > /tmp/nginx_recovery.conf 2>&1

# nginx -t (lowercase t)
# TESTS configuration syntax without applying:
sudo nginx -t
# nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
# nginx: configuration file /etc/nginx/nginx.conf test is successful

# ALWAYS run nginx -t BEFORE reload/restart!
```

> **⚠️ Gotcha:** `nginx -T` vs `nginx -t` — capital T DUMPS config, lowercase t TESTS config. Remember: T = Text dump, t = test.

### 14.5 Using sed with Nginx Configs — Production Patterns

```bash
# ──────────────────── SED FOR NGINX CONFIG CHANGES ────────────────────
# sed is commonly used for automated config changes in scripts/CI-CD.

# ALWAYS backup before sed:
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak.$(date +%s)

# Change worker_processes:
sudo sed -i 's/worker_processes [0-9]*;/worker_processes 4;/' /etc/nginx/nginx.conf

# Change server_name:
sudo sed -i 's/server_name .*/server_name newsite.com www.newsite.com;/' \
    /etc/nginx/sites-available/mysite.conf

# Enable gzip (uncomment line):
sudo sed -i 's/# *gzip on;/gzip on;/' /etc/nginx/nginx.conf

# Disable a setting (comment out):
sudo sed -i 's/^[[:space:]]*access_log/    # access_log/' /etc/nginx/nginx.conf

# Change listen port:
sudo sed -i 's/listen 80;/listen 8080;/' /etc/nginx/sites-available/mysite.conf

# Add a header inside a location block (after proxy_pass):
sudo sed -i '/proxy_pass/a\        proxy_set_header X-Custom-Header "value";' \
    /etc/nginx/sites-available/mysite.conf

# Replace upstream server:
sudo sed -i 's|proxy_pass http://old-backend:3000;|proxy_pass http://new-backend:3000;|' \
    /etc/nginx/sites-available/mysite.conf

# ALWAYS test after sed changes:
sudo nginx -t && sudo systemctl reload nginx
# The && ensures reload ONLY happens if test passes.
```

> **DevOps Pro Tip:** In production pipelines, NEVER use `sed -i` without first doing `sed` (no -i) and piping to review the changes. Pattern: `sed 's/old/new/g' config.conf | diff config.conf -` — shows you exactly what would change. Then apply with `-i`. One wrong sed pattern can break every config file it touches.

### 14.6 Nginx Troubleshooting — Complete Flowchart

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │  NGINX TROUBLESHOOTING FLOWCHART                                     │
  │                                                                      │
  │  SYMPTOM: "Website not working"                                      │
  │      │                                                               │
  │      ▼                                                               │
  │  ┌──────────────────────────────────────┐                            │
  │  │ 1. Is nginx running?                 │                            │
  │  │    systemctl status nginx             │                            │
  │  │    ss -tlnp | grep nginx             │                            │
  │  └──────┬───────────────┬───────────────┘                            │
  │    NOT RUNNING        RUNNING                                        │
  │      │                    │                                          │
  │      ▼                    ▼                                          │
  │  ┌──────────────┐  ┌──────────────────────────────────┐              │
  │  │ Start it:    │  │ 2. Test config syntax:            │              │
  │  │ systemctl    │  │    nginx -t                       │              │
  │  │ start nginx  │  └──────┬───────────────┬────────────┘              │
  │  │              │    CONFIG ERROR       CONFIG OK                     │
  │  │ Won't start? │      │                    │                        │
  │  │ Check logs:  │      ▼                    ▼                        │
  │  │ journalctl   │  ┌──────────────┐  ┌──────────────────────┐        │
  │  │ -u nginx     │  │ Fix the error│  │ 3. Check error log:  │        │
  │  └──────────────┘  │ shown by     │  │ tail -100 /var/log/  │        │
  │                     │ nginx -t     │  │ nginx/error.log      │        │
  │                     └──────────────┘  └──────┬───────────────┘        │
  │                                              │                       │
  │                                              ▼                       │
  │                                  ┌──────────────────────────┐        │
  │                                  │ 4. Check access log:     │        │
  │                                  │ tail -100 /var/log/      │        │
  │                                  │ nginx/access.log         │        │
  │                                  │                          │        │
  │                                  │ Look for status codes:   │        │
  │                                  │ 403 = permission denied  │        │
  │                                  │ 404 = file not found     │        │
  │                                  │ 500 = app error          │        │
  │                                  │ 502 = backend down       │        │
  │                                  │ 504 = backend timeout    │        │
  │                                  └──────┬───────────────────┘        │
  │                                         │                            │
  │                                         ▼                            │
  │  ┌──────────────────────────────────────────────────────────────┐    │
  │  │ COMMON FIXES BY ERROR CODE:                                  │    │
  │  │                                                              │    │
  │  │ 403 Forbidden:                                               │    │
  │  │ • Check file permissions: ls -la /var/www/                   │    │
  │  │ • Check nginx user: ps aux | grep nginx (running as who?)   │    │
  │  │ • Check directory index: index directive in config?          │    │
  │  │ • Check SELinux: getenforce / ausearch -m avc               │    │
  │  │                                                              │    │
  │  │ 502 Bad Gateway:                                             │    │
  │  │ • Backend is DOWN: systemctl status <backend_app>            │    │
  │  │ • Wrong upstream address: check proxy_pass URL               │    │
  │  │ • Socket/port mismatch between nginx and app                 │    │
  │  │                                                              │    │
  │  │ 504 Gateway Timeout:                                         │    │
  │  │ • Backend is TOO SLOW: check app performance                 │    │
  │  │ • Increase timeout: proxy_read_timeout 120s;                │    │
  │  │ • Check backend logs for slow queries                        │    │
  │  │                                                              │    │
  │  │ Connection refused on :80/:443:                              │    │
  │  │ • nginx not listening: ss -tlnp | grep nginx                │    │
  │  │ • Firewall blocking: ufw status / iptables -L               │    │
  │  │ • Wrong listen directive in config                           │    │
  │  └──────────────────────────────────────────────────────────────┘    │
  └──────────────────────────────────────────────────────────────────────┘
```

### 14.7 Nginx Signals and Graceful Operations

```bash
# Nginx responds to these signals:
# TERM, INT  → Quick shutdown (kill connections immediately)
# QUIT       → Graceful shutdown (finish current requests, then stop)
# HUP        → Reload configuration (graceful — no downtime)
# USR1       → Reopen log files (used by logrotate)
# USR2       → Upgrade binary on-the-fly (zero-downtime upgrade)

# The systemctl equivalents:
sudo systemctl stop nginx       # Sends SIGTERM → quick shutdown
sudo systemctl reload nginx     # Sends SIGHUP → graceful config reload
sudo systemctl restart nginx    # SIGTERM + start → brief downtime!

# ──────────── WHY reload IS BETTER THAN restart ────────────
# reload:
# 1. Master process reads new config
# 2. Tests new config syntax
# 3. Spawns NEW workers with new config
# 4. Old workers finish existing requests
# 5. Old workers exit → only new workers remain
# = ZERO DOWNTIME ✅

# restart:
# 1. Sends SIGTERM → all workers killed immediately
# 2. nginx stops completely
# 3. nginx starts fresh with new config
# = BRIEF DOWNTIME ❌ (connections dropped during stop)

# ALWAYS prefer reload over restart for config changes!
sudo nginx -t && sudo systemctl reload nginx
```

### 14.8 Nginx as Reverse Proxy — Production Config

```bash
# Complete reverse proxy config with production settings:

# /etc/nginx/sites-available/myapp.conf
upstream backend {
    server 127.0.0.1:3000;           # Primary app server
    server 127.0.0.1:3001;           # Second app instance
    # server 10.0.0.50:3000 backup;  # Failover server
    keepalive 32;                     # Keep connections to backend alive
}

server {
    listen 80;
    server_name myapp.com www.myapp.com;
    return 301 https://$server_name$request_uri;  # Redirect HTTP → HTTPS
}

server {
    listen 443 ssl http2;
    server_name myapp.com www.myapp.com;

    # SSL certificates:
    ssl_certificate /etc/letsencrypt/live/myapp.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/myapp.com/privkey.pem;

    # SSL hardening:
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;

    # Security headers:
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # Logging:
    access_log /var/log/nginx/myapp_access.log;
    error_log /var/log/nginx/myapp_error.log;

    # Static files (served directly by nginx — fast):
    location /static/ {
        alias /var/www/myapp/static/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # API / Application (proxied to backend):
    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts:
        proxy_connect_timeout 10s;
        proxy_send_timeout 30s;
        proxy_read_timeout 60s;

        # Buffering:
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
    }

    # Health check endpoint:
    location /health {
        access_log off;                # Don't log health checks
        return 200 "OK\n";
        add_header Content-Type text/plain;
    }
}
```

### 14.9 Nginx Location Block Matching — Priority Order

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         LOCATION MATCHING PRIORITY (highest to lowest)               │
  │                                                                      │
  │  ┌──────┬────────────────┬──────────────────────────────────────┐    │
  │  │ Rank │ Syntax         │ Meaning                              │    │
  │  ├──────┼────────────────┼──────────────────────────────────────┤    │
  │  │  1   │ location = /x  │ EXACT match only (highest priority)  │    │
  │  │  2   │ location ^~ /x │ Prefix match, STOPS regex search     │    │
  │  │  3   │ location ~ /x  │ Case-SENSITIVE regex                 │    │
  │  │  3   │ location ~* /x │ Case-INSENSITIVE regex               │    │
  │  │  4   │ location /x    │ Prefix match (lowest priority)       │    │
  │  └──────┴────────────────┴──────────────────────────────────────┘    │
  │                                                                      │
  │  MATCHING ALGORITHM:                                                 │
  │  1. nginx checks ALL prefix locations, finds the LONGEST match      │
  │  2. If longest match is = (exact) → use it immediately. DONE.       │
  │  3. If longest match is ^~ → use it immediately. DONE.              │
  │  4. Otherwise, nginx checks ALL regex locations IN ORDER             │
  │  5. First regex match wins → use it. DONE.                          │
  │  6. If no regex matches → use the longest prefix from step 1.       │
  │                                                                      │
  │  EXAMPLE:                                                            │
  │  location = / { ... }          ← Only matches exactly "/"           │
  │  location / { ... }            ← Matches everything (prefix "/")    │
  │  location /api/ { ... }        ← Matches /api/*, /api/users, etc.  │
  │  location ^~ /images/ { ... }  ← Matches /images/*, blocks regex   │
  │  location ~* \.(jpg|png)$ { }  ← Regex: matches .jpg/.png files    │
  │                                                                      │
  │  For /images/photo.jpg:                                              │
  │  - Prefix "/images/" matches (^~ stops regex search) → USED         │
  │  - Even though regex \.(jpg|png)$ also matches — it's never tested │
  └──────────────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha (Interview Classic):** "What's the difference between `location /app` and `location /app/`?" Both match `/app/page`, but `location /app` ALSO matches `/application` and `/applesauce` (prefix matching). Always use the trailing `/` when you intend to match a directory path only.

### 14.10 Nginx Quick Reference — Emergency Commands

```bash
# ──────────────────── DAILY OPERATIONS ────────────────────
sudo nginx -t                          # Test config (ALWAYS before reload)
sudo nginx -T                          # Dump full effective config
sudo systemctl reload nginx            # Apply config (zero downtime)
sudo systemctl status nginx            # Check status

# ──────────────────── LOG ANALYSIS ────────────────────
# Real-time error monitoring:
sudo tail -f /var/log/nginx/error.log

# Top requesting IPs:
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# Top requested URLs:
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# Status code distribution:
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# 5xx errors in last hour:
awk -v d="$(date -d '1 hour ago' '+%d/%b/%Y:%H')" '$4 ~ d && $9 ~ /^5/' \
    /var/log/nginx/access.log

# Requests per second:
awk '{print $4}' /var/log/nginx/access.log | cut -d: -f1-3 | uniq -c | tail -20

# ──────────────────── SECURITY ────────────────────
# Block an IP:
# Add to server block: deny 1.2.3.4;
# Or create /etc/nginx/conf.d/blocklist.conf:
# deny 1.2.3.4;
# deny 5.6.7.0/24;
sudo nginx -t && sudo systemctl reload nginx

# Rate limiting:
# In http context:
# limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
# In location:
# limit_req zone=api burst=20 nodelay;
```

> **DevOps Pro Tip:** Set up these aliases on every server you manage:
> ```
> alias ngt='sudo nginx -t'
> alias ngr='sudo nginx -t && sudo systemctl reload nginx'
> alias ngl='sudo tail -f /var/log/nginx/error.log'
> ```
> `ngt` = test, `ngr` = test-and-reload (safe), `ngl` = live error log. Three commands that cover 90% of daily nginx work.

---

*Batch 5 complete — Phases 9 & 10 (Networking Commands + Nginx Production Scenarios)*

---

## 15. Windows Server & Active Directory — Domain Controller, OUs, GPOs

### 15.1 Why Windows Server Matters for DevOps

Even in Linux-heavy DevOps environments, you'll encounter Windows servers — especially in enterprise environments. Active Directory manages user authentication for thousands of employees. Understanding AD, GPOs, and Windows Server basics is essential for hybrid infrastructure management.

### 15.2 Windows Server Editions — What You Need to Know

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         WINDOWS SERVER 2022 EDITIONS                                 │
  │                                                                      │
  │  ┌──────────────────┬────────────────────────────────────────────┐   │
  │  │ Edition          │ Use Case                                   │   │
  │  ├──────────────────┼────────────────────────────────────────────┤   │
  │  │ Standard         │ Physical or lightly virtualized servers.   │   │
  │  │                  │ 2 Hyper-V VMs included with license.       │   │
  │  │                  │ Most common in small/medium business.      │   │
  │  ├──────────────────┼────────────────────────────────────────────┤   │
  │  │ Datacenter       │ Highly virtualized / cloud environments.   │   │
  │  │                  │ UNLIMITED Hyper-V VMs. Shielded VMs,       │   │
  │  │                  │ Software-Defined Networking/Storage.       │   │
  │  │                  │ Enterprise & datacenter workloads.         │   │
  │  ├──────────────────┼────────────────────────────────────────────┤   │
  │  │ Essentials       │ Small businesses (≤25 users, 50 devices). │   │
  │  │                  │ No CALs needed. Built-in AD.               │   │
  │  │                  │ Limited roles, no Hyper-V nesting.         │   │
  │  └──────────────────┴────────────────────────────────────────────┘   │
  │                                                                      │
  │  INSTALLATION MODES:                                                 │
  │  ┌────────────────────────┬──────────────────────────────────────┐   │
  │  │ Desktop Experience     │ Full GUI (Server Manager, MMC)       │   │
  │  │ (GUI)                  │ Easier to learn, more resource usage │   │
  │  ├────────────────────────┼──────────────────────────────────────┤   │
  │  │ Server Core            │ CLI only (PowerShell, sconfig)       │   │
  │  │ (No GUI)               │ Smaller attack surface, less updates │   │
  │  │                        │ Less RAM/disk. Production preferred. │   │
  │  └────────────────────────┴──────────────────────────────────────┘   │
  └──────────────────────────────────────────────────────────────────────┘
```

### 15.3 Active Directory — The Central Identity System

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         ACTIVE DIRECTORY (AD) — MENTAL MODEL                         │
  │                                                                      │
  │  Active Directory is a DIRECTORY SERVICE — a centralized database    │
  │  of users, computers, groups, and policies for an organization.     │
  │                                                                      │
  │  Think of it as: "One place to manage ALL user accounts, passwords, │
  │  permissions, and computer settings for the entire company."         │
  │                                                                      │
  │  WITHOUT AD: Each computer has LOCAL accounts. 500 employees =      │
  │  500 × N computers = thousands of accounts to manage individually.  │
  │  Password changes? Log into each machine. NIGHTMARE.                 │
  │                                                                      │
  │  WITH AD: ONE account per user. Log in to ANY domain-joined         │
  │  computer. Central password policy. Central permissions.             │
  │  Change once → applies everywhere.                                   │
  │                                                                      │
  │  KEY COMPONENTS:                                                     │
  │  ┌────────────────────────────────────────────────────────────┐      │
  │  │                                                            │      │
  │  │  FOREST ──── The top-level boundary. One or more DOMAINS. │      │
  │  │    │         All domains in a forest trust each other.     │      │
  │  │    │                                                       │      │
  │  │    ├── DOMAIN ──── simform.com                             │      │
  │  │    │     │         The core unit. Contains users, groups,  │      │
  │  │    │     │         computers, policies.                    │      │
  │  │    │     │                                                 │      │
  │  │    │     ├── DOMAIN CONTROLLER (DC)                        │      │
  │  │    │     │   Server running AD DS (AD Domain Services).    │      │
  │  │    │     │   Stores the AD database (NTDS.dit).            │      │
  │  │    │     │   Handles authentication (Kerberos/NTLM).       │      │
  │  │    │     │   You need at least 2 DCs (redundancy!).        │      │
  │  │    │     │                                                 │      │
  │  │    │     ├── ORGANIZATIONAL UNITS (OUs)                    │      │
  │  │    │     │   Containers to ORGANIZE objects (like folders).│      │
  │  │    │     │   GPOs are linked to OUs.                       │      │
  │  │    │     │                                                 │      │
  │  │    │     ├── USERS (user accounts)                         │      │
  │  │    │     ├── GROUPS (collections of users)                 │      │
  │  │    │     ├── COMPUTERS (domain-joined machines)            │      │
  │  │    │     └── GROUP POLICY OBJECTS (GPOs)                   │      │
  │  │    │                                                       │      │
  │  │    └── CHILD DOMAIN ──── dev.simform.com (optional)        │      │
  │  │                                                            │      │
  │  └────────────────────────────────────────────────────────────┘      │
  └──────────────────────────────────────────────────────────────────────┘
```

### 15.4 Domain Controller (DC) — The Heart of AD

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         DOMAIN CONTROLLER — WHAT IT DOES                             │
  │                                                                      │
  │  A Domain Controller (DC) is a Windows Server that runs the          │
  │  Active Directory Domain Services (AD DS) role.                      │
  │                                                                      │
  │  RESPONSIBILITIES:                                                   │
  │  ┌──────────────────────────────────────────────────────────────┐    │
  │  │ 1. AUTHENTICATION                                            │    │
  │  │    User logs in → DC verifies username + password            │    │
  │  │    Uses Kerberos (default) or NTLM (legacy fallback)        │    │
  │  │                                                              │    │
  │  │ 2. AUTHORIZATION                                             │    │
  │  │    Determines what resources the user can access             │    │
  │  │    Based on group memberships and permissions                │    │
  │  │                                                              │    │
  │  │ 3. DIRECTORY STORAGE                                         │    │
  │  │    Stores the AD database: NTDS.dit                          │    │
  │  │    Located at: C:\Windows\NTDS\ntds.dit                     │    │
  │  │    Contains all users, groups, computers, policies           │    │
  │  │                                                              │    │
  │  │ 4. REPLICATION                                               │    │
  │  │    Multiple DCs replicate changes to each other              │    │
  │  │    Ensures consistency and fault tolerance                   │    │
  │  │                                                              │    │
  │  │ 5. DNS                                                       │    │
  │  │    AD REQUIRES DNS. Clients find DCs via DNS SRV records.   │    │
  │  │    DC typically also runs the DNS Server role.               │    │
  │  │    _ldap._tcp.dc._msdcs.simform.com → DC IP address        │    │
  │  └──────────────────────────────────────────────────────────────┘    │
  │                                                                      │
  │  AUTHENTICATION FLOW:                                                │
  │                                                                      │
  │  ┌────────┐  1. Login     ┌──────┐  2. Kerberos     ┌──────────┐   │
  │  │ User   │──────────────►│ DC   │  authentication   │ Kerberos │   │
  │  │ (PC)   │               │      │◄────────────────►│ KDC      │   │
  │  │        │◄──────────────│      │  3. Ticket        │ (on DC)  │   │
  │  │        │  4. TGT       │      │     Granting      │          │   │
  │  └────┬───┘  (Ticket      └──────┘     Ticket (TGT)  └──────────┘   │
  │       │      Granting                                                │
  │       │      Ticket)                                                 │
  │       │                                                              │
  │       │  5. Present TGT to access file server                       │
  │       ▼                                                              │
  │  ┌──────────┐                                                        │
  │  │ File     │  User accesses \\fileserver\shared                    │
  │  │ Server   │  DC validates the Kerberos ticket                     │
  │  └──────────┘  No password sent over network! ✅                    │
  └──────────────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha:** "Why do you need at least 2 Domain Controllers?" — If your only DC dies, **nobody can log in**. No authentication = no access to anything. All computers show "The trust relationship between this workstation and the primary domain failed." Two DCs replicate to each other — if one fails, the other handles authentication seamlessly.

### 15.5 Organizational Units (OUs) — Logical Organization

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         ORGANIZATIONAL UNITS — STRUCTURING AD                        │
  │                                                                      │
  │  OUs are CONTAINERS within a domain. They organize objects           │
  │  (users, computers, groups) into a logical hierarchy.               │
  │                                                                      │
  │  WHY OUs (not just groups)?                                          │
  │  • Groups = for PERMISSIONS (who can access what)                   │
  │  • OUs = for ADMINISTRATION (who manages what, which policies apply)│
  │  • GPOs are linked to OUs (not to groups!)                          │
  │                                                                      │
  │  EXAMPLE OU STRUCTURE:                                               │
  │                                                                      │
  │  simform.com (Domain)                                                │
  │  ├── 📁 OU=Departments                                              │
  │  │   ├── 📁 OU=Engineering                                          │
  │  │   │   ├── 👤 john.smith                                          │
  │  │   │   ├── 👤 jane.doe                                            │
  │  │   │   └── 💻 ENG-WS-001 (workstation)                           │
  │  │   ├── 📁 OU=Marketing                                            │
  │  │   │   ├── 👤 bob.wilson                                          │
  │  │   │   └── 💻 MKT-WS-001                                         │
  │  │   └── 📁 OU=Finance                                              │
  │  │       ├── 👤 alice.chen                                          │
  │  │       └── 💻 FIN-WS-001                                         │
  │  ├── 📁 OU=Servers                                                   │
  │  │   ├── 💻 WEB-SRV-01                                              │
  │  │   ├── 💻 DB-SRV-01                                               │
  │  │   └── 💻 FILE-SRV-01                                             │
  │  ├── 📁 OU=Service Accounts                                         │
  │  │   ├── 👤 svc-backup                                              │
  │  │   └── 👤 svc-monitoring                                          │
  │  └── 📁 OU=Disabled Accounts                                        │
  │      └── 👤 ex-employee-01 (disabled)                               │
  │                                                                      │
  │  BEST PRACTICES:                                                     │
  │  • Mirror your org structure (departments, locations)               │
  │  • Separate Users, Computers, and Servers into different OUs        │
  │  • Create an OU for disabled/terminated accounts                    │
  │  • Create OUs for service accounts (special permissions)            │
  │  • Don't nest too deep (3-4 levels max)                             │
  │  • GPO inheritance follows the OU tree (parent → child)            │
  └──────────────────────────────────────────────────────────────────────┘
```

### 15.6 Group Policy Objects (GPOs) — Centralized Configuration

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         GROUP POLICY OBJECTS — THE POWER OF AD                       │
  │                                                                      │
  │  GPOs let you configure THOUSANDS of settings across all domain     │
  │  computers and users from ONE central place.                         │
  │                                                                      │
  │  WHAT CAN GPOs DO:                                                   │
  │  ┌──────────────────────────────────────────────────────────────┐    │
  │  │ • Password policies (length, complexity, expiration)         │    │
  │  │ • Account lockout (lock after N failed attempts)             │    │
  │  │ • Software installation (push apps to all computers)         │    │
  │  │ • Desktop restrictions (disable USB, control panel)          │    │
  │  │ • Firewall rules (enforce across all machines)               │    │
  │  │ • Mapped drives (auto-map \\fileserver\shared as Z:)        │    │
  │  │ • Login scripts (run PowerShell on login)                    │    │
  │  │ • Windows Update settings (WSUS configuration)               │    │
  │  │ • Security settings (audit policies, user rights)            │    │
  │  │ • Registry settings (any registry key)                       │    │
  │  │ • Browser configuration (proxy, homepage)                    │    │
  │  └──────────────────────────────────────────────────────────────┘    │
  │                                                                      │
  │  GPO APPLICATION ORDER — LSDOU:                                      │
  │                                                                      │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐            │
  │  │ L-ocal   │──► S-ite    │──► D-omain  │──► OU       │            │
  │  │ Policy   │  │ Policy   │  │ Policy   │  │ Policy   │            │
  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘            │
  │  Applied      Applied         Applied        Applied                │
  │  FIRST        second          third          LAST (wins!)           │
  │  (lowest      ▲               ▲              (highest               │
  │   priority)   │               │               priority)             │
  │               │               │                                     │
  │  If conflicting settings exist, the LAST applied wins.              │
  │  OU policies override Domain policies override Site policies.       │
  │                                                                      │
  │  SPECIAL OVERRIDES:                                                  │
  │  • Enforced (No Override) — parent GPO CANNOT be overridden by OU  │
  │  • Block Inheritance — OU blocks parent GPO from applying           │
  │  • Enforced beats Block Inheritance (enforced always wins)          │
  └──────────────────────────────────────────────────────────────────────┘
```

```
  GPO SECTIONS — COMPUTER vs USER:

  ┌──────────────────────────────────────────────────────────────────────┐
  │  Every GPO has TWO sections:                                         │
  │                                                                      │
  │  ┌───────────────────────────────┬─────────────────────────────┐     │
  │  │ COMPUTER Configuration       │ USER Configuration          │     │
  │  ├───────────────────────────────┼─────────────────────────────┤     │
  │  │ Applied at BOOT time         │ Applied at LOGIN time       │     │
  │  │ Applies to the MACHINE       │ Applies to the USER         │     │
  │  │ regardless of who logs in    │ regardless of which machine │     │
  │  │                              │                             │     │
  │  │ Examples:                    │ Examples:                   │     │
  │  │ • Firewall rules            │ • Desktop wallpaper         │     │
  │  │ • Audit policies            │ • Mapped drives             │     │
  │  │ • Startup scripts           │ • Login scripts             │     │
  │  │ • Windows Update settings   │ • Folder redirection        │     │
  │  │ • Security settings         │ • Software restrictions     │     │
  │  └───────────────────────────────┴─────────────────────────────┘     │
  │                                                                      │
  │  When to use which:                                                  │
  │  "Every MACHINE in the server room must have this firewall rule"    │
  │  → Computer Configuration (linked to Servers OU)                    │
  │                                                                      │
  │  "Every USER in marketing gets the M: drive mapped"                 │
  │  → User Configuration (linked to Marketing OU)                      │
  └──────────────────────────────────────────────────────────────────────┘
```

### 15.7 GPO — Management Commands

```powershell
# ──────────────────── PowerShell GPO Commands ────────────────────
# (Run on Domain Controller or RSAT-installed machine)

# View all GPOs in domain:
Get-GPO -All | Select-Object DisplayName, GpoStatus, CreationTime

# Get details of specific GPO:
Get-GPO -Name "Default Domain Policy"

# Create a new GPO:
New-GPO -Name "Security - Password Policy" -Comment "Enforce password standards"

# Link GPO to OU:
New-GPLink -Name "Security - Password Policy" -Target "OU=Departments,DC=simform,DC=com"

# Generate GPO report (HTML — great for auditing):
Get-GPOReport -All -ReportType Html -Path "C:\GPO_Report.html"

# Force GPO refresh on a client (don't wait for next cycle):
gpupdate /force
# Computer Policy update has completed successfully.
# User Policy update has completed successfully.

# Check which GPOs are applied to current machine/user:
gpresult /r                  # Summary
gpresult /h C:\gpreport.html # Detailed HTML report

# Check from DC which GPOs apply to a specific user/computer:
Get-GPResultantSetOfPolicy -User "john.smith" -Computer "ENG-WS-001" `
    -ReportType Html -Path "C:\john_gpo.html"
```

### 15.8 AD DS DNS and DHCP Integration

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         AD REQUIRES DNS — WHY AND HOW                                │
  │                                                                      │
  │  Active Directory CANNOT function without DNS.                       │
  │  Clients find Domain Controllers using DNS SRV records.             │
  │                                                                      │
  │  When a computer joins the domain or a user logs in:                │
  │  1. Client queries DNS for: _ldap._tcp.dc._msdcs.simform.com       │
  │  2. DNS returns the IP of a Domain Controller                       │
  │  3. Client contacts DC for authentication                           │
  │                                                                      │
  │  If DNS is broken → NOBODY can log in (can't find DC).              │
  │  This is why the DC usually runs the DNS Server role too.           │
  │                                                                      │
  │  AD-INTEGRATED DNS:                                                  │
  │  • DNS zones stored IN Active Directory (not just zone files)       │
  │  • Replicated automatically with AD replication                     │
  │  • Secure dynamic updates (only domain members can update)          │
  │  • Multi-master (any DC can update DNS)                             │
  │                                                                      │
  │  DHCP + AD:                                                          │
  │  • DHCP assigns IPs AND tells clients the DNS/DC addresses          │
  │  • DHCP scope options:                                               │
  │    - Option 003: Default gateway                                    │
  │    - Option 006: DNS servers (point to your DCs!)                   │
  │    - Option 015: DNS domain name (simform.com)                      │
  │  • DHCP authorization in AD prevents rogue DHCP servers             │
  └──────────────────────────────────────────────────────────────────────┘
```

### 15.9 FSMO Roles — The Special DC Roles

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         FSMO ROLES — Flexible Single Master Operations               │
  │                                                                      │
  │  While most AD operations use multi-master replication (any DC      │
  │  can process changes), FIVE specific operations require ONE          │
  │  designated DC to avoid conflicts. These are FSMO roles.            │
  │                                                                      │
  │  FOREST-WIDE (one per entire forest):                                │
  │  ┌──────────────────────┬──────────────────────────────────────┐     │
  │  │ Schema Master        │ Controls AD schema changes (rare)    │     │
  │  │                      │ Adding new attributes, object types  │     │
  │  ├──────────────────────┼──────────────────────────────────────┤     │
  │  │ Domain Naming Master │ Controls adding/removing domains     │     │
  │  │                      │ in the forest                        │     │
  │  └──────────────────────┴──────────────────────────────────────┘     │
  │                                                                      │
  │  DOMAIN-WIDE (one per domain):                                       │
  │  ┌──────────────────────┬──────────────────────────────────────┐     │
  │  │ RID Master           │ Allocates pools of unique IDs (RIDs) │     │
  │  │                      │ to each DC for creating new objects  │     │
  │  ├──────────────────────┼──────────────────────────────────────┤     │
  │  │ PDC Emulator         │ THE most critical role:              │     │
  │  │                      │ • Time source for domain             │     │
  │  │                      │ • Password changes replicate here    │     │
  │  │                      │   FIRST (login failures check PDC)   │     │
  │  │                      │ • GPO editing targets PDC by default │     │
  │  │                      │ • Account lockout processing         │     │
  │  ├──────────────────────┼──────────────────────────────────────┤     │
  │  │ Infrastructure Master│ Handles cross-domain references      │     │
  │  │                      │ (translates GUIDs/SIDs/DNs)          │     │
  │  └──────────────────────┴──────────────────────────────────────┘     │
  │                                                                      │
  │  Interview answer: "The PDC Emulator is the most important FSMO     │
  │  role — it handles password verification, time sync, and is         │
  │  the first DC to process Group Policy changes."                     │
  └──────────────────────────────────────────────────────────────────────┘
```

```powershell
# Check who holds FSMO roles:
netdom query fsmo
# or:
Get-ADForest | Select-Object SchemaMaster, DomainNamingMaster
Get-ADDomain | Select-Object PDCEmulator, RIDMaster, InfrastructureMaster
```

### 15.10 Windows Server — Essential PowerShell for Interviews

```powershell
# ──────────────────── AD USER MANAGEMENT ────────────────────
# Create a new user:
New-ADUser -Name "John Smith" -SamAccountName "john.smith" `
    -UserPrincipalName "john.smith@simform.com" `
    -Path "OU=Engineering,OU=Departments,DC=simform,DC=com" `
    -AccountPassword (ConvertTo-SecureString "P@ssw0rd!" -AsPlainText -Force) `
    -Enabled $true

# Find a user:
Get-ADUser -Identity "john.smith" -Properties *

# Find all users in an OU:
Get-ADUser -Filter * -SearchBase "OU=Engineering,OU=Departments,DC=simform,DC=com"

# Disable an account (employee leaves):
Disable-ADAccount -Identity "john.smith"

# Move user to Disabled OU:
Move-ADObject -Identity "CN=John Smith,OU=Engineering,OU=Departments,DC=simform,DC=com" `
    -TargetPath "OU=Disabled Accounts,DC=simform,DC=com"

# Reset password:
Set-ADAccountPassword -Identity "john.smith" `
    -NewPassword (ConvertTo-SecureString "NewP@ss!" -AsPlainText -Force) -Reset

# Unlock account (after too many failed attempts):
Unlock-ADAccount -Identity "john.smith"

# ──────────────────── AD GROUP MANAGEMENT ────────────────────
# Create a security group:
New-ADGroup -Name "DevOps Team" -GroupScope Global -GroupCategory Security `
    -Path "OU=Groups,DC=simform,DC=com"

# Add user to group:
Add-ADGroupMember -Identity "DevOps Team" -Members "john.smith"

# Check group membership:
Get-ADGroupMember -Identity "DevOps Team"

# Check what groups a user belongs to:
Get-ADPrincipalGroupMembership -Identity "john.smith" | Select-Object Name

# ──────────────────── SERVER MANAGEMENT ────────────────────
# Check installed roles/features:
Get-WindowsFeature | Where-Object Installed

# Install a role:
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# Check services:
Get-Service | Where-Object Status -eq "Running"

# Check event logs (like journalctl on Linux):
Get-EventLog -LogName System -Newest 50
Get-EventLog -LogName Security -EntryType FailureAudit -Newest 20

# Check disk space (like df on Linux):
Get-PSDrive -PSProvider FileSystem

# Check network config (like ip a on Linux):
Get-NetIPConfiguration
Get-NetIPAddress
```

> **DevOps Pro Tip:** When joining a Linux server to Active Directory (common in hybrid environments), use SSSD + Realm: `realm join simform.com`. This allows AD users to SSH into Linux servers with their AD credentials. The authentication path: Linux → SSSD → Kerberos → AD DC. You'll see this in companies that run Linux servers but use AD for identity management.

---

## 16. Additional Deep-Dive Topics — Containers, Advanced Linux, Interview Curveballs

### 16.1 Containers vs VMs — The Architecture Difference

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         VIRTUAL MACHINES vs CONTAINERS                               │
  │                                                                      │
  │  VIRTUAL MACHINE:                     CONTAINER:                     │
  │  ┌──────────────────────┐             ┌──────────────────────┐      │
  │  │ App A    │ App B     │             │ App A    │ App B     │      │
  │  ├──────────┼───────────┤             ├──────────┼───────────┤      │
  │  │ Bins/Libs│ Bins/Libs │             │ Bins/Libs│ Bins/Libs │      │
  │  ├──────────┼───────────┤             └──────────┴───────────┘      │
  │  │ Guest OS │ Guest OS  │                   │                       │
  │  │ (full    │ (full     │             ┌─────┴─────────────────┐     │
  │  │ kernel!) │ kernel!)  │             │ Container Runtime     │     │
  │  ├──────────┴───────────┤             │ (Docker/containerd)   │     │
  │  │ HYPERVISOR            │             ├───────────────────────┤     │
  │  │ (VMware/KVM/Hyper-V) │             │ HOST OS (one kernel)  │     │
  │  ├───────────────────────┤             ├───────────────────────┤     │
  │  │ HOST OS               │             │ Hardware              │     │
  │  ├───────────────────────┤             └───────────────────────┘     │
  │  │ Hardware              │                                          │
  │  └───────────────────────┘                                          │
  │                                                                      │
  │  KEY DIFFERENCE: VMs virtualize HARDWARE (each VM has its own       │
  │  kernel). Containers virtualize the OS (share the host kernel).     │
  │                                                                      │
  │  ┌─────────────────┬──────────────────┬──────────────────────┐      │
  │  │ Aspect          │ VM               │ Container            │      │
  │  ├─────────────────┼──────────────────┼──────────────────────┤      │
  │  │ Boot time       │ Minutes          │ Seconds (< 1s)       │      │
  │  │ Size            │ GBs (full OS)    │ MBs (just app + deps)│      │
  │  │ Isolation       │ Strong (hardware)│ Process-level         │      │
  │  │ Kernel          │ Own kernel       │ Shares host kernel    │      │
  │  │ Overhead        │ High (full OS)   │ Minimal              │      │
  │  │ Density         │ 10s per host     │ 100s-1000s per host  │      │
  │  │ Use case        │ Different OSes,  │ Microservices, CI/CD,│      │
  │  │                 │ strong isolation  │ rapid scaling         │      │
  │  └─────────────────┴──────────────────┴──────────────────────┘      │
  └──────────────────────────────────────────────────────────────────────┘
```

### 16.2 Docker Fundamentals — What You Need for Interviews

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         DOCKER CONCEPTS — MENTAL MODEL                               │
  │                                                                      │
  │  IMAGE ─────────► CONTAINER ─────────► PROCESS                       │
  │  (blueprint)      (running instance)   (what's actually executing)  │
  │                                                                      │
  │  Image = read-only template (like a class in OOP)                   │
  │  Container = running instance of an image (like an object)           │
  │  You can run MANY containers from ONE image.                         │
  │                                                                      │
  │  LAYERS:                                                             │
  │  ┌───────────────────────────────────────────┐                       │
  │  │ Container Layer (read-write, temporary)   │ ← Your changes       │
  │  ├───────────────────────────────────────────┤                       │
  │  │ App code layer (COPY . /app)              │                       │
  │  ├───────────────────────────────────────────┤                       │
  │  │ Dependencies (RUN npm install)            │ ← Image layers       │
  │  ├───────────────────────────────────────────┤   (read-only,        │
  │  │ Node.js runtime (FROM node:18)            │    shared across     │
  │  ├───────────────────────────────────────────┤    containers)       │
  │  │ Ubuntu base (inherited from node image)   │                       │
  │  └───────────────────────────────────────────┘                       │
  │                                                                      │
  │  DOCKERFILE → builds an IMAGE                                        │
  │  docker run → creates a CONTAINER from an image                      │
  │  docker push → uploads image to a REGISTRY (Docker Hub, ECR, etc.) │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# Essential Docker commands:
docker build -t myapp:v1 .           # Build image from Dockerfile
docker run -d -p 80:3000 myapp:v1    # Run container (detached, port map)
docker ps                             # List running containers
docker ps -a                          # List ALL containers (including stopped)
docker logs <container_id>            # View container logs
docker exec -it <id> /bin/bash        # Shell into running container
docker stop <id>                      # Graceful stop (SIGTERM)
docker kill <id>                      # Force stop (SIGKILL)
docker rm <id>                        # Remove stopped container
docker images                         # List local images
docker rmi <image>                    # Remove image

# Dockerfile example:
# FROM node:18-alpine          ← Base image (small!)
# WORKDIR /app                 ← Set working directory
# COPY package*.json ./        ← Copy dependency files first (cache layer)
# RUN npm ci --production      ← Install dependencies (cached if package.json unchanged)
# COPY . .                     ← Copy app code
# EXPOSE 3000                  ← Document the port (doesn't publish it)
# USER node                    ← Don't run as root!
# CMD ["node", "server.js"]   ← Default command when container starts
```

### 16.3 Linux Namespaces & Cgroups — How Containers Actually Work

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         CONTAINERS = NAMESPACES + CGROUPS                            │
  │                                                                      │
  │  Containers are NOT virtual machines. They are processes with        │
  │  two kernel features applied:                                        │
  │                                                                      │
  │  1. NAMESPACES — ISOLATION (what the process can SEE)               │
  │  ┌────────────┬──────────────────────────────────────────────────┐   │
  │  │ Namespace  │ What it isolates                                 │   │
  │  ├────────────┼──────────────────────────────────────────────────┤   │
  │  │ PID        │ Process IDs (container PID 1 ≠ host PID 1)     │   │
  │  │ NET        │ Network interfaces, IPs, ports, routing tables  │   │
  │  │ MNT        │ Filesystem mount points (own /)                 │   │
  │  │ UTS        │ Hostname (container has its own hostname)       │   │
  │  │ IPC        │ Inter-process communication (shared memory)     │   │
  │  │ USER       │ User/group IDs (root in container ≠ root on    │   │
  │  │            │ host — user namespace mapping)                  │   │
  │  │ CGROUP     │ Cgroup root directory                           │   │
  │  └────────────┴──────────────────────────────────────────────────┘   │
  │                                                                      │
  │  2. CGROUPS — RESOURCE LIMITS (what the process can USE)            │
  │  ┌────────────┬──────────────────────────────────────────────────┐   │
  │  │ Resource   │ What it limits                                   │   │
  │  ├────────────┼──────────────────────────────────────────────────┤   │
  │  │ cpu        │ CPU time allocation (shares, quota, period)     │   │
  │  │ memory     │ RAM limit (OOM kill if exceeded)                │   │
  │  │ blkio      │ Block device I/O rate limiting                  │   │
  │  │ pids       │ Maximum number of processes                     │   │
  │  │ devices    │ Access to specific devices (/dev/*)             │   │
  │  │ net_cls    │ Network packet classification                   │   │
  │  └────────────┴──────────────────────────────────────────────────┘   │
  │                                                                      │
  │  So when you say "docker run -m 512m --cpus=2 myapp":              │
  │  • Docker creates new namespaces (PID, NET, MNT...) → isolation    │
  │  • Docker sets cgroup limits (512MB RAM, 2 CPUs) → resource control│
  │  • The container process is just a regular Linux process with       │
  │    these kernel constraints applied. That's it. No virtualization.  │
  └──────────────────────────────────────────────────────────────────────┘
```

### 16.4 SSH — Secure Shell Deep Dive

```bash
# ──────────────────── SSH KEY AUTHENTICATION ────────────────────
# Generate key pair:
ssh-keygen -t ed25519 -C "chaitanya@simform.com"
# ed25519 = modern, fast, secure (preferred over RSA)
# Creates: ~/.ssh/id_ed25519 (private) + ~/.ssh/id_ed25519.pub (public)

# Copy public key to server:
ssh-copy-id user@server
# or manually:
cat ~/.ssh/id_ed25519.pub | ssh user@server "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

# SSH with key (automatic if key is in default location):
ssh user@server

# SSH with specific key:
ssh -i ~/.ssh/mykey user@server

# ──────────────────── SSH CONFIG FILE ────────────────────
# ~/.ssh/config — saves you typing:
# Host web-prod
#     HostName 10.0.0.50
#     User deploy
#     Port 2222
#     IdentityFile ~/.ssh/deploy_key
#
# Now just: ssh web-prod   (instead of ssh -i ~/.ssh/deploy_key -p 2222 deploy@10.0.0.50)

# ──────────────────── SSH TUNNELING ────────────────────
# Local port forward (access remote service locally):
ssh -L 3306:localhost:3306 user@dbserver
# Now: mysql -h 127.0.0.1 connects to dbserver's MySQL through SSH tunnel

# Remote port forward (expose local service on remote):
ssh -R 8080:localhost:3000 user@remote
# Remote server's port 8080 → your local port 3000

# Dynamic SOCKS proxy:
ssh -D 1080 user@server
# Configure browser to use localhost:1080 as SOCKS proxy
# All traffic tunneled through the SSH connection

# ──────────────────── SSH SECURITY HARDENING ────────────────────
# /etc/ssh/sshd_config — critical settings:
# PermitRootLogin no              ← Disable root SSH (ALWAYS)
# PasswordAuthentication no       ← Keys only (disable password auth)
# PubkeyAuthentication yes        ← Enable key auth
# MaxAuthTries 3                  ← Lock after 3 failures
# Port 2222                       ← Change default port (security by obscurity)
# AllowUsers deploy admin         ← Whitelist specific users
# Protocol 2                      ← Only SSHv2 (v1 is broken)

# After editing:
sudo sshd -t                     # Test config syntax
sudo systemctl restart sshd      # Apply changes
# ⚠️ KEEP YOUR CURRENT SESSION OPEN until you verify you can still log in!
```

### 16.5 Environment Variables & Shell Configuration

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         SHELL STARTUP FILES — WHICH RUNS WHEN?                       │
  │                                                                      │
  │  Login shell (ssh, console login, su -):                             │
  │  1. /etc/profile               ← System-wide (all users)            │
  │  2. ~/.bash_profile            ← User-specific (first found wins:   │
  │     OR ~/.bash_login              .bash_profile > .bash_login >      │
  │     OR ~/.profile                 .profile)                          │
  │                                                                      │
  │  Non-login interactive shell (open terminal in GUI):                 │
  │  1. /etc/bash.bashrc           ← System-wide                        │
  │  2. ~/.bashrc                  ← User-specific                      │
  │                                                                      │
  │  COMMON PATTERN:                                                     │
  │  ~/.bash_profile sources ~/.bashrc:                                  │
  │    if [ -f ~/.bashrc ]; then                                         │
  │        source ~/.bashrc                                              │
  │    fi                                                                │
  │  This way, settings in .bashrc apply to BOTH login and non-login.   │
  │                                                                      │
  │  WHAT GOES WHERE:                                                    │
  │  ~/.bash_profile: environment variables (PATH, EDITOR, etc.)        │
  │  ~/.bashrc: aliases, functions, prompt, shell options                │
  │                                                                      │
  │  Non-interactive shell (scripts):                                    │
  │  NONE of the above run! Scripts start clean.                         │
  │  (Unless BASH_ENV is set — rare.)                                   │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# View all environment variables:
env                            # or: printenv

# Set temporarily (current session only):
export MY_VAR="value"

# Set permanently (add to ~/.bashrc):
echo 'export MY_VAR="value"' >> ~/.bashrc
source ~/.bashrc               # Apply immediately

# Important environment variables:
echo $PATH                     # Command search path
echo $HOME                     # Home directory
echo $USER                     # Current username
echo $SHELL                    # Current shell
echo $PWD                      # Current directory
echo $EDITOR                   # Default editor (vim, nano)
echo $LANG                     # Locale/language
echo $PS1                      # Prompt format string

# PATH manipulation:
export PATH="$PATH:/opt/myapp/bin"     # Append to PATH
export PATH="/opt/myapp/bin:$PATH"     # Prepend to PATH (higher priority)
```

### 16.6 LVM — Logical Volume Manager

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         LVM — FLEXIBLE DISK MANAGEMENT                               │
  │                                                                      │
  │  Problem: Traditional partitions are FIXED size. Resizing means      │
  │  backup → delete → recreate → restore. Terrible for production.     │
  │                                                                      │
  │  LVM adds a layer of abstraction between disks and filesystems:     │
  │                                                                      │
  │  Physical Disks    Physical Volumes    Volume Group    Logical Vols  │
  │  ┌─────────┐      ┌─────────┐                                      │
  │  │ /dev/sda │─────►│ PV sda  │──┐                                  │
  │  └─────────┘      └─────────┘  │    ┌──────────┐   ┌──────────┐   │
  │                                 ├───►│ VG: data │──►│ LV: logs │   │
  │  ┌─────────┐      ┌─────────┐  │    │          │   │ 50GB     │   │
  │  │ /dev/sdb │─────►│ PV sdb  │──┘    │ 200GB    │   ├──────────┤   │
  │  └─────────┘      └─────────┘       │ total    │──►│ LV: data │   │
  │                                      │ pool     │   │ 100GB    │   │
  │  ┌─────────┐      ┌─────────┐       │          │   ├──────────┤   │
  │  │ /dev/sdc │─────►│ PV sdc  │──────►│          │──►│ LV: app  │   │
  │  └─────────┘      └─────────┘       └──────────┘   │ 50GB     │   │
  │                                                      └──────────┘   │
  │  WHY LVM:                                                           │
  │  • Resize volumes LIVE (no unmount needed for extending!)           │
  │  • Span one volume across MULTIPLE disks                            │
  │  • Add new disks to the pool without downtime                       │
  │  • Snapshots (point-in-time copies for backups)                     │
  │  • Thin provisioning (allocate more than you have)                  │
  └──────────────────────────────────────────────────────────────────────┘
```

```bash
# ──────────── LVM Commands (Interview Quick Reference) ────────────
# Physical Volumes:
pvcreate /dev/sdb                      # Initialize disk as PV
pvdisplay                              # Show PV details
pvs                                    # Quick PV summary

# Volume Groups:
vgcreate data_vg /dev/sdb /dev/sdc    # Create VG from PVs
vgdisplay                              # Show VG details
vgs                                    # Quick VG summary
vgextend data_vg /dev/sdd             # Add disk to VG (no downtime!)

# Logical Volumes:
lvcreate -L 50G -n logs_lv data_vg    # Create 50GB LV
lvdisplay                              # Show LV details
lvs                                    # Quick LV summary

# Resize (THE killer feature):
lvextend -L +20G /dev/data_vg/logs_lv  # Add 20GB to LV
resize2fs /dev/data_vg/logs_lv         # Extend ext4 filesystem
# or for xfs:
xfs_growfs /dev/data_vg/logs_lv        # Extend xfs filesystem

# One-liner — extend LV AND filesystem:
lvextend -r -L +20G /dev/data_vg/logs_lv   # -r = resize filesystem too

# Format and mount:
mkfs.ext4 /dev/data_vg/logs_lv
mount /dev/data_vg/logs_lv /var/log
# Add to /etc/fstab for persistence:
echo "/dev/data_vg/logs_lv /var/log ext4 defaults 0 2" >> /etc/fstab
```

### 16.7 SWAP — Virtual Memory on Linux

```bash
# SWAP = disk space used as "overflow" memory when RAM is full.
# Kernel moves inactive pages from RAM to swap (swapping/paging).

# Check swap:
free -h
swapon --show

# Create swap file (modern method — no swap partition needed):
sudo fallocate -l 2G /swapfile        # Create 2GB file
sudo chmod 600 /swapfile               # Restrict permissions
sudo mkswap /swapfile                  # Format as swap
sudo swapon /swapfile                  # Enable swap

# Make permanent:
echo "/swapfile none swap sw 0 0" | sudo tee -a /etc/fstab

# Swappiness — how aggressively the kernel uses swap:
cat /proc/sys/vm/swappiness            # Default: 60
# 0   = Avoid swap (only when RAM is completely full)
# 10  = Minimal swapping (recommended for servers)
# 60  = Default (balanced)
# 100 = Aggressively swap to disk

# Set swappiness (runtime):
sudo sysctl vm.swappiness=10
# Permanent:
echo "vm.swappiness=10" | sudo tee -a /etc/sysctl.conf
```

> **DevOps Pro Tip:** For production servers, set `vm.swappiness=10`. High swappiness causes performance problems — disk I/O is ~1000x slower than RAM. If your server is heavily swapping, add more RAM. Some database servers (Redis, Elasticsearch) recommend `swappiness=1` or even disabling swap entirely.

### 16.8 Interview Curveball Questions

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │  CURVEBALL: "What happens when you type google.com in a browser?"   │
  │                                                                      │
  │  1. Browser checks its DNS cache                                     │
  │  2. OS checks /etc/hosts, then DNS resolver cache                   │
  │  3. DNS query sent to resolver → root → .com → google.com          │
  │  4. IP address returned (e.g., 142.250.182.46)                      │
  │  5. TCP 3-way handshake: SYN → SYN-ACK → ACK                      │
  │  6. TLS handshake (if HTTPS): negotiate encryption                  │
  │  7. HTTP GET / request sent                                          │
  │  8. Server (nginx/load balancer) receives request                   │
  │  9. Backend app processes request, returns HTML                     │
  │  10. Browser receives HTML, parses, requests CSS/JS/images          │
  │  11. Browser renders the page                                        │
  │                                                                      │
  │  DEEP ANSWER adds:                                                   │
  │  • ARP resolves gateway MAC address for the IP packet               │
  │  • HSTS may force HTTPS redirect                                    │
  │  • CDN/Anycast may route to nearest edge server                     │
  │  • TCP congestion control affects initial page load speed            │
  │  • Connection: keep-alive reuses TCP connection for subsequent reqs │
  └──────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────┐
  │  CURVEBALL: "What's the difference between a process and a thread?" │
  │                                                                      │
  │  Process: Independent execution unit with its OWN memory space.     │
  │  Thread: Execution unit that SHARES memory with other threads       │
  │          in the same process.                                        │
  │                                                                      │
  │  Process crash → other processes unaffected.                         │
  │  Thread crash → entire process (all threads) may crash.             │
  │                                                                      │
  │  Inter-process communication: pipes, sockets, shared memory (IPC)   │
  │  Inter-thread communication: shared variables (+ mutexes for safety)│
  │                                                                      │
  │  Creating a process (fork): expensive (copy memory space)           │
  │  Creating a thread (pthread_create): cheap (share memory space)     │
  │                                                                      │
  │  Example:                                                            │
  │  nginx: 1 master PROCESS + N worker PROCESSES (process model)       │
  │  Java app: 1 process + many THREADS (thread model)                  │
  │  Node.js: 1 process, 1 main thread + worker thread pool             │
  └──────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────┐
  │  CURVEBALL: "Explain the /etc/fstab file."                          │
  │                                                                      │
  │  /etc/fstab = filesystem table — defines AUTOMATIC mounts at boot.  │
  │                                                                      │
  │  # <device>           <mount>  <type> <options>       <dump> <pass> │
  │  /dev/sda1            /        ext4   errors=remount-ro 0      1    │
  │  /dev/data_vg/logs_lv /var/log ext4   defaults          0      2    │
  │  UUID=abcd-1234       /boot    ext4   defaults          0      2    │
  │  /swapfile            none     swap   sw                0      0    │
  │  //nas/share          /mnt/nas cifs   credentials=/etc/creds 0 0   │
  │                                                                      │
  │  Columns:                                                            │
  │  1. Device (or UUID — preferred, survives disk reorder)             │
  │  2. Mount point                                                      │
  │  3. Filesystem type                                                  │
  │  4. Mount options (defaults = rw,suid,dev,exec,auto,nouser,async)  │
  │  5. Dump (0 = no backup, 1 = backup with dump command)             │
  │  6. Pass (fsck order: 0=skip, 1=root first, 2=others)             │
  │                                                                      │
  │  ⚠️ A TYPO in fstab can PREVENT BOOT. Always test:                 │
  │  sudo mount -a     ← Mounts everything in fstab (test before reboot)│
  └──────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────┐
  │  CURVEBALL: "What is load average and when is it too high?"         │
  │                                                                      │
  │  Load average = average number of processes in the run queue        │
  │  (running + waiting to run) over 1, 5, and 15 minutes.             │
  │                                                                      │
  │  $ uptime                                                            │
  │  10:30:00 up 5 days, load average: 2.50, 1.75, 1.20                │
  │                                    ↑      ↑      ↑                  │
  │                                  1 min  5 min  15 min               │
  │                                                                      │
  │  RULE OF THUMB:                                                      │
  │  Load average ≤ number of CPU cores → healthy                       │
  │  Load average > CPU cores → overloaded (processes queuing)          │
  │                                                                      │
  │  Example: 4-core server                                              │
  │  Load 2.0 → healthy (2 of 4 cores busy)                            │
  │  Load 4.0 → at capacity (all cores busy, no queuing yet)           │
  │  Load 8.0 → overloaded (4 running + 4 waiting)                     │
  │  Load 50.0 → severe problem (check for I/O wait or fork bombs)     │
  │                                                                      │
  │  High load + low CPU usage → I/O bottleneck (processes waiting     │
  │  for disk/network, counted in load avg but not using CPU)           │
  │  Check with: iostat, vmstat, iotop                                  │
  │                                                                      │
  │  Check cores: nproc  or  lscpu | grep "^CPU(s)"                    │
  └──────────────────────────────────────────────────────────────────────┘
```

### 16.9 TCP 3-Way Handshake and Connection States

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         TCP 3-WAY HANDSHAKE                                          │
  │                                                                      │
  │  Client                                              Server          │
  │    │                                                    │            │
  │    │─── SYN (seq=100) ─────────────────────────────────►│            │
  │    │    "I want to connect"                             │            │
  │    │                                                    │            │
  │    │◄── SYN-ACK (seq=300, ack=101) ────────────────────│            │
  │    │    "OK, I acknowledge your seq, here's mine"      │            │
  │    │                                                    │            │
  │    │─── ACK (seq=101, ack=301) ────────────────────────►│            │
  │    │    "Got it, connection established"                │            │
  │    │                                                    │            │
  │    │◄──────────── DATA TRANSFER ───────────────────────►│            │
  │    │                                                    │            │
  │                                                                      │
  │  TCP 4-WAY TEARDOWN:                                                 │
  │    │─── FIN ──────────────────────────────────────────►│             │
  │    │◄── ACK ──────────────────────────────────────────│             │
  │    │◄── FIN ──────────────────────────────────────────│             │
  │    │─── ACK ──────────────────────────────────────────►│             │
  │    │                                                    │            │
  │    │  TIME_WAIT (client waits ~60s to ensure             │            │
  │    │  all packets are received before fully closing)     │            │
  │                                                                      │
  │  TCP STATE REFERENCE (for ss output):                                │
  │  ┌──────────────┬────────────────────────────────────────────┐       │
  │  │ State         │ Meaning                                    │       │
  │  ├──────────────┼────────────────────────────────────────────┤       │
  │  │ LISTEN        │ Server waiting for connections             │       │
  │  │ SYN_SENT      │ Client sent SYN, waiting for SYN-ACK     │       │
  │  │ SYN_RECV      │ Server received SYN, sent SYN-ACK        │       │
  │  │ ESTABLISHED   │ Connection active, data flowing           │       │
  │  │ FIN_WAIT_1    │ Sent FIN, waiting for ACK                 │       │
  │  │ FIN_WAIT_2    │ Got ACK for FIN, waiting for remote FIN  │       │
  │  │ TIME_WAIT     │ Waiting for stale packets to expire       │       │
  │  │ CLOSE_WAIT    │ Remote closed, app hasn't closed yet ⚠️  │       │
  │  │ LAST_ACK      │ Sent FIN after CLOSE_WAIT, waiting ACK   │       │
  │  │ CLOSED         │ Connection fully terminated               │       │
  │  └──────────────┴────────────────────────────────────────────┘       │
  │                                                                      │
  │  CLOSE_WAIT accumulation = APPLICATION BUG (not closing sockets)   │
  │  TIME_WAIT accumulation = normal for high-traffic servers          │
  └──────────────────────────────────────────────────────────────────────┘
```

### 16.10 Essential Linux Commands — Quick Reference for Interviews

```bash
# ──────────────── SYSTEM INFORMATION ────────────────
uname -a               # Kernel version, architecture
hostnamectl             # Hostname, OS, kernel
lsb_release -a          # Distribution info
cat /etc/os-release     # OS version details
uptime                  # Uptime and load average
nproc                   # Number of CPU cores
free -h                 # Memory usage (RAM + swap)
lscpu                   # CPU details
lsblk                   # Block devices (disks, partitions)
lspci                   # PCI devices (network cards, GPUs)
dmidecode               # Hardware info from BIOS

# ──────────────── PACKAGE MANAGEMENT ────────────────
# Debian/Ubuntu:
apt update                              # Refresh package lists
apt upgrade                             # Upgrade all packages
apt install nginx                       # Install package
apt remove nginx                        # Remove package
apt search nginx                        # Search packages
dpkg -l | grep nginx                    # List installed packages

# RHEL/CentOS:
yum update                              # Upgrade all packages
yum install nginx                       # Install package
yum remove nginx                        # Remove package
yum search nginx                        # Search packages
rpm -qa | grep nginx                    # List installed packages

# ──────────────── ARCHIVING & COMPRESSION ────────────────
# Create tar archive:
tar -cvf archive.tar /path/to/dir      # Create (verbose, file)
tar -czvf archive.tar.gz /path/to/dir  # Create + gzip compress
tar -cjvf archive.tar.bz2 /path/to/dir # Create + bzip2 compress

# Extract:
tar -xvf archive.tar                    # Extract
tar -xzvf archive.tar.gz                # Extract gzipped
tar -xzvf archive.tar.gz -C /dest/     # Extract to specific directory

# View contents without extracting:
tar -tzvf archive.tar.gz

# Flags: c=create, x=extract, t=list, v=verbose, f=file,
#        z=gzip, j=bzip2, C=change directory
```

> **DevOps Pro Tip:** For interviews, know the ONE command that shows you a server's health at a glance: `uptime && free -h && df -h && ss -tlnp`. This gives you: load average, memory usage, disk usage, and listening services — the four pillars of server health.

---

*Batch 6 complete — Phases 11 & 12 (Windows Server & AD + Additional Deep-Dive Topics)*

---

## 17. Production Scenario FAQ — "What Would You Do If…"

> These are real-world production scenarios frequently asked in DevOps / Linux Admin interviews. Each answer follows the structure: **diagnose → fix → prevent**.

---

### Scenario 1: Server Is Slow / Unresponsive

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │  INTERVIEWER: "Users report the server is slow. What do you do?"    │
  │                                                                      │
  │  STEP-BY-STEP DIAGNOSIS:                                             │
  │                                                                      │
  │  1. CHECK LOAD AVERAGE — is the system overloaded?                  │
  │     $ uptime                                                         │
  │     load average: 24.50, 22.10, 18.30   ← WAY over 4 cores!       │
  │                                                                      │
  │  2. CHECK CPU — who's eating it?                                    │
  │     $ top -c                         ← -c shows full command        │
  │     Look at %CPU column. Sort by CPU: press 'P'                     │
  │     Look at %wa (I/O wait) — high = disk bottleneck                 │
  │                                                                      │
  │  3. CHECK MEMORY — is RAM exhausted?                                │
  │     $ free -h                                                        │
  │     If "available" is near zero AND swap is heavily used → OOM risk │
  │                                                                      │
  │  4. CHECK DISK SPACE — is the disk full?                            │
  │     $ df -h                                                          │
  │     100% on / or /var → logs filled the disk                        │
  │     $ du -sh /var/log/* | sort -rh | head                           │
  │                                                                      │
  │  5. CHECK DISK I/O — is the disk saturated?                         │
  │     $ iostat -x 1 3                                                  │
  │     %util near 100% → disk can't keep up                            │
  │     $ iotop                          ← who's doing the I/O?        │
  │                                                                      │
  │  6. CHECK PROCESSES — is something stuck?                            │
  │     $ ps aux --sort=-%mem | head     ← top memory consumers        │
  │     $ ps aux --sort=-%cpu | head     ← top CPU consumers           │
  │     $ dmesg -T | tail -20            ← kernel messages (OOM kills?)│
  │                                                                      │
  │  7. CHECK NETWORK — is the server reachable?                        │
  │     $ ss -s                          ← connection summary           │
  │     $ ss -tlnp                       ← listening ports              │
  │     $ ping -c 3 gateway_ip           ← network connectivity        │
  │                                                                      │
  │  SUMMARY COMMAND (run this first):                                   │
  │  $ uptime; free -h; df -h; ss -s; dmesg -T | tail -5               │
  └──────────────────────────────────────────────────────────────────────┘
```

> **DevOps Pro Tip:** In interviews, the order matters. Always start with the **cheapest checks** first: `uptime` (1 second) tells you more about system health than any other single command. Load average trending UP = problem is getting worse. Load average trending DOWN = problem may be resolving.

---

### Scenario 2: Disk Full — 100% Usage

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │  INTERVIEWER: "The server's disk is 100% full. Fix it."             │
  │                                                                      │
  │  IMMEDIATE DIAGNOSIS:                                                │
  │  $ df -h                                                             │
  │  Filesystem    Size  Used  Avail Use% Mounted on                    │
  │  /dev/sda1     50G   50G   0     100% /                             │
  │                                                                      │
  │  FIND THE CULPRIT:                                                   │
  │  $ du -sh /* 2>/dev/null | sort -rh | head                          │
  │  38G  /var                  ← usually the culprit                   │
  │  $ du -sh /var/* | sort -rh | head                                  │
  │  35G  /var/log              ← logs!                                 │
  │  $ du -sh /var/log/* | sort -rh | head                              │
  │  30G  /var/log/nginx/access.log   ← one huge file!                 │
  │                                                                      │
  │  IMMEDIATE FIX:                                                      │
  │  # Truncate (don't rm!) the log file — keeps the file handle:       │
  │  $ > /var/log/nginx/access.log    ← truncates to 0 bytes           │
  │  # or:                                                               │
  │  $ truncate -s 0 /var/log/nginx/access.log                          │
  │                                                                      │
  │  WHY TRUNCATE, NOT RM?                                               │
  │  If nginx has the file open and you rm it, the SPACE IS NOT FREED  │
  │  until nginx releases the file descriptor!                           │
  │  $ lsof +L1                  ← shows deleted files still held open │
  │  rm creates "(deleted)" files that still consume disk space.        │
  │  Truncate empties the file while keeping the file descriptor valid. │
  │                                                                      │
  │  IF SPACE STILL NOT FREE AFTER RM:                                   │
  │  $ lsof +L1 | grep deleted                                          │
  │  nginx  1234  root  5w  REG  8,1  30GB  0  /var/log/access.log     │
  │  $ systemctl restart nginx   ← release the file descriptors        │
  │  # or: kill -HUP $(pgrep nginx)  ← graceful reload, re-opens logs │
  │                                                                      │
  │  PREVENT RECURRENCE:                                                 │
  │  • Set up logrotate (see §11)                                       │
  │  • Monitor disk usage with alerts at 80%, 90%                       │
  │  • Use LVM for easy expansion (see §16.6)                           │
  │  • Consider separate partition for /var/log                          │
  └──────────────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha:** The `df -h` shows 100% but `du -sh /*` doesn't add up? Check for deleted-but-open files: `lsof +L1`. Also check if inodes are exhausted: `df -i`. You can be at 50% disk space but 100% inodes (too many small files, e.g., a mail queue with millions of tiny files).

---

### Scenario 3: SSH Connection Refused / Cannot Log In

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │  INTERVIEWER: "You can't SSH into a server. Troubleshoot it."       │
  │                                                                      │
  │  SYSTEMATIC DIAGNOSIS (from your machine):                           │
  │                                                                      │
  │  1. Can you reach the server at all?                                │
  │     $ ping -c 3 10.0.0.50                                           │
  │     ✗ No reply → network issue (check cables, route, firewall)     │
  │     ✓ Replies → host is up, problem is SSH-specific                │
  │                                                                      │
  │  2. Is SSH port open?                                                │
  │     $ nc -zv 10.0.0.50 22                                           │
  │     Connection refused → sshd not running or firewall blocking      │
  │     Connection timed out → firewall dropping packets                │
  │     Connection succeeded → SSH is listening, problem is auth        │
  │                                                                      │
  │  3. Is sshd running? (from console/IPMI/cloud console)             │
  │     $ systemctl status sshd                                          │
  │     ● inactive (dead) → $ sudo systemctl start sshd                │
  │     ● failed → $ journalctl -u sshd -e (check error)              │
  │                                                                      │
  │  4. Is firewall blocking?                                            │
  │     $ sudo iptables -L -n | grep 22                                 │
  │     $ sudo ufw status           ← Ubuntu                           │
  │     $ sudo firewall-cmd --list-all  ← CentOS/RHEL                 │
  │     Fix: $ sudo ufw allow 22/tcp                                    │
  │                                                                      │
  │  5. Auth failures?                                                   │
  │     $ ssh -vvv user@10.0.0.50   ← verbose SSH (shows auth steps)  │
  │     Look for:                                                        │
  │     • "Permission denied (publickey)" → key issue                  │
  │     • "Too many authentication failures" → too many keys offered   │
  │     • "Host key verification failed" → known_hosts mismatch         │
  │                                                                      │
  │  6. Check SSH config on server:                                      │
  │     $ cat /etc/ssh/sshd_config | grep -i "permit\|password\|allow" │
  │     Common issues:                                                   │
  │     • PermitRootLogin no (can't login as root — use normal user)   │
  │     • PasswordAuthentication no (need SSH key, password won't work) │
  │     • AllowUsers list doesn't include your username                 │
  │                                                                      │
  │  7. File permissions (key auth):                                     │
  │     ~/.ssh/                → 700 (drwx------)                       │
  │     ~/.ssh/authorized_keys → 600 (-rw-------)                       │
  │     ~/.ssh/id_ed25519      → 600 (-rw-------)                       │
  │     Wrong permissions → SSH silently refuses key auth!              │
  └──────────────────────────────────────────────────────────────────────┘
```

---

### Scenario 4: Website Down — Nginx Not Serving Pages

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │  INTERVIEWER: "The company website is down. It runs on Nginx."      │
  │                                                                      │
  │  DIAGNOSIS FLOWCHART:                                                │
  │                                                                      │
  │  ┌───────────────────────┐                                          │
  │  │ Can you reach the IP? │                                          │
  │  │ curl -I http://IP     │                                          │
  │  └─────┬───────────┬─────┘                                          │
  │     NO │           │ YES                                            │
  │        ▼           ▼                                                │
  │  ┌──────────┐  ┌───────────────────┐                                │
  │  │ Network  │  │ What HTTP code?   │                                │
  │  │ problem  │  └──┬────┬────┬──────┘                                │
  │  │ (ping,   │     │    │    │                                       │
  │  │ route,   │   502  403  404                                       │
  │  │ firewall)│     │    │    │                                       │
  │  └──────────┘     ▼    ▼    ▼                                       │
  │           ┌────────┐ ┌──────┐ ┌──────────┐                          │
  │           │Backend │ │Perms │ │Config /  │                          │
  │           │is dead │ │issue │ │root path │                          │
  │           │(app    │ │      │ │wrong     │                          │
  │           │crashed)│ │      │ │          │                          │
  │           └────────┘ └──────┘ └──────────┘                          │
  │                                                                      │
  │  STEP-BY-STEP:                                                       │
  │                                                                      │
  │  1. Is nginx running?                                                │
  │     $ systemctl status nginx                                         │
  │     Not running → $ sudo systemctl start nginx                      │
  │     Failed to start → check config: $ sudo nginx -t                 │
  │                                                                      │
  │  2. Config syntax error?                                             │
  │     $ sudo nginx -t                                                  │
  │     "test failed" → fix the syntax error shown                      │
  │     Common: missing semicolons, unmatched braces                    │
  │                                                                      │
  │  3. Check error logs:                                                │
  │     $ tail -50 /var/log/nginx/error.log                              │
  │     Look for: "permission denied", "no such file", "connect()       │
  │     failed" (backend down), "too many open files"                   │
  │                                                                      │
  │  4. Port conflict?                                                   │
  │     $ ss -tlnp | grep :80                                            │
  │     Another process on port 80? (Apache leftover is common!)        │
  │     $ sudo fuser -k 80/tcp        ← kill whatever's on port 80     │
  │     $ sudo systemctl start nginx                                     │
  │                                                                      │
  │  5. 502 Bad Gateway (reverse proxy)?                                 │
  │     → Backend app (Node, Python, PHP-FPM) is not running            │
  │     $ systemctl status myapp                                         │
  │     $ curl http://localhost:3000   ← test backend directly         │
  │     Fix: restart the backend app                                     │
  │                                                                      │
  │  6. 403 Forbidden?                                                   │
  │     $ ls -la /var/www/html/                                          │
  │     Files not readable by nginx (www-data user)?                    │
  │     $ sudo chown -R www-data:www-data /var/www/html/                │
  │     $ sudo chmod -R 755 /var/www/html/                               │
  │     Also check: SELinux blocking? $ getenforce                      │
  │                                                                      │
  │  7. DNS not resolving to your server?                                │
  │     $ dig +short www.company.com                                     │
  │     Returns wrong IP? → DNS record needs updating                  │
  │     Returns correct IP? → Problem is on the server side            │
  └──────────────────────────────────────────────────────────────────────┘
```

---

### Scenario 5: A Process Is Consuming 100% CPU

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │  INTERVIEWER: "A process is using 100% CPU. Handle it."             │
  │                                                                      │
  │  1. IDENTIFY the process:                                            │
  │     $ top -c                                                         │
  │     Press 'P' to sort by CPU. Note the PID and command.             │
  │     PID 4523  99.9%  /usr/bin/python3 /opt/app/worker.py            │
  │                                                                      │
  │  2. INVESTIGATE before killing:                                      │
  │     $ ps -p 4523 -o pid,ppid,user,etime,args                        │
  │     → Who started it? How long has it been running?                 │
  │                                                                      │
  │     $ ls -l /proc/4523/fd | wc -l   ← open file descriptors       │
  │     $ cat /proc/4523/status          ← memory, threads, state      │
  │     $ strace -p 4523 -c -t 5        ← what syscalls? (5 seconds)  │
  │                                                                      │
  │  3. Is it SUPPOSED to be doing this?                                 │
  │     • Build/compile jobs → high CPU is normal (but maybe nice it)  │
  │     • Web app worker → 100% is abnormal (infinite loop? bug?)      │
  │     • Backup/compression → normal but may need scheduling           │
  │                                                                      │
  │  4. ACTIONS (in order of preference):                                │
  │                                                                      │
  │     a) NICE IT DOWN (if legitimate but low priority):               │
  │        $ sudo renice +15 -p 4523     ← lower priority              │
  │                                                                      │
  │     b) GRACEFUL STOP (let it clean up):                             │
  │        $ kill -15 4523               ← SIGTERM                     │
  │        $ sleep 5                                                     │
  │        $ ps -p 4523                  ← still running?              │
  │                                                                      │
  │     c) FORCE KILL (if SIGTERM didn't work):                         │
  │        $ kill -9 4523                ← SIGKILL (last resort)       │
  │                                                                      │
  │  5. PREVENT RECURRENCE:                                              │
  │     • cgroups: limit CPU for the service                            │
  │       [Service]                                                      │
  │       CPUQuota=50%                                                   │
  │     • ulimits: limit resources per user                             │
  │     • monitoring: alert on sustained high CPU                       │
  │     • fix the bug if it's an infinite loop!                         │
  └──────────────────────────────────────────────────────────────────────┘
```

---

### Scenario 6: Server Ran Out of Memory — OOM Killer

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │  INTERVIEWER: "The app keeps getting killed. What's happening?"     │
  │                                                                      │
  │  LIKELY CAUSE: OOM Killer (Out Of Memory Killer)                    │
  │                                                                      │
  │  When Linux runs out of RAM + swap, the kernel's OOM Killer         │
  │  selects and kills a process to free memory. It prefers:            │
  │  • Processes using the MOST memory                                  │
  │  • Processes with the highest oom_score                             │
  │                                                                      │
  │  DIAGNOSIS:                                                          │
  │  $ dmesg -T | grep -i "oom\|killed"                                 │
  │  [Mar  3 10:30:00] Out of memory: Killed process 4523 (java)       │
  │                     total-vm:4096000kB, anon-rss:3800000kB          │
  │                                                                      │
  │  $ journalctl -k | grep -i oom                                      │
  │  $ grep -i "oom\|killed" /var/log/syslog                            │
  │                                                                      │
  │  CHECK CURRENT MEMORY STATE:                                         │
  │  $ free -h                                                           │
  │  $ ps aux --sort=-%mem | head -10    ← top memory consumers        │
  │  $ cat /proc/meminfo                 ← detailed memory breakdown   │
  │                                                                      │
  │  SOLUTIONS:                                                          │
  │  1. Increase RAM (obvious but sometimes necessary)                  │
  │  2. Add/increase swap (buys time, not a real fix)                   │
  │  3. Set memory limits in systemd:                                    │
  │     [Service]                                                        │
  │     MemoryMax=2G                     ← hard limit                  │
  │     MemoryHigh=1.5G                  ← throttle point              │
  │  4. Fix the memory leak in the application                          │
  │  5. Protect critical processes:                                      │
  │     $ echo -1000 > /proc/$(pgrep sshd)/oom_score_adj               │
  │     → Makes sshd almost immune to OOM killer                       │
  │     (So you can still SSH in to fix things!)                        │
  │                                                                      │
  │  OOM SCORE:                                                          │
  │  $ cat /proc/<PID>/oom_score        ← current score (higher=more  │
  │                                        likely to be killed)         │
  │  $ cat /proc/<PID>/oom_score_adj    ← adjustment (-1000 to 1000)  │
  │  -1000 = never kill this process                                    │
  │  +1000 = kill this process first                                    │
  └──────────────────────────────────────────────────────────────────────┘
```

---

### Scenario 7: DNS Resolution Failing

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │  INTERVIEWER: "Apps can't resolve DNS names. Troubleshoot."         │
  │                                                                      │
  │  1. Can you resolve ANYTHING?                                        │
  │     $ nslookup google.com                                            │
  │     $ dig google.com                                                 │
  │     ✗ Fails → DNS server is unreachable or down                    │
  │     ✓ Works → only specific names failing (check zone/records)     │
  │                                                                      │
  │  2. Check DNS configuration:                                         │
  │     $ cat /etc/resolv.conf                                           │
  │     nameserver 10.0.0.2    ← is this correct?                      │
  │     nameserver 8.8.8.8     ← fallback to Google DNS                │
  │                                                                      │
  │     ⚠️ On Ubuntu with systemd-resolved:                             │
  │     $ resolvectl status     ← actual DNS config                    │
  │     /etc/resolv.conf may point to 127.0.0.53 (stub resolver)       │
  │                                                                      │
  │  3. Can you reach the DNS server?                                    │
  │     $ ping -c 2 10.0.0.2                                            │
  │     $ dig @10.0.0.2 google.com   ← query specific server          │
  │     Timeout → DNS server is down or firewall blocks UDP 53         │
  │                                                                      │
  │  4. Check /etc/nsswitch.conf:                                        │
  │     hosts: files dns                                                 │
  │     "files" = check /etc/hosts first                                │
  │     "dns" = then query DNS servers                                  │
  │     Missing "dns"? → add it back!                                  │
  │                                                                      │
  │  5. Is it a specific domain?                                         │
  │     $ dig myapp.internal.company.com                                 │
  │     NXDOMAIN → record doesn't exist in DNS                         │
  │     SERVFAIL → DNS server has a problem with that zone             │
  │                                                                      │
  │  6. Flush DNS cache:                                                 │
  │     $ sudo systemd-resolve --flush-caches  ← systemd-resolved     │
  │     $ sudo resolvectl flush-caches          ← newer syntax         │
  │                                                                      │
  │  QUICK FIX (to restore connectivity NOW):                           │
  │  $ echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf           │
  │  (Temporary — will be overwritten by DHCP/Netplan on next renewal) │
  └──────────────────────────────────────────────────────────────────────┘
```

---

### Scenario 8: Nginx Config Deleted — Recovery

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │  INTERVIEWER: "Someone deleted the Nginx config. Recover it."       │
  │                                                                      │
  │  CRITICAL RULE: DO NOT RESTART NGINX!                                │
  │  If nginx is still running, the config is loaded in memory.         │
  │  Restarting = losing it forever.                                     │
  │                                                                      │
  │  RECOVERY STEPS:                                                     │
  │                                                                      │
  │  1. Verify nginx is still running:                                   │
  │     $ systemctl status nginx                                         │
  │     $ ps aux | grep nginx                                            │
  │                                                                      │
  │  2. Dump the running configuration from memory:                      │
  │     $ sudo nginx -T > /tmp/nginx_recovered.conf                     │
  │     nginx -T = Test config + DUMP entire effective configuration    │
  │     This outputs ALL included files, all server blocks, everything. │
  │                                                                      │
  │  3. Recreate the config files:                                       │
  │     $ sudo mkdir -p /etc/nginx/sites-available                       │
  │     $ sudo mkdir -p /etc/nginx/sites-enabled                         │
  │     Parse the nginx -T output to recreate individual files.          │
  │     The output shows "# configuration file /etc/nginx/nginx.conf:"  │
  │     before each file's contents.                                     │
  │                                                                      │
  │  4. Test the restored config:                                        │
  │     $ sudo nginx -t                                                  │
  │                                                                      │
  │  5. Reload (not restart!) to apply:                                  │
  │     $ sudo nginx -s reload                                           │
  │                                                                      │
  │  IF NGINX IS ALREADY STOPPED (config lost):                          │
  │  • Check backups (you DO have backups, right?)                      │
  │  • Check version control (config should be in git!)                 │
  │  • Check /etc/nginx/*.dpkg-old or *.bak files                      │
  │  • Reinstall nginx for default config: apt install --reinstall nginx│
  │  • Check etckeeper if installed (auto-commits /etc changes)         │
  │                                                                      │
  │  PREVENTION:                                                         │
  │  • Keep all configs in version control (Git)                        │
  │  • Use configuration management (Ansible/Puppet)                    │
  │  • Automated backups of /etc/                                       │
  │  • Use etckeeper: auto-commits /etc/ changes to git                 │
  └──────────────────────────────────────────────────────────────────────┘
```

> **DevOps Pro Tip:** This is one of the most asked questions in DevOps interviews. The key insight: **`nginx -T` (capital T)** dumps the running config. This only works while nginx is still running. Lowercase `-t` only tests syntax. Know the difference — it could save your production environment.

---

### Scenario 9: Cannot Start a Service — Port Already in Use

```bash
# SYMPTOM:
$ sudo systemctl start nginx
Job for nginx.service failed. See "systemctl status nginx.service".

$ systemctl status nginx
nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)

# DIAGNOSIS — who's using port 80?
$ sudo ss -tlnp | grep :80
LISTEN  0  511  0.0.0.0:80  0.0.0.0:*  users:(("apache2",pid=1234,fd=4))
# ^ Apache is running on port 80!

# or:
$ sudo lsof -i :80
COMMAND   PID   USER  FD  TYPE DEVICE SIZE/OFF NODE NAME
apache2  1234   root   4u  IPv4  ...        TCP *:http (LISTEN)

# FIX OPTIONS:
# Option 1: Stop the conflicting service
$ sudo systemctl stop apache2
$ sudo systemctl disable apache2    # Prevent it from starting at boot
$ sudo systemctl start nginx

# Option 2: Change nginx to a different port
# In /etc/nginx/sites-available/default:
# listen 8080;

# Option 3: Kill whatever's on that port (aggressive):
$ sudo fuser -k 80/tcp              # Kills ALL processes using port 80
$ sudo systemctl start nginx

# PREVENT: Disable services you don't need
$ sudo systemctl disable apache2
$ sudo apt remove apache2           # If you don't need it at all
```

---

### Scenario 10: Zombie Processes Accumulating

```bash
# SYMPTOM — lots of zombie (Z) processes:
$ ps aux | grep Z
USER  PID  %CPU %MEM VSZ RSS  TTY  STAT  START TIME COMMAND
root  5001 0.0  0.0  0   0    ?    Z     10:00 0:00 [worker] <defunct>
root  5002 0.0  0.0  0   0    ?    Z     10:01 0:00 [worker] <defunct>
root  5003 0.0  0.0  0   0    ?    Z     10:02 0:00 [worker] <defunct>

# COUNT zombies:
$ ps aux | awk '$8 ~ /Z/ {count++} END {print count}'
147

# UNDERSTAND: Zombies can't be killed with kill -9!
# They're already dead. They're waiting for their parent to
# collect their exit status (wait()/waitpid()).

# FIND THE PARENT:
$ ps -eo pid,ppid,stat,cmd | grep Z
5001  3000  Z  [worker] <defunct>
5002  3000  Z  [worker] <defunct>
# Parent PID = 3000

$ ps -p 3000 -o pid,cmd
3000  /usr/bin/python3 /opt/app/manager.py

# FIX:
# Option 1: Signal the parent to reap children
$ kill -SIGCHLD 3000      # Remind parent to call wait()

# Option 2: Restart the parent process
$ kill -15 3000           # SIGTERM the parent
# When parent dies, zombies are re-parented to PID 1 (systemd),
# which WILL reap them.

# Option 3: Fix the application (it should call wait() on children!)

# PREVENTION:
# The app developer must properly handle SIGCHLD and call wait()/waitpid()
# In shell scripts: use "wait" command after background processes
```

---

## 18. Interview Prep — Rapid-Fire Q&A Bank

> These are short, direct questions and answers designed for interview practice. Read the question, try to answer, then check.

### 18.1 Linux Fundamentals

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │  Q: What is the difference between a hard link and a soft link?     │
  │  A: Hard link = same inode, same data. Deleting original doesn't   │
  │     affect hard link. Can't cross filesystems. Can't link dirs.    │
  │     Soft link = separate inode, points to PATH. Deleting original  │
  │     breaks the link (dangling). Can cross filesystems. Can link    │
  │     directories.                                                    │
  │                                                                      │
  │  Q: What is an inode?                                                │
  │  A: An inode stores file METADATA: permissions, owner, size,       │
  │     timestamps, and pointers to data blocks. It does NOT store     │
  │     the filename. The filename-to-inode mapping is in the          │
  │     directory entry.                                                 │
  │                                                                      │
  │  Q: What happens when you run "rm" on a file?                       │
  │  A: rm removes the directory entry (filename→inode link) and       │
  │     decrements the inode's link count. If link count reaches 0     │
  │     AND no process has the file open, the inode and data blocks    │
  │     are freed. If a process still has it open, space is NOT freed  │
  │     until the process closes the file.                              │
  │                                                                      │
  │  Q: What is the difference between /etc/passwd and /etc/shadow?    │
  │  A: /etc/passwd: readable by all, contains username, UID, GID,     │
  │     home dir, shell. Password field shows 'x' (means check shadow).│
  │     /etc/shadow: readable by root only, contains the actual        │
  │     password HASH, password aging info, account expiration.         │
  │                                                                      │
  │  Q: Explain the Linux boot process.                                 │
  │  A: BIOS/UEFI → GRUB2 bootloader → Linux kernel loads →          │
  │     initramfs (temp rootfs, loads drivers) → real root mounted →  │
  │     systemd (PID 1) → starts services → login prompt.             │
  │                                                                      │
  │  Q: What is the sticky bit?                                         │
  │  A: When set on a directory (chmod +t), only the file OWNER        │
  │     (or root) can delete/rename files in that directory, even      │
  │     if others have write permission. Classic example: /tmp          │
  │     (drwxrwxrwt). Prevents users from deleting each other's files. │
  │                                                                      │
  │  Q: What does "chmod 755" mean?                                     │
  │  A: 7=rwx (owner), 5=r-x (group), 5=r-x (others).                │
  │     Owner: read+write+execute. Group and others: read+execute.     │
  │     Binary: 111 101 101.                                            │
  │                                                                      │
  │  Q: What is umask 022?                                               │
  │  A: Default umask. Files created with 666-022=644 (rw-r--r--).    │
  │     Directories created with 777-022=755 (rwxr-xr-x).             │
  │     It REMOVES permissions from the default.                        │
  └──────────────────────────────────────────────────────────────────────┘
```

### 18.2 Process Management & Signals

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │  Q: Difference between kill -9 and kill -15?                        │
  │  A: kill -15 (SIGTERM): asks the process to terminate gracefully.  │
  │     Process can catch it, clean up (close files, save state), then │
  │     exit. ALWAYS try this first.                                    │
  │     kill -9 (SIGKILL): kernel forcefully kills the process.        │
  │     Cannot be caught or ignored. No cleanup. Data may be lost.     │
  │     Use only as last resort.                                        │
  │                                                                      │
  │  Q: What is a zombie process?                                       │
  │  A: A process that has finished executing but still has an entry    │
  │     in the process table. The parent hasn't read its exit status   │
  │     (hasn't called wait()). Shows as 'Z' in ps. Can't be killed   │
  │     with kill -9 (already dead). Fix: signal the parent or restart │
  │     the parent.                                                     │
  │                                                                      │
  │  Q: What is a daemon?                                                │
  │  A: A background process that runs without a terminal. Typically    │
  │     started at boot, runs continuously, provides services.          │
  │     Examples: sshd, nginx, crond. Convention: name ends in 'd'.    │
  │                                                                      │
  │  Q: What does nohup do?                                              │
  │  A: nohup runs a command immune to hangup (SIGHUP) signals.       │
  │     When you close your terminal, SIGHUP is sent to all child      │
  │     processes. nohup prevents that, so the process survives.       │
  │     Output goes to nohup.out. Usage: nohup ./script.sh &           │
  │                                                                      │
  │  Q: Difference between & and nohup?                                 │
  │  A: & puts a process in background (still attached to terminal).   │
  │     Close terminal → process gets SIGHUP → dies.                   │
  │     nohup ignores SIGHUP → survives terminal close.               │
  │     Best: nohup command &  (background + survive terminal close)   │
  │                                                                      │
  │  Q: What is /proc?                                                   │
  │  A: A virtual filesystem (doesn't exist on disk). The kernel       │
  │     exposes process and system information as files.                │
  │     /proc/<PID>/status = process info                               │
  │     /proc/cpuinfo = CPU info                                        │
  │     /proc/meminfo = memory info                                     │
  │     /proc/sys/ = tunable kernel parameters (sysctl)                │
  └──────────────────────────────────────────────────────────────────────┘
```

### 18.3 Networking

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │  Q: What is the difference between TCP and UDP?                     │
  │  A: TCP: connection-oriented, reliable, ordered delivery,           │
  │     3-way handshake, error checking, flow control. Slower.         │
  │     Used for: HTTP, SSH, FTP, SMTP, databases.                     │
  │     UDP: connectionless, no guarantee of delivery or order,         │
  │     no handshake, minimal overhead. Faster.                         │
  │     Used for: DNS, DHCP, video streaming, VoIP, gaming.            │
  │                                                                      │
  │  Q: What are well-known ports?                                      │
  │  A: 22=SSH, 53=DNS, 80=HTTP, 443=HTTPS, 25=SMTP, 110=POP3,       │
  │     143=IMAP, 3306=MySQL, 5432=PostgreSQL, 6379=Redis,             │
  │     3389=RDP, 389=LDAP, 636=LDAPS, 8080=HTTP alt.                 │
  │                                                                      │
  │  Q: Explain the TCP 3-way handshake.                                │
  │  A: Client → SYN → Server (I want to connect)                     │
  │     Server → SYN-ACK → Client (OK, I acknowledge)                 │
  │     Client → ACK → Server (Connection established)                 │
  │     Now data can flow bidirectionally.                               │
  │                                                                      │
  │  Q: What is the difference between a firewall and a security group?│
  │  A: Firewall: software/hardware that filters traffic based on      │
  │     rules (iptables, ufw, firewalld — host-level).                 │
  │     Security Group: cloud-level firewall (AWS/Azure) applied to    │
  │     instances. Stateful (if you allow inbound, return traffic is   │
  │     auto-allowed). Operates at the hypervisor level.               │
  │                                                                      │
  │  Q: What does "netstat -tlnp" show?                                 │
  │  A: t=TCP, l=listening, n=numeric (no DNS resolution),             │
  │     p=show process. Shows all TCP ports the server is listening on │
  │     and which process owns each port.                               │
  │     Modern equivalent: ss -tlnp                                     │
  │                                                                      │
  │  Q: What is NAT?                                                     │
  │  A: Network Address Translation. Translates private IPs             │
  │     (192.168.x.x, 10.x.x.x) to public IPs for internet access.   │
  │     Your home router does this — many devices share one public IP. │
  │     Types: SNAT (source), DNAT (destination), PAT (port-based).    │
  │                                                                      │
  │  Q: Private IP ranges?                                               │
  │  A: 10.0.0.0/8 (10.0.0.0 – 10.255.255.255)                       │
  │     172.16.0.0/12 (172.16.0.0 – 172.31.255.255)                   │
  │     192.168.0.0/16 (192.168.0.0 – 192.168.255.255)                │
  └──────────────────────────────────────────────────────────────────────┘
```

### 18.4 Nginx & Web Servers

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │  Q: What is a reverse proxy?                                        │
  │  A: A server that sits in FRONT of backend servers and forwards    │
  │     client requests to them. Clients don't know about backends.    │
  │     Benefits: load balancing, SSL termination, caching, security   │
  │     (hide backend IPs), compression.                                │
  │     Nginx, HAProxy, Apache (mod_proxy) can act as reverse proxies. │
  │                                                                      │
  │  Q: Nginx location block priority order?                            │
  │  A: 1. Exact match: location = /path { } (highest)                │
  │     2. Preferential prefix: location ^~ /path { }                  │
  │     3. Regex (first match): location ~ or ~* /pattern { }         │
  │     4. Standard prefix: location /path { } (longest match wins)   │
  │     5. Default: location / { } (lowest)                            │
  │                                                                      │
  │  Q: What is the difference between nginx reload and restart?       │
  │  A: Reload (nginx -s reload): master reads new config, starts     │
  │     new workers, old workers finish current requests then die.     │
  │     ZERO downtime. Use for config changes.                          │
  │     Restart (systemctl restart nginx): full stop then start.       │
  │     Brief downtime. Connections are dropped. Use only when needed. │
  │                                                                      │
  │  Q: What does "nginx -t" do?                                        │
  │  A: Tests config syntax WITHOUT applying it. Always run before     │
  │     reload to catch errors. "nginx -T" (capital T) dumps the       │
  │     full running configuration — critical for recovery.            │
  │                                                                      │
  │  Q: 502 Bad Gateway vs 504 Gateway Timeout?                        │
  │  A: 502 = nginx contacted the backend but got an INVALID response  │
  │     (backend crashed, wrong protocol, connection refused).          │
  │     504 = nginx contacted the backend but it DIDN'T RESPOND in    │
  │     time (backend is overloaded, hanging, or too slow).            │
  │     502 → check if backend is running.                             │
  │     504 → check backend performance, increase proxy_read_timeout.  │
  │                                                                      │
  │  Q: How does nginx handle 10,000 concurrent connections?           │
  │  A: Event-driven, non-blocking I/O. One worker process handles    │
  │     thousands of connections using epoll (Linux) — NOT one thread  │
  │     per connection like Apache (prefork). Workers never block      │
  │     waiting for I/O; they process whatever's ready.                │
  └──────────────────────────────────────────────────────────────────────┘
```

### 18.5 Windows Server & Active Directory

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │  Q: What is Active Directory?                                       │
  │  A: A Microsoft directory service that provides centralized         │
  │     authentication and authorization. It stores user accounts,     │
  │     computer accounts, groups, and policies. One account = access  │
  │     to all domain-joined resources. Uses LDAP and Kerberos.        │
  │                                                                      │
  │  Q: What is a Domain Controller?                                    │
  │  A: A Windows Server running AD DS (Active Directory Domain        │
  │     Services). It stores the AD database (NTDS.dit), handles       │
  │     authentication (Kerberos), and replicates with other DCs.      │
  │     Minimum 2 DCs for redundancy.                                   │
  │                                                                      │
  │  Q: What is the difference between an OU and a Group?              │
  │  A: OU = organizational container for applying GPOs and delegating │
  │     administration. A user is IN one OU.                            │
  │     Group = collection of users/computers for assigning PERMISSIONS.│
  │     A user can be in MANY groups. GPOs are linked to OUs, not      │
  │     groups. Groups control access; OUs control management.          │
  │                                                                      │
  │  Q: What is GPO processing order?                                   │
  │  A: LSDOU — Local → Site → Domain → OU.                          │
  │     Last applied wins. OU policies override Domain policies.       │
  │     "Enforced" flag prevents child OUs from overriding.            │
  │     "Block Inheritance" on an OU blocks parent GPOs.               │
  │     Enforced beats Block Inheritance.                                │
  │                                                                      │
  │  Q: What is Kerberos?                                                │
  │  A: Authentication protocol used by AD. User logs in → DC issues  │
  │     a Ticket Granting Ticket (TGT). User presents TGT to access   │
  │     services, getting Service Tickets. Password never sent over    │
  │     the network after initial auth. Time-sensitive (clocks must    │
  │     be within 5 minutes).                                           │
  │                                                                      │
  │  Q: Why does AD need DNS?                                            │
  │  A: Clients find Domain Controllers via DNS SRV records             │
  │     (_ldap._tcp.dc._msdcs.domain.com). Without DNS, clients       │
  │     can't locate a DC → can't authenticate → can't log in.       │
  │     That's why DCs typically also run the DNS Server role.          │
  │                                                                      │
  │  Q: What are FSMO roles?                                             │
  │  A: 5 special single-master roles: Schema Master, Domain Naming    │
  │     Master (forest-wide), RID Master, PDC Emulator, Infrastructure │
  │     Master (domain-wide). PDC Emulator is most critical — handles  │
  │     password changes, time sync, and GPO editing.                   │
  │                                                                      │
  │  Q: What is "gpupdate /force"?                                      │
  │  A: Forces immediate re-application of all Group Policies           │
  │     (both Computer and User). Without it, GPOs refresh every       │
  │     ~90 minutes (with random offset). Useful after making          │
  │     GPO changes that you need applied immediately.                  │
  └──────────────────────────────────────────────────────────────────────┘
```

### 18.6 Shell Scripting & Automation

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │  Q: What does "set -euo pipefail" do?                               │
  │  A: -e: Exit on any command failure (non-zero exit code).          │
  │     -u: Error on undefined variables (prevents typo bugs).         │
  │     -o pipefail: Pipeline fails if ANY command in it fails         │
  │     (not just the last one). Essential for reliable scripts.       │
  │                                                                      │
  │  Q: Difference between $* and $@?                                   │
  │  A: Unquoted: identical. Quoted: "$*" = all args as ONE string.   │
  │     "$@" = each arg as SEPARATE strings (preserves word splitting).│
  │     Almost always use "$@".                                         │
  │                                                                      │
  │  Q: Difference between ' ' and " "?                                 │
  │  A: Single quotes: LITERAL (no expansion). 'Hello $USER' → Hello  │
  │     $USER. Double quotes: EXPANDED. "Hello $USER" → Hello john.   │
  │     Use single quotes for exact strings; double for variables.     │
  │                                                                      │
  │  Q: What is a here document (heredoc)?                               │
  │  A: Multi-line string input using <<DELIMITER:                      │
  │     cat <<EOF                                                        │
  │     Hello $USER                                                      │
  │     Today is $(date)                                                 │
  │     EOF                                                              │
  │     Variables are expanded. Use <<'EOF' to prevent expansion.       │
  │                                                                      │
  │  Q: How do you schedule a recurring task?                            │
  │  A: cron (traditional): crontab -e, define schedule with            │
  │     minute hour day month weekday syntax.                           │
  │     systemd timer (modern): create .timer + .service unit files.   │
  │     Timers offer: logging (journalctl), dependency management,     │
  │     randomized delays, missed-run handling.                         │
  │                                                                      │
  │  Q: What does logrotate do?                                          │
  │  A: Automatically rotates, compresses, and removes old log files   │
  │     to prevent disk space exhaustion. Config in /etc/logrotate.d/. │
  │     Key directives: rotate N, daily/weekly, compress, copytruncate │
  │     (for apps that don't reopen log files), postrotate scripts.    │
  │                                                                      │
  │  Q: Difference between systemd Type=simple and Type=forking?       │
  │  A: simple: systemd considers the service started immediately      │
  │     (main process IS the service). For modern apps that stay in    │
  │     foreground. forking: service forks a child and parent exits.   │
  │     systemd waits for the fork, then tracks the child. For legacy  │
  │     daemons that daemonize themselves.                              │
  └──────────────────────────────────────────────────────────────────────┘
```

### 18.7 Containers & Docker

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │  Q: What is the difference between a VM and a container?           │
  │  A: VM = full OS + kernel on virtual hardware (hypervisor).        │
  │     Container = process with isolated namespaces sharing the       │
  │     host kernel. VMs: heavy, minutes to boot, strong isolation.    │
  │     Containers: lightweight, seconds to start, process isolation.  │
  │                                                                      │
  │  Q: What are namespaces and cgroups?                                │
  │  A: Namespaces = ISOLATION (what a process can see).               │
  │     PID, NET, MNT, UTS, IPC, USER namespaces.                     │
  │     Cgroups = RESOURCE LIMITS (what a process can use).            │
  │     CPU, memory, disk I/O, network, PID count limits.              │
  │     Containers = namespaces + cgroups. That's the whole secret.    │
  │                                                                      │
  │  Q: What is a Docker image layer?                                   │
  │  A: Each Dockerfile instruction creates a read-only layer.          │
  │     Layers are cached and shared between images. If you change     │
  │     one layer, only that layer and above are rebuilt. This is why  │
  │     you COPY package.json before COPY . . — dependency install     │
  │     is cached unless package.json changes.                          │
  │                                                                      │
  │  Q: Difference between CMD and ENTRYPOINT in Dockerfile?           │
  │  A: CMD: default command, easily overridden by docker run args.    │
  │     ENTRYPOINT: the executable, harder to override.                 │
  │     Combined: ENTRYPOINT ["python"] CMD ["app.py"]                 │
  │     → runs "python app.py" by default                              │
  │     → docker run myimage test.py → runs "python test.py"          │
  │                                                                      │
  │  Q: What is docker-compose?                                         │
  │  A: Tool to define and run multi-container applications using a    │
  │     YAML file (docker-compose.yml). Define services, networks,     │
  │     volumes. One command (docker compose up) starts everything.    │
  │     Great for local development with multiple services              │
  │     (app + database + cache + message queue).                       │
  │                                                                      │
  │  Q: How do you persist data in Docker?                              │
  │  A: Volumes (docker volume create / -v mydata:/app/data).          │
  │     Managed by Docker, survives container removal.                  │
  │     Bind mounts (-v /host/path:/container/path) for development.   │
  │     Without volumes, ALL data is lost when container is removed.   │
  └──────────────────────────────────────────────────────────────────────┘
```

### 18.8 DevOps Concepts & Culture

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │  Q: What is CI/CD?                                                   │
  │  A: CI (Continuous Integration): developers merge code frequently  │
  │     (daily). Each merge triggers automated build + tests.          │
  │     Catch bugs early. Tools: Jenkins, GitHub Actions, GitLab CI.   │
  │     CD (Continuous Delivery): code is always in a deployable state.│
  │     Deploy to staging automatically, production with one click.    │
  │     CD (Continuous Deployment): auto-deploy to production after    │
  │     all tests pass. No manual approval. Requires mature testing.   │
  │                                                                      │
  │  Q: What is Infrastructure as Code (IaC)?                           │
  │  A: Managing infrastructure (servers, networks, databases) through │
  │     code files instead of manual setup. Benefits: version control, │
  │     reproducibility, automation, documentation. Tools: Terraform,  │
  │     Ansible, CloudFormation, Pulumi.                                │
  │                                                                      │
  │  Q: What is the difference between Ansible and Terraform?          │
  │  A: Terraform: PROVISIONING infrastructure (create VMs, networks,  │
  │     databases). Declarative. Tracks state.                          │
  │     Ansible: CONFIGURING infrastructure (install software, copy    │
  │     files, manage services). Procedural (playbooks). Agentless     │
  │     (uses SSH). They complement each other.                         │
  │                                                                      │
  │  Q: What is a load balancer?                                        │
  │  A: Distributes incoming traffic across multiple backend servers.  │
  │     Benefits: high availability, scalability, health checks        │
  │     (remove unhealthy servers). Types: Layer 4 (TCP — fast) vs    │
  │     Layer 7 (HTTP — content-based routing). Examples: nginx,      │
  │     HAProxy, AWS ALB/NLB, Azure LB.                                │
  │                                                                      │
  │  Q: What is the 12-Factor App methodology?                          │
  │  A: 12 best practices for cloud-native apps:                        │
  │     Key ones: store config in environment variables, treat backing │
  │     services as attached resources, keep dev/staging/prod as       │
  │     similar as possible, treat logs as event streams, run admin    │
  │     tasks as one-off processes.                                     │
  │                                                                      │
  │  Q: What is a microservices architecture?                            │
  │  A: Application split into small, independent services, each       │
  │     running in its own process, communicating via APIs (HTTP/gRPC).│
  │     Each service can be deployed, scaled, and updated independently.│
  │     Contrast with monolith (one big application).                  │
  │     Containers + orchestration (Kubernetes) make microservices      │
  │     practical.                                                       │
  └──────────────────────────────────────────────────────────────────────┘
```

### 18.9 Final Quick-Fire Round — One-Liners

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │  Q: Find all .log files larger than 100MB:                          │
  │  A: find / -name "*.log" -size +100M 2>/dev/null                   │
  │                                                                      │
  │  Q: Count lines in a file:                                          │
  │  A: wc -l filename                                                   │
  │                                                                      │
  │  Q: Find which process is using a specific port:                    │
  │  A: ss -tlnp | grep :80  or  lsof -i :80                          │
  │                                                                      │
  │  Q: Check if a remote port is open:                                 │
  │  A: nc -zv hostname 80  or  telnet hostname 80                     │
  │                                                                      │
  │  Q: Search for a string in all files recursively:                   │
  │  A: grep -rn "pattern" /path/                                      │
  │                                                                      │
  │  Q: Show the top 10 largest files in a directory:                   │
  │  A: du -ah /path | sort -rh | head -10                             │
  │                                                                      │
  │  Q: Watch a log file in real-time:                                  │
  │  A: tail -f /var/log/syslog                                         │
  │                                                                      │
  │  Q: Replace a string in all files:                                   │
  │  A: sed -i 's/old/new/g' *.conf                                    │
  │     or: find . -name "*.conf" -exec sed -i 's/old/new/g' {} +     │
  │                                                                      │
  │  Q: Kill all processes by name:                                      │
  │  A: pkill process_name  or  killall process_name                   │
  │                                                                      │
  │  Q: Check who is logged in:                                          │
  │  A: who  or  w  (w shows what they're doing too)                   │
  │                                                                      │
  │  Q: Add a user to a group:                                           │
  │  A: sudo usermod -aG groupname username                             │
  │     ⚠️ -a = append. Without -a, user is REMOVED from all other    │
  │     groups!                                                          │
  │                                                                      │
  │  Q: Check Linux kernel version:                                      │
  │  A: uname -r                                                         │
  │                                                                      │
  │  Q: Show mounted filesystems:                                        │
  │  A: mount | column -t  or  df -hT  or  lsblk -f                   │
  │                                                                      │
  │  Q: Find files modified in the last 24 hours:                       │
  │  A: find /path -mtime -1                                            │
  │                                                                      │
  │  Q: Create a compressed backup of a directory:                      │
  │  A: tar -czvf backup_$(date +%Y%m%d).tar.gz /path/to/dir          │
  │                                                                      │
  │  Q: Check if a service will start at boot:                          │
  │  A: systemctl is-enabled servicename                                │
  │                                                                      │
  │  Q: List all open files by a process:                                │
  │  A: lsof -p PID                                                      │
  │                                                                      │
  │  Q: Show routing table:                                              │
  │  A: ip route  or  route -n  or  netstat -rn                        │
  │                                                                      │
  │  Q: What command checks disk inode usage?                           │
  │  A: df -i                                                            │
  │                                                                      │
  │  Q: Difference between "su" and "su -"?                             │
  │  A: su: switch user, keep current environment (PATH, HOME, etc.)   │
  │     su -: switch user AND load their full environment               │
  │     (login shell — runs their .bash_profile). Always prefer su -.  │
  └──────────────────────────────────────────────────────────────────────┘
```

---

## 🎓 Conclusion — Interview Preparation Strategy

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         YOUR INTERVIEW PREPARATION ROADMAP                           │
  │                                                                      │
  │  TIER 1 — MUST KNOW (asked in every interview):                     │
  │  ✅ Linux boot process (§5)                                         │
  │  ✅ File permissions, umask, sticky bit (§10)                       │
  │  ✅ Process management, signals, kill -9 vs -15 (§8)               │
  │  ✅ Networking basics — TCP/UDP, ports, DNS (§13)                   │
  │  ✅ SSH key auth and tunneling (§16.4)                              │
  │  ✅ Basic shell scripting (§11)                                     │
  │  ✅ systemctl, journalctl (§12)                                     │
  │  ✅ Troubleshooting "server is slow" (§17, Scenario 1)             │
  │                                                                      │
  │  TIER 2 — FREQUENTLY ASKED (differentiates candidates):            │
  │  ✅ Inodes, hard links vs soft links (§6)                           │
  │  ✅ grep/sed/awk text processing (§9)                               │
  │  ✅ Nginx reverse proxy, location blocks (§14)                      │
  │  ✅ Containers vs VMs, Docker basics (§16.1-16.3)                  │
  │  ✅ Active Directory, GPOs, FSMO roles (§15)                        │
  │  ✅ Disk full troubleshooting, deleted-but-open files (§17, Sc.2)  │
  │  ✅ LVM basics (§16.6)                                              │
  │                                                                      │
  │  TIER 3 — ADVANCED (senior-level / impressive answers):            │
  │  ✅ fork/exec internals, /proc filesystem (§1, §8)                 │
  │  ✅ initramfs, systemd targets (§5)                                 │
  │  ✅ Namespaces & cgroups (§16.3)                                    │
  │  ✅ TCP connection states (§16.9)                                    │
  │  ✅ OOM killer, oom_score (§17, Scenario 6)                        │
  │  ✅ FSMO roles deep dive (§15.9)                                    │
  │  ✅ The "google.com in browser" answer (§16.8)                      │
  │                                                                      │
  │  HOW TO USE THIS HANDBOOK:                                           │
  │  1. Read each section for understanding (don't memorize commands)  │
  │  2. Practice scenarios in a lab (VM or cloud instance)              │
  │  3. Use the FAQ section (§17) for mock interviews                  │
  │  4. Use the Q&A bank (§18) for rapid-fire practice                 │
  │  5. Explain concepts OUT LOUD — if you can teach it, you know it   │
  │                                                                      │
  │  GOLDEN RULE FOR INTERVIEWS:                                         │
  │  Don't just give the COMMAND — explain the THINKING.                │
  │  "I'd first check uptime because load average tells me..."         │
  │  shows more than "I'd run top."                                     │
  │                                                                      │
  │  Good luck! 🚀                                                       │
  └──────────────────────────────────────────────────────────────────────┘
```

---

*Batch 7 complete — Production Scenario FAQ + Interview Prep Q&A Bank*

---

## 19. Linux User Login Troubleshooting — "Users Can't Log In"

### 19.1 The Login Process — What Happens Behind the Scenes

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         LINUX LOGIN FLOW — WHAT THE OS DOES                          │
  │                                                                      │
  │  User types username + password at console, SSH, or GUI              │
  │                                                                      │
  │  ┌──────────┐   ┌─────────┐   ┌──────────┐   ┌───────────────┐     │
  │  │ getty /   │──►│ login / │──►│  PAM     │──►│ Shell session │     │
  │  │ sshd     │   │ sshd    │   │ Stack    │   │ (bash, etc.)  │     │
  │  └──────────┘   └────┬────┘   └────┬─────┘   └───────────────┘     │
  │                      │             │                                 │
  │                      ▼             ▼                                 │
  │               ┌──────────┐   ┌──────────────────────────────┐       │
  │               │/etc/passwd│   │ PAM modules check:           │       │
  │               │ Lookup    │   │ 1. pam_unix → /etc/shadow   │       │
  │               │ username  │   │ 2. pam_faillock → lockout   │       │
  │               │ → UID,   │   │ 3. pam_nologin → /etc/nologin│      │
  │               │   GID,    │   │ 4. pam_limits → ulimits     │       │
  │               │   home,   │   │ 5. pam_motd → login message │       │
  │               │   shell   │   └──────────────────────────────┘       │
  │               └──────────┘                                           │
  │                                                                      │
  │  IF ANY STEP FAILS → LOGIN DENIED                                   │
  │                                                                      │
  │  THE KEY FILES:                                                      │
  │  ┌────────────────┬──────────────────────────────────────────┐       │
  │  │ /etc/passwd    │ User account info (readable by all)      │       │
  │  │ /etc/shadow    │ Password hashes (readable by root only)  │       │
  │  │ /etc/group     │ Group memberships                        │       │
  │  │ /etc/gshadow   │ Group passwords (rare)                   │       │
  │  │ /etc/nologin   │ If exists → blocks ALL non-root logins  │       │
  │  │ /etc/securetty │ Terminals where root can log in          │       │
  │  │ /etc/pam.d/    │ PAM configuration files per service      │       │
  │  │ /etc/security/ │ PAM module configs (limits, access)      │       │
  │  └────────────────┴──────────────────────────────────────────┘       │
  └──────────────────────────────────────────────────────────────────────┘
```

### 19.2 /etc/passwd — Deep Dive

```bash
# /etc/passwd format — 7 colon-separated fields:
# username:x:UID:GID:comment:home_directory:shell

# Example:
john:x:1001:1001:John Smith:/home/john:/bin/bash
#  │   │  │    │     │          │          │
#  │   │  │    │     │          │          └── Login shell
#  │   │  │    │     │          └── Home directory
#  │   │  │    │     └── GECOS (comment/full name)
#  │   │  │    └── Primary GID
#  │   │  └── UID
#  │   └── 'x' means password is in /etc/shadow
#  └── Username

# IMPORTANT FIELDS THAT BREAK LOGIN:
# 1. Shell field:
#    /bin/bash        → normal login ✅
#    /usr/sbin/nologin → login BLOCKED ❌ (shows "This account is 
#                        currently not available")
#    /bin/false        → login BLOCKED ❌ (silent failure)
#
# 2. The 'x' in password field:
#    x        → password is in /etc/shadow (normal) ✅
#    *        → account DISABLED ❌
#    (empty)  → NO PASSWORD REQUIRED ⚠️ (security risk!)

# Quick check — does the user exist?
grep "^john:" /etc/passwd
id john
# uid=1001(john) gid=1001(john) groups=1001(john),27(sudo)

# Check what shell is assigned:
getent passwd john | cut -d: -f7
```

### 19.3 /etc/shadow — The Password File

```bash
# /etc/shadow format — 9 colon-separated fields:
# username:password_hash:lastchanged:min:max:warn:inactive:expire:reserved

# Example:
john:$6$xyz...hash...:19784:0:99999:7:::
#  │       │             │   │   │    │ │ │
#  │       │             │   │   │    │ │ └── Reserved
#  │       │             │   │   │    │ └── Account expiration (days since epoch)
#  │       │             │   │   │    └── Days of warning before expiry
#  │       │             │   │   └── Maximum days between password changes
#  │       │             │   └── Minimum days between password changes
#  │       │             └── Last password change (days since Jan 1, 1970)
#  │       └── Password hash ($6$ = SHA-512, $y$ = yescrypt)
#  └── Username

# PASSWORD HASH FIELD — WHAT EACH VALUE MEANS:
#  $6$salt$hash    → Valid SHA-512 password ✅
#  $y$salt$hash    → Valid yescrypt password ✅ (modern distros)
#  !!              → Password NEVER SET ❌ (account locked)
#  !               → Account LOCKED ❌
#  !$6$salt$hash   → Account LOCKED (! prepended to valid hash) ❌
#  *               → Account DISABLED (no password login possible) ❌
#  (empty)         → NO PASSWORD ⚠️ (anyone can login!)
```

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         /etc/shadow — PASSWORD FIELD CHEAT SHEET                     │
  │                                                                      │
  │  ┌─────────────────────┬───────────────────────────────────────┐     │
  │  │ Value               │ Meaning                               │     │
  │  ├─────────────────────┼───────────────────────────────────────┤     │
  │  │ $6$salt$hash        │ Valid password (SHA-512)              │     │
  │  │ $y$j9T$salt$hash    │ Valid password (yescrypt — Ubuntu 22+)│    │
  │  │ $5$salt$hash        │ Valid password (SHA-256, less common) │     │
  │  │ !!                  │ Password never set (can't login)      │     │
  │  │ !                   │ Account locked by admin               │     │
  │  │ !$6$salt$hash       │ Locked (! prepended = locked)        │     │
  │  │ *                   │ Account disabled (system accounts)    │     │
  │  │ (empty)             │ NO password! Anyone can login ⚠️     │     │
  │  └─────────────────────┴───────────────────────────────────────┘     │
  │                                                                      │
  │  KEY INSIGHT:                                                        │
  │  • "!" or "!!" at the START = account is LOCKED                     │
  │  • Locking PREPENDS "!" to the hash. Unlocking REMOVES the "!".   │
  │  • The hash itself is preserved — unlock restores the old password.│
  └──────────────────────────────────────────────────────────────────────┘
```

### 19.4 Systematic Login Troubleshooting Flowchart

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │  SCENARIO: "Users cannot log in to the Linux system"                │
  │                                                                      │
  │  STEP 1: Does the user EXIST?                                       │
  │  ┌───────────────────────────────────────────────────────────────┐   │
  │  │ $ id john                                                     │   │
  │  │ id: 'john': no such user    → User doesn't exist! Create it  │   │
  │  │ uid=1001(john)...           → User exists, continue ▼        │   │
  │  └───────────────────────────────────────────────────────────────┘   │
  │                                                                      │
  │  STEP 2: Is the account LOCKED?                                     │
  │  ┌───────────────────────────────────────────────────────────────┐   │
  │  │ $ sudo passwd -S john                                         │   │
  │  │ john L 2026-01-15 ...    ← "L" = LOCKED ❌                  │   │
  │  │ john P 2026-01-15 ...    ← "P" = Password set ✅            │   │
  │  │ john NP 2026-01-15 ...   ← "NP" = No Password ⚠️           │   │
  │  │                                                               │   │
  │  │ Also check shadow directly:                                   │   │
  │  │ $ sudo grep "^john:" /etc/shadow                              │   │
  │  │ john:!$6$xyz...    ← Leading "!" = LOCKED                   │   │
  │  │ john:!!            ← Password never set                      │   │
  │  └───────────────────────────────────────────────────────────────┘   │
  │                                                                      │
  │  STEP 3: Is the PASSWORD correct / set?                             │
  │  ┌───────────────────────────────────────────────────────────────┐   │
  │  │ $ sudo grep "^john:" /etc/shadow | cut -d: -f2               │   │
  │  │ !!          → No password set. Set one:                      │   │
  │  │               $ sudo passwd john                              │   │
  │  │ (empty)     → Empty password! Set one immediately.           │   │
  │  │ $6$salt$... → Password exists. Maybe user forgot it:         │   │
  │  │               $ sudo passwd john   (reset it)                 │   │
  │  └───────────────────────────────────────────────────────────────┘   │
  │                                                                      │
  │  STEP 4: Is the SHELL valid?                                        │
  │  ┌───────────────────────────────────────────────────────────────┐   │
  │  │ $ getent passwd john | cut -d: -f7                            │   │
  │  │ /usr/sbin/nologin   → Login blocked! Change shell:          │   │
  │  │   $ sudo usermod -s /bin/bash john                            │   │
  │  │ /bin/false          → Login blocked! Change shell.           │   │
  │  │ /bin/bash           → Shell is fine ✅                       │   │
  │  │                                                               │   │
  │  │ Also verify the shell is listed in /etc/shells:               │   │
  │  │ $ cat /etc/shells                                             │   │
  │  │ (shell must appear here or PAM may reject it)                │   │
  │  └───────────────────────────────────────────────────────────────┘   │
  │                                                                      │
  │  STEP 5: Is the account EXPIRED?                                    │
  │  ┌───────────────────────────────────────────────────────────────┐   │
  │  │ $ sudo chage -l john                                          │   │
  │  │ Account expires        : Jan 01, 2026   ← EXPIRED! ❌       │   │
  │  │ Password expires        : Feb 01, 2026   ← Maybe expired    │   │
  │  │                                                               │   │
  │  │ Fix account expiry:                                           │   │
  │  │ $ sudo chage -E -1 john        ← Remove expiry (never)      │   │
  │  │ $ sudo chage -E 2027-12-31 john ← Set new expiry date       │   │
  │  │                                                               │   │
  │  │ Fix password expiry:                                          │   │
  │  │ $ sudo chage -M 99999 john      ← Max days (effectively no  │   │
  │  │                                    password expiry)           │   │
  │  └───────────────────────────────────────────────────────────────┘   │
  │                                                                      │
  │  STEP 6: Is /etc/nologin present?                                   │
  │  ┌───────────────────────────────────────────────────────────────┐   │
  │  │ $ ls -la /etc/nologin                                         │   │
  │  │ If this file EXISTS → ALL non-root logins are BLOCKED.       │   │
  │  │ This is used during maintenance. Remove it:                   │   │
  │  │ $ sudo rm /etc/nologin                                        │   │
  │  └───────────────────────────────────────────────────────────────┘   │
  │                                                                      │
  │  STEP 7: Is the HOME DIRECTORY intact?                              │
  │  ┌───────────────────────────────────────────────────────────────┐   │
  │  │ $ ls -la /home/john/                                          │   │
  │  │ No such directory? → Login may fail or dump to /             │   │
  │  │ $ sudo mkdir -p /home/john                                    │   │
  │  │ $ sudo cp -r /etc/skel/. /home/john/                         │   │
  │  │ $ sudo chown -R john:john /home/john                         │   │
  │  │ $ sudo chmod 700 /home/john                                   │   │
  │  └───────────────────────────────────────────────────────────────┘   │
  │                                                                      │
  │  STEP 8: Check PAM and auth logs                                    │
  │  ┌───────────────────────────────────────────────────────────────┐   │
  │  │ $ tail -50 /var/log/auth.log          ← Ubuntu/Debian        │   │
  │  │ $ tail -50 /var/log/secure            ← RHEL/CentOS          │   │
  │  │ $ journalctl -u sshd -n 50           ← SSH specific         │   │
  │  │                                                               │   │
  │  │ Common PAM errors in logs:                                    │   │
  │  │ "pam_unix: authentication failure"    → wrong password       │   │
  │  │ "pam_faillock: locked"               → too many failures     │   │
  │  │ "pam_nologin: preventing login"      → /etc/nologin exists  │   │
  │  │ "pam_securetty: access denied"       → root on wrong tty    │   │
  │  └───────────────────────────────────────────────────────────────┘   │
  └──────────────────────────────────────────────────────────────────────┘
```

### 19.5 Locking and Unlocking Users — Complete Reference

```bash
# ══════════════════════════════════════════════════════════════
# METHOD 1: passwd command (most common)
# ══════════════════════════════════════════════════════════════

# LOCK a user account:
sudo passwd -l john
# passwd: password expiry information changed.
# This PREPENDS "!" to the password hash in /etc/shadow
# john:$6$xyz... → john:!$6$xyz...
# ⚠️ SSH KEY LOGIN STILL WORKS! passwd -l only blocks password auth.

# UNLOCK a user account:
sudo passwd -u john
# passwd: password expiry information changed.
# Removes the leading "!" from the hash
# john:!$6$xyz... → john:$6$xyz...

# Check lock status:
sudo passwd -S john
# john P 2026-01-15 0 99999 7 -1
#      │
#      P = Password set (usable)
#      L = Locked
#      NP = No Password

# ══════════════════════════════════════════════════════════════
# METHOD 2: usermod command
# ══════════════════════════════════════════════════════════════

# LOCK:
sudo usermod -L john         # Same as passwd -l (prepends "!")

# UNLOCK:
sudo usermod -U john         # Same as passwd -u (removes "!")

# ══════════════════════════════════════════════════════════════
# METHOD 3: faillock (for PAM-based lockouts — too many failed attempts)
# ══════════════════════════════════════════════════════════════

# View failed attempts:
sudo faillock --user john
# john:
# When        Type   Source        Valid
# 2026-03-03  RHOST  192.168.1.50  V     ← V = valid (counts toward lockout)
# 2026-03-03  RHOST  192.168.1.50  V
# 2026-03-03  RHOST  192.168.1.50  V

# UNLOCK (reset failed attempts):
sudo faillock --user john --reset

# Old method (RHEL 7 and earlier — deprecated):
# pam_tally2 --user john              ← View
# pam_tally2 --user john --reset      ← Reset

# ══════════════════════════════════════════════════════════════
# RESET PASSWORD (if user forgot it):
# ══════════════════════════════════════════════════════════════
sudo passwd john
# Enter new UNIX password: 
# Retype new UNIX password:
# passwd: password updated successfully

# Force user to change password on next login:
sudo chage -d 0 john
# Next login → "You are required to change your password immediately"
```

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         passwd -l vs usermod -L vs faillock — WHEN TO USE WHAT      │
  │                                                                      │
  │  ┌──────────────┬────────────────────────────────────────────────┐   │
  │  │ Command       │ Use Case                                      │   │
  │  ├──────────────┼────────────────────────────────────────────────┤   │
  │  │ passwd -l    │ Admin intentionally locks a user account.     │   │
  │  │              │ Prepends "!" to shadow hash.                  │   │
  │  │              │ SSH keys STILL work! Only blocks password.    │   │
  │  ├──────────────┼────────────────────────────────────────────────┤   │
  │  │ usermod -L   │ Same effect as passwd -l.                     │   │
  │  │              │ Also use: usermod -L -e 1 john                │   │
  │  │              │ (lock + expire = block ALL access including   │   │
  │  │              │ SSH keys)                                      │   │
  │  ├──────────────┼────────────────────────────────────────────────┤   │
  │  │ faillock     │ System auto-locked due to too many failed     │   │
  │  │ --reset      │ login attempts (PAM pam_faillock module).     │   │
  │  │              │ Use this when user says "I'm locked out       │   │
  │  │              │ after wrong passwords."                        │   │
  │  ├──────────────┼────────────────────────────────────────────────┤   │
  │  │ chage -E 1   │ Expire the account entirely. Blocks all      │   │
  │  │ (date = 1)   │ login methods including SSH keys.             │   │
  │  │              │ Reverse: chage -E -1 john                     │   │
  │  └──────────────┴────────────────────────────────────────────────┘   │
  │                                                                      │
  │  TO FULLY BLOCK A USER (employee termination):                      │
  │  $ sudo passwd -l john              ← Lock password                │
  │  $ sudo usermod -e 1 john           ← Expire account              │
  │  $ sudo usermod -s /usr/sbin/nologin john  ← Block shell          │
  │  $ sudo pkill -u john               ← Kill all their sessions     │
  │  This blocks: password login, SSH keys, su, and cron jobs.         │
  └──────────────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha:** `passwd -l` does NOT block SSH key authentication! If the user has their public key in `~/.ssh/authorized_keys`, they can still log in. To fully block a user, you must ALSO either expire their account (`usermod -e 1`) or set their shell to `/usr/sbin/nologin`. Many admins miss this — it's a common interview trick question.

> **DevOps Pro Tip:** When an employee leaves, the proper offboarding sequence is: 1) Lock the account (`passwd -l`), 2) Expire it (`chage -E 1`), 3) Kill active sessions (`pkill -u john`), 4) Move their home directory to a backup location, 5) Remove from all groups, 6) Move their AD/LDAP account to Disabled OU. Automate this with a script — manual steps get missed.

---

## 20. Physical Disk Mounting — End-to-End Walkthrough

### 20.1 The Complete Flow — From Physical Disk to Mounted Filesystem

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         DISK MOUNTING — THE FULL PICTURE                             │
  │                                                                      │
  │  Physical  ──► Detect  ──► Partition ──► Format ──► Mount ──► fstab │
  │  Disk          Device      (fdisk/     (mkfs)     (mount)   (persist)│
  │  Attached      (/dev/sdb)  gdisk/                                    │
  │                             parted)                                   │
  │                                                                      │
  │  STEP-BY-STEP:                                                       │
  │                                                                      │
  │  ┌─────────┐  Plugged in   ┌──────────────┐  Kernel detects        │
  │  │ Physical│──────────────►│ /dev/sdb      │  (check dmesg)         │
  │  │ Disk    │               │ (raw block    │                         │
  │  │ (SSD/   │               │  device)      │                         │
  │  │  HDD)   │               └──────┬───────┘                         │
  │  └─────────┘                      │                                  │
  │                                   ▼                                  │
  │                    ┌──────────────────────────┐                      │
  │                    │ Partition with fdisk      │                      │
  │                    │ /dev/sdb → /dev/sdb1     │                      │
  │                    │ (create partition table)  │                      │
  │                    └──────────┬───────────────┘                      │
  │                               │                                      │
  │                               ▼                                      │
  │                    ┌──────────────────────────┐                      │
  │                    │ Format with mkfs          │                      │
  │                    │ mkfs.ext4 /dev/sdb1       │                      │
  │                    │ (creates filesystem)       │                      │
  │                    └──────────┬───────────────┘                      │
  │                               │                                      │
  │                               ▼                                      │
  │                    ┌──────────────────────────┐                      │
  │                    │ Mount to directory         │                      │
  │                    │ mount /dev/sdb1 /mnt/data │                      │
  │                    │ (accessible now!)          │                      │
  │                    └──────────┬───────────────┘                      │
  │                               │                                      │
  │                               ▼                                      │
  │                    ┌──────────────────────────┐                      │
  │                    │ Add to /etc/fstab         │                      │
  │                    │ (survives reboot)          │                      │
  │                    └──────────────────────────┘                      │
  └──────────────────────────────────────────────────────────────────────┘
```

### 20.2 Step-by-Step Commands

```bash
# ══════════════════════════════════════════════════════════════
# STEP 1: Detect the new disk
# ══════════════════════════════════════════════════════════════
# After physically attaching the disk:

sudo dmesg | tail -20          # Kernel messages about new device
# [  120.456] sd 2:0:0:0: [sdb] 209715200 512-byte sectors (107 GB)
# [  120.457] sd 2:0:0:0: [sdb] Attached SCSI disk

lsblk                           # List all block devices
# NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
# sda      8:0    0    50G  0 disk
# ├─sda1   8:1    0     1G  0 part /boot
# └─sda2   8:2    0    49G  0 part /
# sdb      8:16   0   100G  0 disk              ← NEW DISK (no partitions yet)

sudo fdisk -l /dev/sdb          # Disk details

# ══════════════════════════════════════════════════════════════
# STEP 2: Partition the disk
# ══════════════════════════════════════════════════════════════
sudo fdisk /dev/sdb
# Interactive commands:
#   n    → New partition
#   p    → Primary partition
#   1    → Partition number 1
#   (Enter) → Default first sector
#   (Enter) → Default last sector (use whole disk)
#   w    → Write changes and exit

# VERIFY:
lsblk
# sdb      8:16   0   100G  0 disk
# └─sdb1   8:17   0   100G  0 part              ← Partition created!

# FOR GPT DISKS (>2TB or UEFI):
# Use gdisk or parted instead of fdisk:
# sudo parted /dev/sdb mklabel gpt
# sudo parted /dev/sdb mkpart primary ext4 0% 100%

# ══════════════════════════════════════════════════════════════
# STEP 3: Create filesystem (format)
# ══════════════════════════════════════════════════════════════
sudo mkfs.ext4 /dev/sdb1       # ext4 (most common for Linux)
# or:
# sudo mkfs.xfs /dev/sdb1      # XFS (default on RHEL/CentOS 7+)
# sudo mkfs.ntfs /dev/sdb1     # NTFS (for Windows compatibility)

# VERIFY:
sudo blkid /dev/sdb1
# /dev/sdb1: UUID="a1b2c3d4-..." TYPE="ext4" PARTUUID="..."
#                   ↑
#                   SAVE THIS UUID! You'll need it for fstab.

# ══════════════════════════════════════════════════════════════
# STEP 4: Create mount point and mount
# ══════════════════════════════════════════════════════════════
sudo mkdir -p /mnt/data         # Create the mount point directory
sudo mount /dev/sdb1 /mnt/data  # Mount it!

# VERIFY:
df -h /mnt/data
# Filesystem    Size  Used  Avail Use% Mounted on
# /dev/sdb1      98G   60M    93G   1% /mnt/data

mount | grep sdb1
# /dev/sdb1 on /mnt/data type ext4 (rw,relatime)

# Test it:
echo "Hello from new disk" | sudo tee /mnt/data/test.txt
cat /mnt/data/test.txt          # Hello from new disk ✅

# ══════════════════════════════════════════════════════════════
# STEP 5: Make PERMANENT (survive reboot) → /etc/fstab
# ══════════════════════════════════════════════════════════════

# Get the UUID (ALWAYS use UUID, not /dev/sdb1 — device names can change!):
sudo blkid /dev/sdb1
# /dev/sdb1: UUID="a1b2c3d4-e5f6-7890-abcd-ef1234567890" TYPE="ext4"

# Add to /etc/fstab:
echo "UUID=a1b2c3d4-e5f6-7890-abcd-ef1234567890  /mnt/data  ext4  defaults  0  2" \
    | sudo tee -a /etc/fstab

# ⚠️ CRITICAL — TEST BEFORE REBOOT:
sudo mount -a                    # Tries to mount everything in fstab
echo $?                          # Must be 0 (success)
# If mount -a fails → YOU HAVE A TYPO → fix it BEFORE rebooting!
# A bad fstab entry can PREVENT YOUR SYSTEM FROM BOOTING.

# VERIFY:
df -h /mnt/data                  # Should show the disk mounted
```

### 20.3 /etc/fstab — The Definitive Reference

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         /etc/fstab — FIELD-BY-FIELD BREAKDOWN                        │
  │                                                                      │
  │  # <filesystem>       <mountpoint> <type> <options>      <dump><pass>│
  │  UUID=a1b2c3d4-...    /mnt/data    ext4   defaults       0     2    │
  │    │                    │           │       │             │     │    │
  │    │                    │           │       │             │     │    │
  │    ▼                    ▼           ▼       ▼             ▼     ▼    │
  │  FIELD 1: Device       FIELD 2:   FIELD 3: FIELD 4:     F5    F6   │
  │  UUID=... (preferred)  Where to   Type of  Mount        Dump  fsck │
  │  /dev/sdb1 (fragile)   mount it   FS       options      flag  order│
  │  LABEL=data (by label)                                               │
  │                                                                      │
  │  MOUNT OPTIONS (Field 4):                                            │
  │  ┌──────────────┬────────────────────────────────────────────┐       │
  │  │ Option        │ Meaning                                    │       │
  │  ├──────────────┼────────────────────────────────────────────┤       │
  │  │ defaults      │ rw, suid, dev, exec, auto, nouser, async │       │
  │  │ ro            │ Read-only                                  │       │
  │  │ rw            │ Read-write                                 │       │
  │  │ noexec        │ Can't execute binaries (security for /tmp)│       │
  │  │ nosuid        │ Ignore SUID/SGID bits (security)          │       │
  │  │ nodev         │ Ignore device files (security)            │       │
  │  │ noatime       │ Don't update access times (performance)   │       │
  │  │ nofail        │ Don't fail boot if device missing ⭐      │       │
  │  │ auto / noauto │ Mount automatically at boot / don't       │       │
  │  │ user / nouser │ Allow regular users to mount / don't      │       │
  │  └──────────────┴────────────────────────────────────────────┘       │
  │                                                                      │
  │  FIELD 5 (dump): 0 = don't backup, 1 = backup (rarely used now)   │
  │  FIELD 6 (pass): 0 = skip fsck, 1 = root (check first),           │
  │                   2 = other partitions (check after root)            │
  │                                                                      │
  │  WHY UUID INSTEAD OF /dev/sdb1?                                      │
  │  • Device names (/dev/sdb) can CHANGE between reboots              │
  │    (new disk added → old sdb becomes sdc → WRONG MOUNT!)          │
  │  • UUID is unique and PERMANENT for that filesystem                 │
  │  • LABEL is also stable but must be manually set                    │
  │  • Find UUID: sudo blkid  or  lsblk -f                             │
  └──────────────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha:** The single most dangerous mistake in Linux admin: a typo in `/etc/fstab`. If a required mount fails at boot and `nofail` is NOT set, the system drops to an **emergency shell** (or hangs entirely). Always: 1) Run `sudo mount -a` to test, 2) Use the `nofail` option for non-critical mounts (like data disks), 3) Keep a rescue USB ready. For cloud VMs, a bad fstab means rebuilding the instance from a snapshot.

> **DevOps Pro Tip:** In cloud environments (AWS, Azure, GCP), **never** use `/dev/xvdf` or `/dev/sdb` in fstab — device names change on stop/start. Always use `UUID` or cloud-specific labels. AWS EBS volumes: use `UUID`. Azure: use `UUID`. Google Cloud: use `UUID`. Also add `nofail` so your instance boots even if the extra volume isn't attached.

---

## 21. systemctl stop vs disable — The Deep Comparison

### 21.1 Mental Model — Two Independent Dimensions

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         systemctl stop vs disable — TWO SEPARATE THINGS              │
  │                                                                      │
  │  Think of it as TWO SWITCHES for every service:                     │
  │                                                                      │
  │  ┌─────────────────────────────────────────────────────────────┐     │
  │  │                                                             │     │
  │  │  SWITCH 1: Running NOW?        SWITCH 2: Start at BOOT?    │     │
  │  │  ┌────────────┐                ┌────────────┐               │     │
  │  │  │ start      │ ON ────────    │ enable     │ ON ────────   │     │
  │  │  │            │           │    │            │           │   │     │
  │  │  │ stop       │ OFF ──────┘    │ disable    │ OFF ──────┘   │     │
  │  │  └────────────┘                └────────────┘               │     │
  │  │                                                             │     │
  │  │  They are COMPLETELY INDEPENDENT:                           │     │
  │  │                                                             │     │
  │  │  ┌──────────────────┬────────────┬──────────────────────┐  │     │
  │  │  │ Combination       │ Running?   │ Starts at boot?      │  │     │
  │  │  ├──────────────────┼────────────┼──────────────────────┤  │     │
  │  │  │ start + enable   │ YES ✅     │ YES ✅               │  │     │
  │  │  │ start + disable  │ YES ✅     │ NO ❌                │  │     │
  │  │  │ stop + enable    │ NO ❌      │ YES ✅ (starts later)│  │     │
  │  │  │ stop + disable   │ NO ❌      │ NO ❌                │  │     │
  │  │  └──────────────────┴────────────┴──────────────────────┘  │     │
  │  │                                                             │     │
  │  └─────────────────────────────────────────────────────────────┘     │
  │                                                                      │
  │  COMMON MISTAKE: "I disabled nginx, why is it still running?"       │
  │  Because disable only affects BOOT behavior. The service is still   │
  │  running right now. You need BOTH:                                   │
  │  $ sudo systemctl stop nginx      ← stop NOW                       │
  │  $ sudo systemctl disable nginx   ← don't start at boot            │
  │                                                                      │
  │  SHORTCUT:                                                           │
  │  $ sudo systemctl disable --now nginx  ← disable + stop in one     │
  │  $ sudo systemctl enable --now nginx   ← enable + start in one     │
  └──────────────────────────────────────────────────────────────────────┘
```

### 21.2 What Happens Under the Hood

```bash
# ══════════════════════════════════════════════════════════════
# systemctl STOP — what it actually does:
# ══════════════════════════════════════════════════════════════
sudo systemctl stop nginx

# 1. systemd reads the unit file's ExecStop= directive
# 2. If ExecStop is defined → runs that command
# 3. If not → sends SIGTERM to the main process
# 4. Waits TimeoutStopSec (default: 90s)
# 5. If still running → sends SIGKILL
# 6. Service state: active → inactive (dead)

# The service can be started again manually or by dependencies.
# If the service is "enabled", it WILL start again at next boot.

# ══════════════════════════════════════════════════════════════
# systemctl DISABLE — what it actually does:
# ══════════════════════════════════════════════════════════════
sudo systemctl disable nginx

# 1. REMOVES the symlink from /etc/systemd/system/multi-user.target.wants/
#    that points to the unit file.
#
# Before disable:
# /etc/systemd/system/multi-user.target.wants/nginx.service → 
#     /lib/systemd/system/nginx.service
#
# After disable:
# Symlink REMOVED. systemd won't pull nginx into the boot target.
#
# 2. Does NOT stop the service! It just removes the boot-time link.
# 3. The service can still be started manually: systemctl start nginx

# SEE THE SYMLINKS:
ls -la /etc/systemd/system/multi-user.target.wants/
# lrwxrwxrwx 1 root root ... nginx.service → /lib/systemd/system/nginx.service
# (After disable, this symlink is gone)

# ══════════════════════════════════════════════════════════════
# systemctl MASK — the NUCLEAR option (beyond disable)
# ══════════════════════════════════════════════════════════════
sudo systemctl mask nginx

# Creates a symlink to /dev/null:
# /etc/systemd/system/nginx.service → /dev/null
#
# This PREVENTS the service from being started AT ALL — even manually!
# systemctl start nginx → "Unit nginx.service is masked."
#
# Use mask when you want to ABSOLUTELY PREVENT a service from running
# (e.g., you replaced Apache with Nginx and never want Apache to start)

sudo systemctl unmask nginx     # Undo the mask
```

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │         stop vs disable vs mask — COMPARISON TABLE                   │
  │                                                                      │
  │  ┌──────────────┬───────────────┬────────────┬──────────────────┐    │
  │  │ Command       │ Stops now?    │ Blocks     │ Can be started   │    │
  │  │               │               │ boot start?│ manually?        │    │
  │  ├──────────────┼───────────────┼────────────┼──────────────────┤    │
  │  │ stop          │ YES ✅        │ NO         │ YES              │    │
  │  │ disable       │ NO            │ YES ✅     │ YES              │    │
  │  │ stop+disable  │ YES ✅        │ YES ✅     │ YES              │    │
  │  │ mask          │ NO            │ YES ✅     │ NO ❌ (blocked!) │    │
  │  │ stop+mask     │ YES ✅        │ YES ✅     │ NO ❌ (blocked!) │    │
  │  └──────────────┴───────────────┴────────────┴──────────────────┘    │
  │                                                                      │
  │  PRACTICAL SCENARIOS:                                                │
  │  ┌─────────────────────────────────────────────────────────────┐     │
  │  │ "Stop nginx for maintenance, start later today"             │     │
  │  │ → systemctl stop nginx (don't disable — it should boot)    │     │
  │  │                                                             │     │
  │  │ "We replaced Apache with Nginx permanently"                 │     │
  │  │ → systemctl disable --now apache2  (stop + no boot)        │     │
  │  │ → OR: systemctl mask apache2  (prevent ANY start)           │     │
  │  │                                                             │     │
  │  │ "Temporarily disable a service for testing"                 │     │
  │  │ → systemctl stop myservice (leave enabled for boot)         │     │
  │  │                                                             │     │
  │  │ "Server is being decommissioned next week"                  │     │
  │  │ → systemctl disable --now all-services (stop + no reboot)  │     │
  │  └─────────────────────────────────────────────────────────────┘     │
  └──────────────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha:** `systemctl disable` does NOT stop a running service. This confuses many people — they disable a service thinking it will stop, then wonder why it's still running. Similarly, `systemctl enable` does NOT start a service. Always pair stop/start with disable/enable, or use the `--now` flag: `systemctl enable --now nginx` (enable + start) or `systemctl disable --now nginx` (disable + stop).

> **DevOps Pro Tip:** Use `systemctl mask` for services you absolutely never want running — like preventing `apache2` from starting after you've switched to nginx, or preventing `firewalld` on a system managed by iptables directly. Without mask, another package installation or a colleague running `systemctl start apache2` could accidentally bring it back. Mask makes it impossible.

---

### Production FAQ — New Scenarios

### Scenario 11: User Reports "Permission Denied" When Logging In via SSH

```bash
# DIAGNOSIS STEPS:

# 1. Check auth logs for the exact error:
sudo tail -30 /var/log/auth.log | grep john
# Mar 3 10:30:00 server sshd[4523]: pam_unix(sshd:auth): authentication
#   failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=10.0.0.50 user=john

# 2. Is the account locked?
sudo passwd -S john
# john L 2026-02-15 0 99999 7 -1    ← "L" = LOCKED!

# 3. Was it auto-locked by faillock (too many failed attempts)?
sudo faillock --user john
# 5 failures from 10.0.0.50         ← Locked due to brute force!

# 4. FIX:
sudo faillock --user john --reset   # Clear the lockout counter
# If manually locked:
sudo passwd -u john                  # Unlock the account

# 5. VERIFY:
sudo passwd -S john
# john P 2026-02-15 0 99999 7 -1    ← "P" = Password set, unlocked ✅

# 6. PREVENT FUTURE LOCKOUTS:
# Review /etc/security/faillock.conf:
# deny = 5              ← lock after 5 failures
# unlock_time = 600     ← auto-unlock after 600 seconds (10 min)
# even_deny_root = no   ← don't lock root (dangerous if yes!)
```

### Scenario 12: Password Is Missing in /etc/shadow

```bash
# SYMPTOM: User can't log in, and /etc/shadow shows no hash:
sudo grep "^john:" /etc/shadow
# john::19784:0:99999:7:::
#      ^^
#      EMPTY! No password hash at all.

# This means ANYONE can log in as john with NO password
# (if PAM allows it — most configs reject empty passwords).

# WHY THIS HAPPENS:
# - Account created with useradd (without -p or passwd)
# - Someone ran: passwd -d john  (deletes the password!)
# - Misconfigured automation/provisioning script

# FIX — Set a password:
sudo passwd john
# Enter new password:
# Retype new password:
# passwd: password updated successfully

# VERIFY:
sudo grep "^john:" /etc/shadow
# john:$y$j9T$...:19784:0:99999:7:::
#      ^^^^^^^^
#      Now has a proper hash ✅

# FORCE PASSWORD CHANGE on next login:
sudo chage -d 0 john
# User must set their OWN password on next login.

# AUDIT — Find ALL accounts with empty passwords:
sudo awk -F: '($2 == "") {print $1}' /etc/shadow
# Lists every user with no password — FIX THEM ALL!
```

---

*Batch 8 complete — User Login Troubleshooting + Physical Disk Mounting + systemctl stop vs disable*

*📘 1st Review Handbook — Gap content added (8 batches total)*
