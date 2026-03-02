# Nginx — Level 1 (🟢 Easy)

## 1. Prerequisites and Key Concepts

Before working with Nginx, familiarize yourself with these foundational terms:

### Key Terms

| Term | Definition |
|------|-----------|
| **Web Server** | Software that listens for HTTP/HTTPS requests and serves content (HTML, CSS, JS, images, etc.) to clients (browsers). |
| **Reverse Proxy** | A server that sits in front of backend servers and forwards client requests to them. Clients never communicate directly with the backend. |
| **Upstream** | A group of one or more backend servers that Nginx can proxy requests to. Defined with the `upstream` directive. |
| **Server Block** | An Nginx configuration block (similar to Apache's VirtualHost) that defines how to handle requests for a specific domain or IP. |
| **Directive** | A configuration instruction in Nginx (e.g., `listen 80;`, `root /var/www/html;`). Each directive ends with a semicolon. |
| **Context / Block** | A grouping of directives enclosed in curly braces `{}`. Examples: `http {}`, `server {}`, `location {}`. |
| **Location Block** | A block inside a `server` block that defines how Nginx should process requests matching a specific URI pattern. |
| **Worker Process** | An Nginx child process that handles actual client connections. The master process manages workers. |
| **PCI-DSS** | Payment Card Industry Data Security Standard — a set of security requirements for organizations that handle credit card data. Relevant for TLS/cipher configuration. |
| **TLS** | Transport Layer Security — the cryptographic protocol that secures HTTPS connections. Successor to SSL. |

### How Nginx Processes a Request

```
┌──────────┐       ┌─────────────────────────────────────────────────┐
│  Client   │       │                  Nginx Server                   │
│ (Browser) │       │                                                 │
│           │       │  ┌─────────────┐                                │
│           │──────▶│  │ Master Proc │ (reads config, manages workers)│
│  Request: │       │  └──────┬──────┘                                │
│  GET /    │       │         │ fork                                  │
│  Host:    │       │  ┌──────▼──────┐                                │
│  example  │       │  │Worker Proc 1│──┐                             │
│  .com     │       │  ├─────────────┤  │                             │
│           │       │  │Worker Proc 2│  │  1. Match server_name       │
│           │       │  ├─────────────┤  ├─▶2. Match location block    │
│           │       │  │Worker Proc N│  │  3. Apply directives        │
│           │       │  └─────────────┘  │  4. Serve content / proxy   │
│           │       │                   │                             │
│           │◀──────│───────────────────┘                             │
│  Response │       │                                                 │
└──────────┘       └─────────────────────────────────────────────────┘
```

**Request Processing Flow:**

1. Client sends HTTP request with `Host` header
2. Master process delegates to a worker process
3. Worker matches the `server_name` in server blocks
4. Worker matches the best `location` block for the URI
5. Worker applies directives (serve file, proxy, redirect, etc.)
6. Response is sent back to the client

### Nginx vs Apache — Comparison

| Feature | Nginx | Apache |
|---------|-------|--------|
| **Architecture** | Event-driven, asynchronous | Process/thread-based |
| **Performance** | Excellent for static content & high concurrency | Good, but higher memory per connection |
| **Configuration** | Centralized config files | `.htaccess` for per-directory overrides |
| **Modules** | Compiled at build time (mostly) | Dynamically loadable at runtime |
| **Use Case** | Reverse proxy, load balancer, static serving | Dynamic content (mod_php), flexible per-dir config |
| **Memory Usage** | Low and predictable | Higher under load |
| **Rewrite Rules** | `rewrite` directive, `try_files` | `mod_rewrite` with `.htaccess` |
| **Market Share** | #1 (growing) | #2 (declining) |

---

---

## 2. Installing Nginx

### Step 1 — Update Package Index

```bash
sudo apt update
```

### Step 2 — Install Nginx

```bash
sudo apt install nginx -y
```

### Step 3 — Start and Enable Nginx

```bash
# Start Nginx immediately
sudo systemctl start nginx

# Enable Nginx to start on boot
sudo systemctl enable nginx
```

### Step 4 — Verify Nginx is Running

```bash
# Check service status
sudo systemctl status nginx
```

You should see **active (running)** in the output.

```bash
# Verify with curl
curl -I http://localhost
```

Expected output:

```
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
...
```

### Step 5 — Check Nginx Version

```bash
nginx -v
```

Output:

```
nginx version: nginx/1.18.0 (Ubuntu)
```

For detailed build information:

```bash
nginx -V
```

### Step 6 — Allow Nginx Through the Firewall (UFW)

```bash
# Check available Nginx UFW profiles
sudo ufw app list
```

Output:

```
Available applications:
  Nginx Full
  Nginx HTTP
  Nginx HTTPS
  OpenSSH
```

| Profile | Description |
|---------|-------------|
| `Nginx HTTP` | Opens port 80 only |
| `Nginx HTTPS` | Opens port 443 only |
| `Nginx Full` | Opens both port 80 and 443 |

```bash
# Allow HTTP traffic (port 80)
sudo ufw allow 'Nginx HTTP'

# Verify firewall rules
sudo ufw status
```

> **Note:** If UFW is not enabled, enable it with `sudo ufw enable`. Make sure to allow SSH first (`sudo ufw allow OpenSSH`) to avoid locking yourself out.

---

---

## 3. Understanding Nginx Architecture and Configuration

### Key Files and Directories

| Path | Purpose |
|------|---------|
| `/etc/nginx/nginx.conf` | Main configuration file — global settings |
| `/etc/nginx/sites-available/` | Directory for all server block config files |
| `/etc/nginx/sites-enabled/` | Symlinks to active configs from `sites-available` |
| `/etc/nginx/conf.d/` | Additional configuration files (auto-included) |
| `/etc/nginx/snippets/` | Reusable configuration fragments |
| `/etc/nginx/mime.types` | Maps file extensions to MIME types |
| `/var/log/nginx/access.log` | Request log — every request Nginx handles |
| `/var/log/nginx/error.log` | Error log — startup errors, config issues, upstream failures |
| `/var/www/html/` | Default web root directory |
| `/usr/share/nginx/html/` | Alternative default web root |

### Configuration Structure

Nginx configuration follows a hierarchical block structure. Directives are inherited from parent to child contexts:

```nginx
# ─── Main Context (global settings) ───
user www-data;
worker_processes auto;
pid /run/nginx.pid;
error_log /var/log/nginx/error.log;

# ─── Events Context (connection handling) ───
events {
    worker_connections 1024;
}

# ─── HTTP Context (web server settings) ───
http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    sendfile      on;
    keepalive_timeout 65;

    # ─── Server Context (virtual host) ───
    server {
        listen       80;
        server_name  example.com;
        root         /var/www/example;

        # ─── Location Context (URI matching) ───
        location / {
            try_files $uri $uri/ =404;
        }

        location /images/ {
            alias /var/www/images/;
        }
    }
}
```

**Hierarchy:**

```
main
├── events { }
└── http { }
    ├── server { }           ← Virtual Host 1
    │   ├── location / { }
    │   └── location /api { }
    └── server { }           ← Virtual Host 2
        ├── location / { }
        └── location /docs { }
```

### Essential Management Commands

| Command | Purpose |
|---------|---------|
| `sudo nginx -t` | Test configuration for syntax errors **without** restarting |
| `sudo systemctl reload nginx` | Reload configuration gracefully (no downtime) |
| `sudo systemctl restart nginx` | Full restart (brief downtime) |
| `sudo systemctl stop nginx` | Stop Nginx |
| `sudo systemctl start nginx` | Start Nginx |
| `sudo nginx -T` | Test config **and** dump the full merged configuration |
| `ps aux \| grep nginx` | View running Nginx processes (master + workers) |

**Always test before reloading:**

```bash
# Step 1: Test
sudo nginx -t

# Step 2: If OK, reload
sudo systemctl reload nginx
```

Example `nginx -t` output:

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Example `ps aux | grep nginx`:

```
root      1234  ...  nginx: master process /usr/sbin/nginx
www-data  1235  ...  nginx: worker process
www-data  1236  ...  nginx: worker process
```

---

---

## 4. Deploying a Static Website

### Step 1 — Get a Static Website

**Option A: Clone a sample static site from GitHub**

```bash
cd /tmp
git clone https://github.com/StartBootstrap/startbootstrap-freelancer.git mysite
```

**Option B: Create a simple site manually**

```bash
mkdir -p /tmp/mysite/{css,js,images}
```

Create the HTML file:

```bash
cat > /tmp/mysite/index.html << 'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>My Nginx Site</title>
    <link rel="stylesheet" href="css/style.css">
</head>
<body>
    <header>
        <h1>Welcome to My Nginx Site</h1>
        <p>Served by Nginx on Ubuntu 22.04</p>
    </header>
    <main>
        <section>
            <h2>About</h2>
            <p>This is a sample static website deployed on Nginx.</p>
            <img src="images/logo.png" alt="Logo" width="200">
        </section>
        <section>
            <h2>Downloads</h2>
            <p><a href="files/sample.pdf">Download Sample PDF</a></p>
        </section>
    </main>
    <footer>
        <p>&copy; 2026 My Nginx Site</p>
    </footer>
    <script src="js/main.js"></script>
</body>
</html>
EOF
```

Create the CSS file:

```bash
cat > /tmp/mysite/css/style.css << 'EOF'
* { margin: 0; padding: 0; box-sizing: border-box; }
body { font-family: Arial, sans-serif; line-height: 1.6; color: #333; }
header { background: #2c3e50; color: white; padding: 2rem; text-align: center; }
main { max-width: 800px; margin: 2rem auto; padding: 0 1rem; }
section { margin-bottom: 2rem; }
h2 { color: #2c3e50; margin-bottom: 0.5rem; }
footer { background: #ecf0f1; padding: 1rem; text-align: center; margin-top: 2rem; }
a { color: #3498db; }
EOF
```

Create the JavaScript file:

```bash
cat > /tmp/mysite/js/main.js << 'EOF'
document.addEventListener('DOMContentLoaded', function() {
    console.log('Site loaded successfully!');
});
EOF
```

Create a sample PDF directory and placeholder:

```bash
mkdir -p /tmp/mysite/files
echo "Sample PDF content" > /tmp/mysite/files/sample.pdf
```

### Step 2 — Deploy to `/var/www/mysite`

```bash
# Create the web root directory
sudo mkdir -p /var/www/mysite

# Copy site files
sudo cp -r /tmp/mysite/* /var/www/mysite/

# Set ownership to www-data (Nginx user)
sudo chown -R www-data:www-data /var/www/mysite

# Set permissions (directories: 755, files: 644)
sudo chmod -R 755 /var/www/mysite
sudo find /var/www/mysite -type f -exec chmod 644 {} \;
```

### Step 3 — Create a Server Block

```bash
sudo nano /etc/nginx/sites-available/mysite
```

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name mysite.local;

    root /var/www/mysite;
    index index.html index.htm;

    # Logging
    access_log /var/log/nginx/mysite-access.log;
    error_log  /var/log/nginx/mysite-error.log;

    location / {
        try_files $uri $uri/ =404;
    }

    # Serve static assets with caching
    location ~* \.(css|js|png|jpg|jpeg|gif|ico|svg)$ {
        expires 7d;
        add_header Cache-Control "public, no-transform";
    }
}
```

### Step 4 — Enable the Site

```bash
# Create symlink to enable the site
sudo ln -s /etc/nginx/sites-available/mysite /etc/nginx/sites-enabled/

