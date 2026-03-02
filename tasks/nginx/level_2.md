# Nginx — Level 2 (🟡 Medium)

## 1. Caching JS Files Only (No Images)

### How Browser Caching Works

When Nginx sends a response, it can include cache headers that tell the browser how long to store the file locally. On subsequent requests, the browser uses the cached copy instead of downloading again:

```
First Visit:
┌────────┐   GET /app.js    ┌────────┐   Read from    ┌──────┐
│ Browser│──────────────────▶│ Nginx  │──────────────▶ │ Disk │
│        │◀──────────────────│        │◀────────────── │      │
│        │   200 OK + Cache  │        │                └──────┘
│        │   Headers         │        │
└───┬────┘                   └────────┘
    │ Store in cache
    ▼
┌────────┐
│ Cache  │
└────────┘

Subsequent Visits (within cache period):
┌────────┐   GET /app.js    ┌────────┐
│ Browser│─── ✗ NOT SENT ──▶│ Nginx  │  ← Not contacted at all!
│        │                   └────────┘
│        │◀──┐
└────────┘   │ Served from local cache
         ┌───┴───┐
         │ Cache  │
         └────────┘
```

### Server Block Configuration

```nginx
server {
    listen 80;
    server_name mysite.local;
    root /var/www/mysite;
    index index.html;

    # ─── Cache JavaScript files for 30 days ───
    location ~* \.js$ {
        expires 30d;
        add_header Cache-Control "public, max-age=2592000, immutable";
        add_header X-Cache-Rule "JS-30d";
    }

    # ─── Do NOT cache images — always fetch fresh ───
    location ~* \.(jpg|jpeg|png|gif|ico|svg|webp|bmp)$ {
        expires -1;
        add_header Cache-Control "no-store, no-cache, must-revalidate, proxy-revalidate, max-age=0";
        add_header X-Cache-Rule "Images-NoCache";
    }

    # ─── Default: no cache for everything else ───
    location / {
        add_header Cache-Control "no-cache";
        try_files $uri $uri/ =404;
    }
}
```

### Cache Headers Explanation

| Header / Directive | Value | Meaning |
|--------------------|-------|---------|
| `expires 30d` | 30 days | Sets the `Expires` header to 30 days from now |
| `Cache-Control: public` | — | Any cache (browser, CDN, proxy) may store the response |
| `max-age=2592000` | 30 days in seconds | Browser should use cached copy for this duration |
| `immutable` | — | Browser should NOT even revalidate during max-age (no conditional requests) |
| `expires -1` | Expired | Sets `Expires` to a past date — tells browser the content is already stale |
| `no-store` | — | Do not store the response in any cache at all |
| `no-cache` | — | Cache may store it but must revalidate with the server before using it |
| `must-revalidate` | — | Once stale, the cache must not use the response without revalidating |
| `proxy-revalidate` | — | Same as `must-revalidate` but for shared/proxy caches |

### Test and Verify

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Verify JavaScript caching:

```bash
curl -I http://mysite.local/js/main.js
```

Expected output:

```
HTTP/1.1 200 OK
Cache-Control: public, max-age=2592000, immutable
Expires: Wed, 01 Apr 2026 00:00:00 GMT
X-Cache-Rule: JS-30d
Content-Type: application/javascript
...
```

Verify images are NOT cached:

```bash
curl -I http://mysite.local/images/logo.png
```

Expected output:

```
HTTP/1.1 200 OK
Cache-Control: no-store, no-cache, must-revalidate, proxy-revalidate, max-age=0
Expires: Thu, 01 Jan 1970 00:00:01 GMT
X-Cache-Rule: Images-NoCache
Content-Type: image/png
...
```

---

---

## 2. Hiding Server Identity (Header Obfuscation)

### Why Hide Server Identity?

Exposing server software and version information makes it easier for attackers to find and exploit known vulnerabilities. Security best practices (including PCI-DSS) recommend removing or obfuscating server identification headers.

