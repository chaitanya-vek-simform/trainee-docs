# 🐍 Python for DevOps Automation

> Real-world Python scripts for automating DevOps tasks.
> Libraries: subprocess, boto3, azure-sdk, fabric, paramiko, requests, pyyaml, docker-py

---

## 1. Why Python Over Bash for DevOps

```
Use BASH when:                    Use PYTHON when:
─────────────────────────────     ──────────────────────────────────
Chaining CLI commands             Complex logic / data transformation
Quick one-off scripts             Working with APIs (REST, Azure, AWS)
File system operations            Parsing JSON/YAML at scale
Calling existing CLI tools        Error handling needs to be robust
Script < 50 lines                 Script > 50 lines
                                  Reusable modules needed
                                  Working with SDKs (boto3, azure)
                                  Sending reports / emails
```

---

## 2. Running Shell Commands from Python

```python
import subprocess

# Run command, capture output, raise on failure
result = subprocess.run(
    ["kubectl", "get", "pods", "-n", "production"],
    capture_output=True,
    text=True,
    check=True          # Raises CalledProcessError on non-zero exit
)
print(result.stdout)

# Capture stdout and stderr separately
result = subprocess.run(
    ["helm", "upgrade", "--install", "myapp", "./chart"],
    capture_output=True, text=True
)
if result.returncode != 0:
    print(f"FAILED: {result.stderr}")
else:
    print(f"OK: {result.stdout}")

# Run shell pipeline (needs shell=True — use carefully)
result = subprocess.run(
    "kubectl get pods -A | grep Error | wc -l",
    shell=True, capture_output=True, text=True, check=True
)
error_count = int(result.stdout.strip())

# Stream output live (don't capture — print as it runs)
proc = subprocess.Popen(
    ["kubectl", "rollout", "status", "deployment/myapp", "-n", "prod"],
    stdout=subprocess.PIPE, stderr=subprocess.STDOUT, text=True
)
for line in proc.stdout:
    print(line, end="")
proc.wait()
if proc.returncode != 0:
    raise RuntimeError("Rollout failed")

# Helper wrapper used throughout
def run(cmd: list[str], check=True, **kwargs) -> subprocess.CompletedProcess:
    print(f"  → {' '.join(cmd)}")
    return subprocess.run(cmd, capture_output=True, text=True, check=check, **kwargs)
```

---

## 3. Working with YAML & JSON

```python
import yaml, json
from pathlib import Path

# Read YAML (Kubernetes manifest)
with open("deployment.yaml") as f:
    manifest = yaml.safe_load(f)

# Navigate
image = manifest["spec"]["template"]["spec"]["containers"][0]["image"]
replicas = manifest["spec"]["replicas"]
print(f"Image: {image}, Replicas: {replicas}")

# Update and write back
manifest["spec"]["replicas"] = 3
manifest["spec"]["template"]["spec"]["containers"][0]["image"] = "myapp:v2"
with open("deployment.yaml", "w") as f:
    yaml.dump(manifest, f, default_flow_style=False)

# Update image tag in ALL manifests in a directory
def update_image_tag(manifests_dir: str, old_tag: str, new_tag: str):
    for path in Path(manifests_dir).rglob("*.yaml"):
        with open(path) as f:
            content = f.read()
        if old_tag in content:
            updated = content.replace(old_tag, new_tag)
            with open(path, "w") as f:
                f.write(updated)
            print(f"Updated: {path}")

update_image_tag("./k8s", "myapp:old-sha", "myapp:new-sha")

# Read multi-document YAML (Kubernetes resources separated by ---)
with open("all-resources.yaml") as f:
    docs = list(yaml.safe_load_all(f))
for doc in docs:
    print(f"{doc['kind']}: {doc['metadata']['name']}")

# Parse JSON API response
import json, urllib.request
with urllib.request.urlopen("https://api.example.com/health") as r:
    data = json.loads(r.read())
print(data["status"])
```

---

## 4. HTTP Requests & API Automation

