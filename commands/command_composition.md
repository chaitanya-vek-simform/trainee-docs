# 🧠 DevOps Command Composition — Thinking Process & Complex Commands

> How to build complex commands from scratch, the mental model behind piping,
> and a library of real DevOps compound commands.

---

## 1. The Thinking Framework — How to Build Any Complex Command

```
┌──────────────────────────────────────────────────────────────┐
│  5-STEP MENTAL MODEL FOR BUILDING COMPLEX COMMANDS           │
│                                                              │
│  Step 1: STATE THE GOAL IN PLAIN ENGLISH                    │
│  "I want to find the 10 largest files on this server"       │
│                                                              │
│  Step 2: BREAK IT INTO VERBS                                │
│  find files → measure size → sort → take top 10            │
│                                                              │
│  Step 3: MAP EACH VERB TO A LINUX TOOL                      │
│  find files   → find                                        │
│  measure size → du -h                                       │
│  sort         → sort -hr                                    │
│  take top 10  → head -n 10                                  │
│                                                              │
│  Step 4: CONNECT WITH PIPES |                               │
│  find / -type f | du -h | sort -hr | head -n 10            │
│                                                              │
│  Step 5: ADD ERROR HANDLING & EDGE CASES                    │
│  find / -type f -exec du -h {} + 2>/dev/null | sort -hr    │
│  | head -n 10                                               │
│                                                              │
│  KEY INSIGHT: Each tool does ONE thing.                     │
│  Pipes connect tools: output of left = input of right.      │
│  You compose simple tools into powerful pipelines.          │
└──────────────────────────────────────────────────────────────┘
```

---

## 2. Core Building Blocks — Master These First

```bash
# ── FILTERING ────────────────────────────────────────────────
grep "pattern" file          # Find lines matching pattern
grep -v "pattern"            # Exclude lines matching pattern
grep -i "pattern"            # Case insensitive
grep -r "pattern" ./dir      # Recursive search in directory
grep -E "pat1|pat2"          # Extended regex (OR)

# ── TRANSFORMATION ───────────────────────────────────────────
awk '{print $1}'             # Print first column (space delimited)
awk -F: '{print $1}'         # Print first column (: delimited)
awk '{sum += $1} END {print sum}'  # Sum a column
sed 's/old/new/g'            # Replace all occurrences in line
sed -n '5,10p'               # Print lines 5-10
cut -d',' -f1,3              # Cut CSV: columns 1 and 3
tr 'a-z' 'A-Z'              # Translate: lowercase to uppercase
tr -d '\n'                   # Delete newlines

# ── SORTING & DEDUPLICATION ───────────────────────────────────
sort                         # Alphabetical sort
sort -n                      # Numeric sort
sort -r                      # Reverse sort
sort -h                      # Human readable (10K < 1M < 1G)
sort -k2                     # Sort by column 2
sort -u                      # Sort and deduplicate
uniq                         # Remove consecutive duplicates (needs sorted input)
uniq -c                      # Count occurrences
uniq -d                      # Show only duplicates

# ── COUNTING & STATS ─────────────────────────────────────────
wc -l                        # Count lines
wc -w                        # Count words
wc -c                        # Count bytes

# ── SELECTION ────────────────────────────────────────────────
head -n 10                   # First 10 lines
tail -n 10                   # Last 10 lines
tail -f                      # Follow file (live)
tail -n +2                   # Skip first line (headers)

# ── TEXT EXTRACTION ───────────────────────────────────────────
awk '{print $NF}'            # Print last column
awk '{print $(NF-1)}'        # Print second-to-last column
grep -oP '(?<=key=)\S+'      # Extract value after "key=" (Perl regex)
sed 's/.*: //'               # Extract everything after ": "

# ── EXECUTION HELPERS ─────────────────────────────────────────
xargs                        # Convert stdin to arguments
xargs -I {}                  # Placeholder: xargs -I {} echo "file: {}"
xargs -P 4                   # Parallel execution with 4 processes
tee file.txt                 # Write to file AND pass to stdout
command1 && command2         # Run command2 only if command1 succeeds
command1 || command2         # Run command2 only if command1 fails
command1 ; command2          # Always run both
$(command)                   # Command substitution: use output as value
```

---

## 3. How Pipes Work — Visual Mental Model

```
┌──────────────────────────────────────────────────────────────┐
│  PIPE MENTAL MODEL                                           │
│                                                              │
│  find / -type f | sort | head -n 5                          │
│                                                              │
│  ┌──────────┐  stdout  ┌──────────┐  stdout  ┌──────────┐   │
│  │   find   │ ───────► │   sort   │ ───────► │   head   │   │
│  │ (source) │          │(transform│          │ (output) │   │
│  └──────────┘          └──────────┘          └──────────┘   │
│                                                              │
│  Every program has:                                         │
│  stdin  (fd 0) = input  (keyboard by default)               │
│  stdout (fd 1) = output (terminal by default)               │
│  stderr (fd 2) = errors (terminal by default)               │
│                                                              │
│  | pipes stdout of left → stdin of right                   │
│  2>/dev/null  discards errors                               │
│  2>&1         merges errors into stdout                     │
│  > file       redirects stdout to file (overwrite)          │
│  >> file      redirects stdout to file (append)             │
│  < file       reads file as stdin                           │
│  <<EOF        here document (multiline stdin)               │
└──────────────────────────────────────────────────────────────┘
```

---

## 4. Real DevOps Compound Commands — By Category

### Linux File System

```bash
# Top 10 biggest files anywhere on system
find / -type f -exec du -h {} + 2>/dev/null | sort -rh | head -n 10

# Top 10 biggest directories
du -h --max-depth=1 /var | sort -rh | head -n 10

# Find all files larger than 1GB
find / -type f -size +1G 2>/dev/null

# Find files modified in last 24 hours
find /var/log -type f -mtime -1 -name "*.log"

# Find and delete files older than 30 days
find /tmp -type f -mtime +30 -delete

# Find and delete empty directories
find /tmp -type d -empty -delete

# Find all world-writable files (security check)
find / -type f -perm -o+w 2>/dev/null | grep -v proc

# Find files with SUID bit set (privilege escalation check)
find / -type f -perm -4000 2>/dev/null

# Disk usage breakdown sorted
du -sh /var/* 2>/dev/null | sort -rh

# Find which process is using a port
ss -tlnp | grep :8080
# OR
lsof -i :8080

# Find all listening ports
ss -tlnp

# Check if file contains a pattern, count matches
grep -c "ERROR" /var/log/app.log

# Count unique IPs in Nginx access log
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -20

# Find the most frequent HTTP status codes
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# Watch log file for errors in real time
tail -f /var/log/app.log | grep --color=auto "ERROR\|WARN"

# Archive and compress logs older than 7 days
find /var/log/app -name "*.log" -mtime +7 | xargs gzip

# Find duplicate files by MD5 hash
find . -type f -exec md5sum {} + | sort | awk 'BEGIN{last=""} $1==last{print $2} {last=$1}'
```

### Process Management

