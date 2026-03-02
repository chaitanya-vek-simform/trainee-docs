# Apache — Level 1 (🟢 Easy)

## 1. Prerequisites & Key Concepts

Before diving in, make sure you understand these foundational terms:

| Term | Definition |
|------|-----------|
| **Reverse Proxy** | A server that sits in front of backend servers, forwarding client requests to them and returning the response back to the client. |
| **Load Balancer** | A device or software that distributes incoming network traffic across multiple servers to ensure no single server is overwhelmed. |
| **Backend / Upstream** | The actual application server(s) that process requests behind the proxy. |
| **Virtual Host** | An Apache configuration block that allows you to run multiple websites/services on a single server. |
| **ProxyPass** | The Apache directive that maps a local path to a remote (backend) server URL. |
| **ProxyPassReverse** | Adjusts the URL in HTTP response headers from the backend so the client sees the proxy's address. |
| **Module** | A loadable Apache extension that adds functionality (e.g., `mod_proxy`, `mod_ssl`). |

### Visual: How a Reverse Proxy Works

```
┌──────────┐         ┌──────────────┐         ┌──────────────┐
│  Client   │ ──────▶ │  Apache       │ ──────▶ │  Backend     │
│ (Browser) │ ◀────── │  Reverse Proxy│ ◀────── │  Server      │
└──────────┘         └──────────────┘         └──────────────┘
   Request               Forwards               Processes
   & Response             Traffic                 Request
```

### Visual: How Load Balancing Works

```
                          ┌──────────────┐
                     ┌──▶ │  Backend #1  │
┌──────────┐         │    └──────────────┘
│  Client   │ ──────▶ │    ┌──────────────┐
│ (Browser) │         ├──▶ │  Backend #2  │
└──────────┘         │    └──────────────┘
       │              │    ┌──────────────┐
       │         Apache    └──────────────┘
       │        Load       │  Backend #3  │
       └──── Balancer ──▶  └──────────────┘
```

---

---

## 2. Installing Apache

### Step 1 — Update system packages

```bash
sudo apt update && sudo apt upgrade -y
```

### Step 2 — Install Apache

```bash
sudo apt install apache2 -y
```

### Step 3 — Start and enable Apache

```bash
sudo systemctl start apache2
sudo systemctl enable apache2
```

### Step 4 — Verify Apache is running

```bash
sudo systemctl status apache2
```

You should see **active (running)** in the output. You can also verify in a browser by visiting `http://<your-server-ip>`. You should see the default Apache2 Ubuntu page.

---

---

## 3. Understanding Apache Modules

Apache uses a modular architecture. The features we need (proxying, load balancing, etc.) are provided by specific modules.

### Key Modules

| Module | Purpose |
|--------|---------|
| `proxy` | Core proxy functionality — required for all proxying features. |
| `proxy_http` | Enables proxying of HTTP and HTTPS requests to backend servers. |
| `proxy_balancer` | Adds load balancing capabilities across multiple backends. |
| `lbmethod_byrequests` | Load balancing method — distributes by number of requests (Round Robin). |
| `lbmethod_bytraffic` | Load balancing method — distributes by amount of traffic (bytes). |
| `lbmethod_bybusyness` | Load balancing method — sends to the least busy backend. |
| `headers` | Allows manipulation of HTTP request and response headers. |
| `ssl` | Enables HTTPS / TLS support. |
| `proxy_wstunnel` | Enables proxying of WebSocket connections. |
| `rewrite` | URL rewriting engine for advanced routing rules. |

### Enable modules

```bash
sudo a2enmod proxy proxy_http proxy_balancer lbmethod_byrequests headers
```

### Verify enabled modules

```bash
sudo apachectl -M | grep proxy
```

Expected output should include:

```
proxy_module (shared)
proxy_http_module (shared)
proxy_balancer_module (shared)
```

---

---

## 4. Reverse Proxy Fundamentals

### 4.1 What is a Reverse Proxy?

A **reverse proxy** accepts requests from clients and forwards them to one or more backend servers. The client never communicates directly with the backend — it only sees the proxy.

**Why use a reverse proxy?**

