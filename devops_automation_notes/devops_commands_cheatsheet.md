# 🛠️ Most Used Commands for a DevOps Engineer

> Quick reference — tools you'll run daily on the job.

---

## 1. Linux / Bash

```bash
# System info
uname -a                        # kernel + OS info
uptime                          # how long running, load avg
whoami && id                    # current user + groups
hostname -I                     # server IP address

# Process management
ps aux | grep nginx             # find process
top / htop                      # live resource usage
kill -9 <PID>                   # force kill process
systemctl status nginx          # service status
systemctl restart nginx         # restart service
systemctl enable nginx          # start on boot
journalctl -u nginx -f          # follow service logs
journalctl -u nginx --since "1 hour ago"

# Disk & filesystem
df -h                           # disk usage (human readable)
du -sh /var/log/*               # folder sizes
lsblk                           # list block devices
mount | grep sda                # what's mounted

# Network
curl -I https://example.com     # HTTP headers only
curl -sf http://localhost:8080/health  # health check
ss -tlnp                        # open TCP ports + which process
netstat -tlnp                   # same (older tool)
ping -c 4 8.8.8.8               # ping test
traceroute google.com           # trace route
dig google.com                  # DNS lookup
nslookup google.com             # DNS lookup (simpler)

# File operations
tail -f /var/log/nginx/access.log    # follow log in real-time
tail -n 100 /var/log/app.log         # last 100 lines
grep -r "ERROR" /var/log/app/        # search recursively
grep -i "timeout" app.log | wc -l   # count matches
cat file.txt | grep "error" | awk '{print $5}'  # pipe chain
sed -i 's/old/new/g' config.txt     # find & replace in file
find /var/log -name "*.log" -mtime +7  # files older than 7 days
find / -name "nginx.conf" 2>/dev/null  # find config file

# Permissions
chmod +x deploy.sh              # make executable
chmod 600 ~/.ssh/id_rsa         # private key permissions
chown -R ubuntu:ubuntu /app     # change owner recursively
ls -la                          # list with permissions

# SSH
ssh -i ~/.ssh/key.pem ubuntu@10.0.1.5        # SSH with key
ssh-keygen -t ed25519 -C "email@example.com" # generate key pair
scp file.txt ubuntu@10.0.1.5:/tmp/          # copy file to server
rsync -avz ./dist/ ubuntu@10.0.1.5:/app/    # sync directory

# Archives
tar -czf backup.tar.gz /app/              # compress directory
tar -xzf backup.tar.gz -C /restore/      # extract
zip -r archive.zip ./folder/             # zip
unzip archive.zip -d /target/            # unzip

# Text processing
jq '.[].name' data.json                 # parse JSON
jq '.[] | select(.status=="running")' data.json  # filter JSON
awk '{print $1}' access.log             # print first column
awk -F',' '{print $2}' data.csv         # CSV second column
cut -d':' -f1 /etc/passwd               # extract usernames
sort | uniq -c | sort -rn               # count unique, sort by freq
```

---

## 2. Git

```bash
# Setup
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

# Daily workflow
git status                      # what changed?
git diff                        # see unstaged changes
git diff --staged               # see staged changes
git add .                       # stage all changes
git add -p                      # interactively stage hunks
git commit -m "feat: add health endpoint"
git push origin feature/my-branch

# Branching
git checkout -b feature/new-thing   # create + switch
git checkout main                   # switch branch
git branch -d feature/done         # delete local branch
git branch -a                      # list all branches

# Merging & rebasing
git merge feature/new-thing        # merge into current
git rebase main                    # rebase onto main
git rebase -i HEAD~3               # interactive rebase (squash)

# History
git log --oneline -20              # compact log
git log --graph --oneline          # visual branch graph
git blame file.py                  # who wrote each line
git show abc1234                   # show specific commit

# Undoing
git restore file.py                # discard unstaged change
git restore --staged file.py       # unstage file
git revert abc1234                 # undo commit (safe, creates new commit)
git reset --hard HEAD~1            # delete last commit (DANGER in shared branches)
git stash                          # save work temporarily
git stash pop                      # restore stashed work

# Remote
git remote -v                      # list remotes
git fetch --all                    # fetch all remote changes
git pull --rebase origin main      # pull + rebase
git push origin --delete feature/old   # delete remote branch
```

---

