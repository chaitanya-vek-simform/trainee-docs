[🏠 Home](../README.md) · [Kubernetes](README.md)

# ☸️ Kubernetes — Comprehensive DevOps Handbook

> **Audience:** DevOps / Cloud / Platform Engineers & Trainees | **Focus:** Architecture, Workloads, Networking, Security, Production Readiness
> **Covers:** K8s Architecture, Pods, Deployments, Services, Storage, Ingress, RBAC, Helm, Operators, Observability, GitOps
> **Tip:** Sections build on each other — read sequentially first, then use TOC for reference.

---

## Table of Contents

1. [Why Kubernetes Matters for DevOps](#1-why-kubernetes-matters-for-devops)
2. [Kubernetes Architecture](#2-kubernetes-architecture)
3. [Pods & Container Basics](#3-pods--container-basics)
4. [ReplicaSets & Deployments](#4-replicasets--deployments)
5. [Services & Service Discovery](#5-services--service-discovery)
6. [Namespaces & Resource Organization](#6-namespaces--resource-organization)
7. [ConfigMaps & Secrets](#7-configmaps--secrets)
8. [Persistent Volumes & Storage](#8-persistent-volumes--storage)
9. [Ingress & Traffic Management](#9-ingress--traffic-management)
10. [StatefulSets & Stateful Workloads](#10-statefulsets--stateful-workloads)
11. [DaemonSets, Jobs & CronJobs](#11-daemonsets-jobs--cronjobs)
12. [RBAC & Security](#12-rbac--security)
13. [Network Policies](#13-network-policies)
14. [Helm & Package Management](#14-helm--package-management)
15. [Operators & Custom Resource Definitions (CRDs)](#15-operators--custom-resource-definitions-crds)
16. [Scheduling & Resource Management](#16-scheduling--resource-management)
17. [Monitoring & Observability](#17-monitoring--observability)
18. [Kubernetes Security Hardening](#18-kubernetes-security-hardening)
19. [Multi-Cluster, Federation & GitOps](#19-multi-cluster-federation--gitops)
20. [Production Gotchas & Scenario-Based FAQ](#20-production-gotchas--scenario-based-faq)

---

## 1. Why Kubernetes Matters for DevOps

### 1.1 The Evolution — From Bare Metal to Orchestration

```
  ┌─────────────────────────────────────────────────────────────────┐
  │  INFRASTRUCTURE EVOLUTION → KUBERNETES                          │
  │                                                                 │
  │  Era 1: Bare Metal (1990s-2000s)                               │
  │  ├── One app per physical server                                │
  │  ├── Weeks to provision, 10-20% utilization                     │
  │  └── Manual scaling, single point of failure                    │
  │                                                                 │
  │  Era 2: Virtualization (2000s-2010s)                           │
  │  ├── Multiple VMs per host (VMware, KVM, Hyper-V)              │
  │  ├── Hours to provision, 40-60% utilization                     │
  │  └── VM sprawl, still heavy (GB-sized images)                  │
  │                                                                 │
  │  Era 3: Containers (2013-present)                              │
  │  ├── Docker (2013) — lightweight, fast, portable               │
  │  ├── Seconds to start, MB-sized images                          │
  │  ├── 100s of containers per host                                │
  │  └── Problem: How do you MANAGE 1000s of containers?           │
  │                                                                 │
  │  Era 4: Container Orchestration (2014-present)                 │
  │  ├── Kubernetes (Google → CNCF, 2014)                          │
  │  ├── Automated deployment, scaling, self-healing               │
  │  ├── Declarative — you say WHAT, K8s figures out HOW           │
  │  └── De-facto standard for container orchestration             │
  └─────────────────────────────────────────────────────────────────┘
```

### 1.2 What Problems Does Kubernetes Solve?

```
  ┌─────────────────────────────────────────────────────────────────┐
  │  THE PROBLEMS K8s SOLVES                                        │
  │                                                                 │
  │  Without K8s:                    With K8s:                      │
  │  ┌────────────────────────┐     ┌────────────────────────┐     │
  │  │ Manual container       │     │ Declarative desired    │     │
  │  │ placement on hosts     │     │ state — K8s places     │     │
  │  │                        │     │ containers optimally   │     │
  │  │ Restart crashed        │     │ Self-healing — auto    │     │
  │  │ containers manually    │     │ restart, reschedule    │     │
  │  │                        │     │                        │     │
  │  │ Manual scaling         │     │ HPA / VPA — auto       │     │
  │  │ (SSH, scripts)         │     │ scale on metrics       │     │
  │  │                        │     │                        │     │
  │  │ Downtime during        │     │ Rolling updates with   │     │
  │  │ deployments            │     │ zero downtime          │     │
  │  │                        │     │                        │     │
  │  │ Hardcoded IPs for      │     │ Service discovery      │     │
  │  │ service communication  │     │ via DNS & labels       │     │
  │  │                        │     │                        │     │
  │  │ Custom health checks   │     │ Built-in liveness,     │     │
  │  │ and monitoring         │     │ readiness, startup     │     │
  │  │                        │     │ probes                 │     │
  │  └────────────────────────┘     └────────────────────────┘     │
  └─────────────────────────────────────────────────────────────────┘
```

### 1.3 Kubernetes vs Other Orchestrators

| Feature | Kubernetes | Docker Swarm | Nomad (HashiCorp) | ECS (AWS) |
|---------|-----------|-------------|-------------------|-----------|
| **Complexity** | High (steep learning curve) | Low | Medium | Low (AWS-only) |
| **Scalability** | 5000+ nodes | ~1000 nodes | 10000+ nodes | AWS-managed |
| **Ecosystem** | Massive (CNCF) | Minimal | Growing | AWS-integrated |
| **Multi-cloud** | ✅ Any cloud / on-prem | ✅ | ✅ | ❌ AWS-only |
| **Self-healing** | ✅ Advanced | ✅ Basic | ✅ | ✅ |
| **Stateful workloads** | ✅ StatefulSets | Limited | ✅ | Limited |
| **Community** | Largest | Declining | Growing | AWS-supported |
| **Best for** | Production at scale | Simple setups | Mixed workloads | AWS-native shops |

### 1.4 Managed Kubernetes — Cloud Comparison

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  MANAGED K8s — CLOUD PROVIDER COMPARISON                        │
  │                                                                  │
  │  Feature          │ EKS (AWS)      │ AKS (Azure)   │ GKE (GCP)  │
  │  ─────────────────┼────────────────┼───────────────┼──────────   │
  │  Control plane    │ $0.10/hr       │ FREE          │ FREE (1)   │
  │  cost             │ ($73/mo)       │               │ ($73 std)  │
  │  ─────────────────┼────────────────┼───────────────┼──────────   │
  │  Node management  │ Managed Node   │ Node Pools    │ Node Pools │
  │                   │ Groups,Fargate │ + Virtual     │ + Autopilot│
  │                   │               │ Nodes (ACI)   │            │
  │  ─────────────────┼────────────────┼───────────────┼──────────   │
  │  Max nodes/       │ 5000 (soft)    │ 5000          │ 15000      │
  │  cluster          │               │               │            │
  │  ─────────────────┼────────────────┼───────────────┼──────────   │
  │  Networking       │ VPC CNI (AWS)  │ Azure CNI     │ VPC-native │
  │                   │ Calico         │ Calico, Cilium│ Calico     │
  │  ─────────────────┼────────────────┼───────────────┼──────────   │
  │  IAM integration  │ IRSA (OIDC)    │ Workload      │ Workload   │
  │                   │               │ Identity      │ Identity   │
  │  ─────────────────┼────────────────┼───────────────┼──────────   │
  │  Best at          │ AWS ecosystem  │ Azure AD,     │ GKE Auto-  │
  │                   │ integration    │ .NET, hybrid  │ pilot, ML  │
  │                                                                  │
  │  (1) GKE Autopilot: free control plane, pay only per pod       │
  └──────────────────────────────────────────────────────────────────┘
```

> **DevOps Relevance:** As a DevOps engineer, you'll interact with Kubernetes daily — deploying apps, managing infrastructure, debugging production issues, and designing CI/CD pipelines. K8s is the #1 skill employers look for in DevOps roles (2024 CNCF survey: 96% of organizations use or evaluate K8s).

---

## 2. Kubernetes Architecture

### 2.1 High-Level Architecture

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │  KUBERNETES CLUSTER ARCHITECTURE                                      │
  │                                                                       │
  │  ┌─────────────── Control Plane (Master) ───────────────────────┐    │
  │  │                                                               │    │
  │  │  ┌──────────┐  ┌───────────┐  ┌───────────┐  ┌──────────┐  │    │
  │  │  │  kube-   │  │  kube-    │  │  kube-    │  │  etcd    │  │    │
  │  │  │  api-    │  │  scheduler│  │  controller│  │(key-value│  │    │
  │  │  │  server  │  │           │  │  manager  │  │  store)  │  │    │
  │  │  │          │  │ Assigns   │  │           │  │          │  │    │
  │  │  │ Gateway  │  │ pods to   │  │ Runs      │  │ Stores   │  │    │
  │  │  │ to the   │  │ nodes     │  │ control   │  │ ALL      │  │    │
  │  │  │ cluster  │  │ based on  │  │ loops     │  │ cluster  │  │    │
  │  │  │          │  │ resources │  │ (replica, │  │ state    │  │    │
  │  │  │ REST API │  │ & affinity│  │  node,    │  │          │  │    │
  │  │  │          │  │           │  │  endpoint)│  │ Raft     │  │    │
  │  │  │ AuthN,   │  │           │  │           │  │ consensus│  │    │
  │  │  │ AuthZ,   │  │           │  │           │  │          │  │    │
  │  │  │ Admission│  │           │  │           │  │ 3 or 5   │  │    │
  │  │  │          │  │           │  │           │  │ node HA  │  │    │
  │  │  └──────────┘  └───────────┘  └───────────┘  └──────────┘  │    │
  │  │       ▲                                                      │    │
  │  └───────┼──────────────────────────────────────────────────────┘    │
  │          │ kubectl / API calls                                       │
  │          │                                                            │
  │  ┌───────┼─────────── Worker Nodes ─────────────────────────────┐   │
  │  │       ▼                                                       │   │
  │  │  ┌─── Node 1 ──────────────┐  ┌─── Node 2 ──────────────┐  │   │
  │  │  │                          │  │                          │  │   │
  │  │  │  ┌────────┐ ┌────────┐  │  │  ┌────────┐ ┌────────┐  │  │   │
  │  │  │  │ Pod A  │ │ Pod B  │  │  │  │ Pod C  │ │ Pod D  │  │  │   │
  │  │  │  │┌─────┐│ │┌─────┐│  │  │  │┌─────┐│ │┌─────┐│  │  │   │
  │  │  │  ││Cont.││ ││Cont.││  │  │  ││Cont.││ ││Cont.││  │  │   │
  │  │  │  │└─────┘│ │└─────┘│  │  │  │└─────┘│ │└─────┘│  │  │   │
  │  │  │  └────────┘ └────────┘  │  │  └────────┘ └────────┘  │  │   │
  │  │  │                          │  │                          │  │   │
  │  │  │  ┌────────────────────┐  │  │  ┌────────────────────┐  │  │   │
  │  │  │  │ kubelet            │  │  │  │ kubelet            │  │  │   │
  │  │  │  │ (manages pods)     │  │  │  │ (manages pods)     │  │  │   │
  │  │  │  ├────────────────────┤  │  │  ├────────────────────┤  │  │   │
  │  │  │  │ kube-proxy         │  │  │  │ kube-proxy         │  │  │   │
  │  │  │  │ (network rules)    │  │  │  │ (network rules)    │  │  │   │
  │  │  │  ├────────────────────┤  │  │  ├────────────────────┤  │  │   │
  │  │  │  │ Container Runtime  │  │  │  │ Container Runtime  │  │  │   │
  │  │  │  │ (containerd/CRI-O) │  │  │  │ (containerd/CRI-O) │  │  │   │
  │  │  │  └────────────────────┘  │  │  └────────────────────┘  │  │   │
  │  │  └──────────────────────────┘  └──────────────────────────┘  │   │
  │  └──────────────────────────────────────────────────────────────┘   │
  └──────────────────────────────────────────────────────────────────────┘
```

### 2.2 Control Plane Components — Deep Dive

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  CONTROL PLANE COMPONENTS — WHAT EACH DOES                      │
  │                                                                  │
  │  Component          │ Role                  │ Key Details        │
  │  ───────────────────┼───────────────────────┼──────────────────  │
  │  kube-apiserver     │ Front door to cluster │ Only component     │
  │                     │ All comms go through  │ that talks to etcd │
  │                     │ REST API (port 6443)  │ Horizontally       │
  │                     │                       │ scalable           │
  │  ───────────────────┼───────────────────────┼──────────────────  │
  │  etcd               │ Cluster state store   │ Key-value store    │
  │                     │ Stores ALL config,    │ Raft consensus     │
  │                     │ secrets, objects      │ Back up regularly! │
  │                     │                       │ 3 or 5 members     │
  │  ───────────────────┼───────────────────────┼──────────────────  │
  │  kube-scheduler     │ Assigns pods to nodes │ Considers:         │
  │                     │                       │ • Resource requests│
  │                     │                       │ • Affinity rules   │
  │                     │                       │ • Taints/toleratns │
  │                     │                       │ • Topology spread  │
  │  ───────────────────┼───────────────────────┼──────────────────  │
  │  kube-controller-   │ Runs control loops    │ Key controllers:   │
  │  manager            │ Ensures desired state │ • ReplicaSet       │
  │                     │ = actual state        │ • Deployment       │
  │                     │                       │ • Node             │
  │                     │                       │ • Job              │
  │                     │                       │ • Endpoint         │
  │                     │                       │ • ServiceAccount   │
  │  ───────────────────┼───────────────────────┼──────────────────  │
  │  cloud-controller-  │ Cloud-specific logic  │ Node lifecycle,    │
  │  manager (optional) │ (EKS/AKS/GKE)        │ load balancers,    │
  │                     │                       │ routes, volumes    │
  └──────────────────────────────────────────────────────────────────┘
```

### 2.3 Worker Node Components

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  WORKER NODE COMPONENTS                                          │
  │                                                                  │
  │  kubelet                                                         │
  │  ├── Agent running on every node                                │
  │  ├── Receives PodSpecs from API server                          │
  │  ├── Ensures containers are running and healthy                 │
  │  ├── Reports node and pod status back to control plane          │
  │  └── Does NOT manage containers not created by K8s              │
  │                                                                  │
  │  kube-proxy                                                      │
  │  ├── Maintains network rules on each node                       │
  │  ├── Implements Service abstraction                             │
  │  ├── Modes: iptables (default), IPVS (high perf), nftables     │
  │  └── Handles ClusterIP, NodePort, LoadBalancer routing          │
  │                                                                  │
  │  Container Runtime                                               │
  │  ├── Runs containers via CRI (Container Runtime Interface)      │
  │  ├── containerd (default since K8s 1.24)                        │
  │  ├── CRI-O (Red Hat / OpenShift)                                │
  │  └── Docker removed in K8s 1.24 ("Dockershim deprecation")     │
  │      └── Docker images still work! Only the runtime changed     │
  └──────────────────────────────────────────────────────────────────┘
```

### 2.4 How kubectl Commands Flow Through the Cluster

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  FLOW: kubectl apply -f deployment.yaml                         │
  │                                                                  │
  │  ┌──────────┐                                                    │
  │  │ kubectl  │                                                    │
  │  │ (client) │                                                    │
  │  └────┬─────┘                                                    │
  │       │ 1. HTTP POST to API server (YAML → JSON)                │
  │       ▼                                                          │
  │  ┌──────────────┐                                                │
  │  │ kube-apiserver│                                               │
  │  │ (port 6443)  │                                                │
  │  └────┬─────────┘                                                │
  │       │ 2. AuthN → AuthZ → Admission Controllers → Validation   │
  │       │ 3. Persist object to etcd                                │
  │       ▼                                                          │
  │  ┌──────────┐                                                    │
  │  │  etcd    │  (Deployment object stored)                       │
  │  └────┬─────┘                                                    │
  │       │ 4. Deployment controller (watch loop) detects new object │
  │       ▼                                                          │
  │  ┌──────────────────┐                                            │
  │  │ Controller       │  5. Creates ReplicaSet object              │
  │  │ Manager          │  6. ReplicaSet controller creates Pod objs │
  │  └────┬─────────────┘                                            │
  │       │ 7. Scheduler watches for unscheduled pods               │
  │       ▼                                                          │
  │  ┌──────────────┐                                                │
  │  │ kube-scheduler│  8. Assigns pod to Node 2 (best fit)         │
  │  └────┬─────────┘                                                │
  │       │ 9. Updates pod spec with nodeName in etcd               │
  │       ▼                                                          │
  │  ┌──────────────┐                                                │
  │  │ kubelet      │  10. Watches API server, sees pod assigned    │
  │  │ (Node 2)     │  11. Pulls image via container runtime       │
  │  │              │  12. Starts container, reports status          │
  │  └──────────────┘                                                │
  │                                                                  │
  │  Total time: ~5-30 seconds (depending on image pull)            │
  └──────────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha (etcd):** etcd is the **single source of truth** for your entire cluster. If etcd is lost and there's no backup, you lose your cluster. Always configure automated etcd snapshots:
> ```bash
> # Manual etcd snapshot
> ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db \
>   --endpoints=https://127.0.0.1:2379 \
>   --cacert=/etc/kubernetes/pki/etcd/ca.crt \
>   --cert=/etc/kubernetes/pki/etcd/server.crt \
>   --key=/etc/kubernetes/pki/etcd/server.key
> ```

### 2.5 Kubernetes Networking Model — The 4 Rules

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  KUBERNETES NETWORKING — FUNDAMENTAL RULES                      │
  │                                                                  │
  │  Rule 1: Every Pod gets its own unique IP address               │
  │  ┌─────────┐  ┌─────────┐  ┌─────────┐                        │
  │  │  Pod A  │  │  Pod B  │  │  Pod C  │                         │
  │  │10.244.  │  │10.244.  │  │10.244.  │                         │
  │  │ 1.5    │  │ 1.6    │  │ 2.3    │                         │
  │  └─────────┘  └─────────┘  └─────────┘                        │
  │                                                                  │
  │  Rule 2: All Pods can communicate with all other Pods           │
  │          WITHOUT NAT (flat network)                              │
  │                                                                  │
  │  Rule 3: All Nodes can communicate with all Pods (and reverse)  │
  │          WITHOUT NAT                                             │
  │                                                                  │
  │  Rule 4: The IP a Pod sees for itself is the same IP other     │
  │          Pods see for it                                         │
  │                                                                  │
  │  Implementation: CNI Plugins                                     │
  │  ┌──────────┬──────────┬───────────┬──────────────┐            │
  │  │ Calico   │ Cilium   │ Flannel   │ AWS VPC CNI  │            │
  │  ├──────────┼──────────┼───────────┼──────────────┤            │
  │  │ BGP-based│ eBPF-    │ VXLAN     │ Assigns real │            │
  │  │ routing  │ based    │ overlay   │ VPC IPs to   │            │
  │  │ Network  │ No       │ Simple    │ pods         │            │
  │  │ policies │ iptables │ No net-   │ Native cloud │            │
  │  │ ✅       │ policies │ work      │ integration  │            │
  │  │          │ ✅       │ policies  │ ✅           │            │
  │  └──────────┴──────────┴───────────┴──────────────┘            │
  └──────────────────────────────────────────────────────────────────┘
```

> **DevOps Relevance:** CNI choice matters in production — Calico is the most common, Cilium is rising fast due to eBPF (better performance, no iptables), and AWS VPC CNI is mandatory on EKS if you need pods to have VPC-routable IPs. Choose your CNI BEFORE going to production — switching later is painful.

---

## 3. Pods & Container Basics

### 3.1 What Is a Pod?

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  POD — THE SMALLEST DEPLOYABLE UNIT IN K8s                      │
  │                                                                  │
  │  A Pod = one or more containers that:                            │
  │  • Share the same network namespace (same IP, same ports)       │
  │  • Share the same storage volumes                                │
  │  • Are always co-located on the same node                       │
  │  • Are always co-scheduled (start/stop together)                │
  │                                                                  │
  │  ┌─── Pod (IP: 10.244.1.5) ───────────────────────────┐        │
  │  │                                                      │        │
  │  │  ┌────────────┐  ┌────────────┐                     │        │
  │  │  │ Container 1│  │ Container 2│ (optional sidecar)  │        │
  │  │  │ (app)      │  │ (logging)  │                     │        │
  │  │  │ port 8080  │  │ port 9090  │                     │        │
  │  │  └────────────┘  └────────────┘                     │        │
  │  │       │                │                             │        │
  │  │  ┌────┴────────────────┴────┐                       │        │
  │  │  │  Shared Volume (emptyDir)│                       │        │
  │  │  └──────────────────────────┘                       │        │
  │  │                                                      │        │
  │  │  Network: localhost communication between containers │        │
  │  │  Lifecycle: containers restart independently         │        │
  │  │  Node: always same node                              │        │
  │  └──────────────────────────────────────────────────────┘        │
  │                                                                  │
  │  IMPORTANT: In production, you almost NEVER create pods         │
  │  directly. Use Deployments, StatefulSets, or DaemonSets.       │
  └──────────────────────────────────────────────────────────────────┘
```

### 3.2 Pod YAML Anatomy

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: default
  labels:                        # Labels = how K8s finds/selects pods
    app: my-app
    version: v1
    tier: frontend
  annotations:                   # Annotations = non-identifying metadata
    prometheus.io/scrape: "true"
spec:
  containers:
  - name: app
    image: nginx:1.25-alpine    # Always pin versions! Never use :latest
    ports:
    - containerPort: 80
    resources:                   # ALWAYS set resource requests/limits
      requests:
        cpu: 100m               # 100 millicores = 0.1 CPU
        memory: 128Mi           # 128 MiB
      limits:
        cpu: 500m
        memory: 256Mi
    livenessProbe:              # Is the container alive?
      httpGet:
        path: /healthz
        port: 80
      initialDelaySeconds: 15
      periodSeconds: 10
    readinessProbe:             # Is the container ready for traffic?
      httpGet:
        path: /ready
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
    startupProbe:               # Slow-starting apps (K8s 1.18+)
      httpGet:
        path: /healthz
        port: 80
      failureThreshold: 30
      periodSeconds: 10
  restartPolicy: Always          # Always | OnFailure | Never
```

### 3.3 Pod Lifecycle

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  POD LIFECYCLE PHASES                                            │
  │                                                                  │
  │  ┌────────┐   ┌────────┐   ┌────────┐   ┌────────────────┐    │
  │  │Pending │──▶│Running │──▶│Succeed │   │  Termination   │    │
  │  │        │   │        │   │(if Job)│   │  Sequence:     │    │
  │  │Waiting │   │At least│   └────────┘   │                │    │
  │  │for     │   │one     │                │ 1. preStop hook│    │
  │  │schedule│   │container│   ┌────────┐  │ 2. SIGTERM     │    │
  │  │+ image │   │running │──▶│Failed  │   │ 3. Grace (30s) │    │
  │  │pull    │   │        │   │        │   │ 4. SIGKILL     │    │
  │  └────────┘   └────────┘   └────────┘   └────────────────┘    │
  │                                                                  │
  │  Container States:                                               │
  │  ├── Waiting:    Pulling image, creating container              │
  │  ├── Running:    Container is executing                          │
  │  └── Terminated: Container exited (exit code 0 = success)      │
  │                                                                  │
  │  Restart Policies:                                               │
  │  ├── Always:     Restart any terminated container (default)     │
  │  ├── OnFailure:  Only restart if exit code ≠ 0                 │
  │  └── Never:      Never restart (useful for Jobs)                │
  │                                                                  │
  │  Backoff: Restarts use exponential backoff                      │
  │  (10s, 20s, 40s, 80s, 160s, max 5 min)                        │
  └──────────────────────────────────────────────────────────────────┘
```

### 3.4 Multi-Container Pod Patterns

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  MULTI-CONTAINER PATTERNS                                        │
  │                                                                  │
  │  SIDECAR PATTERN (most common)                                   │
  │  ┌──────────────────────────────────────┐                       │
  │  │  Pod                                  │                       │
  │  │  ┌──────────┐    ┌──────────┐         │                       │
  │  │  │  App     │───▶│  Log     │         │                       │
  │  │  │  Server  │    │ Shipper  │         │                       │
  │  │  │  (main)  │    │(fluentd) │         │                       │
  │  │  └──────────┘    └──────────┘         │                       │
  │  │       └── shared volume ──┘           │                       │
  │  └──────────────────────────────────────┘                       │
  │  Use: Log collection, monitoring agents, service mesh proxies   │
  │                                                                  │
  │  INIT CONTAINER PATTERN                                          │
  │  ┌──────────────────────────────────────┐                       │
  │  │  Pod                                  │                       │
  │  │  ┌──────────┐    ┌──────────┐         │                       │
  │  │  │  Init    │──▶│  App     │         │                       │
  │  │  │ Container│    │  Server  │         │                       │
  │  │  │ (runs    │    │ (starts  │         │                       │
  │  │  │  first,  │    │  after   │         │                       │
  │  │  │  exits)  │    │  init)   │         │                       │
  │  │  └──────────┘    └──────────┘         │                       │
  │  └──────────────────────────────────────┘                       │
  │  Use: DB migration, config download, wait for dependency       │
  │                                                                  │
  │  AMBASSADOR PATTERN                                              │
  │  ┌──────────────────────────────────────┐                       │
  │  │  Pod                                  │                       │
  │  │  ┌──────────┐    ┌──────────┐         │                       │
  │  │  │  App     │───▶│ Ambassador│        │                       │
  │  │  │ (talks to│    │ (proxy to│         │                       │
  │  │  │localhost)│    │ external │         │                       │
  │  │  │          │    │ service) │         │                       │
  │  │  └──────────┘    └──────────┘         │                       │
  │  └──────────────────────────────────────┘                       │
  │  Use: DB connection proxy, API gateway, caching proxy           │
  └──────────────────────────────────────────────────────────────────┘
```

### 3.5 Health Probes — Liveness vs Readiness vs Startup

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  HEALTH PROBES COMPARISON                                        │
  │                                                                  │
  │  Probe     │ Question it answers  │ Failure action │ When       │
  │  ──────────┼──────────────────────┼────────────────┼──────────  │
  │  Startup   │ Has the app finished │ Keep waiting   │ At boot    │
  │            │ starting up?         │ (kill after    │ only       │
  │            │                      │ threshold)     │            │
  │  ──────────┼──────────────────────┼────────────────┼──────────  │
  │  Liveness  │ Is the app alive     │ RESTART the    │ After      │
  │            │ (not deadlocked)?    │ container      │ startup    │
  │  ──────────┼──────────────────────┼────────────────┼──────────  │
  │  Readiness │ Can the app serve    │ REMOVE from    │ Continuous │
  │            │ traffic?             │ Service (no    │            │
  │            │                      │ traffic sent)  │            │
  │                                                                  │
  │  Probe Types:                                                    │
  │  ├── httpGet:     HTTP GET → 200-399 = success                  │
  │  ├── tcpSocket:   TCP connect → open = success                  │
  │  ├── exec:        Run command → exit 0 = success                │
  │  └── grpc:        gRPC health check (K8s 1.24+)                │
  │                                                                  │
  │  TIMING:                                                         │
  │   Boot ──▶ startupProbe ──▶ livenessProbe (loop)               │
  │                             readinessProbe (loop)               │
  └──────────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha (Liveness Probes):** Don't make your liveness probe check external dependencies (like a database). If the DB is down and the liveness probe fails, K8s will restart your pod — which won't fix the DB issue and will cause cascading restarts. **Liveness = is MY process healthy?** Use readiness probes for dependency checks.

> **⚠️ Gotcha (:latest tag):** Never use `image: nginx:latest` in production. The `:latest` tag is mutable — a new push can change what `:latest` points to. Pin to a specific version (`nginx:1.25.3`) or use image digests (`nginx@sha256:abc123...`). Set `imagePullPolicy: IfNotPresent` for pinned tags.

---

## 4. ReplicaSets & Deployments

### 4.1 Why Not Just Pods?

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  BARE PODS vs MANAGED WORKLOADS                                  │
  │                                                                  │
  │  Bare Pod:                        Deployment (manages pods):    │
  │  ┌──────┐                         ┌──────────────────────┐     │
  │  │ Pod  │ ← If it dies,           │ Deployment           │     │
  │  │      │   nobody recreates it    │ ├── ReplicaSet       │     │
  │  └──────┘                          │ │   ├── Pod 1 ✅     │     │
  │                                    │ │   ├── Pod 2 ✅     │     │
  │  ❌ No self-healing                │ │   └── Pod 3 ✅     │     │
  │  ❌ No scaling                     │ │       (if one dies,│     │
  │  ❌ No rolling updates             │ │       a new one    │     │
  │  ❌ No rollback                    │ │       is created)  │     │
  │                                    │ └────────────────────│     │
  │                                    └──────────────────────┘     │
  │                                    ✅ Self-healing              │
  │                                    ✅ Scaling (replicas: N)     │
  │                                    ✅ Rolling updates            │
  │                                    ✅ Rollback (revision hist)   │
  └──────────────────────────────────────────────────────────────────┘
```

### 4.2 ReplicaSet — Ensuring N Copies

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  REPLICASET — MAINTAINS DESIRED NUMBER OF POD REPLICAS          │
  │                                                                  │
  │  Spec: replicas: 3                                               │
  │                                                                  │
  │  Desired: 3    Actual: 3    → Do nothing                        │
  │  ┌──────┐ ┌──────┐ ┌──────┐                                    │
  │  │Pod 1 │ │Pod 2 │ │Pod 3 │                                    │
  │  └──────┘ └──────┘ └──────┘                                    │
  │                                                                  │
  │  Desired: 3    Actual: 2    → Create 1 more pod                 │
  │  ┌──────┐ ┌──────┐ ┌──────┐                                    │
  │  │Pod 1 │ │Pod 2 │ │ NEW! │                                    │
  │  └──────┘ └──────┘ └──────┘                                    │
  │                                                                  │
  │  Desired: 3    Actual: 4    → Terminate 1 pod                   │
  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐                          │
  │  │Pod 1 │ │Pod 2 │ │Pod 3 │ │ DEL  │                          │
  │  └──────┘ └──────┘ └──────┘ └──────┘                          │
  │                                                                  │
  │  Selection: ReplicaSet uses label selectors to find its pods    │
  │  matchLabels:                                                    │
  │    app: my-app  ← all pods with this label are counted          │
  │                                                                  │
  │  NOTE: You almost never create ReplicaSets directly.            │
  │        Deployments manage ReplicaSets for you.                  │
  └──────────────────────────────────────────────────────────────────┘
```

### 4.3 Deployment — The Standard Workload Controller

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
spec:
  replicas: 3
  revisionHistoryLimit: 10        # Keep 10 old ReplicaSets for rollback
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate           # or Recreate
    rollingUpdate:
      maxSurge: 1                 # Max pods ABOVE desired during update
      maxUnavailable: 0           # Max pods BELOW desired (0 = zero downtime)
  template:                       # Pod template
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: my-app:v2.1.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
```

### 4.4 Deployment Strategies

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  DEPLOYMENT STRATEGIES                                           │
  │                                                                  │
  │  ROLLING UPDATE (default — zero downtime)                        │
  │                                                                  │
  │  Step 1: v1 v1 v1           (3 old pods running)                │
  │  Step 2: v1 v1 v1 v2        (1 new pod created, maxSurge=1)    │
  │  Step 3: v1 v1 v2           (old pod terminated after ready)    │
  │  Step 4: v1 v1 v2 v2        (another new pod)                  │
  │  Step 5: v1 v2 v2           (another old terminated)            │
  │  Step 6: v1 v2 v2 v2        (last new pod)                     │
  │  Step 7: v2 v2 v2           (last old terminated) ✅ Done       │
  │                                                                  │
  │  RECREATE (downtime — use for stateful apps that can't coexist) │
  │                                                                  │
  │  Step 1: v1 v1 v1           (3 old pods running)                │
  │  Step 2: __ __ __           (ALL old pods terminated) ⚠️ DOWN  │
  │  Step 3: v2 v2 v2           (ALL new pods created)              │
  │                                                                  │
  │  BLUE-GREEN (via Service label switch — not built-in)           │
  │                                                                  │
  │  ┌──────────┐   Service selector: version=blue                  │
  │  │ "Blue"   │   ┌──────┐ ┌──────┐ ┌──────┐                    │
  │  │ Deploy   │───│ v1   │ │ v1   │ │ v1   │  ← live traffic    │
  │  └──────────┘   └──────┘ └──────┘ └──────┘                    │
  │  ┌──────────┐                                                    │
  │  │ "Green"  │   ┌──────┐ ┌──────┐ ┌──────┐                    │
  │  │ Deploy   │───│ v2   │ │ v2   │ │ v2   │  ← staging/test    │
  │  └──────────┘   └──────┘ └──────┘ └──────┘                    │
  │                                                                  │
  │  Switch: Update Service selector → version=green                │
  │  Rollback: Switch selector back to version=blue                 │
  │                                                                  │
  │  CANARY (gradual traffic shift — not built-in)                  │
  │                                                                  │
  │  ┌──────────┐   ┌──────┐ ┌──────┐ ┌──────┐   90% traffic     │
  │  │ Stable   │───│ v1   │ │ v1   │ │ v1   │                    │
  │  └──────────┘   └──────┘ └──────┘ └──────┘                    │
  │  ┌──────────┐   ┌──────┐                       10% traffic     │
  │  │ Canary   │───│ v2   │                                        │
  │  └──────────┘   └──────┘                                        │
  │                                                                  │
  │  Implement with: Istio, Argo Rollouts, Flagger, or Ingress    │
  │  weight-based routing                                           │
  └──────────────────────────────────────────────────────────────────┘
```

### 4.5 Essential Deployment Commands

```bash
# Create / Apply
kubectl apply -f deployment.yaml

# Check status
kubectl rollout status deployment/my-app
kubectl get deploy my-app -o wide

# Scale
kubectl scale deployment my-app --replicas=5

# Update image (triggers rolling update)
kubectl set image deployment/my-app app=my-app:v2.2.0

# View rollout history
kubectl rollout history deployment/my-app

# Rollback to previous version
kubectl rollout undo deployment/my-app

# Rollback to specific revision
kubectl rollout undo deployment/my-app --to-revision=3

# Pause/Resume rollout (for canary-like control)
kubectl rollout pause deployment/my-app
kubectl rollout resume deployment/my-app

# Watch pods during rollout
kubectl get pods -l app=my-app -w
```

### 4.6 Horizontal Pod Autoscaler (HPA)

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  HPA — AUTOMATIC HORIZONTAL SCALING                              │
  │                                                                  │
  │  Replicas                                                        │
  │  ▲                                                               │
  │  │  10 ─ ─ ─ ─ ─ ─ ─ ─ ─ maxReplicas                          │
  │  │             ┌──────┐                                          │
  │  │          ┌──┘      └──┐     ← scales with CPU/memory/       │
  │  │       ┌──┘            └──┐      custom metrics               │
  │  │   ┌───┘                  └──┐                                 │
  │  │   2 ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ minReplicas                     │
  │  └──────────────────────────────────────────────────────────────▶│
  │                          Time                                    │
  │                                                                  │
  │  Formula:                                                        │
  │  desiredReplicas = ceil(currentReplicas ×                       │
  │                    (currentMetric / targetMetric))               │
  │                                                                  │
  │  Example: 3 pods, CPU at 90%, target 50%                        │
  │  = ceil(3 × (90/50)) = ceil(5.4) = 6 pods                     │
  └──────────────────────────────────────────────────────────────────┘
```

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70        # Scale when avg CPU > 70%
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scaling down
      policies:
      - type: Percent
        value: 10                      # Max 10% reduction per interval
        periodSeconds: 60
```

> **⚠️ Gotcha (HPA + Resource Requests):** HPA requires `resources.requests` to be set on your containers. Without requests, K8s cannot calculate CPU/memory utilization percentages. If requests are missing, HPA silently does nothing. Always check: `kubectl describe hpa my-app-hpa` — look for "unable to calculate" errors.

> **⚠️ Gotcha (HPA + VPA Conflict):** Do NOT use HPA and VPA (Vertical Pod Autoscaler) on the same metric. HPA scales horizontally (more pods), VPA scales vertically (more CPU/memory per pod). If both target CPU, they'll fight. Use HPA for CPU + VPA for memory, or use Multidimensional Pod Autoscaling.

> **DevOps Relevance:** For production deployments, always use Deployments (never bare pods), set `maxUnavailable: 0` for zero-downtime updates, implement readiness probes so traffic only goes to ready pods, and configure HPA for auto-scaling. This is the baseline expectation for any production K8s workload.

---

## 5. Services & Service Discovery

### 5.1 Why Services? — The Pod IP Problem

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  THE PROBLEM: POD IPs ARE EPHEMERAL                              │
  │                                                                  │
  │  Pods come and go — IPs change with every restart/reschedule:   │
  │                                                                  │
  │  Time T1:                     Time T2 (pod restarted):          │
  │  ┌──────────────┐             ┌──────────────┐                  │
  │  │ my-app pod   │             │ my-app pod   │                  │
  │  │ 10.244.1.5   │  ──died──▶  │ 10.244.2.9   │  ← NEW IP!     │
  │  └──────────────┘             └──────────────┘                  │
  │                                                                  │
  │  The Solution: SERVICES                                          │
  │  ┌──────────────────────────────────────────────────────┐       │
  │  │  Service: my-app-svc                                  │       │
  │  │  ClusterIP: 10.96.45.12  (STABLE — never changes)   │       │
  │  │  DNS: my-app-svc.default.svc.cluster.local           │       │
  │  │                                                       │       │
  │  │  Endpoints (auto-updated by endpoint controller):    │       │
  │  │  ├── 10.244.1.5:8080  (Pod 1)                       │       │
  │  │  ├── 10.244.2.3:8080  (Pod 2)                       │       │
  │  │  └── 10.244.1.8:8080  (Pod 3)                       │       │
  │  │                                                       │       │
  │  │  Traffic → Service VIP → random healthy pod          │       │
  │  └──────────────────────────────────────────────────────┘       │
  └──────────────────────────────────────────────────────────────────┘
```

### 5.2 Service Types

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  KUBERNETES SERVICE TYPES                                        │
  │                                                                  │
  │  TYPE 1: ClusterIP (default — internal only)                    │
  │  ┌──────────────────────────────────────────────────┐           │
  │  │  Accessible ONLY within the cluster               │           │
  │  │  ClusterIP: 10.96.X.X (virtual IP)               │           │
  │  │  Use for: inter-service communication             │           │
  │  │                                                    │           │
  │  │  frontend-pod ──▶ backend-svc:8080 ──▶ backend-pod│          │
  │  └──────────────────────────────────────────────────┘           │
  │                                                                  │
  │  TYPE 2: NodePort (exposes on every node's IP)                  │
  │  ┌──────────────────────────────────────────────────┐           │
  │  │  Accessible via <NodeIP>:<NodePort>               │           │
  │  │  NodePort range: 30000-32767                      │           │
  │  │  Also creates a ClusterIP automatically           │           │
  │  │                                                    │           │
  │  │  External ──▶ Node1:30080 ──▶ Service ──▶ Pod     │           │
  │  │  External ──▶ Node2:30080 ──▶ Service ──▶ Pod     │           │
  │  │  Use for: Development, testing, on-prem without LB│           │
  │  └──────────────────────────────────────────────────┘           │
  │                                                                  │
  │  TYPE 3: LoadBalancer (cloud provider LB)                       │
  │  ┌──────────────────────────────────────────────────┐           │
  │  │  Provisions a CLOUD load balancer (ALB/NLB/GLB)  │           │
  │  │  Gets external IP from cloud provider             │           │
  │  │  Also creates NodePort + ClusterIP                │           │
  │  │                                                    │           │
  │  │  Internet ──▶ Cloud LB ──▶ NodePort ──▶ Pod      │           │
  │  │  Use for: Exposing single service externally      │           │
  │  │  ⚠️ Each LoadBalancer = separate cloud LB = $$$  │           │
  │  └──────────────────────────────────────────────────┘           │
  │                                                                  │
  │  TYPE 4: ExternalName (DNS CNAME alias)                         │
  │  ┌──────────────────────────────────────────────────┐           │
  │  │  Maps service to an external DNS name             │           │
  │  │  No proxying — just DNS CNAME record              │           │
  │  │                                                    │           │
  │  │  my-db-svc ──CNAME──▶ mydb.rds.amazonaws.com     │           │
  │  │  Use for: Referencing external services via K8s   │           │
  │  │  DNS (easy migration from external → internal)    │           │
  │  └──────────────────────────────────────────────────┘           │
  └──────────────────────────────────────────────────────────────────┘
```

### 5.3 Service YAML Examples

```yaml
# ClusterIP Service (internal)
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  type: ClusterIP                  # default — can omit
  selector:
    app: backend                   # matches pods with label app=backend
  ports:
  - port: 80                      # Service port (what clients connect to)
    targetPort: 8080               # Container port (where app listens)
    protocol: TCP
---
# LoadBalancer Service (external)
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
  annotations:
    # AWS-specific: use NLB instead of classic LB
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
    # Internal LB (not internet-facing)
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
  - port: 443
    targetPort: 8443
```

### 5.4 DNS-Based Service Discovery

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  KUBERNETES DNS (CoreDNS)                                        │
  │                                                                  │
  │  Every Service gets a DNS record automatically:                 │
  │                                                                  │
  │  Format: <service>.<namespace>.svc.cluster.local                │
  │                                                                  │
  │  Examples:                                                       │
  │  ├── backend-svc.default.svc.cluster.local                      │
  │  ├── backend-svc.default    (short form — same namespace)       │
  │  ├── backend-svc            (shortest — same namespace)         │
  │  └── redis.database.svc.cluster.local  (cross-namespace)       │
  │                                                                  │
  │  Pod DNS:                                                        │
  │  <pod-ip-dashed>.<namespace>.pod.cluster.local                  │
  │  10-244-1-5.default.pod.cluster.local                           │
  │                                                                  │
  │  ┌─ Inside a pod ────────────────────────────────────────┐      │
  │  │  # Same namespace — just service name                  │      │
  │  │  curl http://backend-svc:8080/api                     │      │
  │  │                                                        │      │
  │  │  # Cross namespace — add namespace                     │      │
  │  │  curl http://backend-svc.staging:8080/api             │      │
  │  │                                                        │      │
  │  │  # FQDN — explicit                                    │      │
  │  │  curl http://backend-svc.staging.svc.cluster.local    │      │
  │  └────────────────────────────────────────────────────────┘      │
  │                                                                  │
  │  Headless Service (clusterIP: None):                             │
  │  ├── No virtual IP assigned                                     │
  │  ├── DNS returns individual pod IPs directly                    │
  │  ├── Used by StatefulSets for stable pod DNS                    │
  │  └── Example: pod-0.my-svc.default.svc.cluster.local           │
  └──────────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha (Service + No Endpoints):** If your Service selector doesn't match any pod labels, the service will have **zero endpoints** and all traffic will fail silently ( connection refused). Debug with: `kubectl get endpoints my-svc` — if it shows `<none>`, your selector/labels don't match. This is the #1 K8s networking mistake.

> **⚠️ Gotcha (LoadBalancer cost):** Each `type: LoadBalancer` creates a **separate cloud load balancer** (~$16-25/month on AWS). If you have 20 services, that's $400+/month just for LBs. Use **Ingress** (Section 9) to share one LB across multiple services — route by hostname/path instead.

---

## 6. Namespaces & Resource Organization

### 6.1 What Are Namespaces?

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  NAMESPACES — VIRTUAL CLUSTERS WITHIN A CLUSTER                 │
  │                                                                  │
  │  ┌─── K8s Cluster ─────────────────────────────────────────┐    │
  │  │                                                          │    │
  │  │  ┌─ default ──────┐  ┌─ staging ──────┐               │    │
  │  │  │ app-deploy     │  │ app-deploy     │                │    │
  │  │  │ app-svc        │  │ app-svc        │  Same names!  │    │
  │  │  │ redis-deploy   │  │ redis-deploy   │  No conflict  │    │
  │  │  └────────────────┘  └────────────────┘                │    │
  │  │                                                          │    │
  │  │  ┌─ production ───┐  ┌─ monitoring ───┐               │    │
  │  │  │ app-deploy     │  │ prometheus     │                │    │
  │  │  │ app-svc        │  │ grafana        │                │    │
  │  │  │ db-statefulset │  │ alertmanager   │                │    │
  │  │  └────────────────┘  └────────────────┘                │    │
  │  │                                                          │    │
  │  └──────────────────────────────────────────────────────────┘    │
  │                                                                  │
  │  Default namespaces (always present):                            │
  │  ├── default:          User workloads (if no ns specified)      │
  │  ├── kube-system:      K8s system components (API server, etc)  │
  │  ├── kube-public:      Publicly accessible data (cluster-info)  │
  │  └── kube-node-lease:  Node heartbeat leases                    │
  │                                                                  │
  │  NOT namespaced (cluster-wide):                                  │
  │  ├── Nodes, PersistentVolumes, ClusterRoles                     │
  │  ├── Namespaces themselves, StorageClasses                       │
  │  └── IngressClasses, PriorityClasses                             │
  └──────────────────────────────────────────────────────────────────┘
```

### 6.2 Namespace Best Practices

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  NAMESPACE ORGANIZATION PATTERNS                                 │
  │                                                                  │
  │  PATTERN 1: By Environment                                      │
  │  ├── dev, staging, production                                   │
  │  └── Simple, but not recommended for prod (use separate clusters)│
  │                                                                  │
  │  PATTERN 2: By Team / Squad (recommended for multi-team)        │
  │  ├── team-frontend, team-backend, team-data                     │
  │  └── Each team owns their namespace, applies RBAC               │
  │                                                                  │
  │  PATTERN 3: By Application                                      │
  │  ├── ecommerce-app, payment-service, auth-service               │
  │  └── Good for microservices isolation                            │
  │                                                                  │
  │  PATTERN 4: System vs User                                      │
  │  ├── kube-system, monitoring, logging, ingress (system)         │
  │  ├── app-prod, app-staging (user workloads)                     │
  │  └── Clear separation of platform vs application                │
  └──────────────────────────────────────────────────────────────────┘
```

### 6.3 Resource Quotas & Limit Ranges

```yaml
# ResourceQuota — cap total resources per namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-frontend-quota
  namespace: team-frontend
spec:
  hard:
    requests.cpu: "10"             # Max 10 CPU cores requested
    requests.memory: 20Gi          # Max 20 GiB memory requested
    limits.cpu: "20"               # Max 20 CPU cores limit
    limits.memory: 40Gi            # Max 40 GiB memory limit
    pods: "50"                     # Max 50 pods
    services: "20"                 # Max 20 services
    persistentvolumeclaims: "10"   # Max 10 PVCs
    configmaps: "30"               # Max 30 configmaps
    secrets: "30"                  # Max 30 secrets
---
# LimitRange — default/min/max per container
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: team-frontend
spec:
  limits:
  - type: Container
    default:                       # Default LIMITS if not specified
      cpu: 500m
      memory: 256Mi
    defaultRequest:                # Default REQUESTS if not specified
      cpu: 100m
      memory: 128Mi
    min:                           # Minimum allowed
      cpu: 50m
      memory: 64Mi
    max:                           # Maximum allowed
      cpu: "2"
      memory: 2Gi
```

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  RESOURCE QUOTAS vs LIMIT RANGES                                 │
  │                                                                  │
  │  ResourceQuota:                  LimitRange:                     │
  │  ├── Scope: ENTIRE namespace    ├── Scope: PER container/pod   │
  │  ├── Caps total resource use    ├── Sets default request/limit │
  │  ├── "Budget for the team"      ├── "Guardrails per workload"  │
  │  └── Fails pod creation if     └── Auto-applies if missing    │
  │      quota exceeded                                              │
  │                                                                  │
  │  USE BOTH TOGETHER:                                              │
  │  LimitRange ensures every pod has requests/limits set           │
  │  ResourceQuota ensures the namespace doesn't consume too much   │
  └──────────────────────────────────────────────────────────────────┘
```

### 6.4 Essential Namespace Commands

```bash
# Create namespace
kubectl create namespace staging

# List namespaces
kubectl get namespaces

# Set default namespace for kubectl context
kubectl config set-context --current --namespace=staging

# List pods in all namespaces
kubectl get pods --all-namespaces    # or -A

# Apply resource to a specific namespace
kubectl apply -f deploy.yaml -n production

# View resource quota usage
kubectl describe resourcequota -n team-frontend

# Delete namespace (⚠️ deletes ALL resources inside!)
kubectl delete namespace staging
```

> **⚠️ Gotcha (Namespace Deletion):** Deleting a namespace deletes **EVERYTHING** inside it — pods, services, configmaps, secrets, PVCs, deployments. There is no undo. If a namespace gets stuck in `Terminating` state, check for finalizers: `kubectl get ns staging -o json | jq '.spec.finalizers'` — resource finalizers can block deletion indefinitely.

> **DevOps Relevance:** In production multi-tenant clusters, always use namespaces + ResourceQuotas + LimitRanges + NetworkPolicies. This prevents one team from consuming all cluster resources ("noisy neighbor" problem) and provides security boundaries. Combine with RBAC (Section 12) to restrict who can do what per namespace.

---

## 7. ConfigMaps & Secrets

### 7.1 ConfigMaps — Externalizing Configuration

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  CONFIGMAPS — DECOUPLE CONFIG FROM CONTAINER IMAGES             │
  │                                                                  │
  │  Problem: Hardcoding config in images means rebuilding for      │
  │  every environment (dev/staging/prod). ConfigMaps externalize   │
  │  config so the SAME image works everywhere.                     │
  │                                                                  │
  │  ┌── Image ──────┐   ┌── ConfigMap ──────────────────────┐     │
  │  │  my-app:v2.0  │ + │  DATABASE_HOST=db.prod.internal  │     │
  │  │  (code only)  │   │  LOG_LEVEL=info                   │     │
  │  │               │   │  MAX_CONNECTIONS=100               │     │
  │  └───────────────┘   └───────────────────────────────────┘     │
  │         │                         │                              │
  │         └──────── Pod ────────────┘                              │
  │                                                                  │
  │  Injection methods:                                              │
  │  ├── Environment variables (simple key-value)                   │
  │  ├── Volume mount (entire file, e.g., nginx.conf)              │
  │  └── Command-line arguments                                     │
  └──────────────────────────────────────────────────────────────────┘
```

```yaml
# ConfigMap from literal values
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  DATABASE_HOST: "db.prod.internal"
  DATABASE_PORT: "5432"
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
  # Multi-line config file
  nginx.conf: |
    server {
        listen 80;
        server_name example.com;
        location / {
            proxy_pass http://backend:8080;
        }
    }
---
# Using ConfigMap in a Pod
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: app
    image: my-app:v2.0
    # Method 1: As environment variables
    envFrom:
    - configMapRef:
        name: app-config              # Injects ALL keys as env vars
    # Method 2: Cherry-pick specific keys
    env:
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DATABASE_HOST
    # Method 3: As a volume (file mount)
    volumeMounts:
    - name: config-volume
      mountPath: /etc/nginx/nginx.conf
      subPath: nginx.conf              # Mount single file, not dir
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

### 7.2 Secrets — Sensitive Data

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  SECRETS — LIKE CONFIGMAPS BUT FOR SENSITIVE DATA               │
  │                                                                  │
  │  ⚠️ CRITICAL: K8s Secrets are base64-ENCODED, NOT encrypted!   │
  │     Anyone with API access can decode them.                     │
  │     base64 is NOT security — it's just encoding.                │
  │                                                                  │
  │  Secret Types:                                                   │
  │  ┌─────────────────────┬────────────────────────────────────┐   │
  │  │ Type                │ Use Case                            │   │
  │  ├─────────────────────┼────────────────────────────────────┤   │
  │  │ Opaque (default)    │ Generic key-value (passwords, keys)│   │
  │  │ kubernetes.io/tls   │ TLS certificate + key               │   │
  │  │ kubernetes.io/      │ Docker registry credentials        │   │
  │  │  dockerconfigjson   │                                     │   │
  │  │ kubernetes.io/      │ ServiceAccount token (legacy)      │   │
  │  │  service-account-   │                                     │   │
  │  │  token              │                                     │   │
  │  │ kubernetes.io/      │ Basic auth credentials             │   │
  │  │  basic-auth         │                                     │   │
  │  └─────────────────────┴────────────────────────────────────┘   │
  └──────────────────────────────────────────────────────────────────┘
```

```yaml
# Create Secret
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: production
type: Opaque
data:                              # base64-encoded values
  username: YWRtaW4=               # echo -n "admin" | base64
  password: cEBzc3cwcmQxMjM=      # echo -n "p@ssw0rd123" | base64
# OR use stringData (plain text — K8s encodes for you)
stringData:
  api-key: "sk-live-abc123def456"
---
# TLS Secret
apiVersion: v1
kind: Secret
metadata:
  name: tls-cert
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
```

### 7.3 Secrets Management — Production Solutions

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  SECRETS IN PRODUCTION — DON'T USE PLAIN K8s SECRETS ALONE     │
  │                                                                  │
  │  Problem: K8s secrets stored unencrypted in etcd by default     │
  │                                                                  │
  │  SOLUTION 1: Encrypt etcd at Rest                               │
  │  ├── EncryptionConfiguration with aescbc or kms provider       │
  │  ├── Managed K8s (EKS/AKS/GKE) does this by default           │
  │  └── Self-managed: configure API server --encryption-provider  │
  │                                                                  │
  │  SOLUTION 2: External Secrets Operator (ESO)                    │
  │  ┌──────────────────────────────────────────────────────┐      │
  │  │  AWS Secrets Manager ──▶ ESO ──▶ K8s Secret          │      │
  │  │  Azure Key Vault     ──▶ ESO ──▶ K8s Secret          │      │
  │  │  HashiCorp Vault     ──▶ ESO ──▶ K8s Secret          │      │
  │  │  GCP Secret Manager  ──▶ ESO ──▶ K8s Secret          │      │
  │  │                                                       │      │
  │  │  Syncs external secrets into K8s automatically       │      │
  │  │  Single source of truth outside the cluster          │      │
  │  └──────────────────────────────────────────────────────┘      │
  │                                                                  │
  │  SOLUTION 3: Sealed Secrets (Bitnami)                           │
  │  ├── Encrypt secrets client-side with kubeseal                  │
  │  ├── Only the cluster can decrypt (asymmetric crypto)          │
  │  ├── Safe to store encrypted SealedSecret in Git               │
  │  └── Enables GitOps for secrets                                 │
  │                                                                  │
  │  SOLUTION 4: SOPS + Age/KMS                                     │
  │  ├── Mozilla SOPS encrypts YAML/JSON values in-place           │
  │  ├── Integrates with AWS KMS, GCP KMS, Azure Key Vault         │
  │  └── Works with Flux, ArgoCD for GitOps                        │
  └──────────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha (Secrets in Git):** NEVER commit plain Kubernetes Secret manifests to Git. Anyone with repo access can `base64 -d` the values. Use Sealed Secrets, SOPS, or External Secrets Operator. Run `git log --all -p -- '*.yaml' | grep -i "kind: Secret"` to audit your repo for leaked secrets.

> **⚠️ Gotcha (ConfigMap updates):** When you update a ConfigMap, pods using it as **environment variables** do NOT see the change — they need a restart. Pods using it as **volume mounts** DO get updated automatically (~60-90 seconds), but the app must watch/reload the file. Trigger a rollout with: `kubectl rollout restart deployment/my-app`

---

## 8. Persistent Volumes & Storage

### 8.1 The Storage Problem in K8s

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  WHY K8s STORAGE IS DIFFERENT                                    │
  │                                                                  │
  │  Containers are EPHEMERAL — data dies with the container:       │
  │                                                                  │
  │  ┌─── Pod ────────┐                                              │
  │  │  ┌───────────┐ │  Pod dies → data GONE                      │
  │  │  │ Container │ │                                              │
  │  │  │ /data     │ │  Rescheduled to new node → NO data         │
  │  │  │ (tmpfs)   │ │                                              │
  │  │  └───────────┘ │                                              │
  │  └────────────────┘                                              │
  │                                                                  │
  │  Solution: Persistent Volumes — storage that outlives pods      │
  │                                                                  │
  │  ┌─── Pod ────────┐   ┌─── PersistentVolume ──────────────┐    │
  │  │  ┌───────────┐ │   │  EBS Volume / Azure Disk /        │    │
  │  │  │ Container │ │   │  GCE PD / NFS / Ceph              │    │
  │  │  │ /data     │─┼──▶│                                    │    │
  │  │  │           │ │   │  Data persists even if pod dies    │    │
  │  │  └───────────┘ │   │  Data survives rescheduling        │    │
  │  └────────────────┘   └────────────────────────────────────┘    │
  └──────────────────────────────────────────────────────────────────┘
```

### 8.2 PV, PVC, and StorageClass — The Three Actors

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  STORAGE ARCHITECTURE — PV / PVC / StorageClass                 │
  │                                                                  │
  │  StorageClass                                                    │
  │  (defines HOW storage is provisioned)                           │
  │       │                                                          │
  │       │ dynamic provisioning                                     │
  │       ▼                                                          │
  │  PersistentVolume (PV)          PersistentVolumeClaim (PVC)     │
  │  (the actual storage)           (pod's REQUEST for storage)     │
  │  ┌────────────────────┐         ┌─────────────────────┐        │
  │  │ 100Gi EBS gp3      │◀──bind──│ Request: 50Gi       │        │
  │  │ us-east-1a         │         │ Access: ReadWrite    │        │
  │  │ ReadWriteOnce      │         │ StorageClass: gp3   │        │
  │  └────────────────────┘         └─────────────────────┘        │
  │       ▲                               │                         │
  │       │ (provisioned by              │ (used by pods)          │
  │       │  cloud provider)              ▼                         │
  │                                  ┌────────────┐                 │
  │                                  │ Pod        │                 │
  │                                  │ mountPath: │                 │
  │                                  │  /data     │                 │
  │                                  └────────────┘                 │
  │                                                                  │
  │  Analogy:                                                        │
  │  StorageClass = "type of housing" (apartment, house, mansion)  │
  │  PV           = "the actual house" (address, size, features)    │
  │  PVC          = "rental application" (I need 2BR, parking)     │
  └──────────────────────────────────────────────────────────────────┘
```

### 8.3 Access Modes

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  VOLUME ACCESS MODES                                             │
  │                                                                  │
  │  Mode            │ Abbrev │ Description              │ Use Case │
  │  ────────────────┼────────┼──────────────────────────┼────────  │
  │  ReadWriteOnce   │ RWO    │ Read/write by ONE node   │ DBs,     │
  │                  │        │ (multiple pods same node) │ single   │
  │                  │        │                           │ instance │
  │  ────────────────┼────────┼──────────────────────────┼────────  │
  │  ReadOnlyMany    │ ROX    │ Read-only by MANY nodes  │ Config,  │
  │                  │        │                           │ static   │
  │                  │        │                           │ assets   │
  │  ────────────────┼────────┼──────────────────────────┼────────  │
  │  ReadWriteMany   │ RWX    │ Read/write by MANY nodes │ Shared   │
  │                  │        │                           │ data,    │
  │                  │        │                           │ CMS      │
  │  ────────────────┼────────┼──────────────────────────┼────────  │
  │  ReadWriteOnce   │ RWOP   │ Read/write by ONE pod    │ Strict   │
  │  Pod (K8s 1.22+) │        │ (not node, literal pod)  │ single   │
  │                  │        │                           │ writer   │
  │                                                                  │
  │  Cloud support:                                                  │
  │  EBS/Azure Disk/GCE PD:  RWO only (block storage)             │
  │  EFS/Azure Files/Filestore: RWX supported (file storage)       │
  │  NFS/CephFS/GlusterFS:  RWX supported                          │
  └──────────────────────────────────────────────────────────────────┘
```

### 8.4 StorageClass & Dynamic Provisioning

```yaml
# StorageClass — defines HOW volumes are provisioned
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com       # CSI driver
parameters:
  type: gp3                         # AWS EBS type
  iops: "3000"
  throughput: "125"
  encrypted: "true"
reclaimPolicy: Delete               # Delete | Retain
volumeBindingMode: WaitForFirstConsumer  # Important for topology!
allowVolumeExpansion: true          # Allow PVC resize
---
# PersistentVolumeClaim — request storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-data
  namespace: production
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 100Gi
---
# Using PVC in a Pod
apiVersion: v1
kind: Pod
metadata:
  name: postgres
spec:
  containers:
  - name: postgres
    image: postgres:16
    volumeMounts:
    - name: data
      mountPath: /var/lib/postgresql/data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: db-data
```

### 8.5 Volume Types Quick Reference

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  COMMON VOLUME TYPES                                             │
  │                                                                  │
  │  emptyDir:                                                       │
  │  ├── Temp storage — created with pod, deleted with pod          │
  │  ├── Shared between containers in same pod                      │
  │  └── Use: scratch space, caching, sidecar data sharing         │
  │                                                                  │
  │  hostPath:                                                       │
  │  ├── Mounts directory from the HOST node                        │
  │  ├── ⚠️ Data tied to specific node — breaks rescheduling       │
  │  └── Use: node-level agents (log collectors), DaemonSets        │
  │                                                                  │
  │  persistentVolumeClaim:                                          │
  │  ├── References a PVC → PV → actual storage                    │
  │  ├── Survives pod restarts and rescheduling                     │
  │  └── Use: databases, stateful applications                      │
  │                                                                  │
  │  configMap / secret:                                             │
  │  ├── Mounts ConfigMap or Secret as files                        │
  │  ├── Auto-updates when source changes (with delay)              │
  │  └── Use: config files, TLS certs                               │
  │                                                                  │
  │  projected:                                                      │
  │  ├── Combines multiple sources into one mount                   │
  │  └── Sources: configMap, secret, downwardAPI, serviceAccountToken│
  │                                                                  │
  │  csi (Container Storage Interface):                              │
  │  ├── Plugin-based storage (AWS EBS CSI, Azure Disk CSI, etc)   │
  │  ├── Standard interface — works across clouds                   │
  │  └── Replaces in-tree volume plugins (deprecated)               │
  └──────────────────────────────────────────────────────────────────┘
```

### 8.6 Reclaim Policies & PV Lifecycle

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  PV RECLAIM POLICIES — WHAT HAPPENS WHEN PVC IS DELETED        │
  │                                                                  │
  │  Delete (default for dynamic provisioning):                     │
  │  PVC deleted → PV deleted → underlying storage deleted         │
  │  ⚠️ DATA LOST! Use for temporary/reproducible data             │
  │                                                                  │
  │  Retain:                                                         │
  │  PVC deleted → PV status = "Released" → storage kept            │
  │  Admin must manually reclaim (delete PV, clean storage)         │
  │  ✅ Safest for databases — prevents accidental data loss       │
  │                                                                  │
  │  PV Status Lifecycle:                                            │
  │  Available → Bound → Released → (manually) Available            │
  │                                                                  │
  │  PRODUCTION RECOMMENDATION:                                      │
  │  ├── Databases: reclaimPolicy: Retain                           │
  │  ├── Caches/temp: reclaimPolicy: Delete                         │
  │  └── Always have backup strategy regardless of reclaim policy   │
  └──────────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha (WaitForFirstConsumer):** Always use `volumeBindingMode: WaitForFirstConsumer` for cloud block storage. Without it, K8s might provision the volume in `us-east-1a` but schedule the pod to `us-east-1b` — the pod will be stuck in Pending because EBS volumes are AZ-bound. `WaitForFirstConsumer` provisions the volume in the same AZ as the scheduled pod.

> **⚠️ Gotcha (PVC Resize):** You can expand a PVC (`kubectl edit pvc db-data` → increase storage) but you can NEVER shrink it. The StorageClass must have `allowVolumeExpansion: true`. For filesystem-based volumes, the pod must be restarted for the resize to take effect.

> **DevOps Relevance:** Storage is the most common source of production K8s issues — pods stuck in Pending due to AZ mismatch, PVCs not binding because no matching PV exists, running out of IOPS on gp2 volumes (use gp3 instead — baseline 3000 IOPS free). Always set `reclaimPolicy: Retain` for databases and implement automated backups (Velero, VolumeSnapshots).

---

## 9. Ingress & Traffic Management

### 9.1 Why Ingress? — The LoadBalancer Problem

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  THE PROBLEM: One LoadBalancer per Service = $$$                 │
  │                                                                  │
  │  Without Ingress (3 services = 3 cloud LBs = ~$75/month):      │
  │  ┌──────────┐      ┌──────────┐      ┌──────────┐             │
  │  │ LB ($25) │      │ LB ($25) │      │ LB ($25) │             │
  │  └────┬─────┘      └────┬─────┘      └────┬─────┘             │
  │       ▼                 ▼                  ▼                    │
  │  ┌─────────┐      ┌─────────┐       ┌─────────┐              │
  │  │ api-svc │      │ web-svc │       │ admin-  │              │
  │  │         │      │         │       │ svc     │              │
  │  └─────────┘      └─────────┘       └─────────┘              │
  │                                                                  │
  │  With Ingress (1 LB + routing rules):                           │
  │  ┌──────────────────────────────────────────────┐               │
  │  │           Single Load Balancer ($25)          │               │
  │  └──────────────────┬───────────────────────────┘               │
  │                     ▼                                            │
  │  ┌──────────────────────────────────────────────┐               │
  │  │         Ingress Controller (nginx/traefik)    │               │
  │  │                                                │               │
  │  │  api.example.com    → api-svc:8080            │               │
  │  │  www.example.com    → web-svc:80              │               │
  │  │  admin.example.com  → admin-svc:8080          │               │
  │  │  example.com/api/*  → api-svc:8080 (path)    │               │
  │  └──────────────────────────────────────────────┘               │
  │                                                                  │
  │  Benefits: Single LB, host/path-based routing, TLS termination │
  └──────────────────────────────────────────────────────────────────┘
```

### 9.2 Ingress Controllers Comparison

| Controller | Provider | Strengths | Best For |
|-----------|----------|-----------|----------|
| **NGINX Ingress** | Community/F5 | Most popular, well-documented | General purpose |
| **AWS ALB Ingress** | AWS | Native ALB integration, WAF | EKS workloads |
| **Traefik** | Traefik Labs | Auto-discovery, middleware, Let's Encrypt | Small/medium clusters |
| **HAProxy** | HAProxy | High performance, TCP/UDP | Enterprise, high traffic |
| **Istio Gateway** | Google/IBM | Full service mesh, mTLS | Microservices at scale |
| **Contour** | VMware | Envoy-based, HTTPProxy CRD | Advanced routing |
| **Kong** | Kong Inc | API gateway features, plugins | API-heavy architectures |

### 9.3 Ingress YAML Examples

```yaml
# Basic Ingress with host and path routing
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "10"
spec:
  ingressClassName: nginx          # Which controller handles this
  tls:
  - hosts:
    - api.example.com
    - www.example.com
    secretName: tls-secret         # TLS cert (or use cert-manager)
  rules:
  # Host-based routing
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 8080
  - host: www.example.com
    http:
      paths:
      # Path-based routing
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 8080
```

### 9.4 TLS / HTTPS with cert-manager

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  AUTOMATED TLS WITH cert-manager + Let's Encrypt               │
  │                                                                  │
  │  ┌──────────┐    ┌──────────────┐    ┌───────────────┐         │
  │  │ Ingress  │───▶│ cert-manager │───▶│ Let's Encrypt │         │
  │  │ (needs   │    │ (watches for │    │ (issues free  │         │
  │  │  TLS)    │    │  Ingress TLS │    │  certs, auto  │         │
  │  └──────────┘    │  annotations)│    │  renewal)     │         │
  │                  └──────┬───────┘    └───────────────┘         │
  │                         │                                       │
  │                         ▼                                       │
  │                  ┌──────────────┐                               │
  │                  │ K8s Secret   │  (auto-created TLS cert)     │
  │                  │ tls-secret   │                               │
  │                  └──────────────┘                               │
  │                                                                  │
  │  Flow:                                                          │
  │  1. Deploy cert-manager (Helm chart)                            │
  │  2. Create ClusterIssuer (Let's Encrypt config)                │
  │  3. Add annotation to Ingress                                   │
  │  4. cert-manager auto-provisions + renews certificates         │
  └──────────────────────────────────────────────────────────────────┘
```

```yaml
# ClusterIssuer for Let's Encrypt
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: devops@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
    - http01:
        ingress:
          class: nginx
---
# Ingress with cert-manager annotation
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-app
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod   # Auto TLS!
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls-cert       # cert-manager creates this
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-svc
            port:
              number: 8080
```

> **⚠️ Gotcha (Ingress pathType):** `pathType` has three values: `Exact` (exact match), `Prefix` (prefix match), and `ImplementationSpecific`. `Prefix` with `/api` also matches `/api/v1`, `/api-docs`, etc. For strict matching use `Exact`. Ordering matters — more specific paths should come first.

> **⚠️ Gotcha (Default Backend):** If no Ingress rule matches, NGINX returns **404**. Configure a default backend for a custom error page or redirect: `nginx.ingress.kubernetes.io/default-backend: default-error-svc`

> **DevOps Relevance:** In production, always use Ingress (not LoadBalancer per service), install cert-manager for automated TLS, and apply rate limiting + WAF rules. For AWS, the AWS Load Balancer Controller with ALB Ingress is preferred over NGINX for native integration. For multi-cluster, consider using Gateway API (the successor to Ingress — graduating to GA).

---

## 10. StatefulSets & Stateful Workloads

### 10.1 Why StatefulSets?

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  DEPLOYMENT vs STATEFULSET                                       │
  │                                                                  │
  │  Deployment (stateless apps):    StatefulSet (stateful apps):   │
  │  ┌──────────────────────┐       ┌──────────────────────┐       │
  │  │ Pod names: random     │       │ Pod names: ORDERED    │       │
  │  │ web-7d8f4b-abcde     │       │ db-0, db-1, db-2     │       │
  │  │ web-7d8f4b-fghij     │       │                       │       │
  │  │                       │       │ Startup: sequential   │       │
  │  │ Startup: parallel     │       │ db-0 → db-1 → db-2  │       │
  │  │ Any order             │       │                       │       │
  │  │                       │       │ Storage: per-pod PVC  │       │
  │  │ Storage: shared or    │       │ db-0 → pvc-db-0      │       │
  │  │ none                  │       │ db-1 → pvc-db-1      │       │
  │  │                       │       │ db-2 → pvc-db-2      │       │
  │  │ Network: random IPs   │       │                       │       │
  │  │                       │       │ Network: stable DNS   │       │
  │  │                       │       │ db-0.db-svc.ns.svc   │       │
  │  └──────────────────────┘       └──────────────────────┘       │
  │                                                                  │
  │  Use StatefulSets for:                                           │
  │  ├── Databases (PostgreSQL, MySQL, MongoDB)                     │
  │  ├── Message queues (Kafka, RabbitMQ)                           │
  │  ├── Distributed systems (Elasticsearch, Zookeeper)             │
  │  └── Any app requiring stable identity or persistent storage   │
  └──────────────────────────────────────────────────────────────────┘
```

### 10.2 StatefulSet Guarantees

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  STATEFULSET GUARANTEES                                          │
  │                                                                  │
  │  1. STABLE POD IDENTITY                                         │
  │  ├── Pods named: <statefulset>-<ordinal>  (db-0, db-1, db-2)  │
  │  ├── Name persists across rescheduling                          │
  │  └── Hostname inside pod matches pod name                       │
  │                                                                  │
  │  2. STABLE NETWORK IDENTITY (Headless Service required)         │
  │  ├── DNS: <pod>.<service>.<namespace>.svc.cluster.local        │
  │  ├── db-0.db-headless.production.svc.cluster.local             │
  │  └── Resolves to the SAME pod even after restart               │
  │                                                                  │
  │  3. ORDERED DEPLOYMENT & SCALING                                 │
  │  ├── Scale up: 0 → 1 → 2 (sequential, waits for Ready)        │
  │  ├── Scale down: 2 → 1 → 0 (reverse order)                    │
  │  ├── Updates: reverse order by default (2 → 1 → 0)            │
  │  └── podManagementPolicy: Parallel (opt-in for parallel)       │
  │                                                                  │
  │  4. STABLE STORAGE                                               │
  │  ├── volumeClaimTemplates creates a PVC per pod                 │
  │  ├── PVC named: <claim>-<statefulset>-<ordinal>                │
  │  ├── data-db-0, data-db-1, data-db-2                           │
  │  └── PVCs are NOT deleted when StatefulSet is deleted           │
  │      (data safety — must delete PVCs manually)                  │
  └──────────────────────────────────────────────────────────────────┘
```

### 10.3 StatefulSet YAML

```yaml
# Headless Service (required for StatefulSet DNS)
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: production
spec:
  clusterIP: None                  # ← Makes it headless
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres-headless   # Must reference headless service
  replicas: 3
  podManagementPolicy: OrderedReady  # Default (or Parallel)
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0                 # Update pods with ordinal >= partition
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:16
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: pg-secret
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: "2"
            memory: 4Gi
        readinessProbe:
          exec:
            command: ["pg_isready", "-U", "postgres"]
          periodSeconds: 10
  volumeClaimTemplates:            # ← Creates PVC per pod
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 100Gi
```

> **⚠️ Gotcha (StatefulSet deletion):** Deleting a StatefulSet does NOT delete its PVCs (by design — data safety). This means orphaned PVCs and cloud volumes keep existing (and costing money). Clean up manually: `kubectl delete pvc -l app=postgres -n production`. From K8s 1.27+, use `persistentVolumeClaimRetentionPolicy` to auto-delete.

> **⚠️ Gotcha (Headless Service):** StatefulSets REQUIRE a headless Service (`clusterIP: None`). Forgetting this means pods won't get stable DNS names. The `serviceName` field in the StatefulSet spec MUST match the headless service name.

---

## 11. DaemonSets, Jobs & CronJobs

### 11.1 DaemonSets — One Pod Per Node

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  DAEMONSET — RUNS EXACTLY ONE POD ON EVERY NODE                 │
  │                                                                  │
  │  ┌─── Cluster ────────────────────────────────────────────┐     │
  │  │                                                         │     │
  │  │  Node 1              Node 2              Node 3        │     │
  │  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │     │
  │  │  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │ │     │
  │  │  │ │ fluentd  │ │  │ │ fluentd  │ │  │ │ fluentd  │ │ │     │
  │  │  │ │ (daemon) │ │  │ │ (daemon) │ │  │ │ (daemon) │ │ │     │
  │  │  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │ │     │
  │  │  │ + app pods   │  │ + app pods   │  │ + app pods   │ │     │
  │  │  └──────────────┘  └──────────────┘  └──────────────┘ │     │
  │  │                                                         │     │
  │  │  New Node 4 added → DaemonSet auto-schedules pod       │     │
  │  │  Node 3 removed → DaemonSet pod automatically removed  │     │
  │  └─────────────────────────────────────────────────────────┘     │
  │                                                                  │
  │  Common DaemonSet workloads:                                     │
  │  ├── Log collectors: Fluentd, Fluent Bit, Filebeat             │
  │  ├── Monitoring agents: Prometheus Node Exporter, Datadog      │
  │  ├── Network plugins: Calico, Cilium, kube-proxy               │
  │  ├── Storage drivers: CSI node plugins                          │
  │  └── Security agents: Falco, Twistlock                         │
  └──────────────────────────────────────────────────────────────────┘
```

```yaml
# DaemonSet — log collector on every node
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: logging
spec:
  selector:
    matchLabels:
      app: fluentd
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1             # Update one node at a time
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      tolerations:                   # Run on ALL nodes including master
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluent/fluentd:v1.16
        resources:
          requests:
            cpu: 100m
            memory: 200Mi
          limits:
            cpu: 500m
            memory: 500Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: containers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: containers
        hostPath:
          path: /var/lib/docker/containers
```

### 11.2 Jobs — Run to Completion

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  JOBS — ONE-TIME OR BATCH TASKS                                  │
  │                                                                  │
  │  Deployment:                     Job:                            │
  │  ├── Runs forever                ├── Runs until completion      │
  │  ├── Restarts on exit            ├── Tracks successful completns│
  │  └── restartPolicy: Always       └── restartPolicy: OnFailure  │
  │                                      or Never                    │
  │                                                                  │
  │  Job Patterns:                                                   │
  │  ┌─────────────────────────────────────────────────────────┐    │
  │  │  Single Job:        completions: 1, parallelism: 1      │    │
  │  │  (one pod, one run)                                      │    │
  │  │                                                          │    │
  │  │  Fixed Completion:  completions: 5, parallelism: 2      │    │
  │  │  (5 tasks, 2 at a time)                                 │    │
  │  │  ┌───┐ ┌───┐                                            │    │
  │  │  │ 1 │ │ 2 │  → ┌───┐ ┌───┐ → ┌───┐                  │    │
  │  │  └───┘ └───┘    │ 3 │ │ 4 │   │ 5 │                  │    │
  │  │                  └───┘ └───┘   └───┘                  │    │
  │  │                                                          │    │
  │  │  Work Queue:       completions: unset, parallelism: 3   │    │
  │  │  (workers pull from queue until empty)                  │    │
  │  └─────────────────────────────────────────────────────────┘    │
  └──────────────────────────────────────────────────────────────────┘
```

```yaml
# Job — database migration
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  namespace: production
spec:
  backoffLimit: 3                  # Retry up to 3 times on failure
  activeDeadlineSeconds: 600       # Timeout after 10 minutes
  ttlSecondsAfterFinished: 3600   # Auto-delete after 1 hour
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: migrate
        image: my-app:v2.0
        command: ["python", "manage.py", "migrate"]
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
```

### 11.3 CronJobs — Scheduled Tasks

```yaml
# CronJob — nightly database backup
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-backup
  namespace: production
spec:
  schedule: "0 2 * * *"           # 2:00 AM daily (standard cron syntax)
  concurrencyPolicy: Forbid        # Don't start if previous still running
  successfulJobsHistoryLimit: 3    # Keep last 3 successful jobs
  failedJobsHistoryLimit: 5        # Keep last 5 failed jobs
  startingDeadlineSeconds: 300     # Skip if missed by > 5 minutes
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: postgres:16
            command:
            - /bin/sh
            - -c
            - |
              pg_dump -h $DB_HOST -U $DB_USER $DB_NAME | \
              gzip > /backup/db-$(date +%Y%m%d-%H%M).sql.gz
            env:
            - name: DB_HOST
              value: "postgres-headless"
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: username
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: password
            volumeMounts:
            - name: backup-vol
              mountPath: /backup
          volumes:
          - name: backup-vol
            persistentVolumeClaim:
              claimName: backup-storage
```

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  CRONJOB CONCURRENCY POLICIES                                    │
  │                                                                  │
  │  Allow (default):  Multiple jobs can run concurrently           │
  │  Forbid:           Skip new job if previous still running       │
  │  Replace:          Cancel running job and start new one         │
  │                                                                  │
  │  CRON SCHEDULE QUICK REFERENCE:                                  │
  │  ┌───────┬──────┬──────┬──────┬──────────┐                     │
  │  │ Min   │ Hour │ Day  │Month │ Day/Week │                     │
  │  │ (0-59)│(0-23)│(1-31)│(1-12)│ (0-6)    │                     │
  │  ├───────┼──────┼──────┼──────┼──────────┤                     │
  │  │ */5   │ *    │ *    │ *    │ *        │ Every 5 minutes     │
  │  │ 0     │ */2  │ *    │ *    │ *        │ Every 2 hours       │
  │  │ 0     │ 2    │ *    │ *    │ *        │ Daily at 2 AM       │
  │  │ 0     │ 0    │ *    │ *    │ 0        │ Weekly (Sunday)     │
  │  │ 0     │ 0    │ 1    │ *    │ *        │ Monthly (1st)       │
  │  └───────┴──────┴──────┴──────┴──────────┘                     │
  └──────────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha (CronJob timezone):** CronJobs use the **kube-controller-manager's timezone** (usually UTC). If you need a specific timezone, use `timeZone` field (K8s 1.25+, beta in 1.27): `timeZone: "America/New_York"`. Without this, your "2 AM" backup runs at 2 AM UTC, not local time.

> **⚠️ Gotcha (Job cleanup):** Completed Jobs and their pods are NOT auto-deleted by default — they accumulate forever, cluttering `kubectl get pods`. Set `ttlSecondsAfterFinished` on Jobs and `successfulJobsHistoryLimit` / `failedJobsHistoryLimit` on CronJobs to auto-clean.

---

## 12. RBAC & Security

### 12.1 RBAC Mental Model

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  RBAC — ROLE-BASED ACCESS CONTROL                               │
  │                                                                  │
  │  WHO              can do          WHAT           on WHICH        │
  │  (Subject)        (Verb)          (Resource)     (Scope)         │
  │  ┌──────────┐    ┌──────────┐   ┌──────────┐  ┌──────────┐    │
  │  │ User     │    │ get      │   │ pods     │  │ Namespace │    │
  │  │ Group    │    │ list     │   │ deploys  │  │ (Role)    │    │
  │  │ Service  │    │ create   │   │ services │  │           │    │
  │  │ Account  │    │ update   │   │ secrets  │  │ Cluster   │    │
  │  │          │    │ delete   │   │ nodes    │  │ (ClusterR)│    │
  │  │          │    │ watch    │   │ ingress  │  │           │    │
  │  │          │    │ patch    │   │ configmap│  │           │    │
  │  └──────────┘    └──────────┘   └──────────┘  └──────────┘    │
  │                                                                  │
  │  LINKING: BINDINGS                                               │
  │                                                                  │
  │  ┌──────────┐    RoleBinding      ┌──────────┐                 │
  │  │ Subject  │ ──────────────────▶ │  Role    │                 │
  │  │ (who)    │                      │  (perms) │                 │
  │  └──────────┘    ClusterRole      └──────────┘                 │
  │                  Binding                                         │
  │                                                                  │
  │  KEY RULES:                                                      │
  │  ├── RBAC is ADDITIVE (no deny rules — only allow)             │
  │  ├── No permissions by default (deny-all baseline)              │
  │  └── A subject can have multiple bindings                        │
  └──────────────────────────────────────────────────────────────────┘
```

### 12.2 Role vs ClusterRole

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  ROLE vs CLUSTERROLE                                             │
  │                                                                  │
  │  Role:                           ClusterRole:                    │
  │  ├── Namespace-scoped            ├── Cluster-wide               │
  │  ├── Permissions in ONE          ├── Permissions across ALL     │
  │  │   namespace                   │   namespaces                  │
  │  ├── Bound by RoleBinding        ├── Bound by ClusterRoleBinding│
  │  └── Use for: app permissions    │   OR RoleBinding (to grant  │
  │                                   │   in single namespace)       │
  │                                   └── Use for: cluster admins,  │
  │                                       non-namespaced resources   │
  │                                                                  │
  │  COMBINATION TRICK:                                              │
  │  ClusterRole + RoleBinding = reuse ClusterRole in 1 namespace  │
  │  (define permissions once, grant per-namespace)                 │
  │                                                                  │
  │  Example:                                                        │
  │  ClusterRole: "pod-reader" (get, list pods)                     │
  │  RoleBinding in "staging": john → pod-reader (staging only)    │
  │  RoleBinding in "prod":    john → pod-reader (prod only)       │
  └──────────────────────────────────────────────────────────────────┘
```

### 12.3 RBAC YAML Examples

```yaml
# Role — namespace-scoped permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: staging
rules:
- apiGroups: [""]                  # "" = core API group
  resources: ["pods", "pods/log", "pods/exec", "services", "configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]           # Read-only for secrets
---
# RoleBinding — bind user to role
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: staging
subjects:
- kind: User
  name: jane@example.com
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: dev-team
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: ci-deployer
  namespace: staging
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
---
# ClusterRole — cluster-wide permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-viewer
rules:
- apiGroups: [""]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps", "batch", "networking.k8s.io"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: []                        # Explicitly NO access to secrets
```

### 12.4 ServiceAccounts — Pod Identity

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  SERVICEACCOUNTS — IDENTITY FOR PODS                             │
  │                                                                  │
  │  Every pod runs as a ServiceAccount:                             │
  │  ├── Default: "default" SA (in each namespace)                  │
  │  ├── Production: create dedicated SAs per workload              │
  │  └── K8s 1.24+: tokens are time-limited (no more eternal tokens)│
  │                                                                  │
  │  ┌─────────────────────────────────────────────────────────┐    │
  │  │  Pod (runs as SA "ci-deployer")                          │    │
  │  │  ├── Token auto-mounted at /var/run/secrets/...         │    │
  │  │  ├── Can authenticate to API server                      │    │
  │  │  └── Permissions = whatever Role/ClusterRole is bound   │    │
  │  └─────────────────────────────────────────────────────────┘    │
  │                                                                  │
  │  Cloud integration:                                              │
  │  ├── AWS: IRSA (IAM Roles for Service Accounts)                │
  │  │   SA → annotate with IAM role → pods get AWS creds         │
  │  ├── GCP: Workload Identity                                     │
  │  │   SA → bound to GCP service account → pods get GCP creds   │
  │  └── Azure: Workload Identity (AAD)                             │
  │      SA → federated credential → pods get Azure creds          │
  └──────────────────────────────────────────────────────────────────┘
```

```yaml
# ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: production
  annotations:
    # AWS IRSA — attach IAM role to pods
    eks.amazonaws.com/role-arn: "arn:aws:iam::123456789:role/app-role"
automountServiceAccountToken: false   # Don't mount token unless needed
---
# Pod using ServiceAccount
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  serviceAccountName: app-sa       # Use specific SA, not default
  automountServiceAccountToken: true  # Override if needed
  containers:
  - name: app
    image: my-app:v2.0
```

### 12.5 RBAC Best Practices for Production

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  RBAC PRODUCTION BEST PRACTICES                                  │
  │                                                                  │
  │  1. PRINCIPLE OF LEAST PRIVILEGE                                 │
  │  ├── Start with zero permissions, add only what's needed        │
  │  ├── Never use cluster-admin for apps                           │
  │  └── Review permissions quarterly                                │
  │                                                                  │
  │  2. USE GROUPS, NOT INDIVIDUAL USERS                             │
  │  ├── Bind ClusterRoleBindings to groups                         │
  │  ├── Manage group membership in IdP (Okta, Azure AD)           │
  │  └── Easier onboarding/offboarding                              │
  │                                                                  │
  │  3. DEDICATED SERVICE ACCOUNTS                                   │
  │  ├── Never use the "default" SA for applications                │
  │  ├── One SA per application/microservice                        │
  │  └── Set automountServiceAccountToken: false on default SA      │
  │                                                                  │
  │  4. AUDIT REGULARLY                                              │
  │  ├── kubectl auth can-i --list --as=jane@example.com           │
  │  ├── Use tools: rbac-lookup, kubectl-who-can, rakkess          │
  │  └── Enable API server audit logging                             │
  │                                                                  │
  │  5. AVOID WILDCARDS                                              │
  │  ├── Don't use resources: ["*"] or verbs: ["*"] in production  │
  │  └── Explicitly list resources and verbs                        │
  └──────────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha (Default SA):** The `default` ServiceAccount in every namespace has no RBAC permissions by default, BUT it still gets a mounted API token. If an attacker compromises a pod, they can use this token to discover cluster information. Set `automountServiceAccountToken: false` on the default SA and on all pods that don't need API access.

> **⚠️ Gotcha (RBAC debugging):** RBAC errors show as `403 Forbidden` — but the error message doesn't tell you WHICH role is missing. Debug with: `kubectl auth can-i create deployments --as=jane@example.com -n staging`. For systematic auditing: `kubectl auth can-i --list --as=system:serviceaccount:staging:ci-deployer -n staging`.

> **DevOps Relevance:** RBAC is critical for multi-tenant clusters and compliance (SOC2, ISO 27001, HIPAA). DevOps teams typically define 3-4 standard ClusterRoles (viewer, developer, deployer, admin) and grant them via RoleBindings per namespace. Integrate with your IdP (OIDC) so users authenticate with SSO — never distribute kubeconfig files with long-lived tokens.

---

## 13. Network Policies

### 13.1 Why Network Policies?

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  DEFAULT: EVERY POD CAN TALK TO EVERY OTHER POD                 │
  │                                                                  │
  │  Without Network Policies (flat network — zero security):       │
  │  ┌─────────┐     ┌─────────┐     ┌─────────┐                  │
  │  │frontend │←───▶│ backend │←───▶│  redis  │                  │
  │  └─────────┘     └─────────┘     └─────────┘                  │
  │       ↕               ↕               ↕                         │
  │  ┌─────────┐     ┌─────────┐     ┌─────────┐                  │
  │  │  admin  │←───▶│ logging │←───▶│ payment │                  │
  │  └─────────┘     └─────────┘     └─────────┘                  │
  │  ⚠️ Everything can reach everything — if one pod is            │
  │  compromised, attacker has access to entire cluster network    │
  │                                                                  │
  │  With Network Policies (zero-trust — deny by default):         │
  │  ┌─────────┐     ┌─────────┐     ┌─────────┐                  │
  │  │frontend │────▶│ backend │────▶│  redis  │                  │
  │  └─────────┘     └─────────┘     └─────────┘                  │
  │       ✗               ✗               ✗                         │
  │  ┌─────────┐     ┌─────────┐     ┌─────────┐                  │
  │  │  admin  │     │ logging │     │ payment │                  │
  │  └─────────┘     └─────────┘     └─────────┘                  │
  │  ✅ Only explicitly allowed traffic flows                      │
  │                                                                  │
  │  REQUIRES: CNI that supports Network Policies                   │
  │  ├── Calico ✅    Cilium ✅    Weave ✅                        │
  │  ├── AWS VPC CNI ✅ (with Calico addon)                        │
  │  └── Flannel ❌ (no support — add Calico alongside)           │
  └──────────────────────────────────────────────────────────────────┘
```

### 13.2 Network Policy YAML

```yaml
# Default deny all ingress in namespace (zero-trust baseline)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}                  # Applies to ALL pods in namespace
  policyTypes:
  - Ingress                        # Deny all incoming traffic
---
# Default deny all egress (lock down outbound too)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress                         # Deny all outgoing traffic
---
# Allow DNS egress (required for service discovery!)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}                  # ALL pods
  policyTypes:
  - Egress
  egress:
  - to: []
    ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
---
# Allow frontend → backend on port 8080
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend                 # Applies TO backend pods
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend            # Allow FROM frontend pods
    ports:
    - port: 8080
      protocol: TCP
---
# Allow backend → Redis (cross-namespace)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-redis
  namespace: data
spec:
  podSelector:
    matchLabels:
      app: redis
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: production         # From production namespace
      podSelector:
        matchLabels:
          app: backend             # AND from backend pods
    ports:
    - port: 6379
```

### 13.3 Network Policy Mental Model

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  NETWORK POLICY RULES                                            │
  │                                                                  │
  │  INGRESS (who can talk TO my pods):                             │
  │  from:                                                           │
  │  ├── podSelector:     by pod labels (same namespace)            │
  │  ├── namespaceSelector: by namespace labels (cross-ns)          │
  │  ├── ipBlock:         by CIDR range (external IPs)              │
  │  └── combine:        AND logic within one rule,                 │
  │                       OR logic between rules                     │
  │                                                                  │
  │  EGRESS (who can my pods talk TO):                              │
  │  to:                                                             │
  │  ├── podSelector:     allow traffic to specific pods            │
  │  ├── namespaceSelector: allow traffic to specific namespaces    │
  │  ├── ipBlock:         allow traffic to external CIDR            │
  │  └── ports:           restrict to specific ports                │
  │                                                                  │
  │  ⚠️ IMPORTANT LOGIC:                                            │
  │  ┌──────────────────────────────────────────────────────────┐   │
  │  │  One rule with both podSelector AND namespaceSelector    │   │
  │  │  = AND (pod must match AND namespace must match)         │   │
  │  │                                                           │   │
  │  │  Two separate rules (each with one selector)             │   │
  │  │  = OR (either pod matches OR namespace matches)          │   │
  │  └──────────────────────────────────────────────────────────┘   │
  └──────────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha (DNS + NetworkPolicy):** When you apply a default deny-all egress policy, pods can no longer resolve DNS — breaking ALL service discovery. Always add a DNS allow policy (port 53 UDP/TCP to kube-system) alongside deny-all egress. This is the #1 mistake with Network Policies.

> **⚠️ Gotcha (AND vs OR):** Putting `podSelector` and `namespaceSelector` in the **same** `from` entry means AND. Putting them in **separate** `from` entries means OR. Getting this wrong opens your network policy much wider than intended. Always test with `kubectl exec` to verify traffic flows.

> **DevOps Relevance:** Network Policies are required for any compliance-driven environment (PCI-DSS, HIPAA, SOC2). Start with default-deny in every namespace, then add specific allow rules. Use tools like NetworkPolicy Editor (cilium.io/editor) to visualize policies before applying.

---

## 14. Helm & Package Management

### 14.1 Why Helm?

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  THE PROBLEM: MANAGING MANY YAML FILES                          │
  │                                                                  │
  │  Without Helm (raw YAML):        With Helm (templated charts):  │
  │  ├── deployment.yaml              ├── One chart = one package   │
  │  ├── service.yaml                 ├── Values file per env       │
  │  ├── configmap.yaml               ├── Versioned releases        │
  │  ├── ingress.yaml                 ├── Rollback with one command │
  │  ├── secret.yaml                  ├── Dependencies managed      │
  │  ├── hpa.yaml                     ├── Shared via repositories   │
  │  ├── networkpolicy.yaml           └── Community charts for      │
  │  └── pdb.yaml                         common apps (Prometheus,  │
  │                                        Grafana, NGINX, etc.)    │
  │  Problems:                                                       │
  │  ├── Duplication across envs                                    │
  │  ├── No versioning of "releases"                                │
  │  ├── No rollback mechanism                                      │
  │  └── Hard to share/reuse                                        │
  │                                                                  │
  │  Helm = "apt/yum for Kubernetes"                                │
  └──────────────────────────────────────────────────────────────────┘
```

### 14.2 Helm Architecture

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  HELM 3 ARCHITECTURE (Tiller removed since Helm 3)              │
  │                                                                  │
  │  ┌──────────┐                                                    │
  │  │  helm    │  (CLI client)                                     │
  │  │  client  │                                                    │
  │  └────┬─────┘                                                    │
  │       │ 1. Reads chart + values                                  │
  │       │ 2. Renders templates → K8s manifests                    │
  │       │ 3. Sends to API server via kubeconfig                   │
  │       ▼                                                          │
  │  ┌──────────────┐                                                │
  │  │ kube-apiserver│  (no Tiller — direct API calls)              │
  │  └──────────────┘                                                │
  │                                                                  │
  │  Chart Structure:                                                │
  │  my-app/                                                         │
  │  ├── Chart.yaml        # Name, version, description            │
  │  ├── values.yaml       # Default configuration values           │
  │  ├── templates/        # Go template YAML files                 │
  │  │   ├── deployment.yaml                                        │
  │  │   ├── service.yaml                                           │
  │  │   ├── ingress.yaml                                           │
  │  │   ├── _helpers.tpl  # Template helper functions              │
  │  │   ├── NOTES.txt     # Post-install instructions              │
  │  │   └── tests/        # Helm test pods                         │
  │  ├── charts/           # Dependency charts (subcharts)          │
  │  └── .helmignore       # Files to ignore when packaging         │
  │                                                                  │
  │  Key Concepts:                                                   │
  │  ├── Chart:   package of K8s resources (templates + values)    │
  │  ├── Release: a running instance of a chart in a cluster       │
  │  ├── Repository: collection of chart packages (like npm)       │
  │  └── Revision: each upgrade creates a new release revision     │
  └──────────────────────────────────────────────────────────────────┘
```

### 14.3 Helm Templating Basics

```yaml
# values.yaml (per-environment config)
replicaCount: 3
image:
  repository: my-app
  tag: v2.1.0
  pullPolicy: IfNotPresent
service:
  type: ClusterIP
  port: 80
resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
ingress:
  enabled: true
  host: app.example.com
---
# templates/deployment.yaml (Go template)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: {{ .Values.service.port }}
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
```

### 14.4 Essential Helm Commands

```bash
# Add a chart repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Search for charts
helm search repo prometheus
helm search hub grafana              # Search Artifact Hub

# Install a chart
helm install my-release bitnami/nginx -n production
helm install my-app ./my-chart -f values-prod.yaml

# Install with overrides
helm install my-app ./chart \
  --set replicaCount=5 \
  --set image.tag=v3.0.0 \
  -f values-prod.yaml               # File values + --set overrides

# List releases
helm list -n production

# Check what YAML will be generated (dry-run)
helm template my-app ./chart -f values-prod.yaml
helm install my-app ./chart --dry-run --debug

# Upgrade a release
helm upgrade my-app ./chart -f values-prod.yaml

# Rollback
helm rollback my-app 2              # Rollback to revision 2
helm history my-app                 # View revision history

# Uninstall
helm uninstall my-app -n production

# Package chart for distribution
helm package ./my-chart
helm push my-chart-1.0.0.tgz oci://registry.example.com/charts
```

### 14.5 Helmfile — Managing Multiple Releases

```yaml
# helmfile.yaml — declarative Helm release management
repositories:
- name: bitnami
  url: https://charts.bitnami.com/bitnami
- name: prometheus-community
  url: https://prometheus-community.github.io/helm-charts

releases:
- name: nginx-ingress
  namespace: ingress
  chart: bitnami/nginx-ingress-controller
  version: 9.7.0
  values:
  - ingress-values.yaml

- name: prometheus
  namespace: monitoring
  chart: prometheus-community/kube-prometheus-stack
  version: 51.0.0
  values:
  - monitoring-values.yaml

- name: my-app
  namespace: production
  chart: ./charts/my-app
  values:
  - values/{{ .Environment.Name }}.yaml   # Environment-specific
  secrets:
  - secrets/{{ .Environment.Name }}.yaml  # Encrypted with SOPS
```

```bash
# Apply all releases
helmfile -e production apply

# Diff before applying
helmfile -e production diff

# Sync specific release
helmfile -e production -l name=my-app sync
```

> **⚠️ Gotcha (Helm values precedence):** Values are merged in order: `values.yaml` (defaults) → `-f file.yaml` → `--set key=val`. Later sources override earlier ones. If you use both `-f` and `--set`, `--set` wins. Document your values source chain to avoid "mystery values" in production.

> **DevOps Relevance:** Helm is the de facto package manager for K8s and essential in CI/CD pipelines. Use Helmfile for managing multiple releases declaratively. Store charts in OCI registries (ECR, ACR, GCR) alongside your container images. Always pin chart versions in production — never use `latest` or ranges.

---

## 15. Operators & Custom Resource Definitions (CRDs)

### 15.1 Extending Kubernetes

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  K8s IS EXTENSIBLE — CRDs + OPERATORS                          │
  │                                                                  │
  │  Built-in resources:             Custom resources (CRDs):       │
  │  ├── Pod                          ├── PostgresCluster           │
  │  ├── Deployment                   ├── Certificate              │
  │  ├── Service                      ├── VaultSecret              │
  │  ├── ConfigMap                    ├── PrometheusRule            │
  │  └── Ingress                      └── KafkaTopic               │
  │                                                                  │
  │  CRD = Custom Resource Definition                               │
  │  Teaches K8s about a new resource type.                         │
  │  After creating a CRD, you can:                                 │
  │  kubectl get postgresclusters                                   │
  │  kubectl describe certificate my-cert                           │
  │                                                                  │
  │  OPERATOR = CRD + Custom Controller                             │
  │  A controller that watches CRDs and takes action.              │
  │  "Human operator knowledge encoded in software"                │
  │                                                                  │
  │  Example: PostgreSQL Operator                                   │
  │  ┌───────────────┐     ┌──────────────────────────┐            │
  │  │ You create:   │     │ Operator automatically:   │            │
  │  │ PostgresClust │────▶│ Creates StatefulSet       │            │
  │  │ replicas: 3   │     │ Configures replication    │            │
  │  │ version: 16   │     │ Sets up backups           │            │
  │  │ backup: s3    │     │ Handles failover          │            │
  │  └───────────────┘     │ Manages upgrades          │            │
  │                         └──────────────────────────┘            │
  └──────────────────────────────────────────────────────────────────┘
```

### 15.2 The Operator Pattern

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  OPERATOR PATTERN — RECONCILIATION LOOP                         │
  │                                                                  │
  │  ┌───────────────────────────────────────────────────────┐      │
  │  │                                                       │      │
  │  │  1. OBSERVE:  Watch for changes to custom resources  │      │
  │  │       │                                               │      │
  │  │       ▼                                               │      │
  │  │  2. COMPARE:  Desired state (CR) vs Actual state     │      │
  │  │       │                                               │      │
  │  │       ▼                                               │      │
  │  │  3. ACT:      Create/update/delete resources to      │      │
  │  │       │        reach desired state                    │      │
  │  │       │                                               │      │
  │  │       └────────── Loop back to OBSERVE ──────────────│      │
  │  │                                                       │      │
  │  └───────────────────────────────────────────────────────┘      │
  │                                                                  │
  │  This is the SAME pattern as built-in controllers:              │
  │  ├── Deployment controller watches Deployment CRs              │
  │  ├── ReplicaSet controller watches ReplicaSet CRs              │
  │  └── Your operator watches YOUR custom resources               │
  └──────────────────────────────────────────────────────────────────┘
```

### 15.3 Popular Production Operators

| Operator | Purpose | Managed by |
|----------|---------|------------|
| **Prometheus Operator** | Monitoring stack (Prometheus + Alertmanager) | CoreOS / Red Hat |
| **cert-manager** | TLS certificate lifecycle | Jetstack / CNCF |
| **External Secrets** | Sync secrets from external vaults | External Secrets Inc |
| **CloudNativePG** | PostgreSQL clusters with HA, backups | EDB / CNCF |
| **Strimzi** | Apache Kafka on Kubernetes | Strimzi / CNCF |
| **Istio Operator** | Service mesh installation and management | Google / Istio |
| **Argo CD** | GitOps continuous deployment | Intuit / CNCF |
| **Velero** | Backup and disaster recovery | VMware / CNCF |
| **Crossplane** | Infrastructure provisioning (cloud resources as CRs) | Upbound / CNCF |

### 15.4 CRD Example

```yaml
# Custom Resource Definition
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: appconfigs.mycompany.io
spec:
  group: mycompany.io
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              replicas:
                type: integer
                minimum: 1
                maximum: 20
              environment:
                type: string
                enum: ["dev", "staging", "production"]
              features:
                type: object
                properties:
                  autoScaling:
                    type: boolean
                  monitoring:
                    type: boolean
    additionalPrinterColumns:
    - name: Replicas
      type: integer
      jsonPath: .spec.replicas
    - name: Environment
      type: string
      jsonPath: .spec.environment
  scope: Namespaced
  names:
    plural: appconfigs
    singular: appconfig
    kind: AppConfig
    shortNames: ["ac"]
---
# Using the Custom Resource
apiVersion: mycompany.io/v1
kind: AppConfig
metadata:
  name: my-app
  namespace: production
spec:
  replicas: 5
  environment: production
  features:
    autoScaling: true
    monitoring: true
```

> **⚠️ Gotcha (CRD deletion):** Deleting a CRD deletes ALL instances of that custom resource across the entire cluster. If you delete the `postgresclusters.postgres-operator.crunchydata.com` CRD, ALL PostgreSQL clusters managed by that operator are gone. Always be careful when `kubectl delete crd`.

> **DevOps Relevance:** As a DevOps engineer, you'll primarily consume operators (cert-manager, Prometheus, External Secrets) rather than build them. Understanding the operator pattern helps you debug issues — check the operator logs when a CR isn't reconciling: `kubectl logs -n operator-namespace deployment/operator-name`. For building operators, use Kubebuilder or Operator SDK.

---

## 16. Scheduling & Resource Management

### 16.1 How the Scheduler Works

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  KUBE-SCHEDULER — HOW PODS GET ASSIGNED TO NODES               │
  │                                                                  │
  │  New pod (no nodeName) ─▶ Scheduler picks best node            │
  │                                                                  │
  │  Phase 1: FILTERING (eliminate unsuitable nodes)                │
  │  ├── Does node have enough CPU/memory? (resource requests)     │
  │  ├── Does node satisfy nodeSelector / nodeAffinity?            │
  │  ├── Does pod tolerate node's taints?                           │
  │  ├── Does node have required ports available?                   │
  │  └── Does node match topologySpreadConstraints?                │
  │                                                                  │
  │  Phase 2: SCORING (rank remaining nodes)                        │
  │  ├── LeastRequestedPriority (prefer less loaded nodes)         │
  │  ├── BalancedResourceAllocation (even CPU/memory ratio)        │
  │  ├── Inter-pod affinity/anti-affinity scoring                  │
  │  └── Topology spread scoring                                    │
  │                                                                  │
  │  Result: Pod assigned to highest-scoring node                   │
  │                                                                  │
  │  If NO node passes filtering → Pod stays Pending               │
  └──────────────────────────────────────────────────────────────────┘
```

### 16.2 Node Selectors, Affinity & Anti-Affinity

```yaml
# nodeSelector — simplest scheduling constraint
apiVersion: v1
kind: Pod
metadata:
  name: gpu-app
spec:
  nodeSelector:
    gpu-type: nvidia-a100           # Only schedule on nodes with this label
  containers:
  - name: app
    image: ml-training:v1
---
# Node Affinity — more expressive than nodeSelector
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          # MUST schedule on these nodes (hard requirement)
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/arch
                operator: In
                values: ["amd64"]
              - key: node-type
                operator: In
                values: ["compute", "general"]
          # PREFER these nodes (soft preference — not required)
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 80
            preference:
              matchExpressions:
              - key: availability-zone
                operator: In
                values: ["us-east-1a"]
        # Pod Anti-Affinity — spread replicas across nodes
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: web-app
              topologyKey: kubernetes.io/hostname  # Spread per node
      containers:
      - name: web
        image: web-app:v2
```

### 16.3 Taints & Tolerations

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  TAINTS & TOLERATIONS — "REPELLING" PODS FROM NODES            │
  │                                                                  │
  │  TAINT (on node):  "I don't accept pods UNLESS they tolerate"  │
  │  TOLERATION (on pod): "I'm okay with being on a tainted node"  │
  │                                                                  │
  │  Taint Effects:                                                  │
  │  ├── NoSchedule:      Won't schedule new pods (existing stay)  │
  │  ├── PreferNoSchedule: Soft version (try not to schedule)      │
  │  └── NoExecute:       Evict existing pods + prevent new ones   │
  │                                                                  │
  │  Common Use Cases:                                               │
  │  ┌────────────────────────────────────────────────────────┐     │
  │  │  GPU Nodes:           taint = gpu=true:NoSchedule      │     │
  │  │  Only GPU pods have toleration → non-GPU pods excluded │     │
  │  │                                                         │     │
  │  │  Dedicated Nodes:     taint = team=data:NoSchedule     │     │
  │  │  Only data team pods have toleration                    │     │
  │  │                                                         │     │
  │  │  Control Plane:       taint = NoSchedule (auto-applied)│     │
  │  │  System pods (kube-proxy, CoreDNS) have tolerations    │     │
  │  │                                                         │     │
  │  │  Spot/Preemptible:    taint = spot=true:NoSchedule     │     │
  │  │  Only fault-tolerant workloads scheduled                │     │
  │  └────────────────────────────────────────────────────────┘     │
  └──────────────────────────────────────────────────────────────────┘
```

```bash
# Add taint to node
kubectl taint nodes node-1 gpu=true:NoSchedule

# Remove taint
kubectl taint nodes node-1 gpu=true:NoSchedule-

# View node taints
kubectl describe node node-1 | grep -A5 Taints
```

```yaml
# Pod with toleration (can run on tainted node)
spec:
  tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
  # Tolerate ALL NoSchedule taints (dangerous — use sparingly)
  - operator: "Exists"
    effect: "NoSchedule"
```

### 16.4 Topology Spread Constraints

```yaml
# Spread pods evenly across availability zones
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 6
  template:
    spec:
      topologySpreadConstraints:
      - maxSkew: 1                     # Max difference between zones
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule  # or ScheduleAnyway
        labelSelector:
          matchLabels:
            app: web-app
      - maxSkew: 1                     # Also spread across nodes
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: web-app
      containers:
      - name: web
        image: web-app:v2
```

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  TOPOLOGY SPREAD — RESULT WITH 6 REPLICAS, 3 ZONES            │
  │                                                                  │
  │  Without topology spread:         With topology spread:         │
  │  zone-a: 4 pods                   zone-a: 2 pods               │
  │  zone-b: 2 pods                   zone-b: 2 pods               │
  │  zone-c: 0 pods                   zone-c: 2 pods               │
  │  ⚠️ zone-c outage = no impact    ✅ any zone down = 4 pods    │
  │     but zone-a down = 4 pods lost     still serving             │
  └──────────────────────────────────────────────────────────────────┘
```

### 16.5 Pod Disruption Budgets (PDB)

```yaml
# PDB — protect availability during voluntary disruptions
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-app-pdb
  namespace: production
spec:
  # Use ONE of these (not both):
  minAvailable: 2                  # At least 2 pods must stay running
  # maxUnavailable: 1             # At most 1 pod can be down
  selector:
    matchLabels:
      app: web-app
```

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  PDB — WHEN IT APPLIES                                          │
  │                                                                  │
  │  Voluntary disruptions (PDB enforced):                          │
  │  ├── kubectl drain node (maintenance)                           │
  │  ├── Cluster autoscaler removing nodes                          │
  │  ├── kubectl delete pod (if using eviction API)                │
  │  └── Deployment rolling updates                                 │
  │                                                                  │
  │  Involuntary disruptions (PDB NOT enforced):                    │
  │  ├── Node hardware failure                                      │
  │  ├── Kernel panic / VM crash                                    │
  │  ├── OOM killer                                                 │
  │  └── Node network partition                                     │
  │                                                                  │
  │  Example: 3 replicas, PDB minAvailable=2                        │
  │  ├── Node drain: will evict at most 1 pod at a time            │
  │  ├── Waits for replacement pod before evicting next             │
  │  └── If only 2 pods running, drain will BLOCK                   │
  └──────────────────────────────────────────────────────────────────┘
```

### 16.6 Resource Requests vs Limits — Deep Dive

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  REQUESTS vs LIMITS — WHAT REALLY HAPPENS                       │
  │                                                                  │
  │  REQUESTS:                                                       │
  │  ├── Scheduler uses to PLACE pods (guaranteed minimum)          │
  │  ├── Node must have this much available to accept pod           │
  │  ├── Affects QoS class (see below)                              │
  │  └── cpu: 200m = pod guaranteed 0.2 CPU cores                  │
  │                                                                  │
  │  LIMITS:                                                         │
  │  ├── Max a container CAN use                                    │
  │  ├── CPU: throttled (not killed) when exceeding limit           │
  │  ├── Memory: OOM-killed (container killed) when exceeding      │
  │  └── memory: 512Mi = if container uses > 512Mi → OOMKilled    │
  │                                                                  │
  │  QoS CLASSES (determined by requests/limits):                   │
  │  ┌──────────────────────────────────────────────────────────┐   │
  │  │  Guaranteed:  requests == limits (for all containers)    │   │
  │  │  → Highest priority, last to be evicted                  │   │
  │  │  → Use for: databases, critical services                 │   │
  │  │                                                           │   │
  │  │  Burstable:   requests < limits (or only requests set)   │   │
  │  │  → Medium priority, evicted after BestEffort             │   │
  │  │  → Use for: most workloads                               │   │
  │  │                                                           │   │
  │  │  BestEffort:  no requests or limits set at all           │   │
  │  │  → Lowest priority, evicted FIRST during pressure        │   │
  │  │  → NEVER use in production                               │   │
  │  └──────────────────────────────────────────────────────────┘   │
  │                                                                  │
  │  PRODUCTION RECOMMENDATION:                                      │
  │  Set requests = expected usage (from metrics)                   │
  │  Set CPU limits = 2-5x requests (allow bursting)               │
  │  Set memory limits = close to requests (OOM is fatal)          │
  │  Some teams AVOID CPU limits entirely (only requests)          │
  └──────────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha (CPU limits debate):** There's an active debate about whether to set CPU limits. CPU limits cause throttling even when the node has spare CPU. Google's internal guidance and many SRE teams (including Tim Hockin, K8s co-founder) recommend setting CPU requests but NOT CPU limits — let pods burst when CPU is available. Always set memory limits though — OOM is harder to recover from.

> **⚠️ Gotcha (PDB + Single Replica):** If you have `replicas: 1` and a PDB with `minAvailable: 1`, node drain will BLOCK forever — it can never evict the only pod. Either run at least 2 replicas or use `maxUnavailable: 1` instead.

> **DevOps Relevance:** Proper scheduling is critical for cost optimization, HA, and performance. Use topology spread constraints to survive AZ failures, taints to isolate workloads (GPU, spot instances), and PDBs to prevent disruptions during maintenance. Set resource requests based on actual metrics (Prometheus + VPA recommender) not guesswork — overprovisioning wastes money, underprovisioning causes OOMKills.

---

## 17. Monitoring & Observability

### 17.1 The Three Pillars of Observability

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  THREE PILLARS OF OBSERVABILITY                                  │
  │                                                                  │
  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
  │  │   METRICS    │  │    LOGS      │  │   TRACES     │          │
  │  │              │  │              │  │              │          │
  │  │ "What is     │  │ "What       │  │ "How does a  │          │
  │  │  happening?" │  │  happened?"  │  │  request     │          │
  │  │              │  │              │  │  flow?"      │          │
  │  │ Prometheus   │  │ EFK/Loki    │  │ Jaeger       │          │
  │  │ + Grafana    │  │ (Elastic,   │  │ Zipkin       │          │
  │  │              │  │  Fluent,    │  │ Tempo        │          │
  │  │ CPU, memory, │  │  Kibana)    │  │ OpenTelemetry│          │
  │  │ request rate │  │              │  │              │          │
  │  │ error rate   │  │ App errors,  │  │ Latency per  │          │
  │  │ latency p99  │  │ audit logs,  │  │ service hop, │          │
  │  │              │  │ system events│  │ bottleneck   │          │
  │  └──────────────┘  └──────────────┘  └──────────────┘          │
  │                                                                  │
  │  All three are needed — metrics tell you WHAT is wrong,         │
  │  logs tell you WHY, traces tell you WHERE in the chain.        │
  └──────────────────────────────────────────────────────────────────┘
```

### 17.2 Prometheus + Grafana Stack

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  PROMETHEUS + GRAFANA ARCHITECTURE                               │
  │                                                                  │
  │  ┌─────────────┐    scrape /metrics                             │
  │  │ Prometheus   │◄─────────────────── App Pods                  │
  │  │ (TSDB)       │◄─────────────────── Node Exporter             │
  │  │              │◄─────────────────── kube-state-metrics         │
  │  │              │◄─────────────────── cAdvisor (kubelet)        │
  │  └──────┬───────┘                                                │
  │         │ PromQL                                                 │
  │         ▼                                                        │
  │  ┌─────────────┐    ┌──────────────────┐                        │
  │  │  Grafana     │    │  AlertManager     │                        │
  │  │ (dashboards) │    │ (routing, silencing│                       │
  │  │              │    │  dedup, grouping)  │                       │
  │  └─────────────┘    └────────┬─────────┘                        │
  │                              │ Notifications                     │
  │                              ▼                                   │
  │                   ┌──────────────────────┐                      │
  │                   │ Slack, PagerDuty,    │                      │
  │                   │ OpsGenie, Email,     │                      │
  │                   │ Webhook              │                      │
  │                   └──────────────────────┘                      │
  │                                                                  │
  │  Install: helm install kube-prometheus-stack                    │
  │  (includes Prometheus, Grafana, AlertManager, exporters)        │
  └──────────────────────────────────────────────────────────────────┘
```

### 17.3 Key Metrics & PromQL

```yaml
# ServiceMonitor — tell Prometheus to scrape your app
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-monitor
  namespace: production
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
---
# PrometheusRule — define alerting rules
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: app-alerts
  namespace: production
spec:
  groups:
  - name: app.rules
    rules:
    - alert: HighErrorRate
      expr: |
        rate(http_requests_total{status=~"5.."}[5m])
        / rate(http_requests_total[5m]) > 0.05
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "High 5xx error rate (> 5%)"
    - alert: PodCrashLooping
      expr: |
        rate(kube_pod_container_status_restarts_total[15m]) * 60 * 15 > 3
      for: 5m
      labels:
        severity: warning
```

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  ESSENTIAL PROMQL QUERIES                                        │
  │                                                                  │
  │  # CPU usage by pod                                              │
  │  rate(container_cpu_usage_seconds_total{                        │
  │    namespace="production"}[5m])                                  │
  │                                                                  │
  │  # Memory usage by pod (in MiB)                                 │
  │  container_memory_working_set_bytes{                            │
  │    namespace="production"} / 1024^2                              │
  │                                                                  │
  │  # Request rate per service (RED method)                        │
  │  rate(http_requests_total[5m])                                  │
  │                                                                  │
  │  # Error rate percentage                                         │
  │  rate(http_requests_total{status=~"5.."}[5m])                   │
  │  / rate(http_requests_total[5m]) * 100                          │
  │                                                                  │
  │  # P99 latency                                                   │
  │  histogram_quantile(0.99,                                        │
  │    rate(http_request_duration_seconds_bucket[5m]))              │
  │                                                                  │
  │  # Pods not ready                                                │
  │  kube_pod_status_ready{condition="false"}                       │
  │                                                                  │
  │  # PVC usage percentage                                          │
  │  kubelet_volume_stats_used_bytes                                │
  │  / kubelet_volume_stats_capacity_bytes * 100                    │
  └──────────────────────────────────────────────────────────────────┘
```

### 17.4 Logging Stack — EFK / Loki

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  LOGGING STACKS COMPARISON                                       │
  │                                                                  │
  │  EFK Stack:                      Loki Stack (lightweight):      │
  │  ┌──────────┐                    ┌──────────┐                   │
  │  │Fluentd / │  (collect)         │Promtail /│  (collect)        │
  │  │Fluent Bit│                    │Fluent Bit│                   │
  │  └────┬─────┘                    └────┬─────┘                   │
  │       ▼                               ▼                          │
  │  ┌──────────┐                    ┌──────────┐                   │
  │  │Elastic-  │  (store/index)     │  Loki    │  (store — no     │
  │  │search    │                    │          │   full indexing)  │
  │  └────┬─────┘                    └────┬─────┘                   │
  │       ▼                               ▼                          │
  │  ┌──────────┐                    ┌──────────┐                   │
  │  │ Kibana   │  (visualize)       │ Grafana  │  (visualize)     │
  │  └──────────┘                    └──────────┘                   │
  │                                                                  │
  │  EFK: Powerful, full-text search, expensive (RAM-heavy)         │
  │  Loki: Lightweight, label-based, cheaper, fits Grafana stack   │
  │                                                                  │
  │  Cloud alternatives:                                             │
  │  ├── AWS: CloudWatch Logs, OpenSearch                           │
  │  ├── GCP: Cloud Logging (Stackdriver)                           │
  │  └── Azure: Azure Monitor Logs                                  │
  └──────────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha (Log volume):** Kubernetes generates massive log volumes. Without retention policies, Elasticsearch will consume all disk and OOM. Set index lifecycle management (ILM) in Elasticsearch or retention periods in Loki. A medium cluster can easily produce 50-100 GB/day of logs.

> **DevOps Relevance:** Use the RED method (Rate, Errors, Duration) for services and USE method (Utilization, Saturation, Errors) for infrastructure. Install kube-prometheus-stack via Helm as your first observability step — it includes pre-built Grafana dashboards. For cost-effective logging, prefer Loki over EFK unless you need full-text search.

---

## 18. Kubernetes Security Hardening

### 18.1 Security Layers

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  K8s SECURITY — DEFENSE IN DEPTH                                │
  │                                                                  │
  │  Layer 1: CLUSTER INFRASTRUCTURE                                │
  │  ├── Keep K8s version updated (patch CVEs)                      │
  │  ├── Encrypt etcd at rest                                        │
  │  ├── Use private API server endpoint                             │
  │  ├── Enable audit logging                                        │
  │  └── Rotate certificates regularly                               │
  │                                                                  │
  │  Layer 2: AUTHENTICATION & AUTHORIZATION                         │
  │  ├── RBAC (minimal privileges)                                   │
  │  ├── OIDC integration (no static tokens)                        │
  │  ├── Dedicated ServiceAccounts per workload                     │
  │  └── Disable anonymous access                                    │
  │                                                                  │
  │  Layer 3: NETWORK                                                │
  │  ├── Network Policies (zero-trust)                               │
  │  ├── Encrypt in-transit (mTLS via service mesh)                 │
  │  ├── Restrict egress (prevent data exfiltration)                │
  │  └── Use private node networks                                  │
  │                                                                  │
  │  Layer 4: WORKLOAD / POD                                        │
  │  ├── Pod Security Standards                                      │
  │  ├── Read-only root filesystem                                   │
  │  ├── Non-root containers                                         │
  │  ├── Drop all capabilities                                       │
  │  └── Resource limits (prevent DoS)                               │
  │                                                                  │
  │  Layer 5: SUPPLY CHAIN                                           │
  │  ├── Scan container images (Trivy, Snyk)                        │
  │  ├── Use minimal base images (distroless, alpine)               │
  │  ├── Sign images (cosign / Sigstore)                             │
  │  └── Enforce trusted registries (OPA/Kyverno)                   │
  └──────────────────────────────────────────────────────────────────┘
```

### 18.2 Pod Security Standards (PSS)

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  POD SECURITY STANDARDS (replaced PodSecurityPolicy in 1.25)    │
  │                                                                  │
  │  Three Profiles:                                                 │
  │  ┌────────────────────────────────────────────────────────┐     │
  │  │                                                         │     │
  │  │  PRIVILEGED    No restrictions at all                   │     │
  │  │  (system pods)  Use for: kube-system, CNI, storage     │     │
  │  │                                                         │     │
  │  │  BASELINE      Sensible defaults, prevent known        │     │
  │  │  (default)      escalations. Blocks: hostNetwork,      │     │
  │  │                 privileged containers, hostPath         │     │
  │  │                                                         │     │
  │  │  RESTRICTED    Maximum security. Requires: non-root,   │     │
  │  │  (hardened)     no caps, read-only rootfs, seccomp     │     │
  │  │                 Use for: all production workloads       │     │
  │  │                                                         │     │
  │  └────────────────────────────────────────────────────────┘     │
  │                                                                  │
  │  Enforcement Modes:                                              │
  │  ├── enforce: reject pods that violate                          │
  │  ├── audit:   log violations but allow                          │
  │  └── warn:    show warnings to users but allow                  │
  └──────────────────────────────────────────────────────────────────┘
```

```yaml
# Apply Pod Security Standards to a namespace
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
---
# Hardened pod spec (passes "restricted" PSS)
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: my-app:v2.0
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
    volumeMounts:
    - name: tmp
      mountPath: /tmp               # Writable tmp since rootfs is read-only
  volumes:
  - name: tmp
    emptyDir: {}
```

### 18.3 Image Security & Supply Chain

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  CONTAINER IMAGE SECURITY PIPELINE                               │
  │                                                                  │
  │  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐ │
  │  │  Build   │───▶│  Scan    │───▶│  Sign    │───▶│  Enforce │ │
  │  │          │    │          │    │          │    │          │ │
  │  │ Multi-   │    │ Trivy    │    │ cosign   │    │ Kyverno  │ │
  │  │ stage    │    │ Snyk     │    │ Notation │    │ OPA      │ │
  │  │ Distro-  │    │ Grype    │    │ Sigstore │    │ Connaiss.│ │
  │  │ less     │    │          │    │          │    │          │ │
  │  └──────────┘    └──────────┘    └──────────┘    └──────────┘ │
  │                                                                  │
  │  Best Practices:                                                 │
  │  ├── Use distroless or alpine base images (smaller attack      │
  │  │   surface — distroless has no shell, no package manager)    │
  │  ├── Multi-stage builds (build in one stage, run in another)   │
  │  ├── Pin image digests in production, not just tags            │
  │  │   image: nginx@sha256:abc123...  (immutable reference)      │
  │  ├── Scan in CI/CD pipeline — block deployment if critical CVE │
  │  └── Private registries with access control (ECR, ACR, GCR)   │
  └──────────────────────────────────────────────────────────────────┘
```

```bash
# Scan image with Trivy
trivy image my-app:v2.0
trivy image --severity HIGH,CRITICAL my-app:v2.0

# Sign image with cosign
cosign sign --key cosign.key my-registry/my-app:v2.0

# Verify signature before deploying
cosign verify --key cosign.pub my-registry/my-app:v2.0
```

```yaml
# Kyverno policy — enforce image registry
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-image-registries
spec:
  validationFailureAction: Enforce
  rules:
  - name: validate-registries
    match:
      any:
      - resources:
          kinds: ["Pod"]
    validate:
      message: "Images must be from approved registries"
      pattern:
        spec:
          containers:
          - image: "123456789.dkr.ecr.*.amazonaws.com/*"
```

> **⚠️ Gotcha (Read-only rootfs):** When enabling `readOnlyRootFilesystem: true`, many apps break because they write to `/tmp`, `/var/cache`, or log files. Mount `emptyDir` volumes at all writable paths your app needs. Test thoroughly before enforcing.

> **DevOps Relevance:** Security is non-negotiable for production. Start with: (1) Enable Pod Security Standards (`restricted`) on all non-system namespaces, (2) Scan images in CI with Trivy, (3) Run as non-root with dropped capabilities, (4) Apply Network Policies. Use Kyverno over OPA/Gatekeeper for simpler policy management — it uses K8s-native YAML instead of Rego.

---

## 19. Multi-Cluster, Federation & GitOps

### 19.1 Why Multi-Cluster?

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  MULTI-CLUSTER PATTERNS                                          │
  │                                                                  │
  │  1. ENVIRONMENT SEPARATION                                       │
  │  ┌────────────┐  ┌────────────┐  ┌────────────┐               │
  │  │ Dev Cluster│  │ Stg Cluster│  │ Prod Cluster│               │
  │  └────────────┘  └────────────┘  └────────────┘               │
  │  Blast radius: staging issue can't impact production           │
  │                                                                  │
  │  2. REGIONAL / GEO DISTRIBUTION                                  │
  │  ┌────────────┐  ┌────────────┐  ┌────────────┐               │
  │  │ US-East    │  │ EU-West    │  │ AP-South   │               │
  │  │ Cluster    │  │ Cluster    │  │ Cluster    │               │
  │  └────────────┘  └────────────┘  └────────────┘               │
  │  Lower latency for global users, data sovereignty             │
  │                                                                  │
  │  3. HYBRID / MULTI-CLOUD                                         │
  │  ┌────────────┐  ┌────────────┐  ┌────────────┐               │
  │  │ AWS EKS    │  │ GCP GKE   │  │ On-Prem    │               │
  │  └────────────┘  └────────────┘  └────────────┘               │
  │  Avoid vendor lock-in, leverage best services per cloud       │
  │                                                                  │
  │  Management tools:                                               │
  │  ├── Argo CD (multi-cluster GitOps)                             │
  │  ├── Rancher (multi-cluster management UI)                      │
  │  ├── Loft / vCluster (virtual clusters)                         │
  │  └── Crossplane (infrastructure as CRDs)                        │
  └──────────────────────────────────────────────────────────────────┘
```

### 19.2 GitOps with Argo CD

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  GITOPS — GIT AS THE SINGLE SOURCE OF TRUTH                    │
  │                                                                  │
  │  Traditional CI/CD:               GitOps:                        │
  │  ├── CI pushes to cluster         ├── CI pushes to Git repo    │
  │  ├── kubectl apply in pipeline    ├── Argo CD PULLS from Git   │
  │  ├── Cluster state unknown        ├── Git = desired state      │
  │  └── "Did someone kubectl edit?"  └── Drift auto-corrected    │
  │                                                                  │
  │  ┌──────────────────────────────────────────────────────────┐   │
  │  │                     GITOPS FLOW                           │   │
  │  │                                                           │   │
  │  │  Developer ──▶ PR to Git repo ──▶ Merge to main         │   │
  │  │                                       │                   │   │
  │  │                                       ▼                   │   │
  │  │                              ┌───────────────┐           │   │
  │  │                              │   Argo CD     │           │   │
  │  │                              │ (watches repo) │           │   │
  │  │                              └───────┬───────┘           │   │
  │  │                                      │ sync              │   │
  │  │                                      ▼                   │   │
  │  │                              ┌───────────────┐           │   │
  │  │                              │  K8s Cluster  │           │   │
  │  │                              │  (desired ==  │           │   │
  │  │                              │   actual)     │           │   │
  │  │                              └───────────────┘           │   │
  │  │                                                           │   │
  │  │  Rollback = git revert → Argo CD auto-syncs             │   │
  │  └──────────────────────────────────────────────────────────┘   │
  │                                                                  │
  │  GitOps Principles:                                              │
  │  ├── 1. Declarative: all config in YAML/Helm/Kustomize         │
  │  ├── 2. Versioned: Git history = audit trail                   │
  │  ├── 3. Automated: changes auto-applied by operator            │
  │  └── 4. Self-healing: drift detected and corrected             │
  └──────────────────────────────────────────────────────────────────┘
```

```yaml
# Argo CD Application CRD
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/k8s-manifests.git
    targetRevision: main
    path: apps/my-app/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true                  # Delete resources removed from Git
      selfHeal: true               # Revert manual changes to cluster
    syncOptions:
    - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### 19.3 Argo CD vs Flux Comparison

| Feature | Argo CD | Flux v2 |
|---------|---------|---------|
| **UI** | Rich web UI + CLI | CLI only (or Weave GitOps UI) |
| **Multi-cluster** | Built-in (register clusters) | Via Kustomization CRDs |
| **Helm support** | Native | Via HelmRelease CRD |
| **OCI support** | Helm OCI charts | Full OCI support |
| **RBAC** | Built-in, SSO integration | K8s native RBAC |
| **Notifications** | Argo CD Notifications | Notification Controller |
| **Architecture** | Centralized (one Argo CD) | Distributed (per cluster) |
| **Best for** | Teams wanting UI + visibility | Teams preferring K8s-native |
| **CNCF status** | Graduated | Graduated |

> **⚠️ Gotcha (GitOps secrets):** Never store plain secrets in Git. Use Sealed Secrets, External Secrets Operator, or SOPS to encrypt secrets in your Git repo. Argo CD integrates with all of these via plugins.

> **DevOps Relevance:** GitOps is the standard deployment model for production Kubernetes. Argo CD is the most popular choice for its UI, multi-cluster support, and RBAC. Structure your Git repos either as monorepo (all apps in one repo) or multi-repo (one repo per team). Use ApplicationSets to template apps across environments/clusters.

---

## 20. Production Scenario-Based FAQ

### FAQ 1: Pod stuck in `Pending` state

```
Q: My pod has been Pending for 10 minutes. How do I debug?

A: "Pending" means the scheduler can't find a suitable node.

  kubectl describe pod <pod-name> -n <namespace>
  → Check the "Events" section at the bottom

Common causes:
├── Insufficient resources:
│   "0/5 nodes are available: 5 Insufficient cpu"
│   Fix: Scale up cluster or reduce resource requests
│
├── No matching node (nodeSelector/affinity):
│   "0/5 nodes are available: 5 node(s) didn't match Pod's
│   node selector"
│   Fix: Check labels on nodes: kubectl get nodes --show-labels
│
├── Taints not tolerated:
│   "0/5 nodes are available: 5 node(s) had untolerated taint"
│   Fix: Add toleration to pod spec or remove taint from node
│
├── PVC not bound:
│   "persistentvolumeclaim 'data' not found" or PVC Pending
│   Fix: Check StorageClass, PV availability, and AZ matching
│
└── Too many pods on node:
    Fix: Check maxPods setting on kubelet
```

### FAQ 2: Pod stuck in `CrashLoopBackOff`

```
Q: My pod keeps restarting. How do I fix CrashLoopBackOff?

A: The container is starting and crashing repeatedly.

  kubectl logs <pod> -n <ns>                # Current logs
  kubectl logs <pod> -n <ns> --previous     # Previous crash logs
  kubectl describe pod <pod> -n <ns>        # Exit codes & events

Common causes:
├── Exit Code 1 (application error):
│   App crashed — check logs for stack trace / config error
│   Fix: Fix application code or configuration
│
├── Exit Code 137 (OOMKilled):
│   Container exceeded memory limit
│   Fix: Increase memory limits or fix memory leak
│   kubectl describe pod → "OOMKilled" in Last State
│
├── Exit Code 0 but still restarting:
│   App exited successfully but restartPolicy is Always
│   Fix: Use a Job instead of Deployment for one-shot tasks
│
├── Failing health probes:
│   Liveness probe failing → kubelet kills container
│   Fix: Check probe path/port, increase initialDelaySeconds
│
└── Missing ConfigMap/Secret:
    "Error: configmap 'app-config' not found"
    Fix: Create the missing ConfigMap/Secret first
```

### FAQ 3: Service has no endpoints

```
Q: My Service exists but traffic isn't reaching pods.

A: The Service selector doesn't match any running pods.

  kubectl get endpoints <service-name> -n <ns>
  → If ENDPOINTS is empty → selector mismatch

  kubectl describe svc <service-name> -n <ns>
  → Check the Selector field

  kubectl get pods -n <ns> --show-labels
  → Verify pod labels match service selector

Checklist:
├── Labels match? Service selector must match pod labels exactly
├── Pods running? kubectl get pods -n <ns> (are they Ready?)
├── Port correct? Service targetPort must match container port
├── Right namespace? Service and pods must be in same namespace
└── Pods passing readiness probe? Only Ready pods get endpoints
```

### FAQ 4: Ingress returning 502/503/504

```
Q: My Ingress returns 502 Bad Gateway or 503 Service Unavailable.

A: Traffic reaches the Ingress controller but can't reach backend.

  502 Bad Gateway:
  ├── Backend service is down or unreachable
  ├── Fix: Check if backend pods are Running and Ready
  └── Fix: Verify Service has endpoints

  503 Service Unavailable:
  ├── No backends available
  ├── Fix: Check pod count (maybe scaled to 0 or all failing probes)
  └── Fix: Check if HPA scaled down too aggressively

  504 Gateway Timeout:
  ├── Backend is too slow to respond
  ├── Fix: Increase proxy-read-timeout annotation
  │   nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
  └── Fix: Optimize backend performance

  Debug steps:
  kubectl get pods -n <ns>                    # Pods running?
  kubectl get endpoints <svc> -n <ns>         # Endpoints exist?
  kubectl logs <ingress-controller-pod> -n ingress  # Controller logs
```

### FAQ 5: Node is `NotReady`

```
Q: A node shows NotReady. What should I do?

A: The kubelet on that node stopped reporting to the API server.

  kubectl describe node <node-name>
  → Check Conditions section (MemoryPressure, DiskPressure, PIDPressure)

Common causes:
├── Kubelet crashed:
│   SSH to node → systemctl status kubelet → journalctl -u kubelet
│
├── Disk pressure:
│   Node ran out of disk → kubelet marks NotReady
│   Fix: Clean up images: docker system prune / crictl rmi --prune
│
├── Memory pressure:
│   Fix: Identify memory-hungry pods, set proper resource limits
│
├── Network issues:
│   Node can't reach API server
│   Fix: Check security groups, firewall rules, VPN
│
└── Certificate expired:
    kubelet can't authenticate to API server
    Fix: Rotate certificates (kubeadm certs renew)

  Immediate action:
  kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
  (reschedule pods to healthy nodes while investigating)
```

### FAQ 6: Persistent Volume stuck in `Terminating`

```
Q: I deleted a PVC but it's stuck in Terminating.

A: A pod is still using the PVC (finalizer protection).

  kubectl get pvc <name> -n <ns> -o yaml | grep finalizers
  → Shows: kubernetes.io/pvc-protection

  Fix properly:
  1. Delete the pod using the PVC first
  2. PVC will then terminate automatically

  Force delete (dangerous — may cause data loss):
  kubectl patch pvc <name> -n <ns> -p '{"metadata":{"finalizers":null}}'
```

### FAQ 7: Rolling update stuck

```
Q: My deployment update is stuck. New pods won't become Ready.

A:
  kubectl rollout status deployment/<name> -n <ns>
  kubectl get pods -n <ns>                    # Check new pods status
  kubectl logs <new-pod> -n <ns>              # Check for errors
  kubectl describe pod <new-pod> -n <ns>      # Check events

Common causes:
├── New image doesn't exist or can't be pulled
│   "ImagePullBackOff" → Check image name/tag and pull secrets
│
├── New version crashes:
│   Fix: kubectl rollout undo deployment/<name> -n <ns>
│
├── Not enough cluster resources for new + old pods:
│   maxSurge pods need resources too
│   Fix: Reduce maxSurge or add cluster capacity
│
└── Readiness probe failing on new version:
    Fix: Check probe configuration or fix app health endpoint

  Rollback: kubectl rollout undo deployment/<name> -n <ns>
  History: kubectl rollout history deployment/<name> -n <ns>
```

### FAQ 8: How to handle Kubernetes upgrades safely

```
Q: How do I upgrade my cluster without downtime?

A: Follow this step-by-step process:

  PRE-UPGRADE:
  ├── 1. Read the changelog for BREAKING CHANGES
  │      (K8s removes APIs — e.g., ingress v1beta1 removed in 1.22)
  ├── 2. Check deprecated API usage:
  │      kubectl deprecations (pluto scan)
  ├── 3. Ensure PDBs exist for critical workloads
  ├── 4. Test upgrade on staging cluster FIRST
  └── 5. Backup etcd snapshot

  UPGRADE PROCESS (managed K8s):
  ├── 1. Upgrade control plane (EKS/GKE/AKS handles this)
  ├── 2. Upgrade node pools one at a time:
  │      ├── Create new node pool with new version
  │      ├── Cordon old nodes (prevent new pods)
  │      ├── Drain old nodes (evict pods gracefully)
  │      └── Delete old node pool
  └── 3. Verify all workloads healthy after each step

  RULE: Only upgrade ONE minor version at a time (1.27→1.28, never 1.27→1.29)
```

### FAQ 9: How to right-size resource requests

```
Q: How do I set the right CPU/memory requests?

A: Use actual usage data, not guesswork.

  Step 1: Install metrics-server + Prometheus
  Step 2: Run workloads for 1-2 weeks under normal load
  Step 3: Query actual usage:

  # CPU — P95 usage over 7 days
  quantile_over_time(0.95,
    rate(container_cpu_usage_seconds_total{
      namespace="production",
      container="my-app"}[5m])[7d:])

  # Memory — max usage over 7 days
  max_over_time(
    container_memory_working_set_bytes{
      namespace="production",
      container="my-app"}[7d])

  Step 4: Set requests = P95 usage + 20% buffer
  Step 5: Set memory limits = P99 usage + 30% buffer
  Step 6: (Optional) Install VPA in recommend-only mode

  Tools: Kubecost, Goldilocks (VPA recommendations dashboard)
```

### FAQ 10: Debugging DNS resolution failures

```
Q: My pods can't resolve service names.

A: DNS issues are common and often subtle.

  kubectl run debug --image=busybox -it --rm -- nslookup <service-name>
  kubectl run debug --image=busybox -it --rm -- nslookup \
    <service>.<namespace>.svc.cluster.local

  Checklist:
  ├── CoreDNS pods running?
  │   kubectl get pods -n kube-system -l k8s-app=kube-dns
  │
  ├── CoreDNS has capacity?
  │   kubectl logs -n kube-system -l k8s-app=kube-dns
  │   (look for "SERVFAIL" or "i/o timeout")
  │
  ├── Network Policy blocking DNS?
  │   Port 53 UDP/TCP must be allowed to kube-system
  │
  ├── Correct search domain?
  │   kubectl exec <pod> -- cat /etc/resolv.conf
  │   Should show: search <ns>.svc.cluster.local svc.cluster.local
  │
  └── ndots:5 causing slow lookups?
      External domains try 5 internal suffixes first
      Fix: Set dnsConfig ndots: 2 in pod spec for external-heavy apps
```

### FAQ 11: How to set up zero-downtime deployments

```
Q: How do I deploy without any user-facing downtime?

A: Combine these four elements:

  1. READINESS PROBES (don't send traffic until app is ready)
     readinessProbe:
       httpGet:
         path: /healthz
         port: 8080
       initialDelaySeconds: 5
       periodSeconds: 5

  2. ROLLING UPDATE STRATEGY
     strategy:
       type: RollingUpdate
       rollingUpdate:
         maxSurge: 1          # Create 1 new before removing old
         maxUnavailable: 0    # Never have fewer than desired count

  3. PREEMPTIVE CONNECTION DRAINING
     spec:
       terminationGracePeriodSeconds: 60
       containers:
       - lifecycle:
           preStop:
             exec:
               command: ["sh", "-c", "sleep 10"]
     # Why: Endpoints update is async — old pod may still receive
     # traffic for a few seconds after SIGTERM. Sleep lets in-flight
     # requests complete.

  4. POD DISRUPTION BUDGET
     minAvailable: 80%  (or absolute number)
```

### FAQ 12: Cost optimization strategies

```
Q: Our K8s cluster costs are too high. How to optimize?

A: Common cost reduction strategies:

  1. RIGHT-SIZE RESOURCES (biggest impact — 30-50% savings)
  ├── Use VPA recommendations or Goldilocks dashboard
  ├── Most teams over-provision by 50-70%
  └── Set requests = actual P95 usage, not "just in case" values

  2. SPOT/PREEMPTIBLE NODES (60-90% cheaper)
  ├── Run stateless workloads on spot instances
  ├── Use taints + tolerations to isolate
  └── PDBs + multiple AZs for fault tolerance

  3. CLUSTER AUTOSCALER / KARPENTER
  ├── Scale nodes down during off-hours
  ├── Karpenter (AWS) is faster and smarter than Cluster Autoscaler
  └── Configure scale-down-delay to avoid flapping

  4. NAMESPACE RESOURCE QUOTAS
  ├── Prevent teams from requesting unlimited resources
  └── Force developers to think about resource needs

  5. MONITORING COSTS
  ├── Kubecost — per-namespace, per-pod cost visibility
  ├── AWS Cost Explorer with K8s tags
  └── Set up cost alerts and chargeback reports
```

### FAQ 13: etcd performance issues

```
Q: API server is slow. How do I check etcd health?

A:
  # Check etcd cluster health
  kubectl get --raw /healthz/etcd

  # etcd metrics (if accessible)
  etcdctl endpoint health
  etcdctl endpoint status --write-out=table

  Common issues:
  ├── Too many objects: etcd has millions of events
  │   Fix: Reduce event TTL, clean up completed Jobs
  │
  ├── Slow disk: etcd needs fast SSDs (< 10ms fsync)
  │   Fix: Use provisioned IOPS SSDs (io1/io2 on AWS)
  │
  ├── Large objects: Secrets/ConfigMaps > 1MB
  │   Fix: Split large configs, use external stores
  │
  └── No compaction: db size keeps growing
      Fix: Ensure auto-compaction is enabled
```

### FAQ 14: Multi-tenant cluster isolation

```
Q: Multiple teams share one cluster. How do I isolate them?

A: Layer these isolation mechanisms:

  ┌─────────────────────────────────────────────────────────┐
  │  ISOLATION LAYERS                                        │
  │                                                          │
  │  1. Namespaces          (logical separation)            │
  │  2. RBAC                (permission boundaries)          │
  │  3. ResourceQuotas      (resource limits per team)      │
  │  4. LimitRanges         (per-pod defaults/maxes)         │
  │  5. Network Policies    (network isolation)              │
  │  6. Pod Security Stds   (workload restrictions)          │
  │  7. Priority Classes    (scheduling guarantees)          │
  └─────────────────────────────────────────────────────────┘

  For stronger isolation:
  ├── vCluster: virtual clusters within a physical cluster
  ├── Separate node pools per tenant (with taints)
  └── Separate clusters per tenant (strongest, most expensive)
```

### FAQ 15: Disaster recovery for Kubernetes

```
Q: How do I back up and restore my K8s cluster?

A: Back up both cluster state AND persistent data.

  CLUSTER STATE BACKUP:
  ├── Velero (standard tool):
  │   velero backup create daily-backup \
  │     --include-namespaces production,staging
  │   velero restore create --from-backup daily-backup
  │
  ├── etcd snapshots (full cluster backup):
  │   etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db
  │
  └── Git repo (GitOps = automatic state backup):
      All manifests in Git → cluster is rebuildable from Git

  PERSISTENT DATA BACKUP:
  ├── VolumeSnapshots (K8s native):
  │   apiVersion: snapshot.storage.k8s.io/v1
  │   kind: VolumeSnapshot
  │   Create CSI snapshots of PVCs
  │
  ├── Application-level backups:
  │   pg_dump, mysqldump, mongodump (more reliable than PV snaps)
  │
  └── Cross-region replication:
      Replicate PVs to another region for DR

  RECOVERY STRATEGIES:
  ├── RPO (Recovery Point Objective): How much data can you lose?
  │   → Schedule backup frequency accordingly
  │
  └── RTO (Recovery Time Objective): How fast must you recover?
      → Practice restores regularly (fire drills)
      → Document the full recovery runbook

  Pro tip: Test your backups! An untested backup is not a backup.
```

> **DevOps Relevance:** These 15 scenarios represent the most common production issues. Bookmark this FAQ and use it as a first-response troubleshooting guide. The key to fast resolution is following a systematic debug flow: describe → logs → events → metrics. Practice these scenarios in a staging cluster so you're prepared when they happen in production at 3 AM.

---

[🏠 Home](../README.md) · [Kubernetes](README.md)
