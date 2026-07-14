# 🐳 Docker Compose Notes — Part 2
> Part 2 of 2 | Covers: Docker Compose every field explained, tech stack setups, production patterns

---

## 1. Docker Compose — What & Why

```
┌──────────────────────────────────────────────────────────────┐
│  DOCKER COMPOSE = Run multiple containers together           │
│                                                              │
│  Without compose:                                            │
│  docker run -d --name db -e POSTGRES_PASSWORD=pass postgres  │
│  docker run -d --name redis redis:7                          │
│  docker run -d --name app -e DB_HOST=db -p 8000:8000 myapp  │
│  → Must remember all flags, order, manual networking         │
│                                                              │
│  With compose (docker-compose.yml):                          │
│  docker compose up -d                                        │
│  → One command, all services start in correct order         │
│  → Automatic networking between services                    │
│  → Stored in version control                                │
└──────────────────────────────────────────────────────────────┘
```

---

## 2. Every Docker Compose Field Explained

```yaml
# docker-compose.yml — complete field reference

name: myapp                      # Project name (default: directory name)

# ─── SERVICES ────────────────────────────────────────────────
services:

  api:                           # Service name (used as DNS hostname too)

    # ── Image source (pick ONE of: image OR build) ────────────
    image: nginx:1.25-alpine     # Use pre-built image from registry
    # OR
    build:
      context: ./backend         # Path to Dockerfile directory
      dockerfile: Dockerfile     # Default is "Dockerfile"
      target: runtime            # Multi-stage: which stage to build
      args:                      # Build arguments (ARG in Dockerfile)
        BUILD_ENV: production
        NODE_VERSION: "20"
      cache_from:                # External cache sources
        - myacr.azurecr.io/myapp:latest
      labels:
        com.example.version: "1.0"

    # ── Container name ────────────────────────────────────────
    container_name: myapp-api    # Fixed name (can't scale if set)

    # ── Ports: HOST:CONTAINER ─────────────────────────────────
    ports:
      - "8000:8000"              # Publish on all interfaces
      - "127.0.0.1:8000:8000"   # Publish on localhost ONLY (more secure)
      - "8000"                   # Random host port → container 8000

    # ── Environment variables ─────────────────────────────────
    environment:
      NODE_ENV: production
      PORT: "8000"
      DB_HOST: db                # Use service name as hostname
      DB_PORT: "5432"
    # OR load from file:
    env_file:
      - .env                     # Load all KEY=VALUE from file
      - .env.local               # Can stack multiple env files

    # ── Volumes ───────────────────────────────────────────────
    volumes:
      - ./src:/app/src           # Bind mount: host path → container path
      - app-data:/data           # Named volume (managed by Docker)
      - /tmp/cache:/cache        # Absolute host path
      - type: bind               # Long syntax
        source: ./config
        target: /app/config
        read_only: true          # Mount as read-only
      - type: volume
        source: db-data
        target: /var/lib/postgresql/data
        volume:
          nocopy: true           # Don't copy existing data from container

    # ── Dependencies ─────────────────────────────────────────
    depends_on:
      db:
        condition: service_healthy    # Wait for healthcheck to pass
      redis:
        condition: service_started    # Wait for container to start (not healthy)
      migration:
        condition: service_completed_successfully  # Wait for one-shot to finish

    # ── Networks ─────────────────────────────────────────────
    networks:
      - frontend-net             # Which networks this service joins
      - backend-net

    # ── Restart policy ────────────────────────────────────────
    restart: unless-stopped      # Options: no | always | on-failure | unless-stopped
    # no            → never restart (default)
    # always        → always restart (even on manual stop)
    # on-failure    → only restart on non-zero exit
    # unless-stopped→ restart always EXCEPT when manually stopped (BEST for prod)

    # ── Resource limits ───────────────────────────────────────
    deploy:
      resources:
        limits:
          cpus: "0.5"            # Max 50% of one CPU core
          memory: 512M           # Max 512MB RAM
        reservations:
          cpus: "0.25"           # Guaranteed 25% CPU
          memory: 256M           # Guaranteed 256MB RAM
      replicas: 1                # Number of instances (swarm mode)

    # ── Healthcheck ───────────────────────────────────────────
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s              # How often to check
      timeout: 10s               # Timeout per check
      retries: 3                 # Failures before unhealthy
      start_period: 20s          # Grace period at startup

    # ── Command & entrypoint ──────────────────────────────────
    command: uvicorn main:app --host 0.0.0.0 --port 8000
    # OR as list (preferred):
    command: ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
    entrypoint: ["/bin/sh", "-c"]

    # ── Logging ───────────────────────────────────────────────
    logging:
      driver: json-file          # Default driver
      options:
        max-size: "10m"          # Rotate at 10MB
        max-file: "3"            # Keep 3 rotated files
    # OR: driver: none (discard all logs)
    # OR: driver: syslog
    # OR: driver: fluentd (ship to Fluentd)

    # ── Labels ────────────────────────────────────────────────
    labels:
      - "traefik.enable=true"
      - "com.example.team=backend"

    # ── User ──────────────────────────────────────────────────
    user: "1001:1001"            # Override USER in Dockerfile

    # ── Working directory ─────────────────────────────────────
    working_dir: /app

    # ── stdin / tty (for interactive services) ────────────────
    stdin_open: true             # docker run -i
    tty: true                    # docker run -t

    # ── Privileged / capabilities ─────────────────────────────
    privileged: false            # NEVER true in prod
    cap_add:
      - NET_ADMIN                # Add specific capability
    cap_drop:
      - ALL                      # Drop all, then add only needed
    security_opt:
      - no-new-privileges:true   # Prevent privilege escalation

    # ── Extra hosts ───────────────────────────────────────────
    extra_hosts:
      - "myhost.internal:192.168.1.100"  # Add to /etc/hosts

    # ── Profile (selectively start services) ──────────────────
    profiles:
      - debug                    # Only starts with: docker compose --profile debug up

# ─── VOLUMES ────────────────────────────────────────────────
volumes:
  db-data:                       # Named volume (persists across up/down)
    driver: local                # Default driver
  app-data:
    external: true               # Volume created outside this compose file
  cache:
    driver_opts:
      type: tmpfs                # In-memory volume (not persisted)
      device: tmpfs

# ─── NETWORKS ───────────────────────────────────────────────
networks:
  frontend-net:
    driver: bridge               # Default (isolated bridge network)
  backend-net:
    driver: bridge
    internal: true               # No external internet access
  shared-net:
    external: true               # Network created outside this file
  overlay-net:
    driver: overlay              # For Docker Swarm multi-host
    attachable: true

# ─── SECRETS (Docker Swarm / Compose v3) ────────────────────
secrets:
  db_password:
    file: ./secrets/db_password.txt
  api_key:
    external: true               # Already created in Docker secret store
```

