# 🐚 Bash Scripting for DevOps Automation

> Production-grade bash patterns for automating real DevOps tasks.

---

## 1. Non-Negotiable Script Header

```bash
#!/usr/bin/env bash
set -euo pipefail
# -e  : exit immediately on any error
# -u  : treat unset variables as errors
# -o pipefail : pipeline fails if ANY step fails
IFS=$'\n\t'   # safer word splitting
```

---

## 2. Script Template

```bash
#!/usr/bin/env bash
set -euo pipefail

readonly SCRIPT_NAME="$(basename "${BASH_SOURCE[0]}")"
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly LOG_FILE="/var/log/${SCRIPT_NAME%.sh}.log"

# Colors (only when stdout is a terminal)
if [[ -t 1 ]]; then
  RED='\033[0;31m'; GREEN='\033[0;32m'; YELLOW='\033[1;33m'; NC='\033[0m'
else
  RED=''; GREEN=''; YELLOW=''; NC=''
fi

log()  { echo -e "${GREEN}[$(date '+%H:%M:%S')] ✅ $*${NC}" | tee -a "$LOG_FILE"; }
warn() { echo -e "${YELLOW}[$(date '+%H:%M:%S')] ⚠️  $*${NC}" | tee -a "$LOG_FILE" >&2; }
err()  { echo -e "${RED}[$(date '+%H:%M:%S')] ❌ $*${NC}" | tee -a "$LOG_FILE" >&2; }
die()  { err "$*"; exit 1; }

usage() {
  cat <<EOF
Usage: ${SCRIPT_NAME} [OPTIONS]
  -e, --env ENV       Target environment (dev|staging|prod)
  -v, --version TAG   Image tag to deploy
  -d, --dry-run       Print actions without executing
  -h, --help          Show this help
EOF
  exit 0
}

# Argument parsing
ENV=""; VERSION=""; DRY_RUN=false
while [[ $# -gt 0 ]]; do
  case "$1" in
    -e|--env)      ENV="$2";     shift 2 ;;
    -v|--version)  VERSION="$2"; shift 2 ;;
    -d|--dry-run)  DRY_RUN=true; shift   ;;
    -h|--help)     usage ;;
    *) die "Unknown option: $1" ;;
  esac
done

# Validation
[[ -z "$ENV" ]]     && die "--env is required"
[[ -z "$VERSION" ]] && die "--version is required"
[[ "$ENV" =~ ^(dev|staging|prod)$ ]] || die "Invalid env: $ENV"

# Cleanup on exit
TEMP_DIR=""
cleanup() {
  local code=$?
  [[ -n "$TEMP_DIR" && -d "$TEMP_DIR" ]] && rm -rf "$TEMP_DIR"
  [[ $code -ne 0 ]] && err "Script failed (exit $code). Check: $LOG_FILE"
}
trap cleanup EXIT
trap 'die "Interrupted"' INT TERM

# Dry-run wrapper
run() {
  if [[ "$DRY_RUN" == "true" ]]; then
    echo "[DRY-RUN] $*"
  else
    log "Running: $*"
    "$@"
  fi
}

main() {
  TEMP_DIR=$(mktemp -d)
  log "Starting ${SCRIPT_NAME} — env=$ENV version=$VERSION"
  run kubectl set image deployment/myapp app="myapp:${VERSION}" -n "$ENV"
  log "Done ✅"
}

main "$@"
```

---

## 3. Error Handling Patterns

```bash
# Pattern 1: Check before acting
if ! kubectl get namespace "$NS" &>/dev/null; then
  kubectl create namespace "$NS"
fi

# Pattern 2: OR-die
az group show --name "$RG" &>/dev/null || die "RG '$RG' not found"

# Pattern 3: Capture output and check
if OUTPUT=$(kubectl apply -f deploy.yaml 2>&1); then
  log "Apply ok: $OUTPUT"
else
  err "Apply failed: $OUTPUT"; exit 1
fi

# Pattern 4: Retry with exponential backoff
retry() {
  local max=$1; shift
  local delay=1 attempt=1
  while true; do
    "$@" && return 0
    [[ $attempt -ge $max ]] && { err "Failed after $max attempts: $*"; return 1; }
    warn "Attempt $attempt/$max failed. Retry in ${delay}s..."
    sleep "$delay"; delay=$(( delay * 2 )); (( attempt++ ))
  done
}
retry 5 curl -sf http://api/health
retry 3 helm upgrade --install myapp ./chart

# Pattern 5: Timeout wrapper
run_timeout() {
  local t=$1; shift
  timeout "$t" "$@" || {
    local c=$?
    [[ $c -eq 124 ]] && die "Timed out after ${t}s: $*"
    die "Failed (exit $c): $*"
  }
}
run_timeout 60 kubectl rollout status deployment/myapp

# Pattern 6: Lock file (prevent concurrent runs)
LOCK="/tmp/${SCRIPT_NAME}.lock"
mkdir "$LOCK" 2>/dev/null || die "Already running (lock: $LOCK)"
echo $$ > "$LOCK/pid"
trap 'rm -rf "$LOCK"' EXIT
```