# Remove the default site (optional, but recommended)
sudo rm /etc/nginx/sites-enabled/default

# Test configuration
sudo nginx -t

# Reload Nginx
sudo systemctl reload nginx
```

### Step 5 — Test the Deployment

Add a local hosts entry (if not using a real domain):

```bash
echo "127.0.0.1 mysite.local" | sudo tee -a /etc/hosts
```

Test with `curl`:

```bash
curl -I http://mysite.local
```

Expected output:

```
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Content-Type: text/html
...
```

```bash
# Fetch the full page
curl http://mysite.local
```

You should see the HTML content of your site.

---

---

## 5. Gzip Compression

### How Gzip Works

Gzip compresses response bodies before sending them to the client. The client decompresses them. This significantly reduces bandwidth usage:

```
Without Gzip:
┌────────┐    200 KB HTML     ┌────────┐
│ Nginx  │───────────────────▶│ Client │
└────────┘                    └────────┘

With Gzip:
┌────────┐  Compress   40 KB gzipped   Decompress  ┌────────┐
│ Nginx  │──────────▶──────────────────▶───────────▶│ Client │
└────────┘  (on fly)                    (browser)   └────────┘

Bandwidth savings: ~80% for text-based content!
```

### Configuration

Edit the main Nginx config to add gzip settings inside the `http` block:

```bash
sudo nano /etc/nginx/nginx.conf
```

Add or modify within the `http { }` block:

```nginx
http {
    # ... existing settings ...

    # ─── Gzip Compression ───
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_min_length 256;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/x-javascript
        application/xml
        application/xml+rss
        application/vnd.ms-fontobject
        font/opentype
        font/ttf
        image/svg+xml
        image/x-icon;
    gzip_disable "msie6";
}
```

### Settings Explanation

| Directive | Value | Purpose |
|-----------|-------|---------|
| `gzip on` | `on` | Enable gzip compression |
| `gzip_vary` | `on` | Adds `Vary: Accept-Encoding` header so proxies cache both gzipped and non-gzipped versions |
| `gzip_proxied` | `any` | Compress responses for all proxied requests regardless of request headers |
| `gzip_comp_level` | `6` | Compression level 1–9. Level 6 is a good balance between CPU usage and compression ratio |
| `gzip_min_length` | `256` | Don't compress responses smaller than 256 bytes (overhead not worth it) |
| `gzip_types` | (list) | MIME types to compress. Images (JPEG, PNG) are already compressed — don't gzip them |
| `gzip_disable` | `"msie6"` | Disable gzip for Internet Explorer 6 (buggy decompression support) |

### Test and Reload

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### Verify Gzip is Working

```bash
# Request with gzip support
curl -H "Accept-Encoding: gzip" -I http://mysite.local
```

Expected — look for `Content-Encoding: gzip`:

```
HTTP/1.1 200 OK
Content-Encoding: gzip
Vary: Accept-Encoding
Content-Type: text/html
...
```

Compare response sizes:

```bash
# Without gzip
curl -so /dev/null -w "Size: %{size_download} bytes\n" http://mysite.local