---

## 3. Docker Compose Commands

```bash
# ── Start & Stop ─────────────────────────────────────────────
docker compose up                    # Start all services (foreground)
docker compose up -d                 # Start all (detached/background)
docker compose up api db             # Start only specific services
docker compose up --build            # Rebuild images before starting
docker compose up --force-recreate   # Recreate containers even if unchanged
docker compose up --scale api=3      # Start 3 instances of api

docker compose down                  # Stop and remove containers + networks
docker compose down -v               # Also remove named volumes (DATA LOSS!)
docker compose down --rmi all        # Also remove built images
docker compose stop                  # Stop containers (don't remove)
docker compose start                 # Start stopped containers
docker compose restart api           # Restart specific service

# ── Logs ─────────────────────────────────────────────────────
docker compose logs                  # All service logs
docker compose logs -f               # Follow all logs
docker compose logs -f api           # Follow specific service
docker compose logs --tail=100 api   # Last 100 lines
docker compose logs --since=1h api   # Logs from last 1 hour

# ── Status & Inspection ───────────────────────────────────────
docker compose ps                    # List containers + status
docker compose ps -a                 # Include stopped containers
docker compose top                   # Show processes in containers
docker compose port api 8000         # Show host port for container port

# ── Execute commands ─────────────────────────────────────────
docker compose exec api bash         # Shell into running container
docker compose exec api python manage.py migrate  # Run command
docker compose exec db psql -U postgres mydb      # DB shell
docker compose run --rm api pytest   # One-off command in new container

# ── Build ────────────────────────────────────────────────────
docker compose build                 # Build all images
docker compose build api             # Build specific service
docker compose build --no-cache api  # Build without cache
docker compose pull                  # Pull latest images

# ── Config & Validate ────────────────────────────────────────
docker compose config                # Validate + show merged config
docker compose config --services     # List service names only

# ── Multiple compose files (override pattern) ─────────────────
docker compose -f docker-compose.yml -f docker-compose.override.yml up
docker compose -f docker-compose.yml -f docker-compose.prod.yml up
```