```python
import requests

# Basic GET with auth
response = requests.get(
    "https://api.example.com/deployments",
    headers={"Authorization": f"Bearer {token}"},
    timeout=10
)
response.raise_for_status()   # Raises on 4xx/5xx
data = response.json()

# POST JSON
response = requests.post(
    "https://api.example.com/deploy",
    json={"environment": "staging", "image_tag": "sha-abc123"},
    headers={"Authorization": f"Bearer {token}"},
    timeout=30
)
response.raise_for_status()

# Session (reuse connection + auth headers)
session = requests.Session()
session.headers.update({"Authorization": f"Bearer {token}"})
session.timeout = 10

r1 = session.get("https://api.example.com/pods")
r2 = session.get("https://api.example.com/services")

# Retry with backoff
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

retry = Retry(total=3, backoff_factor=1, status_forcelist=[500, 502, 503])
adapter = HTTPAdapter(max_retries=retry)
session.mount("https://", adapter)

# Health check with retries
import time

def wait_for_health(url: str, max_wait: int = 120, interval: int = 5) -> bool:
    start = time.time()
    while time.time() - start < max_wait:
        try:
            r = requests.get(url, timeout=5)
            if r.status_code == 200:
                print(f"✅ {url} healthy after {int(time.time()-start)}s")
                return True
        except requests.RequestException:
            pass
        print(f"  Waiting... {int(time.time()-start)}s elapsed")
        time.sleep(interval)
    print(f"❌ Timeout waiting for {url}")
    return False

wait_for_health("https://staging.example.com/health")

# Send Slack notification
def notify_slack(webhook_url: str, message: str, emoji: str = "rocket"):
    requests.post(webhook_url, json={"text": f":{emoji}: {message}"}, timeout=5)

notify_slack(SLACK_WEBHOOK, f"✅ Deployed myapp:{IMAGE_TAG} → staging")
```

---

## 5. Azure SDK Automation

```python
from azure.identity import DefaultAzureCredential, ManagedIdentityCredential
from azure.mgmt.compute import ComputeManagementClient
from azure.mgmt.resource import ResourceManagementClient
from azure.keyvault.secrets import SecretClient

SUBSCRIPTION_ID = "your-sub-id"
credential = DefaultAzureCredential()   # Uses MI in Azure, CLI locally

# ── Resource Groups ───────────────────────────────────────────
rg_client = ResourceManagementClient(credential, SUBSCRIPTION_ID)

# List all resource groups
for rg in rg_client.resource_groups.list():
    print(f"{rg.name}: {rg.location}")

# Create resource group
rg_client.resource_groups.create_or_update(
    "rg-myapp-prod",
    {"location": "eastus", "tags": {"env": "prod", "team": "backend"}}
)

# ── Virtual Machines ──────────────────────────────────────────
compute = ComputeManagementClient(credential, SUBSCRIPTION_ID)

# List VMs and their power states
for vm in compute.virtual_machines.list_all():
    iv = compute.virtual_machines.instance_view(vm.id.split("/")[4], vm.name)
    statuses = [s.display_status for s in iv.statuses]
    print(f"{vm.name}: {', '.join(statuses)}")

# Deallocate all VMs in a resource group (stop billing)
def deallocate_all_vms(rg_name: str):
    from concurrent.futures import ThreadPoolExecutor
    vms = list(compute.virtual_machines.list(rg_name))
    print(f"Deallocating {len(vms)} VMs in {rg_name}...")

    def deallocate(vm):
        poller = compute.virtual_machines.begin_deallocate(rg_name, vm.name)
        poller.result()   # Wait for completion
        print(f"  ✅ Deallocated: {vm.name}")

    with ThreadPoolExecutor(max_workers=5) as ex:
        ex.map(deallocate, vms)

# ── Key Vault ─────────────────────────────────────────────────
kv_client = SecretClient(
    vault_url="https://kv-myapp.vault.azure.net",
    credential=credential
)

# Get a secret
db_password = kv_client.get_secret("db-password").value

# Set a secret
kv_client.set_secret("api-key", "myapikey123")

# List all secret names
for secret in kv_client.list_properties_of_secrets():
    print(f"{secret.name}: enabled={secret.enabled}")

# Load all secrets into a dict
def load_all_secrets(vault_url: str) -> dict:
    client = SecretClient(vault_url=vault_url, credential=DefaultAzureCredential())
    secrets = {}
    for prop in client.list_properties_of_secrets():
        if prop.enabled:
            secrets[prop.name] = client.get_secret(prop.name).value
    return secrets

env_secrets = load_all_secrets("https://kv-myapp.vault.azure.net")
```

---

## 6. Docker SDK Automation