- **Security** — Hides backend server details (IP, port, technology) from the public.
- **SSL Termination** — Handle HTTPS at the proxy level so backends can run plain HTTP.
- **Caching** — Cache frequent responses at the proxy to reduce backend load.
- **Single Entry Point** — Expose one public URL that routes to many internal services.

---

### 4.2 Basic Reverse Proxy Setup

Create a virtual host configuration that proxies all traffic to a backend server running on port `8080`:

```apache
<VirtualHost *:80>
    ServerName example.com

    ProxyPreserveHost On
    ProxyPass / http://127.0.0.1:8080/
    ProxyPassReverse / http://127.0.0.1:8080/
</VirtualHost>
```

#### Line-by-line explanation

| Line | Meaning |
|------|---------|
| `<VirtualHost *:80>` | Listen on all interfaces, port 80 (HTTP). |
| `ServerName example.com` | The domain name this virtual host responds to. |
| `ProxyPreserveHost On` | Pass the original `Host` header from the client to the backend. |
| `ProxyPass / http://127.0.0.1:8080/` | Forward all requests from `/` to the backend at `127.0.0.1:8080`. |
| `ProxyPassReverse / http://127.0.0.1:8080/` | Rewrite response headers (e.g., `Location`) so the client sees the proxy URL, not the backend URL. |
| `</VirtualHost>` | End of virtual host block. |

#### Enable and test the configuration

```bash
# Enable the site
sudo a2ensite your-config-name.conf

# Test configuration syntax
sudo apache2ctl configtest

# Reload Apache to apply
sudo systemctl reload apache2
```

---

### 4.3 Path-Based Routing

You can route different URL paths to different backend services:

```apache
<VirtualHost *:80>
    ServerName example.com

    ProxyPreserveHost On

    # Route /api to the API service
    ProxyPass /api http://127.0.0.1:4000/api
    ProxyPassReverse /api http://127.0.0.1:4000/api

    # Route /app2 to the second app
    ProxyPass /app2 http://127.0.0.1:3002/
    ProxyPassReverse /app2 http://127.0.0.1:3002/

    # Route /app1 to the first app
    ProxyPass /app1 http://127.0.0.1:3001/
    ProxyPassReverse /app1 http://127.0.0.1:3001/
</VirtualHost>
```

> ⚠️ **Important Note on Ordering:**
> Apache evaluates `ProxyPass` directives **in the order they appear**. More specific paths (like `/api`) must be listed **before** less specific ones (like `/app1` or `/`). If you put `/` first, it will match everything and the other rules will never be reached.

---

### 4.4 Preserving Original Client Headers

When Apache proxies a request, the backend sees Apache's IP, not the client's. To preserve the original client information, add these header directives:

```apache
<VirtualHost *:80>
    ServerName example.com

    ProxyPreserveHost On

    # Preserve original client information
    RequestHeader set X-Forwarded-For "%{REMOTE_ADDR}s"
    RequestHeader set X-Real-IP "%{REMOTE_ADDR}s"
    RequestHeader set X-Forwarded-Proto "http"

    ProxyPass / http://127.0.0.1:8080/
    ProxyPassReverse / http://127.0.0.1:8080/
</VirtualHost>
```

| Header | Purpose |
|--------|---------|
| `X-Forwarded-For` | Contains the original client IP address. |
| `X-Real-IP` | Another common header for the real client IP (used by Nginx/backends). |
| `X-Forwarded-Proto` | Tells the backend whether the original request was HTTP or HTTPS. |

---

---

## 5. Practice Scenario 1 — Single Backend Reverse Proxy

**Goal:** Set up Apache as a reverse proxy forwarding all traffic to a single Python HTTP backend on port 8080.

### Step 1 — Start a simple backend server

Open a terminal and start a basic Python HTTP server:

```bash
mkdir -p /tmp/backend && echo "Hello from Backend on port 8080" > /tmp/backend/index.html
cd /tmp/backend && python3 -m http.server 8080
```

Leave this terminal running.

### Step 2 — Enable required Apache modules

```bash
sudo a2enmod proxy proxy_http headers
sudo systemctl restart apache2
```

### Step 3 — Create the virtual host configuration