---

## 4. Docker Compose — Full Tech Stack Setups

### Python FastAPI + PostgreSQL + Redis + Celery

```yaml
name: fastapi-app

services:

  api:
    build:
      context: .
      target: runtime
    container_name: fastapi-api
    ports:
      - "127.0.0.1:8000:8000"
    environment:
      DATABASE_URL: postgresql://postgres:password@db:5432/appdb
      REDIS_URL: redis://redis:6379/0
      SECRET_KEY: ${SECRET_KEY}               # From .env file
      DEBUG: "false"
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    volumes:
      - ./uploads:/app/uploads               # Persistent uploads
    networks:
      - app-net
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 20s
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    deploy:
      resources:
        limits:
          memory: 512M

  worker:                                    # Celery background worker
    build:
      context: .
      target: runtime
    command: celery -A tasks worker --loglevel=info --concurrency=4
    environment:
      DATABASE_URL: postgresql://postgres:password@db:5432/appdb
      REDIS_URL: redis://redis:6379/0
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - app-net
    restart: unless-stopped

  beat:                                      # Celery scheduler
    build:
      context: .
      target: runtime
    command: celery -A tasks beat --loglevel=info
    environment:
      REDIS_URL: redis://redis:6379/0
    depends_on:
      - redis
    networks:
      - app-net
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    container_name: fastapi-db
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password            # Use secret in prod!
      POSTGRES_DB: appdb
    volumes:
      - db-data:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql  # Run on first start
    networks:
      - app-net
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s
    deploy:
      resources:
        limits:
          memory: 1G

  redis:
    image: redis:7-alpine
    container_name: fastapi-redis
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis-data:/data
    networks:
      - app-net
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3

volumes:
  db-data:
  redis-data:

networks:
  app-net:
    driver: bridge
```

---

### Node.js + MongoDB + Nginx Reverse Proxy

```yaml
name: node-mongo-app

services:

  nginx:
    image: nginx:1.25-alpine
    container_name: nginx-proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - ./nginx/certs:/etc/nginx/certs:ro    # TLS certificates
    depends_on:
      - api
    networks:
      - frontend-net
    restart: unless-stopped

  api:
    build:
      context: ./backend
      target: runtime
    container_name: node-api
    # No ports exposed to host — nginx proxies internally
    environment:
      NODE_ENV: production
      MONGO_URI: mongodb://mongo:27017/mydb
      JWT_SECRET: ${JWT_SECRET}
      PORT: "3000"
    depends_on:
      mongo:
        condition: service_healthy
    networks:
      - frontend-net
      - backend-net
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  mongo:
    image: mongo:7
    container_name: mongodb
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_PASSWORD}
      MONGO_INITDB_DATABASE: mydb
    volumes:
      - mongo-data:/data/db
      - ./mongo/init.js:/docker-entrypoint-initdb.d/init.js:ro
    networks:
      - backend-net                          # Not exposed to frontend
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  mongo-data:

networks:
  frontend-net:
    driver: bridge
  backend-net:
    driver: bridge
    internal: true                           # No internet access
```

---

### Full Stack: React + Django + PostgreSQL + Celery + Flower

