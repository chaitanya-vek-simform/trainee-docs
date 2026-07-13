# ☸️ Kubernetes (kubectl) — Complete Command Reference

> All commands grouped by resource type with usage explanation.

---

## 1. Cluster & Context

```bash
# List all clusters in your kubeconfig
kubectl config get-contexts

# Switch to a different cluster
kubectl config use-context aks-prod-cluster

# Show current active cluster
kubectl config current-context

# Show cluster API server + DNS endpoint
kubectl cluster-info

# Show detailed kubeconfig file
kubectl config view

# Rename a context
kubectl config rename-context old-name new-name

# Delete a context from kubeconfig
kubectl config delete-context my-old-cluster

# Set default namespace for current context
kubectl config set-context --current --namespace=production
```

---

## 2. Pods

```bash
# List pods in current namespace
kubectl get pods

# List pods in a specific namespace
kubectl get pods -n production

# List pods across ALL namespaces
kubectl get pods -A

# List pods with extra info (node, IP)
kubectl get pods -o wide

# List pods and auto-refresh every 2 seconds
kubectl get pods -w

# Filter pods by label
kubectl get pods -l app=myapp
kubectl get pods -l app=myapp,env=prod

# Show pod details: events, conditions, volumes, resource limits
kubectl describe pod myapp-abc123 -n production

# Get pod logs
kubectl logs myapp-abc123

# Follow (stream) logs live
kubectl logs -f myapp-abc123

# Get last 100 lines
kubectl logs --tail=100 myapp-abc123

# Logs from previous (crashed) container
kubectl logs myapp-abc123 --previous

# Logs from a specific container in a multi-container pod
kubectl logs myapp-abc123 -c sidecar-container

# Shell into a running pod
kubectl exec -it myapp-abc123 -- bash
kubectl exec -it myapp-abc123 -- sh          # if bash not available

# Run a single command inside pod without interactive shell
kubectl exec myapp-abc123 -- cat /etc/hosts
kubectl exec myapp-abc123 -- df -h

# Run a temporary debug pod (auto-deleted after exit)
kubectl run debug --rm -it --image=busybox -- sh
kubectl run debug --rm -it --image=curlimages/curl -- sh

# Copy file from pod to local machine
kubectl cp myapp-abc123:/app/logs/error.log ./error.log

# Copy file from local to pod
kubectl cp ./config.yaml myapp-abc123:/app/config.yaml

# Delete a pod (Deployment will recreate it automatically)
kubectl delete pod myapp-abc123

# Delete all pods matching a label (Deployment recreates them)
kubectl delete pods -l app=myapp

# Force delete a stuck pod (skips graceful shutdown)
kubectl delete pod myapp-abc123 --force --grace-period=0
```

---

## 3. Deployments

```bash
# List deployments
kubectl get deployments
kubectl get deploy -n production

# Show full details: replicas, strategy, conditions
kubectl describe deployment myapp

# Create/update from YAML file
kubectl apply -f deployment.yaml

# Apply all YAML files in a directory
kubectl apply -f ./k8s/

# Preview what apply would change (dry run)
kubectl apply -f deployment.yaml --dry-run=client

# Scale replicas manually
kubectl scale deployment myapp --replicas=5

# Update container image (triggers rolling update)
kubectl set image deployment/myapp app=myapp:v2.1.0

# Watch rolling update progress
kubectl rollout status deployment/myapp

# Pause a rolling update
kubectl rollout pause deployment/myapp

# Resume a paused rollout
kubectl rollout resume deployment/myapp

# Show rollout revision history
kubectl rollout history deployment/myapp

# Show details of a specific revision
kubectl rollout history deployment/myapp --revision=3

# Rollback to previous version
kubectl rollout undo deployment/myapp

# Rollback to a specific revision number
kubectl rollout undo deployment/myapp --to-revision=2

# Edit deployment live (opens in vi)
kubectl edit deployment myapp

# Delete a deployment (removes all pods managed by it)
kubectl delete deployment myapp
kubectl delete -f deployment.yaml
```

---

## 4. Services

```bash
# List services
kubectl get services
kubectl get svc -n production

# Describe service (ports, selector, endpoints)
kubectl describe service myapp-svc

# Expose a deployment as a service
kubectl expose deployment myapp --port=80 --target-port=8080 --type=ClusterIP

# Forward a local port to a service (for local debugging)
kubectl port-forward svc/myapp 8080:80

# Forward to a specific pod
kubectl port-forward pod/myapp-abc123 8080:8080

# Delete service
kubectl delete svc myapp-svc
```

---

## 5. Ingress

```bash
# List ingress resources
kubectl get ingress
kubectl get ingress -n production

# Describe ingress (rules, TLS, host)
kubectl describe ingress myapp-ingress

# Apply ingress from file
kubectl apply -f ingress.yaml

# Delete ingress
kubectl delete ingress myapp-ingress
```

---

## 6. ConfigMaps

