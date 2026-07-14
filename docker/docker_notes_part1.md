# 🐳 Dockerfile & Docker Compose Notes — Part 1
> Location: trainee-docs/docker/
> Covers: Dockerfile fundamentals, every instruction explained, multi-stage builds, tech stack Dockerfiles

---

## 1. What is a Dockerfile?

```
┌──────────────────────────────────────────────────────────────┐
│  DOCKERFILE = Recipe to build a Docker image                 │
│                                                              │
│  Dockerfile                                                  │
│       │ docker build -t myapp:v1 .                          │
│       ▼                                                      │
│  Docker Image (immutable snapshot)                           │
│       │ docker run myapp:v1                                  │
│       ▼                                                      │
│  Docker Container (running instance of the image)            │
│                                                              │
│  Analogy:                                                    │
│  Dockerfile = blueprint of a house                           │
│  Image      = the house built from the blueprint             │
│  Container  = a person living in the house                   │
│                                                              │
│  Key Properties:                                             │
│  ├── Reproducible: same Dockerfile = same image everywhere  │
│  ├── Layered: each instruction adds a layer (cached)        │
│  ├── Immutable: images never change once built              │
│  └── Portable: runs on any machine with Docker              │
└──────────────────────────────────────────────────────────────┘
```

---

## 2. Docker Image Layers — How Caching Works

```
┌──────────────────────────────────────────────────────────────┐
│  IMAGE LAYERS — MENTAL MODEL                                 │
│                                                              │
│  Each instruction in Dockerfile = one layer                  │
│                                                              │
│  FROM ubuntu:22.04          → Layer 1 (base OS)             │
│  RUN apt-get update         → Layer 2 (package index)       │
│  RUN apt-get install nginx  → Layer 3 (nginx binary)        │
│  COPY nginx.conf /etc/      → Layer 4 (your config)         │
│  COPY ./html /var/www/      → Layer 5 (your code)           │
│                                                              │
│  Layers are CACHED. If a layer hasn't changed:              │
│  → Docker reuses the cached layer (no re-execution)         │
│                                                              │
│  CACHE INVALIDATION RULE:                                    │
│  If any layer changes → ALL layers BELOW it are re-run      │
│                                                              │
│  Bad order (slow):          Good order (fast):              │
│  COPY . /app         ←──── changes every build              │
│  RUN pip install ... ←──── re-runs every build!             │
│                                                              │
│  COPY requirements.txt /app/  ←── rarely changes            │
│  RUN pip install ...          ←── cached when deps unchanged │
│  COPY . /app                  ←── changes every build (OK)  │
│                                                              │
│  RULE: Put what changes RARELY at the TOP.                  │
│        Put what changes OFTEN at the BOTTOM.                │
└──────────────────────────────────────────────────────────────┘
```

---

## 3. Every Dockerfile Instruction Explained

### FROM — Base Image

```dockerfile
# Always the first instruction
FROM ubuntu:22.04

# Use a specific digest (most reproducible — recommended for prod)
FROM python:3.11-slim@sha256:abc123...

# Use build arg to parameterize base image
ARG BASE_IMAGE=python:3.11-slim
FROM ${BASE_IMAGE}

# Multi-stage: name each stage
FROM node:20-alpine AS builder
FROM nginx:1.25-alpine AS runtime

# When to use which base:
# ubuntu/debian  → full OS, large (~100-200MB), good for complex setups
# alpine         → tiny (~5MB), missing glibc (breaks some apps)
# slim           → debian-based, no extras (~50MB), best for Python/Node
# distroless     → no shell, no package manager, most secure, hard to debug
# scratch        → empty image, for statically compiled binaries (Go)
```

### RUN — Execute Commands During Build

```dockerfile
# Single command
RUN apt-get update

# BEST PRACTICE: Chain commands with && to reduce layers
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
       curl \
       git \
       ca-certificates \
    && rm -rf /var/lib/apt/lists/*
#              ^^^^^^^^^^^^^^^^^^^
#   ALWAYS clean up apt cache — otherwise image stays bloated

# Why --no-install-recommends?
# Skips "recommended" packages that you don't need → smaller image

# RUN with shell form vs exec form
RUN echo "hello"                    # shell form: runs in /bin/sh -c
RUN ["echo", "hello"]              # exec form: runs directly, no shell

# Run as specific user (security)
RUN useradd -m appuser
RUN su appuser -c "pip install -r requirements.txt"
# OR use USER instruction before RUN (cleaner)

# Conditional in RUN
RUN if [ "$BUILD_ENV" = "prod" ]; then \
        pip install gunicorn; \
    else \
        pip install flask-debug; \
    fi
```