```yaml
name: fullstack-app

services:

  frontend:
    build:
      context: ./frontend
      target: runtime
    container_name: react-app
    ports:
      - "3000:80"
    depends_on:
      - backend
    networks:
      - web-net
    restart: unless-stopped

  backend:
    build:
      context: ./backend
      target: runtime
    container_name: django-api
    command: >
      sh -c "python manage.py migrate &&
             python manage.py collectstatic --noinput &&
             gunicorn myproject.wsgi:application --bind 0.0.0.0:8000 --workers 4"
    ports:
      - "8000:8000"
    environment:
      DJANGO_SETTINGS_MODULE: myproject.settings.production
      DATABASE_URL: postgres://postgres:${DB_PASSWORD}@db:5432/mydb
      CELERY_BROKER_URL: redis://redis:6379/0
      DJANGO_SECRET_KEY: ${DJANGO_SECRET_KEY}
      ALLOWED_HOSTS: "localhost,api.example.com"
    env_file:
      - .env
    volumes:
      - static-files:/app/staticfiles
      - media-files:/app/media
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - web-net
      - db-net
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/api/health/"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  celery:
    build:
      context: ./backend
      target: runtime
    command: celery -A myproject worker -l info -Q default,celery
    environment:
      DATABASE_URL: postgres://postgres:${DB_PASSWORD}@db:5432/mydb
      CELERY_BROKER_URL: redis://redis:6379/0
    env_file:
      - .env
    volumes:
      - media-files:/app/media
    depends_on:
      - redis
      - db
    networks:
      - db-net
    restart: unless-stopped

  flower:                                    # Celery monitoring UI
    image: mher/flower:2.0
    command: celery flower --broker=redis://redis:6379/0 --port=5555
    ports:
      - "127.0.0.1:5555:5555"              # Localhost only
    depends_on:
      - redis
    networks:
      - db-net
    restart: unless-stopped
    profiles:
      - monitoring                          # Only with --profile monitoring

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: mydb
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - db-net
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis-data:/data
    networks:
      - db-net
    restart: unless-stopped

volumes:
  postgres-data:
  redis-data:
  static-files:
  media-files:

networks:
  web-net:
    driver: bridge
  db-net:
    driver: bridge
    internal: true
```

---

## 5. Dev vs Prod Compose Override Pattern

```
project/
├── docker-compose.yml          # Base (shared config)
├── docker-compose.dev.yml      # Dev overrides
├── docker-compose.prod.yml     # Prod overrides
└── .env                        # Environment variables
```

```yaml
# docker-compose.yml (base)
services:
  api:
    build: .
    environment:
      DB_HOST: db
    depends_on:
      db:
        condition: service_healthy
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: password
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      retries: 5
volumes:
  db-data:
```

```yaml
# docker-compose.dev.yml (dev overrides)
services:
  api:
    build:
      target: development         # Use dev stage of multi-stage Dockerfile
    command: uvicorn main:app --reload --host 0.0.0.0 --port 8000
    volumes:
      - .:/app                    # Live reload: mount entire source code
    ports:
      - "8000:8000"
    environment:
      DEBUG: "true"
      LOG_LEVEL: debug

  db:
    ports:
      - "5432:5432"               # Expose DB port for local tools (TablePlus, DBeaver)

  # Dev-only: PgAdmin web UI
  pgadmin:
    image: dpage/pgadmin4:8
    ports:
      - "5050:80"
    environment:
      PGADMIN_DEFAULT_EMAIL: dev@example.com
      PGADMIN_DEFAULT_PASSWORD: admin
    volumes:
      - pgadmin-data:/var/lib/pgadmin

volumes:
  pgadmin-data:
```

```yaml
# docker-compose.prod.yml (prod overrides)
services:
  api:
    image: myacr.azurecr.io/myapp:${IMAGE_TAG}  # Use pre-built image
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 512M
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "5"

  db:
    restart: unless-stopped
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}    # From secure .env
    deploy:
      resources:
        limits:
          memory: 1G
```

```bash
# Run dev
docker compose -f docker-compose.yml -f docker-compose.dev.yml up

# Run prod
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Shortcut: docker-compose.override.yml is auto-loaded
# Rename docker-compose.dev.yml → docker-compose.override.yml
# Then: docker compose up (auto-merges override file)
```

---

## 6. Common Scenarios & Gotchas

### Run DB Migration Before App Starts

```yaml
services:
  migrate:
    build: .
    command: python manage.py migrate
    depends_on:
      db:
        condition: service_healthy
    restart: "no"                          # Run once and exit

  api:
    build: .
    depends_on:
      migrate:
        condition: service_completed_successfully
      db:
        condition: service_healthy
```

### Wait for Service with Custom Script

```bash
#!/bin/sh
# wait-for.sh — wait for host:port to be ready
set -e
host="$1"; shift; port="$1"; shift
until nc -z "$host" "$port"; do
  echo "Waiting for $host:$port..."
  sleep 2
done
exec "$@"
```

```yaml
command: ["./wait-for.sh", "db", "5432", "--", "python", "app.py"]
```

### Environment Variable Best Practices