## 3. Docker

```bash
# Images
docker build -t myapp:v1.0 .              # build from Dockerfile
docker build --no-cache -t myapp:v1.0 .  # force fresh build
docker images                             # list local images
docker rmi myapp:old                      # remove image
docker pull nginx:1.25                    # pull from registry
docker push myacr.azurecr.io/myapp:v1.0  # push to registry
docker tag myapp:latest myapp:v1.0        # re-tag image
docker image prune -a                     # remove all unused images

# Containers
docker run -d --name myapp -p 8080:80 nginx:1.25  # run detached
docker run -it ubuntu:22.04 /bin/bash              # interactive shell
docker run --rm -e ENV=prod myapp:v1.0             # auto-remove on exit
docker ps                                          # running containers
docker ps -a                                       # all containers
docker stop myapp                                  # graceful stop
docker kill myapp                                  # force stop
docker rm myapp                                    # remove container
docker rm $(docker ps -aq)                         # remove ALL containers

# Debugging
docker logs myapp                         # container logs
docker logs -f myapp                      # follow logs
docker logs --tail 100 myapp             # last 100 lines
docker exec -it myapp bash               # shell into running container
docker exec myapp cat /etc/nginx/nginx.conf  # run command in container
docker inspect myapp                     # full container config JSON
docker stats                             # live CPU/memory per container
docker top myapp                         # processes inside container

# Volumes & Networks
docker volume ls                         # list volumes
docker volume prune                      # delete unused volumes
docker network ls                        # list networks
docker network inspect bridge            # inspect network

# Registry auth
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789.dkr.ecr.us-east-1.amazonaws.com

az acr login --name myacr                # Azure Container Registry login

# System cleanup
docker system df                         # disk usage
docker system prune -af                  # remove EVERYTHING unused
```

---

## 4. Kubernetes (kubectl)

```bash
# Context / cluster
kubectl config get-contexts              # list clusters
kubectl config use-context prod-cluster  # switch cluster
kubectl config current-context          # current cluster
kubectl cluster-info                    # cluster endpoints

# Pods
kubectl get pods                        # current namespace
kubectl get pods -A                     # ALL namespaces
kubectl get pods -n production          # specific namespace
kubectl get pods -l app=myapp           # filter by label
kubectl get pods -o wide                # show node placement
kubectl describe pod myapp-abc123       # full details + events
kubectl logs myapp-abc123               # container logs
kubectl logs -f myapp-abc123            # follow logs
kubectl logs myapp-abc123 --previous    # logs from crashed container
kubectl exec -it myapp-abc123 -- bash   # shell into pod
kubectl delete pod myapp-abc123         # delete (Deployment recreates it)

# Deployments
kubectl get deployments                 # list deployments
kubectl describe deployment myapp       # full details
kubectl rollout status deployment/myapp # watch rollout progress
kubectl rollout history deployment/myapp  # revision history
kubectl rollout undo deployment/myapp   # rollback to previous
kubectl rollout undo deployment/myapp --to-revision=3  # specific version
kubectl scale deployment myapp --replicas=5  # manual scale
kubectl set image deployment/myapp app=myapp:v2.1.0  # update image

# Services & Ingress
kubectl get svc                         # services
kubectl get ingress                     # ingress rules
kubectl port-forward svc/myapp 8080:80  # local port forward (debugging)

# ConfigMaps & Secrets
kubectl get configmaps
kubectl get secrets
kubectl describe secret myapp-secret
kubectl create secret generic db-pass --from-literal=password=sup3rs3cr3t
kubectl get secret myapp-secret -o jsonpath='{.data.password}' | base64 -d

# Namespaces
kubectl get namespaces
kubectl create namespace staging
kubectl get all -n staging              # everything in namespace

# Nodes
kubectl get nodes                       # node status
kubectl get nodes -o wide               # more info
kubectl describe node node-1            # CPU/mem pressure, conditions
kubectl top nodes                       # live CPU/memory per node
kubectl top pods                        # live CPU/memory per pod

# Apply / Delete
kubectl apply -f deployment.yaml        # create/update resource
kubectl apply -f ./k8s/                 # apply whole directory
kubectl delete -f deployment.yaml       # delete from manifest
kubectl delete pod -l app=myapp         # delete by label

# Debugging
kubectl get events --sort-by='.lastTimestamp'  # recent events
kubectl get events -n production --field-selector type=Warning
kubectl auth can-i create pods          # check permissions
kubectl run debug --rm -it --image=busybox -- sh  # temp debug pod
```