---

## 4. Real Script — Kubernetes Deployment

```bash
#!/usr/bin/env bash
set -euo pipefail
log() { echo "[$(date '+%H:%M:%S')] $*"; }
die() { echo "ERROR: $*" >&2; exit 1; }

ENV="${1:-}"; IMAGE_TAG="${2:-}"
[[ -z "$ENV" || -z "$IMAGE_TAG" ]] && die "Usage: $0 <env> <tag>"
[[ "$ENV" =~ ^(staging|production)$ ]] || die "Invalid env: $ENV"

CHART_DIR="$(dirname "$0")/../charts/myapp"
VALUES="$(dirname "$0")/../values/${ENV}.yaml"
[[ -f "$VALUES" ]] || die "Values file not found: $VALUES"

log "Pre-deploy checks..."
kubectl cluster-info &>/dev/null || die "Cannot connect to cluster"
kubectl get namespace "$ENV" &>/dev/null || kubectl create namespace "$ENV"

log "Deploying myapp:${IMAGE_TAG} to ${ENV}..."
helm upgrade --install myapp "$CHART_DIR" \
  --namespace "$ENV" --values "$VALUES" \
  --set image.tag="$IMAGE_TAG" \
  --wait --timeout 5m --atomic

log "Smoke testing..."
RETRIES=0
until curl -sf "https://${ENV}.example.com/health" &>/dev/null; do
  (( RETRIES++ )); [[ $RETRIES -gt 10 ]] && die "Health check failed"
  log "  Attempt $RETRIES/10, waiting 10s..."
  sleep 10
done

log "✅ Deployed myapp:${IMAGE_TAG} → ${ENV}"

[[ -n "${SLACK_WEBHOOK:-}" ]] && \
  curl -s -X POST "$SLACK_WEBHOOK" \
    -H "Content-Type: application/json" \
    -d "{\"text\":\"✅ myapp \`${IMAGE_TAG}\` → \`${ENV}\`\"}" &>/dev/null || true
```

---

## 5. Real Script — Infrastructure Health Check

```bash
#!/usr/bin/env bash
set -euo pipefail
PASS=0; FAIL=0; WARN=0

ok()   { echo "✅ PASS: $1"; ((PASS++)); }
fail() { echo "❌ FAIL: $1"; ((FAIL++)); }
warn() { echo "⚠️  WARN: $1"; ((WARN++)); }

check() {
  local name="$1"; shift
  if "$@" &>/dev/null; then ok "$name"; else fail "$name"; fi
}

echo "=== Health Check: $(date) ==="

echo "--- Kubernetes ---"
check "Cluster reachable"    kubectl cluster-info
check "No CrashLoop pods"    bash -c '! kubectl get pods -A | grep -E "CrashLoop|OOMKill"'
check "No Pending pods"      bash -c '! kubectl get pods -A | grep Pending'

echo "--- Endpoints ---"
for url in "https://api.example.com/health" "https://app.example.com/health"; do
  code=$(curl -s -o /dev/null -w '%{http_code}' --max-time 5 "$url")
  [[ "$code" == "200" ]] && ok "$url" || fail "$url (got $code)"
done

echo "--- SSL Certs ---"
for domain in api.example.com app.example.com; do
  days=$(echo | openssl s_client -servername "$domain" -connect "$domain:443" 2>/dev/null \
    | openssl x509 -noout -enddate 2>/dev/null | cut -d= -f2 \
    | xargs -I{} bash -c 'echo $(( ($(date -d "{}" +%s) - $(date +%s)) / 86400 ))')
  if   [[ "$days" -gt 30 ]]; then ok "SSL $domain (${days}d)"
  elif [[ "$days" -gt 7  ]]; then warn "SSL $domain expires in ${days}d!"
  else                              fail "SSL $domain expires in ${days}d!"
  fi
done

echo "--- Disk Space ---"
df -h | grep -v tmpfs | awk 'NR>1' | while IFS= read -r line; do
  fs=$(echo "$line" | awk '{print $1}')
  use=$(echo "$line" | awk '{gsub(/%/,"",$5); print $5}')
  if   [[ "$use" -lt 80 ]]; then ok "$fs (${use}%)"
  elif [[ "$use" -lt 90 ]]; then warn "$fs (${use}%)"
  else                            fail "$fs CRITICAL (${use}%)"
  fi
done

echo ""; echo "Results: ✅ ${PASS} | ⚠️  ${WARN} | ❌ ${FAIL}"
[[ $FAIL -gt 0 ]] && exit 1 || exit 0
```