```bash
# Find top 5 memory-hungry processes
ps aux --sort=-%mem | head -6

# Find top 5 CPU-hungry processes
ps aux --sort=-%cpu | head -6

# Kill all processes matching a name
pkill -f "python app.py"
# OR safer:
pgrep -f "python app.py" | xargs kill

# Find which process owns a file
fuser /var/log/app.log

# Watch processes live sorted by memory
watch -n 2 'ps aux --sort=-%mem | head -20'

# Find all zombie processes
ps aux | awk '$8 == "Z" {print $0}'

# How long a process has been running
ps -p <PID> -o pid,etime,cmd
```

### Networking

```bash
# Check if a host is reachable and measure latency
ping -c 4 google.com | tail -2

# Check if port is open (without telnet)
nc -zv 10.0.0.5 5432 2>&1
# OR
timeout 3 bash -c 'cat < /dev/null > /dev/tcp/10.0.0.5/5432' && echo "Open" || echo "Closed"

# Trace route with DNS names
traceroute -n google.com

# Check all established connections grouped by remote IP
ss -tn state established | awk 'NR>1 {print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn

# Find connections to a specific port
ss -tn | grep :443

# Download file and show progress
curl -L --progress-bar -o output.tar.gz https://example.com/file.tar.gz

# Test HTTP response code
curl -o /dev/null -s -w "%{http_code}" https://api.example.com/health

# Test response time breakdown
curl -o /dev/null -s -w "DNS: %{time_namelookup}s\nConnect: %{time_connect}s\nTTFB: %{time_starttransfer}s\nTotal: %{time_total}s\n" https://api.example.com

# Check certificate expiry
echo | openssl s_client -servername api.example.com -connect api.example.com:443 2>/dev/null \
  | openssl x509 -noout -dates

# Scan open ports on a host (without nmap)
for port in 22 80 443 3306 5432 6379 8080; do
  timeout 1 bash -c "cat < /dev/null > /dev/tcp/10.0.0.5/$port" 2>/dev/null \
    && echo "Port $port: OPEN" || echo "Port $port: closed"
done
```

### Docker

```bash
# Remove all stopped containers
docker container prune -f

# Remove all unused images (not tagged, not used)
docker image prune -a -f

# Full system cleanup (images, containers, volumes, cache)
docker system prune -a --volumes -f

# Show docker disk usage
docker system df

# Show all containers sorted by memory usage
docker stats --no-stream --format "table {{.Name}}\t{{.MemUsage}}\t{{.CPUPerc}}" \
  | sort -k2 -h

# Get IP address of a container
docker inspect mycontainer | jq -r '.[0].NetworkSettings.IPAddress'

# Get all environment variables in a container
docker inspect mycontainer | jq -r '.[0].Config.Env[]'

# Follow logs from multiple containers matching a pattern
docker ps --format '{{.Names}}' | grep "api" | xargs -I {} docker logs -f {} &

# Export container filesystem as tar
docker export mycontainer | gzip > container-backup.tar.gz

# Find all images not being used by any container
docker images | grep -v "$(docker ps -a --format '{{.Image}}' | sort -u | tr '\n' '|' | sed 's/|$//')"

# Run a quick one-off debug container in same network
docker run --rm -it --network container:myapp nicolaka/netshoot

# Pull all images in a compose file without starting
docker compose pull

# Check which container is consuming most disk
docker ps -q | xargs docker inspect --format '{{.Name}} {{.GraphDriver.Data.MergedDir}}' \
  | awk '{print $2}' | xargs du -sh 2>/dev/null | sort -rh | head -10
```

### Kubernetes

```bash
# Get all pods NOT in Running state across all namespaces
kubectl get pods -A | grep -v Running | grep -v Completed | grep -v NAME

# Get pods consuming most memory (needs metrics-server)
kubectl top pods -A --sort-by=memory | head -20

# Get all images running in cluster
kubectl get pods -A -o jsonpath='{range .items[*]}{.spec.containers[*].image}{"\n"}{end}' | sort -u

# Find all pods NOT using resource limits
kubectl get pods -A -o json | jq -r '
  .items[] |
  select(.spec.containers[].resources.limits == null) |
  "\(.metadata.namespace)/\(.metadata.name)"'

# Get events sorted by time (spot recent issues)
kubectl get events -A --sort-by='.lastTimestamp' | tail -30

# Watch for pod restarts in real time
watch -n 5 'kubectl get pods -A | awk "$4 > 0" | sort -k4 -rn'

# Get all pods on a specific node
kubectl get pods -A -o wide | grep node-1

# Restart a deployment (rolling restart without changes)
kubectl rollout restart deployment/myapp -n production

# Scale all deployments in a namespace to 0 (shutdown dev env)
kubectl get deployments -n dev -o name | xargs -I {} kubectl scale {} --replicas=0 -n dev

# Force delete all pods stuck in Terminating
kubectl get pods -A | grep Terminating | awk '{print $2 " -n " $1}' \
  | xargs -I {} bash -c 'kubectl delete pod {} --force --grace-period=0'

# Get secrets decoded (pipe to jq for single key)
kubectl get secret myapp-secret -n prod -o jsonpath='{.data.password}' | base64 -d

# Copy a secret from one namespace to another
kubectl get secret myapp-tls -n production -o yaml \
  | sed 's/namespace: production/namespace: staging/' \
  | kubectl apply -f -

# Count running pods per node
kubectl get pods -A -o wide | awk 'NR>1 {print $8}' | sort | uniq -c | sort -rn

# Check resource usage per namespace
kubectl get pods -A -o json | jq -r '
  [.items[] | {
    ns: .metadata.namespace,
    mem: (.spec.containers[].resources.requests.memory // "0")
  }] | group_by(.ns)[] | {
    namespace: .[0].ns,
    pods: length
  }' | jq -s 'sort_by(.pods) | reverse | .[]'
```

### Git

```bash
# Find the commit that introduced a bug (binary search)
git bisect start
git bisect bad HEAD
git bisect good v1.0.0
# Git checks out middle commit — test it, then:
git bisect good  # or git bisect bad
# Repeat until git identifies the culprit commit
git bisect reset

# Show all files changed in last 5 commits
git log --name-only --pretty=format:"" HEAD~5..HEAD | sort -u | grep -v '^$'

# Find commits that modified a specific file
git log --follow --oneline -- src/auth/login.py

# Find which commit deleted a file
git log --all --full-history -- "path/to/deleted_file.py"

# Show diff of a single file across branches
git diff main..feature/new-login -- src/auth.py

# Find branches containing a specific commit
git branch -a --contains abc1234

# Remove a file from ALL history (secret leak cleanup)
git filter-repo --path secrets.txt --invert-paths

# Show authors by number of commits
git log --format='%an' | sort | uniq -c | sort -rn

# Find large files in git history
git rev-list --objects --all \
  | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' \
  | awk '$1=="blob" && $3>1000000 {print $3/1024/1024 "MB " $4}' \
  | sort -rn | head -10

# Undo last commit but keep changes staged
git reset --soft HEAD~1

# Undo last commit and discard changes
git reset --hard HEAD~1

# Create a patch from last commit (share without push)
git format-patch -1 HEAD -o ./patches/
```

