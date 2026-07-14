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