---

## 6. Real Script — DB Backup with Rotation

```bash
#!/usr/bin/env bash
set -euo pipefail
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
die() { echo "ERROR: $*" >&2; exit 1; }

: "${DB_HOST:?DB_HOST not set}" "${DB_NAME:?}" "${DB_USER:?}" "${DB_PASSWORD:?}"
BACKUP_DIR="${BACKUP_DIR:-/backups/postgres}"
RETENTION_DAYS="${RETENTION_DAYS:-7}"

TIMESTAMP=$(date '+%Y%m%d_%H%M%S')
FILE="${BACKUP_DIR}/${DB_NAME}_${TIMESTAMP}.sql.gz"
mkdir -p "$BACKUP_DIR"

log "Backing up ${DB_NAME} → ${FILE}"
PGPASSWORD="$DB_PASSWORD" pg_dump -h "$DB_HOST" -U "$DB_USER" -d "$DB_NAME" \
  | gzip > "$FILE" || die "pg_dump failed"

[[ $(stat -c%s "$FILE") -lt 1000 ]] && die "Backup suspiciously small"
gunzip -c "$FILE" | grep -q "CREATE TABLE" || die "Backup may be corrupt"
log "✅ Backup valid: $(du -sh "$FILE" | cut -f1)"

log "Rotating backups older than ${RETENTION_DAYS} days..."
find "$BACKUP_DIR" -name "${DB_NAME}_*.sql.gz" -mtime "+${RETENTION_DAYS}" -delete

COUNT=$(find "$BACKUP_DIR" -name "${DB_NAME}_*.sql.gz" | wc -l)
SIZE=$(du -sh "$BACKUP_DIR" | cut -f1)
log "Done. ${COUNT} backups, ${SIZE} total."
```

---

## 7. Real Script — Key Vault → Kubernetes Secret Sync

```bash
#!/usr/bin/env bash
set -euo pipefail
log() { echo "[$(date '+%H:%M:%S')] $*"; }
die() { echo "ERROR: $*" >&2; exit 1; }

VAULT="${1:?Usage: $0 <vault-name> <k8s-namespace> <secret-name>}"
NS="${2:?}"; SECRET="${3:?}"

az keyvault show --name "$VAULT" &>/dev/null || die "Vault '$VAULT' not found"

log "Fetching secrets from $VAULT..."
mapfile -t NAMES < <(az keyvault secret list --vault-name "$VAULT" \
  --query "[?attributes.enabled].name" -o tsv)

[[ ${#NAMES[@]} -eq 0 ]] && die "No enabled secrets in vault"

ARGS=()
for name in "${NAMES[@]}"; do
  log "  Fetching: $name"
  val=$(az keyvault secret show --vault-name "$VAULT" --name "$name" --query value -o tsv)
  ARGS+=("--from-literal=${name}=${val}")
done

log "Syncing ${#NAMES[@]} secrets → k8s secret '$SECRET' in '$NS'..."
kubectl create secret generic "$SECRET" \
  --namespace "$NS" "${ARGS[@]}" \
  --save-config --dry-run=client -o yaml | kubectl apply -f -

log "✅ Synced ${#NAMES[@]} secrets"
```

---

## 8. Bash Functions Library

