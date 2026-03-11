[🏠 Home](../README.md) · [Edge Cases](.)

# 🌐 Nginx — Edge Cases (110)

> **Audience:** DevOps engineers, SREs, and platform engineers running Nginx in production as reverse proxy, load balancer, TLS terminator, and static file server.  
> **Coverage:** Config syntax traps, proxy pitfalls, SSL/TLS gotchas, upstream failures, caching nightmares, rate limiting quirks, header manipulation, logging, performance, and containerised deployments.

---

## Table of Contents

1. [Configuration Syntax & Parsing (EC-001–015)](#configuration-syntax--parsing)
2. [Reverse Proxy & Upstream (EC-016–030)](#reverse-proxy--upstream)
3. [SSL / TLS & Certificates (EC-031–045)](#ssl--tls--certificates)
4. [Headers, Buffers & Body Handling (EC-046–058)](#headers-buffers--body-handling)
5. [Caching (EC-059–070)](#caching)
6. [Rate Limiting & Access Control (EC-071–082)](#rate-limiting--access-control)
7. [Logging & Monitoring (EC-083–090)](#logging--monitoring)
8. [Performance & Tuning (EC-091–100)](#performance--tuning)
9. [Containers, K8s & Cloud (EC-101–110)](#containers-k8s--cloud)
10. [Production FAQ](#production-faq)

---

## Configuration Syntax & Parsing

### EC-001 — Missing Semicolon Causes Cryptic "unexpected }" Error
**Category:** Config | **Severity:** High | **Env:** Both

**Scenario:** A single missing semicolon in `nginx.conf`. Error message points to a line BELOW the actual problem — the closing `}` — not the line where the semicolon is missing.

```
  server {
      listen 80;
      server_name example.com     ← MISSING semicolon!
      root /var/www/html;         ← nginx reports error HERE or on }
  }
  
  nginx -t output:
  "unexpected '}' in /etc/nginx/nginx.conf:5"
  ← Points to line 5 but the bug is on line 3!
```

**Detection:**
```bash
nginx -t   # always test before reload
# Parse error messages — look at the lines ABOVE the reported line
```

**Fix:**
```bash
# Add the missing semicolon, then:
nginx -t && nginx -s reload
```

> ⚠️ **DevOps Gotcha:** ALWAYS run `nginx -t` before `nginx -s reload` in CI/CD pipelines. Chain them: `nginx -t && nginx -s reload`. If the test fails, reload never runs.

---

### EC-002 — `if` Is Evil Inside `location` Blocks
**Category:** Config | **Severity:** High | **Env:** Both

**Scenario:** Using `if` inside a `location` block for conditional logic. Nginx `if` creates an implicit nested location that inherits (or doesn't inherit) directives unpredictably.

```
  location / {
      set $flag 0;
      if ($request_uri ~* "^/api") {
          set $flag 1;
      }
      # ⚠️ proxy_pass inside if may or may not work
      # depending on Nginx version and other directives
      if ($flag = 1) {
          proxy_pass http://backend;  ← UNRELIABLE inside if!
      }
  }
```

**Fix:**
```nginx
# Use map + separate location blocks instead:
map $request_uri $backend {
    ~^/api   http://api-backend;
    default  http://default-backend;
}
server {
    location / {
        proxy_pass $backend;
    }
}
```

> ⚠️ **DevOps Gotcha:** The official Nginx wiki page "If Is Evil" documents this extensively. Only safe uses of `if` inside location: `return`, `rewrite`, `set`. Everything else is unpredictable.

---

### EC-003 — `location` Block Order Doesn't Match Expectation
**Category:** Config | **Severity:** High | **Env:** Both

**Scenario:** Developer expects first-match-wins for location blocks. Nginx actually uses a complex matching priority: exact (`=`) → prefix (longest) → regex (`~` / `~*`, first match) → general prefix.

```
  Nginx location matching priority:
  
  ┌─────────────────────────────────────────────────┐
  │ 1. = /exact       (exact match — highest)       │
  │ 2. ^~ /prefix     (prefix, stops regex search)  │
  │ 3. ~ regex        (case-sensitive regex, FIFO)   │
  │    ~* regex       (case-insensitive regex, FIFO) │
  │ 4. /prefix        (longest prefix match)         │
  │ 5. /              (default catch-all — lowest)   │
  └─────────────────────────────────────────────────┘
  
  Request: /api/v1/users
  
  location /api { }          ← prefix match (4 chars)
  location /api/v1 { }       ← LONGER prefix match (7 chars) — WINS!
  location ~ ^/api { }       ← regex match — checked ONLY if no ^~ prefix
```

> ⚠️ **DevOps Gotcha:** Regex locations are matched in CONFIG FILE ORDER (first match wins). Prefix locations use LONGEST match. This dual system causes most location-routing bugs. Test with `curl -v` and check which location block handles the request.

---

### EC-004 — `include` Glob Pattern Silently Matches Nothing
**Category:** Config | **Severity:** Medium | **Env:** Both

**Scenario:** `include /etc/nginx/conf.d/*.conf;` — directory exists but is empty. No error — Nginx silently includes nothing. If config files are expected (e.g., from a ConfigMap mount), missing includes go unnoticed.

**Detection:**
```bash
ls /etc/nginx/conf.d/*.conf   # check if files actually exist
nginx -T   # dumps effective configuration — shows what was actually included
```

> ⚠️ **DevOps Gotcha:** Use `nginx -T` (capital T) to dump the full, merged, effective configuration. This shows exactly what Nginx will use — including all included files. Essential for debugging.

---

### EC-005 — Duplicate `server_name` — Wrong Server Block Chosen
**Category:** Config | **Severity:** High | **Env:** Prod

**Scenario:** Two server blocks both declare `server_name example.com`. Nginx picks the FIRST one in config file order. If `include` order changes (alphabetical glob), the wrong server block handles requests.

**Detection:**
```bash
# Find all server_name declarations:
grep -rn "server_name" /etc/nginx/conf.d/
# Test which server block handles a request:
curl -H "Host: example.com" http://localhost -v
```

---

### EC-006 — `default_server` Not Set — First Server Block Becomes Default
**Category:** Config | **Severity:** High | **Env:** Prod

**Scenario:** No server block has `listen 80 default_server`. Request with unknown `Host` header arrives. Nginx routes to the FIRST server block found (include file order = alphabetical). Unexpected service exposed.

```
  Request: Host: unknown-domain.com
  
  Without default_server:
  ┌──────────────────────────────────────────────┐
  │ Nginx picks FIRST server block in config     │
  │ order (alphabetical include glob)            │
  │ → Could be any service!                      │
  └──────────────────────────────────────────────┘
  
  With explicit default_server:
  ┌──────────────────────────────────────────────┐
  │ Designated server block catches unknown hosts │
  │ → Return 444 (close connection) or 403       │
  └──────────────────────────────────────────────┘
```

**Fix:**
```nginx
# Create a catch-all default server:
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    return 444;   # close connection with no response
}
```

> ⚠️ **DevOps Gotcha:** ALWAYS set an explicit `default_server` that returns 444 or 403. This prevents accidental exposure of internal services via direct IP or spoofed Host headers. Critical for security audits.

---

### EC-007 — `root` vs `alias` — Trailing Slash Trap
**Category:** Config | **Severity:** High | **Env:** Both

**Scenario:** `alias` requires a trailing slash when the `location` has a trailing slash, but `root` doesn't. Mixing them up causes 404s or serves wrong files.

```
  location /images/ {
      root /var/www;
  }
  # Request: /images/photo.jpg → serves /var/www/images/photo.jpg ✅
  # root APPENDS the location path to the root
  
  location /images/ {
      alias /var/www/photos/;
  }
  # Request: /images/photo.jpg → serves /var/www/photos/photo.jpg ✅
  # alias REPLACES the location path with the alias path
  
  # TRAP — missing trailing slash on alias:
  location /images/ {
      alias /var/www/photos;    ← MISSING trailing slash!
  }
  # Request: /images/photo.jpg → serves /var/www/photosphoto.jpg ← BROKEN!
```

> ⚠️ **DevOps Gotcha:** When using `alias`, always match trailing slashes between location and alias paths. Or better: use `root` whenever possible — it's less error-prone.

---

### EC-008 — `nginx -s reload` Silently Fails if Worker Can't Bind to Port
**Category:** Config | **Severity:** Medium | **Env:** Prod

**Scenario:** Config change adds `listen 8443`. Reload signals the master process. New worker tries to bind port 8443 but it's already in use. Old workers continue serving with old config — partial reload.

**Detection:**
```bash
# Check Nginx error log after reload:
tail -f /var/log/nginx/error.log
# Check which ports Nginx is actually listening on:
ss -tlnp | grep nginx
```

---

### EC-009 — Variable Interpolation Inside `upstream` Block Fails
**Category:** Config | **Severity:** Medium | **Env:** Both

**Scenario:** Trying to use variables inside `upstream` block for dynamic server addresses. Nginx `upstream` block is parsed at config load time — variables are NOT resolved.

```nginx
# THIS DOESN'T WORK:
upstream backend {
    server $backend_host:$backend_port;   # ← variables NOT resolved!
}

# For dynamic resolution, use variable in proxy_pass directly:
location / {
    set $backend "http://dynamic-host:8080";
    proxy_pass $backend;   # ← resolved per-request
    resolver 127.0.0.1;    # ← REQUIRED for variable proxy_pass!
}
```

> ⚠️ **DevOps Gotcha:** When using a variable in `proxy_pass`, you MUST set `resolver`. Without it, DNS resolution fails silently. In Kubernetes, use the cluster DNS: `resolver kube-dns.kube-system.svc.cluster.local valid=10s;`

---

### EC-010 — `worker_processes auto` Allocates Too Many Workers on Large VM
**Category:** Config | **Severity:** Medium | **Env:** Prod

**Scenario:** `worker_processes auto` sets one worker per CPU core. On a 96-core VM, 96 worker processes are created. Mostly idle, consuming RAM unnecessarily.

**Fix:**
```nginx
# Set explicitly for proxy workloads:
worker_processes 4;   # usually enough for reverse proxy
# Or limit in container environments:
worker_processes auto;
worker_cpu_affinity auto;   # bind workers to specific cores
```

---

### EC-011 — Config Refers to Non-Existent Log Directory — Nginx Won't Start
**Category:** Config | **Severity:** High | **Env:** Prod

**Scenario:** `access_log /var/log/nginx/app/access.log;` — but `/var/log/nginx/app/` directory doesn't exist. Nginx refuses to start entirely (not just the server block).

**Fix:**
```bash
mkdir -p /var/log/nginx/app
chown www-data:www-data /var/log/nginx/app
nginx -t && systemctl start nginx
```

---

### EC-012 — `server_name` With Wildcard Doesn't Match Subdomains as Expected
**Category:** Config | **Severity:** Medium | **Env:** Prod

**Scenario:** `server_name *.example.com` matches `api.example.com` but NOT `deep.sub.example.com` (only one level of wildcard). Also doesn't match `example.com` itself.

```
  *.example.com matches:
  ✅ api.example.com
  ✅ www.example.com
  ❌ deep.sub.example.com  (two levels)
  ❌ example.com           (no subdomain)
```

**Fix:**
```nginx
server_name example.com *.example.com;   # match bare domain AND one-level wildcards
# For multi-level, use regex:
server_name ~^(.+\.)?example\.com$;
```

---

### EC-013 — `try_files` Last Argument Must Be URI or Error Code
**Category:** Config | **Severity:** Medium | **Env:** Both

**Scenario:** `try_files $uri $uri/ /index.html` — if none of the files exist and the last argument isn't a valid URI or `=404`, Nginx enters an infinite loop.

```nginx
# WRONG — last argument is a file path, not a URI:
try_files $uri $uri/ /var/www/fallback.html;   # ← internal redirect loop!

# CORRECT — last argument is a URI:
try_files $uri $uri/ /index.html;   # ← URI, Nginx processes this as internal request
try_files $uri $uri/ =404;          # ← error code, returns 404
```

---

### EC-014 — `proxy_pass` With and Without Trailing Slash — Different Behavior
**Category:** Config | **Severity:** High | **Env:** Both

**Scenario:** Trailing slash on `proxy_pass` URI controls whether the `location` path is stripped or passed through.

```
  location /api/ {
      proxy_pass http://backend;       # NO trailing slash
  }
  # Request: /api/users → backend receives: /api/users (path preserved)
  
  location /api/ {
      proxy_pass http://backend/;      # WITH trailing slash
  }
  # Request: /api/users → backend receives: /users (location prefix stripped!)
  
  ┌────────────────────────────────────────────────────────────────┐
  │ proxy_pass without URI: forwards full original request path   │
  │ proxy_pass with URI (/): replaces location prefix with URI    │
  └────────────────────────────────────────────────────────────────┘
```

> ⚠️ **DevOps Gotcha:** This single trailing slash has caused countless production incidents. Document the intent in comments. Test with `curl` against both Nginx and the backend directly to verify paths match.

---

### EC-015 — `nginx -t` Passes But Reload Fails Due to Runtime Dependency
**Category:** Config | **Severity:** Medium | **Env:** Prod

**Scenario:** `nginx -t` validates config syntax and basic structure. But runtime checks (DNS resolution for upstream names, SSL cert file permissions under worker user) only happen at reload/start time.

**Detection:**
```bash
nginx -t          # tests syntax only
systemctl reload nginx
journalctl -u nginx --since "1 min ago"   # check for runtime errors
```

---

## Reverse Proxy & Upstream

### EC-016 — Upstream Server Returns 502 After Backend Restart
**Category:** Proxy | **Severity:** High | **Env:** Prod

**Scenario:** Backend application restarted (deployment). Nginx has an established connection in its connection pool. Sends request on stale connection. Backend rejects → 502 Bad Gateway.

```
  Timeline:
  1. Nginx opens connection to backend (keep-alive pool)
  2. Backend restarts (connection closed on backend side)
  3. Client request arrives
  4. Nginx reuses pooled connection → sends request
  5. Backend TCP stack sends RST (connection unknown)
  6. Nginx returns 502 to client
  
  ┌──────────┐  stale conn  ┌──────────┐
  │  Nginx   │──────────────│ Backend  │ ← restarted, doesn't
  │          │  request →   │ (new PID)│   recognize connection
  │          │  ← RST       │          │
  └──────────┘              └──────────┘
```

**Fix:**
```nginx
upstream backend {
    server 10.0.0.1:8080;
    keepalive 32;
}
location / {
    proxy_pass http://backend;
    proxy_next_upstream error timeout http_502;   # retry on 502
    proxy_connect_timeout 5s;
    proxy_http_version 1.1;
    proxy_set_header Connection "";   # enable keepalive to upstream
}
```

> ⚠️ **DevOps Gotcha:** During rolling deployments, short bursts of 502s are expected. Set `proxy_next_upstream` to retry on another upstream. Combine with health checks (Nginx Plus or third-party module) for zero-downtime deployments.

---

### EC-017 — Upstream Health Check Missing — Traffic Sent to Dead Backends
**Category:** Proxy | **Severity:** Critical | **Env:** Prod

**Scenario:** Upstream backend crashes. Nginx (open source) has NO active health checks — only passive. It keeps sending requests to the dead backend until enough failures accumulate.

```
  Open Source Nginx passive health checks:
  ┌────────────────────────────────────────────────────────┐
  │ max_fails=3 fail_timeout=30s (defaults)                │
  │ → 3 failed requests within 30s → mark server "down"   │
  │ → After 30s, try again                                 │
  │                                                        │
  │ Problem: 3 real client requests FAIL before detection!  │
  └────────────────────────────────────────────────────────┘
  
  Nginx Plus active health checks:
  ┌────────────────────────────────────────────────────────┐
  │ Periodic background health probes (/health endpoint)   │
  │ Removes server from pool BEFORE client requests fail   │
  └────────────────────────────────────────────────────────┘
```

**Fix:**
```nginx
upstream backend {
    server 10.0.0.1:8080 max_fails=1 fail_timeout=10s;
    server 10.0.0.2:8080 max_fails=1 fail_timeout=10s;
}
# For active health checks without Nginx Plus, use:
# - nginx_upstream_check_module (third-party)
# - External health checker (Consul, HAProxy in front)
# - Kubernetes readiness probes (if running in K8s)
```

---

### EC-018 — `proxy_set_header Host` Not Set — Backend Gets Wrong Host
**Category:** Proxy | **Severity:** High | **Env:** Prod

**Scenario:** Nginx proxies to backend. `Host` header defaults to the upstream block name (e.g., `backend`) instead of the client's original `Host: example.com`. Backend with virtual host routing serves wrong content or returns 404.

**Fix:**
```nginx
location / {
    proxy_pass http://backend;
    proxy_set_header Host $host;                    # pass original Host
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

> ⚠️ **DevOps Gotcha:** These four `proxy_set_header` lines should be in EVERY `proxy_pass` location. Without them: wrong Host routing, no real client IP, no protocol detection. Create a reusable include file.

---

### EC-019 — WebSocket Connections Fail Through Nginx Proxy
**Category:** Proxy | **Severity:** High | **Env:** Prod

**Scenario:** WebSocket client connects to `wss://example.com/ws`. Nginx doesn't forward the `Upgrade` and `Connection` headers by default. WebSocket handshake fails — falls back to polling.

**Fix:**
```nginx
location /ws {
    proxy_pass http://websocket-backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_read_timeout 86400s;   # prevent WebSocket timeout (24h)
    proxy_send_timeout 86400s;
}
```

```
  WebSocket upgrade flow through Nginx:
  
  Client → Nginx → Backend
  GET /ws HTTP/1.1                     MUST forward:
  Upgrade: websocket          ──────→  Upgrade: websocket
  Connection: Upgrade         ──────→  Connection: upgrade
  
  Without these headers:
  Backend never sees upgrade request → returns normal HTTP response
  Client sees: "WebSocket handshake failed"
```

---

### EC-020 — gRPC Proxy Returns "upstream sent too big header"
**Category:** Proxy | **Severity:** High | **Env:** Prod

**Scenario:** gRPC over HTTP/2 through Nginx. gRPC metadata (headers) exceeds default buffer size. "upstream sent too big header while reading response header from upstream."

**Fix:**
```nginx
location / {
    grpc_pass grpc://backend:50051;
    grpc_buffer_size 64k;
    # Or for HTTP/2 proxy:
    proxy_buffer_size 64k;
    proxy_buffers 8 64k;
    proxy_busy_buffers_size 128k;
}
```

---

### EC-021 — Upstream `least_conn` Doesn't Account for Connection Weight
**Category:** Proxy | **Severity:** Medium | **Env:** Prod

**Scenario:** `least_conn` balancing across servers with different capacities. Sends to server with fewest connections — but a beefy 16-core server and a tiny 2-core server get similar traffic.

**Fix:**
```nginx
upstream backend {
    least_conn;
    server 10.0.0.1:8080 weight=8;   # powerful server
    server 10.0.0.2:8080 weight=1;   # small server
}
```

---

### EC-022 — `proxy_next_upstream` Retries Unsafe POST Requests
**Category:** Proxy | **Severity:** Critical | **Env:** Prod

**Scenario:** `proxy_next_upstream error timeout` causes Nginx to retry a failed POST request on the next upstream server. If the first server partially processed the request (e.g., payment), the retry creates a DUPLICATE transaction.

```
  Dangerous retry scenario:
  1. Client sends POST /payment (charge $100)
  2. Nginx → Backend A: processes payment, but response times out
  3. Nginx retries → Backend B: processes payment AGAIN → $200 charged!
```

**Fix:**
```nginx
# Disable retry for non-idempotent methods:
proxy_next_upstream error timeout http_502;
proxy_next_upstream_tries 2;
# Critical: limit to idempotent methods only:
location /api/payment {
    proxy_next_upstream off;   # NEVER retry payments
}
```

> ⚠️ **DevOps Gotcha:** `proxy_next_upstream` is ON by default for `error` and `timeout`. For any non-idempotent endpoint (payments, orders, writes), set `proxy_next_upstream off`. This is one of the most dangerous Nginx defaults for financial applications.

---

### EC-023 — `proxy_read_timeout` Kills Long-Running API Calls
**Category:** Proxy | **Severity:** High | **Env:** Prod

**Scenario:** Report generation API takes 120 seconds. Default `proxy_read_timeout` is 60s. After 60s Nginx returns 504 Gateway Timeout even though the backend is still working.

**Fix:**
```nginx
location /api/reports {
    proxy_pass http://backend;
    proxy_read_timeout 300s;    # 5 minutes for long operations
    proxy_connect_timeout 10s;
    proxy_send_timeout 60s;
}
```

---

### EC-024 — Connection Draining During Deployment — Active Requests Killed
**Category:** Proxy | **Severity:** High | **Env:** Prod

**Scenario:** Upstream backend removed from pool during deployment. Active requests on that backend are immediately terminated. Users see 502 errors.

**Fix:**
```nginx
# Open source Nginx: use graceful backend shutdown
# Backend should:
# 1. Stop accepting new connections
# 2. Finish processing active requests
# 3. Then exit
# Nginx will detect "connection refused" for new requests → route to other upstreams
```

---

### EC-025 — `ip_hash` Breaks Session Stickiness Behind CDN or Load Balancer
**Category:** Proxy | **Severity:** High | **Env:** Prod

**Scenario:** `ip_hash` for session stickiness. All requests come from CDN (CloudFront) — same source IP. All users routed to same backend → server overloaded, stickiness meaningless.

```
  Without CDN:     Client IP varies → ip_hash distributes well
  Behind CDN:      CDN IP is always the same → ALL traffic → one backend!
  
  Client A ─┐                    ┌─ Backend 1 ← ALL traffic!
  Client B ─┤→ CDN (1 IP) → Nginx → ip_hash
  Client C ─┘                    └─ Backend 2 ← no traffic
```

**Fix:**
```nginx
# Use X-Forwarded-For based hash instead:
upstream backend {
    hash $http_x_forwarded_for consistent;
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
}
# Or use cookie-based stickiness (Nginx Plus):
# sticky cookie srv_id expires=1h;
```

---

### EC-026 — `proxy_pass` to HTTPS Upstream Without `proxy_ssl_*` Directives
**Category:** Proxy | **Severity:** Medium | **Env:** Prod

**Scenario:** `proxy_pass https://backend-server;` — Nginx connects over TLS to backend. But it doesn't verify the backend's certificate by default. MITM between Nginx and backend is undetected.

**Fix:**
```nginx
location / {
    proxy_pass https://backend;
    proxy_ssl_verify on;
    proxy_ssl_trusted_certificate /etc/ssl/certs/backend-ca.pem;
    proxy_ssl_server_name on;   # send SNI header
}
```

---

### EC-027 — DNS Resolution Cached Forever for Upstream Names
**Category:** Proxy | **Severity:** High | **Env:** Prod

**Scenario:** `upstream` block with hostname: `server backend.service:8080;`. DNS resolved ONCE at config load. Backend IP changes (autoscaling, failover). Nginx keeps sending to old IP.

```
  Config load: DNS resolves backend.service → 10.0.0.5
  
  1 hour later: backend.service → 10.0.0.9 (new instance)
  Nginx still sends to: 10.0.0.5 ← STALE!
  
  Nginx only re-resolves on:
  - Config reload (nginx -s reload)
  - Using variable in proxy_pass with resolver directive
```

**Fix:**
```nginx
# Use variable-based proxy_pass for dynamic resolution:
resolver 10.0.0.2 valid=30s ipv6=off;   # cluster DNS
set $backend "http://backend.service:8080";
proxy_pass $backend;
# Now DNS is re-resolved every 30 seconds
```

> ⚠️ **DevOps Gotcha:** In Kubernetes and cloud environments, backend IPs change constantly. If using `upstream` blocks with hostnames, you MUST reload Nginx when backends change. Or use variable proxy_pass with `resolver`. This is the #1 Nginx issue in dynamic infrastructure.

---

### EC-028 — `keepalive` Connections to Upstream Not Reused
**Category:** Proxy | **Severity:** Medium | **Env:** Prod

**Scenario:** `keepalive 32` set in `upstream` block. But `proxy_http_version` defaults to 1.0 (no keep-alive). Connections are opened and closed per-request despite the pool.

**Fix:**
```nginx
upstream backend {
    server 10.0.0.1:8080;
    keepalive 32;
}
location / {
    proxy_pass http://backend;
    proxy_http_version 1.1;              # REQUIRED for keepalive
    proxy_set_header Connection "";       # REQUIRED — clear Connection header
}
```

---

### EC-029 — `proxy_buffering off` Causes Slow Client to Block Backend Worker
**Category:** Proxy | **Severity:** High | **Env:** Prod

**Scenario:** `proxy_buffering off` used for streaming responses. Slow client on mobile connection. Backend worker thread is held open until the slow client receives all data. Backend thread pool exhausted.

```
  With buffering ON (default):
  Backend → [Nginx buffer] → Client (slow)
  Backend freed immediately after response written to buffer.
  
  With buffering OFF:
  Backend → Client (slow)
  Backend worker BLOCKED until client finishes receiving!
```

> ⚠️ **DevOps Gotcha:** Only disable proxy buffering for SSE/streaming endpoints. For normal APIs, keep buffering ON. Nginx is designed to buffer backend responses and drip-feed them to slow clients, protecting your backend from being tied up.

---

### EC-030 — Upstream `backup` Server Never Activated
**Category:** Proxy | **Severity:** Medium | **Env:** Prod

**Scenario:** `server 10.0.0.3:8080 backup;` in upstream. Primary servers are overloaded (slow but not failing). Backup never activates because primary servers aren't returning errors — just slow.

> ⚠️ **DevOps Gotcha:** `backup` only activates when ALL non-backup servers are marked down (by `max_fails`). It does NOT activate on slow responses. For overflow handling, use `least_conn` with weighted servers instead.

---

## SSL / TLS & Certificates

### EC-031 — Certificate Chain Incomplete — Works in Browser, Fails in API Clients
**Category:** TLS | **Severity:** High | **Env:** Prod

**Scenario:** SSL certificate configured but intermediate CA not included. Browsers auto-fetch intermediates (AIA). API clients (curl, Python, Java) don't — they fail with "unable to verify the first certificate."

```
  Certificate chain:
  Root CA (trusted, in OS store)
    └─ Intermediate CA ← MUST be included in Nginx config!
         └─ Your cert (leaf)
  
  Browser: downloads missing intermediate automatically ✅
  curl/Python/Java: "SSL certificate problem: unable to get local issuer certificate" ❌
```

**Fix:**
```bash
# Concatenate cert + intermediate(s) in order:
cat your-cert.pem intermediate.pem > fullchain.pem
# In nginx.conf:
ssl_certificate /etc/ssl/fullchain.pem;
ssl_certificate_key /etc/ssl/private.key;
# Verify chain:
openssl s_client -connect example.com:443 -servername example.com
```

> ⚠️ **DevOps Gotcha:** Test your SSL setup with `curl https://example.com` (not a browser). If curl fails, your API clients and webhooks WILL fail too. Use `ssllabs.com/ssltest` for comprehensive chain validation.

---

### EC-032 — Let's Encrypt Certificate Renewal Fails — Nginx Serving Expired Cert
**Category:** TLS | **Severity:** Critical | **Env:** Prod

**Scenario:** Certbot renewal cron fails silently (permissions, DNS issue, port 80 blocked). Certificate expires. Nginx continues serving the expired cert — browsers show scary warning, API clients refuse to connect.

**Detection:**
```bash
# Check certificate expiry:
echo | openssl s_client -connect example.com:443 2>/dev/null | openssl x509 -noout -dates
# Check certbot renewal status:
certbot certificates
certbot renew --dry-run
```

**Fix:**
```bash
# Ensure renewal timer/cron works:
systemctl status certbot.timer
# Add monitoring alert on cert expiry < 14 days:
# Prometheus: probe_ssl_earliest_cert_expiry - time() < 14*86400
```

> ⚠️ **DevOps Gotcha:** Monitor certificate expiry with Prometheus blackbox_exporter or external services (UptimeRobot, Datadog). Certbot can fail silently for months — you won't know until the cert expires.

---

### EC-033 — HTTPS Redirect Loop — `$scheme` Returns "http" Behind Load Balancer
**Category:** TLS | **Severity:** High | **Env:** Prod

**Scenario:** TLS terminated at AWS ALB/CloudFlare. Nginx receives plain HTTP from the LB. `return 301 https://$host$request_uri` creates infinite redirect — Nginx always sees HTTP.

```
  Client → HTTPS → [ALB/CloudFlare] → HTTP → Nginx
  
  Nginx config: if ($scheme = http) { return 301 https://...; }
  Problem: $scheme is ALWAYS "http" because ALB sends plain HTTP!
  → Infinite redirect loop!
```

**Fix:**
```nginx
# Check X-Forwarded-Proto header instead:
server {
    listen 80;
    if ($http_x_forwarded_proto = "http") {
        return 301 https://$host$request_uri;
    }
}
```

---

### EC-034 — OCSP Stapling Fails — Causes TLS Handshake Delay
**Category:** TLS | **Severity:** Medium | **Env:** Prod

**Scenario:** `ssl_stapling on;` configured but OCSP responder is unreachable from the server (firewall blocks outbound). First connections are slow as clients fall back to checking OCSP directly.

**Detection:**
```bash
# Test OCSP stapling:
openssl s_client -connect example.com:443 -status
# Look for "OCSP Response Status: successful"
# If "no response sent" → stapling not working
```

**Fix:**
```nginx
ssl_stapling on;
ssl_stapling_verify on;
ssl_trusted_certificate /etc/ssl/chain.pem;
resolver 8.8.8.8 valid=300s;   # DNS resolver for OCSP lookup
```

---

### EC-035 — TLS 1.0/1.1 Still Enabled — Fails PCI DSS Compliance
**Category:** TLS | **Severity:** High | **Env:** Prod

**Scenario:** Default Nginx SSL settings include TLS 1.0/1.1. PCI DSS 3.2+ requires TLS 1.2 minimum. Security scan flags non-compliance.

**Fix:**
```nginx
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
ssl_prefer_server_ciphers on;
```

---

### EC-036 — SNI Not Configured — Wrong Certificate Served for Multi-Domain
**Category:** TLS | **Severity:** High | **Env:** Prod

**Scenario:** Multiple server blocks with different SSL certs on same IP. Client doesn't send SNI (old clients, some API tools). Nginx serves the DEFAULT server's certificate — wrong domain, cert mismatch.

```
  IP: 10.0.0.5
  ├─ server_name api.example.com   → cert for api.example.com
  └─ server_name www.example.com   → cert for www.example.com
  
  Client without SNI → Nginx serves default_server cert
  If default is api.example.com → www.example.com users see cert error
```

---

### EC-037 — SSL Session Cache Not Shared Across Workers — Repeat Handshakes
**Category:** TLS | **Severity:** Medium | **Env:** Prod

**Scenario:** Each Nginx worker has its own SSL session cache. Client reconnects, gets a different worker, must do full TLS handshake again. Increases latency and CPU.

**Fix:**
```nginx
ssl_session_cache shared:SSL:50m;   # 50MB shared across all workers
ssl_session_timeout 1d;
ssl_session_tickets on;
```

---

### EC-038 — `ssl_certificate_key` Has Wrong Permissions — Nginx Won't Start
**Category:** TLS | **Severity:** High | **Env:** Prod

**Scenario:** Private key file readable by `other` users. Some Nginx builds refuse to start (security check). Or worse — it starts, and anyone on the server can read your private key.

**Fix:**
```bash
chmod 600 /etc/ssl/private/server.key
chown root:root /etc/ssl/private/server.key
# Or for Nginx to read without root:
chown root:www-data /etc/ssl/private/server.key
chmod 640 /etc/ssl/private/server.key
```

---

### EC-039 — HTTP/2 Enabled But `proxy_http_version` Still 1.0 to Backend
**Category:** TLS | **Severity:** Medium | **Env:** Prod

**Scenario:** Frontend: `listen 443 ssl http2;` — clients use HTTP/2. Backend: `proxy_http_version` defaults to 1.0. Multiplexing benefit lost — Nginx opens a new connection per request to backend.

**Fix:**
```nginx
proxy_http_version 1.1;              # at minimum use HTTP/1.1 for keepalive
proxy_set_header Connection "";       # enable keepalive to upstream
# Note: Nginx open-source does NOT support HTTP/2 to upstream
# Nginx Plus supports grpc_pass for HTTP/2 backends
```

---

### EC-040 — HSTS Header Set With Long `max-age` — Can't Downgrade to HTTP
**Category:** TLS | **Severity:** Medium | **Env:** Prod

**Scenario:** `add_header Strict-Transport-Security "max-age=31536000; includeSubDomains"` deployed. Later, you need to serve a subdomain over HTTP (e.g., internal tool). HSTS cached in browsers prevents HTTP access for up to a year.

> ⚠️ **DevOps Gotcha:** Start with short `max-age` (300 seconds) and gradually increase. Don't add `includeSubDomains` until you've confirmed ALL subdomains support HTTPS. Preloading (`preload` flag) is essentially PERMANENT.

---

### EC-041 — Mixed Content After Partial HTTPS Migration
**Category:** TLS | **Severity:** Medium | **Env:** Prod

**Scenario:** Main site on HTTPS. But some assets (images, scripts) still referenced with `http://`. Browsers block mixed content — page partially loads or shows security warnings.

---

### EC-042 — Certificate PEM File Has Wrong Line Endings (Windows `\r\n`)
**Category:** TLS | **Severity:** Medium | **Env:** Both

**Scenario:** Certificate PEM file edited on Windows, has `\r\n` line endings. Nginx (or OpenSSL) fails to parse: "PEM_read_bio:no start line" or "bad base64 decode."

**Fix:**
```bash
dos2unix /etc/ssl/certs/server.pem
# Or:
sed -i 's/\r$//' /etc/ssl/certs/server.pem
```

---

### EC-043 — Diffie-Hellman Parameters Not Generated — Weak Key Exchange
**Category:** TLS | **Severity:** Medium | **Env:** Prod

**Scenario:** `ssl_dhparam` not configured. Nginx uses default 1024-bit DH parameters. Weak against Logjam attack. Security scanners flag it.

**Fix:**
```bash
openssl dhparam -out /etc/ssl/dhparam.pem 4096   # takes several minutes
```
```nginx
ssl_dhparam /etc/ssl/dhparam.pem;
```

---

### EC-044 — Client Certificate Authentication Fails With "400 No Required SSL Certificate"
**Category:** TLS | **Severity:** Medium | **Env:** Prod

**Scenario:** `ssl_verify_client on;` requires client certificate. API clients without a client cert get 400 error. Difficult to debug because the error looks like a request problem, not a TLS problem.

**Fix:**
```nginx
# Use optional verification for gradual rollout:
ssl_verify_client optional;
# Then check in location blocks:
if ($ssl_client_verify != "SUCCESS") {
    return 403 "Client certificate required";
}
```

---

### EC-045 — Wildcard Certificate Doesn't Cover Sub-Subdomains
**Category:** TLS | **Severity:** Medium | **Env:** Prod

**Scenario:** Wildcard cert for `*.example.com`. Covers `api.example.com` but NOT `v2.api.example.com`. Browsers show certificate mismatch error.

```
  *.example.com matches:
  ✅ api.example.com
  ✅ www.example.com
  ❌ v2.api.example.com
  ❌ example.com  (bare domain needs separate SAN entry)
```

---

## Headers, Buffers & Body Handling

### EC-046 — `add_header` in Child Block Removes Parent's Headers
**Category:** Headers | **Severity:** High | **Env:** Prod

**Scenario:** Security headers set in `server` block (`X-Frame-Options`, `CSP`). A `location` block adds `add_header X-Custom "value"`. ALL parent `add_header` directives are DROPPED for that location.

```
  server {
      add_header X-Frame-Options "DENY";          ← security header
      add_header X-Content-Type-Options "nosniff"; ← security header
      
      location /api {
          add_header X-Request-Id $request_id;     ← custom header
          # ⚠️ X-Frame-Options and X-Content-Type-Options are GONE here!
      }
  }
```

**Fix:**
```nginx
# Repeat ALL headers in every location that uses add_header:
location /api {
    add_header X-Frame-Options "DENY";
    add_header X-Content-Type-Options "nosniff";
    add_header X-Request-Id $request_id;
}
# Or use the ngx_headers_more module:
more_set_headers "X-Frame-Options: DENY";   # inherited properly
```

> ⚠️ **DevOps Gotcha:** This is one of the most surprising Nginx behaviors. If ANY `add_header` exists in a child block, ALL parent `add_header` directives are ignored. Use `nginx -T` and `curl -I` to verify headers on every location. Consider `headers-more-nginx-module` for proper inheritance.

---

### EC-047 — `client_max_body_size` Defaults to 1MB — File Uploads Fail
**Category:** Body | **Severity:** High | **Env:** Prod

**Scenario:** File upload endpoint behind Nginx. User uploads 5MB file. Nginx returns `413 Request Entity Too Large` before the request even reaches the backend.

**Fix:**
```nginx
# In server or location block:
client_max_body_size 50m;   # allow up to 50MB uploads
# Set to 0 to disable limit entirely (not recommended)
```

---

### EC-048 — Large Request Headers — "400 Bad Request: Request Header Or Cookie Too Large"
**Category:** Headers | **Severity:** High | **Env:** Prod

**Scenario:** Application uses large JWT tokens or many cookies. Total header size exceeds Nginx buffer. User sees "400 Bad Request" with a generic error page.

**Fix:**
```nginx
large_client_header_buffers 4 32k;   # 4 buffers of 32KB each
# Default is 4 8k — too small for large JWTs
```

---

### EC-049 — `proxy_set_header` Cleared by Subsequent Directive at Same Level
**Category:** Headers | **Severity:** Medium | **Env:** Prod

**Scenario:** `proxy_set_header` in `http` block for global settings. Another `proxy_set_header` in `server` block. The server-level directive REPLACES all http-level headers, not appends.

```
  http {
      proxy_set_header X-Real-IP $remote_addr;       ← global
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      
      server {
          proxy_set_header Host $host;                ← replaces ALL above!
          # X-Real-IP and X-Forwarded-For are GONE!
      }
  }
```

> ⚠️ **DevOps Gotcha:** Same inheritance rule as `add_header` — child blocks replace, not extend. Set ALL `proxy_set_header` directives at the same level, or use an `include` file.

---

### EC-050 — Response Body Truncated Due to Small `proxy_buffer_size`
**Category:** Buffers | **Severity:** High | **Env:** Prod

**Scenario:** Backend returns large response headers (Set-Cookie, X-Custom headers). `proxy_buffer_size` (for the first chunk of the response, typically headers) is too small. Nginx returns 502.

**Fix:**
```nginx
proxy_buffer_size 16k;           # buffer for response headers
proxy_buffers 8 16k;             # buffers for response body
proxy_busy_buffers_size 32k;     # max size of busy buffers
```

---

### EC-051 — `X-Forwarded-For` Chain Grows Infinitely Through Multiple Proxies
**Category:** Headers | **Severity:** Medium | **Env:** Prod

**Scenario:** Request passes through CDN → Load Balancer → Nginx. Each proxy appends to `X-Forwarded-For`. Header grows: `1.2.3.4, 10.0.0.1, 10.0.0.2`. Backend parses the wrong IP (last vs first).

```
  X-Forwarded-For chain:
  Client (1.2.3.4) → CDN → LB → Nginx → Backend
  
  Header at backend: X-Forwarded-For: 1.2.3.4, 172.16.0.1, 10.0.0.5
                                       ^real IP   ^CDN         ^LB
  
  Common mistake: backend reads LAST IP → gets LB IP, not client IP
  Correct: read FIRST IP, but only after validating trusted proxies
```

> ⚠️ **DevOps Gotcha:** Always configure `set_real_ip_from` for each trusted proxy in the chain. Use `real_ip_header X-Forwarded-For;` and `real_ip_recursive on;` to extract the true client IP.

---

### EC-052 — `chunked_transfer_encoding` Conflicts With Legacy Backend
**Category:** Body | **Severity:** Medium | **Env:** Prod

**Scenario:** Nginx sends chunked transfer encoding to backend (HTTP/1.1). Legacy backend doesn't understand chunked encoding and reads garbled data.

**Fix:**
```nginx
chunked_transfer_encoding off;
# Or buffer the full request before proxying:
proxy_request_buffering on;   # (default is on)
```

---

### EC-053 — `proxy_hide_header` Doesn't Hide Headers Added by Nginx Itself
**Category:** Headers | **Severity:** Low | **Env:** Both

**Scenario:** `proxy_hide_header X-Powered-By;` hides the header from the backend response. But if Nginx itself adds the header via `add_header`, `proxy_hide_header` doesn't affect it.

---

### EC-054 — Request Smuggling via Ambiguous `Content-Length` and `Transfer-Encoding`
**Category:** Security | **Severity:** Critical | **Env:** Prod

**Scenario:** Attacker sends request with BOTH `Content-Length` and `Transfer-Encoding: chunked`. Nginx interprets one, backend interprets the other. Request boundaries misaligned — attacker can smuggle a second request.

**Fix:**
```nginx
# Nginx handles this correctly by default in recent versions (1.21.1+)
# Ensure you're on a recent version:
nginx -v
# Additional protection:
proxy_http_version 1.1;   # less ambiguity than 1.0
```

---

### EC-055 — `sub_filter` Replacement Only Works on First Match
**Category:** Body | **Severity:** Medium | **Env:** Both

**Scenario:** Using `sub_filter` to replace all occurrences of `http://` with `https://` in response body. Only the first occurrence is replaced by default.

**Fix:**
```nginx
sub_filter 'http://' 'https://';
sub_filter_once off;   # replace ALL occurrences, not just the first
sub_filter_types text/html text/css application/javascript;
```

---

### EC-056 — `proxy_ignore_headers` Causes Stale Content
**Category:** Headers | **Severity:** Medium | **Env:** Prod

**Scenario:** `proxy_ignore_headers Cache-Control;` used to force Nginx to cache responses. Backend sends `Cache-Control: no-cache` for dynamic content. Nginx ignores it and serves stale cached data.

---

### EC-057 — `underscores_in_headers off` Drops Custom Headers With Underscores
**Category:** Headers | **Severity:** High | **Env:** Prod

**Scenario:** Application sends `X-Custom_Header: value`. Nginx default `underscores_in_headers off` silently DROPS headers with underscores. Backend never receives the header.

**Fix:**
```nginx
underscores_in_headers on;   # in server block
# Or rename headers to use hyphens: X-Custom-Header
```

> ⚠️ **DevOps Gotcha:** This catches many teams off-guard. Any header with underscore (common in legacy systems, AWS API Gateway) is silently dropped. Enable `underscores_in_headers on` or standardize on hyphens.

---

### EC-058 — `proxy_pass` Encodes URI Components Twice
**Category:** Headers | **Severity:** Medium | **Env:** Prod

**Scenario:** Request URL contains encoded characters: `/api/search?q=hello%20world`. Nginx decodes then re-encodes when proxying → backend receives `/api/search?q=hello%2520world` (double-encoded).

**Fix:**
```nginx
# Pass the original URI without decoding:
location /api/ {
    proxy_pass http://backend$request_uri;
}
# Or use $uri with explicit encoding handling
```

---

## Caching

### EC-059 — Cache Serves Stale Content After Backend Update
**Category:** Caching | **Severity:** High | **Env:** Prod

**Scenario:** Nginx `proxy_cache` enabled. Backend content updated. Cache still serves old content until expiry. Users see outdated data.

```
  Request flow with caching:
  
  Client → Nginx Cache → Backend
  
  If cache HIT:    Client ← Nginx (serves from cache, backend not contacted)
  If cache MISS:   Client ← Nginx ← Backend (response cached for next time)
  
  Problem: Cache TTL = 1 hour → stale content served for up to 1 hour
```

**Fix:**
```nginx
# Purge cache on deployment:
# Nginx Plus: PURGE method
# Open source: rm -rf /var/cache/nginx/* && nginx -s reload
# Or use cache bypass header:
proxy_cache_bypass $http_x_purge_cache;
```

---

### EC-060 — Cache Key Doesn't Include Query String — Wrong Content Served
**Category:** Caching | **Severity:** Critical | **Env:** Prod

**Scenario:** Custom `proxy_cache_key` that omits `$args`. `/api/users?page=1` and `/api/users?page=2` both return the same cached page 1 response.

**Fix:**
```nginx
proxy_cache_key "$scheme$request_method$host$request_uri";
# $request_uri includes query string
# Or explicitly:
proxy_cache_key "$scheme$host$uri$is_args$args";
```

---

### EC-061 — Cache Stores Personalized Responses — User A Sees User B's Data
**Category:** Caching | **Severity:** Critical | **Env:** Prod

**Scenario:** API returns user-specific data. Nginx caches the response. Next user gets cached response from first user. Data leak / privacy violation.

```
  User A: GET /api/profile → {"name":"Alice"} → CACHED
  User B: GET /api/profile → {"name":"Alice"} ← WRONG! Gets Alice's data!
```

**Fix:**
```nginx
# Never cache personalized responses:
proxy_cache_bypass $http_authorization $cookie_session;
proxy_no_cache $http_authorization $cookie_session;
# Or include auth token in cache key (rarely practical)
# Best: backend should set Cache-Control: private, no-store for personalized content
```

> ⚠️ **DevOps Gotcha:** This is a data-breach-level bug. NEVER cache responses that vary by user identity. Backend MUST send `Cache-Control: private` for user-specific content. Nginx should respect it — don't override with `proxy_ignore_headers Cache-Control`.

---

### EC-062 — `proxy_cache_valid` Overrides Backend `Cache-Control` Headers
**Category:** Caching | **Severity:** High | **Env:** Prod

**Scenario:** Backend sends `Cache-Control: max-age=60` (cache 1 minute). Nginx config has `proxy_cache_valid 200 1h` (cache 1 hour). Nginx honors its own directive, ignoring the backend — content stale for 59 extra minutes.

```
  Priority of cache duration directives:
  1. proxy_cache_valid (Nginx config) ← takes precedence!
  2. X-Accel-Expires (backend header)
  3. Cache-Control / Expires (backend headers) 
```

---

### EC-063 — Cache Stampede — All Caches Expire Simultaneously
**Category:** Caching | **Severity:** High | **Env:** Prod

**Scenario:** Popular endpoint cached with 5-minute TTL. At expiry, 1000 simultaneous requests all miss cache and flood the backend. Backend overwhelmed.

```
  Cache stampede (thundering herd):
  
  Cache HIT period:      1 request/5min to backend
  Cache expires at T=300: 1000 requests → ALL go to backend!
  
  ┌────────┐  1000 reqs  ┌──────────┐
  │ Nginx  │────────────→│ Backend  │ ← overwhelmed!
  │ MISS   │             │          │
  └────────┘             └──────────┘
```

**Fix:**
```nginx
# Use proxy_cache_lock — only ONE request goes to backend during cache fill:
proxy_cache_lock on;
proxy_cache_lock_timeout 5s;
proxy_cache_lock_age 5s;
# Or serve stale content while revalidating:
proxy_cache_use_stale updating;
proxy_cache_background_update on;
```

> ⚠️ **DevOps Gotcha:** `proxy_cache_lock on` + `proxy_cache_use_stale updating` is the production-grade combination. One request refreshes the cache, all others get the stale (but recent) cached response instantly.

---

### EC-064 — Cache Fills Disk — No `max_size` Set
**Category:** Caching | **Severity:** High | **Env:** Prod

**Scenario:** `proxy_cache_path` without `max_size`. Cache grows unbounded. Disk fills up. Nginx stops writing cache, logs fill with errors, potentially cascading into other services on the same disk.

**Fix:**
```nginx
proxy_cache_path /var/cache/nginx
    levels=1:2
    keys_zone=my_cache:100m
    max_size=10g              # ← ALWAYS set max_size
    inactive=60m
    use_temp_path=off;
```

---

### EC-065 — Cache Bypass Not Working — `$cookie_` Variable Empty
**Category:** Caching | **Severity:** Medium | **Env:** Prod

**Scenario:** `proxy_cache_bypass $cookie_nocache;` — expects a cookie named `nocache` to bypass cache. But cookie name is case-sensitive and the variable name must match exactly.

**Detection:**
```bash
curl -H "Cookie: nocache=1" https://example.com/api -v
# Check X-Cache-Status header (if configured)
```

---

### EC-066 — Cached 404 Responses Served After Content Is Created
**Category:** Caching | **Severity:** Medium | **Env:** Prod

**Scenario:** Request for `/new-page` returns 404 (not yet published). `proxy_cache_valid 404 1m` caches the 404. Page published seconds later. Users still see cached 404 for 1 minute.

**Fix:**
```nginx
# Don't cache error responses, or use very short TTL:
proxy_cache_valid 404 5s;
# Or don't cache 404 at all (default behavior if not explicitly set)
```

---

### EC-067 — `vary: Accept-Encoding` Causes Cache Fragmentation
**Category:** Caching | **Severity:** Medium | **Env:** Prod

**Scenario:** Backend sends `Vary: Accept-Encoding`. Nginx stores separate cache entries for gzip, br, and uncompressed. Cache hit rate drops dramatically because of key fragmentation.

**Fix:**
```nginx
# Normalize Accept-Encoding before caching:
proxy_cache_key "$scheme$host$request_uri $http_accept_encoding";
# Or let Nginx handle compression (remove Vary from cache key):
gzip on;
gzip_proxied any;
proxy_cache_key "$scheme$host$request_uri";   # ignore encoding in key
```

---

### EC-068 — `proxy_cache_use_stale` Serves Permanently Stale Content If Backend Dies
**Category:** Caching | **Severity:** High | **Env:** Prod

**Scenario:** `proxy_cache_use_stale error timeout http_500 http_502 http_503` configured. Backend goes down for 24 hours. Nginx serves increasingly stale cached content. Users think the site is working but see yesterday's data.

> ⚠️ **DevOps Gotcha:** `proxy_cache_use_stale` is a double-edged sword. It keeps your site "up" during backend failures — but users may see very stale data. Add monitoring to detect when stale serving exceeds a threshold.

---

### EC-069 — Microcaching (1s TTL) Breaks CSRF Token Validation
**Category:** Caching | **Severity:** High | **Env:** Prod

**Scenario:** Microcaching (cache everything for 1 second) for high-traffic site. HTML form contains CSRF token. Cached page served to different users — all get same CSRF token. Form submission fails (token mismatch).

**Fix:**
```nginx
# Exclude pages with CSRF from caching:
proxy_no_cache $http_cookie;   # if session cookie present, don't cache
# Or cache only truly static/API-token-free pages
```

---

### EC-070 — Cache-Control `no-store` Ignored When `proxy_ignore_headers` Is Set
**Category:** Caching | **Severity:** High | **Env:** Prod

**Scenario:** DevOps team sets `proxy_ignore_headers Cache-Control` to "improve performance." Backend sends `Cache-Control: no-store` for sensitive data (banking, health records). Nginx caches it anyway. Sensitive data cached on disk and served to wrong users.

> ⚠️ **DevOps Gotcha:** NEVER use `proxy_ignore_headers Cache-Control` without thoroughly auditing every backend endpoint. It's almost always wrong for APIs with mixed public/private data.

---

## Rate Limiting & Access Control

### EC-071 — Rate Limit Zone Too Small — Nginx Returns 503 on Startup
**Category:** Rate Limiting | **Severity:** High | **Env:** Prod

**Scenario:** `limit_req_zone $binary_remote_addr zone=api:1m rate=10r/s;` — 1MB zone. Each entry (~128 bytes for IPv4). 1MB ≈ 8000 IPs. If more than 8000 unique IPs appear, oldest entries evicted. Under extreme traffic, zone overflows and Nginx returns 503.

**Fix:**
```nginx
limit_req_zone $binary_remote_addr zone=api:50m rate=10r/s;
# 50MB ≈ 400,000 unique IPs — generous but not wasteful
```

---

### EC-072 — `limit_req burst` Drops Requests Instead of Queuing
**Category:** Rate Limiting | **Severity:** High | **Env:** Prod

**Scenario:** `limit_req zone=api burst=5;` (without `nodelay`). Excess requests are queued and processed slowly. API clients experience increasing latency as requests queue up.

```
  Without nodelay:
  Request 1-10: rate=10r/s → processed immediately
  Request 11-15: burst=5 → QUEUED, processed at 10r/s rate
  Request 16+: → 503 rejected
  
  Queued requests experience delay: req 11 waits 100ms, req 12 waits 200ms...
  
  With nodelay:
  Request 1-15: all processed IMMEDIATELY
  Request 16+: → 503 rejected
  Burst allows immediate processing without queuing delay
```

**Fix:**
```nginx
limit_req zone=api burst=20 nodelay;
# nodelay: process burst requests immediately, don't queue them
```

---

### EC-073 — Rate Limiting by IP Blocks All Users Behind Corporate NAT
**Category:** Rate Limiting | **Severity:** High | **Env:** Prod

**Scenario:** `$binary_remote_addr` rate limit. Large company with 5000 employees behind one NAT IP. All 5000 users share one rate limit. After 10 requests/second from the company, everyone gets 429.

```
  Corporate office (5000 users):
  All traffic exits via NAT IP: 203.0.113.50
  
  Rate limit: 10 req/s per IP
  → 10 req/s shared among 5000 users
  → Each user effectively gets 0.002 req/s!
```

**Fix:**
```nginx
# Use a combination of IP and other identifiers:
map $http_x_api_key $rate_limit_key {
    default $binary_remote_addr;
    ~.+     $http_x_api_key;   # if API key exists, rate limit per key
}
limit_req_zone $rate_limit_key zone=api:50m rate=100r/s;
```

---

### EC-074 — `allow/deny` Order Matters — First Match Wins
**Category:** Access | **Severity:** Critical | **Env:** Prod

**Scenario:** Admin wants to block one IP but allow all others. Places `allow all;` before `deny 1.2.3.4;`. The `allow all` matches first — attacker IP never denied.

```nginx
# WRONG — allow all matches first:
allow all;
deny 1.2.3.4;   # ← never reached!

# CORRECT — deny first, then allow rest:
deny 1.2.3.4;
allow all;

# Or for whitelist approach:
allow 10.0.0.0/8;
deny all;   # deny everyone except the allowed range
```

---

### EC-075 — `geo` Block Falls Back to Default for IPv6 Clients
**Category:** Access | **Severity:** Medium | **Env:** Prod

**Scenario:** `geo $blocked { 1.2.3.4 1; }` only has IPv4 rules. IPv6 clients hit the `default 0` — bypassing all geo-based blocks.

**Fix:**
```nginx
geo $blocked {
    default 0;
    1.2.3.4 1;
    2001:db8::/32 1;   # include IPv6 ranges
}
```

---

### EC-076 — `limit_conn` Doesn't Count WebSocket Connections Properly
**Category:** Rate Limiting | **Severity:** Medium | **Env:** Prod

**Scenario:** `limit_conn_zone $binary_remote_addr zone=addr:10m; limit_conn addr 10;` — limits 10 concurrent connections per IP. WebSocket connections are long-lived. Users hit the connection limit and can't open new tabs.

---

### EC-077 — Basic Auth Exposed Over HTTP — Credentials in Plaintext
**Category:** Access | **Severity:** Critical | **Env:** Prod

**Scenario:** `auth_basic` configured but accessible over HTTP (not HTTPS). Basic auth sends credentials base64-encoded (NOT encrypted) — visible to anyone sniffing the network.

**Fix:**
```nginx
server {
    listen 80;
    return 301 https://$host$request_uri;   # force HTTPS first
}
server {
    listen 443 ssl;
    auth_basic "Restricted";
    auth_basic_user_file /etc/nginx/.htpasswd;
}
```

---

### EC-078 — `satisfy any` Creates Unintended Access Bypass
**Category:** Access | **Severity:** Critical | **Env:** Prod

**Scenario:** `satisfy any;` with `auth_basic` and `allow/deny`. Intent: require BOTH password AND IP whitelist. Reality: `satisfy any` means EITHER one is sufficient. Password-only access from any IP is allowed.

```
  satisfy all:  BOTH access module AND auth must pass
  satisfy any:  EITHER access module OR auth is sufficient
  
  With satisfy any:
  ✅ Allowed IP, no password → access granted
  ✅ Any IP, correct password → access granted  ← unintended!
```

---

### EC-079 — Rate Limit Custom Error Page Not Returned — Generic 503
**Category:** Rate Limiting | **Severity:** Low | **Env:** Prod

**Scenario:** `limit_req` returns 503 by default. Custom `error_page 429` configured. But rate limiting returns 503, not 429 — custom page never triggered.

**Fix:**
```nginx
limit_req zone=api burst=20 nodelay;
limit_req_status 429;   # return 429 instead of 503
error_page 429 /rate-limited.html;
```

---

### EC-080 — JWT Validation in Nginx — Token Expired Still Passes
**Category:** Access | **Severity:** High | **Env:** Prod

**Scenario:** Using `auth_jwt` (Nginx Plus) or Lua-based JWT validation. JWT expiry (`exp` claim) not checked, or server clock is skewed. Expired tokens accepted.

**Fix:**
```bash
# Ensure NTP is synced:
timedatectl status   # check if NTP is active
# Configure auth_jwt with proper validation:
# Nginx Plus:
auth_jwt "Protected";
auth_jwt_key_file /etc/nginx/jwt-key.jwk;
```

---

### EC-081 — GeoIP Database Outdated — Wrong Country Detection
**Category:** Access | **Severity:** Medium | **Env:** Prod

**Scenario:** `geoip2` module with MaxMind database for geo-blocking. Database not updated in a year. New IP ranges assigned to different countries. Legitimate users blocked, attackers allowed.

**Fix:**
```bash
# Set up automatic GeoIP database updates:
# Install geoipupdate tool and configure cron
geoipupdate -v   # manual update
# Cron: 0 3 * * 3 /usr/bin/geoipupdate && nginx -s reload
```

---

### EC-082 — `limit_except` Doesn't Restrict as Expected
**Category:** Access | **Severity:** Medium | **Env:** Prod

**Scenario:** `limit_except GET { deny all; }` — intends to allow only GET. But `limit_except GET` also allows HEAD (HEAD is always allowed with GET in HTTP spec). POST, PUT, DELETE properly denied.

---

## Logging & Monitoring

### EC-083 — Access Log Fills Disk — No Log Rotation Configured
**Category:** Logging | **Severity:** High | **Env:** Prod

**Scenario:** High-traffic site. Access log grows 10GB/day. No logrotate configured. Disk fills. Nginx can't write logs, potentially can't write to cache or temp files — cascading failures.

**Fix:**
```bash
# /etc/logrotate.d/nginx:
/var/log/nginx/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0640 www-data adm
    sharedscripts
    postrotate
        [ -s /run/nginx.pid ] && kill -USR1 $(cat /run/nginx.pid)
    endscript
}
```

> ⚠️ **DevOps Gotcha:** The `kill -USR1` signal tells Nginx to reopen log files. Without it, Nginx keeps writing to the old (rotated) file descriptor. Logs appear empty after rotation.

---

### EC-084 — `error_log off` Doesn't Disable Error Logging
**Category:** Logging | **Severity:** Medium | **Env:** Both

**Scenario:** `error_log off;` — expects no error logging. Actually creates a file NAMED `off` in the Nginx prefix directory and logs errors there.

**Fix:**
```nginx
# Correct way to discard error logs:
error_log /dev/null crit;   # log only critical errors to /dev/null
# You can't fully disable error_log — it's required
```

---

### EC-085 — Conditional Logging Misses Important Requests
**Category:** Logging | **Severity:** Medium | **Env:** Prod

**Scenario:** `access_log /var/log/nginx/access.log combined if=$loggable;` with `map` to skip health checks. Map condition too broad — also skips legitimate API requests with similar patterns.

---

### EC-086 — `$upstream_response_time` Shows "-" for Cached Responses
**Category:** Logging | **Severity:** Low | **Env:** Prod

**Scenario:** Custom log format includes `$upstream_response_time` for latency monitoring. Cached responses never contact upstream — variable is `-`. Monitoring dashboard misinterprets `-` as zero or error.

**Fix:**
```nginx
# Use $request_time instead for total request time (includes cache lookup)
# Or handle "-" in your log parser:
log_format custom '$remote_addr - $request_time $upstream_response_time $upstream_cache_status';
```

---

### EC-087 — JSON Log Format Breaks With Unescaped Characters
**Category:** Logging | **Severity:** Medium | **Env:** Prod

**Scenario:** JSON-structured access logs: `log_format json '{"uri":"$request_uri",...}'`. Request URI contains quotes or backslashes. JSON output becomes invalid — log parsers (Fluentd, Filebeat) fail.

**Fix:**
```nginx
# Use escape=json (Nginx 1.11.8+):
log_format json escape=json '{"uri":"$request_uri","status":$status}';
```

---

### EC-088 — Syslog Destination Unreachable — Nginx Log Calls Block
**Category:** Logging | **Severity:** High | **Env:** Prod

**Scenario:** `access_log syslog:server=10.0.0.5:514;` — syslog server is down. TCP syslog can block Nginx worker processes. UDP syslog silently drops log messages.

**Fix:**
```nginx
# Use UDP for syslog (non-blocking):
access_log syslog:server=10.0.0.5:514,facility=local7,severity=info combined;
# UDP is fire-and-forget — won't block workers
```

---

### EC-089 — `stub_status` Exposed Publicly — Information Leak
**Category:** Monitoring | **Severity:** Medium | **Env:** Prod

**Scenario:** `stub_status` endpoint at `/nginx_status` without access restriction. Exposes active connections, request counts. Attackers use this for reconnaissance.

**Fix:**
```nginx
location /nginx_status {
    stub_status;
    allow 10.0.0.0/8;
    allow 127.0.0.1;
    deny all;
}
```

---

### EC-090 — Duplicate Access Log Entries — Logging Inherited
**Category:** Logging | **Severity:** Low | **Env:** Both

**Scenario:** `access_log` set in `http` block AND in `server` block. Both are active — each request logged twice. Log volume doubles.

```nginx
# access_log in http block: applied to ALL servers
# access_log in server block: ADDITIONAL log, not override
# To override, disable parent logging:
server {
    access_log /var/log/nginx/myapp.log;
    # Parent http-level log still active! Add:
    access_log /var/log/nginx/myapp.log combined;   # explicitly set only this log
}
```

---

## Performance & Tuning

### EC-091 — `worker_connections` Too Low — "worker_connections are not enough"
**Category:** Performance | **Severity:** Critical | **Env:** Prod

**Scenario:** Default `worker_connections 512`. Each proxied request uses 2 connections (client→nginx + nginx→upstream). Max concurrent requests = `worker_processes × worker_connections / 2`. On traffic spike, connections refused.

```
  Capacity calculation:
  worker_processes 4 × worker_connections 512 = 2048 total connections
  Reverse proxy: 2 connections per request → 1024 concurrent requests max
  
  With keepalive from clients: each idle tab holds a connection
  10,000 users with tabs open → 10,000 connections needed → 512 WAY too low!
```

**Fix:**
```nginx
events {
    worker_connections 4096;    # or higher for busy proxies
    multi_accept on;            # accept multiple connections at once
    use epoll;                  # Linux: most efficient event method
}
```

---

### EC-092 — Gzip Compression Not Applied — `Content-Type` Not in `gzip_types`
**Category:** Performance | **Severity:** Medium | **Env:** Prod

**Scenario:** `gzip on;` enabled. But `gzip_types` only includes `text/html` (default). JSON API responses (`application/json`) not compressed. 10x larger than necessary on the wire.

**Fix:**
```nginx
gzip on;
gzip_types
    text/plain
    text/css
    text/xml
    text/javascript
    application/json
    application/javascript
    application/xml
    application/xml+rss
    image/svg+xml;
gzip_min_length 256;
gzip_vary on;
gzip_proxied any;
gzip_comp_level 5;
```

---

### EC-093 — `sendfile` Off by Default in Docker — Static File Serving Slow
**Category:** Performance | **Severity:** Medium | **Env:** Prod

**Scenario:** Serving static files in Docker container. `sendfile` is on by default, but with overlayfs (Docker), `sendfile` can cause stale content issues. Some Docker Nginx images disable it.

**Fix:**
```nginx
sendfile on;
tcp_nopush on;
tcp_nodelay on;
# If serving from Docker volume mount and seeing stale files:
sendfile off;   # disable for Docker overlayfs edge cases
```

---

### EC-094 — Open File Cache Not Configured — Stat Syscall on Every Request
**Category:** Performance | **Severity:** Medium | **Env:** Prod

**Scenario:** Serving thousands of static files. Every request triggers a `stat()` syscall to check if the file exists and get metadata. Under high load, these syscalls become a bottleneck.

**Fix:**
```nginx
open_file_cache max=10000 inactive=30s;
open_file_cache_valid 60s;
open_file_cache_min_uses 2;
open_file_cache_errors on;
```

---

### EC-095 — Upstream Keepalive Set But Not Effective
**Category:** Performance | **Severity:** High | **Env:** Prod

**Scenario:** `keepalive 32` in upstream block. But `proxy_http_version` not set to 1.1 AND `Connection` header not cleared. Connections are closed after every request despite keepalive config.

```
  ┌─────────────────────────────────────────────────────────┐
  │ Three things MUST be set for upstream keepalive:        │
  │                                                         │
  │ 1. upstream { keepalive 32; }                           │
  │ 2. proxy_http_version 1.1;                              │
  │ 3. proxy_set_header Connection "";                      │
  │                                                         │
  │ Missing ANY one → connections closed per-request        │
  └─────────────────────────────────────────────────────────┘
```

---

### EC-096 — `proxy_buffering on` But Temp Directory on Slow Disk
**Category:** Performance | **Severity:** Medium | **Env:** Prod

**Scenario:** Large responses buffered to disk. Default temp directory on slow HDD. Under load, disk I/O becomes bottleneck — responses slow down dramatically.

**Fix:**
```nginx
proxy_temp_path /dev/shm/nginx_temp 1 2;   # use RAM-backed filesystem
# Or mount a fast SSD for temp files
proxy_max_temp_file_size 1024m;
```

---

### EC-097 — `client_body_timeout` Too Short — Large Uploads Fail
**Category:** Performance | **Severity:** Medium | **Env:** Prod

**Scenario:** Default `client_body_timeout 60s` — timeout between TWO successive read operations from client body, not total upload time. Slow client connections on mobile timeout during large file upload.

---

### EC-098 — Thread Pool Not Enabled — Blocking File I/O Stalls Workers
**Category:** Performance | **Severity:** Medium | **Env:** Prod

**Scenario:** Serving large files from HDD. Without thread pools, file read operations block the event loop. One slow disk read stalls ALL connections handled by that worker.

**Fix:**
```nginx
# Compile with --with-threads
# Configure thread pool:
aio threads;
# Or named pool:
thread_pool disk_pool threads=16 max_queue=65536;
location /downloads {
    aio threads=disk_pool;
    directio 4m;
    sendfile on;
}
```

---

### EC-099 — HTTP/2 Server Push Wastes Bandwidth — Removed in HTTP/3
**Category:** Performance | **Severity:** Low | **Env:** Prod

**Scenario:** HTTP/2 server push configured to push CSS/JS with HTML. Chrome removed support for HTTP/2 push (2022). Pushed resources wasted bandwidth — client ignores them.

> ⚠️ **DevOps Gotcha:** HTTP/2 server push is effectively dead. Chrome, Firefox have disabled it. Use `<link rel="preload">` and 103 Early Hints instead.

---

### EC-100 — `resolver` TTL Too Long — Upstream DNS Changes Not Detected
**Category:** Performance | **Severity:** High | **Env:** Prod

**Scenario:** `resolver 10.0.0.2 valid=300s;` — DNS cached for 5 minutes. In Kubernetes, pod IPs change within seconds during deployments. Nginx sends traffic to non-existent pods for up to 5 minutes.

**Fix:**
```nginx
resolver kube-dns.kube-system.svc.cluster.local valid=10s ipv6=off;
# 10 seconds is a reasonable trade-off: low DNS load, fast failover
```

---

## Containers, K8s & Cloud

### EC-101 — Nginx Container Runs as Root — Security Risk
**Category:** Container | **Severity:** High | **Env:** Prod

**Scenario:** Default Nginx Docker image runs as root. Kubernetes Pod Security Policy (or Standards) rejects it. Container fails to start in hardened clusters.

**Fix:**
```dockerfile
FROM nginx:1.25-alpine
# Run as non-root:
RUN chown -R nginx:nginx /var/cache/nginx /var/log/nginx /etc/nginx/conf.d
RUN sed -i 's/listen       80;/listen       8080;/' /etc/nginx/conf.d/default.conf
USER nginx
EXPOSE 8080
```

---

### EC-102 — Kubernetes Ingress `nginx.ingress.kubernetes.io/rewrite-target` Breaks Paths
**Category:** K8s | **Severity:** High | **Env:** Prod

**Scenario:** Ingress rewrite rule strips path prefix. Internal links and API paths break because they reference the original prefix. Frontend JavaScript makes requests to wrong paths.

```
  Ingress annotation:
  nginx.ingress.kubernetes.io/rewrite-target: /$2
  
  External: /app/dashboard → Nginx rewrites → Backend receives: /dashboard
  Backend serves HTML with link: <a href="/dashboard/settings">
  Browser requests: /dashboard/settings → Ingress doesn't match /app/ prefix → 404!
```

---

### EC-103 — ConfigMap Update Requires Pod Restart — Nginx Doesn't Reload
**Category:** K8s | **Severity:** Medium | **Env:** Prod

**Scenario:** Nginx config stored in ConfigMap. ConfigMap updated. Nginx pod still uses old config — ConfigMap volume updates eventually (kubelet sync), but Nginx doesn't know to reload.

**Fix:**
```yaml
# Use inotify-based sidecar to detect changes and trigger reload:
# Or use a deployment annotation hash trick:
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
# ConfigMap change → annotation changes → rolling restart triggered
```

---

### EC-104 — Nginx Ingress Controller OOM Killed Under High Traffic
**Category:** K8s | **Severity:** Critical | **Env:** Prod

**Scenario:** Nginx Ingress Controller pod has resource limits. During traffic spike, worker connections and buffers consume more memory than the limit. Kubernetes OOM-kills the pod. ALL ingress traffic drops.

**Fix:**
```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "2"
    memory: "2Gi"   # generous memory limit for buffer-heavy workloads
# Also tune Nginx settings to match:
# proxy_buffer_size, proxy_buffers — align with available memory
```

---

### EC-105 — Nginx Container Ignores `SIGTERM` — Slow Pod Termination
**Category:** Container | **Severity:** Medium | **Env:** Prod

**Scenario:** Kubernetes sends SIGTERM to stop pod. Nginx master process receives it but waits for workers to finish. If `worker_shutdown_timeout` is not set, workers can hang indefinitely. Pod killed with SIGKILL after `terminationGracePeriodSeconds`.

**Fix:**
```nginx
worker_shutdown_timeout 30s;   # force worker exit after 30s
```

---

### EC-106 — Health Check Endpoint Returns 200 But Upstream Is Down
**Category:** K8s | **Severity:** High | **Env:** Prod

**Scenario:** Kubernetes readiness probe hits `/healthz` on Nginx. Nginx returns 200 (it's running fine). But all upstream backends are dead. Nginx is "healthy" but returning 502 to every real request. Pod stays in rotation.

**Fix:**
```nginx
location /healthz {
    access_log off;
    # Check if upstream is actually reachable:
    proxy_pass http://backend/health;
    proxy_connect_timeout 2s;
    proxy_read_timeout 2s;
}
# Now readiness fails if backend is unreachable
```

---

### EC-107 — Nginx Ingress `proxy-body-size` Annotation Conflicts With Global Setting
**Category:** K8s | **Severity:** Medium | **Env:** Prod

**Scenario:** Global Ingress ConfigMap sets `proxy-body-size: 1m`. Individual Ingress resource sets `nginx.ingress.kubernetes.io/proxy-body-size: "50m"`. Annotation should override, but sometimes ConfigMap values stick due to controller caching.

**Fix:**
```bash
# Force controller config reload:
kubectl rollout restart deployment ingress-nginx-controller -n ingress-nginx
# Or check effective config:
kubectl exec -n ingress-nginx <controller-pod> -- cat /etc/nginx/nginx.conf | grep client_max_body
```

---

### EC-108 — Multiple Ingress Resources Conflict — Last Writer Wins
**Category:** K8s | **Severity:** High | **Env:** Prod

**Scenario:** Two teams create Ingress resources for the same host + path combination. Nginx Ingress Controller merges them unpredictably — one team's configuration overrides the other's. Inconsistent routing.

**Detection:**
```bash
kubectl get ingress --all-namespaces -o wide | grep example.com
# Check for conflicts:
kubectl logs -n ingress-nginx <controller-pod> | grep "conflict"
```

---

### EC-109 — Cloud Load Balancer Health Check Fails — Nginx Returns 301
**Category:** Cloud | **Severity:** High | **Env:** Prod

**Scenario:** AWS ALB health check targets port 80. Nginx redirects HTTP→HTTPS (301). ALB interprets 301 as unhealthy. All targets marked unhealthy. ALB returns 503 to all users.

**Fix:**
```nginx
# Return 200 for health check path even on HTTP:
location /health {
    access_log off;
    return 200 "ok\n";
}
# Redirect everything else:
location / {
    return 301 https://$host$request_uri;
}
# ALB health check targets /health on port 80
```

> ⚠️ **DevOps Gotcha:** When Nginx sits behind AWS ALB/NLB, the health check MUST return 200 on the health check path. Don't apply global HTTP→HTTPS redirects without carving out the health check path first.

---

### EC-110 — Nginx Proxy Protocol Misconfiguration — Client IP Shows as 127.0.0.1
**Category:** Cloud | **Severity:** High | **Env:** Prod

**Scenario:** AWS NLB with Proxy Protocol v2 enabled. Nginx not configured to accept Proxy Protocol. All client IPs show as the NLB IP (127.0.0.1 or private IP). Rate limiting and logging useless.

**Fix:**
```nginx
server {
    listen 80 proxy_protocol;   # accept Proxy Protocol
    listen 443 ssl proxy_protocol;
    
    set_real_ip_from 10.0.0.0/8;
    real_ip_header proxy_protocol;
}
```

---

## Production FAQ

**Q1: Nginx returns 502 Bad Gateway intermittently. How do I debug it?**  
A: Check `error.log` for upstream connection errors. Common causes: (1) Backend crashed — check backend health. (2) Connection pool stale — add `proxy_next_upstream error timeout http_502`. (3) Buffer too small for response headers — increase `proxy_buffer_size`. (4) Backend socket queue full — increase backend's `listen()` backlog.

**Q2: How do I achieve zero-downtime Nginx config changes?**  
A: `nginx -t && nginx -s reload` is already zero-downtime. The master process spawns new workers with new config, old workers finish existing requests then exit. For upstream changes (add/remove servers), reload is the only way in open-source Nginx.

**Q3: Nginx is using 100% CPU — what's wrong?**  
A: Check `worker_connections` (may be too low, causing tight loops). Check `access_log` — logging to a slow destination blocks workers. Check `gzip_comp_level` — level 9 is CPU-heavy. Check for regex-heavy `location` blocks or `map` directives. Use `perf top -p $(pgrep nginx)` to find hot functions.

**Q4: How do I handle WebSocket + HTTP in the same server block?**  
A: Use `map` to conditionally set upgrade headers: `map $http_upgrade $connection_upgrade { default upgrade; '' close; }`. Then apply `proxy_set_header Upgrade $http_upgrade; proxy_set_header Connection $connection_upgrade;` in the location block.

**Q5: My cache hit rate is below 30%. How do I improve it?**  
A: Check `proxy_cache_key` — remove unnecessary variables (cookies, random headers). Set `proxy_cache_min_uses 2` to only cache popular content. Use `proxy_cache_lock on` to prevent stampedes. Check `Vary` headers from backend — each unique combination creates a separate cache entry.

**Q6: How do I rate-limit by API key instead of IP address?**  
A: `limit_req_zone $http_x_api_key zone=api:50m rate=100r/s;`. Falls back to no limiting if header is absent. Combine with IP: `map $http_x_api_key $rate_key { default $binary_remote_addr; ~.+ $http_x_api_key; }`.

**Q7: Nginx serves stale static files after deployment. How to fix?**  
A: (1) Cache-busting filenames: `app.abc123.js`. (2) Add version query strings: `app.js?v=2.0`. (3) If using `open_file_cache`, reload Nginx to clear it. (4) In Docker with overlayfs, try `sendfile off`.

**Q8: How do I proxy to different backends based on URL path?**  
A: Use separate `location` blocks: `location /api/ { proxy_pass http://api-backend; }` and `location / { proxy_pass http://web-backend; }`. Remember the trailing slash rules for path rewriting.

**Q9: Connection timeouts to upstream after deploying new backend version?**  
A: Backend pods are being replaced. Old connections in Nginx's keepalive pool point to terminated pods. Fix: set `proxy_next_upstream error timeout` for automatic retry. In K8s, configure preStop hooks on backends to drain connections before shutting down.

**Q10: How do I configure Nginx as a TCP/UDP load balancer (not HTTP)?**  
A: Use the `stream` module (not `http`): `stream { upstream dns { server 10.0.0.1:53; } server { listen 53 udp; proxy_pass dns; } }`. Compiled with `--with-stream` module. Note: no HTTP-level features (headers, caching, rewriting) in stream context.

---

> **Next:** [Apache Edge Cases](apache_edge_cases.md) · [IIS Edge Cases](iis_edge_cases.md)  
> **Back:** [Git Edge Cases](git_edge_cases.md) · [Batch State](batch_state.md) · [🏠 Home](../README.md)