### Terraform

```bash
# Show all resources in state
terraform state list

# Show details of one resource
terraform state show azurerm_resource_group.main

# Move resource to new address (after rename in .tf)
terraform state mv azurerm_resource_group.old azurerm_resource_group.new

# Remove resource from state without destroying it
terraform state rm azurerm_resource_group.main

# Count resources by type
terraform state list | sed 's/\..*//' | sort | uniq -c | sort -rn

# Show what plan would change (human readable)
terraform plan -out=tfplan && terraform show -no-color tfplan

# Show only destroy actions in plan (scary check before apply)
terraform plan -out=tfplan 2>/dev/null && terraform show tfplan | grep "will be destroyed"

# Format all tf files recursively
terraform fmt -recursive .

# Find all resources with a specific tag using AWS CLI
aws resourcegroupstaggingapi get-resources \
  --tag-filters Key=environment,Values=production \
  --query 'ResourceTagMappingList[].ResourceARN' \
  --output text

# Apply only a specific resource
terraform apply -target=azurerm_virtual_machine.web01

# Refresh state from real infrastructure (drift detection)
terraform plan -refresh-only
```

### Azure CLI

```bash
# List all VMs and their power states
az vm list -d --query '[].{Name:name,State:powerState,RG:resourceGroup}' -o table

# Deallocate all VMs in a resource group (stop billing)
az vm list --resource-group rg-dev -o tsv --query '[].name' \
  | xargs -I {} az vm deallocate --resource-group rg-dev --name {}

# Find all unattached managed disks (orphaned, costing money)
az disk list --query "[?diskState=='Unattached'].{Name:name,SizeGB:diskSizeGb,RG:resourceGroup}" -o table

# Get all Key Vault secrets as environment variables
while IFS= read -r name; do
  value=$(az keyvault secret show --vault-name kv-myapp --name "$name" --query value -o tsv)
  export "$name=$value"
done < <(az keyvault secret list --vault-name kv-myapp --query '[].name' -o tsv)

# Count resources by type across subscription
az resource list --query "[].type" -o tsv | sort | uniq -c | sort -rn

# Find all resources missing environment tag
az resource list --query "[?tags.environment == null].{Name:name,Type:type,RG:resourceGroup}" -o table

# Get all public IPs currently assigned
az network public-ip list --query "[?ipAddress != null].{Name:name,IP:ipAddress,RG:resourceGroup}" -o table

# Monitor activity log for deletions in last 24h
az monitor activity-log list \
  --offset 24h \
  --query "[?operationName.localizedValue contains 'Delete'].{Time:eventTimestamp,Op:operationName.localizedValue,Caller:caller}" \
  -o table

# Bulk tag all resources in a resource group
az resource list --resource-group rg-myapp-prod -o tsv --query '[].id' \
  | xargs -I {} az resource tag --tags environment=prod team=backend --ids {}
```

---

## 5. Building Your Own Complex Commands — Step by Step

### Example 1: "Find all ERROR lines in logs from last hour, count by type"

```bash
# STEP 1: What do I want?
# Lines with ERROR from the last hour, grouped and counted by error type

# STEP 2: Break into parts:
# → find log lines from last hour
# → filter only ERROR lines
# → extract the error type
# → count + sort

# STEP 3: Build piece by piece (test each piece):

# Piece 1: get log lines from last hour
awk -v start="$(date -d '1 hour ago' '+%Y-%m-%d %H')" '$0 > start' /var/log/app.log

# Piece 2: filter ERROR lines
grep "ERROR"

# Piece 3: extract error type (assume format: "ERROR [ErrorType] message")
grep -oP 'ERROR \[\K[^\]]+'

# Piece 4: count and sort
sort | uniq -c | sort -rn

# STEP 4: Combine:
awk -v start="$(date -d '1 hour ago' '+%Y-%m-%d %H')" '$0 > start' /var/log/app.log \
  | grep "ERROR" \
  | grep -oP 'ERROR \[\K[^\]]+' \
  | sort | uniq -c | sort -rn
```

### Example 2: "Kill all docker containers using more than 1GB RAM"

```bash
# STEP 1: What do I want?
# Find containers using > 1GB RAM and kill them

# STEP 2: Break into parts:
# → get memory usage per container
# → filter ones over 1GB
# → extract container name/ID
# → run docker kill

# STEP 3: Build piece by piece:

# Piece 1: get memory stats (no-stream = snapshot not live)
docker stats --no-stream --format "{{.Name}} {{.MemUsage}}"

# Piece 2: filter over 1GiB (MiB value > 1024)
# Memory shows as "1.2GiB / 16GiB" — extract first number + unit
awk '{if ($2 ~ /GiB/ && $2+0 > 1) print $1}'

# Piece 3: kill those containers
xargs docker kill

# STEP 4: Combine (add echo first to dry-run!):
docker stats --no-stream --format "{{.Name}} {{.MemUsage}}" \
  | awk '{if ($2 ~ /GiB/ && $2+0 > 1) print $1}' \
  | xargs echo "Would kill:"    # DRY RUN FIRST

# When confident, remove echo:
docker stats --no-stream --format "{{.Name}} {{.MemUsage}}" \
  | awk '{if ($2 ~ /GiB/ && $2+0 > 1) print $1}' \
  | xargs docker kill
```

### Example 3: "Deploy only if all pods are Ready"

```bash
# STEP 1: What do I want?
# Check if deployment is fully ready, then run deploy script

# STEP 2: Break into parts:
# → get pod count for deployment
# → get ready pod count
# → compare
# → proceed or abort

# STEP 3: Build it:
DEPLOYMENT="myapp"
NAMESPACE="production"

DESIRED=$(kubectl get deployment "$DEPLOYMENT" -n "$NAMESPACE" \
  -o jsonpath='{.spec.replicas}')

READY=$(kubectl get deployment "$DEPLOYMENT" -n "$NAMESPACE" \
  -o jsonpath='{.status.readyReplicas}')

if [[ "$READY" == "$DESIRED" ]]; then
  echo "All $READY/$DESIRED pods ready. Deploying..."
  ./deploy.sh
else
  echo "Only $READY/$DESIRED pods ready. Aborting."
  exit 1
fi
```

---

## 6. Debugging Complex Commands — The Isolation Technique

```bash
# RULE: When a complex command breaks, test each piece independently

# Bad approach: stare at this not knowing which part broke:
find / -type f -name "*.log" -mtime +7 | xargs grep "FATAL" | awk '{print $5}' | sort | uniq -c | sort -rn

# Good approach: test each piece:

# Does find work?
find / -type f -name "*.log" -mtime +7 | head -5

# Does xargs grep work on that?
find / -type f -name "*.log" -mtime +7 | head -5 | xargs grep "FATAL" | head -5

# Does awk give what I expect?
echo "2025-07-14 10:00:00 FATAL OOMKilled" | awk '{print $5}'

# Once each piece works, combine them.

# USE set -x IN SCRIPTS to trace:
set -x           # Print every command before executing
# your commands
set +x           # Stop tracing

# USE echo to dry-run destructive commands:
echo "rm -rf ${DIR}"    # Print what WOULD be deleted
# Only when you're sure:
rm -rf "${DIR}"
```