```bash
# Source with: source /usr/local/lib/devops-lib.sh

# Require tools to be installed
require() {
  for cmd in "$@"; do
    command -v "$cmd" &>/dev/null || { echo "Missing tool: $cmd"; exit 1; }
  done
}

# Require environment variables
require_env() {
  for var in "$@"; do
    [[ -z "${!var:-}" ]] && { echo "Required env var not set: $var"; exit 1; }
  done
}

# Confirm before destructive action
confirm() {
  read -rp "${1:-Are you sure?} [y/N] " ans
  [[ "${ans,,}" == "y" ]]
}

# Check if running in CI
is_ci() { [[ -n "${CI:-}" || -n "${GITHUB_ACTIONS:-}" || -n "${TF_BUILD:-}" ]]; }

# Wait for URL health
wait_for_url() {
  local url="$1" max="${2:-120}" elapsed=0
  echo "Waiting for $url (max ${max}s)..."
  until curl -sf "$url" &>/dev/null; do
    sleep 5; elapsed=$((elapsed + 5))
    [[ $elapsed -ge $max ]] && { echo "Timeout: $url"; return 1; }
    echo "  ${elapsed}s elapsed..."
  done
  echo "✅ $url healthy (${elapsed}s)"
}

# Check kubectl context matches expected
assert_context() {
  local expected="$1"
  local actual; actual=$(kubectl config current-context 2>/dev/null)
  [[ "$actual" == "$expected" ]] || {
    echo "Wrong context! Expected: $expected, got: $actual"; return 1
  }
}

# Notify Slack
slack() {
  [[ -z "${SLACK_WEBHOOK:-}" ]] && return 0
  curl -s -X POST "$SLACK_WEBHOOK" \
    -H "Content-Type: application/json" \
    -d "{\"text\": \"$1\"}" &>/dev/null || true
}

# Port reachability check
port_open() {
  timeout "${3:-3}" bash -c "cat < /dev/null > /dev/tcp/$1/$2" 2>/dev/null
}

# Human-readable duration from seconds
human_duration() {
  printf '%dh %dm %ds' $(($1/3600)) $(($1%3600/60)) $(($1%60))
}
```

---

## 9. Cron Automation

```bash
# Crontab syntax:  min  hour  dom  mon  dow  cmd
# 0 2 * * *      → daily 2AM
# 0 2 * * 1-5   → weekdays 2AM
# */15 * * * *  → every 15 minutes
# 0 0 1 * *     → first of month midnight
# @reboot        → on startup

# Add cron without editing manually:
(crontab -l 2>/dev/null; echo "0 2 * * * /opt/scripts/db-backup.sh >> /var/log/db-backup.log 2>&1") | crontab -

# Example crontab for DevOps:
# DB backup daily 2AM
# 0 2 * * * /opt/scripts/db-backup.sh >> /var/log/db-backup.log 2>&1
#
# Log cleanup weekly Sunday 3AM
# 0 3 * * 0 /opt/scripts/log-cleanup.sh >> /var/log/log-cleanup.log 2>&1
#
# Health check every 5 minutes
# */5 * * * * /opt/scripts/health-check.sh >> /var/log/health.log 2>&1
#
# SSL expiry check daily 6AM
# 0 6 * * * /opt/scripts/check-ssl.sh | mail -s "SSL Report" ops@company.com

# Use flock to prevent overlapping cron runs:
# */5 * * * * flock -n /tmp/health.lock /opt/scripts/health-check.sh
```

---

## 10. Variable & String Tricks

```bash
VAR="hello-world-foo"

# Manipulation
echo "${VAR^^}"        # HELLO-WORLD-FOO   uppercase
echo "${VAR,,}"        # hello-world-foo   lowercase
echo "${#VAR}"         # 15               length
echo "${VAR/foo/bar}"  # hello-world-bar  replace first
echo "${VAR//l/L}"     # heLLo-worLd-foo  replace all
echo "${VAR#hello-}"   # world-foo        remove prefix (shortest)
echo "${VAR##*-}"      # foo              remove prefix (longest)
echo "${VAR%-foo}"     # hello-world      remove suffix
echo "${VAR:6:5}"      # world            substring offset:length

# Defaults
echo "${UNSET:-default}"        # Use default if unset
echo "${UNSET:=default}"        # Set and use default
echo "${UNSET:?Error: not set}" # Exit with error if unset

# Arrays
SERVERS=("web01" "web02" "db01")
echo "${SERVERS[@]}"   # All elements
echo "${#SERVERS[@]}"  # Count: 3
echo "${SERVERS[0]}"   # First
echo "${SERVERS[-1]}"  # Last
SERVERS+=("cache01")   # Append

# Export .env safely
set -a; source .env; set +a

# Check required vars set
REQUIRED=(DB_URL API_KEY SECRET_KEY)
for var in "${REQUIRED[@]}"; do
  [[ -z "${!var}" ]] && { echo "ERROR: $var not set"; exit 1; }
done