```bash
sudo tee /etc/apache2/sites-available/reverse-proxy.conf > /dev/null <<'EOF'
<VirtualHost *:80>
    ServerName localhost

    ProxyPreserveHost On

    RequestHeader set X-Forwarded-For "%{REMOTE_ADDR}s"
    RequestHeader set X-Real-IP "%{REMOTE_ADDR}s"
    RequestHeader set X-Forwarded-Proto "http"

    ProxyPass / http://127.0.0.1:8080/
    ProxyPassReverse / http://127.0.0.1:8080/

    ErrorLog ${APACHE_LOG_DIR}/reverse-proxy-error.log
    CustomLog ${APACHE_LOG_DIR}/reverse-proxy-access.log combined
</VirtualHost>
EOF
```

### Step 4 — Enable the site and activate

```bash
# Disable the default site to avoid conflicts
sudo a2dissite 000-default.conf

# Enable the reverse proxy site
sudo a2ensite reverse-proxy.conf

# Test the configuration
sudo apache2ctl configtest

# Reload Apache
sudo systemctl reload apache2
```

### Step 5 — Test with curl

```bash
curl http://localhost
```

**Expected output:**

```
Hello from Backend on port 8080
```

### Step 6 — Verify logs

```bash
# Check access log
sudo tail -5 /var/log/apache2/reverse-proxy-access.log

# Check error log (should be empty if everything is working)
sudo tail -5 /var/log/apache2/reverse-proxy-error.log
```

### Step 7 — Cleanup

```bash
# Stop the Python backend (Ctrl+C in its terminal)

# Disable the site and re-enable the default
sudo a2dissite reverse-proxy.conf
sudo a2ensite 000-default.conf
sudo systemctl reload apache2

# Remove the config file
sudo rm /etc/apache2/sites-available/reverse-proxy.conf

# Remove the temporary backend directory
rm -rf /tmp/backend
```

---

---

## 6. Practice Scenario 2 — Multiple Backends via Path Routing

**Goal:** Route `/app1` to a backend on port 8081 and `/app2` to a backend on port 8082.

### Step 1 — Start two backend servers

Open **two separate terminals**:

**Terminal 1 — Backend on port 8081:**

```bash
mkdir -p /tmp/app1 && echo "Hello from App1 (port 8081)" > /tmp/app1/index.html
cd /tmp/app1 && python3 -m http.server 8081
```

**Terminal 2 — Backend on port 8082:**

```bash
mkdir -p /tmp/app2 && echo "Hello from App2 (port 8082)" > /tmp/app2/index.html
cd /tmp/app2 && python3 -m http.server 8082
```

### Step 2 — Create the virtual host configuration

```bash
sudo tee /etc/apache2/sites-available/multi-backend.conf > /dev/null <<'EOF'
<VirtualHost *:80>
    ServerName localhost

    ProxyPreserveHost On

    # Route /app1 to backend on port 8081
    ProxyPass /app1 http://127.0.0.1:8081/
    ProxyPassReverse /app1 http://127.0.0.1:8081/

    # Route /app2 to backend on port 8082
    ProxyPass /app2 http://127.0.0.1:8082/
    ProxyPassReverse /app2 http://127.0.0.1:8082/

    ErrorLog ${APACHE_LOG_DIR}/multi-backend-error.log
    CustomLog ${APACHE_LOG_DIR}/multi-backend-access.log combined
</VirtualHost>
EOF
```

### Step 3 — Enable and activate the configuration

```bash
sudo a2dissite 000-default.conf
sudo a2ensite multi-backend.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

### Step 4 — Test `/app1`

```bash
curl http://localhost/app1
```

**Expected output:**

```
Hello from App1 (port 8081)
```

### Step 5 — Test `/app2`

```bash
curl http://localhost/app2
```

**Expected output:**

```
Hello from App2 (port 8082)
```

### Step 6 — Test an invalid path

```bash
curl -I http://localhost/app3
```

**Expected output:** You should receive a `403 Forbidden` or `404 Not Found` response since `/app3` is not mapped to any backend.

### Step 7 — Cleanup

```bash
# Stop both Python backends (Ctrl+C in their terminals)

# Disable the site and re-enable the default
sudo a2dissite multi-backend.conf
sudo a2ensite 000-default.conf
sudo systemctl reload apache2

# Remove the config file
sudo rm /etc/apache2/sites-available/multi-backend.conf

# Remove the temporary backend directories
rm -rf /tmp/app1 /tmp/app2
```