# With gzip
curl -so /dev/null -w "Size: %{size_download} bytes\n" -H "Accept-Encoding: gzip" http://mysite.local
```

The gzipped response should be significantly smaller.

---

---

## 6. Virtual Hosts (Multiple Sites on One Server)

### How Virtual Hosts Work

Virtual hosting allows a single Nginx instance to serve multiple websites. Nginx uses the `Host` header from the HTTP request to determine which server block should handle the request:

```
                        ┌─────────────────────────────┐
                        │        Nginx Server          │
                        │         (Single IP)          │
                        │                              │
  site-a.local ────────▶│  server_name site-a.local;  │──▶ /var/www/site-a/
                        │                              │
  site-b.local ────────▶│  server_name site-b.local;  │──▶ /var/www/site-b/
                        │                              │
  unknown.host ────────▶│  server_name _;  (default)   │──▶ Default page
                        │                              │
                        └─────────────────────────────┘
```

### Step 1 — Create Directories and Content for Each Site

```bash
# Create web root directories
sudo mkdir -p /var/www/site-a
sudo mkdir -p /var/www/site-b

# Create index pages
cat << 'EOF' | sudo tee /var/www/site-a/index.html
<!DOCTYPE html>
<html>
<head><title>Site A</title></head>
<body style="background:#3498db;color:white;text-align:center;padding:50px;">
  <h1>Welcome to Site A</h1>
  <p>This is site-a.local served by Nginx</p>