### What Leaks by Default

By default, Nginx exposes information in:

```
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)     ← Software + version + OS
X-Powered-By: Express              ← Backend framework (if proxying)
ETag: "5f3c8b2d-264"              ← Can reveal inode info on some setups
```

Error pages also show:

```html
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.18.0 (Ubuntu)</center>    ← Version in HTML
```

### Method 1: `server_tokens off` (Built-in)

The simplest approach — removes the version number from the `Server` header and error pages:

```nginx
# In http { }, server { }, or location { } context
http {
    server_tokens off;

    # ...
}
```

**Before:** `Server: nginx/1.18.0 (Ubuntu)`
**After:** `Server: nginx`

> **Limitation:** This still reveals that you're using Nginx. The `Server` header remains.

### Method 2: `headers-more` Module (Complete Removal)

The `ngx_headers_more` module allows you to clear or replace any header:

```bash
# Install the module
sudo apt install libnginx-mod-http-headers-more-filter -y
```

```nginx
http {
    server_tokens off;

    # Completely remove the Server header
    more_clear_headers 'Server';

    # Remove X-Powered-By (often added by backend frameworks)
    more_clear_headers 'X-Powered-By';

    # Or set a fake Server header to mislead scanners
    # more_set_headers 'Server: CustomServer';

    # ...
}
```

### Method 3: Custom Error Pages

Default error pages show the Nginx version. Replace them with custom pages:

```nginx
server {
    # ...

    # Custom error pages that don't reveal server info
    error_page 404 /custom_404.html;
    error_page 403 /custom_403.html;
    error_page 500 502 503 504 /custom_50x.html;

    location = /custom_404.html {
        root /var/www/error-pages;
        internal;
    }

    location = /custom_403.html {
        root /var/www/error-pages;
        internal;
    }

    location = /custom_50x.html {
        root /var/www/error-pages;
        internal;
    }
}
```

### Method 4: Disable ETag

ETag headers can sometimes leak information (e.g., inode numbers in Apache, but less of a concern in Nginx):

```nginx
http {
    etag off;

    # ...
}
```

### Combined Configuration

```nginx
http {
    server_tokens off;
    more_clear_headers 'Server';
    more_clear_headers 'X-Powered-By';
    etag off;

    # ...
}
```

### Verify

```bash
sudo nginx -t
sudo systemctl reload nginx
```

```bash
# Check response headers
curl -I http://mysite.local
```

Before hardening:

```
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
ETag: "5f3c8b2d-264"
```

After hardening:

```
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 612
```

No `Server`, no `X-Powered-By`, no `ETag` — attacker cannot identify the software.

```bash
# Test error page — should not reveal Nginx version
curl -I http://mysite.local/nonexistent-page
```

---

---

## 3. Custom Error Pages (404, 403, 502)

### Create Error Page HTML Files

```bash
sudo mkdir -p /var/www/error-pages
```

**404 — Not Found (Purple Gradient):**

```bash
cat << 'ERROREOF' | sudo tee /var/www/error-pages/404.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>404 — Page Not Found</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            min-height: 100vh;
            display: flex;
            align-items: center;
            justify-content: center;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            text-align: center;
        }
        .container { padding: 2rem; }
        .error-code { font-size: 8rem; font-weight: 700; line-height: 1; opacity: 0.9; }
        .error-title { font-size: 2rem; margin: 1rem 0; }
        .error-message { font-size: 1.1rem; opacity: 0.85; margin-bottom: 2rem; max-width: 500px; }
        .btn {
            display: inline-block;
            padding: 0.8rem 2rem;
            margin: 0.5rem;
            border-radius: 50px;
            text-decoration: none;
            font-weight: 600;
            transition: transform 0.2s, box-shadow 0.2s;
        }
        .btn-home { background: white; color: #764ba2; }
        .btn-back { background: rgba(255,255,255,0.2); color: white; border: 2px solid white; }
        .btn:hover { transform: translateY(-2px); box-shadow: 0 5px 20px rgba(0,0,0,0.3); }
    </style>
</head>
<body>
    <div class="container">
        <div class="error-code">404</div>
        <h1 class="error-title">Page Not Found</h1>
        <p class="error-message">
            The page you're looking for doesn't exist or has been moved.
            Check the URL or navigate back home.
        </p>
        <a href="/" class="btn btn-home">Go Home</a>
        <a href="javascript:history.back()" class="btn btn-back">Go Back</a>
    </div>
</body>
</html>
ERROREOF
```