---

## 7. Quick Command Composition Reference

```bash
# PATTERN: Find → Filter → Transform → Output
find [where] [what] | grep [filter] | awk/sed [transform] | sort | head

# PATTERN: Get list → Do something to each item
kubectl get pods -o name | xargs -I {} kubectl describe {}
az vm list -o tsv --query '[].name' | xargs -I {} az vm start --name {} -g rg-prod

# PATTERN: Get → Check → Act (conditional)
STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://app/health)
[[ "$STATUS" == "200" ]] && echo "Healthy" || echo "UNHEALTHY: $STATUS"

# PATTERN: Loop with delay (polling)
until kubectl get pod myapp -o jsonpath='{.status.phase}' | grep -q Running; do
  echo "Waiting..."; sleep 5
done
echo "Pod is running!"

# PATTERN: Parallel execution
cat servers.txt | xargs -P 10 -I {} ssh {} 'df -h /'
# -P 10 = run 10 ssh commands in parallel

# PATTERN: Timeout wrapper (don't block forever)
timeout 30 bash -c 'until nc -z db 5432; do sleep 2; done' \
  && echo "DB ready" || echo "Timed out waiting for DB"

# PATTERN: Retry with backoff
for i in 1 2 3 4 5; do
  curl -sf http://api/health && break
  echo "Attempt $i failed, retrying in ${i}s..."
  sleep "$i"
done

# PATTERN: Capture output and check
OUTPUT=$(some_command 2>&1)
EXIT_CODE=$?
if [[ $EXIT_CODE -ne 0 ]]; then
  echo "Command failed: $OUTPUT"
  exit 1
fi
```

---

## 8. Most Useful Compound Commands — Quick Library

```bash
# Largest running processes by memory
ps aux --sort=-%mem | awk 'NR<=6{printf "%-10s %6s %6s %s\n",$1,$3,$4,$11}'

# Real-time error count per second in a log file
tail -f /var/log/app.log | pv -l -i 1 | grep --line-buffered ERROR | wc -l

# Show all environment variables sorted alphabetically
env | sort

# Extract all unique domains from a log file
grep -oP 'https?://\K[^/]+' access.log | sort -u

# Count lines of code by language
find . -name "*.py" -not -path "*/venv/*" | xargs wc -l | sort -rn | head

# Watch disk usage of a directory every 5 seconds
watch -n 5 'du -sh /var/log/* | sort -rh | head -10'

# Check all systemd services that failed
systemctl list-units --state=failed

# Restart all failed systemd services (dangerous — know what you're doing)
systemctl list-units --state=failed --no-legend | awk '{print $1}' | xargs systemctl restart

# Quick HTTP server to share files (Python)
python3 -m http.server 8080

# Check what changed in /etc in the last hour
find /etc -newer /tmp/timestamp -type f 2>/dev/null
# (First: touch -d "1 hour ago" /tmp/timestamp)

# Diff two server configs over SSH
diff <(ssh server1 cat /etc/nginx/nginx.conf) <(ssh server2 cat /etc/nginx/nginx.conf)

# Monitor a deployment rollout and notify when done
kubectl rollout status deployment/myapp -n production \
  && curl -s -X POST "$SLACK_WEBHOOK" -d '{"text":"✅ myapp deployed!"}'

# Get all pods that have restarted more than 5 times
kubectl get pods -A -o json \
  | jq -r '.items[] | select(.status.containerStatuses[]?.restartCount > 5)
  | "\(.metadata.namespace)/\(.metadata.name): \(.status.containerStatuses[0].restartCount) restarts"'

# Check if Terraform plan has any destroys (CI safety gate)
terraform plan -out=tfplan -no-color 2>&1 | tee plan.txt
if grep -q "will be destroyed" plan.txt; then
  echo "⚠️  DESTROY detected in plan. Manual review required."
  exit 1
fi
```

---

## 9. Log Analysis — Advanced Patterns

```bash
# ── Parse structured JSON logs ────────────────────────────────

# Stream live JSON logs, show only ERROR level
tail -f /var/log/app.log | jq 'select(.level == "ERROR")'

# Count log events by level in last 1000 lines
tail -1000 /var/log/app.log | jq -r '.level' | sort | uniq -c | sort -rn

# Extract slow requests (>500ms)
cat /var/log/app.log \
  | jq -r 'select(.duration_ms > 500) | "\(.timestamp) \(.method) \(.path) \(.duration_ms)ms"'

# Find top 10 slowest endpoints
cat /var/log/app.log \
  | jq -r 'select(.duration_ms != null) | "\(.duration_ms) \(.method) \(.path)"' \
  | sort -rn | head -10

# ── Nginx / Apache log analysis ───────────────────────────────

# Top 10 most requested URLs
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# Top 10 IPs by request count
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# All 5xx errors with their URLs
awk '$9 ~ /^5/' /var/log/nginx/access.log | awk '{print $9, $7}' | sort | uniq -c | sort -rn

# Bandwidth per IP (bytes in column 10)
awk '{ip[$1] += $10} END {for (i in ip) print ip[i], i}' \
  /var/log/nginx/access.log | sort -rn | head -10

# Find log lines between two timestamps
awk '/2025-07-14 10:00/,/2025-07-14 11:00/' /var/log/app.log

# Compare error rate: last 10 min vs previous 10 min
NOW=$(date +%H:%M)
TEN=$(date -d "10 minutes ago" +%H:%M)
TWENTY=$(date -d "20 minutes ago" +%H:%M)
RECENT=$(awk -v s="$TEN" -v e="$NOW" '$2>=s && $2<=e && /ERROR/' /var/log/app.log | wc -l)
PREV=$(awk -v s="$TWENTY" -v e="$TEN" '$2>=s && $2<=e && /ERROR/' /var/log/app.log | wc -l)
echo "Recent errors: $RECENT | Previous period: $PREV"
```

---

## 10. CI/CD Pipeline Commands