</body>
</html>
EOF

cat << 'EOF' | sudo tee /var/www/site-b/index.html
<!DOCTYPE html>
<html>
<head><title>Site B</title></head>
<body style="background:#e74c3c;color:white;text-align:center;padding:50px;">
  <h1>Welcome to Site B</h1>
  <p>This is site-b.local served by Nginx</p>
</body>
</html>
EOF

# Set ownership
sudo chown -R www-data:www-data /var/www/site-a /var/www/site-b
```

### Step 2 — Create Server Blocks

**Site A:**

```bash
sudo nano /etc/nginx/sites-available/site-a
```

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name site-a.local;

    root /var/www/site-a;
    index index.html;

    access_log /var/log/nginx/site-a-access.log;
    error_log  /var/log/nginx/site-a-error.log;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

**Site B:**

```bash
sudo nano /etc/nginx/sites-available/site-b
```

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name site-b.local;

    root /var/www/site-b;
    index index.html;

    access_log /var/log/nginx/site-b-access.log;
    error_log  /var/log/nginx/site-b-error.log;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### Step 3 — Default Catch-All Server

Create a default server block that handles requests for unrecognized hostnames:

```bash
sudo nano /etc/nginx/sites-available/default-catch-all
```

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    server_name _;

    return 444;  # Drop connection (Nginx-specific: close with no response)
}
```

> **Note:** `server_name _` is a convention for a catch-all. The `_` doesn't match any real hostname. The `default_server` parameter on `listen` makes this the fallback.

### Step 4 — Enable Sites