**403 — Forbidden (Red-Orange Gradient):**

```bash
cat << 'ERROREOF' | sudo tee /var/www/error-pages/403.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>403 — Access Denied</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            min-height: 100vh;
            display: flex;
            align-items: center;
            justify-content: center;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #f093fb 0%, #f5576c 50%, #ff6a00 100%);
            color: white;
            text-align: center;
        }
        .container { padding: 2rem; }
        .error-code { font-size: 8rem; font-weight: 700; line-height: 1; opacity: 0.9; }
        .error-title { font-size: 2rem; margin: 1rem 0; }
        .error-message { font-size: 1.1rem; opacity: 0.85; margin-bottom: 2rem; max-width: 500px; }
        .btn {
            display: inline-block;
            padding: 0.8rem 2rem;
            margin: 0.5rem;
            border-radius: 50px;
            text-decoration: none;
            font-weight: 600;
            transition: transform 0.2s, box-shadow 0.2s;
        }
        .btn-home { background: white; color: #f5576c; }
        .btn-back { background: rgba(255,255,255,0.2); color: white; border: 2px solid white; }
        .btn:hover { transform: translateY(-2px); box-shadow: 0 5px 20px rgba(0,0,0,0.3); }
    </style>
</head>
<body>
    <div class="container">
        <div class="error-code">403</div>
        <h1 class="error-title">Access Denied</h1>
        <p class="error-message">
            You don't have permission to access this resource.
            If you believe this is an error, contact the administrator.
        </p>
        <a href="/" class="btn btn-home">Go Home</a>
        <a href="javascript:history.back()" class="btn btn-back">Go Back</a>
    </div>
</body>
</html>
ERROREOF
```

**502 — Bad Gateway (Dark Teal Gradient):**

```bash
cat << 'ERROREOF' | sudo tee /var/www/error-pages/502.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>502 — Bad Gateway</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            min-height: 100vh;
            display: flex;
            align-items: center;
            justify-content: center;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #0f2027 0%, #203a43 50%, #2c5364 100%);
            color: white;
            text-align: center;
        }
        .container { padding: 2rem; }
        .error-code { font-size: 8rem; font-weight: 700; line-height: 1; opacity: 0.9; }
        .error-title { font-size: 2rem; margin: 1rem 0; }
        .error-message { font-size: 1.1rem; opacity: 0.85; margin-bottom: 2rem; max-width: 500px; }
        .btn {
            display: inline-block;
            padding: 0.8rem 2rem;
            margin: 0.5rem;
            border-radius: 50px;
            text-decoration: none;
            font-weight: 600;
            transition: transform 0.2s, box-shadow 0.2s;
        }
        .btn-retry { background: #2ecc71; color: white; }
        .btn-home { background: rgba(255,255,255,0.2); color: white; border: 2px solid white; }
        .btn:hover { transform: translateY(-2px); box-shadow: 0 5px 20px rgba(0,0,0,0.3); }
    </style>
</head>
<body>
    <div class="container">
        <div class="error-code">502</div>
        <h1 class="error-title">Bad Gateway</h1>
        <p class="error-message">
            The server received an invalid response from an upstream server.
            This is usually temporary — please try again in a moment.
        </p>
        <a href="javascript:location.reload()" class="btn btn-retry">Retry</a>
        <a href="/" class="btn btn-home">Go Home</a>
    </div>
</body>
</html>
ERROREOF
```

### Set Permissions