```python
import docker

client = docker.from_env()

# List running containers
for c in client.containers.list():
    print(f"{c.name}: {c.status} | {c.image.tags}")

# Get containers over memory threshold
def find_high_memory_containers(threshold_mb: int = 512):
    results = []
    for c in client.containers.list():
        stats = c.stats(stream=False)
        mem_mb = stats["memory_stats"]["usage"] / 1024 / 1024
        if mem_mb > threshold_mb:
            results.append({"name": c.name, "memory_mb": round(mem_mb, 1)})
    return sorted(results, key=lambda x: x["memory_mb"], reverse=True)

high_mem = find_high_memory_containers(512)
for c in high_mem:
    print(f"⚠️  {c['name']}: {c['memory_mb']}MB")

# Build and push image
def build_and_push(path: str, tag: str, registry: str):
    image, logs = client.images.build(path=path, tag=f"{registry}/{tag}")
    for log in logs:
        if "stream" in log:
            print(log["stream"], end="")

    print(f"\nPushing {registry}/{tag}...")
    for line in client.images.push(f"{registry}/{tag}", stream=True, decode=True):
        if "status" in line:
            print(f"  {line['status']}")

    print("✅ Push complete")

# Cleanup: remove stopped containers and dangling images
def docker_cleanup(dry_run: bool = False):
    stopped = client.containers.list(filters={"status": "exited"})
    dangling = client.images.list(filters={"dangling": True})

    print(f"Found {len(stopped)} stopped containers, {len(dangling)} dangling images")
    if dry_run:
        print("[DRY-RUN] Would remove all of the above")
        return

    client.containers.prune()
    client.images.prune()
    print("✅ Cleanup done")

# Pull image with retry
def pull_image(image_ref: str, max_retries: int = 3):
    for attempt in range(1, max_retries + 1):
        try:
            client.images.pull(image_ref)
            print(f"✅ Pulled: {image_ref}")
            return
        except docker.errors.APIError as e:
            print(f"Attempt {attempt}/{max_retries} failed: {e}")
            time.sleep(attempt * 2)
    raise RuntimeError(f"Failed to pull {image_ref}")
```

---

## 7. Kubernetes Python Client

```python
from kubernetes import client, config, watch

# Load config (in-cluster or local kubeconfig)
try:
    config.load_incluster_config()    # Running inside a pod
except config.ConfigException:
    config.load_kube_config()         # Running locally

v1 = client.CoreV1Api()
apps_v1 = client.AppsV1Api()

# List pods by namespace
pods = v1.list_namespaced_pod("production")
for pod in pods.items:
    phase = pod.status.phase
    restarts = sum(cs.restart_count for cs in (pod.status.container_statuses or []))
    print(f"{pod.metadata.name}: {phase} (restarts: {restarts})")

# Find pods with high restarts
def find_crashing_pods(namespace: str = "production", threshold: int = 5):
    pods = v1.list_namespaced_pod(namespace)
    crashing = []
    for pod in pods.items:
        for cs in pod.status.container_statuses or []:
            if cs.restart_count >= threshold:
                crashing.append({
                    "pod": pod.metadata.name,
                    "container": cs.name,
                    "restarts": cs.restart_count,
                    "state": list(cs.state.to_dict().keys())[0]
                })
    return crashing

# Scale a deployment
def scale_deployment(name: str, namespace: str, replicas: int):
    apps_v1.patch_namespaced_deployment_scale(
        name=name, namespace=namespace,
        body={"spec": {"replicas": replicas}}
    )
    print(f"✅ Scaled {namespace}/{name} → {replicas} replicas")

# Rolling restart (patch annotation to force new rollout)
import datetime
def rolling_restart(name: str, namespace: str):
    now = datetime.datetime.utcnow().isoformat() + "Z"
    body = {"spec": {"template": {"metadata": {"annotations": {
        "kubectl.kubernetes.io/restartedAt": now
    }}}}}
    apps_v1.patch_namespaced_deployment(name, namespace, body)
    print(f"✅ Triggered rolling restart: {namespace}/{name}")

# Watch events for errors (stream live)
w = watch.Watch()
for event in w.stream(v1.list_event_for_all_namespaces, timeout_seconds=60):
    ev = event["object"]
    if ev.type in ("Warning", "Failed"):
        print(f"⚠️  [{ev.metadata.namespace}] {ev.reason}: {ev.message}")

# Create or update a ConfigMap
def upsert_configmap(name: str, namespace: str, data: dict):
    cm = client.V1ConfigMap(
        metadata=client.V1ObjectMeta(name=name, namespace=namespace),
        data=data
    )
    try:
        v1.replace_namespaced_config_map(name, namespace, cm)
        print(f"Updated ConfigMap: {name}")
    except client.ApiException as e:
        if e.status == 404:
            v1.create_namespaced_config_map(namespace, cm)
            print(f"Created ConfigMap: {name}")
        else:
            raise
```