### COPY vs ADD

```dockerfile
# COPY — preferred, simple and explicit
COPY src/           /app/src/        # Copy directory
COPY requirements.txt /app/          # Copy single file
COPY *.py           /app/            # Copy with glob

# COPY with ownership (Docker 17.09+)
COPY --chown=appuser:appgroup requirements.txt /app/

# COPY from another build stage (multi-stage)
COPY --from=builder /app/dist /usr/share/nginx/html

# ADD — has extra features but avoid unless you need them
ADD archive.tar.gz /app/             # Auto-extracts tar archives
ADD https://example.com/file /tmp/   # Download from URL (don't use — use curl in RUN)

# RULE: Always use COPY unless you specifically need ADD's tar extraction
# ADD from URL is a bad practice — no checksum verification

# What NOT to copy (use .dockerignore):
# - node_modules/    (copy package.json, then npm install inside)
# - .git/            (not needed in image)
# - .env files       (never bake secrets into image)
# - test files       (not needed in prod image)
# - *.log
# - docs/
```

### .dockerignore — What NOT to Send to Docker Daemon

```dockerignore
# .dockerignore (same syntax as .gitignore)
# Reduces build context size = faster builds

.git
.gitignore
.github
node_modules
__pycache__
*.pyc
*.pyo
.venv
venv
*.egg-info
dist
build
.env
.env.*
*.log
logs
.DS_Store
docs
README.md
tests
Makefile
docker-compose*.yml
*.md
coverage/
.coverage
.pytest_cache
.mypy_cache
```

### WORKDIR — Set Working Directory

```dockerfile
# Sets the working directory for subsequent instructions
WORKDIR /app

# NEVER use: RUN cd /app && ...
# cd doesn't persist between RUN instructions!
# Use WORKDIR instead — it creates the directory if not exists
# and persists for all following instructions + CMD/ENTRYPOINT

# Convention: /app is standard for apps
# /usr/local/bin for binaries
# /etc/<service> for config files

WORKDIR /app          # All following instructions run in /app
COPY . .              # Copies to /app
RUN npm install       # Runs in /app
```

### ENV — Environment Variables

```dockerfile
# Set at build time AND available at runtime
ENV NODE_ENV=production
ENV PORT=3000
ENV APP_HOME=/app

# Multiple in one instruction
ENV NODE_ENV=production \
    PORT=3000 \
    LOG_LEVEL=info

# Reference earlier ENV
ENV APP_HOME=/app
WORKDIR ${APP_HOME}

# RUNTIME override: docker run -e PORT=8080 myapp

# DIFFERENCE: ENV vs ARG
# ENV → available at BUILD time and RUNTIME
# ARG → available ONLY at BUILD time (not in final container)
```

### ARG — Build-Time Variables

```dockerfile
# Declare build argument with optional default
ARG NODE_VERSION=20
ARG BUILD_ENV=production
ARG APP_VERSION

# Use in FROM (the only place ARG works before FROM)
ARG BASE_TAG=3.11-slim
FROM python:${BASE_TAG}

# Pass at build time:
# docker build --build-arg NODE_VERSION=18 --build-arg BUILD_ENV=dev .

# GOTCHA: ARG values are visible in docker history (not secure for secrets)
# Never use ARG for secrets — use Docker secrets or runtime env vars

# Reset ARG scope after FROM
ARG BUILD_ENV           # Must re-declare after FROM to use it
RUN echo "Building for $BUILD_ENV"
```

### EXPOSE — Document Port (Does NOT Publish It)

```dockerfile
# EXPOSE tells Docker which port the app listens on
# It's DOCUMENTATION — it doesn't actually publish the port
EXPOSE 8080
EXPOSE 443

# To actually publish: docker run -p 8080:8080 myapp
#                  or: docker run -P myapp (publishes all EXPOSED ports)

# EXPOSE with protocol
EXPOSE 53/udp
EXPOSE 8080/tcp    # tcp is default

# RULE: Always EXPOSE the port your app listens on — it's self-documenting
# and some tools (Kubernetes, docker-compose) use it for configuration
```

### CMD and ENTRYPOINT