```bash
sudo chown -R www-data:www-data /var/www/error-pages
sudo chmod 644 /var/www/error-pages/*.html
```

### Configure Nginx

```bash
sudo nano /etc/nginx/sites-available/mysite
```

```nginx
server {
    listen 80;
    server_name mysite.local;
    root /var/www/mysite;
    index index.html;

    # ─── Custom Error Pages ───
    error_page 404 /404.html;
    error_page 403 /403.html;
    error_page 502 /502.html;

    location = /404.html {
        root /var/www/error-pages;
        internal;    # Only accessible via internal redirects, not directly by URL
    }

    location = /403.html {
        root /var/www/error-pages;
        internal;
    }

    location = /502.html {
        root /var/www/error-pages;
        internal;
    }

    location / {
        try_files $uri $uri/ =404;
    }
}
```

> **Note:** The `internal` directive ensures error pages can only be served through Nginx's internal error handling, not by directly visiting `/404.html`.

### Verify

```bash
sudo nginx -t
sudo systemctl reload nginx
```

```bash
# Test 404 — request a page that doesn't exist
curl http://mysite.local/this-does-not-exist
# Expected: styled 404 page HTML with purple gradient

# Test 403 — try to access a forbidden location
curl http://mysite.local/admin
# Expected: styled 403 page (if /admin is denied)

# Test that error pages are not directly accessible
curl -I http://mysite.local/404.html
# Expected: 404 (the internal directive blocks direct access)
```

---

---

## 4. Nginx as a Reverse Proxy

### What is a Reverse Proxy?

A reverse proxy sits between clients and backend servers, forwarding requests and returning responses. The client only communicates with Nginx — never directly with the backend.

```
                    ┌──────────────────────────────────────────────┐
                    │              Nginx (Reverse Proxy)            │
                    │                                              │
┌────────┐         │  ┌──────────┐      ┌──────────────────────┐  │
│ Client  │────────▶│  │  Listen  │─────▶│  proxy_pass to       │  │
│(Browser)│         │  │  :80     │      │  backend server(s)   │  │
│         │◀────────│  │  :443    │◀─────│                      │  │
└────────┘         │  └──────────┘      └──────────────────────┘  │
                    │                            │                  │
                    └────────────────────────────│──────────────────┘
                                                 │
                    ┌────────────────────────────▼──────────────────┐
                    │             Backend Servers                    │
                    │  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
                    │  │ Node.js  │  │  Python  │  │  Static  │   │
                    │  │ :3000    │  │  :8000   │  │  Files   │   │
                    │  └──────────┘  └──────────┘  └──────────┘   │
                    └──────────────────────────────────────────────┘
```

**Why use a reverse proxy?**

- **SSL Termination** — Handle HTTPS at the proxy, backends use plain HTTP
- **Load Balancing** — Distribute traffic across multiple backends
- **Caching** — Cache responses to reduce backend load
- **Security** — Hide backend infrastructure, add rate limiting, WAF rules
- **Compression** — Gzip responses before sending to client
- **Static Files** — Serve static files directly without hitting the backend

### Step 1 — Create a Node.js Backend App

```bash
# Install Node.js (if not installed)
sudo apt install nodejs npm -y

# Create app directory
mkdir -p ~/myapp
cd ~/myapp
```

```bash
cat > ~/myapp/app.js << 'EOF'
const http = require('http');

const PORT = 3000;

const server = http.createServer((req, res) => {
    const response = {
        message: 'Hello from Node.js backend!',
        path: req.url,
        method: req.method,
        headers: req.headers,
        timestamp: new Date().toISOString(),
        server: `Backend running on port ${PORT}`
    };

    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify(response, null, 2));
});

server.listen(PORT, '127.0.0.1', () => {
    console.log(`Backend server running at http://127.0.0.1:${PORT}`);
});
EOF
```

### Step 2 — Run the Backend

```bash
# Run in background
node ~/myapp/app.js &