---

## 5. Helm

```bash
# Repo management
helm repo add stable https://charts.helm.sh/stable
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update                        # refresh repo cache
helm repo list                          # list added repos
helm search repo nginx                  # search charts

# Install / Upgrade
helm install myapp ./charts/myapp                       # first install
helm install myapp ./charts/myapp -f values-prod.yaml  # with values file
helm upgrade myapp ./charts/myapp                       # upgrade
helm upgrade --install myapp ./charts/myapp             # install or upgrade
helm upgrade myapp ./charts/myapp --set image.tag=v2.1.0  # override values
helm upgrade myapp ./charts/myapp --atomic              # rollback auto if fails
helm upgrade myapp ./charts/myapp --dry-run             # preview changes

# Inspect
helm list                               # installed releases
helm list -n production                 # specific namespace
helm history myapp                      # revision history
helm get values myapp                   # deployed values
helm get manifest myapp                 # generated K8s YAML

# Rollback / Delete
helm rollback myapp 2                   # rollback to revision 2
helm rollback myapp 0                   # rollback to previous
helm uninstall myapp                    # delete release

# Lint / Template
helm lint ./charts/myapp                # validate chart
helm template myapp ./charts/myapp      # render YAML locally (no deploy)
helm template myapp ./charts/myapp -f values-prod.yaml | kubectl apply -f -
```

---

## 6. Terraform

```bash
# Core workflow
terraform init                          # download providers + setup backend
terraform init -upgrade                 # upgrade providers
terraform fmt                           # auto-format .tf files
terraform fmt -check                    # check formatting (for CI)
terraform validate                      # syntax check (no cloud calls)
terraform plan                          # preview changes
terraform plan -out=tfplan              # save plan to file
terraform plan -var-file=prod.tfvars    # use variable file
terraform apply                         # apply (prompts confirmation)
terraform apply -auto-approve           # apply without prompt (CI only)
terraform apply tfplan                  # apply saved plan
terraform destroy                       # destroy all resources (CAREFUL!)
terraform destroy -target=aws_instance.web  # destroy only one resource

# State management
terraform state list                    # list all managed resources
terraform state show aws_instance.web   # show resource state
terraform state rm aws_instance.web     # remove from state (doesn't delete cloud resource)
terraform state mv old_addr new_addr    # rename resource in state
terraform import aws_instance.web i-abc123  # import existing resource

# Workspace
terraform workspace list
terraform workspace new staging
terraform workspace select prod

# Debugging
terraform plan -detailed-exitcode       # exit 0=no changes, 2=changes, 1=error
terraform graph | dot -Tpng > graph.png # visualize dependency graph
TF_LOG=DEBUG terraform apply            # verbose logging
terraform output                        # show output values
terraform output -json                  # machine-readable outputs
```

---

## 7. AWS CLI

```bash
# Auth / Config
aws configure                           # set access key + region
aws sts get-caller-identity             # who am I?
aws configure list                      # current config

# EC2
aws ec2 describe-instances --output table
aws ec2 describe-instances --filters "Name=tag:Environment,Values=dev"
aws ec2 start-instances --instance-ids i-abc123
aws ec2 stop-instances --instance-ids i-abc123
aws ec2 describe-instance-status --instance-ids i-abc123

# S3
aws s3 ls                               # list buckets
aws s3 ls s3://my-bucket/              # list objects
aws s3 cp file.txt s3://my-bucket/     # upload file
aws s3 sync ./dist/ s3://my-bucket/    # sync directory
aws s3 rm s3://my-bucket/file.txt      # delete object
aws s3 presign s3://my-bucket/file.txt --expires-in 3600  # presigned URL

# EKS
aws eks list-clusters
aws eks update-kubeconfig --name my-cluster --region us-east-1
aws eks describe-cluster --name my-cluster

# ECR
aws ecr get-login-password --region us-east-1 | docker login ...
aws ecr describe-repositories
aws ecr list-images --repository-name myapp

# CloudWatch Logs
aws logs describe-log-groups
aws logs tail /aws/lambda/my-function --follow
aws logs filter-log-events --log-group-name /app/prod --filter-pattern "ERROR"

# Secrets Manager
aws secretsmanager list-secrets
aws secretsmanager get-secret-value --secret-id prod/db/password
aws secretsmanager create-secret --name prod/api-key --secret-string "mykey"
aws secretsmanager rotate-secret --secret-id prod/db/password

# RDS
aws rds describe-db-instances --output table
aws rds describe-db-snapshots --db-instance-identifier mydb
aws rds create-db-snapshot --db-instance-identifier mydb --db-snapshot-identifier manual-backup-20250710

# Cost
aws ce get-cost-and-usage \
  --time-period Start=2025-07-01,End=2025-07-10 \
  --granularity MONTHLY --metrics BlendedCost \
  --output table

# SSM (run commands without SSH)
aws ssm start-session --target i-abc123   # SSH-less session
aws ssm send-command \
  --instance-ids i-abc123 \
  --document-name AWS-RunShellScript \
  --parameters commands="df -h"
```