```bash
# ── GitHub Actions via CLI ────────────────────────────────────

# Watch currently running workflows
watch -n 10 'gh run list --limit 10 --json status,name,createdAt \
  | jq -r ".[] | \"\(.status) | \(.name) | \(.createdAt)\""'

# Get failure reason for last failed run
gh run list --status failure --limit 1 --json databaseId \
  | jq -r '.[0].databaseId' | xargs gh run view --log-failed

# Trigger deploy and wait for completion
gh workflow run deploy.yml --field environment=staging
sleep 5
RUN_ID=$(gh run list --workflow=deploy.yml --limit 1 --json databaseId \
  | jq -r '.[0].databaseId')
gh run watch "$RUN_ID"

# ── Docker image pipeline helpers ─────────────────────────────

# Build, tag with git SHA, and push
IMAGE="ghcr.io/myorg/myapp"
TAG=$(git rev-parse --short HEAD)
docker build -t "$IMAGE:$TAG" -t "$IMAGE:latest" . \
  && docker push "$IMAGE:$TAG" && docker push "$IMAGE:latest"

# Scan and fail on CRITICAL CVEs
trivy image --exit-code 1 --severity CRITICAL myapp:latest \
  && echo "✅ No critical CVEs" || { echo "❌ CVEs found!"; exit 1; }

# Skip rebuild if image already in registry
if docker manifest inspect "ghcr.io/myorg/myapp:$(git rev-parse --short HEAD)" 2>/dev/null; then
  echo "Image exists, skipping build"
else
  docker build -t "ghcr.io/myorg/myapp:$(git rev-parse --short HEAD)" .
fi

# ── Helm deployment ───────────────────────────────────────────

# Deploy and auto-rollback on failure
helm upgrade --install myapp ./charts/myapp \
  --namespace production \
  --set image.tag="$(git rev-parse --short HEAD)" \
  --wait --timeout 5m --atomic

# Diff what would change before upgrade
helm diff upgrade myapp ./charts/myapp --set image.tag=new-tag

# List all releases sorted by last deployed
helm list -A --output json \
  | jq -r '.[] | "\(.updated) \(.namespace)/\(.name) \(.chart)"' \
  | sort -r | head -20

# Rollback with history
helm history myapp -n production | tail -5
read -rp "Roll back to revision: " rev
helm rollback myapp "$rev" -n production --wait
```

---

## 11. Security Audit Commands

```bash
# ── Linux security ────────────────────────────────────────────

# All users with UID 0
awk -F: '$3 == 0 {print $1}' /etc/passwd

# All SSH authorized_keys files
find / -name "authorized_keys" 2>/dev/null

# SUID files (privilege escalation vectors)
find / -type f -perm -4000 2>/dev/null

# Cron jobs running as root
crontab -l 2>/dev/null
cat /etc/cron* 2>/dev/null | grep -v "^#" | grep -v "^$"

# Files owned by no user
find / -nouser 2>/dev/null | grep -v proc | head -20

# World-writable files in sensitive dirs
find /etc /usr /bin /sbin -type f -perm -o+w 2>/dev/null

# ── Container security ────────────────────────────────────────

# Containers running as root
docker ps -q | xargs -I {} docker inspect {} \
  | jq -r '.[] | select(.Config.User == "" or .Config.User == "0") | .Name'

# Privileged containers
docker ps -q | xargs -I {} docker inspect {} \
  | jq -r '.[] | select(.HostConfig.Privileged == true) | .Name'

# Containers using host network
docker ps -q | xargs -I {} docker inspect {} \
  | jq -r '.[] | select(.HostConfig.NetworkMode == "host") | .Name'

# ── Kubernetes security ───────────────────────────────────────

# Privileged containers
kubectl get pods -A -o json \
  | jq -r '.items[] | select(.spec.containers[].securityContext.privileged == true) |
    "\(.metadata.namespace)/\(.metadata.name)"'

# Cluster-admin bindings
kubectl get clusterrolebindings -o json \
  | jq -r '.items[] | select(.roleRef.name == "cluster-admin") |
    "\(.metadata.name): \(.subjects[]? | "\(.kind)/\(.name)")"'

# Pods with hostPath mounts
kubectl get pods -A -o json \
  | jq -r '.items[] | select(.spec.volumes[]?.hostPath != null) |
    "\(.metadata.namespace)/\(.metadata.name)"'
```

---

## 12. System Health & Monitoring

```bash
# Quick system snapshot
echo "CPU:" && top -bn1 | grep "Cpu(s)"
echo "Memory:" && free -h
echo "Load:" && uptime
echo "Disk:" && df -h | grep -v tmpfs

# OOM killer activity
dmesg | grep -i "oom\|killed process" | tail -20

# Find inode exhaustion (no space left even with free disk)
df -i | sort -k5 -rn

# TCP connection states summary
ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn

# Top talkers by connection count
ss -tan state established | awk 'NR>1 {print $5}' \
  | cut -d: -f1 | sort | uniq -c | sort -rn | head -10

# Services with errors in last hour
journalctl -p err --since "1 hour ago" --no-pager | tail -30

# Service restart count
systemctl show nginx --property=NRestarts

# Network bytes per interface since boot
cat /proc/net/dev | awk 'NR>2 {
  printf "%s  RX: %.1fMB  TX: %.1fMB\n", $1, $2/1048576, $10/1048576}'

# Watch service live
watch -n 2 'systemctl status nginx --no-pager | head -20'
```

---

## 13. JSON & YAML Processing

```bash
# ── jq ────────────────────────────────────────────────────────

# Filter array by condition
cat data.json | jq '.users[] | select(.age > 30) | .name'

# Reshape JSON
cat data.json | jq '{name: .user.name, email: .user.email}'

# Array to CSV
cat data.json | jq -r '.[] | [.name, .email, .role] | @csv'

# Merge two JSON objects
jq -s '.[0] * .[1]' base.json override.json

# JSON keys to shell exports
cat secrets.json | jq -r 'to_entries[] | "export \(.key)=\(.value)"'

# ── yq ────────────────────────────────────────────────────────

# Update field in place
yq -i '.spec.replicas = 3' deployment.yaml

# Update all container images
yq -i '.spec.template.spec.containers[].image = "myapp:v2"' deployment.yaml

# Convert YAML to JSON
yq -o=json deployment.yaml

# Merge two YAML files
yq '. * load("override.yaml")' base.yaml

# Validate YAML syntax
yq . deployment.yaml > /dev/null && echo "Valid" || echo "Invalid YAML"

# ── K8s manifest manipulation ─────────────────────────────────

# All images across all manifests in a directory
grep -r "image:" ./k8s/ | grep -v "#" | awk '{print $3}' | sort -u

# Replace image tag in all manifest files
find ./k8s -name "*.yaml" -exec sed -i "s|myapp:.*|myapp:${NEW_TAG}|g" {} +

# Dry-run apply: list all resources that would be created
kubectl apply -f ./k8s/ --dry-run=client -o json \
  | jq -r '.items[] | "\(.kind)/\(.metadata.name)"'
```

---

## 14. SSH & Remote Execution

```bash
# Sequential execution across servers
while IFS= read -r host; do
  echo "=== $host ===" && ssh -o ConnectTimeout=5 "$host" 'df -h / && free -h'
done < servers.txt

# Parallel execution (10 at a time)
cat servers.txt | xargs -P 10 -I {} ssh -o ConnectTimeout=5 {} 'hostname && uptime'

# Copy file to many servers in parallel
cat servers.txt | xargs -P 10 -I {} scp ./app.conf {}:/etc/myapp/app.conf

# Push and execute script on all servers
while IFS= read -r host; do
  scp deploy.sh "$host:/tmp/deploy.sh"
  ssh "$host" "bash /tmp/deploy.sh && rm /tmp/deploy.sh"
done < servers.txt

# Collect per-server output into separate log files
cat servers.txt | xargs -P 5 -I {} sh -c 'ssh {} "df -h" > logs/{}.log 2>&1'

# Forward remote DB port to local machine
ssh -L 5433:db-server:5432 bastion-host -N &
# Connect: psql -h localhost -p 5433 -U postgres

# rsync with dry-run first
rsync -avz --dry-run --exclude='*.log' ./app/ user@server:/opt/app/
rsync -avz --delete ./dist/ user@server:/var/www/html/
```