```bash
# List configmaps
kubectl get configmaps
kubectl get cm -n production

# View configmap contents
kubectl describe configmap app-config
kubectl get configmap app-config -o yaml

# Create configmap from literal values
kubectl create configmap app-config \
  --from-literal=DB_HOST=localhost \
  --from-literal=DB_PORT=5432

# Create configmap from a file
kubectl create configmap nginx-conf --from-file=nginx.conf

# Create configmap from a directory (all files become keys)
kubectl create configmap app-configs --from-file=./configs/

# Edit configmap live
kubectl edit configmap app-config

# Delete configmap
kubectl delete configmap app-config
```

---

## 7. Secrets

```bash
# List secrets
kubectl get secrets
kubectl get secrets -n production

# Describe secret (shows key names but NOT values)
kubectl describe secret myapp-secret

# View secret raw (values are base64 encoded)
kubectl get secret myapp-secret -o yaml

# Decode a specific secret value
kubectl get secret myapp-secret \
  -o jsonpath='{.data.password}' | base64 -d

# Create a generic secret from literal
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=sup3rs3cr3t

# Create secret from file
kubectl create secret generic tls-cert \
  --from-file=tls.crt=./cert.crt \
  --from-file=tls.key=./cert.key

# Create TLS secret directly
kubectl create secret tls myapp-tls \
  --cert=cert.crt --key=cert.key

# Create Docker registry secret (for private image pulls)
kubectl create secret docker-registry acr-auth \
  --docker-server=myacr.azurecr.io \
  --docker-username=myacr \
  --docker-password=<token>

# Delete secret
kubectl delete secret myapp-secret
```

---

## 8. Namespaces

```bash
# List namespaces
kubectl get namespaces
kubectl get ns

# Create namespace
kubectl create namespace staging

# Apply namespace from YAML
kubectl apply -f namespace.yaml

# Set a label on namespace (required for Network Policies)
kubectl label namespace production environment=prod

# Show all resources in a namespace
kubectl get all -n staging

# Delete namespace (deletes EVERYTHING inside it)
kubectl delete namespace staging
```

---

## 9. Nodes

```bash
# List nodes and their status
kubectl get nodes

# List nodes with more details (OS, container runtime, version)
kubectl get nodes -o wide

# Show detailed info: labels, conditions, capacity, allocatable
kubectl describe node node-1

# Show CPU and memory usage per node (requires metrics-server)
kubectl top nodes

# Show CPU and memory usage per pod
kubectl top pods
kubectl top pods -n production --sort-by=memory

# Cordon a node (mark as unschedulable — no new pods placed here)
kubectl cordon node-1

# Uncordon a node (re-enable scheduling)
kubectl uncordon node-1

# Drain a node (evict all pods before maintenance)
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data

# Add a label to a node
kubectl label node node-1 disktype=ssd

# Remove a label from a node
kubectl label node node-1 disktype-

# Taint a node (repel pods without matching tolerations)
kubectl taint nodes node-1 dedicated=gpu:NoSchedule

# Remove taint
kubectl taint nodes node-1 dedicated=gpu:NoSchedule-
```

---

## 10. Resource Quotas & Limits

```bash
# List resource quotas in a namespace
kubectl get resourcequota -n production

# Describe resource quota (used vs limit)
kubectl describe resourcequota -n production

# List limit ranges (default requests/limits for pods)
kubectl get limitrange -n production

# Describe limit range
kubectl describe limitrange -n production
```

---

## 11. HPA (Horizontal Pod Autoscaler)

```bash
# List HPAs
kubectl get hpa
kubectl get hpa -n production

# Describe HPA (current metrics, targets, replicas)
kubectl describe hpa myapp-hpa

# Create HPA on a deployment
kubectl autoscale deployment myapp \
  --min=2 --max=10 --cpu-percent=70

# Watch HPA scale in real time
kubectl get hpa -w

# Delete HPA
kubectl delete hpa myapp-hpa
```

---

## 12. Jobs & CronJobs

```bash
# List jobs
kubectl get jobs
kubectl get jobs -n production

# Describe job (pod status, completions)
kubectl describe job db-migration

# List cronjobs
kubectl get cronjobs

# Manually trigger a cronjob immediately
kubectl create job --from=cronjob/db-backup manual-backup-$(date +%s)

# Suspend a cronjob
kubectl patch cronjob db-backup -p '{"spec":{"suspend":true}}'

# Resume a cronjob
kubectl patch cronjob db-backup -p '{"spec":{"suspend":false}}'

# Delete a job
kubectl delete job db-migration

# Delete a cronjob
kubectl delete cronjob db-backup
```

---

## 13. StatefulSets

```bash
# List statefulsets
kubectl get statefulsets
kubectl get sts -n databases

# Describe statefulset
kubectl describe statefulset mysql

# Scale statefulset
kubectl scale statefulset mysql --replicas=3

# Watch rollout (statefulsets update one pod at a time)
kubectl rollout status statefulset/mysql

# Rollback statefulset
kubectl rollout undo statefulset/mysql
```

---

## 14. DaemonSets

```bash
# List daemonsets (runs one pod on every node)
kubectl get daemonsets
kubectl get ds -n monitoring

# Describe daemonset
kubectl describe daemonset fluent-bit

# Watch daemonset rollout
kubectl rollout status daemonset/fluent-bit

# Rollback daemonset
kubectl rollout undo daemonset/fluent-bit
```