```bash
# Remove existing default
sudo rm -f /etc/nginx/sites-enabled/default

# Enable all sites
sudo ln -s /etc/nginx/sites-available/site-a /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/site-b /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/default-catch-all /etc/nginx/sites-enabled/

# Test and reload
sudo nginx -t
sudo systemctl reload nginx
```

### Step 5 — Test Locally

Add entries to `/etc/hosts`:

```bash
echo "127.0.0.1 site-a.local site-b.local" | sudo tee -a /etc/hosts
```

Test each site:

```bash
# Test Site A
curl http://site-a.local
# Expected: "Welcome to Site A"

# Test Site B
curl http://site-b.local
# Expected: "Welcome to Site B"

# Test with Host header directly
curl -H "Host: site-a.local" http://127.0.0.1
curl -H "Host: site-b.local" http://127.0.0.1

# Test catch-all (unknown host) — connection should be dropped
curl -H "Host: unknown.local" http://127.0.0.1
# Expected: curl error (empty reply from server)
```

### Step 6 — Cleanup (Optional)

When done testing:

```bash
# Remove test entries from /etc/hosts
sudo sed -i '/site-a.local/d' /etc/hosts
sudo sed -i '/site-b.local/d' /etc/hosts

# Disable sites
sudo rm /etc/nginx/sites-enabled/site-a
sudo rm /etc/nginx/sites-enabled/site-b
sudo rm /etc/nginx/sites-enabled/default-catch-all

# Re-enable your main site or default
sudo ln -s /etc/nginx/sites-available/mysite /etc/nginx/sites-enabled/

sudo nginx -t && sudo systemctl reload nginx
```

---

---

## 7. URL Rewriting and Redirects

### Redirects vs Rewrites