---

## 15. Database Operations (CLI)

```bash
# ── PostgreSQL ────────────────────────────────────────────────

# Dump and restore
pg_dump -h localhost -U postgres mydb | gzip > backup_$(date +%Y%m%d).sql.gz
gunzip -c backup_20250714.sql.gz | psql -h localhost -U postgres mydb

# Biggest tables
psql -h localhost -U postgres mydb -c "
  SELECT relname, pg_size_pretty(pg_total_relation_size(relid)) AS size
  FROM pg_catalog.pg_statio_user_tables
  ORDER BY pg_total_relation_size(relid) DESC LIMIT 10;"

# Long-running queries (>5 min)
psql -h localhost -U postgres -c "
  SELECT pid, now() - query_start AS duration, query, state
  FROM pg_stat_activity
  WHERE state != 'idle'
    AND now() - query_start > interval '5 minutes'
  ORDER BY duration DESC;"

# Kill query (cancel = graceful, terminate = force)
psql -h localhost -U postgres -c "SELECT pg_cancel_backend(<pid>);"
psql -h localhost -U postgres -c "SELECT pg_terminate_backend(<pid>);"

# Table bloat (dead rows)
psql -h localhost -U postgres mydb -c "
  SELECT relname, n_live_tup, n_dead_tup,
    round(n_dead_tup * 100.0 / (n_live_tup + n_dead_tup), 2) AS dead_pct
  FROM pg_stat_user_tables WHERE n_live_tup > 0
  ORDER BY dead_pct DESC LIMIT 10;"

# ── Redis ─────────────────────────────────────────────────────

# Memory and key count
redis-cli INFO memory | grep used_memory_human
redis-cli DBSIZE

# Safe key scan (won't block like KEYS *)
redis-cli --scan --pattern "session:*" | head -20
redis-cli --scan --pattern "cache:*" | wc -l

# Live command monitor
redis-cli MONITOR | grep -v PING | head -50

# Batch delete keys by pattern (100 at a time)
redis-cli --scan --pattern "cache:temp:*" | xargs -L 100 redis-cli DEL

# Replication status
redis-cli INFO replication | grep -E "role|connected_slaves|master_link_status"
```

---

## 16. One-liner Scripts for Common DevOps Tasks

```bash
# Check SSL expiry for multiple domains
for domain in api.example.com app.example.com; do
  expiry=$(echo | openssl s_client -servername "$domain" \
    -connect "$domain:443" 2>/dev/null \
    | openssl x509 -noout -enddate 2>/dev/null | cut -d= -f2)
  echo "$domain → $expiry"
done

# Ping sweep — show unreachable hosts
for host in 10.0.1.{1..10}; do
  ping -c 1 -W 1 "$host" &>/dev/null \
    && echo "✅ $host" || echo "❌ $host UNREACHABLE"
done

# Wait for pod ready then notify Slack
kubectl wait --for=condition=ready pod \
  -l app=myapp -n production --timeout=120s \
  && curl -s -X POST "$SLACK_WEBHOOK" \
     -d '{"text":"✅ myapp pods ready!"}' \
  || curl -s -X POST "$SLACK_WEBHOOK" \
     -d '{"text":"❌ myapp pod readiness timeout!"}'

# Check all deployment health in a namespace
kubectl get deployments -n production -o json \
  | jq -r '.items[] |
    if .status.readyReplicas == .spec.replicas
    then "✅ \(.metadata.name) (\(.status.readyReplicas)/\(.spec.replicas))"
    else "❌ \(.metadata.name) (\(.status.readyReplicas // 0)/\(.spec.replicas))"
    end'

# Rolling restart all deployments in a namespace
kubectl get deployments -n production -o name \
  | xargs -I {} kubectl rollout restart {} -n production

# Delete all evicted pods across all namespaces
kubectl get pods -A | grep Evicted \
  | awk '{print $2 " -n " $1}' \
  | xargs -I {} bash -c 'kubectl delete pod {}'

# Delete all completed jobs
kubectl delete jobs -A --field-selector status.successful=1

# Find orphaned Azure disks
az disk list \
  --query "[?diskState=='Unattached'].{Name:name,Size:diskSizeGb,SKU:sku.name,RG:resourceGroup}" \
  -o table

# Deallocate all stopped (not yet deallocated) Azure VMs
az vm list -d \
  --query "[?powerState=='VM stopped'].{name:name,rg:resourceGroup}" -o tsv \
  | while IFS=$'\t' read -r name rg; do
      echo "Deallocating $name in $rg"
      az vm deallocate --name "$name" --resource-group "$rg" --no-wait
    done

# Bulk tag Azure resources missing environment tag
az resource list \
  --query "[?tags.environment == null].id" -o tsv \
  | xargs -I {} az resource tag --tags environment=prod --ids {}
```

---

## 17. awk Deep Dive — The DevOps Swiss Army Knife

```bash
# ── awk fundamentals ──────────────────────────────────────────
# awk processes line by line. Each line is split into fields $1 $2...
# $0 = entire line, $NF = last field, $(NF-1) = second to last

# Print specific columns from df output
df -h | awk 'NR>1 {print $1, $5}'    # filesystem and usage%

# Filter lines where 5th column (usage%) > 80%
df -h | awk 'NR>1 {gsub(/%/,"",$5); if($5+0 > 80) print $1, $5"%"}'

# Sum a column (e.g., total memory from ps)
ps aux | awk 'NR>1 {sum += $6} END {print sum/1024 "MB total RSS"}'

# Average response time from logs
awk '{sum += $NF; count++} END {print "Avg:", sum/count "ms"}' response_times.log

# Print lines matching pattern with context (like grep -A but more control)
awk '/ERROR/{print NR": "$0; for(i=1;i<=3;i++) {getline; print NR": "$0}}' app.log

# Conditional field replacement
awk '{if($3 == "FAIL") $3="❌FAIL"; else $3="✅PASS"; print}' results.txt

# Multiline records (records separated by blank lines)
awk 'BEGIN{RS=""; FS="\n"} /ERROR/{print $1}' app.log

# Print unique values of column 3, counting occurrences
awk '{count[$3]++} END {for (k in count) print count[k], k}' app.log | sort -rn

# Reformat CSV: swap columns 1 and 2, add prefix to column 3
awk -F',' '{print $2","$1",prod-"$3}' data.csv

# Calculate 95th percentile of response times
awk '{times[NR]=$1} END {
  n=asort(times);
  p95=int(n*0.95);
  print "p95:", times[p95] "ms"
}' response_times.log

# Join two files on a common field (like SQL JOIN)
# file1: id name   file2: id score
awk 'NR==FNR{a[$1]=$2; next} $1 in a {print $0, a[$1]}' file2 file1

# ── Real DevOps awk patterns ──────────────────────────────────

# Parse Terraform plan output: count adds, changes, destroys
terraform plan -no-color 2>/dev/null | \
  awk '/Plan:/{print "Adds:", $2, "Changes:", $4, "Destroys:", $6}'

# Extract pod name and restart count, filter >5 restarts
kubectl get pods -A | awk 'NR>1 && $5+0 > 5 {print $2, "restarts:", $5}'

# Parse docker stats and alert on high memory
docker stats --no-stream | awk 'NR>1 {
  split($4,a,"/"); gsub(/[^0-9.]/,"",a[1]);
  if(a[1]+0 > 1000) print "⚠️ HIGH MEM:", $2, $4
}'

# Extract key=value pairs from log lines into tab-separated
awk '{
  for(i=1;i<=NF;i++) {
    if(match($i,/([^=]+)=([^,]+)/,arr)) printf arr[1]"\t"arr[2]"\t"
  }
  print ""
}' structured.log
```