---

## 8. File & System Automation

```python
import os, shutil, hashlib
from pathlib import Path

# Find files matching pattern recursively
def find_files(base_dir: str, pattern: str, exclude_dirs: list = None):
    exclude_dirs = exclude_dirs or [".venv", "node_modules", "__pycache__"]
    for path in Path(base_dir).rglob(pattern):
        if not any(ex in path.parts for ex in exclude_dirs):
            yield path

# Count lines of code per extension
def count_loc(base_dir: str) -> dict:
    counts = {}
    for path in Path(base_dir).rglob("*"):
        if path.is_file():
            ext = path.suffix or "no-ext"
            try:
                lines = len(path.read_text(errors="ignore").splitlines())
                counts[ext] = counts.get(ext, 0) + lines
            except Exception:
                pass
    return dict(sorted(counts.items(), key=lambda x: x[1], reverse=True))

# Find duplicate files by MD5 hash
def find_duplicates(directory: str) -> dict:
    hashes = {}
    for path in Path(directory).rglob("*"):
        if path.is_file():
            h = hashlib.md5(path.read_bytes()).hexdigest()
            hashes.setdefault(h, []).append(str(path))
    return {h: paths for h, paths in hashes.items() if len(paths) > 1}

# Find files over a size threshold
def find_large_files(directory: str, min_mb: int = 100):
    threshold = min_mb * 1024 * 1024
    return [
        (p, p.stat().st_size / 1024 / 1024)
        for p in Path(directory).rglob("*")
        if p.is_file() and p.stat().st_size > threshold
    ]

# Rotate logs: compress old, delete ancient
def rotate_logs(log_dir: str, compress_after_days: int = 1, delete_after_days: int = 30):
    import gzip, time
    now = time.time()
    for path in Path(log_dir).rglob("*.log"):
        age_days = (now - path.stat().st_mtime) / 86400
        if age_days > delete_after_days:
            path.unlink()
            print(f"Deleted: {path}")
        elif age_days > compress_after_days:
            gz_path = path.with_suffix(".log.gz")
            with path.open("rb") as f_in, gzip.open(gz_path, "wb") as f_out:
                shutil.copyfileobj(f_in, f_out)
            path.unlink()
            print(f"Compressed: {gz_path}")
```

---

## 9. Reporting & Alerting Scripts

```python
import smtplib, json
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from datetime import datetime

# Send HTML email report
def send_email_report(to: str, subject: str, html_body: str):
    msg = MIMEMultipart("alternative")
    msg["Subject"] = subject
    msg["From"] = SMTP_FROM
    msg["To"] = to
    msg.attach(MIMEText(html_body, "html"))

    with smtplib.SMTP(SMTP_HOST, SMTP_PORT) as server:
        server.starttls()
        server.login(SMTP_USER, SMTP_PASS)
        server.send_message(msg)

# Generate infrastructure report
def generate_infra_report() -> str:
    from kubernetes import client, config
    config.load_kube_config()
    v1 = client.CoreV1Api()
    apps_v1 = client.AppsV1Api()

    pods = v1.list_pod_for_all_namespaces()
    deployments = apps_v1.list_deployment_for_all_namespaces()

    total_pods = len(pods.items)
    running_pods = sum(1 for p in pods.items if p.status.phase == "Running")
    crashing = sum(1 for p in pods.items
                   for cs in (p.status.container_statuses or [])
                   if cs.restart_count > 5)

    unhealthy_deps = [
        d for d in deployments.items
        if (d.status.ready_replicas or 0) < d.spec.replicas
    ]

    lines = [
        f"<h2>Infrastructure Report — {datetime.now().strftime('%Y-%m-%d %H:%M')}</h2>",
        f"<p>Pods: {running_pods}/{total_pods} running | {crashing} crashing</p>",
    ]
    if unhealthy_deps:
        lines.append("<h3>⚠️ Unhealthy Deployments</h3><ul>")
        for d in unhealthy_deps:
            lines.append(f"<li>{d.metadata.namespace}/{d.metadata.name}: "
                         f"{d.status.ready_replicas or 0}/{d.spec.replicas}</li>")
        lines.append("</ul>")
    else:
        lines.append("<p>✅ All deployments healthy</p>")

    return "\n".join(lines)

report_html = generate_infra_report()
send_email_report("ops@company.com", "Daily Infra Report", report_html)

# Multi-channel alert (Slack + email on critical)
def alert(message: str, severity: str = "info"):
    # Always Slack
    requests.post(SLACK_WEBHOOK, json={"text": f"[{severity.upper()}] {message}"})

    # Email only on critical
    if severity == "critical":
        send_email_report(
            "oncall@company.com",
            f"🚨 CRITICAL: {message[:60]}",
            f"<h2>🚨 Critical Alert</h2><p>{message}</p>"
        )
```