---

## 8. Azure CLI (az)

```bash
# Auth
az login                                # browser login
az account list --output table          # list subscriptions
az account set --subscription "Prod"    # switch subscription
az account show                         # current subscription

# Resource Groups
az group list --output table
az group create --name rg-myapp-prod --location eastus
az group delete --name rg-myapp-dev --yes --no-wait
az group export --name rg-myapp-prod > exported.json

# Virtual Machines
az vm list --output table
az vm list -d --query "[?powerState=='VM running'].name" -o tsv
az vm create --resource-group rg-myapp-prod --name vm-web01 \
  --image Ubuntu2204 --size Standard_B2s --generate-ssh-keys
az vm start --name vm-web01 --resource-group rg-myapp-prod
az vm stop --name vm-web01 --resource-group rg-myapp-prod
az vm deallocate --name vm-web01 --resource-group rg-myapp-prod  # stops billing!
az vm run-command invoke --resource-group rg-myapp-prod \
  --name vm-web01 --command-id RunShellScript --scripts "df -h"

# AKS
az aks list --output table
az aks get-credentials --resource-group rg-myapp-prod --name aks-main-prod
az aks nodepool list --resource-group rg-myapp-prod --cluster-name aks-main-prod
az aks upgrade --resource-group rg-myapp-prod --name aks-main-prod --kubernetes-version 1.30.0

# ACR (Container Registry)
az acr login --name myacr
az acr repository list --name myacr --output table
az acr repository show-tags --name myacr --repository myapp --output table

# Key Vault
az keyvault list --output table
az keyvault secret list --vault-name kv-myapp-prod
az keyvault secret set --vault-name kv-myapp-prod --name db-password --value "sup3rs3cr3t"
az keyvault secret show --vault-name kv-myapp-prod --name db-password --query value -o tsv
az keyvault secret list-deleted --vault-name kv-myapp-prod  # soft-deleted secrets
az keyvault secret recover --name db-password --vault-name kv-myapp-prod

# Monitor / Logs
az monitor metrics list --resource <resource-id> --metric "Percentage CPU"
az monitor log-analytics query \
  --workspace <workspace-id> \
  --analytics-query "AzureActivity | where OperationName contains 'delete' | take 20"

# Resource Locks
az lock create --name protect-prod-db --lock-type CanNotDelete \
  --resource-group rg-myapp-prod
az lock list --resource-group rg-myapp-prod

# Storage
az storage account list --output table
az storage blob list --container-name tfstate --account-name tfstateacct
az storage blob download --container-name tfstate \
  --name prod.tfstate --file backup.tfstate --account-name tfstateacct

# Useful query tricks
az vm list --query "[].{Name:name, Size:hardwareProfile.vmSize, RG:resourceGroup}" -o table
az resource list --tag environment=dev --output table
az resource list --query "length([?type=='Microsoft.Compute/virtualMachines'])"
```

---

## 9. Bash Scripting Essentials