```bash
# .env file (never commit to git!)
POSTGRES_PASSWORD=mysecretpassword
JWT_SECRET=myverylongrandomsecretkey
IMAGE_TAG=sha-abc1234
```

```yaml
# Reference in compose:
environment:
  DB_PASS: ${POSTGRES_PASSWORD}        # From .env
  PORT: ${PORT:-8000}                  # With default value if not set
```

```bash
# .env.example (commit this — no real values)
POSTGRES_PASSWORD=changeme
JWT_SECRET=changeme
IMAGE_TAG=latest
```

> **Gotcha:** `depends_on` with just service name only waits for container to START, not be READY. Always use `condition: service_healthy` with a `healthcheck` defined on the dependency — otherwise your app tries to connect to DB before Postgres is ready to accept connections.

> **Gotcha:** Named volumes persist across `docker compose down`. To reset completely: `docker compose down -v`. Never run this in production without a backup.

> **Gotcha:** `restart: always` restarts containers even when you manually `docker compose stop`. Use `restart: unless-stopped` instead — it respects manual stops.

> **Gotcha:** Services in the same compose file can talk to each other using the **service name** as hostname. If your api service needs to connect to the db service, use `DB_HOST=db` not `DB_HOST=localhost`.

> **Gotcha:** Port binding `"8000:8000"` exposes to ALL interfaces (0.0.0.0). Use `"127.0.0.1:8000:8000"` to expose only on localhost — critical for security when the host is internet-facing.

---

## 7. Dockerfile Best Practices Summary

```
┌──────────────────────────────────────────────────────────────┐
│  DOCKERFILE CHECKLIST                                        │
│                                                              │
│  Security:                                                   │
│  [ ] Run as non-root user (USER appuser)                    │
│  [ ] No secrets in ENV or ARG (use runtime env vars)        │
│  [ ] Use specific image tags, never :latest in prod         │
│  [ ] Pin digest for critical images (@sha256:...)           │
│  [ ] Use distroless or slim base images                     │
│  [ ] Add HEALTHCHECK instruction                            │
│  [ ] Drop Linux capabilities (cap_drop: [ALL])              │
│                                                              │
│  Size & Performance:                                         │
│  [ ] Use multi-stage builds                                 │
│  [ ] Chain RUN commands with && to reduce layers            │
│  [ ] Clean apt/yum cache in same RUN layer                  │
│  [ ] Use .dockerignore to exclude unnecessary files         │
│  [ ] Copy requirements file BEFORE source code              │
│  [ ] Use --no-install-recommends for apt                    │
│                                                              │
│  Correctness:                                                │
│  [ ] Use ENTRYPOINT exec form (not shell form)              │
│  [ ] Set WORKDIR instead of cd in RUN                       │
│  [ ] Set PYTHONUNBUFFERED=1 for Python apps                 │
│  [ ] Set PYTHONDONTWRITEBYTECODE=1 for Python apps          │
│  [ ] Use npm ci instead of npm install                      │
│  [ ] Add LABEL with version/maintainer info                 │
└──────────────────────────────────────────────────────────────┘
```

---

## 8. Quick Reference — Image Size Comparison

| Base Image | Compressed Size | Use Case |
|---|---|---|
| `ubuntu:22.04` | ~30MB | Full OS, complex setups |
| `debian:bookworm-slim` | ~25MB | Debian without extras |
| `python:3.11` | ~350MB | Python (full) |
| `python:3.11-slim` | ~45MB | Python (recommended for most) |
| `python:3.11-alpine` | ~20MB | Python (small but glibc issues) |
| `node:20` | ~350MB | Node.js (full) |
| `node:20-slim` | ~65MB | Node.js (recommended) |
| `node:20-alpine` | ~40MB | Node.js (small) |
| `golang:1.22-alpine` | ~250MB | Go builder (not runtime) |
| `gcr.io/distroless/static` | ~2MB | Go/static binary runtime |
| `scratch` | 0MB | Statically compiled binaries |
| `nginx:1.25-alpine` | ~10MB | Nginx web server |
| `postgres:16-alpine` | ~85MB | PostgreSQL database |
| `redis:7-alpine` | ~15MB | Redis cache |

*← [Part 1: Dockerfile instructions and tech stack Dockerfiles](./docker_notes_part1.md)*