---

## 18. sed Deep Dive

```bash
# ── sed fundamentals ──────────────────────────────────────────

# Replace first occurrence per line
sed 's/old/new/' file.txt

# Replace ALL occurrences per line (global flag)
sed 's/old/new/g' file.txt

# Replace only on lines matching a pattern
sed '/ERROR/s/localhost/prod-db.internal/g' app.conf

# Delete lines matching pattern
sed '/^#/d' config.txt          # Delete comment lines
sed '/^$/d' config.txt          # Delete empty lines
sed '/ERROR/d' app.log          # Delete ERROR lines

# Print only matching lines (like grep)
sed -n '/ERROR/p' app.log

# Print line range
sed -n '10,20p' file.txt

# In-place edit (modifies file directly)
sed -i 's/staging/production/g' app.conf

# In-place with backup
sed -i.bak 's/staging/production/g' app.conf
# Creates app.conf.bak with original

# Multi-command sed
sed -e 's/foo/bar/g' -e 's/baz/qux/g' file.txt
# OR use semicolons:
sed 's/foo/bar/g; s/baz/qux/g' file.txt

# Insert line before matching pattern
sed '/SERVER_URL/i # Auto-generated — do not edit manually' config.txt

# Insert line after matching pattern
sed '/\[database\]/a url = postgresql://db:5432/mydb' config.ini

# Delete lines between two patterns (inclusive)
sed '/^---BEGIN CERT---/,/^---END CERT---/d' file.txt

# ── Real DevOps sed patterns ──────────────────────────────────

# Update image tag in all Kubernetes manifests
find ./k8s -name "*.yaml" -exec sed -i "s|image: myapp:.*|image: myapp:${TAG}|g" {} +

# Remove ANSI color codes from log output
sed 's/\x1B\[[0-9;]*[mGKHF]//g' colored_output.log

# Extract value from config file (key = value format)
sed -n 's/^database_url\s*=\s*//p' config.ini

# Replace placeholder in template file
sed "s|{{APP_NAME}}|myapp|g; s|{{NAMESPACE}}|production|g" deployment.template.yaml

# Comment out a line matching a pattern
sed -i '/^ENABLE_DEBUG/s/^/# /' app.conf

# Uncomment a line
sed -i '/^# ENABLE_DEBUG/s/^# //' app.conf

# Convert Windows line endings to Unix
sed -i 's/\r//' file.txt

# Add a line number prefix to every line
sed = file.txt | sed 'N;s/\n/\t/'
```

---

## 19. xargs Mastery

```bash
# ── xargs fundamentals ───────────────────────────────────────

# Basic: take stdin lines as arguments
echo "file1 file2 file3" | xargs rm

# One argument at a time (-I {} placeholder)
cat servers.txt | xargs -I {} ssh {} "df -h /"

# Parallel execution (-P = number of parallel processes)
cat urls.txt | xargs -P 10 -I {} curl -s -o /dev/null -w "%{http_code} {}\n" {}

# Limit args per command execution (-n)
cat files.txt | xargs -n 5 rm    # rm 5 files at a time

# Handle filenames with spaces (-d newline delimiter)
find . -name "*.log" -print0 | xargs -0 rm

# Dry run: echo instead of execute
cat servers.txt | xargs -I {} echo "Would SSH to: {}"

# Combine with grep result
grep -r "TODO" ./src --include="*.py" -l | xargs wc -l

# Pass multiple args from same line
cat << 'EOF' | xargs -n2 sh -c 'echo "Host: $1 Port: $2"' _
web01 80
web02 443
db01 5432
EOF

# ── Real DevOps xargs patterns ────────────────────────────────

# Restart all unhealthy containers
docker ps --filter health=unhealthy --format "{{.Names}}" \
  | xargs -I {} docker restart {}

# Pull all images listed in a file
cat required-images.txt | xargs -P 5 -I {} docker pull {}

# Delete all images for a specific registry
docker images --format "{{.Repository}}:{{.Tag}}" \
  | grep "old-registry.example.com" \
  | xargs docker rmi

# Run kubectl on multiple contexts
cat contexts.txt | xargs -I {} kubectl --context={} get nodes

# Apply multiple terraform workspaces
echo "dev staging prod" | tr ' ' '\n' | xargs -I {} sh -c \
  'terraform workspace select {} && terraform apply -auto-approve'

# Parallel SSH health check with results
cat servers.txt | xargs -P 20 -I {} sh -c \
  'result=$(ssh -o ConnectTimeout=3 {} "systemctl is-active nginx" 2>/dev/null)
   echo "{}: $result"'
```

---

## 20. find Mastery

```bash
# ── find fundamentals ────────────────────────────────────────

# By type
find /var/log -type f           # Files only
find /var/log -type d           # Directories only
find /var/log -type l           # Symlinks only

# By name (glob patterns)
find . -name "*.log"            # Case sensitive
find . -iname "*.Log"           # Case insensitive
find . -name "app*" -not -name "*.bak"  # Exclude pattern

# By time
find /tmp -mtime +7             # Modified more than 7 days ago
find /tmp -mtime -1             # Modified less than 1 day ago
find /tmp -newer /etc/passwd    # Modified more recently than passwd
find /var/log -mmin -60         # Modified in last 60 minutes

# By size
find / -size +100M              # Larger than 100MB
find / -size +1G -size -10G    # Between 1GB and 10GB
find . -empty                   # Empty files and directories

# By permissions
find / -perm -4000              # SUID bit set
find / -perm -o+w               # World-writable
find . -perm 600                # Exactly 600

# By owner
find /home -user alice          # Owned by alice
find /tmp -group www-data       # Owned by group www-data

# ── Execution with find ───────────────────────────────────────

# Execute command on each result
find . -name "*.py" -exec python -m py_compile {} \;
# \; = run once per file (slower)
# {} + = batch as many files as possible per command (faster)
find . -name "*.py" -exec python -m py_compile {} +

# Delete (built-in, faster than -exec rm)
find /tmp -name "*.tmp" -delete

# Print with null terminator (safe for filenames with spaces)
find . -name "*.log" -print0 | xargs -0 gzip

# ── Real DevOps find patterns ─────────────────────────────────

# Find recently deployed files (changed in last deployment window)
find /opt/app -newer /tmp/deploy_start_marker -type f

# Find configs that haven't been touched in 90 days (stale?)
find /etc -name "*.conf" -mtime +90 -type f

# Find all Python files excluding virtual envs and cache
find . -name "*.py" \
  -not -path "*/venv/*" \
  -not -path "*/.venv/*" \
  -not -path "*/__pycache__/*" \
  -not -path "*/node_modules/*"

# Find all symlinks pointing to non-existent targets (broken symlinks)
find / -type l ! -e 2>/dev/null | head -20

# Find files containing a pattern (alternative to grep -r)
find . -name "*.yaml" -exec grep -l "image: myapp" {} +

# Find executable scripts in wrong location
find /home -name "*.sh" -perm /111 -not -path "*/bin/*"

# Find files by inode number (useful when name has special chars)
find / -inum 12345678 2>/dev/null
```