---

## 15. Persistent Volumes & Claims

```bash
# List persistent volumes (cluster-wide)
kubectl get persistentvolumes
kubectl get pv

# List persistent volume claims (namespace-scoped)
kubectl get persistentvolumeclaims
kubectl get pvc -n databases

# Describe PVC (bound status, storage class, access mode)
kubectl describe pvc mysql-data -n databases

# Delete PVC (may delete underlying data — be careful!)
kubectl delete pvc mysql-data -n databases

# List storage classes
kubectl get storageclasses
kubectl get sc
```

---

## 16. RBAC (Role Based Access Control)

```bash
# List roles (namespace-scoped permissions)
kubectl get roles -n production

# List cluster roles (cluster-wide permissions)
kubectl get clusterroles

# Describe a role (what verbs on what resources)
kubectl describe role pod-reader -n production

# List role bindings (who gets which role)
kubectl get rolebindings -n production
kubectl get clusterrolebindings

# Check what permissions a user/serviceaccount has
kubectl auth can-i get pods --as=jane
kubectl auth can-i create deployments --as=system:serviceaccount:production:myapp-sa

# Check all permissions for current user
kubectl auth can-i --list

# Create a service account
kubectl create serviceaccount myapp-sa -n production
```

---

## 17. Network Policies

```bash
# List network policies
kubectl get networkpolicies -n production
kubectl get netpol -n production

# Describe network policy (ingress/egress rules)
kubectl describe networkpolicy deny-all -n production

# Apply network policy from file
kubectl apply -f network-policy.yaml

# Delete network policy
kubectl delete networkpolicy deny-all -n production
```

---

## 18. Events & Debugging

```bash
# Show recent events (sorted by time)
kubectl get events --sort-by='.lastTimestamp'

# Show only Warning events
kubectl get events --field-selector type=Warning -n production

# Show events for a specific resource
kubectl get events --field-selector involvedObject.name=myapp-abc123

# Get ALL resource types in a namespace
kubectl get all -n production

# List ALL resources of every type (API resources)
kubectl api-resources

# Explain any resource field (built-in docs)
kubectl explain pod.spec.containers.resources
kubectl explain deployment.spec.strategy

# Get resource as JSON for scripting
kubectl get pod myapp-abc123 -o json
kubectl get pod myapp-abc123 -o jsonpath='{.status.podIP}'

# Get multiple fields with custom columns
kubectl get pods \
  -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName

# Diff what would change before applying
kubectl diff -f deployment.yaml
```

---

## 19. Helm (Kubernetes Package Manager)

```bash
# Add a chart repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add jetstack https://charts.jetstack.io

# Update repo cache
helm repo update

# Search for a chart
helm search repo nginx

# Show chart default values
helm show values ingress-nginx/ingress-nginx

# Install a chart
helm install my-nginx ingress-nginx/ingress-nginx -n ingress --create-namespace

# Install with custom values
helm install my-nginx ingress-nginx/ingress-nginx -f my-values.yaml

# Upgrade an existing release
helm upgrade my-nginx ingress-nginx/ingress-nginx

# Install or upgrade (idempotent)
helm upgrade --install my-nginx ingress-nginx/ingress-nginx \
  --namespace ingress --create-namespace \
  --set controller.replicaCount=2

# List installed releases
helm list -A

# Show history of a release
helm history my-nginx

# Rollback to previous version
helm rollback my-nginx

# Rollback to specific revision
helm rollback my-nginx 2

# Get values currently deployed
helm get values my-nginx

# Render templates locally without deploying
helm template my-nginx ingress-nginx/ingress-nginx -f my-values.yaml

# Lint a chart for errors
helm lint ./charts/myapp

# Uninstall a release
helm uninstall my-nginx -n ingress
```

---

## 20. Quick Troubleshooting Cheatsheet

```
SYMPTOM                          COMMAND TO RUN
───────────────────────────────────────────────────────────────────
Pod stuck in Pending             kubectl describe pod <name>
                                 → Check Events: resource quota? node selector?

Pod in CrashLoopBackOff          kubectl logs <pod> --previous
                                 kubectl describe pod <name>
                                 → Check: OOMKilled? Config error?

Pod in ImagePullBackOff          kubectl describe pod <name>
                                 → Check: image name typo? registry auth?

Pod in Terminating (stuck)       kubectl delete pod <name> --force --grace-period=0

Service not reachable            kubectl describe svc <name>
                                 → Check selector matches pod labels
                                 kubectl get endpoints <svc-name>

Node NotReady                    kubectl describe node <name>
                                 → Check Conditions, Events

HPA not scaling                  kubectl describe hpa <name>
                                 → Check: metrics-server installed?

PVC stuck in Pending             kubectl describe pvc <name>
                                 → Check: storage class exists?

Check if app responds            kubectl port-forward svc/myapp 8080:80
                                 curl http://localhost:8080/health

No permissions error             kubectl auth can-i <verb> <resource>
```