# Verify it's running
curl http://127.0.0.1:3000
```

### Step 3 — Configure Nginx as Reverse Proxy

```bash
sudo nano /etc/nginx/sites-available/mysite
```

```nginx
server {
    listen 80;
    server_name mysite.local;

    # ─── Reverse Proxy to Node.js Backend ───
    location / {
        proxy_pass http://127.0.0.1:3000;

        # ─── Pass original client information to backend ───
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # ─── WebSocket support ───
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection "upgrade";

        # ─── Timeouts ───
        proxy_connect_timeout 60s;
        proxy_send_timeout    60s;
        proxy_read_timeout    60s;
    }
}
```

### `proxy_set_header` Explanation

| Header | Value | Purpose |
|--------|-------|---------|
| `Host` | `$host` | Passes the original `Host` header so the backend knows which domain was requested |
| `X-Real-IP` | `$remote_addr` | Passes the actual client IP (without this, backend sees Nginx's IP: 127.0.0.1) |
| `X-Forwarded-For` | `$proxy_add_x_forwarded_for` | Appends client IP to the `X-Forwarded-For` chain (supports multiple proxies) |
| `X-Forwarded-Proto` | `$scheme` | Tells backend whether the original request was HTTP or HTTPS |
| `Upgrade` | `$http_upgrade` | Required for WebSocket connections — passes the protocol upgrade request |
| `Connection` | `"upgrade"` | Works with `Upgrade` header to establish WebSocket connections |

### Step 4 — Enable and Test

```bash
sudo nginx -t
sudo systemctl reload nginx

# Test the reverse proxy
curl http://mysite.local
```

Expected — the response comes from Node.js but served through Nginx:

```json
{
  "message": "Hello from Node.js backend!",
  "path": "/",
  "method": "GET",
  "headers": {
    "host": "mysite.local",
    "x-real-ip": "127.0.0.1",
    "x-forwarded-for": "127.0.0.1",
    "x-forwarded-proto": "http"
  },
  "timestamp": "2026-03-02T10:00:00.000Z",
  "server": "Backend running on port 3000"
}
```

### Path-Based Routing (Multiple Backends)

Route different URL paths to different backend services:

```nginx
# ─── Upstream Definitions ───
upstream node_backend {
    server 127.0.0.1:3000;
}

upstream python_backend {
    server 127.0.0.1:8000;
}