| Feature | Redirect (`return` / `rewrite … redirect`) | Rewrite (`rewrite … last/break`) |
|---------|---------------------------------------------|----------------------------------|
| **Visible to client?** | ✅ Yes — browser URL changes | ❌ No — internal to Nginx |
| **HTTP round trip?** | ✅ Yes — client makes a new request | ❌ No — single request |
| **Status code** | 301 (permanent) or 302 (temporary) | None (internal) |
| **SEO impact** | 301 passes link juice | No impact (URL doesn't change) |
| **Use case** | Domain migration, HTTP→HTTPS, old→new URLs | Clean URLs, SPA routing, internal mapping |

### Redirects with `return`

The `return` directive is the simplest and most efficient way to redirect:

```bash
sudo nano /etc/nginx/sites-available/mysite
```

```nginx
server {
    listen 80;
    server_name mysite.local;
    root /var/www/mysite;
    index index.html;

    # ─── 301 Permanent Redirect ───
    # Old page permanently moved to new location
    location = /old-page {
        return 301 /new-page;
    }

    # ─── 302 Temporary Redirect ───
    # Temporarily send users elsewhere (e.g., maintenance)
    location = /maintenance {
        return 302 /under-construction.html;
    }

    # ─── External Redirect ───
    # Redirect to a completely different domain
    location = /github {
        return 301 https://github.com;
    }

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### Rewrites with `rewrite`

The `rewrite` directive uses regex to transform URIs internally or externally:

```nginx
server {
    listen 80;
    server_name mysite.local;
    root /var/www/mysite;
    index index.html;

    # ─── Remove .html extension ───
    # /about.html → internally serves /about
    rewrite ^/(.+)\.html$ /$1 permanent;

    # ─── Add trailing slash ───
    # /about → /about/
    rewrite ^(/[^.]*[^/])$ $1/ permanent;

    # ─── Rewrite product URLs ───
    # /product/42 → internally fetches /products.html?id=42
    rewrite ^/product/(\d+)$ /products.html?id=$1 last;

    # ─── Internal rewrite with `last` flag ───
    # /blog/my-post → internally serves /posts/my-post/index.html
    rewrite ^/blog/(.+)$ /posts/$1/index.html last;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### Rewrite Flags

| Flag | Behavior |
|------|----------|
| `last` | Stops processing current `rewrite` rules and **restarts** location matching with the new URI. Use in `server` context. |
| `break` | Stops processing current `rewrite` rules and **continues** processing in the current location. Use in `location` context. |
| `redirect` | Returns a **302 temporary** redirect to the client. |
| `permanent` | Returns a **301 permanent** redirect to the client. |

### `try_files` for SPA Routing

Single Page Applications (React, Vue, Angular) need all routes to serve `index.html`:

```nginx
server {
    listen 80;
    server_name mysite.local;
    root /var/www/mysite;
    index index.html;

    location / {
        # Try the exact file → try as directory → fall back to index.html
        try_files $uri $uri/ /index.html;
    }

    # API requests should NOT fall back to index.html
    location /api/ {
        proxy_pass http://localhost:3000;
    }
}
```

**How `try_files` works:**

```
Request: GET /dashboard/settings

1. Try: /var/www/mysite/dashboard/settings      → not found
2. Try: /var/www/mysite/dashboard/settings/      → not found
3. Serve: /var/www/mysite/index.html             → ✅ SPA handles routing
```

### Verify Redirects

```bash
# Test 301 redirect — follow redirects with -L
curl -I http://mysite.local/old-page
# Expected: HTTP/1.1 301 Moved Permanently
#           Location: http://mysite.local/new-page

# Test 302 redirect
curl -I http://mysite.local/maintenance
# Expected: HTTP/1.1 302 Moved Temporarily

# Follow the redirect chain
curl -L -I http://mysite.local/old-page
```

---

---

## 8. Access Control (IP Allow/Deny)

### Basic Allow/Deny

Nginx processes `allow` and `deny` directives in order — the first match wins.

**Protect an Admin Area:**

```nginx
server {
    listen 80;
    server_name mysite.local;
    root /var/www/mysite;

    # Restrict /admin to specific IPs only
    location /admin {
        allow 192.168.1.100;       # Office IP
        allow 10.0.0.0/24;         # VPN range
        deny  all;                 # Block everyone else

        try_files $uri $uri/ =404;
    }

    # Block specific malicious IPs from the entire site
    location / {
        deny  203.0.113.42;        # Known bad actor
        deny  198.51.100.0/24;     # Blocked range
        allow all;                 # Allow everyone else

        try_files $uri $uri/ =404;
    }
}
```

### `geo` for Complex Access Control

The `geo` module lets you define variables based on client IP, useful for more complex scenarios:

```nginx
# In http { } context (nginx.conf or included file)
geo $blocked {
    default         0;        # Not blocked by default
    203.0.113.0/24  1;        # Block this range
    198.51.100.42   1;        # Block this IP
    10.0.0.0/8      0;        # Explicitly allow internal
}

server {
    listen 80;
    server_name mysite.local;
    root /var/www/mysite;

    # Use the geo variable to deny access
    if ($blocked) {
        return 403;
    }

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### Password Protection (HTTP Basic Auth)

HTTP Basic Authentication prompts users for a username and password.

**Step 1 — Install `apache2-utils` for `htpasswd`:**

```bash
sudo apt install apache2-utils -y
```

**Step 2 — Create a password file:**

```bash
# Create a new file with the first user
sudo htpasswd -c /etc/nginx/.htpasswd admin

# Add another user (without -c to avoid overwriting)
sudo htpasswd /etc/nginx/.htpasswd devuser
```

You'll be prompted to enter and confirm a password for each user.

**Step 3 — Verify the file:**

```bash
cat /etc/nginx/.htpasswd
```

Output (passwords are hashed):

```
admin:$apr1$xyz123$...
devuser:$apr1$abc456$...
```

**Step 4 — Configure Nginx:**

```nginx
server {
    listen 80;
    server_name mysite.local;
    root /var/www/mysite;

    # Protect /admin with password
    location /admin {
        auth_basic "Admin Area — Restricted";
        auth_basic_user_file /etc/nginx/.htpasswd;

        try_files $uri $uri/ =404;
    }

    # Public area
    location / {
        try_files $uri $uri/ =404;
    }
}
```

**Step 5 — Test and Reload:**

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### Verify Access Control

```bash
# Test IP deny — should get 403
curl -I http://mysite.local/admin
# Expected: HTTP/1.1 403 Forbidden

# Test Basic Auth — without credentials
curl -I http://mysite.local/admin
# Expected: HTTP/1.1 401 Authorization Required

# Test Basic Auth — with credentials
curl -u admin:yourpassword http://mysite.local/admin
# Expected: HTTP/1.1 200 OK (or the page content)

# Test Basic Auth — wrong credentials
curl -u admin:wrongpassword -I http://mysite.local/admin
# Expected: HTTP/1.1 401 Authorization Required
```

> **Security Note:** HTTP Basic Auth sends credentials in Base64 (not encrypted). Always use it over HTTPS in production!
