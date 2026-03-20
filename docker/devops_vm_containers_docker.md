# DevOps Notes: VM vs Containers & Docker
### From Beginner to Advanced — A Complete Field Guide

> **How to use these notes:**
> Read linearly as a beginner. Return as a reference as you grow.
> Each section has: concepts → ASCII diagrams → gotchas → production FAQs.

---

## Table of Contents

- [Batch 1 — Foundations & VM vs Containers](#batch-1)
  - [1.1 Why This Topic Matters in DevOps](#11-why-this-topic-matters)
  - [1.2 The Problem Before Containers](#12-the-problem-before-containers)
  - [1.3 Virtual Machines — Mental Model](#13-virtual-machines)
  - [1.4 Containers — Mental Model](#14-containers)
  - [1.5 Side-by-Side Comparison](#15-side-by-side-comparison)
  - [1.6 When to Use What](#16-when-to-use-what)
  - [1.7 Gotchas & DevOps Context](#17-gotchas)
  - [1.8 Production FAQ — Batch 1](#18-production-faq)
- [Batch 2 — Container Internals & Docker Architecture](#batch-2) *(coming soon)*

---

# BATCH 1
## Foundations & VM vs Containers

---

## 1.1 Why This Topic Matters

In DevOps, your job isn't just to write code — it's to **ship it reliably, repeatedly, and fast**.
The core challenge has always been:

> *"It works on my machine"* — every developer, ever.

Understanding VMs and containers gives you the power to:
- Package applications so they run **identically** in dev, staging, and production
- Use infrastructure resources **efficiently**
- Build **scalable, portable** systems
- Reason about **isolation, security, and failure domains**

This is table-stakes knowledge for any modern DevOps/Platform/SRE role.

---

## 1.2 The Problem Before Containers

Imagine a server running 3 apps:

```
+-----------------------------------------+
|              Physical Server            |
|                                         |
|  [App A]   [App B]   [App C]            |
|  Python 2  Python 3  Python 3           |
|  Django    Flask     FastAPI            |
|  libssl v1 libssl v2 libssl v2          |
|                                         |
|  ALL sharing the same OS, same libs     |
+-----------------------------------------+
```

**Problems:**
- App A needs Python 2, App C needs Python 3 → **conflict**
- Upgrading libssl for App B might **break** App A
- One app crashes → it can consume all RAM and **starve** others
- Deploying App B means SSHing into the server and running scripts manually
- Dev laptop has macOS. Server runs CentOS 7. Things break silently.

This is called **dependency hell** and **environment drift**.

---

## 1.3 Virtual Machines — Mental Model

### What is a VM?

A **Virtual Machine** is a software emulation of an entire physical computer.
It uses a piece of software called a **Hypervisor** to fake hardware for each guest OS.

```
+-------------------------------------------------------+
|                   Physical Server                     |
|   (Real CPU, Real RAM, Real Disk, Real NIC)           |
+-------------------------------------------------------+
|              HYPERVISOR (Type 1 or Type 2)            |
|   e.g. VMware ESXi, KVM, Hyper-V, VirtualBox         |
+-------------------------------------------------------+
|         |                |                |           |
|  +------+------+  +------+------+  +------+------+   |
|  |  Guest OS 1 |  |  Guest OS 2 |  |  Guest OS 3 |   |
|  | (Ubuntu 20) |  | (CentOS 7)  |  | (Windows)   |   |
|  +-------------+  +-------------+  +-------------+   |
|  | App A       |  | App B       |  | App C       |   |
|  | Libs/Deps   |  | Libs/Deps   |  | Libs/Deps   |   |
|  +-------------+  +-------------+  +-------------+   |
+-------------------------------------------------------+
```

### Two Types of Hypervisors

```
TYPE 1 — "Bare Metal" Hypervisor          TYPE 2 — "Hosted" Hypervisor
(runs directly on hardware)               (runs inside a host OS)

+---------------------+                   +---------------------+
| VM1 | VM2 | VM3     |                   | VM1 | VM2           |
+---------------------+                   +---------------------+
| Hypervisor (ESXi)   |                   | Hypervisor          |
+---------------------+                   | (VirtualBox/VMware) |
| Physical Hardware   |                   +---------------------+
+---------------------+                   | Host OS (macOS/Win) |
                                          +---------------------+
                                          | Physical Hardware   |
                                          +---------------------+

  Used in: data centers, cloud           Used in: dev laptops, testing
  Examples: VMware ESXi, KVM,            Examples: VirtualBox, VMware
  Microsoft Hyper-V, Xen                 Workstation, Parallels
```

### VM Boot Flow

```
  Power On
      |
      v
  BIOS/UEFI (emulated)
      |
      v
  Bootloader (GRUB)
      |
      v
  Kernel loads (full OS kernel)
      |
      v
  Init system (systemd)
      |
      v
  Services start (sshd, networking, etc.)
      |
      v
  Your App starts           ← This can take 30s to 2+ minutes
```

### What a VM gives you:

| Property | Detail |
|----------|--------|
| **Isolation** | Complete — separate kernel, separate memory space |
| **OS flexibility** | Run Windows on Linux host, or vice versa |
| **Boot time** | Slow — 30 seconds to 5 minutes |
| **Size** | Heavy — typically 1–40 GB per VM image |
| **Resource overhead** | High — each VM runs a full OS consuming RAM/CPU |
| **Security boundary** | Strong — kernel-level separation |

---

## 1.4 Containers — Mental Model

### What is a Container?

A container is an **isolated process** (or group of processes) running on the **host OS kernel**, with its own:
- Filesystem view (isolated via **namespace**)
- Process tree (it can't see the host's other processes)
- Network stack (its own IP, ports)
- Resource limits (CPU/RAM capped via **cgroups**)

> **Key insight:** A container is NOT a mini-VM. It's a process with walls around it.

```
+-------------------------------------------------------+
|                   Physical Server                     |
+-------------------------------------------------------+
|                    Host OS (Linux)                    |
|              ONE shared Linux kernel                  |
+-------------------------------------------------------+
|   Container Runtime (Docker, containerd, podman)      |
+-------------------------------------------------------+
|         |                |                |           |
|  +------+------+  +------+------+  +------+------+   |
|  | Container A |  | Container B |  | Container C |   |
|  | [App A    ] |  | [App B    ] |  | [App C    ] |   |
|  | [Libs/Deps] |  | [Libs/Deps] |  | [Libs/Deps] |   |
|  | /usr /bin   |  | /usr /bin   |  | /usr /bin   |   |
|  +-------------+  +-------------+  +-------------+   |
|       |                  |                |           |
|       +------------------+----------------+           |
|              All sharing host kernel                  |
+-------------------------------------------------------+
```

### Container Startup Flow

```
  docker run myapp
      |
      v
  Container runtime pulls image (if not cached)
      |
      v
  Creates namespaces (pid, net, mnt, uts, ipc, user)
      |
      v
  Sets up cgroup limits (CPU, memory)
      |
      v
  Mounts layered filesystem (UnionFS)
      |
      v
  Starts your process (PID 1 inside container)
      |
      v
  App is running                ← Takes milliseconds to ~2 seconds
```

### The Shipping Container Analogy

```
  Before containers (shipping):         After containers (shipping):

  +-------+  +-------+  +-------+      +=========+  +=========+
  | PIANO |  | VASE  |  | BOOKS |      || PIANO ||  || VASE  ||
  | (wrap |  | (wrap |  | (wrap |      || VASE  ||  || BOOKS ||
  |  each)|  |  each)|  |  each)|      |+-------+|  |+-------+|
  +-------+  +-------+  +-------+      +=========+  +=========+
  Different handling                   Standard container
  for each item                        Same ship, same crane
                                       Same truck — works everywhere

  Same in software:
  "Works on my machine"    →    Container = app + deps + config
  Different env = different      Same image = same behavior
  behavior                       everywhere it runs
```

---

## 1.5 Side-by-Side Comparison

```
FEATURE COMPARISON: VMs vs Containers
======================================

                    VIRTUAL MACHINE          CONTAINER
                    ---------------          ---------
  Isolation         Full OS isolation        Process-level isolation
  
  Kernel            Each VM has own kernel   Shares host kernel
  
  Boot Time         30 sec — 5 min           Milliseconds — 2 sec
  
  Image Size        1 GB — 40+ GB            5 MB — 500 MB (typical)
  
  Overhead          High (full OS per VM)    Low (just app + libs)
  
  Density           ~10s of VMs per server   ~100s of containers
  
  Security          Stronger boundary        Weaker (shared kernel)
  
  OS flexibility    Any OS on any host       Must match host kernel*
  
  Portability       Good (OVA/VMDK/QCOW2)   Excellent (OCI standard)
  
  Startup speed     Slow                     Very fast
  
  Use case          Legacy apps, full OS     Microservices, APIs,
                    control, Windows apps,   CI/CD, stateless apps
                    strong security needs    cloud-native workloads

  * Linux containers need a Linux kernel. On Mac/Windows, Docker
    runs a lightweight Linux VM under the hood automatically.
```

### Resource Usage Visualization

```
  4 VMs on a server:                  20 Containers on same server:

  +--------+--------+                 +-+--+-+--+-+-+--+-+--+
  |  VM1   |  VM2   |                 |C1|C2|C3|C4|C5|C6|C7|
  | 2GB OS | 2GB OS |                 +-+--+-+--+-+-+--+-+--+
  | 100MB  | 200MB  |                 |C8|...                |
  |  App   |  App   |                 +--+----+----+----+----+
  +--------+--------+                 Host OS (1 kernel)
  |  VM3   |  VM4   |                 Small shared overhead
  | 2GB OS | 2GB OS |
  | 50MB   | 300MB  |
  |  App   |  App   |
  +--------+--------+

  ~8GB+ just for OS overhead          Minimal kernel overhead
  Only 4 apps running                 20 apps using same RAM pool
```

---

## 1.6 When to Use What

### Use a VM when:

```
  ✅ You need to run Windows workloads on Linux hosts
  ✅ You need kernel-level isolation (compliance: PCI-DSS, HIPAA)
  ✅ Running legacy apps that need specific OS versions
  ✅ You need full OS control (custom kernel modules, drivers)
  ✅ Strong multi-tenant security requirements (hosting providers)
  ✅ Running untrusted/unknown workloads (security sandboxing)
```

### Use containers when:

```
  ✅ Microservices architecture
  ✅ CI/CD pipelines (fast build/test/deploy cycles)
  ✅ Stateless web apps and APIs
  ✅ Consistent dev → staging → prod environments
  ✅ High-density workloads (many small services)
  ✅ Cloud-native apps (K8s, ECS, Cloud Run)
  ✅ Reproducible builds and environments
```

### The hybrid reality in production:

```
  Most production clouds use BOTH:

  +--------------------------------------------------+
  |           Cloud Provider (AWS/GCP/Azure)          |
  +--------------------------------------------------+
  |        Physical Servers in Data Centers          |
  +--------------------------------------------------+
  |               Hypervisor (KVM, Xen)              |
  +--------------------------------------------------+
  |   EC2 Instance   |   EC2 Instance   |   ...      |
  |   (VM)           |   (VM)           |             |
  |   +---+---+---+  |   +---+---+---+  |             |
  |   |C1 |C2 |C3 |  |   |C4 |C5 |C6 |  |             |
  |   +---+---+---+  |   +---+---+---+  |             |
  +--------------------------------------------------+

  You rent VMs (EC2, GCE, Azure VMs), then run
  containers INSIDE those VMs. Best of both worlds.
```

---

## 1.7 Gotchas & DevOps Context

### ⚠️ Gotcha #1: Containers are not inherently secure

Many teams assume containers give strong isolation. They don't — not by default.

```
  MYTH:  Container A cannot affect Container B
  
  REALITY: If Container A breaks out of its namespace
           (container escape vulnerability), it has
           access to the HOST and ALL other containers.
  
  Example CVEs: CVE-2019-5736 (runc), CVE-2022-0811 (CRI-O)
  
  FIX:
  - Never run containers as root (use USER in Dockerfile)
  - Use read-only filesystems where possible
  - Use seccomp/AppArmor profiles
  - Scan images for CVEs (Trivy, Snyk)
  - Use gVisor or Kata Containers for stronger isolation
```

### ⚠️ Gotcha #2: Containers are ephemeral — data WILL be lost

```
  Container lifecycle:

  docker run myapp   →  [container running]  →  docker stop
       |                                              |
       |                                              v
       |                                    Container is DELETED
       |                                    All data inside = GONE
       |
       v
  If you wrote to /data inside the container,
  that data is gone when the container stops.
  
  FIX: Use Docker Volumes or bind mounts for persistence.
  Never store state inside a container's writable layer.
```

### ⚠️ Gotcha #3: "It works in Docker" ≠ "It works in production"

```
  Dev setup:                   Production (K8s):
  
  docker run -p 80:80 app      Pod with resource limits
  --no network policies        Network policies enforced
  --no resource limits         No privileged mode
  --full disk access           Read-only root filesystem
  --root user                  Non-root user enforced
  
  Result: App works locally, fails or behaves differently
          in Kubernetes because constraints differ.
  
  FIX: Mirror prod constraints in your local docker-compose.yml
       Set resource limits, non-root users from day 1.
```

### ⚠️ Gotcha #4: VMs and containers solve different problems

```
  A common mistake for beginners:
  "Should I use Docker or a VM for my app?"
  
  Wrong framing. In real DevOps:
  
  VM  = your deployment target (EC2, Droplet, VM on prem)
  Container = how you package and run your app ON that VM
  
  They're complementary, not competing.
```

### ⚠️ Gotcha #5: Container clocks and Linux kernel dependency

```
  Containers share the host kernel. This means:
  
  - Container time = Host time (can't set timezone of kernel)
  - A kernel vulnerability on host affects ALL containers
  - Some syscalls may be unavailable (blocked by seccomp)
  - Kernel version matters: newer container features need
    newer kernels (e.g., cgroup v2 needs kernel 4.15+)
  
  FIX: Keep host OS kernels updated. Know your kernel version.
  Run: uname -r on the host.
```

---

## 1.8 Production FAQ — Batch 1

---

**Q: Our monolith runs on a VM. Should we containerize it immediately?**

```
A: Almost certainly NO. Containerizing a monolith gives you:
   - More complexity (Dockerfile, networking, volumes)
   - No scalability benefit (still one big process)
   - Same deployment pain, just in a different format

   Containerize when you're also breaking into services,
   or when you need environment consistency across a team.
   The right order: stabilize → containerize → decompose.
```

---

**Q: Why does my container work locally but fail on the CI server?**

```
A: Classic env drift. Most common causes:

   1. Image tag mismatch
      Local: node:18 (pulled weeks ago, is 18.12)
      CI:    node:18 (pulled today, is 18.19)
      FIX: Pin exact versions: node:18.12.0-alpine3.17

   2. Platform mismatch (Apple Silicon!)
      Dev builds: linux/arm64 (M1/M2 Mac)
      CI runs:    linux/amd64 (x86 server)
      FIX: docker buildx build --platform linux/amd64

   3. Missing env vars in CI
      .env file exists locally, not in CI environment
      FIX: Check CI secrets / environment variable config

   4. Bind mount path differences
      Works locally: -v ./data:/app/data
      Fails in CI because ./data doesn't exist there
```

---

**Q: How many containers can I run on one VM?**

```
A: Depends on workload, but rough rules of thumb:

   Small containers (50-200MB RAM each):
   → A 4GB VM can run ~15-20 comfortably

   Medium containers (500MB-1GB RAM each):
   → A 16GB VM can run ~10-14

   Always leave 20-30% RAM headroom for the OS and kernel.

   Real limiter is often CPU or network, not RAM.
   Use: docker stats to monitor live resource usage.
   Use: docker run --memory=512m --cpus=0.5 to set limits.
   
   Without limits: one runaway container CAN eat all RAM
   and OOM-kill your other containers. Always set limits in prod.
```

---

**Q: Our security team says containers aren't as secure as VMs. Are they right?**

```
A: They're not wrong — but context matters.

   VMs have a stronger isolation boundary (hardware virtualization).
   Containers share the kernel — a kernel exploit = full host access.

   HOWEVER, containers with proper hardening can be very secure:
   - Non-root USER in Dockerfile
   - Read-only root filesystem (--read-only flag)
   - Dropped capabilities (--cap-drop ALL)
   - Seccomp profiles (restrict available syscalls)
   - No privileged mode (never --privileged in prod)
   - Regular CVE scanning (Trivy, Grype, Snyk)
   - Use distroless or minimal base images

   For extremely sensitive workloads: use Kata Containers
   or gVisor — they give container UX with VM isolation.
```

---

**Q: What's the difference between a container image and a container?**

```
A: Same relationship as Class vs Instance in OOP.

   Image = the blueprint (read-only, layered filesystem)
           Built from a Dockerfile
           Stored in a registry (Docker Hub, ECR, GCR)
           e.g., nginx:1.25-alpine

   Container = a running instance of an image
               Has a writable layer on top
               Has its own process, network, filesystem view
               Is ephemeral — dies when stopped (by default)

   You can run 50 containers from 1 image simultaneously.
   Each gets its own writable layer but shares the image layers.

   Image:      [Base OS layer][Nginx layer][Config layer]  ← read-only
   Container:  [writable layer on top]                     ← ephemeral
```

---

**Q: Docker Desktop on Mac — am I running Linux containers natively?**

```
A: No. macOS does not have the Linux kernel.

   Docker Desktop on Mac runs a lightweight Linux VM
   (using Apple Hypervisor.framework or HyperKit) and
   runs your containers INSIDE that VM.

   Architecture:
   macOS → Hypervisor → Linux VM (LinuxKit) → Docker daemon → Containers

   This is why:
   - File sharing between Mac and containers is slow (bind mounts)
   - Some Linux-specific features behave differently
   - docker run --network host doesn't work as expected on Mac
   
   For production Linux behaviour: always test on a Linux host.
```

---

*End of Batch 1.*

---

# BATCH 2
## Container Internals & Docker Architecture

> **Goal:** Understand what's *really* happening under the hood when a container starts.
> This is what separates DevOps engineers who use Docker from those who truly understand it.
> Knowing internals helps you debug, harden, and architect better systems.

---

## 2.1 Linux Primitives That Power Containers

Containers are **not a kernel feature** — they are a combination of three existing Linux kernel features:

```
  +--------------------------------------------------+
  |         What Makes a Container Work              |
  |                                                  |
  |   NAMESPACES         CGROUPS        UnionFS       |
  |   (isolation)        (limits)       (filesystem)  |
  |                                                  |
  |   "What can it see?" "What can it use?" "What    |
  |                                       does it    |
  |                                       see on     |
  |                                       disk?"     |
  +--------------------------------------------------+

  Docker (and other runtimes) are just user-friendly
  wrappers around these three kernel features.
```

Think of it this way:

```
  docker run nginx
       |
       v
  Container Runtime (containerd/runc)
       |
       +---> Creates Namespaces  (isolation walls)
       |
       +---> Applies cgroups     (resource limits)
       |
       +---> Mounts OverlayFS    (layered filesystem)
       |
       v
  A regular Linux process — just with walls and limits
```

---

## 2.2 Namespaces — Isolation Walls

A **namespace** wraps a global system resource in an abstraction so that
processes inside the namespace see their own isolated copy of that resource.

Linux has **7 namespaces** used by containers:

```
  NAMESPACE     WHAT IT ISOLATES              EXAMPLE
  ──────────────────────────────────────────────────────────────
  PID           Process IDs                   Container sees PID 1
                                              Host has PID 8472

  NET           Network interfaces,           Container gets its
                IP addresses, routes,         own eth0, IP, routing
                firewall rules                table, ports

  MNT           Filesystem mount points       Container has its own
                                              / (root filesystem)

  UTS           Hostname and domain name      Container hostname:
                                              "webapp-prod-1"
                                              Host: "ip-10-0-1-5"

  IPC           Inter-process communication   Shared memory, semaphores
                (shared memory, msg queues)   isolated between containers

  USER          User and group IDs            UID 0 (root) inside
                                              container maps to UID
                                              1000 on host (rootless)

  CGROUP        cgroup root directory         Container can't see host
                                              cgroup hierarchy
```

### PID Namespace — The "I Am PID 1" Effect

```
  HOST process tree:              INSIDE the container:

  PID 1: systemd                  PID 1: nginx          ← thinks it's init!
  PID 2: kthreadd                 PID 2: nginx worker
  ...                             PID 3: nginx worker
  PID 8472: nginx  ←──────────────── same process
  PID 8473: nginx worker
  PID 8474: nginx worker

  The container process doesn't know about the host's PID tree.
  But on the host, you can see ALL processes including container ones.
  Run: ps aux | grep nginx  on the host to see them.
```

### ⚠️ PID 1 Gotcha — Zombie Processes

```
  In Linux, PID 1 is responsible for reaping zombie processes.
  systemd does this on the host.

  In a container, YOUR APP is PID 1 (e.g., node server.js).
  Most apps are NOT designed to handle zombie reaping.

  Result: Long-running containers accumulate zombie processes
          → memory leaks → eventual container instability.

  SYMPTOMS:
  docker stats shows container RAM slowly growing over days.
  ps aux inside container shows defunct processes.

  FIX OPTION 1: Use tini as your entrypoint (tiny init process)
  FIX OPTION 2: Docker has --init flag: docker run --init myapp
  FIX OPTION 3: Use dumb-init (from Yelp)

  In Dockerfile:
  RUN apk add --no-cache tini
  ENTRYPOINT ["/sbin/tini", "--"]
  CMD ["node", "server.js"]
```

### NET Namespace — Container Networking Basics

```
  Each container gets its own network namespace:

  HOST                          CONTAINER A          CONTAINER B
  ─────────────────             ────────────         ────────────
  eth0: 192.168.1.5             eth0: 172.17.0.2     eth0: 172.17.0.3
  lo:   127.0.0.1               lo:   127.0.0.1       lo:   127.0.0.1
  docker0: 172.17.0.1
       │                             │                    │
       └─────────────────────────────┴────────────────────┘
                         Virtual bridge (docker0)
                         Host routes traffic between them

  Container A → Container B:  172.17.0.3:port
  Host → Container A:         localhost:hostport (via port mapping)
  Container → Internet:       via host NAT (masquerade)
```

---

## 2.3 cgroups — Resource Limits

**Control Groups (cgroups)** limit, account for, and isolate the resource usage
(CPU, memory, disk I/O, network) of a collection of processes.

```
  Without cgroups:                    With cgroups:

  [Container A: runaway loop]         [Container A: max 0.5 CPU, 256MB RAM]
  [Container B: web server  ]         [Container B: max 1.0 CPU, 512MB RAM]
  [Container C: database    ]         [Container C: max 2.0 CPU, 2GB RAM]

  A consumes all CPU → B and C        A hits limit → throttled by kernel
  get starved → production outage     B and C unaffected → system stable
```

### cgroups v1 vs v2

```
  cgroups v1 (legacy, still common):
  - Different controllers in separate hierarchies
  - /sys/fs/cgroup/memory/, /sys/fs/cgroup/cpu/ etc.
  - Docker used this by default for many years

  cgroups v2 (modern, unified):
  - Single unified hierarchy
  - /sys/fs/cgroup/
  - Better resource accounting, memory limits more accurate
  - Required for Kubernetes 1.25+ with cgroupDriver: systemd
  - Default on Ubuntu 22.04+, RHEL 9+, Fedora 31+

  Check which version your host uses:
  $ stat -fc %T /sys/fs/cgroup/
  tmpfs    → cgroups v1
  cgroup2fs → cgroups v2
```

### Key cgroup Knobs You'll Use

```
  RESOURCE    DOCKER FLAG              WHAT IT DOES
  ──────────────────────────────────────────────────────────────
  Memory      --memory=512m            Hard limit. OOM killer
                                       fires if exceeded.

  Memory      --memory-reservation=    Soft limit. Kernel tries
              256m                     to stay under this.

  CPU         --cpus=0.5               Limit to 50% of one core

  CPU         --cpu-shares=512         Relative weight when
                                       CPU is contested
                                       (default 1024)

  CPU set     --cpuset-cpus=0,1        Pin to specific cores
                                       (useful for NUMA systems)

  Disk I/O    --device-read-bps        Throttle read speed
              /dev/sda:10mb

  PIDs        --pids-limit=100         Max number of processes
                                       (prevents fork bombs)
```

### ⚠️ cgroup Gotcha — OOM Kill in Production

```
  Scenario: Your app container is killed with exit code 137.
  
  Exit code 137 = 128 + 9 = SIGKILL by OOM killer

  This happens when container exceeds --memory limit.

  HOW TO DIAGNOSE:
  $ docker inspect <container_id> | grep -i oom
  "OOMKilled": true

  On the host:
  $ dmesg | grep -i "killed process"

  COMMON CAUSES:
  1. Memory leak in app (e.g., Node.js not releasing buffers)
  2. Limit set too low for actual workload
  3. JVM not respecting container limits (old Java versions)
     Java 8u191+ and Java 11+ are container-aware.
     Older Java reads HOST RAM, not container limit → sets
     heap too high → OOM kill.

  FIX FOR JAVA (Dockerfile):
  ENV JAVA_OPTS="-XX:+UseContainerSupport \
                 -XX:MaxRAMPercentage=75.0"
```

---

## 2.4 UnionFS & Image Layers — The Layered Filesystem

This is one of the most important concepts for understanding Docker image efficiency.

### What is a Union Filesystem?

A Union filesystem allows multiple directories (layers) to be **stacked on top of each other**,
appearing as a single unified directory tree.

```
  Building an nginx image step by step:

  Dockerfile:
  ──────────
  FROM ubuntu:22.04          ← Layer 1
  RUN apt-get update         ← Layer 2
  RUN apt-get install nginx  ← Layer 3
  COPY nginx.conf /etc/      ← Layer 4
  EXPOSE 80                  (metadata, not a layer)

  On disk (OverlayFS):
  ─────────────────────────────────────────────────────
  Layer 4: [nginx.conf at /etc/nginx/nginx.conf]
      ↑ stacked on top of
  Layer 3: [nginx binary, libs at /usr/sbin/nginx ...]
      ↑ stacked on top of
  Layer 2: [updated apt lists at /var/lib/apt/...]
      ↑ stacked on top of
  Layer 1: [ubuntu base filesystem: /bin /usr /etc ...]
  ─────────────────────────────────────────────────────
  Merged view: complete filesystem the container sees
```

### Layer Caching — The Build Speed Superpower

```
  FIRST BUILD:                       SECOND BUILD (changed nginx.conf):

  Step 1: FROM ubuntu      → PULL    Step 1: FROM ubuntu      → CACHED ✓
  Step 2: RUN apt-get upd  → RUN     Step 2: RUN apt-get upd  → CACHED ✓
  Step 3: RUN install nginx→ RUN     Step 3: RUN install nginx→ CACHED ✓
  Step 4: COPY nginx.conf  → COPY    Step 4: COPY nginx.conf  → RUN (changed)
  
  Total: ~3 minutes                  Total: ~2 seconds!

  KEY RULE: Once a layer is invalidated (file changed, command
  changed), ALL subsequent layers are rebuilt. Cache is
  invalidated in order from top to bottom.
```

### ⚠️ Layer Order Gotcha — The #1 Dockerfile Mistake

```
  BAD — slow builds, wastes cache:

  FROM node:18
  COPY . /app          ← copies ALL code first
  WORKDIR /app
  RUN npm install      ← re-runs every time ANY file changes

  GOOD — fast builds, uses cache:

  FROM node:18
  WORKDIR /app
  COPY package*.json . ← only copy dependency manifest first
  RUN npm install      ← only re-runs when package.json changes
  COPY . /app          ← copy rest of code last

  Why? package.json changes rarely. Source code changes constantly.
  Putting npm install AFTER source COPY = cache miss every build.
  Putting npm install BEFORE source COPY = cache hit 95% of builds.
```

### OverlayFS — The Modern UnionFS Implementation

```
  OverlayFS structure (what Docker uses by default on Linux):

  ┌─────────────────────────────────────────────────────┐
  │  Container Merged View (what process sees)          │
  └───────────────────┬─────────────────────────────────┘
                      │ overlayfs merge
          ┌───────────┴───────────┐
          │                       │
  ┌───────┴──────┐        ┌───────┴──────┐
  │  upperdir    │        │  lowerdir    │
  │  (writable   │        │  (read-only  │
  │   container  │        │   image      │
  │   layer)     │        │   layers)    │
  └──────────────┘        └──────────────┘
   Lives on host disk      Shared across all
   Deleted when container  containers using
   is removed              this image

  Copy-on-Write (CoW):
  When a container modifies a file from lowerdir,
  OverlayFS COPIES the file to upperdir first,
  then modifies it there. Original image layer untouched.
```

---

## 2.5 Docker Architecture — The Full Picture

### Components Overview

```
  ┌─────────────────────────────────────────────────────────────┐
  │                      Docker CLI                             │
  │  $ docker build    $ docker run    $ docker push            │
  └───────────────────────────┬─────────────────────────────────┘
                              │ REST API (Unix socket or TCP)
                              │ /var/run/docker.sock
                              ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                    Docker Daemon (dockerd)                  │
  │  - Manages images, containers, networks, volumes            │
  │  - Exposes Docker API                                       │
  │  - Delegates container execution to containerd              │
  └───────────────────────────┬─────────────────────────────────┘
                              │ gRPC
                              ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                     containerd                              │
  │  - Industry-standard container runtime (CNCF project)       │
  │  - Manages container lifecycle (start, stop, pause)         │
  │  - Manages image pulling, storage, snapshots                │
  │  - Used directly by Kubernetes (bypasses Docker entirely)   │
  └───────────────────────────┬─────────────────────────────────┘
                              │ OCI (Open Container Initiative)
                              ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                       runc                                  │
  │  - Low-level OCI runtime                                    │
  │  - Actually calls Linux kernel APIs                         │
  │  - Creates namespaces, cgroups, mounts OverlayFS           │
  │  - Spawns the container process                             │
  │  - Written in Go, maintained by Docker/OCI                  │
  └───────────────────────────┬─────────────────────────────────┘
                              │
                              ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                   Linux Kernel                              │
  │   namespaces  │  cgroups  │  OverlayFS  │  seccomp          │
  └─────────────────────────────────────────────────────────────┘
```

### The docker.sock File — Power and Danger

```
  Docker CLI communicates with dockerd via a Unix socket:
  /var/run/docker.sock

  ┌──────────┐   JSON over Unix socket   ┌──────────┐
  │ docker   │ ─────────────────────────►│ dockerd  │
  │  (CLI)   │ ◄───────────────────────── │ (daemon) │
  └──────────┘                           └──────────┘

  This socket is essentially a ROOT backdoor to the host.

  ⚠️ DANGER: Mounting docker.sock into a container:
  
  docker run -v /var/run/docker.sock:/var/run/docker.sock myapp
  
  The container can now:
  - Start/stop any container on the host
  - Mount host filesystem volumes
  - Escape to the host completely
  - Run: docker run --privileged -v /:/host ubuntu chroot /host
    → full root on the host

  WHEN IS IT USED LEGITIMATELY?
  - CI/CD agents that need to build Docker images (Docker-in-Docker)
  - Portainer, Watchtower, monitoring tools

  PRODUCTION RULE: Never mount docker.sock unless you explicitly
  understand and accept the security implications.
```

### Docker Registry Flow

```
  Developer                  Registry               Production Server
  ─────────                  ────────               ─────────────────

  $ docker build             Docker Hub              $ docker run
      │                      or ECR/GCR/GHCR             │
      │                            │                      │
      ▼                            │                      │
  [image built]                    │                      │
      │                            │                      │
      │  $ docker push             │                      │
      └────────────────────────────►                      │
                            [image stored]                │
                                   │                      │
                                   │  $ docker pull       │
                                   ◄──────────────────────┘
                                   │  (on first run or explicit pull)
                            [image layers sent]
                                   │
                                   └──────────────────────►
                                                    [container starts]

  Image naming convention:
  registry/namespace/repository:tag

  docker.io/library/nginx:1.25-alpine
  ──────── ─────── ───── ────────────
  registry  ns     repo    tag

  ghcr.io/myorg/myapp:v1.2.3-sha256abc
  └────── └────── └───── └────────────
  GitHub   org    repo    semantic + git sha (best practice)
```

---

## 2.6 containerd vs Docker — Why This Matters in 2024+

```
  History:
  ──────────────────────────────────────────────────────────────
  2013: Docker created (monolith: client + daemon + runtime)
  2016: Docker extracts containerd as a separate project
  2017: containerd donated to CNCF
  2020: Kubernetes deprecates dockershim (direct Docker support)
  2022: Kubernetes 1.24 removes dockershim completely

  What this means:
  ──────────────────────────────────────────────────────────────

  KUBERNETES DOESN'T USE DOCKER ANYMORE.
  
  Kubernetes speaks CRI (Container Runtime Interface).
  Docker's daemon does not implement CRI natively.
  containerd DOES implement CRI.

  So in a Kubernetes cluster today:
  
  kubelet → CRI → containerd → runc → container

  Docker is NOT in this chain.
  
  BUT: Images built with Docker are 100% compatible.
  The OCI image format is a standard. Docker images work
  with containerd, podman, buildah, kaniko, etc.

  PRACTICAL IMPACT ON YOU AS DEVOPS ENGINEER:
  ┌────────────────────────────────────────────────────┐
  │ Dev machine:  Docker Desktop is still fine         │
  │ K8s nodes:    Use containerd directly (crictl)     │
  │ CI builds:    docker build still works everywhere  │
  │ Debugging K8s pods: use crictl, not docker         │
  └────────────────────────────────────────────────────┘

  Useful commands on K8s nodes (using containerd):
  $ crictl ps           # list running containers
  $ crictl images       # list images
  $ crictl logs <id>    # container logs
  $ crictl exec -it <id> sh  # exec into container
```

---

## 2.7 Gotchas — Internals Edition

### ⚠️ Gotcha #6: Shared kernel means shared vulnerabilities

```
  All containers on a host share the kernel.
  A kernel CVE affects every container simultaneously.

  In contrast, a VM kernel CVE only affects that VM.

  EXAMPLE SCENARIO:
  Host kernel: 5.4.0 (has CVE-2022-0847 "Dirty Pipe")
  Container A: nginx (unprivileged)
  Container B: postgres
  Container C: your app

  With Dirty Pipe: a process in ANY container can overwrite
  read-only files on the host → full root escape.

  MITIGATION:
  - Keep host OS kernel patched and up to date
  - Use immutable infrastructure (replace nodes, don't patch in place)
  - Apply seccomp profiles (restrict dangerous syscalls)
  - Monitor with Falco (runtime security tool)
```

### ⚠️ Gotcha #7: Layer bloat — deleted files still take space in the image

```
  PROBLEMATIC DOCKERFILE:

  FROM ubuntu:22.04
  RUN apt-get update && apt-get install -y build-essential   ← Layer A: +400MB
  RUN make build                                              ← Layer B: +100MB
  RUN apt-get remove -y build-essential && apt-get clean     ← Layer C: -400MB?

  WRONG! The image is STILL ~500MB.
  Layer A still exists and is still downloaded. Layer C just
  adds a new layer that marks the files as deleted (whiteout files).
  The data is still in Layer A.

  CORRECT — chain commands in a single RUN:

  FROM ubuntu:22.04
  RUN apt-get update \
   && apt-get install -y build-essential \
   && make build \
   && apt-get remove -y build-essential \
   && apt-get clean \
   && rm -rf /var/lib/apt/lists/*

  Now it's ONE layer. Install + build + cleanup = single layer.
  Result: image is ~100MB, not ~500MB.
  
  EVEN BETTER: Use multi-stage builds (covered in Batch 6).
```

### ⚠️ Gotcha #8: docker.sock in CI is a security landmine

```
  Many CI setups mount the docker socket so the CI agent
  can build and push images:

  GitLab Runner config (dangerous pattern):
  [[runners]]
    [runners.docker]
      volumes = ["/var/run/docker.sock:/var/run/docker.sock"]

  If a malicious dependency in your build does:
  docker run -v /:/host --rm ubuntu cat /host/etc/shadow

  → It reads your host's /etc/shadow (password hashes).
  → It can write to host filesystem.
  → Your CI server is compromised.

  SAFER ALTERNATIVES:
  1. Docker-in-Docker (DinD) with --privileged in a clean container
     (isolated, but slower, and still uses privileged mode)
  2. Kaniko — builds Docker images without Docker daemon
     (runs as unprivileged container, K8s-native)
  3. Buildah — daemonless image building
  4. BuildKit with --oci-worker — rootless builds
```

---

## 2.8 Production FAQ — Batch 2

---

**Q: My container keeps getting OOMKilled. How do I find the right memory limit?**

```
A: You need to profile before you set limits. Follow this process:

  STEP 1: Run without limits and observe
  $ docker run --name profiling-run myapp
  $ docker stats profiling-run
  
  Watch the MEM USAGE column over time. Load test the app.
  Note: peak memory usage during normal load.

  STEP 2: Set limit to 20-30% above peak
  If peak was 400MB → set limit to 512MB
  docker run --memory=512m --memory-reservation=256m myapp
  
  STEP 3: Monitor in production
  Alert on memory usage >80% of limit.
  Prometheus metric: container_memory_usage_bytes
  
  STEP 4: For Java apps, explicitly set heap:
  -Xmx300m (hard heap limit)
  Total container memory = heap + metaspace + threads + JVM overhead
  Rule of thumb: container limit = -Xmx value × 1.5
```

---

**Q: I see the same image being pulled multiple times on different servers. How do I speed this up?**

```
A: Pull-through caches and mirroring. Options:

  OPTION 1: Docker Registry Mirror (self-hosted)
  Run a local registry that caches Docker Hub pulls.
  Configure Docker daemon on each node:

  /etc/docker/daemon.json:
  {
    "registry-mirrors": ["https://my-registry-mirror.internal"]
  }
  
  OPTION 2: Amazon ECR Pull Through Cache
  ECR can proxy Docker Hub/GCR/Quay with one config line.
  Pulls are then served from same-region ECR → fast + no rate limits.
  
  OPTION 3: Pre-pull images in your AMI/VM image (baking)
  Include docker pull commands in your Packer/golden image build.
  All nodes start with base images already cached.
  
  WHY THIS MATTERS:
  Docker Hub rate limits: 100 pulls/6hr (anonymous), 200 (free account)
  In Kubernetes with 50 nodes starting at once, you'll hit this fast.
```

---

**Q: We have 3 teams using the same Docker host. How do we prevent one team's container from affecting others?**

```
A: Several layers of isolation:

  RESOURCE ISOLATION (cgroups):
  Set memory + CPU limits on every container.
  Use --pids-limit to prevent fork bombs.
  Consider --blkio-weight for disk I/O fairness.

  NETWORK ISOLATION:
  Put each team's containers in a separate Docker network.
  They can't communicate by default across networks.
  docker network create team-a-net
  docker run --network team-a-net myapp

  NAMESPACE ISOLATION (user namespaces):
  Enable userns-remap in Docker daemon config.
  Remaps root in containers to unprivileged host user.
  Even if container is root, it's unprivileged on host.

  REALLY STRONG ISOLATION: Use separate VMs per team.
  Shared Docker hosts for multi-tenant workloads is
  inherently risky. VMs give you the hard boundary.
```

---

**Q: What's the actual difference between containerd and Docker? Do I need to change anything?**

```
A: For day-to-day development: almost nothing changes.

  BUILD:   docker build still works. (dockerd calls containerd)
  RUN:     docker run still works.
  PUSH:    docker push still works.
  COMPOSE: docker compose still works.

  WHERE IT MATTERS:
  
  1. Kubernetes nodes no longer have Docker.
     kubectl exec still works (via CRI).
     But: ssh to node + docker ps = command not found.
     Use: crictl ps  instead.
  
  2. If you were using docker logs on nodes directly:
     Use: crictl logs <container-id>
  
  3. If you were using docker inspect on nodes:
     Use: crictl inspect <container-id>
  
  4. Some Docker-specific features (docker swarm, docker events)
     don't exist in containerd.
  
  QUICK MENTAL MODEL:
  Docker = containerd + CLI + BuildKit + Compose + Swarm + UX layer
  containerd = just the runtime, lean, CRI-compatible
```

---

*End of Batch 2.*

---

# BATCH 3
## Docker CLI Deep Dive + Images + Dockerfile Fundamentals

> **Goal:** Go from "I know docker run" to commanding Docker with precision.
> Learn every CLI group, understand image internals, and write Dockerfiles
> that are clean, cache-friendly, and production-safe from day one.

---

## 3.1 The Docker CLI — Mental Map

The Docker CLI is organized into logical command groups. Most beginners only
know `docker run`, `docker build`, and `docker ps`. Here's the full map:

```
  docker
  ├── Container lifecycle
  │   ├── run          Start a new container from an image
  │   ├── start        Start a stopped container
  │   ├── stop         Graceful stop (SIGTERM → SIGKILL after timeout)
  │   ├── kill         Immediate stop (SIGKILL or custom signal)
  │   ├── restart      Stop + start
  │   ├── pause        Freeze all container processes (SIGSTOP)
  │   ├── unpause      Resume frozen container
  │   ├── rm           Remove stopped container
  │   └── wait         Block until container stops, print exit code
  │
  ├── Container inspection
  │   ├── ps           List running containers (-a for all)
  │   ├── inspect      Full JSON metadata of container/image/network
  │   ├── logs         Stdout/stderr of container
  │   ├── stats        Live CPU/RAM/net/IO metrics
  │   ├── top          Processes inside a container (like ps)
  │   ├── diff         Files changed in container's writable layer
  │   └── port         Show port mappings
  │
  ├── Container interaction
  │   ├── exec         Run command inside running container
  │   ├── attach       Attach to PID 1's stdio (careful!)
  │   └── cp           Copy files between container and host
  │
  ├── Image management
  │   ├── build        Build image from Dockerfile
  │   ├── pull         Download image from registry
  │   ├── push         Upload image to registry
  │   ├── images       List local images
  │   ├── rmi          Remove image
  │   ├── tag          Create alias tag for image
  │   ├── history      Show layers of an image
  │   ├── inspect      JSON metadata of an image
  │   └── save/load    Export/import image as tar archive
  │
  ├── Network management
  │   ├── network ls       List networks
  │   ├── network create   Create network
  │   ├── network inspect  Details of a network
  │   ├── network connect  Connect container to network
  │   └── network rm       Remove network
  │
  ├── Volume management
  │   ├── volume ls        List volumes
  │   ├── volume create    Create named volume
  │   ├── volume inspect   Details + mountpoint on host
  │   └── volume rm/prune  Remove volumes
  │
  └── System
      ├── system df        Disk usage (images, containers, volumes)
      ├── system prune     Remove all unused resources
      ├── info             Docker daemon info
      └── version          Client + server version
```

---

## 3.2 `docker run` — Every Flag You Need to Know

`docker run` is the most used command. It creates AND starts a container.

```
  docker run [OPTIONS] IMAGE [COMMAND] [ARGS]
```

### Flags reference with context:

```
  EXECUTION MODE
  ──────────────
  -d, --detach          Run in background (daemon mode)
                        Without -d: terminal is attached, Ctrl+C kills it
  
  -it                   -i (interactive) + -t (pseudo-TTY)
                        Use for shells: docker run -it ubuntu bash
  
  --rm                  Auto-remove container when it exits
                        Great for one-off commands and CI tasks
  
  --name myapp          Give the container a name (else random e.g. "brave_tesla")
                        Named containers are easier to reference in scripts

  RESOURCE LIMITS
  ───────────────
  --memory=512m         Hard memory limit (m=MB, g=GB)
  --memory-swap=1g      Total memory+swap limit (-1 = unlimited swap)
  --cpus=1.5            Limit to 1.5 CPU cores
  --cpu-shares=512      Relative CPU weight (default 1024)
  --pids-limit=200      Max process count (prevents fork bombs)

  NETWORKING
  ──────────
  -p 8080:80            Map host port 8080 → container port 80
  -p 127.0.0.1:8080:80  Bind to localhost ONLY (more secure)
  -p 80                 Map container port 80 to random host port
  --network mynet       Attach to named network
  --network host        Use host network stack directly (Linux only)
  --dns 8.8.8.8         Set container DNS server

  FILESYSTEM
  ──────────
  -v /host/path:/container/path    Bind mount (host path → container)
  -v myvolume:/container/path      Named volume mount
  --mount type=tmpfs,dst=/tmp      tmpfs (RAM-backed, fast, ephemeral)
  --read-only                      Make root filesystem read-only
  -w /app                          Set working directory inside container

  ENVIRONMENT
  ───────────
  -e KEY=VALUE          Set environment variable
  --env-file .env       Load env vars from a file
  
  IDENTITY & SECURITY
  ───────────────────
  -u 1000:1000          Run as UID:GID (not root)
  --cap-drop ALL        Drop all Linux capabilities
  --cap-add NET_BIND_SERVICE  Re-add specific capability
  --security-opt no-new-privileges  Prevent privilege escalation
  --privileged          Give ALL capabilities + host devices (NEVER in prod)

  RESTART POLICY
  ──────────────
  --restart no          Never restart (default)
  --restart always      Always restart (even on docker daemon restart)
  --restart unless-stopped  Restart except when manually stopped
  --restart on-failure:3    Restart on non-zero exit, max 3 times
```

### Common run patterns:

```
  # Disposable shell in an image (explore filesystem)
  docker run --rm -it alpine sh

  # Run a command and get output, then clean up
  docker run --rm ubuntu:22.04 cat /etc/os-release

  # Background web server with port mapping and name
  docker run -d --name webserver -p 8080:80 --restart unless-stopped nginx

  # Secure production-like run
  docker run -d \
    --name myapp \
    --memory=512m \
    --cpus=1.0 \
    --pids-limit=100 \
    -u 1000:1000 \
    --read-only \
    --cap-drop ALL \
    --security-opt no-new-privileges \
    -e APP_ENV=production \
    --restart unless-stopped \
    -p 127.0.0.1:3000:3000 \
    myrepo/myapp:v1.2.3
```

---

## 3.3 Inspecting and Debugging Containers

These are the commands you'll reach for every time something goes wrong.

### `docker logs` — Reading Container Output

```
  docker logs myapp                  # All logs since start
  docker logs --tail 100 myapp       # Last 100 lines
  docker logs --follow myapp         # Stream live (like tail -f)
  docker logs --since 30m myapp      # Last 30 minutes
  docker logs --since 2024-01-15T10:00:00 myapp  # Since timestamp
  docker logs --timestamps myapp     # Add timestamp to each line

  WHERE LOGS COME FROM:
  Docker captures everything written to STDOUT and STDERR.
  If your app logs to a file (/app/logs/app.log), docker logs
  won't show it. Best practice: log to STDOUT always.

  LOG DRIVERS (where logs go on the host):
  Default: json-file → /var/lib/docker/containers/<id>/<id>-json.log

  Production drivers:
  --log-driver=awslogs    → CloudWatch Logs
  --log-driver=fluentd    → Fluentd/Fluent Bit
  --log-driver=splunk     → Splunk
  --log-driver=none       → Discard all logs (useful for noisy sidecars)

  ⚠️ GOTCHA: json-file driver with no size limit fills disk.
  Always configure in /etc/docker/daemon.json:
  {
    "log-driver": "json-file",
    "log-opts": {
      "max-size": "50m",
      "max-file": "3"
    }
  }
  This keeps max 3 files × 50MB = 150MB per container.
```

### `docker exec` — Getting Inside a Running Container

```
  docker exec -it myapp sh            # sh shell (Alpine, minimal images)
  docker exec -it myapp bash          # bash shell (Ubuntu/Debian based)
  docker exec -it myapp /bin/bash     # explicit path
  docker exec myapp ls /app           # run single command, no TTY
  docker exec -u root myapp whoami    # exec as root even if container runs as user
  docker exec -e DEBUG=true myapp ./run-debug.sh  # inject env var

  COMMON DEBUGGING WORKFLOW:
  
  1. Check if process is running:
     docker exec myapp ps aux
  
  2. Check open ports inside container:
     docker exec myapp ss -tlnp
     docker exec myapp netstat -tlnp   # if netstat available
  
  3. Check filesystem:
     docker exec myapp ls -la /app
     docker exec myapp cat /app/config.json
  
  4. Check env vars inside container:
     docker exec myapp env | sort
  
  5. Check network connectivity from inside:
     docker exec myapp curl -v http://other-service:8080/health
     docker exec myapp nslookup postgres   # DNS resolution check

  ⚠️ GOTCHA: Many production images (distroless, scratch-based)
  have NO shell at all. You can't docker exec into them.
  
  ALTERNATIVES FOR SHELL-LESS CONTAINERS:
  # Use an ephemeral debug sidecar (K8s):
  kubectl debug -it mypod --image=busybox --target=mycontainer
  
  # Copy a static binary in at build time for debugging:
  COPY --from=busybox /bin/sh /debug-sh
```

### `docker inspect` — The Full Truth

```
  docker inspect myapp                # Full JSON (container)
  docker inspect nginx:alpine         # Full JSON (image)
  
  # Extract specific fields with Go template:
  docker inspect -f '{{.State.Status}}' myapp
  docker inspect -f '{{.State.ExitCode}}' myapp
  docker inspect -f '{{.NetworkSettings.IPAddress}}' myapp
  docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' myapp
  docker inspect -f '{{.HostConfig.Memory}}' myapp   # Memory limit in bytes
  docker inspect -f '{{json .Config.Env}}' myapp | python3 -m json.tool

  USEFUL INSPECT PATTERNS IN PRODUCTION:
  
  # Find all containers with a specific image
  docker inspect $(docker ps -q) --format '{{.Name}} {{.Config.Image}}' 2>/dev/null

  # Check if OOM killed
  docker inspect myapp | grep -i oom
  "OOMKilled": true   ← means container was killed for exceeding memory limit

  # Find mounted volumes
  docker inspect -f '{{json .Mounts}}' myapp | python3 -m json.tool
```

### `docker stats` — Live Resource Monitoring

```
  docker stats                        # All running containers, live
  docker stats myapp postgres redis   # Specific containers
  docker stats --no-stream            # One snapshot, then exit (for scripts)
  docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"

  OUTPUT COLUMNS:
  ┌────────────┬───────────┬───────────────────┬───────────┬───────────────┐
  │ NAME       │ CPU %     │ MEM USAGE / LIMIT │ MEM %     │ NET I/O       │
  ├────────────┼───────────┼───────────────────┼───────────┼───────────────┤
  │ webapp     │ 12.5%     │ 210MiB / 512MiB   │ 41.0%     │ 1.2MB / 800KB │
  │ postgres   │ 0.8%      │ 380MiB / 1GiB     │ 37.1%     │ 500KB / 1MB   │
  └────────────┴───────────┴───────────────────┴───────────┴───────────────┘

  ⚠️ MEM USAGE includes file system cache. The real "working set"
  is lower. Don't panic if memory looks high — check MEM % vs limit.
```

---

## 3.4 Image Management

### Understanding Image IDs, Tags, and Digests

```
  An image has THREE identifiers:

  1. TAG  (mutable, human-readable):
     nginx:1.25-alpine
     myapp:latest
     myapp:v1.2.3
     Tags can be moved/overwritten. "latest" is just a convention.
     ⚠️ latest does NOT mean the newest image — it means the tag
        named "latest". It's often outdated or missing entirely.

  2. IMAGE ID (short content hash, local alias):
     $ docker images
     nginx  1.25-alpine  a64c8e14c3f5  2 weeks ago  42MB
     ────────────────────────────────
     a64c8e14c3f5 is the first 12 chars of the full SHA256 ID.
     Not globally unique — local shorthand only.

  3. DIGEST (immutable, globally unique):
     nginx@sha256:9522864dd661dcadfd9958f9e0de192a1fdda2c162a35668ab6ac42b465f0603
     This is the SHA256 of the image manifest.
     Pinning by digest = guaranteed same image every time, everywhere.

  PRODUCTION RULE — Pin by digest in critical deployments:
  FROM nginx@sha256:9522864dd661dcadfd9958f9e0de192a1fdda2c162a35668ab6ac42b465f0603
  
  Get digest: docker inspect --format='{{index .RepoDigests 0}}' nginx:1.25-alpine
```

### Image Layering — Inspecting What's Inside

```
  docker history nginx:alpine

  IMAGE         CREATED        CREATED BY                      SIZE
  ─────────     ─────────────  ──────────────────────────────  ──────
  a64c8e14c3f5  2 weeks ago    CMD ["nginx" "-g" "daemon of…   0B
  <missing>     2 weeks ago    STOPSIGNAL SIGQUIT              0B
  <missing>     2 weeks ago    EXPOSE map[80/tcp:{}]           0B
  <missing>     2 weeks ago    COPY docker-entrypoint.sh /…    4.62kB
  <missing>     2 weeks ago    RUN /bin/sh -c set -x && …      34.6MB
  <missing>     2 weeks ago    ENV PKG_RELEASE=1               0B
  <missing>     2 weeks ago    ENV NJS_VERSION=0.8.0           0B
  <missing>     2 weeks ago    ENV NGINX_VERSION=1.25.2        0B
  <missing>     2 weeks ago    /bin/sh -c #(nop) ADD file:…    7.34MB  ← Alpine base
  
  <missing> = layers from a parent image (not present as independent image locally)

  DIVE — A better image explorer:
  $ docker run --rm -it wagoodman/dive nginx:alpine
  Lets you browse each layer, see what files changed, and
  calculate image efficiency score.
```

### Cleaning Up — Reclaiming Disk Space

```
  DIAGNOSE FIRST:
  $ docker system df

  TYPE            TOTAL    ACTIVE   SIZE      RECLAIMABLE
  Images          47       12       18.4GB    14.2GB (77%)
  Containers      23       8        1.2GB     890MB (74%)
  Volumes         15       6        45.6GB    12.1GB
  Build Cache     -        -        3.1GB     3.1GB

  SELECTIVE CLEANUP:
  docker image prune          # Remove dangling images (untagged)
  docker image prune -a       # Remove ALL unused images (not used by any container)
  docker container prune      # Remove all stopped containers
  docker volume prune         # Remove all unused volumes ⚠️ DATA LOSS
  docker network prune        # Remove unused networks
  docker builder prune        # Remove build cache

  NUCLEAR OPTION:
  docker system prune -a --volumes
  Removes: stopped containers, unused networks, unused images,
           build cache, unused volumes.
  ⚠️ Will delete volume data. Never run on a database host.

  AUTOMATED CLEANUP IN PRODUCTION:
  # Add to crontab or systemd timer:
  # Remove containers/images unused for >24 hours, keep volumes
  docker system prune -af --filter "until=24h"
```

---

## 3.5 Writing Dockerfiles — From Zero to Production

A `Dockerfile` is a script of instructions that Docker executes top-to-bottom
to build an image layer by layer.

### Anatomy of a Dockerfile

```
  # ── Every instruction creates a layer (or metadata) ──────────────────────

  # 1. Base image — where you start
  FROM node:20-alpine3.18

  # 2. Metadata (no layer created)
  LABEL maintainer="team@company.com"
  LABEL version="1.0"

  # 3. Build arguments (passed at build time, not at runtime)
  ARG BUILD_ENV=production

  # 4. Environment variables (available at build AND runtime)
  ENV NODE_ENV=production \
      PORT=3000 \
      APP_DIR=/app

  # 5. Working directory (creates it if not exists)
  WORKDIR /app

  # 6. Copy files from build context to image
  COPY package*.json ./
  ADD https://example.com/config.json ./  # ADD can fetch URLs (avoid if possible)

  # 7. Run commands during BUILD (installs, compiles, etc.)
  RUN npm ci --only=production \
   && npm cache clean --force

  # 8. Copy rest of application code
  COPY --chown=node:node . .

  # 9. Expose a port (documentation only — does NOT publish)
  EXPOSE 3000

  # 10. Switch to non-root user
  USER node

  # 11. Health check (Docker/K8s will probe this)
  HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD wget -qO- http://localhost:3000/health || exit 1

  # 12. Default command at runtime (can be overridden)
  CMD ["node", "server.js"]
```

### ENTRYPOINT vs CMD — The Most Confused Pair

```
  ENTRYPOINT = the executable that ALWAYS runs
  CMD        = the default arguments to ENTRYPOINT (can be overridden)

  ┌─────────────────────────────────────────────────────────────────┐
  │ Dockerfile                 │ docker run image ARG │ Runs        │
  ├─────────────────────────────────────────────────────────────────┤
  │ CMD ["node","server.js"]   │ (no args)            │ node server │
  │                            │ bash                 │ bash        │ ← overrides CMD
  ├─────────────────────────────────────────────────────────────────┤
  │ ENTRYPOINT ["node"]        │ (no args)            │ node        │
  │ CMD ["server.js"]          │ other.js             │ node other  │ ← CMD overridden
  │                            │ --inspect server.js  │ node --insp │
  ├─────────────────────────────────────────────────────────────────┤
  │ ENTRYPOINT ["node"]        │ bash                 │ node bash   │ ← cannot override
  │ (no CMD)                   │                      │             │   ENTRYPOINT w/o
  │                            │                      │             │   --entrypoint flag
  └─────────────────────────────────────────────────────────────────┘

  BEST PRACTICE for applications:
  ENTRYPOINT ["node"]          ← the program
  CMD ["server.js"]            ← the default argument

  BEST PRACTICE for tools/scripts:
  ENTRYPOINT ["./entrypoint.sh"]   ← wrapper script that handles signals
  CMD []
```

### Shell Form vs Exec Form — The Signal Handling Trap

```
  SHELL FORM:                         EXEC FORM:
  CMD node server.js                  CMD ["node", "server.js"]
       ↓                                    ↓
  Docker runs:                        Docker runs:
  /bin/sh -c "node server.js"         node server.js   (directly)
       ↓                                    ↓
  Process tree:                       Process tree:
  PID 1: /bin/sh                      PID 1: node
  PID 2: node (child of sh)
       ↓
  docker stop sends SIGTERM to PID 1 (/bin/sh)
  /bin/sh does NOT forward signals to children!
  node never gets SIGTERM → waits for kill timeout (10s)
  → gets SIGKILL → UNGRACEFUL SHUTDOWN

  ALWAYS use exec form for CMD and ENTRYPOINT.
  CMD ["node", "server.js"]    ← PID 1 = node → gets SIGTERM directly
```

### COPY vs ADD

```
  COPY:
  - Copies files/dirs from build context to image
  - Simple, predictable, no surprises
  - PREFER THIS in almost all cases

  ADD:
  - Everything COPY does, PLUS:
    - Auto-extracts tar archives: ADD app.tar.gz /app/ (extracts it)
    - Fetches remote URLs: ADD https://example.com/file.txt /app/
  - ⚠️ Fetching URLs is bad practice — no caching, security risk
  - ⚠️ Auto-extraction can surprise you

  RULE: Use COPY unless you specifically need ADD's tar extraction.
        Never use ADD for URLs — use RUN curl or RUN wget instead
        so the layer is explicit and auditable.
```

### ARG vs ENV

```
  ARG:                              ENV:
  Available ONLY during build       Available during build AND runtime
  Not in the final image            Embedded in image, visible to container
  
  docker build --build-arg          docker run -e KEY=value
    KEY=value ...
  
  ⚠️ SECRET TRAP:
  ARG SECRET_KEY=abc123             These are visible in:
  RUN ./build.sh                    - docker history
                                    - docker inspect
                                    - Image layers
  
  NEVER pass secrets via ARG or ENV.
  Use Docker secrets, Vault, or runtime secret injection instead.
  
  CORRECT BUILD-TIME SECRETS (BuildKit):
  # syntax=docker/dockerfile:1
  FROM ubuntu
  RUN --mount=type=secret,id=mysecret cat /run/secrets/mysecret
  
  Build with:
  docker build --secret id=mysecret,src=./secret.txt .
  
  Secret is NEVER stored in any image layer.
```

---

## 3.6 Base Image Selection — The Foundation Matters

```
  IMAGE FAMILY COMPARISON:
  ──────────────────────────────────────────────────────────────────
  Base Image          Size      Shell   Package Mgr  Use Case
  ──────────────────────────────────────────────────────────────────
  ubuntu:22.04        77MB      bash    apt          General purpose
  debian:bookworm     117MB     bash    apt          Wider compatibility
  debian:slim         74MB      bash    apt          Slimmed debian
  alpine:3.18         7MB       sh      apk          Tiny, security-focused
  distroless/static   ~2MB      NONE    NONE         Minimal attack surface
  distroless/base     ~20MB     NONE    NONE         C/C++ apps
  scratch             0MB       NONE    NONE         Static Go binaries
  ──────────────────────────────────────────────────────────────────

  ECOSYSTEM-SPECIFIC OFFICIAL IMAGES (multi-layered):
  ──────────────────────────────────────────────────────────────────
  node:20             1.1GB    bash    apt+npm       Dev (too fat for prod)
  node:20-slim        245MB    bash    apt+npm       Better
  node:20-alpine      135MB    sh      apk+npm       Production Node
  python:3.12         1.0GB    bash    apt+pip       Dev
  python:3.12-slim    131MB    bash    apt+pip       Better
  python:3.12-alpine  57MB     sh      apk+pip       Smallest
  ──────────────────────────────────────────────────────────────────

  DECISION TREE:
  
  Need a tiny image?          → alpine or distroless
  Go binary?                  → scratch or distroless/static
  Java app?                   → eclipse-temurin:21-jre-alpine
  Python app?                 → python:3.12-slim or python:3.12-alpine
  Node app?                   → node:20-alpine
  Need glibc (not musl)?      → debian:slim (Alpine uses musl libc
                                 which can cause subtle C lib issues)
  Maximum security, no shell? → distroless (can't exec in, by design)
```

### ⚠️ Alpine musl libc Gotcha

```
  Alpine Linux uses musl libc instead of glibc.
  Most Linux software is built against glibc.

  This causes subtle bugs with:
  - Python C extensions compiled against glibc
  - Node.js native modules
  - Anything using DNS (musl has a different resolver)
  - Some JVM flags

  SYMPTOM: App works on debian-based image, crashes or
  behaves oddly on alpine.

  DIAGNOSIS:
  docker run --rm -it your-alpine-image ldd /usr/bin/python3
  Look for "not found" on glibc-dependent libs.

  FIX: Switch to debian:slim or python:3.12-slim base.
  The size difference (~70MB) is usually worth the stability.
```

---

## 3.7 Gotchas — CLI & Dockerfile Edition

### ⚠️ Gotcha #9: `.dockerignore` — The forgotten performance file

```
  Every docker build command sends the "build context" to the daemon.
  The build context = everything in your current directory.

  WITHOUT .dockerignore:
  $ docker build .
  Sending build context to Docker daemon  1.2GB    ← includes node_modules!
  
  This means 1.2GB is uploaded to the Docker daemon every build.
  Even if your Dockerfile never copies node_modules.

  CREATE .dockerignore in every project:
  ─────────────────────────────────────
  node_modules/
  .git/
  .gitignore
  README.md
  *.log
  .env
  .env.*
  dist/
  build/
  coverage/
  .DS_Store
  __pycache__/
  *.pyc
  .pytest_cache/
  ─────────────────────────────────────
  
  Result: Build context drops from 1.2GB to ~50KB. Much faster.
  Also prevents accidentally COPYing .env secrets into the image.
```

### ⚠️ Gotcha #10: `docker attach` is almost never what you want

```
  docker attach myapp

  This attaches your terminal to PID 1's stdin/stdout.
  Pressing Ctrl+C sends SIGINT to PID 1 → KILLS THE CONTAINER.

  Detach safely: Ctrl+P then Ctrl+Q (detach sequence)
  But most people don't know this and kill their container.

  USE INSTEAD:
  docker exec -it myapp sh    ← new process, Ctrl+C is safe
  docker logs --follow myapp  ← read-only log streaming
```

### ⚠️ Gotcha #11: Build context path vs Dockerfile path

```
  # This is NOT the same as this:
  
  docker build .                          # context = current dir
  docker build -f ./docker/Dockerfile .  # Dockerfile from ./docker/,
                                         # context STILL = current dir
  docker build -f Dockerfile ..          # context = parent dir!
  
  The build context determines what COPY can access.
  If you COPY ../shared-lib/ inside a Dockerfile but your
  context is the current dir, it will fail:
  
  COPY ../shared-lib /app/shared-lib  ← ERROR: path outside build context
  
  FIX: Set context to a parent directory that contains both:
  docker build -f services/api/Dockerfile .   ← context = root
```

---

## 3.8 Production FAQ — Batch 3

---

**Q: Our Docker images are 3GB+. How do we shrink them?**

```
A: Attack in order of impact:

  1. RIGHT-SIZE THE BASE IMAGE
     node:20 (1.1GB) → node:20-alpine (135MB) = -965MB immediately
     Just doing this one change often cuts 60-80% of image size.

  2. USE MULTI-STAGE BUILDS (Batch 6 covers this fully)
     Stage 1: build environment (big, has compilers, dev deps)
     Stage 2: runtime environment (tiny, only production artifacts)
     Result: final image has NO build tools, NO dev dependencies.

  3. CHAIN RUN COMMANDS
     Each RUN is a layer. Separate RUN commands = accumulated bloat.
     apt-get update + apt-get install + clean = one RUN.

  4. CLEAN UP IN THE SAME LAYER
     RUN apt-get update \
      && apt-get install -y build-essential \
      && make \
      && apt-get remove -y build-essential \
      && apt-get autoremove -y \
      && rm -rf /var/lib/apt/lists/*

  5. DON'T COPY UNNECESSARY FILES
     Use .dockerignore to exclude docs, tests, git history.

  6. USE DISTROLESS FOR FINAL STAGE
     gcr.io/distroless/nodejs20-debian12 is ~170MB vs node:20 at 1.1GB.

  TOOLS TO ANALYZE:
  docker run --rm -it wagoodman/dive your-image:tag
  Shows per-layer size breakdown and "wasted space" score.
```

---

**Q: How do I pass different config to the same image in dev vs prod?**

```
A: Never bake environment-specific config INTO the image.
   The image should be identical across environments.
   Config is injected at runtime. Three patterns:

   PATTERN 1 — Environment Variables (12-Factor App style)
   docker run -e DATABASE_URL=postgres://... myapp
   Best for: simple scalar config, secrets (from a secrets manager)

   PATTERN 2 — Config files via volume mount
   docker run -v /etc/myapp/config.json:/app/config.json myapp
   Best for: complex structured config, certs, feature flags

   PATTERN 3 — Init container or entrypoint script that
               pulls config from Vault/SSM at startup
   ENTRYPOINT ["./docker-entrypoint.sh"]
   # entrypoint fetches secrets, sets env vars, then execs $@
   Best for: production systems with a secrets manager

   NEVER DO THIS:
   COPY config.prod.json /app/config.json   ← prod secrets in image
   ENV DATABASE_PASSWORD=supersecret        ← visible in docker history
```

---

**Q: `docker build` is slow in CI. How do we speed it up?**

```
A: Layer cache is the key. Three strategies:

  STRATEGY 1: Mount a cache volume (GitHub Actions example)
  - name: Build Docker image
    uses: docker/build-push-action@v5
    with:
      cache-from: type=gha
      cache-to: type=gha,mode=max

  STRATEGY 2: Registry-based cache (works across machines)
  docker buildx build \
    --cache-from type=registry,ref=myregistry/myapp:buildcache \
    --cache-to   type=registry,ref=myregistry/myapp:buildcache,mode=max \
    -t myregistry/myapp:latest .

  STRATEGY 3: Optimize Dockerfile layer order (always do this)
  - COPY dependency manifest FIRST
  - RUN install dependencies
  - COPY source code LAST
  95% of builds will hit cache at the "install" step.

  MEASURE: Add --progress=plain to see per-step timing:
  docker build --progress=plain .
```

---

**Q: A container is running but the app inside isn't responding. Where do I start?**

```
A: Systematic debug ladder:

  STEP 1: Is the container actually running?
  docker ps -a | grep myapp
  Check STATUS column. "Exited (1)" = crashed.
  docker logs myapp --tail 50   ← check for error message

  STEP 2: Is the process alive inside the container?
  docker exec myapp ps aux
  Is your app process listed? Is it consuming CPU (stuck loop)?

  STEP 3: Is the port bound inside the container?
  docker exec myapp ss -tlnp
  Do you see your app listening on the expected port?

  STEP 4: Is the port reachable from inside the container?
  docker exec myapp curl -v http://localhost:3000/health
  If this works but external access fails → port mapping issue.

  STEP 5: Is the port mapping correct on the host?
  docker port myapp
  curl -v http://localhost:HOST_PORT/health

  STEP 6: Is there a network policy or firewall blocking?
  iptables -L -n | grep DOCKER    ← check Docker's iptables rules
  ufw status                       ← check ufw if enabled

  STEP 7: Check resource limits — is it being throttled?
  docker stats myapp
  High CPU % (near limit) → throttled. High MEM % → OOM risk.
  docker inspect myapp | grep -E 'OOMKilled|ExitCode'
```

---

*End of Batch 3.*

---

# BATCH 4
## Docker Networking — Bridge, Host, Overlay, DNS & Production Patterns

> **Goal:** Understand how containers talk to each other, to the host, and to the outside
> world. Networking is the #1 source of mysterious container failures in production —
> knowing it deeply separates good DevOps engineers from great ones.

---

## 4.1 The Networking Mental Model

Before diving into Docker specifics, anchor on this:

```
  Every container gets its OWN network namespace (from Batch 2).
  That means its own:
  - Network interfaces (eth0, lo)
  - IP address
  - Routing table
  - iptables rules
  - Port space (container can bind :80 even if host uses :80)

  Docker's job: wire these isolated namespaces together
                in different topologies depending on need.

  FOUR BUILT-IN NETWORK DRIVERS:
  ┌──────────────┬──────────────────────────────────────────────┐
  │ Driver       │ Use Case                                     │
  ├──────────────┼──────────────────────────────────────────────┤
  │ bridge       │ Default. Containers on same host talk        │
  │              │ via virtual switch. Most common.             │
  ├──────────────┼──────────────────────────────────────────────┤
  │ host         │ Container shares host network namespace.     │
  │              │ No isolation. Max performance.               │
  ├──────────────┼──────────────────────────────────────────────┤
  │ overlay      │ Multi-host networking. Swarm / K8s.          │
  │              │ Tunnels packets across hosts (VXLAN).        │
  ├──────────────┼──────────────────────────────────────────────┤
  │ none         │ No network at all. Fully air-gapped.         │
  │              │ Use for batch jobs, security sandboxes.      │
  └──────────────┴──────────────────────────────────────────────┘
```

---

## 4.2 Bridge Networking — The Default

When you run `docker run nginx` without specifying a network, Docker connects
it to the **default bridge network** (`docker0`).

### How the Default Bridge Works

```
  HOST MACHINE
  ┌──────────────────────────────────────────────────────────┐
  │                                                          │
  │  eth0 (host NIC): 192.168.1.10  ◄── public/LAN traffic  │
  │         │                                                │
  │         │  NAT (iptables MASQUERADE)                     │
  │         │                                                │
  │  docker0 (virtual bridge): 172.17.0.1                    │
  │     │              │              │                      │
  │  veth0          veth1          veth2   ← virtual cables  │
  │     │              │              │                      │
  │  ┌──┴──┐        ┌──┴──┐        ┌──┴──┐                  │
  │  │ C1  │        │ C2  │        │ C3  │  containers       │
  │  │.0.2 │        │.0.3 │        │.0.4 │  172.17.0.x      │
  │  └─────┘        └─────┘        └─────┘                  │
  │                                                          │
  └──────────────────────────────────────────────────────────┘

  C1 → C2:  packet goes  C1 → veth0 → docker0 → veth1 → C2
  C1 → web: packet goes  C1 → docker0 → NAT → eth0 → internet
  Web → C1: requires port mapping (-p 8080:80)
```

### Port Mapping Deep Dive

```
  docker run -p 8080:80 nginx

  This does TWO things:
  1. Tells Docker to bind host port 8080
  2. Creates an iptables DNAT rule:

  iptables -t nat -A DOCKER -p tcp --dport 8080 \
    -j DNAT --to-destination 172.17.0.2:80

  Traffic flow:
  Browser → host:8080 → iptables DNAT → 172.17.0.2:80 → nginx

  CHECK THESE RULES:
  $ iptables -t nat -L -n -v   # see all NAT rules
  $ docker port mycontainer     # quick view of mappings

  PORT BINDING FORMATS:
  -p 8080:80            # all interfaces, host 8080 → container 80
  -p 127.0.0.1:8080:80  # ONLY localhost (safer, not public)
  -p 80                 # random host port → container 80
  -p 8080:80/udp        # UDP port mapping
```

### ⚠️ Default Bridge Gotcha — No DNS Between Containers

```
  DEFAULT bridge network (docker0):
  Containers communicate by IP only.
  NO automatic DNS. Container names don't resolve.

  docker run --name app1 myapp
  docker run --name db postgres

  Inside app1:
  $ ping db         → FAILS (name not resolved)
  $ ping 172.17.0.3 → WORKS (if you know the IP)

  BUT IPs change every restart! Hardcoding IP = fragile.

  FIX: Use a USER-DEFINED bridge network.
  Docker gives user-defined bridges automatic DNS.
```

### User-Defined Bridge — The Right Way

```
  $ docker network create myapp-net

  $ docker run --name db --network myapp-net postgres
  $ docker run --name app --network myapp-net myapp

  Inside app container:
  $ ping db         → WORKS ✅ (Docker DNS resolves 'db')
  $ curl db:5432    → WORKS ✅

  WHY IT WORKS:
  Docker runs an embedded DNS server at 127.0.0.11 inside
  each container on a user-defined network.
  It registers container names as DNS records automatically.

  $ cat /etc/resolv.conf (inside container)
  nameserver 127.0.0.11
  options ndots:0

  ALSO: User-defined bridges provide better isolation.
  Containers on different user-defined networks CANNOT
  communicate with each other by default.
```

### Bridge Network Comparison

```
  FEATURE                   DEFAULT BRIDGE    USER-DEFINED BRIDGE
  ──────────────────────────────────────────────────────────────
  Automatic DNS             ✗ No              ✅ Yes (by name)
  Container isolation       Partial           ✅ Full (per network)
  Connect/disconnect live   ✗ No              ✅ Yes
  Custom subnet/gateway     ✗ No              ✅ Yes
  Legacy --link support     ✅ Yes (deprecated) ✗ No
  Recommended for prod      ✗ No              ✅ Yes
```

---

## 4.3 Host Networking

```
  docker run --network host nginx

  Container shares the host's network namespace COMPLETELY.
  No veth pairs. No NAT. No port mapping needed.

  HOST MACHINE
  ┌──────────────────────────────────────────────────┐
  │                                                  │
  │  eth0: 192.168.1.10                              │
  │  lo:   127.0.0.1                                 │
  │                                                  │
  │  ┌─────────────────────────────────────────┐     │
  │  │  Container (host network mode)          │     │
  │  │  Sees SAME eth0, SAME lo                │     │
  │  │  Binds :80 directly on host interface   │     │
  │  └─────────────────────────────────────────┘     │
  └──────────────────────────────────────────────────┘

  BENEFITS:
  - Zero network overhead (no NAT, no veth hop)
  - Best raw throughput — useful for high-perf workloads
  - Container binds directly to host ports (no -p needed)

  DRAWBACKS:
  - No network isolation. Container can see ALL host interfaces.
  - Port conflicts: if nginx in container binds :80,
    nothing else on host can use :80.
  - Not available on Docker Desktop (Mac/Windows) — the Linux
    VM is the "host", not your Mac.

  PRODUCTION USE CASES:
  - High-throughput proxies (HAProxy, Envoy sidecars)
  - Network monitoring tools that need raw socket access
  - Performance-sensitive services where NAT overhead matters
```

---

## 4.4 Overlay Networking — Multi-Host

Overlay networks allow containers on **different hosts** to communicate as if
they were on the same local network. Used by Docker Swarm and as a concept
(implemented differently) in Kubernetes (CNI plugins).

```
  WITHOUT overlay:
  ┌─────────────┐           ┌─────────────┐
  │  Host A     │           │  Host B     │
  │  C1: .0.2   │  ──???──  │  C2: .0.2   │
  │  172.17.0.x │           │  172.17.0.x │
  └─────────────┘           └─────────────┘
  Same subnet, different machines — can't route!

  WITH overlay (VXLAN tunnel):
  ┌──────────────────────────────────────────────────────┐
  │              Overlay Network: 10.0.0.0/24            │
  │                                                      │
  │  ┌─────────────┐              ┌─────────────┐        │
  │  │  Host A     │              │  Host B     │        │
  │  │  C1:10.0.0.2│◄────────────►│  C2:10.0.0.3│       │
  │  │             │  VXLAN UDP   │             │        │
  │  │             │  port 4789   │             │        │
  │  └─────────────┘              └─────────────┘        │
  │   Physical IPs:                                      │
  │   192.168.1.10                192.168.1.11           │
  └──────────────────────────────────────────────────────┘

  VXLAN wraps the container's L2 Ethernet frame
  inside a UDP packet on the physical network.
  Containers see a flat L2 network regardless of host.

  HOW DOCKER OVERLAY WORKS:
  - Requires a key-value store for node discovery
    (Docker Swarm uses built-in Raft consensus)
  - Each host runs a VXLAN tunnel endpoint (VTEP)
  - Docker daemon programs the VTEP and FDB (forwarding table)
  - Packets: Container → VTEP → encapsulate → physical net
                       → remote VTEP → decapsulate → Container
```

### Overlay in Kubernetes Context

```
  Kubernetes doesn't use Docker overlay.
  It uses CNI (Container Network Interface) plugins:

  ┌─────────┬──────────────────────────────────────────┐
  │ Plugin  │ Mechanism                                 │
  ├─────────┼──────────────────────────────────────────┤
  │ Flannel │ VXLAN overlay (simple, like Docker)      │
  │ Calico  │ BGP routing (no overlay, very fast)      │
  │ Weave   │ Encrypted mesh overlay                   │
  │ Cilium  │ eBPF-based (fastest, most powerful)      │
  │ AWS VPC │ Direct VPC routing (cloud-native)        │
  └─────────┴──────────────────────────────────────────┘

  The K8s networking model requirements:
  - Every Pod gets a unique IP
  - All Pods can communicate without NAT
  - Nodes can communicate with all Pods without NAT
  CNI plugins implement these rules differently.
```

---

## 4.5 Docker DNS — How Container Discovery Works

```
  Inside a user-defined network, Docker runs an embedded
  DNS resolver at 127.0.0.11 (magic IP, always available).

  DNS RESOLUTION ORDER inside a container:
  1. Check /etc/hosts (Docker injects container's own hostname)
  2. Query 127.0.0.11 (Docker embedded DNS)
  3. Docker DNS checks its records:
     - Container names on same network → resolves to container IP
     - Service names (in Swarm/Compose) → resolves to VIP
     - Unknown → forwards to host's DNS resolver
  4. Host DNS → upstream DNS (8.8.8.8 etc.)

  INSPECT DNS CONFIG:
  $ docker run --rm alpine cat /etc/resolv.conf
  nameserver 127.0.0.11
  options ndots:0

  OVERRIDE DNS IN CONTAINER:
  docker run --dns 1.1.1.1 myapp         # use Cloudflare DNS
  docker run --dns-search internal.corp  # add search domain

  OVERRIDE DNS FOR ALL CONTAINERS (daemon.json):
  {
    "dns": ["1.1.1.1", "8.8.8.8"],
    "dns-search": ["internal.corp"]
  }
```

### ⚠️ DNS Gotcha — ndots and Slow Lookups

```
  Default Linux DNS config has ndots:5.
  Docker user-defined networks set ndots:0.

  ndots:5 means: if hostname has fewer than 5 dots,
  try appending each search domain before going to root.

  Example with ndots:5 and search domain "cluster.local":
  Lookup "mydb" →
    1. Try mydb.cluster.local   (fails, waits timeout)
    2. Try mydb.svc.cluster.local (fails, waits timeout)
    3. Try mydb.  (absolute, finally resolves)
  
  Each failed attempt = 5ms delay × 3 = ~15ms overhead
  On every DB connection = measurable latency in prod.

  FIX in K8s: Use fully-qualified names: mydb.default.svc.cluster.local
  Or set ndots:2 in pod's dnsConfig:
  spec:
    dnsConfig:
      options:
        - name: ndots
          value: "2"
```

---

## 4.6 Network Debugging Toolkit

```
  PROBLEM                    COMMAND
  ─────────────────────────────────────────────────────────────
  List all networks          docker network ls

  Inspect a network          docker network inspect myapp-net
  (see IPs, containers)

  Which network is           docker inspect mycontainer \
  container on?                | grep -A 20 '"Networks"'

  Can container reach         docker exec myapp ping db
  another container?

  Can container reach         docker exec myapp curl -v google.com
  internet?

  DNS resolution working?    docker exec myapp nslookup db
                             docker exec myapp cat /etc/resolv.conf

  What ports are exposed?    docker port mycontainer
                             docker inspect mycontainer | grep -i port

  Show iptables NAT rules     sudo iptables -t nat -L -n -v | grep DOCKER

  Network bandwidth test      docker run --rm --network container:target \
                               networkstatic/iperf3 -c target

  Packet capture on           sudo tcpdump -i docker0 -n
  bridge interface

  Capture inside container   docker run --rm --network container:myapp \
                              nicolaka/netshoot tcpdump -i eth0
```

### The Netshoot Swiss Army Knife

```
  nicolaka/netshoot is an image packed with networking tools.
  The DevOps go-to for container network debugging.

  $ docker run --rm -it --network container:myapp nicolaka/netshoot

  Tools available inside:
  ping, curl, wget, nslookup, dig, nmap, netstat, ss,
  iperf3, tcpdump, traceroute, iftop, mtr, ip, arp

  Common patterns:
  # Debug DNS from inside another container's network namespace
  docker run --rm --network container:myapp nicolaka/netshoot dig db

  # Check if port is open from container's perspective
  docker run --rm --network container:myapp nicolaka/netshoot \
    nmap -p 5432 db
```

---

## 4.7 Production Network Patterns

### Pattern 1 — Frontend / Backend / DB Segmentation

```
  GOAL: Web tier talks to API, API talks to DB.
        Web tier CANNOT talk directly to DB.

  docker network create frontend-net
  docker network create backend-net

  docker run --name nginx --network frontend-net -p 443:443 nginx
  docker run --name api   --network frontend-net \
                          --network backend-net  myapi
  docker run --name db    --network backend-net  postgres

  RESULT:
  ┌──────────────┐    frontend-net    ┌──────────────┐
  │    nginx     │◄──────────────────►│     api      │
  │  (public)    │                    │              │
  └──────────────┘                    │              │
  cannot reach db                     └──────┬───────┘
  directly ✅                                │ backend-net
                                      ┌──────┴───────┐
                                      │      db      │
                                      │  (isolated)  │
                                      └──────────────┘

  nginx container is on frontend-net only → can't reach db.
  api is on BOTH networks → the bridge between tiers.
  db is on backend-net only → not reachable from internet.
```

### Pattern 2 — Sidecar / Ambassador

```
  Use case: Inject a proxy (Envoy, nginx) as a sidecar
  that shares the network namespace with the main app.

  docker run --name app myapp
  docker run --network container:app envoy-proxy

  Both containers share ONE network namespace.
  Envoy can intercept :8080 traffic before app sees it.
  This is how Istio service mesh works in Kubernetes.

  ┌─────────────────────────────┐
  │  Shared Network Namespace   │
  │                             │
  │  [Envoy :8080] → [App :3000]│
  │                             │
  │  Single IP from outside     │
  └─────────────────────────────┘
```

### Pattern 3 — Exposing Only What's Needed

```
  NEVER expose internal services to host or internet.

  BAD:
  docker run -p 6379:6379 redis    ← Redis exposed on all interfaces!
                                     Anyone on network can connect.

  GOOD — only localhost:
  docker run -p 127.0.0.1:6379:6379 redis

  BETTER — don't expose at all, use container DNS:
  docker run --name redis --network myapp-net redis
  # app connects to redis:6379 internally, no host port needed

  BEST — combine with network segmentation (Pattern 1 above)
```

---

## 4.8 Gotchas — Networking Edition

### ⚠️ Gotcha #9: `--network host` breaks on Docker Desktop (Mac/Windows)

```
  On Linux host:
  docker run --network host nginx
  → nginx binds directly to host's :80. Works perfectly.

  On Docker Desktop (Mac or Windows):
  → Docker runs inside a Linux VM.
  → "host" network = the VM's network, NOT your Mac's.
  → Host port 80 on Mac is NOT the same as VM's port 80.
  → Behaviour differs from Linux production. Bugs hide here.

  FIX: Always test host-network containers on a Linux machine
  before deploying. Use a Linux VM (Multipass, UTM) if needed.
```

### ⚠️ Gotcha #10: Docker modifies iptables — conflicts with other tools

```
  Docker automatically manages iptables rules for:
  - Container-to-internet NAT (MASQUERADE)
  - Port mapping (DNAT rules)
  - Inter-container routing

  This can conflict with:
  - UFW (Uncomplicated Firewall) on Ubuntu
  - firewalld on CentOS/RHEL
  - Custom iptables rules you've written

  CLASSIC BUG:
  You add a UFW rule to block port 8080.
  Docker adds a DNAT rule for port 8080 BEFORE UFW's rules.
  Port 8080 is still accessible from the internet! UFW ignored.

  WHY: Docker inserts rules into DOCKER-USER chain (INPUT chain
  is evaluated BEFORE UFW's chains in some setups).

  FIX:
  1. Add your block rules to DOCKER-USER chain:
     iptables -I DOCKER-USER -p tcp --dport 8080 -j DROP

  2. Or bind ports to localhost only (-p 127.0.0.1:8080:80)
     Docker won't add a public DNAT rule.

  3. Or disable Docker's iptables management (advanced):
     "iptables": false in daemon.json
     (Then you manage all routing manually — expert mode)
```

### ⚠️ Gotcha #11: Container restarts get new IPs

```
  docker stop mydb && docker start mydb
  → mydb gets a NEW IP address from the bridge subnet pool.

  Any container that hardcoded mydb's IP will BREAK.

  This is exactly why:
  - Always use container DNS names, never IPs
  - Always use user-defined networks (automatic DNS)
  - In Compose / Swarm / K8s: use service names

  CHECK: docker inspect mydb | grep IPAddress
  You'll see the IP changed after restart.
```

---

## 4.9 Production FAQ — Batch 4

---

**Q: Two containers on the same host can't communicate. What's wrong?**

```
A: Work through this checklist:

  1. Are they on the same network?
     docker inspect c1 | grep -A5 '"Networks"'
     docker inspect c2 | grep -A5 '"Networks"'
     If different networks → they can't reach each other.
     FIX: Connect both to same user-defined network.
     docker network connect mynet c1

  2. Are they on the default bridge (docker0)?
     Default bridge has no DNS. Use IPs, or switch to
     a user-defined bridge.

  3. Is the app actually listening on 0.0.0.0?
     Some apps bind to 127.0.0.1 by default (localhost only).
     From another container, 127.0.0.1 = that container itself.
     FIX: App must bind to 0.0.0.0 (all interfaces).
     e.g. flask run --host=0.0.0.0

  4. Is there a firewall rule blocking it?
     docker exec c1 nc -zv c2 8080
     iptables -L -n | grep DROP

  5. Is the service actually running in c2?
     docker exec c2 ss -tlnp
     Check the port is bound.
```

---

**Q: We need containers on two different servers to communicate securely. What's the architecture?**

```
A: Several options, depending on your stack:

  OPTION 1: Docker Swarm overlay network
  Initialize swarm, create overlay network.
  Containers join the overlay and get routable IPs.
  Traffic is encrypted with AES-GCM (--opt encrypted).
  Good for: simple multi-host setups, VM-based deployments.

  OPTION 2: Kubernetes with CNI + Network Policies
  Use Calico or Cilium for encrypted pod-to-pod traffic.
  Network Policies control which pods can talk to which.
  Good for: production, scalable, policy-driven.

  OPTION 3: Service Mesh (Istio, Linkerd)
  Sidecar proxies handle mTLS between all services.
  Zero-trust networking — all traffic encrypted + authenticated.
  Good for: microservices, compliance requirements, observability.

  OPTION 4: VPN / WireGuard between hosts
  Establish a WireGuard tunnel between servers.
  Containers use regular bridge networking, traffic traverses VPN.
  Good for: simple setups, on-prem, cross-cloud.

  OPTION 5: Cloud VPC (AWS/GCP/Azure)
  VMs in same VPC have private routable IPs.
  Use VPC security groups / firewall rules to control access.
  Containers communicate via host IPs + port mappings.
```

---

**Q: My container can reach the internet but can't resolve internal company DNS. How do I fix it?**

```
A: Container DNS config doesn't inherit your VPN or corp DNS by default.

  DIAGNOSE:
  $ docker exec myapp cat /etc/resolv.conf
  # If it shows 8.8.8.8 only → missing corp DNS

  $ docker exec myapp nslookup internal.mycompany.com
  # Server failure or NXDOMAIN → confirms the problem

  FIX OPTION 1: Per-container DNS
  docker run --dns 10.10.0.53 --dns-search mycompany.com myapp

  FIX OPTION 2: Daemon-wide DNS (affects all containers)
  /etc/docker/daemon.json:
  {
    "dns": ["10.10.0.53", "8.8.8.8"],
    "dns-search": ["mycompany.com", "internal.mycompany.com"]
  }
  sudo systemctl reload docker

  FIX OPTION 3: Docker Compose
  services:
    myapp:
      dns:
        - 10.10.0.53
      dns_search:
        - mycompany.com

  VERIFY AFTER FIX:
  docker exec myapp nslookup internal.mycompany.com
```

---

**Q: We're moving to Kubernetes. Does our Docker networking knowledge transfer?**

```
A: Yes — the concepts transfer, but the implementation differs.

  DOCKER CONCEPT         K8S EQUIVALENT
  ────────────────────────────────────────────────────────
  User-defined network   Namespace (soft) + NetworkPolicy
  Container DNS name     Service DNS (myservice.namespace.svc)
  Port mapping (-p)      Service type: NodePort or LoadBalancer
  docker network create  CNI plugin handles automatically
  overlay network        CNI plugin (Flannel/Calico/Cilium)
  --network host         hostNetwork: true (in pod spec)
  Bridge isolation       NetworkPolicy (explicit allow/deny)

  KEY DIFFERENCES:
  1. In K8s, all Pods can talk to all Pods by default (flat network).
     You ADD restrictions via NetworkPolicy. (Opposite of Docker!)
  2. Services (ClusterIP) give stable DNS + VIP for a group of pods.
     You don't connect to pod IPs directly — use Service names.
  3. Ingress replaces your -p 80:80 port mapping for HTTP.
  4. No docker0 bridge — CNI plugin creates its own interfaces.

  WHAT STAYS THE SAME:
  - Container-to-container DNS by name
  - Understanding namespaces, network isolation
  - Port concepts, L4 vs L7 routing intuition
  - Overlay/VXLAN networking concepts
```

---

*End of Batch 4.*

---

# BATCH 5
## Docker Volumes & Storage — Persistence, Bind Mounts, tmpfs & Stateful Patterns

> **Goal:** Understand exactly where data lives, how it survives (or doesn't) container
> restarts, and how to architect persistent storage for databases, caches, and stateful
> services in production. Storage is where most beginner Docker mistakes happen.

---

## 5.1 The Core Problem — Container Ephemerality

```
  Containers are designed to be throwaway.

  CONTAINER LIFECYCLE:
  ┌─────────────────────────────────────────────────────┐
  │                                                     │
  │  docker run myapp                                   │
  │       │                                             │
  │       ▼                                             │
  │  [Container starts]                                 │
  │  [App writes to /var/log/app.log]                   │
  │  [App writes to /data/uploads/file.jpg]             │
  │       │                                             │
  │  docker stop myapp && docker rm myapp               │
  │       │                                             │
  │       ▼                                             │
  │  Container's writable layer is DELETED              │
  │  app.log → GONE                                     │
  │  file.jpg → GONE                                    │
  │                                                     │
  └─────────────────────────────────────────────────────┘

  This is a feature, not a bug — for stateless apps.
  But databases, uploads, logs, and config need persistence.
  That's what Docker storage primitives solve.
```

### The Three Storage Types at a Glance

```
  ┌───────────────────────────────────────────────────────────┐
  │                  Docker Storage Types                     │
  ├──────────────────┬────────────────────┬───────────────────┤
  │   NAMED VOLUME   │    BIND MOUNT      │      tmpfs        │
  ├──────────────────┼────────────────────┼───────────────────┤
  │ Managed by       │ You control the    │ Lives in host     │
  │ Docker           │ host path          │ RAM only          │
  │                  │                    │                   │
  │ /var/lib/docker/ │ Any path on host   │ Never touches     │
  │ volumes/mydata   │ /home/user/data    │ disk              │
  │                  │                    │                   │
  │ Best for:        │ Best for:          │ Best for:         │
  │ Databases,       │ Dev config, code   │ Secrets, caches,  │
  │ prod persistence │ hot-reload in dev  │ sensitive data    │
  └──────────────────┴────────────────────┴───────────────────┘

  WHERE DATA LIVES:
  Named volume  → /var/lib/docker/volumes/<name>/_data   (on host)
  Bind mount    → exactly the host path you specify
  tmpfs         → host RAM (kernel memory, never written to disk)
  Container WL  → /var/lib/docker/overlay2/<id>/diff    (ephemeral)
```

---

## 5.2 Named Volumes — Docker-Managed Persistence

Named volumes are the **recommended** way to persist data in production.
Docker owns the storage location — you reference it by name.

### Basic Usage

```bash
  # Create a volume explicitly
  docker volume create pgdata

  # Or let Docker create it on first use
  docker run -v pgdata:/var/lib/postgresql/data postgres

  # New --mount syntax (more verbose, more explicit — preferred in prod)
  docker run \
    --mount type=volume,source=pgdata,target=/var/lib/postgresql/data \
    postgres
```

### Volume Lifecycle

```
  CREATE:   docker volume create pgdata
  USE:      docker run -v pgdata:/var/lib/postgresql/data postgres
  INSPECT:  docker volume inspect pgdata
  LIST:     docker volume ls
  REMOVE:   docker volume rm pgdata      ← DATA IS PERMANENTLY DELETED
  PRUNE:    docker volume prune          ← removes ALL unused volumes ⚠️

  INSPECT OUTPUT (example):
  {
    "Name": "pgdata",
    "Driver": "local",
    "Mountpoint": "/var/lib/docker/volumes/pgdata/_data",
    "Labels": {},
    "Scope": "local"
  }

  That Mountpoint is where data physically lives on the host.
  You can access it directly as root on the host if needed.
```

### Volume Sharing Between Containers

```
  Multiple containers can mount the same volume simultaneously.

  Use case: Nginx serves static files built by your app container.

  docker run -v assets:/app/dist --name builder node-build-container
  docker run -v assets:/usr/share/nginx/html --name nginx nginx

  ┌────────────────────────────────────────────────────────┐
  │              Named Volume: "assets"                    │
  │         /var/lib/docker/volumes/assets/_data           │
  │                                                        │
  │  ┌──────────────┐            ┌──────────────┐          │
  │  │   builder    │            │    nginx     │          │
  │  │  /app/dist ──┼────────────┼── /usr/share │          │
  │  │  (writes)    │   shared   │   (reads)    │          │
  │  └──────────────┘   volume   └──────────────┘          │
  └────────────────────────────────────────────────────────┘

  ⚠️ Concurrent WRITES from multiple containers to the same
  volume without coordination = data corruption risk.
  Only do shared mounts if you control the write pattern.
```

### Volume Drivers — Beyond Local Storage

```
  Default driver is "local" (data on same host as container).
  Third-party volume drivers extend this to remote/cloud storage.

  DRIVER          STORAGE BACKEND              USE CASE
  ─────────────────────────────────────────────────────────────
  local           Host filesystem              Default, single-node
  nfs             NFS network share            Shared across hosts
  rexray/ebs      AWS EBS                      AWS persistent disk
  rexray/s3fs     AWS S3 (via FUSE)            Object storage
  azure-file      Azure File Share             Azure multi-attach
  cloudstor       Amazon EFS                   Elastic, multi-host
  portworx        Portworx cluster             K8s, HA storage

  EXAMPLE — NFS volume:
  docker volume create \
    --driver local \
    --opt type=nfs \
    --opt o=addr=10.0.0.5,rw \
    --opt device=:/exports/data \
    nfs-data

  docker run -v nfs-data:/data myapp
```

---

## 5.3 Bind Mounts — Host Path Control

A bind mount maps a **specific host path** directly into the container.
You control exactly what's mounted and from where.

```bash
  # Short syntax
  docker run -v /host/path:/container/path myapp

  # Long --mount syntax (explicit, recommended)
  docker run \
    --mount type=bind,source=/host/path,target=/container/path \
    myapp

  # Read-only bind mount (container can't modify it)
  docker run \
    --mount type=bind,source=/etc/app-config,target=/config,readonly \
    myapp
```

### Bind Mount Mental Model

```
  HOST FILESYSTEM             CONTAINER FILESYSTEM
  ───────────────             ────────────────────
  /home/alice/
  └── myapp/
      ├── src/          ────────────────►  /app/src/
      │   └── app.py    (bind mount)       └── app.py
      ├── config/       ────────────────►  /config/
      │   └── dev.env   (bind mount)       └── dev.env
      └── .git/                            (NOT mounted)

  Container writes to /app/src/app.py
        ↕ same inode
  Host file /home/alice/myapp/src/app.py is modified instantly

  This is two-way: host changes appear in container immediately.
  Perfect for development hot-reload workflows.
```

### Dev Hot-Reload Pattern

```yaml
  # docker-compose.yml for development
  services:
    api:
      image: node:18-alpine
      working_dir: /app
      command: npm run dev          # nodemon watches for changes
      volumes:
        - type: bind
          source: ./src             # host source code
          target: /app/src          # mounted into container
        - type: bind
          source: ./package.json
          target: /app/package.json
        - node_modules:/app/node_modules  # named volume for deps
          # ↑ keeps node_modules in container, not overwritten by bind

  volumes:
    node_modules:

  # Why node_modules as named volume?
  # Without it: bind mount overwrites /app/node_modules with
  # host's node_modules (different OS, different binaries).
  # The named volume keeps container-compiled deps separate.
```

---

## 5.4 tmpfs Mounts — RAM-Only Storage

tmpfs mounts store data **only in host RAM**. Data is never written to disk
and is destroyed when the container stops.

```bash
  docker run \
    --mount type=tmpfs,target=/run/secrets,tmpfs-size=64m \
    myapp

  # Or short form (size not supported in -v syntax for tmpfs)
  docker run --tmpfs /tmp:rw,size=128m myapp
```

### When to Use tmpfs

```
  USE CASE                    WHY tmpfs WINS
  ─────────────────────────────────────────────────────────
  Secrets / credentials       Never written to disk.
                              No trace left after container stops.
                              No risk of disk forensics exposure.

  Session data                Fast RAM I/O. Ephemeral by design.

  Build caches                Blazing fast. Discarded after build.

  Test data                   No cleanup needed. Gone on stop.

  High-speed scratch space    I/O intensive operations without
                              wearing out SSD or creating I/O wait.

  tmpfs SIZE TIP:
  --tmpfs-size is in bytes by default.
  --tmpfs /tmp:rw,size=128m   → 128 megabytes
  Don't forget to size it — default is unlimited (eats all RAM!)
```

---

## 5.5 Storage Architecture Comparison

```
  ┌──────────────────────────────────────────────────────────────┐
  │                 STORAGE DECISION FLOWCHART                   │
  │                                                              │
  │  Does the data need to SURVIVE container restarts?           │
  │           │                                                  │
  │     NO ◄──┴──► YES                                          │
  │      │               │                                       │
  │      ▼               ▼                                       │
  │   tmpfs         Do you need to access the data               │
  │  (scratch,      directly from the HOST machine?              │
  │   secrets)           │                                       │
  │              NO ◄────┴────► YES                              │
  │               │                    │                         │
  │               ▼                    ▼                         │
  │         Named Volume          Bind Mount                     │
  │         (Docker manages)      (you manage host path)         │
  │               │                    │                         │
  │               ▼                    ▼                         │
  │         Production DB,        Dev hot-reload,                │
  │         Uploads, State        Config injection,              │
  │                               CI workspace mounts            │
  └──────────────────────────────────────────────────────────────┘
```

### Performance Characteristics

```
  STORAGE TYPE     READ         WRITE       DURABILITY    PORTABILITY
  ─────────────────────────────────────────────────────────────────
  tmpfs            ★★★★★        ★★★★★       ✗ None        ✗ None
  Named Volume     ★★★★         ★★★★        ✅ Yes         ★★★ (local)
  Bind Mount       ★★★★         ★★★★        ✅ Yes         ★★ (path dep)
  Container WL     ★★★          ★★★         ✗ None        ✗ None
  NFS Volume       ★★           ★★          ✅ Yes         ★★★★★
  EBS Volume       ★★★★         ★★★★        ✅ Yes         ★★★★ (AZ-bound)
  S3 (FUSE)        ★            ★           ✅ Yes         ★★★★★

  Container writable layer uses Copy-on-Write (OverlayFS) —
  every write goes through CoW overhead → slowest for I/O intensive apps.
  Always use a volume or bind mount for databases and write-heavy workloads.
```

---

## 5.6 Stateful Services in Production — Database Patterns

### Postgres + Named Volume (Standard Pattern)

```yaml
  # docker-compose.yml — production-ready postgres
  services:
    db:
      image: postgres:16-alpine
      restart: unless-stopped
      environment:
        POSTGRES_USER: appuser
        POSTGRES_PASSWORD_FILE: /run/secrets/pg_password  # secret file
        POSTGRES_DB: myapp
      volumes:
        - pgdata:/var/lib/postgresql/data     # persist DB files
        - ./init-scripts:/docker-entrypoint-initdb.d  # init SQL
      healthcheck:
        test: ["CMD-SHELL", "pg_isready -U appuser -d myapp"]
        interval: 10s
        timeout: 5s
        retries: 5

  volumes:
    pgdata:
      driver: local
      # In production on cloud: use EBS / GCP PD / Azure Disk
      # and mount it to host before running container
```

### The Volume Backup Pattern

```bash
  # BACKUP: Run a temporary container to tar the volume
  docker run --rm \
    -v pgdata:/data \                        # mount the volume
    -v $(pwd)/backups:/backup \              # bind mount for output
    alpine \
    tar czf /backup/pgdata-$(date +%Y%m%d).tar.gz -C /data .

  # RESTORE: Reverse the process
  docker run --rm \
    -v pgdata:/data \
    -v $(pwd)/backups:/backup \
    alpine \
    tar xzf /backup/pgdata-20241201.tar.gz -C /data

  # COPY VOLUME to another host:
  # 1. Backup to file (above)
  # 2. scp backup file to new host
  # 3. Create new volume on new host
  # 4. Restore (above)

  # INSPECT what's inside a volume without starting the app:
  docker run --rm -it \
    -v pgdata:/inspect:ro \
    alpine ls -la /inspect
```

---

## 5.7 Gotchas — Storage Edition

### ⚠️ Gotcha #12: `docker volume prune` deletes ALL unused volumes silently

```
  docker volume prune

  This removes every volume NOT currently mounted to a running container.
  If your DB container is stopped (not running), its volume is "unused"
  and WILL be deleted by prune.

  I'VE SEEN THIS DELETE PRODUCTION DATABASES.

  Always confirm with:
  $ docker volume ls               # list all volumes
  $ docker ps -a                   # check stopped containers

  SAFER PRUNE — only remove anonymous volumes:
  $ docker volume prune --filter label!=keep=true

  LABEL VOLUMES YOU CARE ABOUT:
  docker volume create --label keep=true pgdata

  MAKE PRUNE SAFER BY HABIT:
  Always run docker system df first to see what would be affected.
  Never run docker system prune -a --volumes in production.
```

### ⚠️ Gotcha #13: Bind mount on Mac/Windows is slow for large codebases

```
  On Linux: bind mount = zero overhead (direct kernel path)
  On Mac:   bind mount = goes through gRPC file sync layer
            between macOS and the Linux VM inside Docker Desktop

  SYMPTOM: npm install / pip install / gradle build inside
           a bind-mounted container is 5-10x slower than on host.

  BENCHMARKS (rough, Docker Desktop Mac):
  Linux bind mount:    npm install  → ~15 seconds
  Mac bind mount:      npm install  → ~90 seconds
  Mac named volume:    npm install  → ~17 seconds (stays in VM)

  FIXES:
  1. Use named volumes for dependency dirs (node_modules, .venv)
     Bind mount only source code (smaller change surface).

  2. Enable VirtioFS in Docker Desktop settings
     (replaces gRPC FUSE, much faster, macOS 12.5+)

  3. Use :cached or :delegated consistency hints (legacy, less needed
     with VirtioFS but still valid for older setups)

  4. For heavy builds: use devcontainers or Colima instead
     of Docker Desktop.
```

### ⚠️ Gotcha #14: Running a database in a container in production without understanding persistence

```
  This is the #1 data loss scenario:

  SCENARIO:
  Team containerizes MySQL, runs it with docker run.
  No -v flag. Data lives in container writable layer.
  Deployment pipeline runs docker rm + docker run to redeploy.
  Database is wiped clean on every deploy.

  They notice when they go to production.

  PREVENTION CHECKLIST for stateful containers:
  □ Named volume or bind mount attached (docker inspect → Mounts)
  □ Volume is backed up (automated, tested restore)
  □ Container has restart: unless-stopped or always
  □ Healthcheck defined so orchestrator knows it's ready
  □ Data directory explicitly documented in runbook
  □ Volume backup tested: can you actually restore?

  docker inspect mydb | grep -A 20 '"Mounts"'
  If Mounts is empty [] → YOUR DATA IS EPHEMERAL. Fix immediately.
```

### ⚠️ Gotcha #15: File permissions mismatch between host and container

```
  SYMPTOM: Container crashes with "Permission denied" on startup,
           or can't write to a bind-mounted directory.

  CAUSE: Host directory owned by user:group 1000:1000 (your user).
         Container runs as root (UID 0) or a specific UID (e.g., 999).
         UID 999 inside container ≠ your user outside.

  EXAMPLE with Postgres:
  Host: mkdir ./pgdata  (owned by alice:alice, UID 1000)
  Container: postgres process runs as UID 999
  → postgres can't write to ./pgdata → crash

  DIAGNOSE:
  $ ls -la ./pgdata        # check host ownership
  $ docker run --rm postgres id   # check container user

  FIX OPTIONS:
  1. Match UIDs — create the user on host with same UID as container
     sudo useradd -u 999 postgres-user
     sudo chown -R 999:999 ./pgdata

  2. Use named volumes instead of bind mounts
     Docker handles ownership automatically

  3. Set ownership in Dockerfile
     RUN chown -R myuser:myuser /app/data

  4. Use --user flag to run container as your UID
     docker run --user $(id -u):$(id -g) myapp
     (only works if app doesn't need root)
```

---

## 5.8 Volume Management — Operations Cheatsheet

```
  OPERATION                    COMMAND
  ─────────────────────────────────────────────────────────────
  Create named volume          docker volume create mydata

  List all volumes             docker volume ls

  Inspect volume (path etc.)   docker volume inspect mydata

  Remove single volume         docker volume rm mydata

  Remove unused volumes        docker volume prune
  (CAREFUL — see gotcha #12)

  See volume disk usage        docker system df -v

  Mount volume read-only       -v mydata:/data:ro
                               --mount type=volume,src=mydata,
                                 dst=/data,readonly

  Mount bind read-only         -v /host/path:/data:ro

  Find which containers        docker ps -a --filter volume=mydata
  use a volume

  Copy file INTO a volume      docker run --rm \
                                 -v mydata:/data \
                                 -v $(pwd):/src \
                                 alpine cp /src/file.txt /data/

  List files in a volume       docker run --rm \
                                 -v mydata:/data \
                                 alpine ls -la /data

  Watch writes to a volume     docker run --rm \
  (debug)                        -v mydata:/data \
                                 alpine watch -n1 ls -la /data
```

---

## 5.9 Production FAQ — Batch 5

---

**Q: Should I run my production database in Docker?**

```
A: Controversial topic — here's the honest answer:

  YOU CAN, but it requires discipline and extra operational care.

  CHALLENGES with DB in Docker:
  ─────────────────────────────────────────────────────────────
  1. Storage: must use volumes, must back them up, must test restores
  2. Performance: container networking adds latency (use host network
     or collocate app + DB on same host)
  3. Upgrades: major version upgrades (Postgres 14 → 16) require
     careful data migration, not just changing image tag
  4. HA: Docker alone has no built-in HA for stateful services.
     You need Swarm/K8s + operators (postgres-operator, etc.)

  WHEN IT'S FINE:
  - Dev and staging environments (fast reset, isolation)
  - Small internal tools / low-stakes apps
  - Self-hosted apps where you own and accept the risk
  - Teams with mature backup + restore workflows

  WHEN TO USE MANAGED DB (RDS, Cloud SQL, PlanetScale):
  - Customer-facing production data
  - When uptime SLAs matter
  - When your team lacks DB ops expertise
  - Regulated industries (easier compliance with managed services)

  THE PRAGMATIC RULE:
  If you can't answer "how long does a restore take and when
  did you last test it?" → don't run it in Docker in prod.
```

---

**Q: My Postgres data is gone after a server reboot. The container still exists. What happened?**

```
A: Classic anonymous volume problem. Here's the diagnosis:

  STEP 1: Check what mounts the container has
  $ docker inspect mydb | grep -A 20 '"Mounts"'

  If you see:
  "Type": "volume",
  "Name": "a3f9b2c1...",    ← random hash = ANONYMOUS volume
  "Source": "/var/lib/docker/volumes/a3f9b2c1.../_data"

  PROBLEM: An anonymous volume was created (no name given).
  When you removed and recreated the container, a NEW anonymous
  volume was created. Old data is in the old anonymous volume,
  still on disk but not attached.

  RECOVERY:
  $ docker volume ls    # find the old volume with your data
  $ docker run --rm -v <old-volume-id>:/recover alpine ls /recover
  # If data is there, copy it to a named volume

  PREVENTION GOING FORWARD:
  Always name your volumes explicitly:
  -v pgdata:/var/lib/postgresql/data    ← named = survives rm
  Not:
  -v /var/lib/postgresql/data          ← anonymous = dangerous
```

---

**Q: We need to share a volume between containers on different hosts. What are our options?**

```
A: You need distributed storage. Options by complexity:

  OPTION 1: NFS Mount (simple, proven)
  Set up an NFS server. Mount the NFS share as a Docker volume
  using the local driver with NFS options (see 5.2 volume drivers).
  Works everywhere. Slow for write-heavy workloads.

  OPTION 2: Cloud-native block storage
  AWS: EBS volumes can be attached to one EC2 instance at a time.
       Use EFS (Elastic File System) for multi-instance shared access.
  GCP: Persistent Disks (ReadWriteOnce) or Filestore (NFS-based).
  Azure: Azure Disks (single) or Azure Files (shared).

  OPTION 3: Distributed storage (Ceph, GlusterFS)
  Self-hosted, highly available distributed filesystem.
  Complex to operate. Used in on-prem K8s clusters.

  OPTION 4: Object storage with sync (S3-style)
  Use rclone or s3fs to mount an S3 bucket as a volume.
  Only for read-heavy, latency-tolerant workloads.

  OPTION 5: Use K8s PersistentVolumes with StorageClass
  The K8s-native way. StorageClass provisions storage dynamically.
  Use ReadWriteMany StorageClass (NFS, EFS, CephFS) for shared access.

  THE REAL ANSWER FOR MOST TEAMS:
  Design your apps to be stateless. Put state in a managed database
  or object store (S3). Let the cloud handle persistence.
  Shared volumes between containers = operational complexity.
```

---

**Q: How do I back up Docker volumes automatically in production?**

```
A: Several proven patterns:

  PATTERN 1: Cron job with docker run backup container
  ─────────────────────────────────────────────────────
  # /etc/cron.d/docker-backup
  0 2 * * * root docker run --rm \
    -v pgdata:/data:ro \
    -v /backups:/backup \
    -e S3_BUCKET=my-backup-bucket \
    amazon/aws-cli s3 cp /data s3://$S3_BUCKET/pgdata/$(date +%Y%m%d)/ \
    --recursive

  PATTERN 2: Pre-stop hook / sidecar backup container
  Run a sidecar container that continuously ships data off-host.
  (Velero for K8s, custom scripts for Docker Swarm)

  PATTERN 3: DB-native backups (preferred for databases)
  For Postgres: pg_dump > /backup/dump.sql  (logical backup)
  Safer than raw file backup while DB is running.
  docker exec mydb pg_dump -U postgres mydb > backup.sql

  PATTERN 4: Snapshot host disk/volume
  AWS: EBS snapshot (point-in-time, crash-consistent)
  GCP: Disk snapshot
  Fastest recovery but requires cloud provider.

  GOLDEN RULE OF BACKUPS:
  A backup you haven't tested restoring is not a backup.
  Run a restore drill monthly. Measure RTO (recovery time).
  Automate the restore test: spin up a new container with
  the backup volume and run your DB healthcheck against it.
```

---

*End of Batch 5.*

---

# BATCH 6
## Dockerfile Best Practices + Multi-stage Builds

> **Goal:** Go from writing Dockerfiles that "work" to writing ones that are fast to build,
> small in size, secure by default, and maintainable in a team. This batch covers every
> Dockerfile instruction in depth, the layer-caching mental model, and multi-stage builds —
> the single most impactful technique for production image quality.

---

## 6.1 Dockerfile Instruction Reference — The Complete Map

Every instruction, what it does, and when to use it:

```
  INSTRUCTION    PURPOSE & NOTES
  ───────────────────────────────────────────────────────────────────
  FROM           Base image. Every Dockerfile starts here.
                 FROM scratch = truly empty (for Go static binaries)
                 FROM alpine  = minimal Linux (~5MB)
                 FROM ubuntu  = familiar but large (~70MB)

  RUN            Execute commands during BUILD time.
                 Creates a new layer. Chain with && to avoid bloat.

  CMD            Default command when container starts.
                 Can be overridden at runtime: docker run img mycommand
                 Only the LAST CMD takes effect.

  ENTRYPOINT     Fixed executable. CMD becomes its arguments.
                 Cannot be overridden without --entrypoint flag.
                 Use for containers that ARE a tool (e.g., curl image)

  COPY           Copy files from build context into image.
                 Preferred over ADD for simple file copies.

  ADD            Like COPY but also: extracts .tar.gz, fetches URLs.
                 Avoid unless you need tar extraction.
                 URL fetching is unpredictable and not cached well.

  ENV            Set environment variables baked into the image.
                 Available at build time AND runtime.
                 Visible in docker inspect — don't put secrets here.

  ARG            Build-time variable. NOT available at runtime.
                 Pass with: docker build --build-arg KEY=value
                 Use for version pins, build-time config.

  EXPOSE         Document which port the container listens on.
                 Does NOT actually publish the port (-p does that).
                 Informational. Used by docker-compose and tooling.

  WORKDIR        Set working directory for subsequent instructions.
                 Creates dir if it doesn't exist.
                 Prefer over RUN cd /app — it's explicit and persists.

  USER           Switch to non-root user for subsequent instructions.
                 Critical for security. Always set before CMD/ENTRYPOINT.

  VOLUME         Declare a mount point. Docker creates anonymous volume
                 if none is provided at runtime.
                 Gotcha: VOLUME in Dockerfile bypasses build-time writes
                 to that path. Rarely needed — prefer runtime -v flags.

  LABEL          Attach metadata as key=value pairs.
                 Use for: version, maintainer, git-sha, build date.
                 Searchable: docker images --filter label=version=1.2

  HEALTHCHECK    Define how Docker tests if the container is healthy.
                 Orchestrators (Swarm, K8s) use this to route traffic.

  ONBUILD        Trigger instruction when image is used as a base.
                 Rarely used. Useful for base images shared across team.

  SHELL          Change the default shell for RUN commands.
                 Default: /bin/sh -c on Linux, cmd /S /C on Windows.

  STOPSIGNAL     Signal sent to stop the container (default SIGTERM).
                 Override if your app uses a different shutdown signal.
```

---

## 6.2 CMD vs ENTRYPOINT — The Most Confused Pair

This trips up almost every Docker beginner. Here's the full mental model:

```
  ENTRYPOINT = "what this container IS"
  CMD        = "default arguments to ENTRYPOINT"

  ┌─────────────────────────────────────────────────────────┐
  │  ENTRYPOINT ["nginx"]   CMD ["-g", "daemon off;"]       │
  │                                                         │
  │  docker run myimage                                     │
  │  → runs: nginx -g "daemon off;"                         │
  │                                                         │
  │  docker run myimage -t                                  │
  │  → runs: nginx -t      (CMD replaced, ENTRYPOINT kept)  │
  └─────────────────────────────────────────────────────────┘

  FORMS (shell vs exec):
  ─────────────────────────────────────────────────────────
  Shell form:   CMD nginx -g "daemon off;"
                → runs as: /bin/sh -c "nginx -g 'daemon off;'"
                → signals (SIGTERM) go to sh, NOT nginx!
                → nginx won't receive graceful shutdown signal
                → USE EXEC FORM INSTEAD

  Exec form:    CMD ["nginx", "-g", "daemon off;"]
                → runs nginx directly as PID 1
                → signals go directly to nginx ✅
                → ALWAYS use exec form for CMD and ENTRYPOINT

  COMBINATION MATRIX:
  ┌──────────────────┬───────────────┬────────────────────────┐
  │  No ENTRYPOINT   │ CMD ["nginx"] │ runs: nginx            │
  │  No ENTRYPOINT   │ CMD nginx     │ runs: /bin/sh -c nginx │
  │  EP ["nginx"]    │ No CMD        │ runs: nginx            │
  │  EP ["nginx"]    │ CMD ["-t"]    │ runs: nginx -t         │
  │  EP ["nginx"]    │ CMD nginx     │ runs: nginx /bin/sh... │← WRONG
  └──────────────────┴───────────────┴────────────────────────┘

  RULE: If you use ENTRYPOINT, use exec form for both.
        Never mix exec ENTRYPOINT with shell CMD.
```

---

## 6.3 Layer Caching — Engineering Fast Builds

Layer caching is the biggest lever you have on build speed.
Every `RUN`, `COPY`, and `ADD` creates a layer. Docker caches each layer
by its content hash. A cache miss at layer N invalidates ALL layers after N.

```
  CACHE INVALIDATION TRIGGERS:
  ─────────────────────────────────────────────────────────
  RUN apt-get update    → invalidated if the instruction text changes
  COPY ./src /app       → invalidated if ANY file in ./src changes
  ARG VERSION=1.0       → invalidated if --build-arg VERSION changes
  ENV NODE_ENV=prod     → invalidated if value changes

  THE GOLDEN RULE: Put things that change LEAST at the top.
                   Put things that change MOST at the bottom.

  CHANGE FREQUENCY (most stable → most volatile):
  ──────────────────────────────────────────────
  1. Base image (FROM)                   ← changes: rarely
  2. System dependencies (apt/apk)       ← changes: occasionally
  3. App dependencies manifest           ← changes: when adding deps
     (package.json, requirements.txt)
  4. App dependencies install            ← changes: when manifest changes
     (npm install, pip install)
  5. Application source code             ← changes: every commit
  6. Build artifacts / compiled output   ← changes: every commit
```

### Cache-Optimised Dockerfiles by Language

```dockerfile
  ── NODE.JS ─────────────────────────────────────────────
  FROM node:20-alpine
  WORKDIR /app

  # Layer 1: copy ONLY package files (rarely changes)
  COPY package.json package-lock.json ./

  # Layer 2: install deps (cached unless package files change)
  RUN npm ci --only=production

  # Layer 3: copy source (changes every commit, but layer 2 stays cached)
  COPY src/ ./src/

  CMD ["node", "src/index.js"]

  ── PYTHON ──────────────────────────────────────────────
  FROM python:3.12-slim
  WORKDIR /app

  COPY requirements.txt .
  RUN pip install --no-cache-dir -r requirements.txt

  COPY . .
  CMD ["python", "-m", "uvicorn", "app.main:app", "--host", "0.0.0.0"]

  ── GO ──────────────────────────────────────────────────
  FROM golang:1.22-alpine
  WORKDIR /app

  COPY go.mod go.sum ./
  RUN go mod download          # cache module downloads separately

  COPY . .
  RUN go build -o /app/server .

  CMD ["/app/server"]

  ── JAVA (Maven) ────────────────────────────────────────
  FROM maven:3.9-eclipse-temurin-21 AS build
  WORKDIR /app

  COPY pom.xml .
  RUN mvn dependency:go-offline -B   # download all deps first

  COPY src/ ./src/
  RUN mvn package -DskipTests

  FROM eclipse-temurin:21-jre-alpine
  COPY --from=build /app/target/app.jar /app.jar
  CMD ["java", "-jar", "/app.jar"]
```

---

## 6.4 Multi-Stage Builds — The Most Important Advanced Technique

Multi-stage builds let you use **multiple FROM instructions** in one Dockerfile.
Each stage can copy artifacts from previous stages. Final image only contains
what you explicitly copy in — build tools are left behind.

### The Problem Multi-Stage Solves

```
  WITHOUT multi-stage (naive approach):

  FROM node:20                        ← 1.1 GB base image
  RUN npm install                     ← dev + prod deps
  RUN npm run build                   ← TypeScript compiler etc.
  CMD ["node", "dist/index.js"]

  FINAL IMAGE SIZE: ~1.4 GB
  Contains: Node.js runtime, npm, TypeScript compiler,
            all dev deps, source maps, test files...
  Goes to production: ALL of it. Attack surface = massive.

  WITH multi-stage:

  FROM node:20 AS builder             ← Stage 1: build environment
  WORKDIR /app
  COPY package*.json ./
  RUN npm ci                          ← install ALL deps (dev too)
  COPY . .
  RUN npm run build                   ← compile TypeScript → JS

  FROM node:20-alpine AS runtime      ← Stage 2: lean runtime image
  WORKDIR /app
  COPY --from=builder /app/dist ./dist
  COPY --from=builder /app/node_modules ./node_modules
  # (or run npm ci --only=production here)
  USER node                           ← non-root
  CMD ["node", "dist/index.js"]

  FINAL IMAGE SIZE: ~180 MB
  Contains: Node.js runtime, compiled JS, prod deps only.
  TypeScript compiler, source maps, dev deps: GONE.
```

### Multi-Stage Architecture Patterns

```
  PATTERN 1 — Builder + Runtime (most common)
  ────────────────────────────────────────────
  FROM golang:1.22 AS builder
  WORKDIR /src
  COPY . .
  RUN CGO_ENABLED=0 go build -o /app/server .

  FROM scratch                  ← truly empty base image!
  COPY --from=builder /app/server /server
  COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
  ENTRYPOINT ["/server"]

  RESULT: Image is ONLY the compiled binary + TLS certs.
  Size: ~10 MB. No shell, no package manager, no OS.
  Attacker can't exec into it — nothing to run!


  PATTERN 2 — Test stage in pipeline
  ────────────────────────────────────
  FROM python:3.12 AS base
  WORKDIR /app
  COPY requirements*.txt ./
  RUN pip install -r requirements.txt

  FROM base AS test
  RUN pip install -r requirements-dev.txt
  COPY . .
  RUN pytest tests/ -v           ← tests run during build!

  FROM base AS runtime
  COPY --from=base /usr/local/lib /usr/local/lib
  COPY src/ ./src/
  CMD ["python", "-m", "uvicorn", "src.main:app"]

  # In CI:
  docker build --target test .        ← run tests, fail fast
  docker build --target runtime .     ← build prod image


  PATTERN 3 — Shared base with multiple outputs
  ──────────────────────────────────────────────
  FROM node:20 AS base
  WORKDIR /app
  COPY package*.json ./
  RUN npm ci

  FROM base AS build-web
  COPY src/web ./src/web
  RUN npm run build:web

  FROM base AS build-api
  COPY src/api ./src/api
  RUN npm run build:api

  FROM nginx AS web
  COPY --from=build-web /app/dist/web /usr/share/nginx/html

  FROM node:20-alpine AS api
  COPY --from=build-api /app/dist/api ./
  CMD ["node", "index.js"]

  # Build selectively:
  docker build --target web -t myapp-web .
  docker build --target api -t myapp-api .
```

### Copying Between Stages

```dockerfile
  # Copy from a named stage
  COPY --from=builder /app/dist ./dist

  # Copy from a specific stage index (0-based)
  COPY --from=0 /output ./

  # Copy from an EXTERNAL image (not a stage in this file!)
  COPY --from=nginx:1.25 /etc/nginx/nginx.conf /etc/nginx/nginx.conf
  COPY --from=alpine:3.19 /etc/passwd /etc/passwd

  # Copy with ownership set (BuildKit only)
  COPY --from=builder --chown=node:node /app/dist ./dist
```

---

## 6.5 Base Image Selection — Security and Size Tradeoffs

```
  BASE IMAGE HIERARCHY (large → small):
  ──────────────────────────────────────────────────────────────
  ubuntu:22.04        ~77 MB    Full Ubuntu. Familiar tools.
                                Many CVEs. Broad attack surface.

  debian:12-slim      ~74 MB    Debian without docs/man pages.
                                Good balance of size vs familiarity.

  alpine:3.19         ~7 MB     Musl libc, BusyBox. Very small.
                                Some C-extension issues (musl vs glibc).
                                Great for Go, simple Python, static bins.

  distroless          ~2-20 MB  Google-maintained. No shell, no package
  (gcr.io/distroless) manager. Contains only app + runtime deps.
                                Strong security. Hard to debug.

  scratch             0 bytes   Truly empty. For fully static binaries.
                                Go, Rust, C with -static flag.

  SELECTING BY LANGUAGE:
  ─────────────────────────────────────────────────────
  Language   Recommended Base           Notes
  ─────────────────────────────────────────────────────
  Node.js    node:20-alpine             Small, works well
  Python     python:3.12-slim           slim over full
  Go         scratch / distroless/base  Static binary → tiny image
  Java       eclipse-temurin:21-jre-    JRE not JDK for runtime
             alpine
  Ruby       ruby:3.3-alpine
  Nginx      nginx:1.25-alpine
  General    alpine:3.19                For custom setups

  ALWAYS PIN A SPECIFIC VERSION TAG:
  ✗ FROM node:latest          → build differs over time
  ✗ FROM node:20              → minor updates can break things
  ✅ FROM node:20.11.0-alpine3.19  → 100% reproducible builds

  CHECK CURRENT DIGEST FOR MAXIMUM PINNING:
  $ docker pull node:20-alpine
  $ docker inspect node:20-alpine | grep RepoDigests
  → node@sha256:abc123...
  FROM node@sha256:abc123...   ← immutable, no tag drift ever
```

---

## 6.6 Security Hardening in Dockerfiles

### Run as Non-Root — The Most Important Security Rule

```dockerfile
  # BAD — runs as root (Docker default)
  FROM node:20-alpine
  WORKDIR /app
  COPY . .
  RUN npm ci
  CMD ["node", "index.js"]

  # GOOD — explicit non-root user
  FROM node:20-alpine
  WORKDIR /app

  # node:alpine already has a 'node' user (UID 1000)
  # Copy and install as root (need write access)
  COPY --chown=node:node package*.json ./
  RUN npm ci --only=production

  COPY --chown=node:node . .

  # Switch to non-root before CMD
  USER node
  CMD ["node", "index.js"]

  # VERIFY:
  $ docker run myimage whoami   → should print "node", not "root"
```

### Creating a Minimal Custom User

```dockerfile
  FROM alpine:3.19

  # Create app group and user with no home dir, no shell
  RUN addgroup -S appgroup && \
      adduser -S -G appgroup -H -s /sbin/nologin appuser

  WORKDIR /app
  COPY --chown=appuser:appgroup . .

  USER appuser
  CMD ["./myapp"]
```

### Read-Only Filesystem Pattern

```dockerfile
  FROM node:20-alpine
  WORKDIR /app
  COPY --chown=node:node . .
  RUN npm ci --only=production
  USER node

  # Mark directories the app needs to write to
  VOLUME ["/app/tmp", "/app/logs"]

  CMD ["node", "index.js"]

  # Run with read-only root filesystem:
  docker run --read-only \
    --tmpfs /app/tmp:rw,noexec,nosuid \
    --tmpfs /tmp:rw,noexec,nosuid \
    myapp

  # Any write attempt outside /app/tmp or /tmp → permission denied
  # Prevents malware from writing executables to filesystem
```

### Drop Linux Capabilities

```dockerfile
  # In docker-compose.yml or docker run:
  # By default containers get ~14 capabilities.
  # Most apps need NONE of them.

  docker run \
    --cap-drop ALL \               # drop every capability
    --cap-add NET_BIND_SERVICE \   # add back only what's needed
    myapp                          # (binding port < 1024)

  # In docker-compose.yml:
  services:
    api:
      cap_drop:
        - ALL
      cap_add:
        - NET_BIND_SERVICE

  # COMMON CAPABILITIES AND WHEN YOU NEED THEM:
  # NET_BIND_SERVICE → bind ports < 1024 (use port > 1024 instead!)
  # CHOWN            → change file ownership (avoid in prod)
  # SETUID           → change process UID (avoid)
  # SYS_PTRACE       → debuggers (never in prod)
  # NET_ADMIN        → network config (only for network tools)
```

---

## 6.7 ARG and ENV — Build-Time vs Runtime Variables

```
  ARG — build-time only                ENV — baked into image

  Available: during docker build        Available: build + runtime
  Visible in: docker history            Visible in: docker inspect,
  NOT in:    running container              docker history, process env
  Use for:   version pins, build        Use for:   app config, feature
             flags, CI metadata             flags, defaults

  ┌──────────────────────────────────────────────────────┐
  │  ARG NODE_VERSION=20.11.0                            │
  │  FROM node:${NODE_VERSION}-alpine                    │
  │                    ↑ ARG can be used in FROM         │
  │                                                      │
  │  ARG BUILD_DATE                                      │
  │  ARG GIT_SHA                                         │
  │                                                      │
  │  LABEL build-date="${BUILD_DATE}"                    │
  │  LABEL git-sha="${GIT_SHA}"                          │
  │                                                      │
  │  ENV NODE_ENV=production                             │
  │  ENV PORT=3000                                       │
  │  ENV LOG_LEVEL=info                                  │
  └──────────────────────────────────────────────────────┘

  BUILD:
  docker build \
    --build-arg BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
    --build-arg GIT_SHA=$(git rev-parse --short HEAD) \
    -t myapp:$(git rev-parse --short HEAD) .

  ⚠️ ARG SECURITY GOTCHA:
  ARG values appear in docker history in plain text.
  NEVER pass secrets as ARG values.
  $ docker history myimage --no-trunc
  → shows every RUN command + ARG values

  ✅ For secrets at build time:
  Use BuildKit secrets: --secret id=mysecret,src=./secret.txt
  Access in Dockerfile: RUN --mount=type=secret,id=mysecret \
                            cat /run/secrets/mysecret
  Secret does NOT appear in image layers or history.
```

---

## 6.8 HEALTHCHECK — Container Self-Awareness

```dockerfile
  # Basic HTTP healthcheck
  HEALTHCHECK --interval=30s \
              --timeout=10s \
              --start-period=5s \
              --retries=3 \
    CMD curl -f http://localhost:3000/health || exit 1

  # For images without curl (use wget):
  HEALTHCHECK CMD wget -qO- http://localhost:3000/health || exit 1

  # For non-HTTP services (Postgres):
  HEALTHCHECK --interval=10s --timeout=5s --retries=5 \
    CMD pg_isready -U postgres || exit 1

  # For Redis:
  HEALTHCHECK CMD redis-cli ping | grep PONG || exit 1

  HEALTH STATUS LIFECYCLE:
  starting → (start-period expires) → healthy
                                    → unhealthy (after retries fail)

  docker ps shows STATUS:
  Up 2 minutes (healthy)
  Up 2 minutes (unhealthy)    ← Swarm/K8s will restart or reroute

  ⚠️ Without HEALTHCHECK:
  Docker considers container healthy the moment PID 1 starts.
  App might still be initialising (DB migrations running,
  cache warming) and not ready to serve traffic.
  Add a start-period to give app time to initialise.
```

---

## 6.9 .dockerignore — Control Your Build Context

The build context is everything sent to the Docker daemon when you run
`docker build`. A bloated context = slow builds, large caches, secrets leak.

```
  # .dockerignore
  # ─────────────────────────────────────────
  # Version control
  .git
  .gitignore

  # Node.js
  node_modules
  npm-debug.log*

  # Python
  __pycache__
  *.pyc
  .venv
  venv/

  # Build outputs (if any pre-exist locally)
  dist/
  build/
  target/

  # Environment files (NEVER send to build context)
  .env
  .env.*
  *.env

  # Secrets
  *.pem
  *.key
  *.p12
  id_rsa
  credentials.json

  # Editor / OS noise
  .DS_Store
  .idea/
  .vscode/
  *.swp
  Thumbs.db

  # Test artefacts
  coverage/
  .nyc_output/
  test-results/

  # Docker files (don't copy into image)
  Dockerfile*
  docker-compose*.yml
  .dockerignore

  WHY THIS MATTERS:
  Without .dockerignore, node_modules (500MB+) gets sent to
  the daemon on every build even if you don't COPY it.
  Build context of 1GB+ makes every build feel broken.

  CHECK YOUR BUILD CONTEXT SIZE:
  The first line of docker build output shows it:
  [+] Building 0.3s (3/3) FINISHED
  => [internal] load build context
  => => transferring context: 843B     ← GOOD (only what you need)

  vs.
  => => transferring context: 234.5MB  ← BAD (node_modules snuck in)
```

---

## 6.10 Gotchas — Dockerfile Edition

### ⚠️ Gotcha #16: Shell form CMD swallows shutdown signals

```
  CMD node index.js           ← Shell form
  → runs as: /bin/sh -c "node index.js"
  → PID 1 is /bin/sh
  → SIGTERM on container stop goes to sh
  → sh does NOT forward it to node
  → node gets SIGKILL after timeout (dirty shutdown)
  → in-flight requests dropped, DB connections not closed

  CMD ["node", "index.js"]    ← Exec form ✅
  → node runs as PID 1
  → SIGTERM goes directly to node
  → node's graceful shutdown handler fires
  → requests drain, connections close cleanly

  VERIFY WITH:
  docker stop myapp            # sends SIGTERM
  docker logs myapp            # should show "graceful shutdown" message
  If it doesn't: your CMD is shell form or app isn't handling SIGTERM.
```

### ⚠️ Gotcha #17: COPY . . copies secrets and credentials into image

```
  .env contains:
  DATABASE_URL=postgresql://user:password@prod-db/mydb
  AWS_SECRET_ACCESS_KEY=AKIAIOSFODNN7EXAMPLE

  Dockerfile:
  COPY . .     ← .env is now baked into the image layer

  docker push myrepo/myapp:latest

  Now EVERYONE with pull access to that image can run:
  docker run --rm --entrypoint cat myrepo/myapp:latest /app/.env
  → retrieves all secrets

  Even if you delete it in a later layer:
  COPY . .
  RUN rm .env     ← .env still exists in previous layer!

  FIX:
  1. Add .env to .dockerignore (first line of defence)
  2. Pass secrets at runtime via environment variables or
     secret managers (AWS Secrets Manager, Vault, K8s secrets)
  3. Use BuildKit --secret for build-time secrets
  4. Regularly scan images: trivy image myapp:latest
```

### ⚠️ Gotcha #18: `apt-get update` in a separate RUN layer

```
  BAD:
  RUN apt-get update
  RUN apt-get install -y curl git

  Problem: If image is rebuilt weeks later, "apt-get update"
  layer is CACHED (old package lists). "apt-get install" runs
  against stale lists → may install outdated packages or fail
  with "unable to locate package".

  GOOD — always chain update and install:
  RUN apt-get update && \
      apt-get install -y --no-install-recommends \
        curl \
        git \
      && rm -rf /var/lib/apt/lists/*

  The --no-install-recommends flag alone cuts package install
  size by 30-60% by skipping docs and optional dependencies.
  rm -rf /var/lib/apt/lists/* cleans the apt cache from the layer.
```

### ⚠️ Gotcha #19: WORKDIR vs RUN mkdir

```
  BAD (common pattern from beginners):
  RUN mkdir -p /app
  RUN chown -R node:node /app
  WORKDIR /app

  GOOD:
  WORKDIR /app      ← creates the dir automatically, no RUN needed
  # WORKDIR also sets subsequent RUN/COPY/CMD working directory

  Why it matters: Every unnecessary RUN = an extra layer.
  Extra layers = larger image, slower push/pull.
```

---

## 6.11 Complete Production Dockerfile Examples

### Node.js API — Production Grade

```dockerfile
  # syntax=docker/dockerfile:1
  ARG NODE_VERSION=20.11.0
  ARG ALPINE_VERSION=3.19

  # ── Stage 1: Dependencies ──────────────────────────────
  FROM node:${NODE_VERSION}-alpine${ALPINE_VERSION} AS deps
  WORKDIR /app

  COPY package.json package-lock.json ./
  RUN npm ci --only=production && \
      npm cache clean --force

  # ── Stage 2: Build ─────────────────────────────────────
  FROM node:${NODE_VERSION}-alpine${ALPINE_VERSION} AS build
  WORKDIR /app

  COPY package.json package-lock.json ./
  RUN npm ci                        # includes dev deps for build

  COPY tsconfig.json ./
  COPY src/ ./src/
  RUN npm run build                 # compile TypeScript

  # ── Stage 3: Runtime ───────────────────────────────────
  FROM node:${NODE_VERSION}-alpine${ALPINE_VERSION} AS runtime

  # Security hardening
  RUN apk add --no-cache tini       # zombie reaping (Batch 2)
  WORKDIR /app

  # Copy only what's needed
  COPY --from=deps  --chown=node:node /app/node_modules ./node_modules
  COPY --from=build --chown=node:node /app/dist ./dist

  # Metadata
  ARG BUILD_DATE
  ARG GIT_SHA
  LABEL org.opencontainers.image.created="${BUILD_DATE}"
  LABEL org.opencontainers.image.revision="${GIT_SHA}"
  LABEL org.opencontainers.image.source="https://github.com/myorg/myapp"

  ENV NODE_ENV=production \
      PORT=3000

  EXPOSE 3000

  HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
    CMD wget -qO- http://localhost:3000/health || exit 1

  USER node
  ENTRYPOINT ["/sbin/tini", "--"]
  CMD ["node", "dist/index.js"]
```

### Go Binary — Ultra-Minimal

```dockerfile
  # syntax=docker/dockerfile:1
  FROM golang:1.22-alpine AS builder

  WORKDIR /src

  # Cache module downloads
  COPY go.mod go.sum ./
  RUN go mod download

  # Build static binary
  COPY . .
  RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
      go build -ldflags="-w -s" -o /app/server ./cmd/server

  # ── Runtime: truly empty image ─────────────────────────
  FROM scratch

  # TLS certificates (needed for HTTPS calls)
  COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

  # Timezone data (if your app uses time zones)
  COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo

  # The binary
  COPY --from=builder /app/server /server

  EXPOSE 8080
  ENTRYPOINT ["/server"]
  # Final image size: ~8-15 MB. No shell. No attack surface.
```

---

## 6.12 Production FAQ — Batch 6

---

**Q: My image is 1.4 GB. How do I make it smaller without breaking anything?**

```
A: Systematic size reduction process:

  STEP 1: Audit current size by layer
  $ docker history myimage --no-trunc
  See which layers are biggest. Target those first.

  STEP 2: Switch to a smaller base image
  ubuntu → debian:slim → alpine (test carefully for musl compat)
  openjdk:17 → eclipse-temurin:17-jre-alpine (JRE, not JDK)
  python:3.12 → python:3.12-slim

  STEP 3: Introduce multi-stage builds
  Biggest win for compiled languages and frontend apps.
  Build deps stay in builder stage, only artifacts go to runtime.

  STEP 4: Clean up in the same RUN layer
  apt-get clean && rm -rf /var/lib/apt/lists/*
  pip install --no-cache-dir
  npm ci --only=production && npm cache clean --force

  STEP 5: Remove unnecessary files
  COPY only what you need (use .dockerignore)
  Don't copy test files, docs, CI config into the image.

  STEP 6: Check with dive (image layer explorer)
  $ docker run --rm -it \
    -v /var/run/docker.sock:/var/run/docker.sock \
    wagoodman/dive myimage:latest
  Shows what each layer adds/removes. Find wasted space visually.

  REALISTIC TARGETS:
  Node.js API:    1.4 GB → 180 MB  (alpine + multi-stage)
  Python API:     900 MB → 120 MB  (slim + multi-stage)
  Go service:     900 MB → 12 MB   (scratch + multi-stage)
  Java service:   1.2 GB → 220 MB  (JRE alpine + multi-stage)
```

---

**Q: How do I pass secrets to a container without baking them into the image?**

```
A: Multiple approaches ranked safest to simplest:

  BEST: Secret manager injection at runtime
  AWS Secrets Manager / GCP Secret Manager / HashiCorp Vault
  App fetches secrets on startup using IAM role (no credentials
  in environment or image at all).

  GREAT: Kubernetes Secrets / Docker Swarm secrets
  Secrets mounted as files into /run/secrets/
  docker run --secret source=db-pass,target=/run/secrets/db-pass
  Read from file in app: open('/run/secrets/db-pass').read()
  Not in environment, not in image layers.

  GOOD: Environment variables at runtime (not in Dockerfile)
  docker run -e DATABASE_URL=$DATABASE_URL myapp
  # or in compose:
  services:
    api:
      env_file: .env   # .env is on host, NOT in image

  ACCEPTABLE: CI/CD env vars injected at build time via ARG
  Only for non-sensitive config (feature flags, build metadata).
  Never for passwords, tokens, keys.

  NEVER:
  ENV DATABASE_URL=postgresql://user:pass@host/db   ← baked in image
  COPY .env .                                        ← in image layers
  ARG AWS_SECRET_KEY=AKIA...                         ← in docker history
```

---

**Q: How do I make sure my Dockerfiles are consistent across the whole team?**

```
A: Linting, base image standards, and templates.

  LINTING — hadolint (Dockerfile linter):
  $ docker run --rm -i hadolint/hadolint < Dockerfile
  # Or in CI:
  hadolint Dockerfile --failure-threshold warning

  Catches: wrong form CMD, missing --no-cache-dir, apt-get split
  layers, ADD instead of COPY, missing USER, and 60+ more rules.

  TEAM BASE IMAGES:
  Maintain internal base images: myrepo/node-base:20-hardened
  Pre-configure: non-root user, tini, healthcheck defaults,
  security hardening. Teams inherit from your base.

  FROM myrepo/node-base:20-hardened   ← one line gives everything
  COPY . .
  CMD ["node", "index.js"]

  DOCKERFILE TEMPLATES:
  Store templates in a shared repo: github.com/myorg/dockerfile-templates
  New service? Copy the template for the language, customise.

  SCANNING IN CI:
  trivy image myapp:latest            ← scan for CVEs before push
  Fail CI if HIGH or CRITICAL CVEs found.
  Schedule weekly rescans of all images (base image CVEs appear later).
```

---

*End of Batch 6.*

---

# BATCH 7
## Docker Compose — Dev & Production Patterns

> **Goal:** Master Docker Compose as both a local development superpower and a
> lightweight production orchestrator. Understand the full schema, service
> dependency ordering, override files, scaling, and the patterns teams actually
> use in real projects.

---

## 7.1 What Docker Compose Is (and Isn't)

```
  Docker Compose = a tool for defining and running
  MULTI-CONTAINER applications from a single YAML file.

  ONE COMMAND to:
  docker compose up      → build images, create networks,
                           create volumes, start all containers
                           in the right order

  docker compose down    → stop and remove containers, networks
                           (volumes preserved by default)

  WHAT COMPOSE IS:
  ✅ Local development environment orchestrator
  ✅ Integration test environment runner (CI)
  ✅ Lightweight single-host production deployment
  ✅ Self-documenting infrastructure-as-code for a service

  WHAT COMPOSE IS NOT:
  ✗ A replacement for Kubernetes (no auto-healing, no
    cross-host scheduling, no rolling deploys built-in)
  ✗ A secret manager
  ✗ A load balancer for distributed traffic

  VERSION NOTE:
  Compose V1 (python):  docker-compose (hyphen)  ← deprecated
  Compose V2 (Go):      docker compose (space)   ← current standard
  V2 is bundled with Docker Engine 23+.
  Always use V2. Drop the hyphen from your muscle memory.
```

---

## 7.2 Compose File Structure — Full Schema Map

```yaml
  # docker-compose.yml top-level keys
  name: myapp                  # Project name (default: directory name)

  services:                    # Container definitions (required)
    servicename:
      # --- Image ---
      image: nginx:1.25-alpine           # use pre-built image
      build:                             # OR build from Dockerfile
        context: ./backend
        dockerfile: Dockerfile.prod
        args:
          NODE_ENV: production
        target: runtime                  # multi-stage target
        cache_from:
          - myrepo/myapp:latest          # warm cache from registry

      # --- Identity ---
      container_name: myapp-api          # fixed name (breaks scaling!)
      hostname: api

      # --- Runtime ---
      command: ["node", "dist/index.js"] # override CMD
      entrypoint: ["/sbin/tini", "--"]   # override ENTRYPOINT
      working_dir: /app
      user: "1000:1000"

      # --- Environment ---
      environment:
        NODE_ENV: production
        PORT: "3000"
      env_file:
        - .env                           # loaded from file
        - .env.local                     # can stack multiple

      # --- Ports ---
      ports:
        - "3000:3000"                    # host:container
        - "127.0.0.1:9229:9229"         # localhost only (debugger)

      expose:
        - "3000"                         # document internal port only

      # --- Storage ---
      volumes:
        - pgdata:/var/lib/postgresql/data  # named volume
        - ./src:/app/src                   # bind mount
        - type: tmpfs
          target: /tmp

      # --- Networking ---
      networks:
        - frontend
        - backend
      dns:
        - 10.0.0.53

      # --- Dependencies ---
      depends_on:
        db:
          condition: service_healthy     # wait for healthcheck ✅
        redis:
          condition: service_started     # just wait for start

      # --- Health ---
      healthcheck:
        test: ["CMD", "wget", "-qO-", "http://localhost:3000/health"]
        interval: 30s
        timeout: 10s
        retries: 3
        start_period: 10s

      # --- Lifecycle ---
      restart: unless-stopped            # always | no | on-failure
      stop_grace_period: 30s            # time before SIGKILL

      # --- Resources ---
      deploy:
        resources:
          limits:
            cpus: "0.5"
            memory: 512M
          reservations:
            memory: 256M
        replicas: 2                      # scale in Swarm mode

      # --- Logging ---
      logging:
        driver: json-file
        options:
          max-size: "10m"
          max-file: "3"

      # --- Security ---
      security_opt:
        - no-new-privileges:true
      cap_drop:
        - ALL
      cap_add:
        - NET_BIND_SERVICE
      read_only: true
      tmpfs:
        - /tmp
        - /run

  volumes:                     # Named volume declarations
    pgdata:
      driver: local
    assets:

  networks:                    # Network declarations
    frontend:
      driver: bridge
    backend:
      driver: bridge
      internal: true           # no internet access ← security!

  secrets:                     # Swarm secrets (or file-based)
    db_password:
      file: ./secrets/db_password.txt
```

---

## 7.3 Service Dependencies — The `depends_on` Deep Dive

```
  THE PROBLEM:

  App starts → tries to connect to DB → DB isn't ready → CRASH

  This is the most common Compose beginner issue.

  WRONG mental model: depends_on means "wait until DB is ready"
  CORRECT mental model (Compose V1 default): depends_on means
    "start DB container before app container" — that's all.
    Container started ≠ service inside is ready.

  ┌──────────────────────────────────────────────────────────┐
  │  COMPOSE V2 depends_on CONDITIONS:                       │
  │                                                          │
  │  condition: service_started    → container is running    │
  │             (default)            (NOT necessarily ready) │
  │                                                          │
  │  condition: service_healthy    → container's HEALTHCHECK │
  │                                  reports healthy ✅      │
  │                                  USE THIS for databases  │
  │                                                          │
  │  condition: service_completed_ → container exited 0     │
  │             successfully         Use for init jobs,      │
  │                                  DB migrations           │
  └──────────────────────────────────────────────────────────┘
```

### Correct Pattern: `service_healthy`

```yaml
  services:
    db:
      image: postgres:16-alpine
      environment:
        POSTGRES_USER: app
        POSTGRES_PASSWORD: secret
        POSTGRES_DB: myapp
      healthcheck:
        test: ["CMD-SHELL", "pg_isready -U app -d myapp"]
        interval: 5s
        timeout: 5s
        retries: 10
        start_period: 10s       # give postgres time to initialise
      volumes:
        - pgdata:/var/lib/postgresql/data

    api:
      build: ./api
      depends_on:
        db:
          condition: service_healthy   # waits for pg_isready ✅
      environment:
        DATABASE_URL: postgresql://app:secret@db:5432/myapp

  volumes:
    pgdata:
```

### Migration Job Pattern (`service_completed_successfully`)

```yaml
  services:
    db:
      image: postgres:16-alpine
      healthcheck:
        test: ["CMD-SHELL", "pg_isready -U app"]
        interval: 5s
        retries: 10

    migrate:
      build: ./api
      command: ["node", "dist/migrate.js"]   # runs migrations, exits 0
      depends_on:
        db:
          condition: service_healthy
      environment:
        DATABASE_URL: postgresql://app:secret@db:5432/myapp

    api:
      build: ./api
      depends_on:
        migrate:
          condition: service_completed_successfully  # wait for migrate job
        db:
          condition: service_healthy
      environment:
        DATABASE_URL: postgresql://app:secret@db:5432/myapp
```

---

## 7.4 The Override File Pattern — Dev vs Prod

This is the professional way to manage environment-specific config.
You have a **base** compose file and **override** files layered on top.

```
  FILE STRUCTURE:
  ─────────────────────────────────────────────────────────
  docker-compose.yml          ← base (shared across all envs)
  docker-compose.override.yml ← dev overrides (auto-loaded locally)
  docker-compose.prod.yml     ← production overrides
  docker-compose.test.yml     ← CI/test overrides

  HOW MERGING WORKS:
  docker compose up
  → auto-merges: docker-compose.yml + docker-compose.override.yml

  docker compose -f docker-compose.yml \
                 -f docker-compose.prod.yml up
  → merges base + prod overrides

  MERGE RULES:
  - Scalars (strings, numbers): override REPLACES base value
  - Lists (ports, volumes):     override APPENDS to base list
  - Maps (environment):         override MERGES with base map
```

### Base File — Environment-Agnostic

```yaml
  # docker-compose.yml  (base — commit this)
  name: myapp

  services:
    api:
      build:
        context: ./api
        target: runtime
      environment:
        NODE_ENV: production
        PORT: "3000"
      expose:
        - "3000"
      healthcheck:
        test: ["CMD", "wget", "-qO-", "http://localhost:3000/health"]
        interval: 30s
        timeout: 10s
        retries: 3
        start_period: 15s
      depends_on:
        db:
          condition: service_healthy
      restart: unless-stopped
      networks:
        - backend

    db:
      image: postgres:16-alpine
      environment:
        POSTGRES_USER: app
        POSTGRES_DB: myapp
      healthcheck:
        test: ["CMD-SHELL", "pg_isready -U app -d myapp"]
        interval: 5s
        retries: 10
      volumes:
        - pgdata:/var/lib/postgresql/data
      restart: unless-stopped
      networks:
        - backend

    nginx:
      image: nginx:1.25-alpine
      ports:
        - "80:80"
      depends_on:
        api:
          condition: service_healthy
      networks:
        - frontend
        - backend

  volumes:
    pgdata:

  networks:
    frontend:
    backend:
      internal: true
```

### Dev Override — Hot Reload, Debug Ports, Relaxed Security

```yaml
  # docker-compose.override.yml  (auto-loaded in dev, gitignore .env)
  services:
    api:
      build:
        target: development          # uses dev stage with nodemon
      command: ["npm", "run", "dev"]
      environment:
        NODE_ENV: development
        LOG_LEVEL: debug
      volumes:
        - ./api/src:/app/src         # hot-reload source
        - ./api/package.json:/app/package.json
        - node_modules:/app/node_modules
      ports:
        - "3000:3000"                # expose API directly in dev
        - "127.0.0.1:9229:9229"     # Node.js debugger
      restart: "no"                  # don't auto-restart in dev

    db:
      environment:
        POSTGRES_PASSWORD: devpassword   # simple pass in dev
      ports:
        - "127.0.0.1:5432:5432"         # expose DB to host tools
                                         # (TablePlus, pgAdmin etc.)

  volumes:
    node_modules:
```

### Production Override — Resources, Logging, Hardening

```yaml
  # docker-compose.prod.yml
  services:
    api:
      image: myrepo/myapp-api:${IMAGE_TAG}  # pre-built image, no build
      environment:
        NODE_ENV: production
        LOG_LEVEL: warn
      env_file:
        - /etc/myapp/secrets.env       # secrets on host, not in repo
      deploy:
        resources:
          limits:
            cpus: "1.0"
            memory: 512M
      logging:
        driver: json-file
        options:
          max-size: "20m"
          max-file: "5"
      security_opt:
        - no-new-privileges:true
      cap_drop:
        - ALL
      read_only: true
      tmpfs:
        - /tmp

    db:
      environment:
        POSTGRES_PASSWORD_FILE: /run/secrets/pg_password
      secrets:
        - pg_password

    nginx:
      ports:
        - "80:80"
        - "443:443"
      volumes:
        - ./nginx/prod.conf:/etc/nginx/nginx.conf:ro
        - /etc/letsencrypt:/etc/letsencrypt:ro

  secrets:
    pg_password:
      file: /etc/myapp/pg_password

  # Deploy:
  # IMAGE_TAG=v1.2.3 docker compose \
  #   -f docker-compose.yml \
  #   -f docker-compose.prod.yml \
  #   up -d
```

---

## 7.5 Profiles — Conditional Services

Profiles let you define services that only start when explicitly requested.
Ideal for dev tools, one-off jobs, and optional infrastructure.

```yaml
  services:
    api:
      build: ./api
      # no profile = always starts

    db:
      image: postgres:16-alpine
      # no profile = always starts

    adminer:
      image: adminer:latest         # DB web UI — dev only
      profiles: [dev]               # only starts with --profile dev
      ports:
        - "8080:8080"
      depends_on:
        - db

    mailhog:
      image: mailhog/mailhog         # fake SMTP — dev only
      profiles: [dev]
      ports:
        - "1025:1025"
        - "8025:8025"

    migrate:
      build: ./api
      command: ["node", "dist/migrate.js"]
      profiles: [tools]              # run manually when needed
      depends_on:
        db:
          condition: service_healthy

  # USAGE:
  docker compose up                  # starts: api, db
  docker compose --profile dev up    # starts: api, db, adminer, mailhog
  docker compose --profile tools run migrate  # one-off migration
```

---

## 7.6 Compose for CI — Integration Testing Pattern

```yaml
  # docker-compose.test.yml
  services:
    db:
      image: postgres:16-alpine
      environment:
        POSTGRES_USER: test
        POSTGRES_PASSWORD: test
        POSTGRES_DB: testdb
      healthcheck:
        test: ["CMD-SHELL", "pg_isready -U test"]
        interval: 2s
        retries: 15
      # No volumes — fresh DB per CI run (ephemeral is a feature!)

    redis:
      image: redis:7-alpine
      healthcheck:
        test: ["CMD", "redis-cli", "ping"]
        interval: 2s
        retries: 10

    test:
      build:
        context: .
        target: test                 # use test stage from Dockerfile
      command: ["npm", "run", "test:integration"]
      environment:
        DATABASE_URL: postgresql://test:test@db:5432/testdb
        REDIS_URL: redis://redis:6379
        NODE_ENV: test
      depends_on:
        db:
          condition: service_healthy
        redis:
          condition: service_healthy
```

```bash
  # In CI pipeline (GitHub Actions / GitLab CI):

  # Run tests — exit code of 'test' service propagates
  docker compose -f docker-compose.test.yml \
    up --abort-on-container-exit --exit-code-from test

  # Always clean up, even on failure
  docker compose -f docker-compose.test.yml down --volumes

  # --abort-on-container-exit: stops all services when any exits
  # --exit-code-from test:     CI gets exit code of 'test' container
  # --volumes:                 removes anonymous volumes too (clean state)
```

---

## 7.7 Scaling and Load Balancing with Compose

```yaml
  services:
    api:
      build: ./api
      # NOTE: remove container_name for scaling to work
      # container_name: myapp-api  ← breaks scaling
      expose:
        - "3000"
      deploy:
        replicas: 3               # in Swarm mode
      # OR at runtime:
      # docker compose up --scale api=3

    nginx:
      image: nginx:1.25-alpine
      ports:
        - "80:80"
      volumes:
        - ./nginx/upstream.conf:/etc/nginx/conf.d/default.conf
      depends_on:
        - api
```

```nginx
  # nginx/upstream.conf
  upstream api_cluster {
    server api:3000;        # Compose DNS resolves 'api' to ALL
                            # running api containers automatically!
                            # Docker's internal DNS round-robins them.
  }

  server {
    listen 80;
    location / {
      proxy_pass http://api_cluster;
    }
  }
```

```
  SCALING FLOW:
  ─────────────────────────────────────────────────────────
  docker compose up --scale api=3

  Creates: myapp-api-1, myapp-api-2, myapp-api-3
  Each gets its own IP on the backend network.
  Docker DNS for 'api' resolves to ALL THREE IPs.
  Nginx upstream sees all three → distributes traffic.

  ⚠️ COMPOSE SCALE LIMITATIONS:
  - No health-based routing (nginx won't skip unhealthy api)
  - No rolling deploys (all containers restart at once)
  - Single host only
  For true multi-instance HA: use Kubernetes or Docker Swarm.
```

---

## 7.8 Essential Compose Commands Reference

```
  COMMAND                              WHAT IT DOES
  ─────────────────────────────────────────────────────────────────
  docker compose up                    Start all services
  docker compose up -d                 Start detached (background)
  docker compose up --build            Force rebuild before start
  docker compose up api db             Start specific services only

  docker compose down                  Stop + remove containers & networks
  docker compose down -v               Also remove named volumes ⚠️
  docker compose down --rmi all        Also remove built images

  docker compose ps                    List running services
  docker compose ps -a                 Include stopped containers

  docker compose logs                  All service logs
  docker compose logs -f api           Follow logs for 'api' service
  docker compose logs --tail=50 api    Last 50 lines

  docker compose exec api sh           Shell into running api container
  docker compose run --rm api sh       Start NEW api container + shell

  docker compose build                 Build all services
  docker compose build api             Build specific service
  docker compose build --no-cache api  Build without cache

  docker compose pull                  Pull latest images
  docker compose push                  Push built images to registry

  docker compose restart api           Restart specific service
  docker compose stop api              Stop without removing
  docker compose start api             Start a stopped service
  docker compose kill api              SIGKILL a service

  docker compose scale api=3           Scale service (V1 syntax)
  docker compose up --scale api=3      Scale service (V2 syntax)

  docker compose top                   Show processes in each container
  docker compose stats                 Live resource usage

  docker compose config                Validate + print merged config
  docker compose config --services     List service names only

  docker compose cp api:/app/log.txt ./  Copy file from container

  # Override file usage:
  docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
  docker compose -p myproject up       Use custom project name
```

---

## 7.9 Gotchas — Compose Edition

### ⚠️ Gotcha #20: `depends_on` does NOT wait for service readiness by default

```
  Already covered in 7.3, but worth repeating as the #1 Compose trap:

  SYMPTOM: App crashes on startup with "connection refused" to DB.
           Restarting the app manually fixes it.

  CAUSE: depends_on: db (no condition) just ensures DB CONTAINER
         starts before app container. Postgres takes 2-5 seconds
         to initialise. App is already trying to connect.

  FIX: Add healthcheck to db + use condition: service_healthy.
  Or: write retry logic into your app's DB connection code.
  Both are good practices. Do both.
```

### ⚠️ Gotcha #21: `docker compose down -v` destroys your data

```
  docker compose down      → removes containers, networks
                             KEEPS volumes (your data is safe)

  docker compose down -v   → removes containers, networks,
                             AND ALL NAMED VOLUMES
                             YOUR DATABASE IS GONE

  This is the Docker Compose equivalent of rm -rf on your data.

  WHEN IS -v USEFUL?
  - CI pipelines (fresh state each run)
  - Resetting a dev environment deliberately
  - Never in production

  SAFER ALTERNATIVE — remove only anonymous volumes:
  docker compose down --remove-orphans
  # then manually remove specific volumes if needed:
  docker volume rm myapp_pgdata
```

### ⚠️ Gotcha #22: `container_name` breaks horizontal scaling

```
  container_name: myapp-api

  If you set a fixed container name, you can only ever run
  ONE instance of that service. Docker enforces unique names.

  docker compose up --scale api=3
  → ERROR: container name "myapp-api" is already in use

  FIX: Remove container_name from any service you want to scale.
  Let Compose auto-name them: myapp-api-1, myapp-api-2, etc.

  Use container_name ONLY for services that are truly singletons
  (DB, reverse proxy) and where you need a stable name for
  external tooling (backup scripts, monitoring).
```

### ⚠️ Gotcha #23: Env var precedence is a source of "why is this wrong?"

```
  Compose env var precedence (highest → lowest):
  ─────────────────────────────────────────────────────
  1. Shell env vars set before running compose
     $ FOO=bar docker compose up
  2. .env file in the same dir as docker-compose.yml
     (used for COMPOSE variable substitution in YAML)
  3. environment: block in docker-compose.yml
  4. env_file: file referenced in compose service
  5. ENV instruction baked into Dockerfile

  COMMON CONFUSION:
  .env file sets FOO=dev
  docker-compose.yml has environment: FOO: prod
  → container gets FOO=prod (environment: wins over .env)

  BUT:
  .env file sets COMPOSE_PROJECT_NAME=myapp
  → this affects compose itself, not the container!
  .env is for compose variable substitution first.

  DEBUG ENV VARS:
  docker compose config   → shows fully resolved compose file
                            with all substitutions applied
  docker exec api env     → shows what container actually has
```

---

## 7.10 Production FAQ — Batch 7

---

**Q: Should I use Docker Compose in production or switch to Kubernetes?**

```
A: Honest answer — it depends on scale and complexity.

  USE COMPOSE IN PRODUCTION WHEN:
  ─────────────────────────────────────────────────────────
  ✅ Single server / VPS deployment (Hetzner, DigitalOcean droplet)
  ✅ Small team (1-5 engineers), simple service topology
  ✅ Low traffic, modest scaling needs (1-2 instances per service)
  ✅ You want operational simplicity over theoretical scalability
  ✅ Budget constraints (K8s infra + ops overhead is real cost)
  ✅ Internal tools, side projects, early stage startups

  MOVE TO KUBERNETES (or managed equivalent) WHEN:
  ─────────────────────────────────────────────────────────
  ✅ You need zero-downtime rolling deployments
  ✅ You need auto-scaling (scale on CPU/memory/custom metrics)
  ✅ Multi-host / multi-zone HA requirements
  ✅ Many teams deploying to same infrastructure
  ✅ Service mesh, advanced networking, RBAC requirements
  ✅ Compliance requires audit trails, pod security policies

  MANY SUCCESSFUL COMPANIES run Compose in production:
  It's not wrong. It's a different operational point on the
  complexity vs capability tradeoff curve.

  THE MIGRATION PATH:
  Compose → Docker Swarm (easy, same YAML syntax, multi-host)
          → Kubernetes (more powerful, steeper curve)
  Don't jump to K8s before you've outgrown Compose.
```

---

**Q: How do I do zero-downtime deployments with Docker Compose?**

```
A: Compose doesn't have built-in rolling deploys, but you can
   approximate it with a few techniques:

  TECHNIQUE 1 — Blue/Green with nginx upstream swap
  ─────────────────────────────────────────────────
  Run two compose projects simultaneously (blue and green).
  Update nginx upstream to point to new (green) when ready.
  Stop old (blue) after confirming green is healthy.

  docker compose -p blue up -d   # current production
  docker compose -p green up -d  # new version
  # test green
  # swap nginx config to point to green
  docker compose -p blue down    # remove old

  TECHNIQUE 2 — Scale up, then scale down
  ─────────────────────────────────────────────────
  docker compose up --scale api=2 --no-recreate  # add new
  # wait for new containers to pass healthcheck
  docker compose up --scale api=1 --no-recreate  # remove old
  # This is imprecise — Compose doesn't control WHICH to remove

  TECHNIQUE 3 — Use a proper orchestrator for this
  ─────────────────────────────────────────────────
  For true zero-downtime: Docker Swarm rolling updates
  or Kubernetes rolling deployments are the right tool.

  docker service update --image myapp:v2 \
    --update-parallelism 1 \
    --update-delay 10s \
    myapp_api
```

---

**Q: My docker compose up takes 5 minutes because it always rebuilds. How do I fix it?**

```
A: Multiple causes, each with a fix:

  CAUSE 1: Build cache not being used
  ─────────────────────────────────────────────────────
  Check if .dockerignore is missing or too permissive.
  docker compose build --progress=plain api
  → watch each step. "CACHED" = good. Long RUN steps = fix layers.

  CAUSE 2: Layer ordering (covered in Batch 6)
  ─────────────────────────────────────────────────────
  COPY . . before npm install → cache busts on every file change.
  Fix: COPY package.json first, RUN npm ci, THEN COPY src.

  CAUSE 3: BuildKit not enabled (old installs)
  ─────────────────────────────────────────────────────
  export DOCKER_BUILDKIT=1   # enable BuildKit
  # Or in daemon.json: {"features": {"buildkit": true}}
  BuildKit: parallel stage builds, better cache, faster.

  CAUSE 4: Pulling from registry every time
  ─────────────────────────────────────────────────────
  In CI: use cache_from in compose build config:
  build:
    cache_from:
      - myrepo/myapp:latest    # warm from last push
  docker compose build --pull api  # pull cache_from images first

  CAUSE 5: No cache between CI runs
  ─────────────────────────────────────────────────────
  CI runners are stateless — cache is lost between jobs.
  Solutions:
  - Push a cache image after each build
  - Use GitHub Actions cache action with BuildKit
  - Use registry-based cache: --cache-to type=registry,...
```

---

**Q: How do I manage secrets in Docker Compose without putting them in the YAML file?**

```
A: Several levels, pick what fits your threat model:

  LEVEL 1 — .env file (simple, good for dev)
  ─────────────────────────────────────────────────────
  .env:
    POSTGRES_PASSWORD=secretpass
    API_KEY=myapikey

  docker-compose.yml:
    services:
      db:
        environment:
          POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}

  Add .env to .gitignore. Share via password manager or vault.
  docker compose config shows resolved values — don't log it.

  LEVEL 2 — Secrets mounted as files (more secure)
  ─────────────────────────────────────────────────────
  # Store secret in a file on host (not in repo)
  echo "supersecretpassword" > /etc/myapp/db_password

  docker-compose.yml:
    services:
      db:
        environment:
          POSTGRES_PASSWORD_FILE: /run/secrets/db_password
        secrets:
          - db_password
    secrets:
      db_password:
        file: /etc/myapp/db_password

  Secret is mounted at /run/secrets/db_password inside container.
  Not visible in environment, not in compose YAML.

  LEVEL 3 — External secret manager at runtime
  ─────────────────────────────────────────────────────
  App fetches secrets from Vault / AWS SSM / 1Password CLI
  at startup. No secrets in compose files or env at all.
  Requires app-level integration or a sidecar agent.

  WHAT TO NEVER DO:
  - Hardcode in docker-compose.yml (committed to git)
  - Hardcode in Dockerfile ENV (baked into image)
  - Pass via --build-arg (visible in docker history)
```

---

*End of Batch 7.*

---

# BATCH 8
## Docker Security — Scanning, Runtime Hardening, Seccomp, AppArmor, Rootless & Supply Chain

> **Goal:** Understand the full Docker threat model and every layer of defence available.
> Security is not a single feature — it's a stack of overlapping controls. This batch
> goes from image-level scanning to kernel-level syscall filtering to supply chain
> integrity. Production systems need all of it.

---

## 8.1 The Docker Security Threat Model

Before picking controls, understand what you're defending against:

```
  THREAT SURFACE MAP
  ══════════════════════════════════════════════════════════════

  LAYER              THREAT                        CONTROL
  ─────────────────────────────────────────────────────────────
  Supply chain       Malicious base image           Image signing (Cosign)
                     Compromised dependency          CVE scanning (Trivy)
                     Tampered registry image         Digest pinning

  Image build        Secrets baked into layers       .dockerignore, BuildKit secrets
                     Root user baked in              USER instruction
                     Unnecessary packages            Minimal base images

  Container runtime  Privilege escalation            --cap-drop ALL
                     Container escape                seccomp, AppArmor, gVisor
                     Resource exhaustion             --memory, --cpus, --pids-limit
                     Writable filesystem             --read-only
                     Ambient capabilities            no-new-privileges

  Networking         Lateral movement                Network segmentation (Batch 4)
                     Port exposure                   Bind to 127.0.0.1 only
                     Unencrypted traffic             TLS, service mesh mTLS

  Host               docker.sock exposure            Never mount in prod
                     Privileged containers           Never --privileged
                     Kernel vulnerabilities          Keep host patched

  Registry           Unauthorized pull/push          Registry auth, RBAC
                     Image substitution attacks      Tag immutability, digest pins

  ══════════════════════════════════════════════════════════════
  Security is a STACK. Each layer defends against different
  attacker capabilities. Assume every other layer is breached.
```

---

## 8.2 Image Scanning — Finding CVEs Before They Find You

CVE = Common Vulnerability and Exposure. A known, catalogued security flaw
in a software package. Your images contain dozens of packages; each is a
potential CVE carrier.

### Trivy — The Industry Standard Scanner

```bash
  # Install
  brew install trivy                    # macOS
  apt install trivy                     # Ubuntu (via repo)
  docker run aquasec/trivy image myapp  # no install needed

  # Scan a local image
  trivy image myapp:latest

  # Scan with severity filter
  trivy image --severity HIGH,CRITICAL myapp:latest

  # Fail CI if any CRITICAL CVEs found
  trivy image --exit-code 1 --severity CRITICAL myapp:latest

  # Scan a Dockerfile for misconfigurations
  trivy config ./Dockerfile

  # Scan a docker-compose.yml
  trivy config ./docker-compose.yml

  # Scan a running container's filesystem
  trivy rootfs /path/to/rootfs

  # Generate SARIF report (for GitHub Security tab)
  trivy image --format sarif --output results.sarif myapp:latest

  # Scan for secrets accidentally baked into image
  trivy image --scanners secret myapp:latest
```

### Reading Trivy Output

```
  myapp:latest (alpine 3.18.4)
  ════════════════════════════
  Total: 3 (HIGH: 2, CRITICAL: 1)

  ┌────────────────┬───────────────┬──────────┬──────────────────┬───────────┐
  │   Library      │ Vulnerability │ Severity │ Installed Ver    │ Fixed Ver │
  ├────────────────┼───────────────┼──────────┼──────────────────┼───────────┤
  │ openssl        │ CVE-2023-5678 │ CRITICAL │ 3.1.3-r0         │ 3.1.4-r0  │
  │ libssl3        │ CVE-2023-5678 │ HIGH     │ 3.1.3-r0         │ 3.1.4-r0  │
  │ libcrypto3     │ CVE-2023-4321 │ HIGH     │ 3.1.3-r0         │ 3.1.4-r0  │
  └────────────────┴───────────────┴──────────┴──────────────────┴───────────┘

  ACTION: Update base image FROM alpine:3.18.4 → alpine:3.19.0
  openssl 3.1.4-r0 is available there.

  WORKFLOW:
  1. Scan on every image build (CI gate)
  2. Scan scheduled weekly (new CVEs appear after build)
  3. Alert on new HIGH/CRITICAL in registry (ECR Inspector, GCR)
  4. Remediate: rebuild with updated base or dependency
```

### Other Scanning Tools

```
  TOOL          STRENGTHS                        WHEN TO USE
  ───────────────────────────────────────────────────────────────
  Trivy         Free, fast, broad (OS+lang+IaC)  Default choice
  Grype         Accurate OS+lang scanning        Pair with Trivy
  Snyk          Dev-friendly, IDE integration    Shift left in IDE
  Docker Scout  Built into Docker Desktop        Quick local scans
  Clair         Self-hosted, K8s-native          Air-gapped envs
  ECR Inspector AWS-native, auto-scans on push   AWS deployments
  GCR Artifact  GCP-native scanning              GCP deployments
  Analysis
```

---

## 8.3 Runtime Hardening — Defence in Depth

These controls apply when running containers (`docker run` flags or compose config).
Each removes a capability or restricts a resource that an attacker could exploit.

### The Hardened `docker run` Template

```bash
  docker run \
    --name myapp \
    --user 1000:1000 \               # non-root user
    --read-only \                    # immutable root filesystem
    --tmpfs /tmp:rw,noexec,nosuid \  # writable tmp in RAM only
    --tmpfs /run:rw,noexec,nosuid \  #   noexec: can't run binaries
    --cap-drop ALL \                 # drop ALL linux capabilities
    --cap-add NET_BIND_SERVICE \     # add back only what's needed
    --security-opt no-new-privileges:true \  # prevent privilege escalation
    --security-opt seccomp=./seccomp.json \  # syscall allowlist
    --memory=512m \                  # hard memory limit
    --memory-reservation=256m \      # soft memory target
    --cpus=0.5 \                     # CPU limit
    --pids-limit=100 \               # prevent fork bombs
    --network myapp-net \            # isolated network
    --restart unless-stopped \
    myapp:latest
```

### Docker Compose Equivalent

```yaml
  services:
    api:
      image: myapp:latest
      user: "1000:1000"
      read_only: true
      tmpfs:
        - /tmp:rw,noexec,nosuid
        - /run:rw,noexec,nosuid
      cap_drop:
        - ALL
      cap_add:
        - NET_BIND_SERVICE
      security_opt:
        - no-new-privileges:true
        - seccomp:./seccomp-profile.json
      deploy:
        resources:
          limits:
            memory: 512M
            cpus: "0.5"
          reservations:
            memory: 256M
      ulimits:
        nproc: 100               # process limit (fork bomb prevention)
        nofile:
          soft: 1024
          hard: 2048
```

---

## 8.4 Linux Capabilities — What They Are and Why to Drop Them

Linux capabilities break the all-or-nothing `root` privilege into
fine-grained permissions. Containers get a default set — most apps need none.

```
  DEFAULT CONTAINER CAPABILITIES (Docker grants these by default):
  ──────────────────────────────────────────────────────────────
  AUDIT_WRITE     Write audit log records
  CHOWN           Make arbitrary changes to file UIDs/GIDs
  DAC_OVERRIDE    Bypass file permission checks
  FOWNER          Bypass permission checks for file owner
  FSETID          Set file set-user/group-ID bits
  KILL            Send signals to processes
  MKNOD           Create special files (devices)
  NET_BIND_SERVICE Bind to ports < 1024
  NET_RAW         Use raw/packet sockets (ping, sniff traffic!)
  SETFCAP         Set file capabilities
  SETGID          Change process group ID
  SETPCAP         Manage process capabilities
  SETUID          Change process user ID
  SYS_CHROOT      Use chroot()

  Most web apps need ZERO of these.
  NET_RAW alone lets a container do ARP spoofing on the bridge!

  PRINCIPLE: Drop ALL, add back the minimum needed.

  CAPABILITIES YOUR APP MIGHT LEGITIMATELY NEED:
  ──────────────────────────────────────────────────────────────
  NET_BIND_SERVICE → bind to port 80/443 (or use port > 1024!)
  SYS_PTRACE       → debuggers, strace (dev only, never prod)
  NET_ADMIN        → network tools (tcpdump, iptables)
  DAC_READ_SEARCH  → read files regardless of permissions (backups)

  CHECK WHAT YOUR APP ACTUALLY USES:
  # Run with strace and watch for cap-requiring syscalls:
  docker run --cap-add SYS_PTRACE myapp strace -e trace=all ./app 2>&1 \
    | grep "EPERM\|Operation not permitted"
```

---

## 8.5 Seccomp — Syscall Filtering

Seccomp (Secure Computing Mode) is a Linux kernel feature that restricts
which system calls a process can make. It's the deepest defence layer —
even if an attacker escapes all other controls, they hit the syscall filter.

```
  WHY SYSCALL FILTERING MATTERS:
  ─────────────────────────────────────────────────────────────
  Container escape attacks typically use dangerous syscalls:
  - ptrace  → attach to processes, read/write their memory
  - mount   → remount /proc, bypass namespace isolation
  - clone   → create new namespaces (escape attempt)
  - keyctl  → access kernel keyring
  - perf_event_open → side-channel attacks

  If these syscalls are BLOCKED, the attack fails at kernel level.
  No software patch needed. Kernel enforces it.
```

### Docker's Default Seccomp Profile

```
  Docker applies a default seccomp profile that blocks ~44 syscalls.
  It's a good baseline but not minimal.

  Check if seccomp is active:
  $ docker inspect mycontainer | grep -i seccomp
  "SecurityOpt": ["seccomp=..."]   → active
  "SecurityOpt": null              → no seccomp (check daemon config)

  Syscalls blocked by default (important ones):
  - add_key, request_key    → kernel keyring access
  - bpf                     → Berkeley Packet Filter programs
  - clock_adjtime           → adjust system clock
  - clone (with new flags)  → namespace creation
  - create_module           → load kernel modules
  - finit_module            → install kernel module from fd
  - get_mempolicy           → NUMA policy (rarely needed)
  - kexec_file_load         → load new kernel
  - mount                   → filesystem mounting
  - perf_event_open         → performance monitoring (side-channel risk)
  - ptrace                  → process tracing
  - reboot                  → reboot the system
  - setns                   → join existing namespaces
  - settimeofday            → set system time
  - swapon/swapoff          → manage swap
  - syslog                  → read kernel message buffer
  - umount2                 → unmount filesystems
  - unshare                 → create new namespaces
```

### Custom Seccomp Profile — Minimal Allowlist

```json
  // seccomp-minimal.json — allowlist approach (safest)
  {
    "defaultAction": "SCMP_ACT_ERRNO",
    "architectures": ["SCMP_ARCH_X86_64", "SCMP_ARCH_AARCH64"],
    "syscalls": [
      {
        "names": [
          "read", "write", "open", "close", "stat", "fstat",
          "lstat", "poll", "lseek", "mmap", "mprotect", "munmap",
          "brk", "rt_sigaction", "rt_sigprocmask", "rt_sigreturn",
          "ioctl", "pread64", "pwrite64", "readv", "writev",
          "access", "pipe", "select", "sched_yield", "mremap",
          "msync", "mincore", "madvise", "dup", "dup2", "pause",
          "nanosleep", "getitimer", "alarm", "setitimer", "getpid",
          "sendfile", "socket", "connect", "accept", "sendto",
          "recvfrom", "sendmsg", "recvmsg", "shutdown", "bind",
          "listen", "getsockname", "getpeername", "socketpair",
          "getsockopt", "setsockopt", "clone", "fork", "vfork",
          "execve", "exit", "wait4", "kill", "uname", "fcntl",
          "flock", "fsync", "fdatasync", "truncate", "ftruncate",
          "getdents", "getcwd", "chdir", "rename", "mkdir",
          "rmdir", "creat", "link", "unlink", "symlink",
          "readlink", "chmod", "fchmod", "chown", "fchown",
          "lchown", "umask", "gettimeofday", "getrlimit",
          "getrusage", "sysinfo", "times", "ptrace", "getuid",
          "syslog", "getgid", "setuid", "setgid", "geteuid",
          "getegid", "setpgid", "getppid", "getpgrp", "setsid",
          "capget", "capset", "rt_sigpending", "rt_sigtimedwait",
          "rt_sigqueueinfo", "rt_sigsuspend", "sigaltstack",
          "arch_prctl", "setrlimit", "getpriority", "setpriority",
          "futex", "sched_getaffinity", "set_thread_area",
          "get_thread_area", "set_tid_address", "clock_gettime",
          "clock_getres", "clock_nanosleep", "exit_group",
          "epoll_wait", "epoll_ctl", "tgkill", "waitid",
          "openat", "mkdirat", "mknodat", "fchownat", "futimesat",
          "fstatat", "unlinkat", "renameat", "linkat", "symlinkat",
          "readlinkat", "fchmodat", "faccessat", "pselect6",
          "ppoll", "set_robust_list", "get_robust_list",
          "splice", "tee", "sync_file_range", "vmsplice",
          "utimensat", "epoll_pwait", "accept4", "eventfd2",
          "epoll_create1", "dup3", "pipe2", "inotify_init1",
          "preadv", "pwritev", "getrandom", "memfd_create",
          "copy_file_range", "preadv2", "pwritev2"
        ],
        "action": "SCMP_ACT_ALLOW"
      }
    ]
  }
```

```bash
  # Apply custom profile:
  docker run --security-opt seccomp=./seccomp-minimal.json myapp

  # Disable seccomp entirely (testing only, never prod):
  docker run --security-opt seccomp=unconfined myapp

  # Generate a profile from a running container (seccomp-bpf trace):
  docker run --security-opt seccomp=unconfined \
    --annotation io.kubernetes.cri-o.SeccompProfile=localhost/trace \
    myapp
  # Tools: oci-seccomp-bpf-hook, tracee
```

---

## 8.6 AppArmor — Mandatory Access Control

AppArmor is a Linux Security Module (LSM) that restricts what a process can
access by name (file paths, capabilities, network operations).

```
  SECCOMP vs AppArmor:
  ─────────────────────────────────────────────────────────────
  Seccomp   → restricts SYSCALLS (what kernel operations are allowed)
  AppArmor  → restricts RESOURCES (what files, capabilities, networks)
  Both      → complementary, run together, different attack surfaces

  Docker's default AppArmor profile (docker-default):
  - Prevents write to /proc and /sys (most paths)
  - Blocks mount operations
  - Restricts ptrace
  - Allows normal app file operations

  CHECK IF AppArmor IS ACTIVE:
  $ aa-status                          # on host
  $ docker inspect myapp | grep AppArmor
  "AppArmorProfile": "docker-default"  # active
  "AppArmorProfile": ""                # not active

  CUSTOM AppArmor PROFILE EXAMPLE:
  # /etc/apparmor.d/myapp-profile
  #include <tunables/global>
  profile myapp-profile flags=(attach_disconnected) {
    #include <abstractions/base>
    # Allow app binary to run
    /app/server ix,
    # Allow reading config
    /app/config/** r,
    # Allow writing logs
    /var/log/myapp/** w,
    # Deny everything else explicitly
    deny /etc/** w,
    deny /proc/** w,
    deny /sys/** rw,
  }

  LOAD AND APPLY:
  sudo apparmor_parser -r /etc/apparmor.d/myapp-profile
  docker run --security-opt apparmor=myapp-profile myapp
```

---

## 8.7 Rootless Docker — Run Without Root

Standard Docker daemon runs as root on the host. Rootless mode runs the
entire Docker daemon and containers as an unprivileged user.

```
  STANDARD DOCKER:
  ─────────────────────────────────────────────────────────────
  dockerd runs as: root (UID 0)
  Containers run as: root by default (UID 0 in container)
  Container escape → attacker has ROOT on host

  ROOTLESS DOCKER:
  ─────────────────────────────────────────────────────────────
  dockerd runs as: alice (UID 1000)
  Containers run as: "root" in container = alice on host
  Container escape → attacker has alice's privileges only
  alice has no sudo, no special host permissions
  → massive reduction in blast radius

  ROOTLESS ARCHITECTURE:
  ┌─────────────────────────────────────────────────────────┐
  │  User alice (UID 1000)                                  │
  │  ┌───────────────────────────────────────────────────┐  │
  │  │  rootless dockerd (running as alice)              │  │
  │  │  ┌──────────────────────────────────────────┐     │  │
  │  │  │  Container                               │     │  │
  │  │  │  "root" inside = UID 1000 outside        │     │  │
  │  │  └──────────────────────────────────────────┘     │  │
  │  └───────────────────────────────────────────────────┘  │
  │                                                         │
  │  Uses: user namespaces + newuidmap/newgidmap            │
  │  UID mapping: container UID 0 → host UID 1000          │
  └─────────────────────────────────────────────────────────┘
```

### Setting Up Rootless Docker

```bash
  # Prerequisites (Ubuntu 22.04+):
  sudo apt-get install -y uidmap dbus-user-session

  # Install rootless dockerd for current user
  dockerd-rootless-setuptool.sh install

  # Add to shell profile:
  export PATH=/usr/bin:$PATH
  export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock

  # Start rootless daemon:
  systemctl --user start docker
  systemctl --user enable docker   # start on login

  # Verify:
  docker info | grep -i rootless
  # "rootless: true"

  # Run container — "root" inside is UID 1000 on host:
  docker run --rm alpine id
  # uid=0(root) gid=0(root) ← inside container
  ps aux | grep sleep
  # alice  12345  sleep ...   ← alice on host, not root!
```

### Rootless Limitations

```
  LIMITATION                      WORKAROUND
  ─────────────────────────────────────────────────────────────
  Can't bind ports < 1024         Use ports ≥ 1024, proxy in front
  Slower overlay (fuse-overlayfs) Use native overlay with kernel ≥ 5.11
  No --network host               Use port mapping instead
  No macvlan/ipvlan networks      Use bridge networks
  Cgroup v1 resource limits       Ensure cgroup v2 on host
  Some storage drivers limited    Use overlay2 (default, fine)
```

---

## 8.8 Supply Chain Security — Signing and Verifying Images

An attacker who compromises your registry or CI can push a malicious image
under your trusted tag. Signing ensures what you pull is what you built.

### Cosign — Image Signing (Sigstore)

```bash
  # Install cosign
  brew install cosign                    # macOS
  # or download from: github.com/sigstore/cosign

  # GENERATE a signing key pair
  cosign generate-key-pair
  # Creates: cosign.key (private), cosign.pub (public)
  # Store cosign.key in a secret manager, NEVER in git

  # SIGN an image after pushing
  docker build -t myrepo/myapp:v1.2.3 .
  docker push myrepo/myapp:v1.2.3
  cosign sign --key cosign.key myrepo/myapp:v1.2.3

  # VERIFY before pulling/running (in deploy pipeline)
  cosign verify --key cosign.pub myrepo/myapp:v1.2.3
  # If verification fails: exits non-zero → CI/CD stops deploy

  # Keyless signing (uses Sigstore transparency log — no key file)
  # Works in GitHub Actions with OIDC:
  cosign sign --yes myrepo/myapp:v1.2.3
  cosign verify \
    --certificate-identity-regexp 'https://github.com/myorg/.*' \
    --certificate-oidc-issuer 'https://token.actions.githubusercontent.com' \
    myrepo/myapp:v1.2.3
```

### Digest Pinning vs Tag Pinning

```
  TAG PINNING (mutable — can be overwritten):
  FROM node:20-alpine
  → "node:20-alpine" can be overwritten in registry
  → tomorrow it could point to a different image
  → supply chain attack vector

  DIGEST PINNING (immutable — content-addressed):
  FROM node:20-alpine@sha256:a1b2c3d4e5f6...
  → SHA256 of the image manifest, computed from content
  → physically impossible to overwrite
  → if registry serves different content → hash mismatch → fails

  GET THE DIGEST:
  $ docker pull node:20-alpine
  $ docker inspect node:20-alpine | grep RepoDigests
  "node@sha256:a1b2c3d4e5f6abc..."

  OR:
  $ docker buildx imagetools inspect node:20-alpine \
    | grep Digest
  Digest: sha256:a1b2c3d4e5f6...

  USE IN DOCKERFILE:
  FROM node@sha256:a1b2c3d4e5f6abc...

  AUTOMATED DIGEST UPDATE TOOLS:
  - Renovate Bot: auto-raises PRs to update pinned digests
  - Dependabot: similar for GitHub repos
  Don't pin once and forget — digests need to be updated
  when base images get security patches.
```

### SBOM — Software Bill of Materials

```bash
  # Generate SBOM for your image (inventory of all packages)
  # Using Syft (by Anchore):
  syft myapp:latest -o spdx-json > sbom.spdx.json
  syft myapp:latest -o cyclonedx-json > sbom.cyclonedx.json

  # Attach SBOM to image as attestation (with Cosign):
  cosign attest --predicate sbom.spdx.json \
    --type spdxjson \
    --key cosign.key \
    myrepo/myapp:v1.2.3

  # Verify SBOM attestation on the other end:
  cosign verify-attestation \
    --key cosign.pub \
    --type spdxjson \
    myrepo/myapp:v1.2.3

  # WHY SBOM MATTERS:
  # Log4Shell (CVE-2021-44228): teams had no idea they were
  # running log4j. With an SBOM, one grep answers the question:
  cat sbom.spdx.json | jq '.packages[].name' | grep log4j
```

---

## 8.9 Runtime Security Monitoring — Falco

Falco is the CNCF runtime security tool that detects anomalous behaviour
inside containers in real time, at the kernel level via eBPF.

```
  HOW FALCO WORKS:
  ─────────────────────────────────────────────────────────────
  Application (in container)
       │ syscall (read, write, execve, connect...)
       ▼
  Linux Kernel
       │ eBPF probe intercepts syscall
       ▼
  Falco Engine
       │ evaluates against rules
       ▼
  ALERT if rule matches → stdout / Slack / PagerDuty / SIEM

  WHAT FALCO DETECTS:
  ┌────────────────────────────────────────────────────────────┐
  │ ✅ Shell spawned inside container                          │
  │    (docker exec bash → alert!)                            │
  │ ✅ Sensitive file read (/etc/shadow, /etc/passwd)          │
  │ ✅ Unexpected outbound connection (data exfiltration)      │
  │ ✅ Write to /etc or /proc                                  │
  │ ✅ Privilege escalation attempt (setuid)                   │
  │ ✅ Mount operation inside container                        │
  │ ✅ Loading kernel module (potential rootkit)               │
  │ ✅ Crypto mining signatures (high CPU + network pattern)   │
  └────────────────────────────────────────────────────────────┘
```

### Sample Falco Rules

```yaml
  # /etc/falco/falco_rules.yaml (custom rules)

  # Alert when a shell is spawned in a container
  - rule: Terminal shell in container
    desc: A shell was used as the entrypoint/exec point into a container
    condition: >
      spawned_process and container
      and shell_procs and proc.tty != 0
    output: >
      A shell was spawned in a container
      (user=%user.name container=%container.name
       image=%container.image.repository:%container.image.tag
       shell=%proc.name parent=%proc.pname)
    priority: WARNING

  # Alert on outbound connection to unexpected IP range
  - rule: Unexpected outbound connection
    desc: Container made connection to unexpected external IP
    condition: >
      outbound and container and not proc.name in (allowed_processes)
      and not fd.sip in (trusted_ip_ranges)
    output: >
      Unexpected outbound connection
      (container=%container.name ip=%fd.rip port=%fd.rport
       process=%proc.name)
    priority: ERROR
```

---

## 8.10 Security Scanning in CI Pipeline — Complete Flow

```
  CI PIPELINE WITH SECURITY GATES:
  ══════════════════════════════════════════════════════════════

  1. CODE COMMIT
       │
       ▼
  2. SAST (Static Application Security Testing)
     semgrep / bandit / eslint-security
     → scan source code for insecure patterns
       │
       ▼
  3. DEPENDENCY SCAN
     npm audit / pip-audit / trivy fs .
     → find vulnerable dependencies BEFORE building
       │ fail on HIGH/CRITICAL
       ▼
  4. DOCKERFILE LINT
     hadolint Dockerfile
     → catch insecure Dockerfile patterns
       │ fail on warnings
       ▼
  5. BUILD IMAGE
     docker buildx build --sbom=true --provenance=true
     → generates SBOM + build provenance attestation
       │
       ▼
  6. IMAGE CVE SCAN
     trivy image --exit-code 1 --severity CRITICAL myapp:$TAG
     → scan built image for OS + library CVEs
       │ fail on CRITICAL
       ▼
  7. SECRET SCAN
     trivy image --scanners secret myapp:$TAG
     → ensure no API keys / passwords baked in
       │ fail on any finding
       ▼
  8. SIGN IMAGE
     cosign sign --key $COSIGN_KEY myrepo/myapp:$TAG
     → cryptographically sign the verified image
       │
       ▼
  9. PUSH TO REGISTRY
     docker push myrepo/myapp:$TAG
       │
       ▼
  10. DEPLOY — VERIFY BEFORE RUNNING
      cosign verify --key $COSIGN_PUB_KEY myrepo/myapp:$TAG
      → verify signature before any deploy step
       │ fail if signature invalid or missing
       ▼
  11. RUNTIME MONITORING (Falco)
      Continuous: alert on anomalous behaviour post-deploy

  ══════════════════════════════════════════════════════════════
```

### GitHub Actions Security Pipeline Example

```yaml
  # .github/workflows/security.yml
  name: Security Pipeline

  on: [push]

  jobs:
    security:
      runs-on: ubuntu-latest
      steps:
        - uses: actions/checkout@v4

        - name: Lint Dockerfile
          uses: hadolint/hadolint-action@v3.1.0
          with:
            dockerfile: Dockerfile
            failure-threshold: warning

        - name: Scan dependencies
          run: |
            npm audit --audit-level=high
            # or: trivy fs --exit-code 1 --severity HIGH,CRITICAL .

        - name: Build image
          run: |
            docker build \
              --build-arg GIT_SHA=${{ github.sha }} \
              -t ${{ env.IMAGE }}:${{ github.sha }} .

        - name: Scan image for CVEs
          uses: aquasecurity/trivy-action@master
          with:
            image-ref: ${{ env.IMAGE }}:${{ github.sha }}
            format: sarif
            output: trivy-results.sarif
            exit-code: '1'
            severity: CRITICAL

        - name: Upload SARIF to GitHub Security tab
          uses: github/codeql-action/upload-sarif@v3
          if: always()
          with:
            sarif_file: trivy-results.sarif

        - name: Sign image (keyless)
          if: github.ref == 'refs/heads/main'
          env:
            COSIGN_EXPERIMENTAL: "true"
          run: |
            cosign sign --yes ${{ env.IMAGE }}:${{ github.sha }}

        - name: Push image
          if: github.ref == 'refs/heads/main'
          run: docker push ${{ env.IMAGE }}:${{ github.sha }}
```

---

## 8.11 Gotchas — Security Edition

### ⚠️ Gotcha #24: `--privileged` is a complete security bypass

```
  docker run --privileged myapp

  This flag:
  - Gives ALL Linux capabilities (defeats --cap-drop ALL)
  - Disables seccomp (defeats syscall filtering)
  - Disables AppArmor profile
  - Gives access to ALL host devices (/dev/*)
  - Allows mounting arbitrary filesystems

  A process in a privileged container can:
  docker run --privileged --rm -v /:/host ubuntu \
    chroot /host /bin/bash
  → FULL ROOT SHELL ON THE HOST. Game over.

  WHEN IS --privileged LEGITIMATELY NEEDED?
  - Docker-in-Docker (DinD) in CI — use Kaniko instead
  - Network tools that need raw sockets — scope with --cap-add
  - Filesystem tools — very rare, always justify

  RULE: If someone asks you to add --privileged,
        ask why. There is almost always a better-scoped solution.
```

### ⚠️ Gotcha #25: Image tags are mutable — "latest" is a security risk

```
  docker pull myrepo/myapp:latest
  # Today: pulls v1.2.3 — tested, known good
  # Next week: "latest" points to v1.3.0 — untested, possibly broken

  In production:
  - Never use :latest in docker-compose.yml or K8s manifests
  - Always pin to a specific semantic version or git SHA
  - IMAGE=myrepo/myapp:1.2.3
  - IMAGE=myrepo/myapp:$(git rev-parse --short HEAD)

  USE IMAGE DIGESTS for tamper-proof pinning (see 8.8):
  image: myrepo/myapp@sha256:a1b2c3...

  In CI, tag with BOTH semantic version AND git SHA:
  docker tag myapp:build myrepo/myapp:1.2.3
  docker tag myapp:build myrepo/myapp:1.2.3-a1b2c3d
  docker tag myapp:build myrepo/myapp:latest  # for convenience only
  # Deploy using: myrepo/myapp:1.2.3  never latest
```

### ⚠️ Gotcha #26: CVE scanners have false positives AND false negatives

```
  FALSE POSITIVES: CVE reported but not exploitable in your context
  - OS package installed but feature using CVE not compiled in
  - CVE in a code path your app never calls
  - CVE requires local access (you don't give local access)

  FALSE NEGATIVES: CVE exists but scanner missed it
  - New CVE not yet in scanner's database
  - Custom compiled software scanner doesn't recognise
  - CVE in application logic, not in packages

  PRACTICAL APPROACH:
  1. Treat CRITICAL as must-fix before deploy
  2. Treat HIGH as must-fix within 72 hours of discovery
  3. Review MEDIUM monthly
  4. Assess false positives — use trivy .trivyignore to suppress
     documented, reviewed false positives
  5. Layer with DAST (dynamic scanning) and runtime monitoring
     Scanner ≠ complete security. It's one layer.

  .trivyignore:
  # CVE-2023-1234: not exploitable, feature not compiled in
  CVE-2023-1234
```

---

## 8.12 Production FAQ — Batch 8

---

**Q: Someone on our team ran `docker exec myapp bash` in production. Is that a security issue?**

```
A: Yes — and it's worth treating as a security event to investigate.

  WHY IT'S CONCERNING:
  - Manual exec = uncontrolled, unaudited change to a running container
  - What was done inside? Was anything modified? Was data exfiltrated?
  - Bypasses your change management / deployment process
  - If an attacker also has access: impossible to tell exec apart
    from legitimate admin use without audit logs

  IMMEDIATE STEPS:
  1. Check docker logs myapp for anything unusual after the exec
  2. Run docker diff myapp to see if files were modified
  3. If suspicious: docker commit myapp forensics-snapshot
     Preserve the container state before it's replaced
  4. Replace the container from a known-good image (clean deploy)

  PREVENTIVE CONTROLS:
  1. Falco runtime monitoring alerts on any exec/shell spawn
  2. Enforce the principle: no exec in prod, ever
     Use structured logging, not manual investigation
  3. Add --security-opt no-new-privileges:true
     (won't stop exec but limits what they can do once inside)
  4. Use read-only containers — even if someone execs in,
     they can't modify files or install tools

  LOGGING EXECS — audit trail:
  Falco or sysdig can log every exec event:
  user=alice container=myapp command="bash"
  Forward to SIEM (Splunk, Elastic, CloudWatch Logs)
```

---

**Q: We found a CRITICAL CVE in our base image. How do we respond fast?**

```
A: This is your incident response runbook for CVEs:

  T+0: DETECTION (automated scanner alert)
  ─────────────────────────────────────────────────────────
  - Identify: which images are affected?
    trivy image --severity CRITICAL myapp:latest
  - Scope: is the vulnerable package actually used?
    Check CVE details — is the affected code path reachable?
  - Assess: is there a fix available?
    trivy output shows "Fixed Version"

  T+1h: PATCH
  ─────────────────────────────────────────────────────────
  - Update FROM line to patched base image version
  - Rebuild image: docker build --no-cache (bypass stale cache)
  - Rescan rebuilt image to confirm CVE is gone
  - Run full test suite

  T+2h: DEPLOY
  ─────────────────────────────────────────────────────────
  - Push patched image with new tag
  - Deploy via normal pipeline (not manual docker pull!)
  - Verify running containers are on new image:
    docker inspect myapp | grep Image
  - Check docker images for old image digest mismatch

  T+4h: VERIFY AND CLOSE
  ─────────────────────────────────────────────────────────
  - Rescan running containers (ECR Inspector, Trivy)
  - Document: CVE, affected images, fix applied, deploy time
  - Update .trivyignore if any related false positives

  AUTOMATE THIS:
  Set up ECR/GCR scanning + SNS/PagerDuty alert on new CRITICAL.
  Have a "CVE fast-track" CI pipeline that skips slow tests
  but keeps security scan gates. Target: patch in < 4 hours.
```

---

**Q: How do we enforce security policies across all containers in production?**

```
A: Policy-as-code — enforce at admission, not at runtime.

  FOR DOCKER STANDALONE / COMPOSE:
  ─────────────────────────────────────────────────────────
  Use a wrapper script or Makefile that always passes
  hardened flags. Codify in a team convention document.
  CI pipeline: hadolint + trivy before any image is pushed.

  FOR KUBERNETES (the right long-term answer):
  ─────────────────────────────────────────────────────────
  1. Pod Security Standards (built-in since K8s 1.25)
     Enforce "restricted" profile on namespaces:
     - Disallows privileged containers
     - Requires non-root user
     - Drops all capabilities
     - Requires read-only root filesystem

  2. OPA Gatekeeper / Kyverno (policy engine)
     Write custom policies:
     - "All images must come from myrepo.internal"
     - "All containers must have CPU/memory limits"
     - "No :latest tags allowed"
     - "Images must have a valid Cosign signature"
     Policies evaluated at admission — violating workloads
     are REJECTED before they even start.

  3. Image admission controller (Connaisseur, Portieris)
     Verifies Cosign signatures at deploy time.
     If image isn't signed by your key: admission denied.

  QUICK WIN FOR ANY SETUP:
  Add to /etc/docker/daemon.json on every host:
  {
    "userns-remap": "default",      # user namespace remapping
    "no-new-privileges": true,      # daemon-wide no-new-privs
    "seccomp-profile": "/etc/docker/seccomp.json"
  }
  Forces every container to inherit these security defaults.
```

---

*End of Batch 8.*

---

# BATCH 9
## Docker in CI/CD Pipelines — Build, Test, Scan, Push, Deploy

> **Goal:** Understand the complete CI/CD pipeline for containerised applications.
> Learn how to structure pipelines in GitHub Actions and GitLab CI, cache builds
> intelligently, promote images across environments, and deploy with zero downtime.
> This is where all previous batches come together into a production workflow.

---

## 9.1 The CI/CD Pipeline Mental Model

```
  DEVELOPER WORKFLOW:

  Local dev (Compose)
       │ git push
       ▼
  ┌────────────────────────────────────────────────────────────┐
  │                     CI PIPELINE                            │
  │                                                            │
  │  Code quality gates        Image quality gates             │
  │  ───────────────           ────────────────────            │
  │  lint & format    ──►     build image                      │
  │  unit tests       ──►     CVE scan                         │
  │  SAST scan        ──►     secrets scan                     │
  │  dep audit        ──►     sign image                       │
  │                   ──►     push to registry                 │
  │                                    │                       │
  └────────────────────────────────────┼───────────────────────┘
                                       │ image tag / digest
  ┌────────────────────────────────────▼───────────────────────┐
  │                     CD PIPELINE                            │
  │                                                            │
  │  staging deploy ──► integration tests ──► prod deploy      │
  │                                                            │
  │  Trigger:                                                  │
  │  push to main   → staging auto-deploy                      │
  │  manual gate    → production deploy (or tag v*)            │
  └────────────────────────────────────────────────────────────┘

  KEY PRINCIPLE: The IMAGE is the deployable artifact.
  Build once, promote through environments.
  Never rebuild the image for each environment.
  Environment differences = env vars + config, not different images.
```

### Image Promotion Pattern

```
  ❌ WRONG — rebuild per environment:
  build image → deploy to staging (image A)
  build image → deploy to production (image B)
  "Same code" but different builds = untested artifact in prod

  ✅ RIGHT — build once, promote:
  build image → tag as :sha-a1b2c3d (immutable)
      │
      ├── deploy to staging (same image sha-a1b2c3d)
      │   run integration tests against staging
      │   tests pass ✅
      │
      └── promote to production (SAME image sha-a1b2c3d)
          tag as :v1.2.3 in registry
          deploy to prod

  The image that passed staging tests IS the prod image.
  No rebuild risk. Full audit trail from commit → image → env.
```

---

## 9.2 Registry Strategy — Tags, Environments, Retention

```
  TAG STRATEGY FOR A PRODUCTION REPO:
  ─────────────────────────────────────────────────────────────
  myrepo/myapp:main-a1b2c3d      ← every main branch build
                                    immutable, unique per commit

  myrepo/myapp:pr-123-d4e5f6g    ← pull request builds
                                    for review / ephemeral envs

  myrepo/myapp:v1.2.3            ← semantic version (release)
                                    promoted from sha tag

  myrepo/myapp:v1.2.3-sha-a1b2c3d ← version + sha (best traceability)

  myrepo/myapp:latest            ← alias to latest main (convenience)
                                    NEVER deploy using this tag

  myrepo/myapp:staging           ← mutable pointer to current staging
  myrepo/myapp:production        ← mutable pointer to current prod

  REGISTRY RETENTION POLICY:
  ─────────────────────────────────────────────────────────────
  sha tags (main-*):   keep 90 days or last 50 builds
  pr tags (pr-*):      keep 7 days after PR close
  version tags (v*):   keep forever (compliance)
  staging/production:  keep forever (rollback reference)

  Untagged/orphaned:   prune weekly (saves storage costs)

  AWS ECR lifecycle policy example:
  {
    "rules": [
      {
        "rulePriority": 1,
        "description": "Keep last 30 untagged",
        "selection": {
          "tagStatus": "untagged",
          "countType": "imageCountMoreThan",
          "countNumber": 30
        },
        "action": {"type": "expire"}
      }
    ]
  }
```

---

## 9.3 GitHub Actions — Complete Docker Pipeline

### Minimal Working Pipeline

```yaml
  # .github/workflows/docker.yml
  name: Docker Build & Push

  on:
    push:
      branches: [main]
    pull_request:
      branches: [main]

  env:
    REGISTRY: ghcr.io
    IMAGE_NAME: ${{ github.repository }}   # myorg/myapp

  jobs:
    build:
      runs-on: ubuntu-latest
      permissions:
        contents: read
        packages: write          # push to GHCR
        id-token: write          # cosign keyless signing

      steps:
        - name: Checkout
          uses: actions/checkout@v4

        - name: Set up Docker Buildx
          uses: docker/setup-buildx-action@v3
          # Buildx = BuildKit-powered, supports cache export,
          # multi-platform, inline cache

        - name: Log in to registry
          uses: docker/login-action@v3
          with:
            registry: ${{ env.REGISTRY }}
            username: ${{ github.actor }}
            password: ${{ secrets.GITHUB_TOKEN }}

        - name: Extract metadata (tags, labels)
          id: meta
          uses: docker/metadata-action@v5
          with:
            images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
            tags: |
              type=sha,prefix=main-,format=short
              type=ref,event=pr,prefix=pr-
              type=semver,pattern={{version}}
              type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

        - name: Build and push
          uses: docker/build-push-action@v5
          with:
            context: .
            push: ${{ github.event_name != 'pull_request' }}
            tags: ${{ steps.meta.outputs.tags }}
            labels: ${{ steps.meta.outputs.labels }}
            cache-from: type=gha            # GitHub Actions cache
            cache-to: type=gha,mode=max     # save all layers to cache
            build-args: |
              GIT_SHA=${{ github.sha }}
              BUILD_DATE=${{ github.event.head_commit.timestamp }}
```

### Production-Grade Pipeline — Full Gates

```yaml
  # .github/workflows/ci-cd.yml
  name: CI/CD Pipeline

  on:
    push:
      branches: [main]
    pull_request:
      branches: [main]
    release:
      types: [published]

  env:
    REGISTRY: ghcr.io
    IMAGE: ghcr.io/${{ github.repository }}

  jobs:
    # ── Job 1: Code Quality ──────────────────────────────────
    lint-test:
      runs-on: ubuntu-latest
      steps:
        - uses: actions/checkout@v4

        - name: Set up Node.js
          uses: actions/setup-node@v4
          with:
            node-version: '20'
            cache: 'npm'

        - name: Install dependencies
          run: npm ci

        - name: Lint
          run: npm run lint

        - name: Unit tests
          run: npm test -- --coverage

        - name: Upload coverage
          uses: codecov/codecov-action@v4

    # ── Job 2: Dockerfile & Dependency Security ──────────────
    security-scan-code:
      runs-on: ubuntu-latest
      steps:
        - uses: actions/checkout@v4

        - name: Lint Dockerfile
          uses: hadolint/hadolint-action@v3.1.0
          with:
            dockerfile: Dockerfile
            failure-threshold: warning

        - name: Audit npm dependencies
          run: npm audit --audit-level=high

    # ── Job 3: Build Image ───────────────────────────────────
    build:
      needs: [lint-test, security-scan-code]
      runs-on: ubuntu-latest
      permissions:
        contents: read
        packages: write
        id-token: write
      outputs:
        image-digest: ${{ steps.build.outputs.digest }}
        image-tag: ${{ steps.meta.outputs.version }}

      steps:
        - uses: actions/checkout@v4

        - name: Set up QEMU (multi-platform)
          uses: docker/setup-qemu-action@v3

        - name: Set up Docker Buildx
          uses: docker/setup-buildx-action@v3

        - name: Log in to GHCR
          uses: docker/login-action@v3
          with:
            registry: ghcr.io
            username: ${{ github.actor }}
            password: ${{ secrets.GITHUB_TOKEN }}

        - name: Image metadata
          id: meta
          uses: docker/metadata-action@v5
          with:
            images: ${{ env.IMAGE }}
            tags: |
              type=sha,prefix=sha-,format=short
              type=ref,event=pr
              type=semver,pattern={{version}},event=tag
              type=semver,pattern={{major}}.{{minor}},event=tag
              type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

        - name: Build and push
          id: build
          uses: docker/build-push-action@v5
          with:
            context: .
            platforms: linux/amd64,linux/arm64   # multi-arch
            push: true
            tags: ${{ steps.meta.outputs.tags }}
            labels: ${{ steps.meta.outputs.labels }}
            cache-from: type=gha
            cache-to: type=gha,mode=max
            provenance: true      # SLSA provenance attestation
            sbom: true            # SBOM attestation

    # ── Job 4: Image Security Scan ───────────────────────────
    scan-image:
      needs: build
      runs-on: ubuntu-latest
      permissions:
        security-events: write
      steps:
        - name: Run Trivy vulnerability scan
          uses: aquasecurity/trivy-action@master
          with:
            image-ref: ${{ env.IMAGE }}@${{ needs.build.outputs.image-digest }}
            format: sarif
            output: trivy-results.sarif
            exit-code: '1'
            severity: CRITICAL
            scanners: vuln,secret

        - name: Upload results to GitHub Security
          if: always()
          uses: github/codeql-action/upload-sarif@v3
          with:
            sarif_file: trivy-results.sarif

    # ── Job 5: Sign Image ────────────────────────────────────
    sign-image:
      needs: [build, scan-image]
      runs-on: ubuntu-latest
      if: github.ref == 'refs/heads/main' || github.event_name == 'release'
      permissions:
        packages: write
        id-token: write
      steps:
        - name: Install Cosign
          uses: sigstore/cosign-installer@v3

        - name: Sign image (keyless via OIDC)
          env:
            COSIGN_EXPERIMENTAL: "true"
          run: |
            cosign sign --yes \
              ${{ env.IMAGE }}@${{ needs.build.outputs.image-digest }}

    # ── Job 6: Deploy to Staging ─────────────────────────────
    deploy-staging:
      needs: [sign-image]
      runs-on: ubuntu-latest
      if: github.ref == 'refs/heads/main'
      environment: staging
      steps:
        - name: Verify image signature
          run: |
            cosign verify \
              --certificate-identity-regexp \
                'https://github.com/${{ github.repository }}' \
              --certificate-oidc-issuer \
                'https://token.actions.githubusercontent.com' \
              ${{ env.IMAGE }}@${{ needs.build.outputs.image-digest }}

        - name: Deploy to staging
          run: |
            # SSH to staging server and pull new image
            ssh deploy@staging.internal \
              "IMAGE_TAG=${{ needs.build.outputs.image-tag }} \
               docker compose -f /opt/myapp/docker-compose.yml \
                              -f /opt/myapp/docker-compose.prod.yml \
               pull && \
               docker compose up -d --remove-orphans"

    # ── Job 7: Integration Tests ─────────────────────────────
    integration-tests:
      needs: deploy-staging
      runs-on: ubuntu-latest
      steps:
        - uses: actions/checkout@v4
        - name: Run integration test suite
          run: |
            npm run test:integration
          env:
            API_BASE_URL: https://staging.myapp.com

    # ── Job 8: Deploy to Production ──────────────────────────
    deploy-production:
      needs: integration-tests
      runs-on: ubuntu-latest
      if: github.event_name == 'release'
      environment:
        name: production
        url: https://myapp.com
      steps:
        - name: Deploy to production
          run: |
            ssh deploy@prod.internal \
              "IMAGE_DIGEST=${{ needs.build.outputs.image-digest }} \
               docker compose -f /opt/myapp/docker-compose.yml \
                              -f /opt/myapp/docker-compose.prod.yml \
               pull && docker compose up -d --remove-orphans"
```

---

## 9.4 GitLab CI — Complete Docker Pipeline

```yaml
  # .gitlab-ci.yml
  stages:
    - test
    - build
    - scan
    - deploy-staging
    - integration-test
    - deploy-production

  variables:
    DOCKER_DRIVER: overlay2
    DOCKER_TLS_CERTDIR: "/certs"
    IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA

  # ── Stage: Test ──────────────────────────────────────────
  lint-test:
    stage: test
    image: node:20-alpine
    cache:
      key:
        files: [package-lock.json]
      paths: [node_modules/]
    script:
      - npm ci
      - npm run lint
      - npm test -- --coverage
    artifacts:
      reports:
        coverage_report:
          coverage_format: cobertura
          path: coverage/cobertura-coverage.xml

  dockerfile-lint:
    stage: test
    image: hadolint/hadolint:latest-alpine
    script:
      - hadolint Dockerfile --failure-threshold warning

  # ── Stage: Build ─────────────────────────────────────────
  build-image:
    stage: build
    image: docker:24
    services:
      - docker:24-dind        # Docker-in-Docker
    variables:
      DOCKER_BUILDKIT: "1"
    before_script:
      - echo "$CI_REGISTRY_PASSWORD" | docker login $CI_REGISTRY
          -u $CI_REGISTRY_USER --password-stdin
    script:
      # Use registry cache to speed up builds
      - docker buildx create --use
      - docker buildx build
          --platform linux/amd64,linux/arm64
          --cache-from type=registry,ref=$CI_REGISTRY_IMAGE:buildcache
          --cache-to type=registry,ref=$CI_REGISTRY_IMAGE:buildcache,mode=max
          --build-arg GIT_SHA=$CI_COMMIT_SHORT_SHA
          --build-arg BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ)
          --tag $IMAGE
          --push
          .
    only:
      - main
      - merge_requests

  # ── Stage: Scan ──────────────────────────────────────────
  trivy-scan:
    stage: scan
    image: aquasec/trivy:latest
    variables:
      TRIVY_NO_PROGRESS: "true"
      TRIVY_CACHE_DIR: ".trivycache"
    cache:
      paths: [.trivycache/]
    script:
      - trivy image
          --exit-code 1
          --severity CRITICAL
          --format template
          --template "@/contrib/gitlab.tpl"
          --output gl-container-scanning-report.json
          $IMAGE
    artifacts:
      reports:
        container_scanning: gl-container-scanning-report.json
    allow_failure: false
    needs: [build-image]

  # ── Stage: Deploy Staging ─────────────────────────────────
  deploy-staging:
    stage: deploy-staging
    image: alpine:3.19
    before_script:
      - apk add --no-cache openssh-client
      - eval $(ssh-agent -s)
      - echo "$STAGING_SSH_KEY" | ssh-add -
    script:
      - ssh -o StrictHostKeyChecking=no deploy@$STAGING_HOST
          "export IMAGE_TAG=$CI_COMMIT_SHORT_SHA &&
           cd /opt/myapp &&
           docker compose -f docker-compose.yml
                          -f docker-compose.prod.yml
           pull && docker compose up -d --remove-orphans"
    environment:
      name: staging
      url: https://staging.myapp.com
    needs: [trivy-scan]
    only: [main]

  # ── Stage: Integration Tests ─────────────────────────────
  integration-tests:
    stage: integration-test
    image: node:20-alpine
    script:
      - npm ci
      - npm run test:integration
    variables:
      API_BASE_URL: https://staging.myapp.com
    needs: [deploy-staging]
    only: [main]

  # ── Stage: Deploy Production ─────────────────────────────
  deploy-production:
    stage: deploy-production
    image: alpine:3.19
    before_script:
      - apk add --no-cache openssh-client
      - eval $(ssh-agent -s)
      - echo "$PROD_SSH_KEY" | ssh-add -
    script:
      - ssh deploy@$PROD_HOST
          "export IMAGE_TAG=$CI_COMMIT_SHORT_SHA &&
           cd /opt/myapp &&
           docker compose -f docker-compose.yml
                          -f docker-compose.prod.yml
           pull && docker compose up -d --remove-orphans"
    environment:
      name: production
      url: https://myapp.com
    when: manual           # require human approval
    needs: [integration-tests]
    only: [main]
```

---

## 9.5 Build Caching Strategies — Speed Up Every Build

```
  CACHING OPTIONS FOR BUILDKIT:
  ══════════════════════════════════════════════════════════════

  TYPE              HOW IT WORKS            BEST FOR
  ──────────────────────────────────────────────────────────
  type=gha          GitHub Actions cache    GitHub Actions only
                    (Actions Cache API)     Fast, free up to 10GB

  type=registry     Store cache as image    Any CI, any registry
                    in container registry   Persists across runners

  type=local        Cache on local disk     Self-hosted runners
                    path=/path/to/cache     with persistent storage

  type=inline       Cache embedded IN the   Simple setups, but
                    image layers            pollutes the image

  type=s3           Cache in AWS S3 bucket  Large teams, big caches
                    (or compatible store)   Parallel runners share

  ══════════════════════════════════════════════════════════════
```

### Registry Cache Pattern (Most Portable)

```bash
  # Build with registry cache (works in any CI):

  docker buildx build \
    --cache-from type=registry,ref=myrepo/myapp:buildcache \
    --cache-to   type=registry,ref=myrepo/myapp:buildcache,mode=max \
    --tag myrepo/myapp:$GIT_SHA \
    --push \
    .

  # mode=max caches ALL intermediate layers, not just the final image
  # mode=min caches only layers that end up in the final image
  # Always use mode=max for CI speedup

  # In GitHub Actions (type=gha — simplest):
  - uses: docker/build-push-action@v5
    with:
      cache-from: type=gha
      cache-to: type=gha,mode=max
```

### Cache Warmup Pattern for Matrix Builds

```yaml
  # When building for multiple platforms or services:

  jobs:
    warm-cache:
      runs-on: ubuntu-latest
      steps:
        - name: Pre-pull base images
          run: |
            docker pull node:20-alpine || true
            docker pull postgres:16-alpine || true
            # "|| true" prevents failure if offline

    build-services:
      needs: warm-cache
      strategy:
        matrix:
          service: [api, worker, scheduler]
      steps:
        - name: Build ${{ matrix.service }}
          uses: docker/build-push-action@v5
          with:
            context: ./services/${{ matrix.service }}
            cache-from: |
              type=gha,scope=${{ matrix.service }}
              type=gha,scope=base           # shared base cache
            cache-to: type=gha,scope=${{ matrix.service }},mode=max
```

---

## 9.6 Docker-in-Docker (DinD) vs Kaniko vs Buildah

A critical CI decision: how do you BUILD Docker images inside a CI runner
that is itself a container (e.g., Kubernetes-based CI)?

```
  THE PROBLEM:
  ─────────────────────────────────────────────────────────────
  CI runner runs as a container (Kubernetes pod).
  You want to run: docker build inside that container.
  docker build needs a Docker daemon.
  Docker daemon needs root and /dev access.

  OPTIONS:
  ══════════════════════════════════════════════════════════════

  OPTION 1: Docker-in-Docker (DinD)
  ─────────────────────────────────────────────────────────────
  Run a docker daemon INSIDE the CI container.
  Requires: --privileged mode.
  + Works identically to docker build
  + Supports all Docker features
  - Requires privileged mode (security risk — Batch 8!)
  - Nested virtualisation overhead
  - Cache isolation (each build is cold)

  GitLab CI with DinD:
  services:
    - docker:24-dind
  variables:
    DOCKER_HOST: tcp://docker:2376
    DOCKER_TLS_CERTDIR: "/certs"

  OPTION 2: Mount docker.sock (socket passthrough)
  ─────────────────────────────────────────────────────────────
  Pass host's docker socket into CI container.
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock
  + Faster than DinD (uses host daemon, hot cache)
  - Massive security risk (docker.sock = root on host)
  - CI job can escape to host, kill other containers
  - Shared daemon state (image/network conflicts)
  AVOID in multi-tenant or shared CI environments.

  OPTION 3: Kaniko (Recommended for K8s CI)
  ─────────────────────────────────────────────────────────────
  Builds Docker images from a Dockerfile WITHOUT a daemon.
  Runs completely unprivileged inside a container.
  + No daemon, no root, no privileged mode
  + Native Kubernetes-friendly
  + Supports registry cache
  - Slower than native docker build (no BuildKit parallel)
  - Some Dockerfile features not supported (e.g., --mount=cache)

  In Kubernetes CI (Tekton, Argo Workflows, GitLab K8s runner):
  image: gcr.io/kaniko-project/executor:latest
  args:
    - --context=git://github.com/myorg/myapp
    - --destination=myrepo/myapp:latest
    - --cache=true
    - --cache-repo=myrepo/myapp-cache

  OPTION 4: Buildah
  ─────────────────────────────────────────────────────────────
  Daemonless image building. Red Hat / Podman ecosystem.
  + Rootless by default
  + Full OCI compatibility
  + More flexible than Kaniko (rootless user namespaces)
  - Less ecosystem adoption than Kaniko

  SUMMARY TABLE:
  ┌──────────────┬───────────┬──────────┬──────────┬─────────┐
  │              │ DinD      │ sock     │ Kaniko   │ Buildah │
  ├──────────────┼───────────┼──────────┼──────────┼─────────┤
  │ Privileged?  │ ✅ Yes    │ ✅ Yes   │ ✗ No     │ ✗ No   │
  │ Daemon req?  │ ✅ Yes    │ ✅ Yes   │ ✗ No     │ ✗ No   │
  │ Speed        │ Medium    │ Fast     │ Slow     │ Medium  │
  │ K8s-native   │ OK        │ Avoid    │ ✅ Best  │ ✅ Good │
  │ BuildKit     │ ✅ Yes    │ ✅ Yes   │ ✗ No     │ ✗ No   │
  └──────────────┴───────────┴──────────┴──────────┴─────────┘
```

---

## 9.7 Deployment Patterns

### Pattern 1 — SSH + Docker Compose Pull (Simple, Works Everywhere)

```bash
  # In CI deploy step:
  ssh deploy@prod-server \
    "cd /opt/myapp && \
     export IMAGE_TAG=${GIT_SHA} && \
     docker compose -f docker-compose.yml \
                    -f docker-compose.prod.yml \
     pull api && \
     docker compose up -d --no-deps --remove-orphans api"

  # --no-deps: only restart 'api', not its dependencies (db stays up)
  # --remove-orphans: cleans up services removed from compose file
  # pull first, then up = minimal downtime window

  # ROLLBACK:
  ssh deploy@prod-server \
    "cd /opt/myapp && \
     export IMAGE_TAG=${PREVIOUS_SHA} && \
     docker compose -f docker-compose.yml \
                    -f docker-compose.prod.yml \
     up -d --no-deps api"
```

### Pattern 2 — Watchtower (Auto-Update from Registry)

```yaml
  # Watchtower: polls registry and auto-updates containers
  # Best for: non-critical services, internal tools, staging

  services:
    myapp:
      image: myrepo/myapp:latest
      labels:
        - "com.centurylinklabs.watchtower.enable=true"

    watchtower:
      image: containrrr/watchtower
      volumes:
        - /var/run/docker.sock:/var/run/docker.sock
        - /root/.docker/config.json:/config.json:ro
      command: --interval 300 --cleanup
      # polls every 300 seconds, removes old images after update

  # ⚠️ WARNING: Watchtower + :latest = auto-deploys unreviewed images
  # Use only with digest-pinned tags or in low-risk environments
  # Never use in production without manual approval gates
```

### Pattern 3 — Blue/Green Deployment (Zero Downtime)

```bash
  #!/bin/bash
  # blue-green-deploy.sh

  IMAGE_TAG=$1
  NGINX_CONFIG=/etc/nginx/conf.d/myapp.conf

  # Determine current active colour
  if docker ps | grep -q "myapp-blue"; then
    CURRENT=blue
    NEW=green
    NEW_PORT=3001
    OLD_PORT=3000
  else
    CURRENT=green
    NEW=blue
    NEW_PORT=3000
    OLD_PORT=3001
  fi

  echo "Deploying to $NEW (port $NEW_PORT)..."

  # 1. Start new (green/blue) container
  docker run -d \
    --name myapp-$NEW \
    --network myapp-net \
    -p 127.0.0.1:$NEW_PORT:3000 \
    myrepo/myapp:$IMAGE_TAG

  # 2. Wait for healthcheck to pass
  until docker inspect myapp-$NEW \
    | grep -q '"Status": "healthy"'; do
    sleep 2
    echo "Waiting for $NEW to be healthy..."
  done

  # 3. Switch nginx upstream to new container
  sed -i "s/127.0.0.1:$OLD_PORT/127.0.0.1:$NEW_PORT/" \
    $NGINX_CONFIG
  nginx -s reload

  echo "Traffic switched to $NEW"

  # 4. Wait 30s, verify metrics/errors before removing old
  sleep 30

  # 5. Remove old container
  docker rm -f myapp-$CURRENT
  echo "Removed $CURRENT. Deploy complete."
```

### Pattern 4 — Docker Swarm Rolling Update

```bash
  # Initialize swarm (single node)
  docker swarm init

  # Deploy stack from compose file
  docker stack deploy -c docker-compose.yml myapp

  # Rolling update — Swarm handles one-by-one replacement
  docker service update \
    --image myrepo/myapp:v1.2.3 \
    --update-parallelism 1 \
    --update-delay 15s \
    --update-failure-action rollback \
    --health-cmd "wget -qO- http://localhost:3000/health || exit 1" \
    --health-interval 10s \
    myapp_api

  # Swarm replaces containers one at a time:
  # Starts new container → waits for healthy → removes old → repeat
  # If new container fails health check → automatic rollback

  # Check rollout status:
  docker service ps myapp_api

  # Manual rollback if needed:
  docker service rollback myapp_api
```

---

## 9.8 Environment-Specific Configuration Pattern

```
  RULE: Same image, different configuration per environment.

  CONFIG HIERARCHY (lowest → highest priority):
  ────────────────────────────────────────────────────────
  1. Baked defaults in Dockerfile (ENV)
  2. docker-compose.yml environment: block
  3. docker-compose.override.yml or docker-compose.prod.yml
  4. env_file: (loaded from host file)
  5. Runtime -e flags or environment block
  6. Secrets manager (Vault, AWS SSM) fetched at startup

  PRACTICAL PATTERN:
  ┌─────────────────────────────────────────────────────────┐
  │  Environment  │  Config Source                          │
  ├───────────────┼─────────────────────────────────────────┤
  │  Dev          │  .env file + compose.override.yml       │
  │  CI           │  CI environment variables               │
  │  Staging      │  /etc/myapp/staging.env on server       │
  │               │  + docker-compose.prod.yml              │
  │  Production   │  AWS SSM Parameter Store or Vault       │
  │               │  mounted via secrets at startup         │
  └─────────────────────────────────────────────────────────┘

  ANTI-PATTERN TO AVOID:
  Building separate images per environment:
  docker build --build-arg ENV=staging .   ← staging image
  docker build --build-arg ENV=prod .      ← prod image

  These are different artifacts. The staging image was never
  tested in prod context. You're deploying an untested build.
```

---

## 9.9 Observability — Logs, Metrics, Traces in CI/CD

```yaml
  # Structured logging from containers — JSON output
  services:
    api:
      logging:
        driver: json-file
        options:
          max-size: "20m"
          max-file: "5"
          tag: "{{.Name}}/{{.ID}}"   # include container name in log

  # Forward to centralised logging (Loki, CloudWatch, Datadog):
  # Option 1: Docker logging driver
  logging:
    driver: awslogs
    options:
      awslogs-region: us-east-1
      awslogs-group: /myapp/production
      awslogs-stream: api

  # Option 2: Fluentd/Fluent Bit sidecar pattern
  services:
    api:
      labels:
        - "logging=true"
    fluent-bit:
      image: fluent/fluent-bit:latest
      volumes:
        - /var/lib/docker/containers:/var/lib/docker/containers:ro
        - ./fluent-bit.conf:/fluent-bit/etc/fluent-bit.conf
```

```
  HEALTH ENDPOINT PATTERN — what CI/CD verifies:
  ─────────────────────────────────────────────────────────────
  GET /health → 200 OK
  {
    "status": "healthy",
    "version": "1.2.3",
    "git_sha": "a1b2c3d",
    "db": "connected",
    "redis": "connected",
    "uptime_seconds": 3600
  }

  GET /health/ready → 200 = ready to serve traffic
                   → 503 = still initialising (don't route yet)

  GET /health/live  → 200 = process is alive
                   → 500 = deadlocked, needs restart

  CI deploy step waits for /health/ready before switching traffic.
  Load balancer uses /health/live for liveness checks.
  These are the same patterns used in Kubernetes readiness/liveness probes.
```

---

## 9.10 Gotchas — CI/CD Edition

### ⚠️ Gotcha #27: Cold cache on every CI run

```
  Ephemeral CI runners (GitHub-hosted, GitLab SaaS) start fresh.
  No local Docker layer cache.
  Every build pulls base image + reinstalls all dependencies.
  A 2-minute build locally becomes 15 minutes in CI.

  FIX — choose a cache strategy from 9.5:
  GitHub Actions: cache-from/cache-to: type=gha   (automatic)
  GitLab:         registry cache with buildcache tag
  Self-hosted:    persistent runner with local cache volume

  MEASURE YOUR CACHE HIT RATE:
  In buildx output, "CACHED" lines = cache hits.
  If most steps say "CACHED" → your strategy is working.
  If most say the command text → cold build, fix your strategy.
```

### ⚠️ Gotcha #28: Platform mismatch — ARM vs AMD64

```
  SCENARIO:
  Developer runs Apple Silicon Mac (linux/arm64).
  docker build -t myapp .
  docker push myapp:latest
  Deploys to AWS EC2 (linux/amd64).
  Container fails to start: "exec format error"

  CAUSE: ARM image can't run on x86 host. Different ISA.

  FIX — always build for the target platform:
  docker buildx build \
    --platform linux/amd64 \   ← explicit target
    -t myapp:latest .

  FOR MULTI-ARCH (supports both):
  docker buildx build \
    --platform linux/amd64,linux/arm64 \
    -t myapp:latest \
    --push .
  Buildx creates a manifest list → correct arch pulled per host.

  IN CI: Always specify --platform linux/amd64 at minimum.
  Add linux/arm64 if you deploy to Graviton (AWS) or Apple Silicon.

  CHECK IMAGE ARCH:
  docker inspect myapp:latest | grep Architecture
  docker manifest inspect myrepo/myapp:latest   # multi-arch manifest
```

### ⚠️ Gotcha #29: Secrets in CI environment variables appear in logs

```
  CI log output:
  Run docker build --build-arg API_KEY=${{ secrets.API_KEY }} .
  → GitHub masks this: ***
  BUT:
  If your Dockerfile or app prints env vars (debug mode),
  the secret appears in container logs which are captured by CI.

  ALSO:
  docker history myimage --no-trunc
  → shows --build-arg values in layer history

  FIX:
  Use BuildKit secret mounts (Batch 6):
  RUN --mount=type=secret,id=api_key \
      cat /run/secrets/api_key | curl -H "Authorization: $(cat)" ...

  Pass at runtime, not build time.
  Never log environment variables in your application code.
  Add CI log masking rules for any sensitive patterns.
```

### ⚠️ Gotcha #30: `docker compose up` in CI doesn't clean up on failure

```
  SCENARIO:
  CI pipeline runs integration tests with docker compose up.
  Tests fail. Pipeline exits non-zero.
  docker compose down is never reached.
  Networks and containers linger on the runner.
  Next run fails: "network already exists" / port conflicts.

  FIX — always use a trap or finally block:

  # Shell trap (bash):
  trap "docker compose down --volumes" EXIT
  docker compose -f docker-compose.test.yml up --abort-on-container-exit

  # GitHub Actions:
  - name: Run tests
    run: docker compose -f docker-compose.test.yml up
           --abort-on-container-exit --exit-code-from tests
  - name: Cleanup (always runs)
    if: always()
    run: docker compose -f docker-compose.test.yml down --volumes

  # GitLab CI:
  after_script:
    - docker compose -f docker-compose.test.yml down --volumes
  # after_script always runs, even on job failure
```

---

## 9.11 Production FAQ — Batch 9

---

**Q: How do we handle database migrations in CI/CD without downtime?**

```
A: Migrations are one of the hardest zero-downtime problems.
   The right pattern depends on your migration type.

  SAFE MIGRATION PATTERN (expand/contract):
  ─────────────────────────────────────────────────────────
  Phase 1 — EXPAND (backward-compatible change):
  Add new column (nullable, with default).
  Both old and new app code can run simultaneously.
  Deploy: run migration → deploy new app version.

  Phase 2 — MIGRATE DATA:
  Backfill existing rows if needed.
  Run as a background job, not in the migration itself.

  Phase 3 — CONTRACT (remove old):
  Remove old column/table only after new code is stable.
  Deploy: run cleanup migration.

  IN CI/CD PIPELINE:
  ─────────────────────────────────────────────────────────
  # Use the migrate service_completed_successfully pattern
  # from Batch 7 (7.3):

  docker run --rm \
    --network prod-network \
    -e DATABASE_URL=$DATABASE_URL \
    myrepo/myapp:$IMAGE_TAG \
    node dist/migrate.js

  # Run migration BEFORE deploying new app code.
  # New app code must work with BOTH old and new schema.
  # Old app code must still work after migration (expand phase).

  FOR KUBERNETES:
  Use init containers to run migrations before pods start.
  Or a dedicated migration Job that completes before Deployment rollout.
```

---

**Q: Our CI builds take 20 minutes. What's the fastest way to speed them up?**

```
A: Prioritise in this order (biggest gains first):

  1. PARALLELIZE JOBS (5-10x speedup)
     Run lint, test, security scan in parallel jobs.
     Don't run them sequentially if they don't depend on each other.

  2. CACHE DOCKER LAYERS (3-5x speedup)
     If you're not already using type=gha or registry cache,
     add it. This alone often cuts build time by 60-80%.

  3. FIX LAYER ORDER IN DOCKERFILE (2-3x speedup)
     Install deps before copying source code (Batch 6).
     npm ci / pip install should be a CACHED layer 95% of builds.

  4. USE SMALLER BASE IMAGES (1.5-2x speedup)
     alpine pulls in seconds. ubuntu takes 30+ seconds on cold pull.
     Smaller = faster pull, faster push, faster scan.

  5. SPLIT LARGE TESTS (2x speedup)
     Shard test suites across parallel runners:
     strategy:
       matrix:
         shard: [1, 2, 3, 4]
     run: npm test --shard=${{ matrix.shard }}/4

  6. SKIP UNNECESSARY WORK (1.5x speedup)
     Only run full pipeline on main branch.
     On PRs: run lint + unit tests only (skip deploy, scan).
     Use path filters: only run backend pipeline if backend
     files changed.

  7. SELF-HOSTED RUNNERS WITH FAST DISKS
     GitHub-hosted runners use slow HDD storage.
     Self-hosted runners with NVMe SSDs are 5x faster for I/O.
     AWS EC2 r5 instances are popular CI runner hosts.
```

---

**Q: We accidentally pushed a bad image to production. How do we roll back fast?**

```
A: You need a runbook for this. If you don't have one: write it now,
   before the next incident.

  ROLLBACK RUNBOOK:
  ─────────────────────────────────────────────────────────
  T+0: DETECT (alert fires — error rate spike, health check failing)

  T+1min: IDENTIFY the bad image
  $ docker inspect myapp-prod | grep Image
  → "Image": "sha256:abc123..."  ← currently running

  T+2min: FIND the last good image
  # From your CI/CD logs: last successful deploy SHA
  # From registry: previous tag
  $ docker images myrepo/myapp --format "{{.Tag}}\t{{.CreatedAt}}"

  T+3min: EXECUTE ROLLBACK
  # Docker Compose:
  $ IMAGE_TAG=<previous-sha> docker compose up -d --no-deps api

  # Docker Swarm:
  $ docker service rollback myapp_api  # ← automatic to previous

  # Kubernetes:
  $ kubectl rollout undo deployment/myapp
  # or to specific revision:
  $ kubectl rollout undo deployment/myapp --to-revision=3

  T+5min: VERIFY rollback
  curl https://myapp.com/health | grep version
  Watch error rate in monitoring.

  T+15min: INCIDENT REVIEW (after fire is out)
  What broke? Why did it pass CI? What gate would have caught it?
  Add that gate. Update runbook.

  WHAT MAKES ROLLBACK FAST:
  ✅ Previous image is still in registry (don't prune it)
  ✅ Previous image is cached on the host (quick restart)
  ✅ Rollback is a one-command operation, documented in runbook
  ✅ Team drills rollback monthly so it's not a fire-drill moment
```

---

*End of Batch 9.*

---

# BATCH 10 — FINAL
## Advanced Docker: Registry Internals, BuildKit, Multi-Arch, Performance & The Production Cheatsheet

> **Goal:** This is the capstone batch. It ties together everything from Batches 1–9 with
> advanced internals knowledge, performance tuning, and a complete production operations
> reference you can return to every day. After this batch you'll have moved from beginner
> to a DevOps engineer who truly understands containers end to end.

---

## 10.1 Container Registry Internals

Most engineers treat a registry as a black box. Knowing how it works helps you
debug pushes/pulls, design caching strategy, and reason about image security.

### The OCI Distribution Spec — What a Registry Actually Stores

```
  A container image in a registry is NOT a single file.
  It is a set of content-addressable blobs + a manifest.

  REGISTRY STORAGE LAYOUT (conceptual):
  ─────────────────────────────────────────────────────────────
  /blobs/sha256/
    ├── a1b2c3...  ← Layer 1 (compressed tar, e.g. Ubuntu base)
    ├── d4e5f6...  ← Layer 2 (apt packages)
    ├── g7h8i9...  ← Layer 3 (your app code)
    └── j0k1l2...  ← Image config (JSON: env, cmd, labels, history)

  /manifests/myrepo/myapp/
    ├── sha256:abc...  ← Manifest (list of layer digests + config ref)
    ├── v1.2.3         ← Tag → points to manifest sha256:abc...
    └── latest         ← Tag → points to manifest sha256:abc...

  MANIFEST (JSON):
  {
    "schemaVersion": 2,
    "mediaType": "application/vnd.oci.image.manifest.v1+json",
    "config": {
      "digest": "sha256:j0k1l2...",
      "size": 7023
    },
    "layers": [
      {"digest": "sha256:a1b2c3...", "size": 31536000},
      {"digest": "sha256:d4e5f6...", "size": 20480},
      {"digest": "sha256:g7h8i9...", "size": 5120}
    ]
  }
```

### What Happens During `docker pull`

```
  docker pull myrepo/myapp:v1.2.3

  STEP 1: Resolve tag → manifest digest
          GET /v2/myrepo/myapp/manifests/v1.2.3
          Response: manifest JSON + digest sha256:abc...

  STEP 2: Parse manifest — extract layer digests

  STEP 3: For each layer:
          Check local cache: /var/lib/docker/image/.../layerdb/
          If cached → SKIP (this is layer deduplication!)
          If missing → GET /v2/myrepo/myapp/blobs/sha256:a1b2c3...

  STEP 4: Download missing layers in parallel (BuildKit)

  STEP 5: Decompress + unpack layers into OverlayFS

  STEP 6: Write image manifest to local content store

  WHY THIS MATTERS:
  ─────────────────────────────────────────────────────────────
  Shared base layers are NEVER downloaded twice on the same host.
  If 5 images all use node:20-alpine:
  First pull: downloads alpine layers (~7MB) + node layers (~50MB)
  Next 4 pulls: those layers are CACHED, only new layers download.

  Pull time for a 500MB image:  cold = 60s, warm = 2s (just manifest)

  LAYER DEDUPLICATION IN PRACTICE:
  docker images
  REPOSITORY     TAG      IMAGE ID       SIZE
  myapp-api      latest   a1b2c3...      180MB
  myapp-worker   latest   d4e5f6...      185MB

  These look like 365MB total, but they SHARE the node:20-alpine
  base layers on disk. Actual disk usage ≈ 200MB.

  docker system df   ← shows real disk usage vs virtual size
```

### Multi-Architecture Manifest Lists (Image Indexes)

```
  docker pull nginx:latest
  → Docker checks your platform: linux/amd64 or linux/arm64?
  → Pulls the right image automatically.

  HOW? The tag points to a MANIFEST LIST (OCI Image Index):

  Manifest List sha256:XYZ (nginx:latest):
  ┌─────────────────────────────────────────────────────────┐
  │  platform: linux/amd64  → Manifest sha256:amd-abc...   │
  │  platform: linux/arm64  → Manifest sha256:arm-def...   │
  │  platform: linux/arm/v7 → Manifest sha256:armv7-ghi...│
  └─────────────────────────────────────────────────────────┘

  Each platform-specific manifest has its own layers.
  docker pull resolves: tag → list → your platform → manifest → layers.

  INSPECT A MANIFEST LIST:
  $ docker buildx imagetools inspect nginx:latest
  Name:      docker.io/library/nginx:latest
  MediaType: application/vnd.oci.image.index.v1+json
  Digest:    sha256:XYZ...

  Manifests:
    Name:      docker.io/library/nginx:latest@sha256:amd-abc...
    MediaType: application/vnd.oci.image.manifest.v1+json
    Platform:  linux/amd64

    Name:      docker.io/library/nginx:latest@sha256:arm-def...
    MediaType: application/vnd.oci.image.manifest.v1+json
    Platform:  linux/arm64
```

### Self-Hosted Registry

```bash
  # Run a local Docker registry (OCI Distribution Spec compliant)
  docker run -d \
    --name registry \
    --restart unless-stopped \
    -p 5000:5000 \
    -v /mnt/registry:/var/lib/registry \
    -e REGISTRY_STORAGE_DELETE_ENABLED=true \
    registry:2

  # Push to local registry:
  docker tag myapp:latest localhost:5000/myapp:latest
  docker push localhost:5000/myapp:latest

  # Pull from local registry:
  docker pull localhost:5000/myapp:latest

  # List all repositories:
  curl http://localhost:5000/v2/_catalog

  # List tags for an image:
  curl http://localhost:5000/v2/myapp/tags/list

  # Delete an image (requires REGISTRY_STORAGE_DELETE_ENABLED=true):
  # 1. Get digest
  DIGEST=$(curl -s -I -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
    http://localhost:5000/v2/myapp/manifests/latest | grep Docker-Content-Digest | awk '{print $2}')
  # 2. Delete by digest
  curl -X DELETE http://localhost:5000/v2/myapp/manifests/$DIGEST

  # Then run garbage collection to free disk:
  docker exec registry bin/registry garbage-collect /etc/docker/registry/config.yml
```

---

## 10.2 BuildKit Deep Dive

BuildKit is the modern Docker build engine (default since Docker 23).
It replaces the legacy builder with a completely different architecture.

### BuildKit Architecture

```
  LEGACY BUILDER (docker build, pre-BuildKit):
  ─────────────────────────────────────────────
  Sequential: execute one Dockerfile instruction at a time.
  Every RUN spawns a container, commits a layer, removes container.
  No parallelism. All instructions share one build context.

  BUILDKIT:
  ─────────────────────────────────────────────
  LLB (Low-Level Build) — DAG-based execution engine.

  Dockerfile → Frontend parses → LLB graph → Scheduler → Workers

  ┌────────────────────────────────────────────────────────────┐
  │                   LLB Execution Graph (DAG)                │
  │                                                            │
  │  FROM node          FROM golang                            │
  │       │                  │                                 │
  │  npm install         go mod download                       │
  │       │                  │           ← runs in PARALLEL    │
  │  npm run build       go build                              │
  │       │                  │                                 │
  │       └──── COPY ────────┘                                 │
  │              │                                             │
  │         Final image                                        │
  └────────────────────────────────────────────────────────────┘

  KEY BUILDKIT FEATURES:
  ✅ Parallel stage execution (multi-stage builds run concurrently)
  ✅ Content-addressable cache (cache by layer content, not order)
  ✅ Cache mounts (persistent cache dirs across builds, no layer bloat)
  ✅ Secret mounts (secrets available in RUN, never in image)
  ✅ SSH agent forwarding (git clone private repos during build)
  ✅ Inline cache (export cache into the image layers)
  ✅ Better output (progress display, timing per step)
  ✅ Cross-compilation (build for foreign architectures)
```

### BuildKit Cache Mounts — The Hidden Performance Feature

```dockerfile
  # WITHOUT cache mounts:
  # Every build: apt downloads all packages from internet
  # Every build: pip downloads all wheels from PyPI
  # Every build: npm downloads all packages from npm registry

  # WITH cache mounts: packages cached between builds on disk
  # Second build: reads from local cache, no network download

  # syntax=docker/dockerfile:1

  # APT cache mount
  RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
      --mount=type=cache,target=/var/lib/apt,sharing=locked \
      apt-get update && apt-get install -y --no-install-recommends \
        build-essential curl git

  # pip cache mount
  RUN --mount=type=cache,target=/root/.cache/pip \
      pip install -r requirements.txt

  # npm cache mount
  RUN --mount=type=cache,target=/root/.npm \
      npm ci

  # Go module cache mount
  RUN --mount=type=cache,target=/go/pkg/mod \
      go mod download

  # Maven dependency cache
  RUN --mount=type=cache,target=/root/.m2 \
      mvn dependency:go-offline

  # HOW IT WORKS:
  # Cache mount is a persistent directory on the BUILD HOST.
  # Shared across all builds. Never written to image layers.
  # apt-get update, pip install, npm ci all check the cache first.
  # Second build: near-instant. Only changed packages download.

  # SHARING MODES:
  # sharing=locked  → only one build can use this cache at a time
  # sharing=shared  → multiple builds can read simultaneously
  # sharing=private → each build gets its own cache instance
```

### BuildKit Secret Mounts

```dockerfile
  # syntax=docker/dockerfile:1

  # Build-time secret (API key, private token, etc.)
  # Never written to any image layer. Invisible in docker history.

  RUN --mount=type=secret,id=npm_token \
      npm config set //registry.npmjs.org/:_authToken=$(cat /run/secrets/npm_token) \
      && npm ci \
      && npm config delete //registry.npmjs.org/:_authToken

  # Pass secret at build time:
  docker buildx build \
    --secret id=npm_token,env=NPM_TOKEN \
    .

  # Or from a file:
  docker buildx build \
    --secret id=npm_token,src=./npm_token.txt \
    .

  # SSH agent forwarding (for private git repos):
  RUN --mount=type=ssh \
      git clone git@github.com:myorg/private-lib.git /deps/private-lib

  # Build with SSH agent:
  docker buildx build --ssh default=$SSH_AUTH_SOCK .
```

### BuildKit `--mount=type=bind` for Test Runs

```dockerfile
  # Run tests during build without copying test files into image
  # syntax=docker/dockerfile:1
  FROM node:20-alpine AS base
  WORKDIR /app
  COPY package*.json ./
  RUN npm ci

  FROM base AS test
  RUN --mount=type=bind,source=.,target=/app/src,rw \
      npm test -- --passWithNoTests
  # Test files are bind-mounted at build time, NOT in the final image

  FROM base AS runtime
  COPY src/ ./src/
  CMD ["node", "src/index.js"]
  # Final image has no test files
```

---

## 10.3 Multi-Architecture Builds — Full Guide

```bash
  # SETUP (one time):
  # Enable QEMU for cross-compilation emulation
  docker run --privileged --rm tonistiigi/binfmt --install all

  # Create a multi-arch capable builder
  docker buildx create --name multiarch-builder \
    --driver docker-container \
    --use
  docker buildx inspect --bootstrap

  # BUILD for multiple platforms:
  docker buildx build \
    --platform linux/amd64,linux/arm64,linux/arm/v7 \
    --tag myrepo/myapp:v1.2.3 \
    --push \
    .

  # VERIFY the manifest list:
  docker buildx imagetools inspect myrepo/myapp:v1.2.3
```

### Platform-Conditional Dockerfile Logic

```dockerfile
  # syntax=docker/dockerfile:1
  FROM --platform=$BUILDPLATFORM node:20-alpine AS builder
  # $BUILDPLATFORM = the machine doing the building (e.g. linux/amd64)
  # Use this for build tools — run natively, not emulated

  ARG TARGETPLATFORM
  ARG TARGETOS
  ARG TARGETARCH
  # TARGETPLATFORM = the platform being built FOR (e.g. linux/arm64)

  WORKDIR /app
  COPY . .
  RUN npm ci && npm run build

  FROM --platform=$TARGETPLATFORM node:20-alpine AS runtime
  # $TARGETPLATFORM = final image is for this platform
  WORKDIR /app
  COPY --from=builder /app/dist ./dist
  CMD ["node", "dist/index.js"]
```

### Go Cross-Compilation (No Emulation Needed)

```dockerfile
  # syntax=docker/dockerfile:1
  FROM --platform=$BUILDPLATFORM golang:1.22 AS builder
  # Always runs on native builder platform (fast, no QEMU)

  ARG TARGETOS TARGETARCH
  WORKDIR /src
  COPY . .
  RUN --mount=type=cache,target=/go/pkg/mod \
      GOOS=$TARGETOS GOARCH=$TARGETARCH \
      go build -o /app/server ./cmd/server
  # Go cross-compiles natively — no emulation, blazing fast

  FROM --platform=$TARGETPLATFORM scratch AS runtime
  COPY --from=builder /app/server /server
  ENTRYPOINT ["/server"]

  # BUILD:
  # docker buildx build \
  #   --platform linux/amd64,linux/arm64 \
  #   --push \
  #   -t myrepo/myapp:latest .
  #
  # Both images built on AMD64 host without QEMU emulation.
  # AMD64 takes ~5s. ARM64 takes ~6s. Parallel if BuildKit.
```

---

## 10.4 Docker Daemon Configuration — `/etc/docker/daemon.json`

The daemon config file controls host-wide defaults for all containers.
Know every knob:

```json
  {
    "data-root": "/var/lib/docker",
    "storage-driver": "overlay2",

    "log-driver": "json-file",
    "log-opts": {
      "max-size": "10m",
      "max-file": "3",
      "compress": "true"
    },

    "registry-mirrors": [
      "https://mirror.mycompany.internal"
    ],

    "insecure-registries": [
      "localhost:5000",
      "dev-registry.internal:5000"
    ],

    "dns": ["1.1.1.1", "8.8.8.8"],
    "dns-search": ["mycompany.internal"],

    "default-ulimits": {
      "nofile": {
        "Name": "nofile",
        "Hard": 64000,
        "Soft": 64000
      }
    },

    "live-restore": true,
    "userns-remap": "default",
    "no-new-privileges": true,

    "default-address-pools": [
      {"base": "172.30.0.0/16", "size": 24}
    ],

    "features": {
      "buildkit": true
    },

    "experimental": false,

    "metrics-addr": "127.0.0.1:9323",
    "experimental": true
  }
```

```
  KEY SETTINGS EXPLAINED:
  ─────────────────────────────────────────────────────────────
  live-restore         Containers keep running if daemon restarts.
                       Critical for production: upgrade dockerd
                       without killing your apps.

  userns-remap         Enable user namespace remapping (Batch 8).
                       "default" creates a dockremap user.

  no-new-privileges    Daemon-wide: no container can gain new
                       Linux capabilities via setuid/setgid.

  default-address-pools Controls the subnet ranges Docker uses
                       for new bridge networks. Change if it
                       conflicts with your VPN (172.17/16 often does).

  registry-mirrors     Pull-through cache for Docker Hub. Avoids
                       rate limits. Speeds up pulls on many nodes.

  metrics-addr         Expose Prometheus metrics on this address.
                       Scrape with Prometheus for Docker host metrics.

  log-driver           Change for all containers. json-file is default.
                       Use "awslogs" for CloudWatch, "gcplogs" for GCP,
                       "fluentd" for Fluentd. Override per-container.
```

---

## 10.5 Performance Tuning

### Storage Driver Performance

```
  STORAGE DRIVER COMPARISON:
  ─────────────────────────────────────────────────────────────
  overlay2   ← DEFAULT. Best choice for most cases.
               Fast, efficient, supports Linux kernel 4.0+.
               Use with ext4 or xfs filesystem on host.

  btrfs      → Use if host filesystem is btrfs.
               Native btrfs subvolume per layer = fast.
               snapshot-based, efficient space use.

  zfs        → Use with ZFS on host.
               Best for immutable infrastructure.
               Strong deduplication.

  devicemapper → DEPRECATED. Avoid.
  aufs         → DEPRECATED. Linux only, some distros removed it.
  vfs          → NEVER IN PRODUCTION. No copy-on-write, copies
                 entire layer on every container. Absurdly slow.
                 Used only for testing and some CI edge cases.

  CHECK YOUR CURRENT DRIVER:
  $ docker info | grep "Storage Driver"
```

### Container Start Time Optimisation

```
  FACTORS AFFECTING START TIME:
  ─────────────────────────────────────────────────────────────
  Image pull (cold):  30s–3min → Pre-pull images, use registry mirror
  Image unpack:       1–10s   → Smaller images unpack faster
  App init:           Varies  → Optimise startup in your code

  IMAGE SIZE vs STARTUP TIME CORRELATION:
  ubuntu:22.04 (77MB)    → unpack ~3s
  alpine:3.19 (7MB)      → unpack ~0.3s
  distroless/base (2MB)  → unpack ~0.1s
  scratch (0MB)          → unpack ~0s

  RUNTIME START (after image is local):
  Container creation:  ~50ms
  Namespace setup:     ~10ms
  cgroup setup:        ~10ms
  Overlay mount:       ~20ms
  Your app start:      depends entirely on your code

  Total container overhead before your app runs: ~100ms
  Everything after that is your application's responsibility.

  APP STARTUP OPTIMISATION:
  - Lazy-load modules (don't import everything at startup)
  - Pre-compile templates / warm caches asynchronously
  - Health endpoint available immediately, readiness delayed
  - Use /health/ready to signal when app is truly ready
```

### I/O Performance

```
  AVOID WRITING TO CONTAINER WRITABLE LAYER:
  Every write to the writable layer goes through OverlayFS CoW.
  Read: lowerdir → fast (shared with image)
  Write: copy file to upperdir first → SLOW (CoW overhead)

  Write-heavy paths (DB data, logs, uploads) MUST use volumes.

  I/O BENCHMARK COMPARISON (approximate):
  tmpfs (RAM):           ~10 GB/s read,  ~10 GB/s write
  Named Volume (NVMe):   ~3 GB/s  read,  ~2 GB/s  write
  Bind Mount (NVMe):     ~3 GB/s  read,  ~2 GB/s  write
  Container WL (NVMe):   ~1.5 GB/s read, ~0.8 GB/s write  ← CoW penalty
  NFS Volume:            ~200 MB/s read, ~100 MB/s write

  NETWORKING PERFORMANCE:
  Container → Container (same host, bridge):  ~10 Gbps (software bridge)
  Container → Container (overlay, multi-host): ~5–8 Gbps (VXLAN overhead)
  Container with --network host:               ~line rate (no overhead)

  MEMORY PERFORMANCE:
  Container memory is native host memory.
  No virtualisation overhead (unlike VMs).
  cgroup limits are enforced by kernel, near-zero overhead.
```

---

## 10.6 Diagnosing Production Issues — The Complete Runbook

```
  CONTAINER IS DOWN / KEEPS RESTARTING:
  ─────────────────────────────────────────────────────────────
  1. docker ps -a                    # see exit code
  2. docker logs --tail=100 myapp    # last 100 lines of output
  3. docker inspect myapp | grep -E '"ExitCode"|"Error"|"OOMKilled"'
  4. If OOMKilled: true → memory limit too low (Batch 2 / Batch 5)
  5. If ExitCode 1 → app crashed, check logs for stack trace
  6. If ExitCode 137 → SIGKILL (OOM or manual kill)
  7. If ExitCode 143 → SIGTERM not handled, app didn't shutdown cleanly

  APP RUNNING BUT NOT RESPONDING:
  ─────────────────────────────────────────────────────────────
  1. docker exec myapp wget -qO- http://localhost:3000/health
     → if timeout: app deadlocked or not listening
  2. docker exec myapp ss -tlnp      # what ports is app listening on?
  3. docker exec myapp ps aux         # is the process actually running?
  4. docker stats myapp               # CPU 100%? Memory near limit?
  5. docker exec myapp top            # which thread/process is spiking?
  6. docker diff myapp                # unexpected files written?

  CONTAINER CAN'T REACH ANOTHER SERVICE:
  ─────────────────────────────────────────────────────────────
  1. docker exec myapp ping db        # DNS resolves?
  2. docker exec myapp nslookup db    # detailed DNS check
  3. docker exec myapp nc -zv db 5432 # TCP port reachable?
  4. docker network inspect myapp-net # are both on same network?
  5. docker exec db ss -tlnp          # is db actually listening?
  6. docker logs db --tail=50         # did db crash or refuse connection?

  BUILDS ARE SLOW:
  ─────────────────────────────────────────────────────────────
  1. docker buildx build --progress=plain .  # see timing per step
  2. Look for steps with no "CACHED" prefix → fix layer order
  3. Check .dockerignore → build context should be < 5MB
     "transferring context: 234.5MB" → missing .dockerignore entries
  4. Enable BuildKit if not already: DOCKER_BUILDKIT=1
  5. Add --mount=type=cache for package managers

  IMAGES ARE TOO LARGE:
  ─────────────────────────────────────────────────────────────
  1. docker images --format "{{.Repository}}:{{.Tag}}\t{{.Size}}"
  2. docker history myimage --no-trunc  # find big layers
  3. docker run --rm -it \
       -v /var/run/docker.sock:/var/run/docker.sock \
       wagoodman/dive myimage:latest    # visual layer explorer
  4. Apply multi-stage builds (Batch 6)
  5. Switch to alpine/distroless base

  DISK SPACE RUNNING OUT:
  ─────────────────────────────────────────────────────────────
  1. docker system df                  # see what's using space
     TYPE            TOTAL   ACTIVE  SIZE      RECLAIMABLE
     Images          15      3       12.4GB    10.1GB
     Containers       6      3        248MB      124MB
     Local Volumes    4      2        2.1GB     1.05GB
     Build Cache                      5.2GB      5.2GB

  2. SAFE CLEANUP (won't touch running containers/used volumes):
     docker image prune         # remove dangling images only
     docker container prune     # remove stopped containers
     docker builder prune       # remove build cache
     docker network prune       # remove unused networks

  3. NUCLEAR OPTION (UNDERSTAND WHAT YOU'RE DELETING):
     docker system prune -a     # removes ALL unused images, containers,
                                 # networks, build cache
                                 # Does NOT remove named volumes
     docker system prune -a --volumes  # ⚠️ ALSO removes named volumes
```

---

## 10.7 Docker Ecosystem Map — What Tool Does What

```
  CONTAINER RUNTIME LAYER:
  ─────────────────────────────────────────────────────────────
  runc          OCI runtime. Actually creates namespaces, cgroups.
                The lowest level. Used by containerd.
  containerd    Industry-standard runtime. Used by K8s, Docker.
  CRI-O         Kubernetes-specific runtime. Lighter than containerd.
  gVisor        User-space kernel. Stronger isolation (Google).
  Kata Containers  VM-based containers. Container UX + VM security.

  IMAGE BUILD:
  ─────────────────────────────────────────────────────────────
  Docker BuildKit  Default builder. Most full-featured.
  Kaniko        Daemonless K8s-native builds.
  Buildah       Daemonless, Red Hat/Podman ecosystem.
  ko            Go-specific: builds container images from Go source.
  Jib           Java-specific: Maven/Gradle plugin, no Dockerfile.
  Bazel         Hermetic, reproducible builds (Google-originated).
  Nixpacks       Auto-detect language, build without Dockerfile.

  IMAGE DISTRIBUTION:
  ─────────────────────────────────────────────────────────────
  Docker Hub    Default public registry. Rate-limited (free tier).
  GHCR          GitHub Container Registry. Free with GitHub.
  ECR           AWS Elastic Container Registry.
  GCR/AR        Google Artifact Registry.
  ACR           Azure Container Registry.
  Quay.io       Red Hat's registry. Strong security scanning.
  Harbor        Self-hosted, enterprise-grade, CNCF project.
  Zot           Lightweight OCI-native self-hosted registry.

  SECURITY:
  ─────────────────────────────────────────────────────────────
  Trivy         CVE scanner (OS + language + IaC + secrets).
  Grype         CVE scanner. Accurate, fast.
  Syft          SBOM generator.
  Cosign        Image signing (Sigstore).
  Falco         Runtime security monitoring (eBPF).
  OPA/Gatekeeper  Policy enforcement in K8s.
  Kyverno       K8s-native policy engine.

  ORCHESTRATION:
  ─────────────────────────────────────────────────────────────
  Docker Compose  Multi-container on single host.
  Docker Swarm    Multi-host, built into Docker. Simpler than K8s.
  Kubernetes      De facto standard orchestrator. Complex, powerful.
  Nomad           HashiCorp's orchestrator. Simpler than K8s.
  ECS             AWS-managed container orchestration.
  Cloud Run       GCP serverless containers. Auto-scales to zero.
  Azure Container Apps  Azure's managed container platform.

  DEVELOPER TOOLS:
  ─────────────────────────────────────────────────────────────
  Docker Desktop  Mac/Windows local dev. Includes K8s.
  Colima         Lightweight Docker Desktop alternative (Mac).
  Podman Desktop  Daemonless Docker alternative. Red Hat.
  Rancher Desktop  K8s + container runtime for local dev.
  lazydocker      Terminal UI for managing containers.
  dive            Explore image layers and waste.
  ctop            Container resource monitoring TUI.
  Portainer       Web UI for Docker management.

  OBSERVABILITY:
  ─────────────────────────────────────────────────────────────
  Prometheus + Grafana  Metrics + dashboards.
  Loki + Grafana       Container log aggregation.
  Jaeger / Tempo       Distributed tracing.
  cAdvisor             Container metrics exporter (for Prometheus).
  Datadog / New Relic  APM with container awareness.
```

---

## 10.8 The Production Operations Cheatsheet

A single-page reference for every daily container operation.

### Container Operations

```bash
  # Run container with all production flags
  docker run -d \
    --name myapp \
    --restart unless-stopped \
    --user 1000:1000 \
    --read-only \
    --tmpfs /tmp:rw,noexec,nosuid \
    --cap-drop ALL \
    --security-opt no-new-privileges:true \
    --memory=512m --cpus=0.5 \
    --pids-limit=100 \
    --network myapp-net \
    --env-file /etc/myapp/prod.env \
    --log-opt max-size=10m --log-opt max-file=3 \
    myrepo/myapp:v1.2.3

  # Check container status
  docker ps -a --format "table {{.Names}}\t{{.Status}}\t{{.Image}}"

  # Live resource usage
  docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"

  # Follow logs with timestamps
  docker logs -f --timestamps --tail=100 myapp

  # Exec into running container
  docker exec -it myapp sh

  # Copy a file out of a container
  docker cp myapp:/app/config.json ./config-backup.json

  # Inspect a container (full JSON)
  docker inspect myapp | python3 -m json.tool

  # Check just the IP
  docker inspect myapp --format '{{.NetworkSettings.Networks.myapp-net.IPAddress}}'

  # Check environment variables
  docker inspect myapp --format '{{range .Config.Env}}{{println .}}{{end}}'

  # See file changes in writable layer
  docker diff myapp

  # Export container filesystem as tar
  docker export myapp > myapp-filesystem.tar
```

### Image Operations

```bash
  # Build with full metadata
  docker buildx build \
    --platform linux/amd64,linux/arm64 \
    --build-arg GIT_SHA=$(git rev-parse --short HEAD) \
    --build-arg BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
    --cache-from type=registry,ref=myrepo/myapp:buildcache \
    --cache-to   type=registry,ref=myrepo/myapp:buildcache,mode=max \
    --tag myrepo/myapp:$(git rev-parse --short HEAD) \
    --push \
    .

  # Tag image
  docker tag myapp:sha-abc123 myapp:v1.2.3
  docker tag myapp:sha-abc123 myapp:latest

  # Inspect image layers
  docker history myimage --no-trunc

  # Pull specific digest (tamper-proof)
  docker pull myrepo/myapp@sha256:a1b2c3...

  # Scan image for CVEs
  trivy image --severity HIGH,CRITICAL myapp:latest

  # View image size breakdown
  docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
    wagoodman/dive myapp:latest

  # Save image to file (for air-gapped environments)
  docker save myapp:v1.2.3 | gzip > myapp-v1.2.3.tar.gz

  # Load image from file
  docker load < myapp-v1.2.3.tar.gz
```

### Network Operations

```bash
  # Create isolated network
  docker network create --driver bridge myapp-net

  # Create internal network (no internet access)
  docker network create --internal backend-net

  # Connect running container to a network
  docker network connect myapp-net mycontainer

  # Disconnect from a network
  docker network disconnect old-net mycontainer

  # Inspect network (see all containers and IPs)
  docker network inspect myapp-net

  # Debug connectivity from inside a container
  docker run --rm --network container:myapp \
    nicolaka/netshoot ping db

  # Capture traffic on docker bridge
  sudo tcpdump -i docker0 -n -w capture.pcap
```

### Volume Operations

```bash
  # Create named volume
  docker volume create --label env=prod pgdata

  # Backup volume to S3
  docker run --rm \
    -v pgdata:/data:ro \
    -e AWS_DEFAULT_REGION=us-east-1 \
    amazon/aws-cli s3 sync /data s3://my-backups/pgdata/$(date +%Y%m%d)/

  # Restore volume from S3
  docker run --rm \
    -v pgdata:/data \
    -e AWS_DEFAULT_REGION=us-east-1 \
    amazon/aws-cli s3 sync s3://my-backups/pgdata/20241201/ /data

  # List all volumes with sizes
  docker system df -v

  # Find orphaned volumes (not used by any container)
  docker volume ls -qf dangling=true

  # Remove specific volume
  docker volume rm pgdata

  # Safe prune (only unnamed/dangling volumes)
  docker volume prune --filter "label!=keep=true"
```

### System Maintenance

```bash
  # Show disk usage breakdown
  docker system df

  # Safe cleanup (does not affect running containers or named volumes)
  docker container prune   # remove stopped containers
  docker image prune       # remove dangling (untagged) images
  docker network prune     # remove unused networks
  docker builder prune     # remove build cache

  # More aggressive: remove all unused images (not just dangling)
  docker image prune -a

  # Show all images sorted by size
  docker images --format "{{.Size}}\t{{.Repository}}:{{.Tag}}" \
    | sort -rh | head -20

  # Check daemon logs (systemd)
  sudo journalctl -u docker.service -f

  # Reload daemon config without restart
  sudo kill -HUP $(pidof dockerd)

  # Check docker daemon info
  docker info

  # Check docker version
  docker version
```

---

## 10.9 Mental Model Consolidation — The Full Stack

After 10 batches, here is the complete mental model in one diagram:

```
  ╔══════════════════════════════════════════════════════════════════╗
  ║              THE FULL CONTAINER STACK                            ║
  ╠══════════════════════════════════════════════════════════════════╣
  ║                                                                  ║
  ║  DEVELOPER                                                       ║
  ║  ┌─────────────────────────────────────────────────────────┐    ║
  ║  │  Dockerfile → docker build → Image (layers, manifest)   │    ║
  ║  │  docker-compose.yml → docker compose up → Services      │    ║
  ║  └──────────────────────────┬──────────────────────────────┘    ║
  ║                             │ docker push                        ║
  ║  REGISTRY                   ▼                                    ║
  ║  ┌─────────────────────────────────────────────────────────┐    ║
  ║  │  Blobs (layers) + Manifests + Tags                      │    ║
  ║  │  ECR / GHCR / Docker Hub / Harbor                       │    ║
  ║  └──────────────────────────┬──────────────────────────────┘    ║
  ║                             │ docker pull                        ║
  ║  HOST MACHINE               ▼                                    ║
  ║  ┌─────────────────────────────────────────────────────────┐    ║
  ║  │  /var/lib/docker/  (image layers, volumes, networks)    │    ║
  ║  │                                                         │    ║
  ║  │  DOCKER DAEMON (dockerd)                                │    ║
  ║  │    ↓ gRPC                                               │    ║
  ║  │  CONTAINERD                                             │    ║
  ║  │    ↓ OCI                                                │    ║
  ║  │  RUNC ──► creates ──► CONTAINER                         │    ║
  ║  │                         │                               │    ║
  ║  │  LINUX KERNEL           │                               │    ║
  ║  │  ┌──────────────────────▼─────────────────────────┐    │    ║
  ║  │  │  namespaces  │  cgroups  │  OverlayFS  │ seccomp│    │    ║
  ║  │  └────────────────────────────────────────────────┘    │    ║
  ║  └─────────────────────────────────────────────────────────┘    ║
  ║                                                                  ║
  ║  WHAT THE CONTAINER SEES:           WHAT THE HOST SEES:         ║
  ║  PID 1 = my app                     PID 8472 = my app           ║
  ║  IP = 172.17.0.2                    docker0 bridge              ║
  ║  / = my filesystem                  OverlayFS mount             ║
  ║  Hostname = container-1             alice's process             ║
  ║                                                                  ║
  ╚══════════════════════════════════════════════════════════════════╝
```

---

## 10.10 What to Learn Next — The Roadmap Beyond Docker

```
  You now understand Docker from fundamentals to production.
  Here's where to go next in the DevOps journey:

  IMMEDIATE NEXT STEPS (high ROI):
  ─────────────────────────────────────────────────────────────
  1. Kubernetes Fundamentals
     Pods, Deployments, Services, ConfigMaps, Secrets.
     Your Docker knowledge transfers directly.
     Resources: Kubernetes.io docs, KillerCoda playgrounds.

  2. Kubernetes Networking (builds on Batch 4)
     Services, Ingress, Network Policies, CNI plugins.

  3. Helm — Kubernetes Package Manager
     Package K8s manifests into reusable charts.
     Manage deployments across environments.

  4. GitOps (ArgoCD / Flux)
     Declare desired state in git.
     ArgoCD syncs cluster to match git. Auto-healing.

  5. Observability Stack
     Prometheus + Grafana (metrics)
     Loki + Grafana (logs)
     Jaeger / Tempo (traces)
     Learn the "three pillars of observability."

  INTERMEDIATE (3–6 months):
  ─────────────────────────────────────────────────────────────
  6.  Kubernetes RBAC and Security
  7.  Service Mesh (Istio or Linkerd)
  8.  Infrastructure as Code (Terraform, Pulumi)
  9.  Cloud provider certifications (AWS/GCP/Azure)
  10. eBPF (what Cilium and Falco use under the hood)

  ADVANCED (6–12 months):
  ─────────────────────────────────────────────────────────────
  11. Kubernetes Operators (extend K8s with custom controllers)
  12. Platform Engineering (Internal Developer Platforms)
  13. SRE practices (SLOs, error budgets, toil reduction)
  14. Supply chain security (SLSA framework, SBOM pipelines)
  15. Multi-cluster Kubernetes (Argo Rollouts, Federation)
```

---

## 10.11 Final Production FAQ — Advanced Topics

---

**Q: We're migrating a stateful monolith to containers. What's the recommended sequence?**

```
A: Don't try to do everything at once. A phased approach:

  PHASE 1 — Containerise without changing architecture (2–4 weeks)
  ─────────────────────────────────────────────────────────────
  Write a Dockerfile for the monolith as-is.
  Use multi-stage build to keep image lean.
  Run with docker compose locally, verify it behaves identically.
  External state (DB, files) stays on host initially.

  PHASE 2 — Externalise state (2–4 weeks)
  ─────────────────────────────────────────────────────────────
  Move DB to a named volume or managed DB service.
  Move file uploads to S3 / object storage.
  Move sessions to Redis.
  Now the container is stateless and replaceable.

  PHASE 3 — CI/CD pipeline (1–2 weeks)
  ─────────────────────────────────────────────────────────────
  Add Dockerfile lint, CVE scan, sign, push to pipeline.
  Deploy to staging via compose on a VM.
  Validate: does it behave identically to old deployment?

  PHASE 4 — Production deploy (1 week)
  ─────────────────────────────────────────────────────────────
  Blue/green deploy: run containerised version alongside old.
  Route a small % of traffic. Monitor error rates.
  Cut over fully once confident. Keep old running for 24h.

  PHASE 5 — Hardening (ongoing)
  ─────────────────────────────────────────────────────────────
  Add healthchecks, resource limits, security flags.
  Tune image size. Add structured logging.
  Only after this is stable: consider decomposing into services.

  GOLDEN RULE:
  Containerise first. Decompose later (if needed).
  "Containers + monolith" is a valid, stable destination.
  Microservices complexity is a choice, not a requirement.
```

---

**Q: How do we benchmark and tune container performance on production hardware?**

```
A: A structured benchmarking approach:

  STEP 1 — BASELINE (before any tuning)
  docker stats --no-stream    # snapshot of CPU/memory at rest
  wrk or hey for HTTP load:
  wrk -t4 -c100 -d30s http://container-host/api/endpoint

  STEP 2 — IDENTIFY THE BOTTLENECK
  CPU-bound:   docker stats shows CPU near limit, low memory use
               → increase --cpus, or optimise code
  Memory-bound: memory usage near limit, OOMKilled events
               → increase --memory, fix memory leak
  I/O-bound:   docker stats shows high BlockIO, low CPU
               → use volumes instead of container writable layer
               → check disk IOPS with iostat -x 1
  Network-bound: high NetIO, latency in traces
               → consider --network host for hot path
               → use connection pooling in app

  STEP 3 — TUNE AND REMEASURE
  CPU: --cpus sets hard limit. --cpu-shares sets relative priority.
       Pin hot containers to dedicated cores: --cpuset-cpus=0,1

  Memory: Set --memory to measured peak × 1.3.
          Set --memory-reservation to average usage.
          For JVM: -XX:MaxRAMPercentage=75.0

  I/O: Move write paths to named volumes (NVMe-backed).
       For DB: ensure volume is on fast storage, not NFS.
       --device-read-bps / --device-write-bps for throttling.

  STEP 4 — LOAD TEST AT PRODUCTION SCALE
  k6, Gatling, or Locust for realistic load patterns.
  Run for 30+ minutes. Watch for memory growth (leak).
  Measure p50, p95, p99 latency — not just average.

  USEFUL PROFILING TOOLS (inside container):
  Node.js:  node --prof, clinic.js, 0x
  Python:   py-spy, cProfile
  Go:       pprof endpoint built-in
  Java:     async-profiler, JFR
```

---

**Q: What are the differences between Docker, Podman, and containerd for production use?**

```
A: All are OCI-compatible. The choice depends on your context.

  DOCKER:
  ─────────────────────────────────────────────────────────────
  + Best developer experience. Most documentation.
  + Docker Compose built-in. Docker Desktop on Mac/Windows.
  + BuildKit: most advanced build features.
  - Daemon runs as root (unless rootless mode, Batch 8).
  - Docker Desktop requires paid licence for enterprise use.
  - Kubernetes nodes don't use dockerd anymore.
  Best for: developer machines, single-host deployments.

  PODMAN:
  ─────────────────────────────────────────────────────────────
  + Daemonless — no background process required.
  + Rootless by default (each user runs their own podman).
  + Drop-in replacement (alias docker=podman works for basics).
  + Supports pods (group containers like K8s pods).
  + Red Hat/IBM supported, strong in RHEL environments.
  - podman-compose less mature than docker compose.
  - Some Docker features missing or different.
  Best for: RHEL/Fedora environments, high-security requirements.

  CONTAINERD (direct):
  ─────────────────────────────────────────────────────────────
  + Industry standard Kubernetes runtime.
  + Lean, CRI-compliant — designed for orchestrators.
  + No docker UX overhead.
  - No built-in image build (use nerdctl + buildkitd on top).
  - CLI (nerdctl or crictl) less polished than docker.
  Best for: Kubernetes nodes, where you want minimal overhead.

  PRACTICAL RECOMMENDATION:
  Dev machine:       Docker Desktop or Colima (Mac), Docker (Linux)
  CI builds:         Docker (BuildKit), Kaniko (K8s CI)
  K8s nodes:         containerd (set automatically by K8s distro)
  High-security env: Podman (rootless) or containerd + gVisor
```

---

*End of Batch 10 — Course Complete.*

---

## 🎓 You've Completed the Full Curriculum

```
  ┌──────────────────────────────────────────────────────────────┐
  │                   WHAT YOU NOW KNOW                          │
  ├──────────────────────────────────────────────────────────────┤
  │  Batch 1  │ VM vs Container fundamentals, mental models      │
  │  Batch 2  │ namespaces, cgroups, OverlayFS, Docker arch      │
  │  Batch 3  │ Docker CLI mastery, images, Dockerfile basics    │
  │  Batch 4  │ Networking: bridge, host, overlay, DNS           │
  │  Batch 5  │ Volumes, storage, stateful services              │
  │  Batch 6  │ Dockerfile best practices, multi-stage builds    │
  │  Batch 7  │ Docker Compose: dev + production patterns        │
  │  Batch 8  │ Security: scanning, hardening, supply chain      │
  │  Batch 9  │ CI/CD: GitHub Actions, GitLab, deployments       │
  │  Batch 10 │ Registry internals, BuildKit, multi-arch, ops    │
  └──────────────────────────────────────────────────────────────┘

  TOTAL: 30+ gotchas  ·  40+ production FAQs  ·  100+ ASCII diagrams
         Beginner → Production-ready DevOps Engineer
```

---