server {
    listen 80;
    server_name mysite.local;

    # ─── API requests → Node.js backend ───
    location /api/ {
        proxy_pass http://node_backend;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # ─── Admin panel → Python backend ───
    location /admin/ {
        proxy_pass http://python_backend;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # ─── Static files → served directly by Nginx ───
    location /static/ {
        alias /var/www/mysite/static/;
        expires 30d;
    }

    # ─── Frontend SPA → serve index.html for all routes ───
    location / {
        root /var/www/mysite;
        try_files $uri $uri/ /index.html;
    }
}
```

---

---

## 5. Load Balancing with Nginx

### Load Balancing Methods

| Method | Directive | How It Works |
|--------|-----------|-------------|
| **Round Robin** | (default) | Requests are distributed evenly across servers in order |
| **Least Connections** | `least_conn` | Request goes to the server with the fewest active connections |
| **IP Hash** (Sticky) | `ip_hash` | Same client IP always goes to the same server (session persistence) |
| **Weighted** | `weight=N` | Servers with higher weight receive proportionally more requests |
| **Random** | `random` | Requests are sent to a randomly selected server |

### Step 1 — Start Multiple Backends

Create backends on different ports:

```bash
for port in 3001 3002 3003; do
cat > /tmp/backend-${port}.js << EOF
const http = require('http');
const server = http.createServer((req, res) => {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({
        server: 'Backend ${port}',
        port: ${port},
        path: req.url,
        timestamp: new Date().toISOString()
    }));
});
server.listen(${port}, '127.0.0.1', () => console.log('Running on ${port}'));
EOF
node /tmp/backend-${port}.js &
done
```

Verify all backends are running:

```bash
curl http://127.0.0.1:3001
curl http://127.0.0.1:3002
curl http://127.0.0.1:3003
```

### Step 2 — Round Robin (Default)

```nginx
upstream backend_pool {
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003;
}

server {
    listen 80;
    server_name mysite.local;

    location / {
        proxy_pass http://backend_pool;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    }
}
```

### Step 3 — Weighted Load Balancing

Give more traffic to more powerful servers:

```nginx
upstream backend_pool {
    server 127.0.0.1:3001 weight=5;   # Gets 5x more traffic
    server 127.0.0.1:3002 weight=3;   # Gets 3x more traffic
    server 127.0.0.1:3003 weight=1;   # Gets 1x traffic (baseline)
}
```

### Step 4 — Least Connections

Best when request processing times vary:

```nginx
upstream backend_pool {
    least_conn;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003;
}
```

### Step 5 — IP Hash (Sticky Sessions)

Ensures the same client always reaches the same backend (useful for sessions):

```nginx
upstream backend_pool {
    ip_hash;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003;
}
```

### Step 6 — Health Checks and Failover

```nginx
upstream backend_pool {
    server 127.0.0.1:3001 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:3002 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:3003 max_fails=3 fail_timeout=30s backup;
    server 127.0.0.1:3004 down;    # Manually marked as unavailable
}
```

### Health Check / Failover Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `max_fails` | `1` | Number of failed attempts before marking server as unavailable |
| `fail_timeout` | `10s` | Time period for `max_fails` attempts AND how long to mark server unavailable |
| `backup` | — | Server is only used when all primary servers are unavailable |
| `down` | — | Permanently marks server as unavailable (for maintenance) |
| `weight` | `1` | Relative weight for weighted load balancing |

### Verify Load Balancing

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Send multiple requests and observe which backend handles each:

```bash
for i in $(seq 1 9); do
    echo "Request $i: $(curl -s http://mysite.local | grep -o '"server":"[^"]*"')"
done
```

Expected output (round robin):

```
Request 1: "server":"Backend 3001"
Request 2: "server":"Backend 3002"
Request 3: "server":"Backend 3003"
Request 4: "server":"Backend 3001"
Request 5: "server":"Backend 3002"
Request 6: "server":"Backend 3003"
Request 7: "server":"Backend 3001"
Request 8: "server":"Backend 3002"
Request 9: "server":"Backend 3003"
```

---

---

## 6. SSL/TLS with Let's Encrypt (Certbot)

### Prerequisites

- A **registered domain name** pointing to your server's public IP
- Ports **80** and **443** open in the firewall
- Nginx running and serving the domain over HTTP

### Step 1 — Install Certbot

```bash
# Install Certbot and the Nginx plugin
sudo apt install certbot python3-certbot-nginx -y
```

### Step 2 — Configure Server Block for the Domain

Ensure you have a server block for your domain:

```bash
sudo nano /etc/nginx/sites-available/mysite
```

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name yourdomain.com www.yourdomain.com;

    root /var/www/mysite;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### Step 3 — Obtain SSL Certificate

```bash
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

Certbot will:

1. Verify domain ownership (HTTP-01 challenge)
2. Generate SSL certificate
3. **Automatically modify your Nginx config** to add SSL settings
4. Set up HTTP → HTTPS redirect

You'll be prompted for:
- Email address (for renewal notifications)
- Agreement to Terms of Service
- Whether to redirect HTTP to HTTPS (choose **2 - Redirect**)

### Step 4 — Verify Auto-Renewal

Let's Encrypt certificates expire every 90 days. Certbot sets up automatic renewal:

```bash
# Test renewal process (doesn't actually renew)
sudo certbot renew --dry-run
```

Check the systemd timer:

```bash
sudo systemctl status certbot.timer
```

```bash
# List all timers
sudo systemctl list-timers | grep certbot
```

### What Certbot Added to Your Config

After running Certbot, your config will look like:

```nginx
server {
    server_name yourdomain.com www.yourdomain.com;

    root /var/www/mysite;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    listen [::]:443 ssl;                                    # ← Added
    listen 443 ssl;                                         # ← Added
    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;   # ← Added
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem; # ← Added
    include /etc/letsencrypt/options-ssl-nginx.conf;        # ← Added
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;          # ← Added
}

# HTTP → HTTPS redirect (added by Certbot)
server {
    if ($host = www.yourdomain.com) {
        return 301 https://$host$request_uri;
    }

    if ($host = yourdomain.com) {
        return 301 https://$host$request_uri;
    }

    listen 80;
    listen [::]:80;
    server_name yourdomain.com www.yourdomain.com;
    return 404;
}
```

### Certificate File Locations

| File | Path | Description |
|------|------|-------------|
| **Full Chain** | `/etc/letsencrypt/live/yourdomain.com/fullchain.pem` | Certificate + intermediate certificates |
| **Private Key** | `/etc/letsencrypt/live/yourdomain.com/privkey.pem` | Private key (keep secure!) |
| **Certificate** | `/etc/letsencrypt/live/yourdomain.com/cert.pem` | Server certificate only |
| **Chain** | `/etc/letsencrypt/live/yourdomain.com/chain.pem` | Intermediate certificates only |
| **SSL Options** | `/etc/letsencrypt/options-ssl-nginx.conf` | Certbot's recommended SSL settings |
| **DH Params** | `/etc/letsencrypt/ssl-dhparams.pem` | Diffie-Hellman parameters |

### Manual Renewal

```bash
# Renew all certificates
sudo certbot renew

# Renew a specific certificate
sudo certbot certonly --nginx -d yourdomain.com -d www.yourdomain.com

# Force renewal (even if not expiring soon)
sudo certbot renew --force-renewal
```

### Verify HTTPS

```bash
# Test HTTPS
curl -I https://yourdomain.com

# Test HTTP redirects to HTTPS
curl -I http://yourdomain.com
# Expected: 301 redirect to https://yourdomain.com

# Check certificate details
echo | openssl s_client -connect yourdomain.com:443 -servername yourdomain.com 2>/dev/null | openssl x509 -noout -dates
```

---

---

## 7. Log Configuration and Analysis

### Default Log Format (combined)

Nginx uses the `combined` log format by default:

```nginx
log_format combined '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent"';
```

Example log entry:

```
192.168.1.100 - - [02/Mar/2026:10:15:30 +0000] "GET /index.html HTTP/1.1" 200 612 "-" "Mozilla/5.0 ..."
```

### Custom Log Formats

#### Detailed Format with Timing

Add to the `http { }` block in `/etc/nginx/nginx.conf`:

```nginx
log_format detailed '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    'rt=$request_time uct=$upstream_connect_time '
                    'uht=$upstream_header_time urt=$upstream_response_time';
```

Example output:

```
192.168.1.100 - - [02/Mar/2026:10:15:30 +0000] "GET /api/data HTTP/1.1" 200 1234 "-" "curl/7.81.0" rt=0.152 uct=0.001 uht=0.150 urt=0.150
```

#### JSON Format (for ELK Stack / Log Aggregators)

```nginx
log_format json_combined escape=json
    '{'
        '"time": "$time_iso8601",'
        '"remote_addr": "$remote_addr",'
        '"remote_user": "$remote_user",'
        '"request": "$request",'
        '"status": "$status",'
        '"body_bytes_sent": "$body_bytes_sent",'
        '"request_time": "$request_time",'
        '"http_referrer": "$http_referer",'
        '"http_user_agent": "$http_user_agent",'
        '"upstream_addr": "$upstream_addr",'
        '"upstream_response_time": "$upstream_response_time"'
    '}';
```

Example output:

```json
{"time": "2026-03-02T10:15:30+00:00","remote_addr": "192.168.1.100","remote_user": "","request": "GET /api/data HTTP/1.1","status": "200","body_bytes_sent": "1234","request_time": "0.152","http_referrer": "","http_user_agent": "curl/7.81.0","upstream_addr": "127.0.0.1:3000","upstream_response_time": "0.150"}
```

### Per-Site Logging

Each server block can have its own log files:

```nginx
server {
    server_name site-a.local;

    access_log /var/log/nginx/site-a-access.log detailed;
    error_log  /var/log/nginx/site-a-error.log warn;

    # ...
}

server {
    server_name site-b.local;

    access_log /var/log/nginx/site-b-access.log json_combined;
    error_log  /var/log/nginx/site-b-error.log;

    # ...
}
```

### Conditional Logging

Skip logging for health checks and static files to keep logs clean:

```nginx
# Define a map to set a variable based on conditions
map $request_uri $loggable {
    ~*\.(ico|css|js|gif|jpg|jpeg|png|svg|woff2?)$ 0;   # Skip static assets
    /health                                         0;   # Skip health checks
    /ping                                           0;   # Skip ping endpoint
    default                                         1;   # Log everything else
}

server {
    # Only log when $loggable is 1
    access_log /var/log/nginx/mysite-access.log combined if=$loggable;
    error_log  /var/log/nginx/mysite-error.log;

    # ...
}
```

### Analyzing Logs from the Command Line

**Top 10 IP addresses by request count:**

```bash
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10
```

**Top 10 requested URLs:**

```bash
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10
```

**Response status code distribution:**

```bash
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn
```

**Requests per minute (traffic pattern):**

```bash
awk '{print $4}' /var/log/nginx/access.log | cut -d: -f1-3 | uniq -c | tail -20
```

**Find all 404 errors:**

```bash
awk '$9 == 404 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -20
```

**Slow requests (request time > 1 second):**

```bash
awk '$NF > 1.0 {print $7, $NF"s"}' /var/log/nginx/access.log | sort -t' ' -k2 -rn | head -10
```

> **Note:** This works when the log format includes `$request_time` as the last field.

**Bandwidth usage by URL:**

```bash
awk '{url=$7; bytes=$10; total[url]+=bytes} END {for (u in total) printf "%10d  %s\n", total[u], u}' /var/log/nginx/access.log | sort -rn | head -10
```

**Real-time log monitoring:**

```bash
# Follow the log in real-time
tail -f /var/log/nginx/access.log

# Follow with color-coded status codes
tail -f /var/log/nginx/access.log | awk '{
    if ($9 >= 500) color="\033[31m";       # Red for 5xx
    else if ($9 >= 400) color="\033[33m";  # Yellow for 4xx
    else if ($9 >= 300) color="\033[36m";  # Cyan for 3xx
    else color="\033[32m";                 # Green for 2xx
    printf "%s%s\033[0m\n", color, $0
}'
```

### Logrotate for Nginx

Nginx installs a logrotate config by default. View or customize it:

```bash
cat /etc/logrotate.d/nginx
```

Default configuration:

```
/var/log/nginx/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 www-data adm
    sharedscripts
    prerotate
        if [ -d /etc/logrotate.d/httpd-prerotate ]; then \
            run-parts /etc/logrotate.d/httpd-prerotate; \
        fi \
    endscript
    postrotate
        invoke-rc.d nginx rotate >/dev/null 2>&1
    endscript
}
```

To customize (e.g., keep 30 days of logs):

```bash
sudo nano /etc/logrotate.d/nginx
```

```
/var/log/nginx/*.log {
    daily
    missingok
    rotate 30
    compress
    delaycompress
    notifempty
    create 0640 www-data adm
    sharedscripts
    postrotate
        invoke-rc.d nginx rotate >/dev/null 2>&1
    endscript
}
```

Test logrotate:

```bash
sudo logrotate -d /etc/logrotate.d/nginx    # Dry run (debug mode)
sudo logrotate -f /etc/logrotate.d/nginx    # Force rotation
```