---

## 21. curl Mastery for API Testing & Automation

```bash
# ── curl fundamentals ────────────────────────────────────────

# Basic GET
curl https://api.example.com/users

# With headers
curl -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     https://api.example.com/users

# POST with JSON body
curl -X POST \
     -H "Content-Type: application/json" \
     -d '{"name":"alice","role":"admin"}' \
     https://api.example.com/users

# POST with JSON from file
curl -X POST \
     -H "Content-Type: application/json" \
     -d @payload.json \
     https://api.example.com/users

# PUT / PATCH / DELETE
curl -X PUT -d @updated.json https://api.example.com/users/42
curl -X PATCH -d '{"role":"viewer"}' https://api.example.com/users/42
curl -X DELETE https://api.example.com/users/42

# Save response to file
curl -o response.json https://api.example.com/data
curl -O https://example.com/archive.tar.gz   # Use remote filename

# Follow redirects
curl -L https://example.com/redirect

# ── Response inspection ───────────────────────────────────────

# Show only status code
curl -o /dev/null -s -w "%{http_code}" https://api.example.com/health

# Show response headers only
curl -I https://api.example.com/health

# Show both headers and body
curl -i https://api.example.com/health

# Verbose (full request + response headers)
curl -v https://api.example.com/health 2>&1 | grep -E "^[<>*]"

# Timing breakdown
curl -o /dev/null -s -w \
  "DNS: %{time_namelookup}s | TCP: %{time_connect}s | TLS: %{time_appconnect}s | TTFB: %{time_starttransfer}s | Total: %{time_total}s\n" \
  https://api.example.com

# ── Authentication patterns ───────────────────────────────────

# Basic auth
curl -u username:password https://api.example.com/secure

# Bearer token
curl -H "Authorization: Bearer $(cat /tmp/token)" https://api.example.com

# Azure AD token (OIDC)
TOKEN=$(az account get-access-token --query accessToken -o tsv)
curl -H "Authorization: Bearer $TOKEN" https://management.azure.com/subscriptions/...

# API key in header
curl -H "X-API-Key: $API_KEY" https://api.example.com

# ── Real DevOps curl patterns ─────────────────────────────────

# Health check loop with retries
for i in $(seq 1 10); do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://app/health)
  [[ "$STATUS" == "200" ]] && { echo "✅ Healthy"; break; }
  echo "Attempt $i: got $STATUS, retrying in 5s..."
  sleep 5
done

# Send Slack notification
curl -s -X POST "$SLACK_WEBHOOK_URL" \
  -H "Content-Type: application/json" \
  -d "{\"text\": \"🚀 Deployment of \`${APP_NAME}:${IMAGE_TAG}\` to \`${ENV}\` completed\"}"

# Trigger GitHub Actions workflow
curl -s -X POST \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github.v3+json" \
  "https://api.github.com/repos/${ORG}/${REPO}/actions/workflows/deploy.yml/dispatches" \
  -d "{\"ref\":\"main\",\"inputs\":{\"environment\":\"staging\"}}"

# Test all endpoints from a list and report failures
while IFS= read -r url; do
  code=$(curl -s -o /dev/null -w "%{http_code}" --max-time 5 "$url")
  [[ "$code" =~ ^2 ]] && echo "✅ $url ($code)" || echo "❌ $url ($code)"
done < endpoints.txt

# Download with automatic retry
curl --retry 5 --retry-delay 2 --retry-max-time 60 \
  -L -o file.tar.gz https://releases.example.com/app.tar.gz
```

---

## 22. Environment & Variable Tricks

```bash
# ── Variable manipulation ─────────────────────────────────────

VAR="hello-world-foo"

echo "${VAR^^}"            # HELLO-WORLD-FOO   (uppercase)
echo "${VAR,,}"            # hello-world-foo   (lowercase)
echo "${VAR^}"             # Hello-world-foo   (capitalize first)
echo "${#VAR}"             # 15                (length)
echo "${VAR/foo/bar}"      # hello-world-bar   (replace first)
echo "${VAR//l/L}"         # heLLo-worLd-foo   (replace all)
echo "${VAR#hello-}"       # world-foo         (remove prefix)
echo "${VAR##*-}"          # foo               (remove longest prefix)
echo "${VAR%-foo}"         # hello-world       (remove suffix)
echo "${VAR%%-*}"          # hello             (remove longest suffix)
echo "${VAR:6:5}"          # world             (substring: offset:length)

# Default values
echo "${UNSET_VAR:-default}"          # Use default if unset/empty
echo "${UNSET_VAR:=default}"          # Set AND use default if unset
echo "${UNSET_VAR:?Error: not set}"   # Error and exit if unset

# ── Array tricks ─────────────────────────────────────────────

SERVERS=("web01" "web02" "db01" "redis01")

echo "${SERVERS[@]}"          # All elements
echo "${#SERVERS[@]}"         # Count: 4
echo "${SERVERS[0]}"          # First: web01
echo "${SERVERS[-1]}"         # Last: redis01
echo "${SERVERS[@]:1:2}"      # Slice: web02 db01

# Iterate
for srv in "${SERVERS[@]}"; do echo "$srv"; done

# Filter (bash 4.0+)
echo "${SERVERS[@]/#db/DATABASE-}"    # Prefix match: DATABASE-01

# Append
SERVERS+=("cache01")

# ── Environment file parsing ──────────────────────────────────

# Export all variables from .env file safely
set -a; source .env; set +a

# Export from .env without sourcing (safer for CI)
while IFS='=' read -r key value; do
  [[ "$key" =~ ^#|^$ ]] && continue   # Skip comments and blanks
  export "$key=$value"
done < .env

# Check required env vars are set before script runs
REQUIRED=(DB_URL API_KEY SECRET_KEY)
for var in "${REQUIRED[@]}"; do
  [[ -z "${!var}" ]] && { echo "ERROR: $var is not set"; exit 1; }
done
echo "All required variables present ✅"

# Print all env vars matching a prefix
env | grep "^AWS_\|^AZURE_\|^GH_" | sort
```
