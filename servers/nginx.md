[🏠 Home](../README.md) · [Servers](README.md)

# ⚡ Nginx — Advanced Configuration & Production Guide

> **Audience:** DevOps / Cloud Engineers & Trainees | **Focus:** Architecture, Production Config, Gotchas  
> **Covers:** Reverse Proxy, SSL/TLS, Static Hosting, Backend Scripting, Load Balancing, Hardening  
> **Tip:** Sections build on each other — read sequentially first, then use TOC for reference.

---

## Table of Contents

1. [Architecture & Mental Model](#1-architecture--mental-model)
2. [Installation & File Layout](#2-installation--file-layout)
3. [Core Configuration Anatomy](#3-core-configuration-anatomy)
4. [Static Site Hosting](#4-static-site-hosting)
5. [Reverse Proxy](#5-reverse-proxy)
6. [HTTPS & SSL/TLS Certificate Installation](#6-https--ssltls-certificate-installation)
7. [Server-Side Scripting — PHP, Python, Node.js](#7-server-side-scripting--php-python-nodejs)
8. [Load Balancing](#8-load-balancing)
9. [Caching & Performance Tuning](#9-caching--performance-tuning)
10. [Security Hardening](#10-security-hardening)
11. [Logging, Debugging & Monitoring](#11-logging-debugging--monitoring)
12. [Production Gotchas & Tricky Q&A](#12-production-gotchas--tricky-qa)

---

## 1. Architecture & Mental Model

### 1.1 What Is Nginx?

Nginx (pronounced "engine-x") is an **event-driven, non-blocking** web server & reverse proxy. Unlike Apache's process/thread-per-connection model, Nginx uses a small number of **worker processes** each handling **thousands of concurrent connections** via an event loop (epoll/kqueue).

### 1.2 Architecture Diagram

```
                         ┌──────────────────────────────────────────────┐
                         │              NGINX PROCESS MODEL             │
                         │                                              │
  Incoming               │  ┌────────────────────┐                      │
  Connections ──────────►│  │   Master Process   │  (PID 1 of nginx)    │
  (port 80/443)          │  │   • Reads config   │                      │
                         │  │   • Manages workers│                      │
                         │  │   • Binds ports    │                      │
                         │  └────────┬───────────┘                      │
                         │           │ fork()                           │
                         │  ┌────────┼────────────────────────┐         │
                         │  │        │                        │         │
                         │  ▼        ▼                        ▼         │
                         │ ┌──────┐ ┌──────┐      ┌──────────────┐      │
                         │ │Worker│ │Worker│ ...  │ Worker N     │      │
                         │ │  1   │ │  2   │      │ (=CPU cores) │      │
                         │ └──┬───┘ └──┬───┘      └──────┬───────┘      │
                         │    │        │                 │              │
                         │    │  Each worker runs an EVENT LOOP         │
                         │    │  handling thousands of connections      │
                         │    │  (non-blocking I/O via epoll/kqueue)    │
                         └────┼────────┼─────────────────┼──────────────┘
                              │        │                 │
                              ▼        ▼                 ▼
                         ┌─────────┐ ┌──────────┐ ┌──────────────┐
                         │ Static  │ │ Reverse  │ │ FastCGI /    │
                         │ Files   │ │ Proxy    │ │ uWSGI / gRPC │
                         └─────────┘ └──────────┘ └──────────────┘
```

### 1.3 Why Event-Driven Matters

```
  Apache (prefork/worker)              Nginx (event-driven)
  ┌──────────────────────┐             ┌───────────────────────┐
  │ 1 request = 1 thread │             │ 1 worker = thousands  │
  │                      │             │    of connections     │
  │ 10K connections =    │             │                       │
  │   10K threads        │             │ 10K connections =     │
  │   = ~10 GB RAM       │             │   2-4 workers         │
  │   = context switches │             │   = ~50 MB RAM        │
  │   = C10K problem ❌  │             │   = no thread overhead│
  │                      │             │   = C10K solved ✅    │
  └──────────────────────┘             └───────────────────────┘
```

> **DevOps Relevance:** Nginx's low memory footprint makes it the default choice for Kubernetes ingress controllers, API gateways, sidecar proxies, and CDN edge servers.

### 1.4 Nginx vs Apache — When to Use Which

| Aspect | Nginx | Apache |
|--------|-------|--------|
| **Concurrency model** | Event-driven (async) | Process/thread per connection |
| **Static content** | Extremely fast | Good, but more overhead |
| **Dynamic content** | Requires external backend (PHP-FPM) | Built-in via modules (mod_php) |
| **`.htaccess`** | Not supported ❌ | Supported ✅ (per-dir config) |
| **Config style** | Centralized `nginx.conf` | Distributed (.htaccess) |
| **Memory usage** | Very low | Higher under load |
| **Module loading** | Compiled-in (mostly) | Dynamic (load at runtime) |
| **Best for** | Reverse proxy, LB, static CDN | Shared hosting, .htaccess apps |

---

## 2. Installation & File Layout

### 2.1 Installation

```bash
# --- RHEL/CentOS/Rocky/Alma ---
sudo dnf install epel-release -y
sudo dnf install nginx -y
sudo systemctl enable --now nginx
sudo firewall-cmd --permanent --add-service=http --add-service=https
sudo firewall-cmd --reload

# --- Ubuntu/Debian ---
sudo apt update
sudo apt install nginx -y
sudo systemctl enable --now nginx
sudo ufw allow 'Nginx Full'

# --- Verify ---
nginx -v                    # Version
curl -I http://localhost    # Should return HTTP/1.1 200
```

### 2.2 File Layout — Mental Model

```
  /etc/nginx/                          ← Main config directory
  ├── nginx.conf                       ← Master config (entry point)
  ├── conf.d/                          ← Drop-in server blocks (*.conf auto-loaded)
  │   └── default.conf                 ← Default server block
  ├── sites-available/                 ← (Debian/Ubuntu) All vhost definitions
  ├── sites-enabled/                   ← (Debian/Ubuntu) Symlinks to active vhosts
  ├── mime.types                       ← MIME type mappings
  ├── fastcgi_params                   ← FastCGI default parameters
  ├── proxy_params                     ← (Debian) Default proxy headers
  └── snippets/                        ← (Debian) Reusable config snippets

  /var/log/nginx/
  ├── access.log                       ← Request log
  └── error.log                        ← Error log

  /usr/share/nginx/html/               ← Default document root
  /var/www/                            ← Common custom doc root location
  /run/nginx.pid                       ← Master process PID file
```

> **⚠️ Gotcha:** RHEL-family uses `conf.d/` only. Debian-family uses `sites-available/` + `sites-enabled/` symlink pattern. Don't mix them — pick one convention per distro.

### 2.3 Config Testing & Reload

```bash
nginx -t                     # Test config syntax (ALWAYS run before reload)
nginx -T                     # Test + dump full merged config
sudo systemctl reload nginx  # Graceful reload — zero-downtime ✅
sudo systemctl restart nginx # Hard restart — drops active connections ❌
```

> **Production Rule:** NEVER `restart` — always `reload`. Reload forks new workers with new config; old workers finish existing requests gracefully.

---

## 3. Core Configuration Anatomy

### 3.1 Config Structure — Block Hierarchy

```
  ┌─────────────────────────────────────────────────────────────────┐
  │  nginx.conf — Configuration Block Hierarchy                     │
  │                                                                 │
  │  main context (global)                                          │
  │  ├── worker_processes auto;                                     │
  │  ├── error_log /var/log/nginx/error.log;                        │
  │  │                                                              │
  │  ├── events {                   ← Connection handling           │
  │  │     worker_connections 1024;                                 │
  │  │   }                                                          │
  │  │                                                              │
  │  └── http {                     ← All HTTP config               │
  │        ├── include mime.types;                                  │
  │        ├── access_log ...;                                      │
  │        │                                                        │
  │        ├── server {             ← Virtual host / server block   │
  │        │     ├── listen 80;                                     │
  │        │     ├── server_name example.com;                       │
  │        │     │                                                  │
  │        │     ├── location / {   ← URI matching                  │
  │        │     │     root /var/www/html;                          │
  │        │     │   }                                              │
  │        │     └── location /api/ {                               │
  │        │           proxy_pass http://backend;                   │
  │        │       }                                                │
  │        │   }                                                    │
  │        │                                                        │
  │        └── server { ... }       ← Another virtual host          │
  │      }                                                          │
  └─────────────────────────────────────────────────────────────────┘
```

### 3.2 Directive Inheritance

Directives set at a higher context **cascade down** unless overridden:

```
  http {
      gzip on;                  ← applies to ALL server blocks

      server {
          gzip off;             ← overrides for THIS server only

          location /api {
              # inherits gzip off from parent server
          }
      }
  }
```

> **⚠️ Gotcha:** `add_header` directives do NOT inherit into child blocks if the child has its own `add_header`. This is a classic security bug — you add HSTS in `server {}` but a `location {}` block with its own `add_header` silently drops HSTS.

### 3.3 Location Block Matching — Priority Order

This is one of the most **misunderstood** parts of Nginx. Matching order:

```
  Priority (highest to lowest):
  ┌────┬─────────────────────────────┬──────────────────────────────┐
  │ #  │ Modifier                    │ Meaning                      │
  ├────┼─────────────────────────────┼──────────────────────────────┤
  │ 1  │ location = /exact           │ Exact match — stops search   │
  │ 2  │ location ^~ /prefix         │ Prefix + skip regex phase    │
  │ 3  │ location ~ \.php$           │ Case-sensitive regex         │
  │ 3  │ location ~* \.jpg$          │ Case-insensitive regex       │
  │ 4  │ location /prefix            │ Plain prefix (longest wins)  │
  └────┴─────────────────────────────┴──────────────────────────────┘

  Matching Algorithm:
  ┌─────────────────────────────────────────────────────────────┐
  │  1. Check all EXACT (=) matches → if found, STOP            │
  │  2. Check all PREFIX matches → remember longest one         │
  │     a. If longest prefix has ^~ → STOP, use it              │
  │  3. Check all REGEX matches (in config order) → first wins  │
  │  4. If no regex matched → use longest prefix from step 2    │
  └─────────────────────────────────────────────────────────────┘
```

**Example — what matches what?**

```nginx
server {
    location = / {              # Only "GET /" exactly
        return 200 "exact root";
    }
    location ^~ /static/ {      # /static/* — skips regex
        root /var/www;
    }
    location ~ \.php$ {         # Any URI ending in .php
        fastcgi_pass unix:/run/php-fpm.sock;
    }
    location / {                # Catch-all fallback
        try_files $uri $uri/ =404;
    }
}
```

| Request URI | Matches | Why |
|-------------|---------|-----|
| `/` | `= /` | Exact match — highest priority |
| `/index.html` | `location /` | Prefix `/` matches, no regex matches |
| `/static/logo.png` | `^~ /static/` | Prefix with `^~` — regex skipped |
| `/static/script.php` | `^~ /static/` | `^~` blocks regex `\.php$` from matching |
| `/app/test.php` | `~ \.php$` | Regex beats plain prefix `/` |

> **⚠️ Gotcha:** A `^~` prefix prevents regex from overriding it. Without `^~`, a request to `/static/evil.php` would match the `\.php$` regex and be sent to PHP-FPM — a **serious security hole** if `/static/` contains user uploads.

---

## 4. Static Site Hosting

### 4.1 Basic Static Site

```nginx
# /etc/nginx/conf.d/mysite.conf
server {
    listen       80;
    server_name  mysite.com www.mysite.com;

    root         /var/www/mysite;
    index        index.html index.htm;

    # Serve files, then directories, then 404
    location / {
        try_files $uri $uri/ =404;
    }

    # Cache static assets aggressively
    location ~* \.(css|js|jpg|jpeg|png|gif|ico|svg|woff2?)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # Deny hidden files
    location ~ /\. {
        deny all;
        return 404;
    }
}
```

### 4.2 `root` vs `alias` — Critical Difference

```
  root directive:
  ┌──────────────────────────────────────────────────────────────┐
  │  location /images/ {                                         │
  │      root /var/www;                                          │
  │  }                                                           │
  │                                                              │
  │  Request: /images/logo.png                                   │
  │  File:    /var/www/images/logo.png                           │
  │           ^^^^^^^^ ← root + full URI path                    │
  └──────────────────────────────────────────────────────────────┘

  alias directive:
  ┌──────────────────────────────────────────────────────────────┐
  │  location /images/ {                                         │
  │      alias /data/pics/;                                      │
  │  }                                                           │
  │                                                              │
  │  Request: /images/logo.png                                   │
  │  File:    /data/pics/logo.png                                │
  │           ^^^^^^^^^^ ← alias REPLACES the location prefix    │
  └──────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha:** When using `alias`, the trailing `/` on both `location` and `alias` MUST match. `location /images` (no slash) + `alias /data/pics/` = broken path concatenation. Always include or omit the trailing slash consistently.

### 4.3 SPA (Single Page Application) Hosting

For React, Vue, Angular apps that use client-side routing:

```nginx
server {
    listen 80;
    server_name app.example.com;
    root /var/www/spa;

    location / {
        try_files $uri $uri/ /index.html;
        # 1. Try exact file → 2. Try directory → 3. Fallback to index.html
        # This lets client-side router handle /about, /dashboard, etc.
    }

    # API calls go to backend
    location /api/ {
        proxy_pass http://localhost:3000;
    }
}
```

### 4.4 Directory Listing (autoindex)

```nginx
location /files/ {
    alias /data/shared/;
    autoindex on;               # Enable directory listing
    autoindex_exact_size off;   # Show human-readable sizes (KB, MB)
    autoindex_localtime on;     # Show local time instead of UTC
}
```

### 4.5 Custom Error Pages

```nginx
server {
    # ...
    error_page 404 /custom_404.html;
    error_page 500 502 503 504 /custom_50x.html;

    location = /custom_404.html {
        root /var/www/error_pages;
        internal;               # Only served on actual errors, not direct requests
    }
    location = /custom_50x.html {
        root /var/www/error_pages;
        internal;
    }
}
```

---

## 5. Reverse Proxy

### 5.1 What Is a Reverse Proxy?

```
  Forward Proxy                         Reverse Proxy
  (client-side)                         (server-side)

  ┌────────┐     ┌───────┐             ┌───────┐     ┌─────────┐
  │ Client ├────►│ Proxy ├────►Internet│ Nginx ├────►│ Backend │
  │        │     │       │             │(rev.  │     │ App     │
  │ (knows │     │(hides │             │proxy) │     │ Server  │
  │  proxy)│     │client)│             │       │     │         │
  └────────┘     └───────┘             └───┬───┘     └─────────┘
                                           │
                                           │   Client has NO idea
                                           │   about backend topology
                                           │
                                      ┌────▼───────────────────────┐
                                      │ Benefits:                  │
                                      │ • SSL termination          │
                                      │ • Load balancing           │
                                      │ • Caching                  │
                                      │ • Compression              │
                                      │ • Rate limiting            │
                                      │ • Single entry point       │
                                      │ • Backend isolation        │
                                      └────────────────────────────┘
```

### 5.2 Basic Reverse Proxy Configuration

```nginx
server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;  # Backend app

        # --- Essential proxy headers ---
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # --- Timeouts ---
        proxy_connect_timeout  60s;
        proxy_send_timeout     60s;
        proxy_read_timeout     60s;
    }
}
```

### 5.3 Proxy Headers — Why Each One Matters

```
  Client (1.2.3.4) ──► Nginx (10.0.0.1) ──► Backend (10.0.0.2:3000)

  Without proxy headers, backend sees:
  ┌─────────────────────────────────────┐
  │ Remote IP:  10.0.0.1  (Nginx!)      │  ← WRONG — lost client IP
  │ Host:       10.0.0.2:3000           │  ← WRONG — internal host
  │ Scheme:     http                    │  ← WRONG if client used HTTPS
  └─────────────────────────────────────┘

  With proxy headers, backend sees:
  ┌─────────────────────────────────────┐
  │ X-Real-IP:          1.2.3.4         │  ✅ Real client IP
  │ X-Forwarded-For:    1.2.3.4         │  ✅ Full proxy chain
  │ Host:               api.example.com │  ✅ Original Host header
  │ X-Forwarded-Proto:  https           │  ✅ Original scheme
  └─────────────────────────────────────┘
```

> **⚠️ Gotcha:** If your app generates URLs (redirects, OAuth callbacks, CORS origins), it MUST read `X-Forwarded-Proto` and `Host` to build correct URLs. Otherwise you get mixed-content warnings, infinite redirect loops, or broken OAuth.

### 5.4 WebSocket Proxying

```nginx
location /ws/ {
    proxy_pass http://127.0.0.1:8080;
    proxy_http_version 1.1;                    # WebSocket requires HTTP/1.1
    proxy_set_header Upgrade $http_upgrade;     # Connection upgrade header
    proxy_set_header Connection "upgrade";      # Switch from HTTP to WS
    proxy_set_header Host $host;

    proxy_read_timeout 86400s;                  # Keep WS alive for 24h
    proxy_send_timeout 86400s;
}
```

> **⚠️ Gotcha:** Default `proxy_read_timeout` is 60s. WebSocket connections idle for >60s get killed silently. Always increase for WS.

### 5.5 Reverse Proxy with Path Rewriting

```nginx
# Client requests /app/v2/users → Backend receives /users
location /app/v2/ {
    proxy_pass http://127.0.0.1:5000/;    # Trailing slash = strip location prefix
}

# Client requests /app/v2/users → Backend receives /app/v2/users
location /app/v2/ {
    proxy_pass http://127.0.0.1:5000;     # No trailing slash = pass full URI
}
```

> **⚠️ Gotcha:** The trailing `/` in `proxy_pass` URI is the **#1 source of reverse proxy bugs**. With `/`, Nginx strips the matching `location` prefix. Without `/`, it forwards the full original URI. One character changes everything.

### 5.6 Multi-Service Reverse Proxy (Microservices Gateway)

```
  ┌──────────────────────────────────────────────────────────┐
  │                    Nginx (Gateway)                       │
  │                    :443 (SSL termination)                │
  │                                                          │
  │  /api/users/*  ──────►  user-service:3001                │
  │  /api/orders/* ──────►  order-service:3002               │
  │  /api/auth/*   ──────►  auth-service:3003                │
  │  /             ──────►  frontend:8080  (SPA)             │
  └──────────────────────────────────────────────────────────┘
```

```nginx
server {
    listen 443 ssl;
    server_name app.example.com;

    ssl_certificate     /etc/ssl/certs/app.crt;
    ssl_certificate_key /etc/ssl/private/app.key;

    # Frontend SPA
    location / {
        proxy_pass http://frontend:8080;
    }

    # User microservice
    location /api/users/ {
        proxy_pass http://user-service:3001/;
    }

    # Order microservice
    location /api/orders/ {
        proxy_pass http://order-service:3002/;
    }

    # Auth microservice
    location /api/auth/ {
        proxy_pass http://auth-service:3003/;
    }
}
```

---

## 6. HTTPS & SSL/TLS Certificate Installation

### 6.1 TLS Handshake — Mental Model

```
  Client                                    Server (Nginx)
  ──────                                    ──────────────
    │                                            │
    │  1. ClientHello                            │
    │     (supported TLS versions, ciphers)      │
    ├───────────────────────────────────────────►│
    │                                            │
    │  2. ServerHello                            │
    │     (chosen TLS version, cipher)           │
    │     + Server Certificate (public key)      │
    │◄───────────────────────────────────────────┤
    │                                            │
    │  3. Client verifies cert chain             │
    │     (CA → Intermediate → Leaf)             │
    │                                            │
    │  4. Key Exchange (ECDHE)                   │
    │     Both derive shared session key         │
    ├──────────────────────────────────────────► │
    │◄───────────────────────────────────────────┤
    │                                            │
    │  5. Encrypted Application Data             │
    │◄══════════════════════════════════════════►│
    │     (symmetric encryption: AES-256-GCM)    │
    │                                            │
```

### 6.2 Let's Encrypt with Certbot (Free, Automated)

```bash
# --- Install Certbot ---
# RHEL/CentOS
sudo dnf install certbot python3-certbot-nginx -y

# Ubuntu/Debian
sudo apt install certbot python3-certbot-nginx -y

# --- Obtain + Auto-Configure ---
sudo certbot --nginx -d example.com -d www.example.com

# Certbot will:
# 1. Verify domain ownership (HTTP-01 challenge)
# 2. Obtain certificate from Let's Encrypt
# 3. Modify your nginx config to add SSL directives
# 4. Set up auto-redirect HTTP → HTTPS

# --- Auto-Renewal (cron/timer) ---
sudo certbot renew --dry-run               # Test renewal
sudo systemctl enable --now certbot-renew.timer   # Systemd timer

# --- Manual Renewal ---
sudo certbot renew
sudo systemctl reload nginx
```

### 6.3 Manual SSL Configuration (CA-Signed or Self-Signed)

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    # --- Certificate Files ---
    ssl_certificate     /etc/ssl/certs/example.com.crt;      # Full chain (leaf + intermediate)
    ssl_certificate_key /etc/ssl/private/example.com.key;     # Private key

    # --- Protocol & Cipher Hardening ---
    ssl_protocols TLSv1.2 TLSv1.3;                            # Disable TLS 1.0, 1.1
    ssl_prefer_server_ciphers on;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';

    # --- Session Optimization ---
    ssl_session_cache   shared:SSL:10m;     # 10 MB shared cache (~40K sessions)
    ssl_session_timeout 1d;                  # Sessions valid for 1 day
    ssl_session_tickets off;                 # Disable for forward secrecy

    # --- OCSP Stapling (performance + privacy) ---
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/ssl/certs/ca-chain.crt;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 5s;

    # --- HSTS (tell browsers to always use HTTPS) ---
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

    # --- Your site config ---
    root /var/www/example.com;
    index index.html;
}

# --- HTTP → HTTPS Redirect ---
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}
```

### 6.4 Certificate Chain — Understanding the Full Chain

```
  ┌──────────────────────────────┐
  │        Root CA               │  ← Pre-installed in browsers/OS
  │   (DigiCert, Let's Encrypt)  │     Trust anchor
  └──────────┬───────────────────┘
             │ signs
  ┌──────────▼───────────────────┐
  │    Intermediate CA           │  ← Bridges root to your cert
  │                              │     Must be included in chain
  └──────────┬───────────────────┘
             │ signs
  ┌──────────▼───────────────────┐
  │    Your Leaf Certificate     │  ← Your domain certificate
  │    (example.com)             │
  └──────────────────────────────┘

  ssl_certificate must contain:  Leaf + Intermediate(s)
  ssl_certificate_key:           Your private key ONLY
```

```bash
# --- Build full chain file ---
cat example.com.crt intermediate.crt > fullchain.crt

# --- Verify chain ---
openssl verify -CAfile ca-chain.crt fullchain.crt

# --- Check cert details ---
openssl x509 -in fullchain.crt -text -noout | head -20

# --- Test from client side ---
openssl s_client -connect example.com:443 -servername example.com
```

> **⚠️ Gotcha:** If you forget the intermediate cert, browsers will show "Your connection is not private." Desktop Chrome might still work (it can fetch intermediates), but **mobile browsers and API clients will fail**. Always serve the full chain.

### 6.5 Self-Signed Certificate (Dev/Testing)

```bash
# Generate self-signed cert (valid 365 days)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/selfsigned.key \
  -out /etc/ssl/certs/selfsigned.crt \
  -subj "/C=US/ST=State/L=City/O=Dev/CN=localhost"

# Generate DH params for extra security
openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
```

> **Warning:** Self-signed certs trigger browser warnings and break API clients by default. Use only for local dev. For staging, use Let's Encrypt staging environment.

---

## 7. Server-Side Scripting — PHP, Python, Node.js

### 7.1 Architecture — How Nginx Talks to Backends

Nginx does NOT execute code directly. It delegates to external processes:

```
  ┌──────────────────────────────────────────────────────────────┐
  │                   Nginx + Backend Architecture               │
  │                                                              │
  │  ┌───────┐                                                   │
  │  │ Nginx │                                                   │
  │  │       ├──── FastCGI ────► PHP-FPM  (php scripts)          │
  │  │       ├──── uWSGI ──────► uWSGI    (Python/Django/Flask)  │
  │  │       ├──── proxy_pass ─► Node.js  (Express/Koa/Fastify)  │
  │  │       ├──── proxy_pass ─► Gunicorn (Python WSGI)          │
  │  │       ├──── grpc_pass ──► gRPC     (any gRPC service)     │
  │  │       └──── proxy_pass ─► Any HTTP app on any port        │
  │  └───────┘                                                   │
  │                                                              │
  │  Key Insight: Nginx = smart router/load-balancer in front    │
  │  of application processes that do the actual computation.    │
  └──────────────────────────────────────────────────────────────┘
```

### 7.2 PHP with PHP-FPM

```bash
# Install PHP-FPM
# RHEL: sudo dnf install php-fpm php-mysqlnd php-json php-mbstring -y
# Debian: sudo apt install php-fpm php-mysql php-json php-mbstring -y

# Start PHP-FPM
sudo systemctl enable --now php-fpm    # or php8.2-fpm on Debian
```

```nginx
server {
    listen 80;
    server_name phpapp.example.com;
    root /var/www/phpapp;
    index index.php index.html;

    # Try static files first, then PHP
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    # Pass PHP files to PHP-FPM
    location ~ \.php$ {
        # Security: ensure file exists before passing to FPM
        try_files $uri =404;

        fastcgi_pass unix:/run/php-fpm/www.sock;     # RHEL
        # fastcgi_pass unix:/run/php/php8.2-fpm.sock; # Debian

        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;

        # Timeouts for slow scripts
        fastcgi_read_timeout 300s;
    }

    # Block access to .php files in uploads directory
    location ~* /uploads/.*\.php$ {
        deny all;
    }
}
```

> **⚠️ Gotcha:** Without `try_files $uri =404;` in the PHP location, an attacker can upload `evil.jpg` containing PHP code and request `/uploads/evil.jpg/nonexistent.php`. PHP-FPM's `cgi.fix_pathinfo=1` will execute `evil.jpg` as PHP. This is the **path traversal / cgi.fix_pathinfo attack**. Set `cgi.fix_pathinfo=0` in `php.ini` as defense-in-depth.

### 7.3 Python (Gunicorn / uWSGI)

```bash
# Install Gunicorn
pip install gunicorn

# Run Flask/Django app with Gunicorn
gunicorn --bind 127.0.0.1:8000 --workers 4 myapp.wsgi:application
```

```nginx
server {
    listen 80;
    server_name pyapp.example.com;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Serve static files directly (bypass Gunicorn)
    location /static/ {
        alias /var/www/pyapp/staticfiles/;
        expires 30d;
    }

    # Serve media uploads directly
    location /media/ {
        alias /var/www/pyapp/media/;
        expires 7d;
    }
}
```

**uWSGI variant (native protocol — faster than HTTP proxy):**

```nginx
location / {
    uwsgi_pass 127.0.0.1:8001;    # or unix:/run/uwsgi/myapp.sock
    include uwsgi_params;
}
```

### 7.4 Node.js (Express, Fastify, NestJS)

```bash
# Run Node.js app (PM2 recommended for production)
npm install -g pm2
pm2 start app.js --name myapp -i max    # Cluster mode = 1 process per CPU
pm2 save && pm2 startup                  # Survive reboot
```

```nginx
upstream node_backend {
    server 127.0.0.1:3000;
    server 127.0.0.1:3001;    # Multiple Node instances
    keepalive 64;               # Reuse connections to backend
}

server {
    listen 80;
    server_name nodeapp.example.com;

    location / {
        proxy_pass http://node_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";          # Enable keepalive
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Let Nginx serve static assets (much faster than Node)
    location /public/ {
        alias /var/www/nodeapp/public/;
        expires 30d;
    }
}
```

> **Production Tip:** Always put Nginx in front of Node.js. Node's single-threaded model is inefficient at serving static files and doesn't handle slow clients well (slow loris attacks). Nginx buffers requests and shields Node.

---

## 8. Load Balancing

### 8.1 Architecture

```
                         ┌─────────────┐
                         │   Client    │
                         └──────┬──────┘
                                │
                         ┌──────▼──────┐
                         │   Nginx     │
                         │   (LB)      │
                         └──┬───┬───┬──┘
                            │   │   │
                   ┌────────┘   │   └────────┐
                   ▼            ▼            ▼
              ┌─────────┐ ┌─────────┐ ┌─────────┐
              │Backend 1│ │Backend 2│ │Backend 3│
              │ :3001   │ │ :3002   │ │ :3003   │
              └─────────┘ └─────────┘ └─────────┘
```

### 8.2 Load Balancing Methods

```nginx
# --- Round Robin (default) ---
upstream backend {
    server 10.0.0.1:3000;
    server 10.0.0.2:3000;
    server 10.0.0.3:3000;
}

# --- Weighted Round Robin ---
upstream backend {
    server 10.0.0.1:3000 weight=5;    # Gets 5x more traffic
    server 10.0.0.2:3000 weight=3;
    server 10.0.0.3:3000 weight=1;
}

# --- Least Connections ---
upstream backend {
    least_conn;
    server 10.0.0.1:3000;
    server 10.0.0.2:3000;
}

# --- IP Hash (sticky sessions) ---
upstream backend {
    ip_hash;
    server 10.0.0.1:3000;
    server 10.0.0.2:3000;
}

# --- Health Checks + Failure Handling ---
upstream backend {
    server 10.0.0.1:3000 max_fails=3 fail_timeout=30s;
    server 10.0.0.2:3000 max_fails=3 fail_timeout=30s;
    server 10.0.0.3:3000 backup;      # Only used when all others are down
}
```

| Method | Use Case | Gotcha |
|--------|----------|--------|
| **Round Robin** | Stateless APIs, microservices | Uneven if backends differ in speed |
| **Weighted** | Mixed hardware (new + old servers) | Must manually tune weights |
| **Least Conn** | Long-lived connections, varied request times | Slightly more overhead |
| **IP Hash** | Session affinity (stateful apps) | Uneven with NAT/CDN (many clients share IP) |

> **⚠️ Gotcha:** `ip_hash` breaks when clients come through a CDN or corporate proxy — thousands of users share 1 IP, so one backend gets hammered. Use cookie-based sticky sessions or externalize session state (Redis) instead.

---

## 9. Caching & Performance Tuning

### 9.1 Proxy Cache (Reverse Proxy Caching)

```nginx
# Define cache zone in http {} context
http {
    proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m
                     max_size=1g inactive=60m use_temp_path=off;

    server {
        location / {
            proxy_pass http://backend;

            # Enable caching
            proxy_cache my_cache;
            proxy_cache_valid 200 302 10m;     # Cache 200/302 for 10 min
            proxy_cache_valid 404 1m;           # Cache 404 for 1 min
            proxy_cache_use_stale error timeout updating http_500 http_502 http_503;
            proxy_cache_lock on;                # Prevent cache stampede

            # Add header to show cache status (debugging)
            add_header X-Cache-Status $upstream_cache_status;
        }

        # Bypass cache for authenticated users
        location /api/ {
            proxy_pass http://backend;
            proxy_cache my_cache;
            proxy_cache_bypass $cookie_session $http_authorization;
            proxy_no_cache $cookie_session $http_authorization;
        }
    }
}
```

**Cache status values (`$upstream_cache_status`):**

| Status | Meaning |
|--------|---------|
| `MISS` | Not in cache — fetched from backend |
| `HIT` | Served from cache |
| `EXPIRED` | Cache entry expired — re-fetched |
| `STALE` | Served stale (backend unreachable) |
| `UPDATING` | Stale served while background update |
| `BYPASS` | Cache intentionally bypassed |

### 9.2 Gzip / Brotli Compression

```nginx
http {
    # --- Gzip ---
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;          # 1-9 (6 = good balance of speed/ratio)
    gzip_min_length 256;         # Don't compress tiny responses
    gzip_types
        text/plain text/css application/json application/javascript
        text/xml application/xml application/xml+rss text/javascript
        application/vnd.ms-fontobject application/x-font-ttf
        font/opentype image/svg+xml image/x-icon;

    # --- Brotli (requires module) ---
    # brotli on;
    # brotli_comp_level 6;
    # brotli_types text/plain text/css application/json application/javascript;
}
```

> **⚠️ Gotcha:** Don't gzip-compress already-compressed formats (JPEG, PNG, MP4, WOFF2, ZIP). It wastes CPU and can actually increase size. The `gzip_types` directive should only list text-based formats.

### 9.3 Worker Tuning

```nginx
# /etc/nginx/nginx.conf

worker_processes auto;              # = number of CPU cores
worker_rlimit_nofile 65535;         # Max open files per worker

events {
    worker_connections 4096;        # Max connections per worker
    multi_accept on;                # Accept all pending connections at once
    use epoll;                      # Linux optimal (default on Linux)
}

http {
    sendfile on;                    # Zero-copy file transfer
    tcp_nopush on;                  # Send headers + file in one packet
    tcp_nodelay on;                 # Disable Nagle's algorithm
    keepalive_timeout 65;           # Keep connections open for reuse
    keepalive_requests 1000;        # Max requests per keepalive connection
    types_hash_max_size 2048;

    # Client body limits
    client_max_body_size 50m;       # Max upload size
    client_body_buffer_size 128k;
    client_header_buffer_size 1k;
    large_client_header_buffers 4 8k;
}
```

**Capacity calculation:**

```
  Max Concurrent Connections = worker_processes × worker_connections

  Example: 4 cores × 4096 = 16,384 concurrent connections
  (each connection uses ~1-2 KB of memory)
  
  Memory ≈ 16,384 × 2 KB ≈ 32 MB  ← This is why Nginx is legendary
```

---

## 10. Security Hardening

### 10.1 Security Headers

```nginx
server {
    # Prevent clickjacking
    add_header X-Frame-Options "SAMEORIGIN" always;

    # Prevent MIME-type sniffing
    add_header X-Content-Type-Options "nosniff" always;

    # XSS protection (legacy browsers)
    add_header X-XSS-Protection "1; mode=block" always;

    # Referrer policy
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Content Security Policy (customize per app)
    add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline';" always;

    # HSTS — force HTTPS (2 years + subdomains + preload)
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

    # Permissions Policy
    add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;
}
```

### 10.2 Rate Limiting

```nginx
http {
    # Define rate limit zone: 10 requests/second per client IP
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=login_limit:10m rate=1r/s;

    server {
        # API rate limit — allow burst of 20 with no delay for first 10
        location /api/ {
            limit_req zone=api_limit burst=20 nodelay;
            limit_req_status 429;
            proxy_pass http://backend;
        }

        # Login brute-force protection — strict 1 req/sec
        location /login {
            limit_req zone=login_limit burst=5;
            limit_req_status 429;
            proxy_pass http://backend;
        }
    }
}
```

### 10.3 Block Bad Actors

```nginx
# Block by IP
deny 192.168.1.100;
deny 10.0.0.0/8;
allow 172.16.0.0/12;
allow all;

# Block by User-Agent
if ($http_user_agent ~* (curl|wget|python|scrapy|bot)) {
    return 403;
}

# Block sensitive paths
location ~ /\.(git|svn|env|htpasswd) {
    deny all;
    return 404;
}

# Disable unused HTTP methods
if ($request_method !~ ^(GET|HEAD|POST|PUT|PATCH|DELETE)$) {
    return 405;
}
```

### 10.4 Hide Nginx Version

```nginx
http {
    server_tokens off;   # Hides "nginx/1.x.x" from headers & error pages
}
```

### 10.5 Basic Authentication

```bash
# Install htpasswd tool
sudo dnf install httpd-tools -y    # RHEL
sudo apt install apache2-utils -y  # Debian

# Create password file
sudo htpasswd -c /etc/nginx/.htpasswd admin
```

```nginx
location /admin/ {
    auth_basic "Restricted Area";
    auth_basic_user_file /etc/nginx/.htpasswd;
    proxy_pass http://admin_backend;
}
```

---

## 11. Logging, Debugging & Monitoring

### 11.1 Custom Log Formats

```nginx
http {
    # JSON log format (structured logging for ELK/Loki/Datadog)
    log_format json_combined escape=json
        '{'
            '"time": "$time_iso8601",'
            '"remote_addr": "$remote_addr",'
            '"request_method": "$request_method",'
            '"request_uri": "$request_uri",'
            '"status": $status,'
            '"body_bytes_sent": $body_bytes_sent,'
            '"request_time": $request_time,'
            '"upstream_response_time": "$upstream_response_time",'
            '"http_user_agent": "$http_user_agent",'
            '"http_referer": "$http_referer",'
            '"upstream_cache_status": "$upstream_cache_status"'
        '}';

    access_log /var/log/nginx/access.log json_combined;
}
```

### 11.2 Conditional Logging

```nginx
# Don't log health checks (reduce noise)
map $request_uri $loggable {
    ~*^/health   0;
    ~*^/ping     0;
    default      1;
}

access_log /var/log/nginx/access.log combined if=$loggable;
```

### 11.3 Debug Mode

```nginx
# Enable debug logging (VERY verbose — only for troubleshooting)
error_log /var/log/nginx/error.log debug;

# Debug only for specific client IPs
events {
    debug_connection 192.168.1.100;
    debug_connection 10.0.0.0/24;
}
```

### 11.4 Stub Status (Monitoring Endpoint)

```nginx
# Expose metrics for Prometheus/Nagios/Zabbix
location /nginx_status {
    stub_status;
    allow 10.0.0.0/8;      # Only internal IPs
    deny all;
}
```

Output:

```
Active connections: 291
server accepts handled requests
 16630948 16630948 31070465
Reading: 6 Writing: 179 Waiting: 106
```

| Metric | Meaning |
|--------|---------|
| **Active connections** | Currently open connections |
| **Reading** | Reading request headers |
| **Writing** | Sending response |
| **Waiting** | Keepalive connections idle |

---

## 12. Production Gotchas & Tricky Q&A

### Q1: Why does my Nginx return 502 Bad Gateway?

**Answer:** 502 means Nginx **reached the backend but got an invalid response** (or the backend crashed mid-response).

```
  Checklist:
  ┌──────────────────────────────────────────────────────────────┐
  │ 1. Is the backend process running?                           │
  │    → systemctl status php-fpm / pm2 list / ss -tlnp          │
  │                                                              │
  │ 2. Is the socket/port accessible?                            │
  │    → curl -v http://127.0.0.1:3000                           │
  │                                                              │
  │ 3. SELinux blocking network connections?                     │
  │    → setsebool -P httpd_can_network_connect 1                │
  │                                                              │
  │ 4. Socket permissions wrong? (PHP-FPM)                       │
  │    → ls -la /run/php-fpm/www.sock                            │
  │    → Check listen.owner/listen.group in www.conf             │
  │                                                              │
  │ 5. Backend timeout?                                          │
  │    → Increase proxy_read_timeout / fastcgi_read_timeout      │
  └──────────────────────────────────────────────────────────────┘
```

> **The SELinux gotcha is the #1 trap on RHEL/CentOS.** Nginx runs as `httpd_t` context. By default, SELinux blocks `httpd_t` from making outbound network connections. You MUST run `setsebool -P httpd_can_network_connect 1`.

---

### Q2: 502 vs 504 — What's the difference?

| Code | Meaning | Common Cause |
|------|---------|-------------|
| **502** | Bad Gateway — backend sent invalid response | Backend crashed, wrong socket path, SELinux |
| **504** | Gateway Timeout — backend didn't respond in time | Slow query, overloaded backend, low timeout |

**Fix 504:**

```nginx
proxy_connect_timeout 300s;
proxy_read_timeout 300s;
proxy_send_timeout 300s;
```

---

### Q3: Why does `proxy_pass` with a trailing slash break my app?

```
  location /app/ {
      proxy_pass http://backend/;     ← trailing slash
  }

  Request:  /app/v1/users
  Backend:  /v1/users           ← /app/ prefix STRIPPED

  location /app/ {
      proxy_pass http://backend;      ← NO trailing slash
  }

  Request:  /app/v1/users
  Backend:  /app/v1/users       ← Full URI passed through
```

**Rule:** Use trailing slash when the backend doesn't expect the prefix. Omit it when the backend routes include the prefix.

---

### Q4: Why are my `add_header` directives disappearing?

**The inheritance trap:**

```nginx
server {
    add_header X-Frame-Options "SAMEORIGIN";     # ← Set here
    add_header Strict-Transport-Security "...";    # ← Set here

    location /api/ {
        add_header X-Custom "value";               # ← THIS KILLS the above headers!
        # X-Frame-Options and HSTS are gone in this location block
        proxy_pass http://backend;
    }
}
```

**Fix:** Repeat ALL headers in every location block that has its own `add_header`, or use the `always` parameter and an `include` snippet:

```nginx
# /etc/nginx/snippets/security-headers.conf
add_header X-Frame-Options "SAMEORIGIN" always;
add_header Strict-Transport-Security "max-age=63072000" always;
add_header X-Content-Type-Options "nosniff" always;

# In server/location blocks:
location /api/ {
    include snippets/security-headers.conf;
    add_header X-Custom "value" always;
    proxy_pass http://backend;
}
```

---

### Q5: My upstream shows `110: Connection timed out` — but the backend is running!

**Causes:**

1. **Firewall:** `firewall-cmd` or `iptables` blocking the port between Nginx and backend
2. **SELinux:** `httpd_can_network_connect` not set (see Q1)
3. **Backend bound to wrong interface:** Backend listening on `127.0.0.1` but Nginx trying to reach `10.0.0.2`
4. **DNS resolution failure:** If using hostname in `upstream`, Nginx resolves at startup and caches forever

**Fix for DNS caching:**

```nginx
# BAD — Nginx caches DNS at startup forever
upstream backend {
    server my-backend.service.consul:3000;
}

# GOOD — Re-resolve DNS using a variable
server {
    resolver 127.0.0.1 valid=30s;     # Local DNS, re-resolve every 30s
    set $backend "http://my-backend.service.consul:3000";

    location / {
        proxy_pass $backend;
    }
}
```

> **⚠️ This is critical in Docker/K8s/Consul** where backend IPs change frequently. Without `resolver` + variable trick, Nginx keeps routing to dead IPs.

---

### Q6: How do I do zero-downtime deployments with Nginx?

```
  Blue-Green Deployment with Nginx:
  ┌─────────────────────────────────────────────────────┐
  │                                                     │
  │  upstream backend {                                 │
  │      server 10.0.0.1:3000;  # Blue  (current)       │
  │      server 10.0.0.2:3000;  # Green (new version)   │
  │  }                                                  │
  │                                                     │
  │  Step 1: Deploy new version to Green                │
  │  Step 2: Health check Green                         │
  │  Step 3: Update upstream to point to Green only     │
  │  Step 4: nginx -t && systemctl reload nginx         │
  │  Step 5: Drain Blue (old workers finish requests)   │
  │  Step 6: Decommission Blue or make it the new Green │
  └─────────────────────────────────────────────────────┘
```

`reload` is the key — it's graceful. Old workers finish existing connections while new workers use the updated config.

---

### Q7: `worker_connections` exceeded — what happens?

Nginx returns **503 Service Unavailable** to new clients. The error log shows:

```
worker_connections are not enough
```

**Fix:**

```nginx
events {
    worker_connections 4096;     # Increase (default is 512 or 1024)
}
# Also increase OS file descriptor limit:
# /etc/security/limits.conf:
# nginx soft nofile 65535
# nginx hard nofile 65535
```

**Also set:**

```nginx
worker_rlimit_nofile 65535;
```

---

### Q8: How to handle large file uploads through Nginx?

```nginx
server {
    client_max_body_size 100m;        # Max upload size (default 1m!)
    client_body_buffer_size 128k;     # Buffer before writing to disk
    client_body_temp_path /var/nginx/tmp 1 2;

    location /upload {
        proxy_pass http://backend;
        proxy_request_buffering off;   # Stream directly to backend
                                       # (don't buffer entire upload in Nginx)
    }
}
```

> **⚠️ Gotcha:** The default `client_max_body_size` is **1 MB**. Your app returns 200 for small uploads but fails for anything >1 MB with `413 Request Entity Too Large` — and the error comes from Nginx, not your app, making it confusing to debug.

---

### Q9: How to pass the real client IP when behind CloudFlare / AWS ALB?

Behind a CDN or load balancer, `$remote_addr` is the CDN IP, not the client.

```nginx
# Set real IP from CloudFlare
set_real_ip_from 173.245.48.0/20;
set_real_ip_from 103.21.244.0/22;
set_real_ip_from 103.22.200.0/22;
set_real_ip_from 103.31.4.0/22;
set_real_ip_from 141.101.64.0/18;
set_real_ip_from 108.162.192.0/18;
set_real_ip_from 190.93.240.0/20;
set_real_ip_from 188.114.96.0/20;
set_real_ip_from 197.234.240.0/22;
set_real_ip_from 198.41.128.0/17;
set_real_ip_from 2400:cb00::/32;
set_real_ip_from 2606:4700::/32;
set_real_ip_from 2803:f800::/32;
set_real_ip_from 2405:b500::/32;
set_real_ip_from 2405:8100::/32;
set_real_ip_from 2c0f:f248::/32;
set_real_ip_from 2a06:98c0::/29;

real_ip_header CF-Connecting-IP;    # CloudFlare header
# real_ip_header X-Forwarded-For;   # AWS ALB header

# Now $remote_addr = real client IP
```

---

### Q10: What's the difference between `reload` and `restart`?

```
  reload (SIGHUP)                      restart (stop + start)
  ┌──────────────────────────┐          ┌──────────────────────────┐
  │ Master reads new config  │          │ Master stops ALL workers │
  │ Spawns NEW workers       │          │ ALL connections DROPPED  │
  │ OLD workers finish       │          │ New master starts        │
  │   existing requests      │          │ New workers start        │
  │ Then OLD workers exit    │          │                          │
  │                          │          │ DOWNTIME window ❌       │
  │ ZERO downtime ✅         │          │                          │
  └──────────────────────────┘          └──────────────────────────┘
```

**Always `reload` in production.** Only `restart` if the binary itself was updated (not just config).

---

### Q11: How to debug "no resolver defined to resolve" error?

This happens when you use a **variable** in `proxy_pass` (which triggers runtime DNS resolution) but haven't defined a `resolver`:

```nginx
# ERROR:
set $backend "http://myapp.service.local:3000";
location / {
    proxy_pass $backend;    # Needs runtime DNS → needs resolver
}

# FIX:
resolver 127.0.0.53 valid=30s;    # systemd-resolved
# or
resolver 8.8.8.8 valid=60s;       # Google DNS
```

---

### Q12: How to handle mixed HTTP/HTTPS environments (redirect loops)?

```
  Client ──HTTPS──► CloudFlare/ALB ──HTTP──► Nginx ──HTTP──► Backend
                                               │
                    Nginx sees HTTP, redirects │
                    to HTTPS → infinite loop!  │
                    ◄──────────────────────────┘
```

**Fix:** Trust the upstream `X-Forwarded-Proto` header:

```nginx
server {
    listen 80;

    # Only redirect if the ORIGINAL client connection was HTTP
    if ($http_x_forwarded_proto = "http") {
        return 301 https://$host$request_uri;
    }

    # If X-Forwarded-Proto is https, serve normally
    location / {
        proxy_pass http://backend;
    }
}
```

---

### Q13: What's the `map` directive and when to use it?

`map` creates a variable whose value depends on another variable — like a switch/case:

```nginx
# Route to different backends based on a header
map $http_x_api_version $backend {
    "v1"     http://backend-v1;
    "v2"     http://backend-v2;
    default  http://backend-v1;
}

server {
    location /api/ {
        proxy_pass $backend;
        resolver 127.0.0.53 valid=30s;
    }
}

# Feature flags via cookie
map $cookie_feature_flag $feature_backend {
    "new_ui"   http://frontend-beta:8080;
    default    http://frontend-stable:8080;
}
```

> `map` is evaluated **lazily** (only when the variable is used) and is much more efficient than nested `if` blocks.

---

### Q14: Running Nginx in Docker — key gotchas?

```dockerfile
FROM nginx:alpine
COPY nginx.conf /etc/nginx/nginx.conf
COPY dist/ /usr/share/nginx/html/
EXPOSE 80 443
# Nginx must run in foreground (not daemon mode)
CMD ["nginx", "-g", "daemon off;"]
```

**Gotchas:**

1. **`daemon off;`** — Without this, Nginx forks to background and the container exits immediately
2. **Signals:** Docker sends SIGTERM on `docker stop`. Nginx handles this gracefully
3. **Logs:** Send logs to stdout/stderr for `docker logs` to work:
   ```nginx
   access_log /dev/stdout;
   error_log /dev/stderr;
   ```
4. **DNS in Docker Compose:** Use Docker's internal DNS resolver:
   ```nginx
   resolver 127.0.0.11 valid=30s;   # Docker embedded DNS
   ```

---

### Q15: How do I test Nginx configuration changes safely?

```bash
# 1. Always test syntax first
nginx -t

# 2. Dump the full effective config (shows all includes merged)
nginx -T

# 3. Test with a specific config file (not the default)
nginx -t -c /etc/nginx/nginx-staging.conf

# 4. Use a staging server block with a test domain
# 5. Use curl to test specific behaviors:
curl -I https://example.com                       # Check headers
curl -H "Host: test.example.com" http://localhost  # Test vhost routing
curl -k https://localhost                          # Skip cert verification
curl -w "%{http_code} %{time_total}s\n" -o /dev/null -s https://example.com
```

---

> **End of Nginx Guide** | [🏠 Home](../README.md) · [Servers](README.md) · [Next: Apache →](apache.md)