```bash
#!/usr/bin/env bash
set -euo pipefail           # ALWAYS put this on line 2

# Variables
NAME="production"
readonly VERSION="2.1.0"    # constant
TODAY=$(date +%Y-%m-%d)     # command substitution

# Conditionals
if [[ -f "/etc/nginx/nginx.conf" ]]; then
  echo "nginx installed"
elif [[ -d "/etc/apache2" ]]; then
  echo "apache installed"
else
  echo "no web server"
fi

# One-liners
[[ -d "/backups" ]] || mkdir -p /backups         # create if not exists
[[ -z "$VAR" ]] && echo "VAR is empty"           # check empty
command -v docker &>/dev/null || { echo "docker not installed"; exit 1; }

# Loops
for server in web01 web02 db01; do
  echo "Checking $server..."
  ssh "$server" uptime || echo "$server is DOWN"
done

# Retry pattern (very common in DevOps)
until curl -sf http://localhost:8080/health; do
  echo "Waiting for app..."
  sleep 5
done
echo "App is healthy!"

# Functions
log() {
  local level="$1" msg="$2"
  echo "[$(date '+%H:%M:%S')] [$level] $msg"
}
log "INFO" "Deployment started"
log "ERROR" "Something failed"

# Default values
DB_HOST="${DB_HOST:-localhost}"
DB_PORT="${DB_PORT:-5432}"
REQUIRED="${MUST_SET:?'Error: MUST_SET is required'}"

# Argument parsing
if [[ $# -lt 2 ]]; then
  echo "Usage: $0 <environment> <version>"
  exit 1
fi
ENV="$1"; VERSION="$2"

# Trap for cleanup on exit
cleanup() { echo "Cleaning up..."; rm -f /tmp/lockfile; }
trap cleanup EXIT INT TERM

# Check exit codes
if grep -q "ERROR" /var/log/app.log; then
  echo "Errors found"
fi
echo "Exit code: $?"
```

---

## 10. Python (DevOps Automation)

```python
import subprocess, boto3, json, os, sys
from pathlib import Path

# Run shell commands
result = subprocess.run(["kubectl", "get", "pods"], capture_output=True, text=True)
print(result.stdout)
if result.returncode != 0:
    sys.exit(1)

# AWS SDK (boto3)
ec2 = boto3.client('ec2', region_name='us-east-1')
instances = ec2.describe_instances(
    Filters=[{'Name': 'tag:Environment', 'Values': ['dev']}]
)

# HTTP requests
import requests
resp = requests.get("https://api.example.com/health", timeout=10)
resp.raise_for_status()   # raises exception if 4xx/5xx

# JSON / YAML
import yaml
with open("values.yaml") as f:
    config = yaml.safe_load(f)

# Environment variables
db_host = os.environ.get("DB_HOST", "localhost")
secret = os.environ["SECRET_KEY"]   # raises if missing

# Logging (always use logging, not print)
import logging
logging.basicConfig(level=logging.INFO, format='%(asctime)s %(levelname)s %(message)s')
log = logging.getLogger(__name__)
log.info("Deployment started")
log.error("Connection failed", exc_info=True)
```

---

## 11. Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│  SITUATION → COMMAND                                            │
│                                                                 │
│  Check if port is open          ss -tlnp | grep 8080           │
│  Follow app logs                tail -f /var/log/app.log        │
│  See K8s pod crash reason       kubectl describe pod <name>     │
│  Get K8s pod logs (crashed)     kubectl logs <pod> --previous   │
│  Shell into K8s pod             kubectl exec -it <pod> -- bash  │
│  Forward K8s port locally       kubectl port-forward svc/x 8080:80│
│  Roll back K8s deploy           kubectl rollout undo deploy/x   │
│  Find which process uses port   ss -tlnp | grep :443           │
│  Check disk space               df -h                           │
│  Check memory                   free -h                         │
│  Top CPU processes              ps aux --sort=-%cpu | head -10  │
│  Run cmd on remote server       ssh user@host "df -h"           │
│  Copy file from server          scp user@host:/remote ./local   │
│  See terraform changes          terraform plan                   │
│  Roll back Helm release         helm rollback <release> <rev>   │
│  Decode K8s secret              kubectl get secret x -o jsonpath│
│                                 ='{.data.pass}' | base64 -d     │
│  AWS: who am I?                 aws sts get-caller-identity      │
│  Azure: switch subscription     az account set --subscription x  │
│  Docker: shell into container   docker exec -it <ctr> bash      │
│  Docker: see resource usage     docker stats                     │
│  Find file on system            find / -name "nginx.conf" 2>/dev/null│
│  Count log errors               grep -c "ERROR" app.log         │
└─────────────────────────────────────────────────────────────────┘
```