---

## 10. Argparse CLI Tools

```python
#!/usr/bin/env python3
"""DevOps CLI tool — deploy, health-check, cleanup."""

import argparse, sys

def cmd_deploy(args):
    print(f"Deploying {args.image} to {args.env}...")
    # deployment logic here

def cmd_health(args):
    print(f"Checking health of {args.env}...")
    # health check logic here

def cmd_cleanup(args):
    days = args.days
    print(f"Cleaning up resources older than {days} days...")
    # cleanup logic here

def main():
    parser = argparse.ArgumentParser(
        description="DevOps automation CLI",
        formatter_class=argparse.RawDescriptionHelpFormatter
    )
    parser.add_argument("--dry-run", action="store_true", help="Print without executing")
    parser.add_argument("--verbose", "-v", action="store_true")

    subparsers = parser.add_subparsers(dest="command", required=True)

    # deploy subcommand
    deploy_p = subparsers.add_parser("deploy", help="Deploy application")
    deploy_p.add_argument("--env", required=True, choices=["dev", "staging", "prod"])
    deploy_p.add_argument("--image", required=True, help="Image tag")
    deploy_p.set_defaults(func=cmd_deploy)

    # health subcommand
    health_p = subparsers.add_parser("health", help="Check environment health")
    health_p.add_argument("--env", required=True)
    health_p.set_defaults(func=cmd_health)

    # cleanup subcommand
    cleanup_p = subparsers.add_parser("cleanup", help="Remove old resources")
    cleanup_p.add_argument("--days", type=int, default=30)
    cleanup_p.set_defaults(func=cmd_cleanup)

    args = parser.parse_args()
    args.func(args)

if __name__ == "__main__":
    main()

# Usage:
# python tool.py deploy --env staging --image myapp:v1.2
# python tool.py health --env production
# python tool.py cleanup --days 7 --dry-run
```

---

## 11. Key Libraries Reference

```
Library                  Install                         Use Case
──────────────────────────────────────────────────────────────────
requests                 pip install requests            HTTP API calls
azure-identity           pip install azure-identity      Azure auth (OIDC/MI)
azure-keyvault-secrets   pip install azure-keyvault-...  Key Vault secrets
azure-mgmt-compute       pip install azure-mgmt-compute  VM management
azure-mgmt-resource      pip install azure-mgmt-resource Resource groups
boto3                    pip install boto3               AWS SDK
kubernetes               pip install kubernetes          K8s Python client
docker                   pip install docker              Docker SDK
paramiko                 pip install paramiko            SSH connections
fabric                   pip install fabric              SSH + remote exec
pyyaml                   pip install pyyaml              YAML parsing
click                    pip install click               Better CLI than argparse
rich                     pip install rich                Pretty terminal output
pydantic                 pip install pydantic            Config validation
schedule                 pip install schedule            In-process scheduling
python-dotenv            pip install python-dotenv       Load .env files
```

```python
# Load .env file (development)
from dotenv import load_dotenv
import os
load_dotenv()
DB_URL = os.environ["DATABASE_URL"]   # Raises if not set
PORT   = os.getenv("PORT", "8000")    # With default

# Config validation with pydantic
from pydantic import BaseSettings

class Config(BaseSettings):
    db_url: str
    api_key: str
    environment: str = "dev"
    replicas: int = 2

    class Config:
        env_file = ".env"

config = Config()   # Validates and loads from env / .env
print(config.environment)

# Pretty output with rich
from rich.console import Console
from rich.table import Table
console = Console()

table = Table(title="Pod Status")
table.add_column("Pod", style="cyan")
table.add_column("Status", style="green")
table.add_column("Restarts", style="yellow")
table.add_row("myapp-abc123", "Running", "0")
table.add_row("myapp-def456", "CrashLoop", "12")
console.print(table)
```