```dockerfile
# CMD — default command (overridable at runtime)
CMD ["python", "app.py"]                 # exec form (PREFERRED)
CMD python app.py                        # shell form (avoid in prod)
CMD ["--port", "8080"]                   # args only (used with ENTRYPOINT)

# ENTRYPOINT — fixed command (not overridden easily)
ENTRYPOINT ["python", "app.py"]          # exec form
ENTRYPOINT python app.py                 # shell form (avoid)

# COMBINATION (most flexible pattern):
ENTRYPOINT ["python", "app.py"]          # fixed executable
CMD ["--port", "8080", "--workers", "4"] # default args (overridable)
# docker run myapp --port=9090           # overrides CMD, keeps ENTRYPOINT

# OVERRIDE at runtime:
# CMD:        docker run myapp python other_script.py
# ENTRYPOINT: docker run --entrypoint python myapp other_script.py

# SHELL FORM GOTCHA:
# ENTRYPOINT python app.py → runs as /bin/sh -c "python app.py"
# docker stop sends SIGTERM to /bin/sh (PID 1), NOT python!
# Python won't get signal → container takes 10 seconds to stop
# ALWAYS use exec form for ENTRYPOINT

# PID 1 PROBLEM:
# In containers, your process should be PID 1 to receive signals
# Exec form: ["python", "app.py"] → python IS PID 1 ✅
# Shell form: python app.py → /bin/sh is PID 1, python is a child ❌

# SOLUTION for init process (zombie reaping + signal forwarding):
FROM ubuntu:22.04
RUN apt-get install -y tini
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["python", "app.py"]
```

### USER — Run as Non-Root

```dockerfile
# Create a non-root user
RUN groupadd -r appgroup && useradd -r -g appgroup appuser

# Switch to that user for RUN, CMD, ENTRYPOINT
USER appuser

# Or use numeric UID (more portable)
USER 1001

# GOTCHA: If you need to write to directories, change ownership BEFORE USER
RUN chown -R appuser:appgroup /app /var/log/myapp
USER appuser

# In Alpine Linux:
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# WHY NON-ROOT?
# Running as root in container = root on host (if container escapes)
# Non-root = defense in depth
# Many K8s clusters REJECT root containers via policy
```

### VOLUME — Declare Mount Points

```dockerfile
# Declare that this path should be persisted outside the container
VOLUME ["/data"]
VOLUME /var/log/myapp

# VOLUMES declared here are created as anonymous volumes if
# the user doesn't explicitly mount anything
# docker run myapp → /data is auto-created as anonymous volume

# PREFER: define volumes in docker-compose.yml instead
# Dockerfile VOLUME is documentation + fallback

# Use VOLUME for:
# - Database data directories
# - Log directories
# - Upload directories
```

### HEALTHCHECK

```dockerfile
# Tell Docker how to check if container is healthy
HEALTHCHECK --interval=30s \
            --timeout=10s \
            --start-period=20s \
            --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

# Parameters:
# --interval:     How often to check (default 30s)
# --timeout:      How long before check is considered failed (default 30s)
# --start-period: Ignore failures during startup (default 0s)
# --retries:      Failures needed before unhealthy (default 3)

# Exit codes:
# 0 = healthy
# 1 = unhealthy
# 2 = reserved (don't use)

# For apps without curl (minimal images):
HEALTHCHECK CMD wget -q -O /dev/null http://localhost:8080/health || exit 1

# Disable inherited healthcheck:
HEALTHCHECK NONE
```

### LABEL — Metadata

```dockerfile
# Add metadata to image
LABEL org.opencontainers.image.title="My App" \
      org.opencontainers.image.description="Production API" \
      org.opencontainers.image.version="1.2.3" \
      org.opencontainers.image.authors="team@company.com" \
      org.opencontainers.image.source="https://github.com/myorg/myapp" \
      org.opencontainers.image.created="2025-07-14" \
      maintainer="platform-team@company.com"

# View labels: docker inspect myapp:v1 | jq '.[].Config.Labels'
```

---

## 4. Multi-Stage Builds — The Most Important Pattern

```
┌──────────────────────────────────────────────────────────────┐
│  MULTI-STAGE BUILD                                           │
│                                                              │
│  Problem: Build tools (compilers, test runners) are large   │
│  but not needed in production image.                        │
│                                                              │
│  Single stage (bad):                                        │
│  FROM node:20       (1GB image)                             │
│  + node_modules     (500MB)                                 │
│  + source code      (10MB)                                  │
│  = 1.5GB prod image (contains compilers, npm, test tools)   │
│                                                              │
│  Multi-stage (good):                                        │
│  Stage 1 (builder): Install deps, compile, run tests        │
│       → only the OUTPUT (dist/) is kept                     │
│  Stage 2 (runtime): Minimal base + only the compiled output │
│  = 50MB prod image (no compiler, no npm, no test tools)    │
└──────────────────────────────────────────────────────────────┘
```

```dockerfile
# ── Multi-stage: Node.js ─────────────────────────────────────
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build      # Compile TypeScript, bundle, etc.
RUN npm test           # Run tests during build

FROM node:20-alpine AS runtime
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY --from=deps /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY package.json ./
USER appuser
EXPOSE 3000
HEALTHCHECK CMD wget -qO- http://localhost:3000/health || exit 1
CMD ["node", "dist/server.js"]
```

---

## 5. Dockerfiles by Tech Stack

### Python (FastAPI / Django / Flask)

```dockerfile
# ── Python Production Dockerfile ─────────────────────────────
FROM python:3.11-slim AS base

# Prevents Python from writing pyc files
ENV PYTHONDONTWRITEBYTECODE=1
# Prevents Python from buffering stdout/stderr
ENV PYTHONUNBUFFERED=1

WORKDIR /app

# Install system dependencies (only what's needed)
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
       libpq-dev \
       gcc \
    && rm -rf /var/lib/apt/lists/*

# ── Builder stage (install deps with build tools) ─────────────
FROM base AS builder
COPY requirements.txt .
RUN pip install --upgrade pip \
    && pip install --no-cache-dir --user -r requirements.txt

# ── Runtime stage (minimal, no build tools) ──────────────────
FROM base AS runtime

# Create non-root user
RUN groupadd -r appgroup && useradd -r -g appgroup appuser

# Copy installed packages from builder
COPY --from=builder /root/.local /home/appuser/.local

# Set PATH to include user-installed packages
ENV PATH=/home/appuser/.local/bin:$PATH

# Copy app source
COPY --chown=appuser:appgroup . .

USER appuser

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=10s --start-period=15s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1

# FastAPI with uvicorn
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]

# Django with gunicorn
# CMD ["gunicorn", "myproject.wsgi:application", "--bind", "0.0.0.0:8000", "--workers", "4"]
```

### Node.js (Express / NestJS / Next.js)

```dockerfile
# ── Node.js Production Dockerfile ────────────────────────────
FROM node:20-alpine AS deps
WORKDIR /app

# Copy only lockfile and package.json for caching
COPY package.json package-lock.json ./

# npm ci = clean install from lockfile (more reproducible than npm install)
# --only=production = skip devDependencies for runtime
RUN npm ci --only=production

# ── Build stage ───────────────────────────────────────────────
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci                             # Install ALL deps (including dev)
COPY . .
RUN npm run build                      # Compile TypeScript / bundle

# ── Runtime stage ─────────────────────────────────────────────
FROM node:20-alpine AS runtime
WORKDIR /app

RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser

# Copy prod dependencies + compiled output
COPY --from=deps /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY package.json ./

USER appuser
EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD wget -qO- http://localhost:3000/health || exit 1

CMD ["node", "dist/server.js"]
```

```dockerfile
# ── Next.js Dockerfile (SSR/SSG) ─────────────────────────────
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Next.js collects anonymous telemetry — disable it
ENV NEXT_TELEMETRY_DISABLED=1
RUN npm run build

FROM node:20-alpine AS runtime
WORKDIR /app

ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1

RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser

# Next.js standalone output (set in next.config.js: output: 'standalone')
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public

USER appuser
EXPOSE 3000

CMD ["node", "server.js"]
```

### Go (Golang)

```dockerfile
# ── Go Dockerfile — smallest possible image ───────────────────
FROM golang:1.22-alpine AS builder
WORKDIR /app

# Download dependencies first (cache layer)
COPY go.mod go.sum ./
RUN go mod download
RUN go mod verify

# Build the binary (statically linked)
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-w -s" -o /app/server ./cmd/server

# -w: omit DWARF debug info (smaller binary)
# -s: omit symbol table (smaller binary)
# CGO_ENABLED=0: fully static binary (no glibc dependency)

# ── Distroless runtime (most secure — no shell, no package manager) ──
FROM gcr.io/distroless/static:nonroot AS runtime
WORKDIR /app

COPY --from=builder /app/server .

# distroless/static:nonroot already runs as nonroot user (65532)
EXPOSE 8080

ENTRYPOINT ["/app/server"]
```

```dockerfile
# ── Go with scratch (absolute minimum size) ───────────────────
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-w -s" -o server .

FROM scratch AS runtime
# Copy TLS certificates (needed for HTTPS calls from inside scratch)
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /app/server /server
EXPOSE 8080
ENTRYPOINT ["/server"]
# Result: ~10MB image with ZERO OS footprint
```

### Java (Spring Boot)

```dockerfile
# ── Java Spring Boot Dockerfile ───────────────────────────────
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app

# Maven build
COPY mvnw pom.xml ./
COPY .mvn .mvn
RUN ./mvnw dependency:go-offline -B    # Cache dependencies

COPY src ./src
RUN ./mvnw package -DskipTests         # Build JAR

# ── Extract layers (Spring Boot 2.3+ supports layered JARs) ──
FROM eclipse-temurin:21-jdk-alpine AS extractor
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

# ── Runtime (layered for better caching) ─────────────────────
FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app

RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Copy layers in order (least-changed first → better cache)
COPY --from=extractor /app/dependencies/ ./
COPY --from=extractor /app/spring-boot-loader/ ./
COPY --from=extractor /app/snapshot-dependencies/ ./
COPY --from=extractor /app/application/ ./

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
    CMD wget -qO- http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", \
    "-XX:+UseContainerSupport", \
    "-XX:MaxRAMPercentage=75.0", \
    "org.springframework.boot.loader.launch.JarLauncher"]
# -XX:+UseContainerSupport: Java respects container memory limits (Java 8u191+)
# -XX:MaxRAMPercentage=75.0: Use 75% of container memory for heap
```

### PHP (Laravel)

```dockerfile
# ── PHP Laravel Dockerfile ────────────────────────────────────
FROM php:8.3-fpm-alpine AS base

# Install PHP extensions
RUN apk add --no-cache \
        libpq-dev \
        libzip-dev \
        libpng-dev \
        oniguruma-dev \
    && docker-php-ext-install \
        pdo_pgsql \
        pdo_mysql \
        zip \
        gd \
        mbstring \
        opcache

# Install Composer
COPY --from=composer:2.7 /usr/bin/composer /usr/bin/composer

WORKDIR /var/www/html

# ── Builder stage ─────────────────────────────────────────────
FROM base AS builder
COPY composer.json composer.lock ./
RUN composer install \
    --no-dev \
    --no-scripts \
    --no-autoloader \
    --prefer-dist

COPY . .
RUN composer dump-autoload --optimize

# ── Runtime stage ─────────────────────────────────────────────
FROM base AS runtime
WORKDIR /var/www/html

COPY --from=builder /var/www/html .

# Configure opcache for production
RUN echo "opcache.enable=1\nopcache.memory_consumption=128\nopcache.max_accelerated_files=10000\nopcache.validate_timestamps=0" \
    >> /usr/local/etc/php/conf.d/opcache.ini

RUN chown -R www-data:www-data storage bootstrap/cache
USER www-data

EXPOSE 9000
CMD ["php-fpm"]
```

### Ruby on Rails

```dockerfile
FROM ruby:3.3-slim AS base
WORKDIR /app

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
       libpq-dev gcc make nodejs yarn \
    && rm -rf /var/lib/apt/lists/*

FROM base AS builder
COPY Gemfile Gemfile.lock ./
RUN bundle install --without development test --jobs 4

COPY . .
RUN SECRET_KEY_BASE=dummy bundle exec rails assets:precompile

FROM base AS runtime
WORKDIR /app

RUN groupadd -r railsuser && useradd -r -g railsuser railsuser

COPY --from=builder /usr/local/bundle /usr/local/bundle
COPY --from=builder /app .

RUN chown -R railsuser:railsuser /app
USER railsuser

EXPOSE 3000

HEALTHCHECK CMD curl -f http://localhost:3000/health || exit 1

CMD ["bundle", "exec", "puma", "-C", "config/puma.rb"]
```

### Nginx (Static Site)

```dockerfile
# ── Nginx serving static files ────────────────────────────────
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build       # Output to /app/dist or /app/build

FROM nginx:1.25-alpine AS runtime

# Remove default nginx config
RUN rm /etc/nginx/conf.d/default.conf

# Copy custom nginx config
COPY nginx.conf /etc/nginx/conf.d/app.conf

# Copy built files
COPY --from=builder /app/dist /usr/share/nginx/html

# Nginx runs as root by default but serves as nginx user
EXPOSE 80 443

HEALTHCHECK CMD wget -qO- http://localhost/health || exit 1

CMD ["nginx", "-g", "daemon off;"]
```

```nginx
# nginx.conf — for SPA (React/Vue/Angular)
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    # Gzip compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;

    # Cache static assets aggressively
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # SPA fallback: route all 404s to index.html (client-side routing)
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Health check endpoint
    location /health {
        return 200 "OK";
        add_header Content-Type text/plain;
    }
}
```

---

*Next → [Part 2: Docker Compose — complete reference, all scenarios, tech stacks](./docker_notes_part2.md)*
