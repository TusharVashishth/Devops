# 🚀 Nginx Zero to Hero - Complete Guide

> **Master Nginx from basics to production-ready configurations**

---

## 📋 Table of Contents

0. [Initial Setup - Install Nginx & Node.js](#0️⃣-initial-setup---install-nginx--nodejs)
1. [Single Site Deployment](#1️⃣-single-site-deployment)
2. [Multiple Sites on One Server](#2️⃣-multiple-sites-on-one-server)
3. [Advanced Configuration](#4️⃣-advanced-configuration)
4. [Domain Setup & SSL Configuration](#5️⃣-domain-setup--ssl-configuration)
5. [Load Balancing](#5️⃣-load-balancing)
6. [Quick Commands](#6️⃣-quick-commands)

---

## 0️⃣ Initial Setup - Install Nginx & Node.js

**Before starting, let's set up our EC2 instance with all required software**

### Step 1: Update System Packages

```bash
# Update package list and upgrade existing packages
sudo apt update && sudo apt upgrade -y
```

### Step 2: Install Node.js & npm

```bash
# Install npm
sudo apt-get install npm -y

# Install 'n' (Node version manager)
sudo npm i -g n

# Install LTS version of Node.js
sudo n lts

# Or install specific version
# sudo n 22.0.1
```

> **💡 Important:** After installation, **exit and re-login** to your EC2 instance
>
> ```bash
> exit
> # Then SSH back into your instance
> ssh -i your-key.pem ubuntu@your-ec2-ip
> ```
>
> **Verify installation:**
>
> ```bash
> node -v   # Should show v20.x.x or higher
> npm -v    # Should show 10.x.x or higher
> ```

### Step 3: Install Nginx

```bash
# Install Nginx
sudo apt install nginx -y

# Start Nginx service
sudo systemctl start nginx

# Enable Nginx to start on boot
sudo systemctl enable nginx

# Check Nginx status
sudo systemctl status nginx
```

> **✅ Verify Nginx is Running:**
>
> Open your browser and visit: `http://YOUR_EC2_PUBLIC_IP`
>
> You should see the **"Welcome to nginx!"** page

### Step 4: Verify Installation

```bash
# Check Nginx version
nginx -v

# Check if Nginx is listening on port 80
sudo netstat -tulpn | grep :80

# Or use
sudo lsof -i :80
```

> **🎉 Setup Complete!** You're now ready to configure Nginx for your applications.

---

## 1️⃣ Single Site Deployment

**Use Case:** Deploy one Node.js/Express app running on port 8000

```nginx
# File: /etc/nginx/sites-available/backend
server {
    listen 80;
    server_name 13.232.24.15;  # Your EC2/Server Public IP
    # OR use _ to accept any domain/IP
    # server_name _;

    location / {
        proxy_pass http://localhost:8000;  # Your Node.js app

        # Essential Proxy Headers
        proxy_set_header Host $host;                              # Preserves original Host header (domain name)
        proxy_set_header X-Real-IP $remote_addr;                  # Client's actual IP address
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;  # Chain of proxy IPs (for tracking)
        proxy_set_header X-Forwarded-Proto $scheme;               # Original protocol (http/https)

        # WebSocket Support (for real-time apps like chat, notifications)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_cache_bypass $http_upgrade;
    }
}
```

> **💡 Important Workflow:**
>
> 1. Create config file: `sudo nano /etc/nginx/sites-available/backend`
> 2. Enable site: `sudo ln -s /etc/nginx/sites-available/backend /etc/nginx/sites-enabled/`
> 3. Test configuration: `sudo nginx -t`
> 4. Reload Nginx: `sudo systemctl reload nginx`
>
> **⚠️ Note:** We DON'T modify `/etc/nginx/nginx.conf`. Always create separate files in `sites-available/` and symlink them!

---

## 2️⃣ Multiple Sites on One Server

**Use Case:** Run multiple apps on different ports using the same server IP

```nginx
server {
    listen 80;
    server_name _;   # or 13.201.67.207

    # Optional: if you have some static content for root
    #root /var/www/backend;
    #index index.html;

    # Backend 1: /app -> port 8000
    location / {
        # This strips /app from the path:
        # /app/users  -> backend sees /users
        proxy_pass http://localhost:8000;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_cache_bypass $http_upgrade;
    }

    # Backend 2: /api -> port 3000
    location /api/ {
        # This strips /api from the path:
        # /api/users -> backend sees /users
        proxy_pass http://localhost:3000;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_cache_bypass $http_upgrade;
    }

    # Optional: handle root `/`

}

```

## 4️⃣ Advanced Configuration

### Production-Ready Settings

```nginx
# File: /etc/nginx/sites-available/production-app
server {
    listen 80;
    server_name 13.232.24.15;  # Your server IP

    # Security
    server_tokens off;  # Hide Nginx version (security best practice)
    client_max_body_size 20M;  # Max upload size (for file uploads)

    # Performance
    gzip on;  # Enable compression for faster load times
    gzip_types text/plain text/css application/json application/javascript;

    # Logging
    access_log /var/log/nginx/myapp_access.log;
    error_log /var/log/nginx/myapp_error.log warn;

    # Timeouts (prevent hanging connections)
    proxy_connect_timeout 60s;  # Time to connect to backend
    proxy_send_timeout 60s;     # Time to send request to backend
    proxy_read_timeout 60s;     # Time to read response from backend

    # Buffer Settings (memory optimization)
    client_body_buffer_size 128k;
    client_header_buffer_size 1k;

    location / {
        proxy_pass http://localhost:8000;

        # Complete proxy header set with explanations
        proxy_set_header Host $host;                              # Original hostname
        proxy_set_header X-Real-IP $remote_addr;                  # Client's actual IP
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;  # IP forwarding chain
        proxy_set_header X-Forwarded-Proto $scheme;               # Original protocol
    }
}
```

> **⚠️ Important:** All these settings are in a **separate file** in `sites-available/`
>
> **📝 Enable:**
>
> ```bash
> sudo ln -s /etc/nginx/sites-available/production-app /etc/nginx/sites-enabled/
> sudo nginx -t && sudo systemctl reload nginx
> ```

---

## 5️⃣ Domain Setup & SSL Configuration

### Step 1: Point Domain to Your Server

**In Your Domain Registrar (GoDaddy, Namecheap, Cloudflare, etc.):**

```
Type    Name    Value               TTL
A       @       13.232.24.15        3600
A       www     13.232.24.15        3600
```

> **💡 Explanation:**
>
> - `@` record → `example.com` points to your IP
> - `www` record → `www.example.com` points to your IP
> - Wait 5-10 minutes for DNS propagation

### Step 2: Update Nginx Config with Domain

```nginx
# File: /etc/nginx/sites-available/myapp
server {
    listen 80;
    server_name example.com www.example.com;  # Now use your domain!

    location / {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;                              # Domain name
        proxy_set_header X-Real-IP $remote_addr;                  # Client IP
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;  # Proxy IPs
        proxy_set_header X-Forwarded-Proto $scheme;               # http/https
    }
}
```

```bash
# Reload Nginx
sudo nginx -t && sudo systemctl reload nginx
```

### Step 3: Install SSL with Certbot (Free HTTPS)

```bash
# Install Certbot
sudo apt update
sudo apt install certbot python3-certbot-nginx -y

# Get SSL certificate (automatic Nginx config)
sudo certbot --nginx -d example.com -d www.example.com

# Follow the prompts:
# - Enter email
# - Agree to terms
# - Choose redirect HTTP to HTTPS (option 2)
```

**Certbot automatically creates:**

```nginx
# File: /etc/nginx/sites-available/myapp (auto-updated by certbot)
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$server_name$request_uri;  # Redirect to HTTPS
}

server {
    listen 443 ssl;  # HTTPS port
    server_name example.com www.example.com;

    # SSL Certificates (added by certbot)
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;                              # Domain
        proxy_set_header X-Real-IP $remote_addr;                  # Real IP
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;  # Proxy chain
        proxy_set_header X-Forwarded-Proto $scheme;               # https
    }
}
```

### Step 4: Auto-Renew SSL (Certbot does this automatically)

```bash
# Test auto-renewal
sudo certbot renew --dry-run

# Check renewal timer
sudo systemctl status certbot.timer
```

> **✅ SSL Certificate automatically renews every 60 days!**

### 🎯 Complete Domain + SSL Workflow

```bash
# 1. Point domain to server (in DNS)
# 2. Update nginx config with domain
sudo nano /etc/nginx/sites-available/myapp

# 3. Test and reload
sudo nginx -t
sudo systemctl reload nginx

# 4. Install SSL
sudo certbot --nginx -d example.com -d www.example.com

# 5. Verify HTTPS
curl https://example.com
```

> **🔒 Result:**
>
> - `http://example.com` → Redirects to `https://example.com`
> - `https://example.com` → Secure connection ✅
> - Free SSL certificate from Let's Encrypt
> - Auto-renewal configured

---

## 5️⃣ Load Balancing

**Use Case:** Distribute traffic across 3 Node.js instances for high availability

```nginx
# File: /etc/nginx/sites-available/loadbalanced-app

# Step 1: Define Upstream Block (at the top of the file)

# Option 1: Multiple ports on SAME server (localhost)
upstream backend_servers {
    # Round Robin (default) - distributes requests evenly
    server localhost:8000;
    server localhost:8001;
    server localhost:8002;
}

# Option 2: Multiple EC2 instances (DIFFERENT servers)
upstream backend_servers {
    # Replace with your actual EC2 private IPs
    server 172.31.10.5:8000;   # EC2 Instance 1
    server 172.31.10.6:8000;   # EC2 Instance 2
    server 172.31.10.7:8000;   # EC2 Instance 3

    # Or use public IPs if in different VPCs
    # server 13.232.24.15:8000;
    # server 13.232.24.16:8000;
}

# Step 2: Configure Server Block
server {
    listen 80;
    server_name 13.232.24.15;  # Your server IP

    location / {
        proxy_pass http://backend_servers;  # Points to upstream block above

        # Essential Proxy Headers
        proxy_set_header Host $host;                              # Maintains original host
        proxy_set_header X-Real-IP $remote_addr;                  # Client's real IP
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;  # All proxy IPs in chain
        proxy_set_header X-Forwarded-Proto $scheme;               # Original protocol

        # Connection settings
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

> **📊 Load Balancing Algorithms:**
>
> ```nginx
> upstream backend_servers {
>     least_conn;  # Send to server with fewest connections
>     server localhost:8000;
>     server localhost:8001;
> }
> ```
>
> ```nginx
> upstream backend_servers {
>     ip_hash;  # Same client always goes to same server (sticky sessions)
>     server localhost:8000;
>     server localhost:8001;
> }
> ```
>
> **📝 Enable:**
>
> ```bash
> sudo ln -s /etc/nginx/sites-available/loadbalanced-app /etc/nginx/sites-enabled/
> sudo nginx -t && sudo systemctl reload nginx
> ```

---

## 6️⃣ Quick Commands

**Essential Nginx commands for daily operations**

```bash
# Test configuration (always run before reload/restart)
sudo nginx -t

# Reload Nginx (no downtime - preferred method)
sudo systemctl reload nginx

# Restart Nginx (brief downtime)
sudo systemctl restart nginx

# Check Nginx status
sudo systemctl status nginx

# Start Nginx
sudo systemctl start nginx

# Stop Nginx
sudo systemctl stop nginx

# View error logs (real-time)
sudo tail -f /var/log/nginx/error.log

# View access logs (real-time)
sudo tail -f /var/log/nginx/access.log

# Check Nginx version
nginx -v

# Check which sites are enabled
ls -la /etc/nginx/sites-enabled/

# Remove symbolic link (disable site)
sudo rm /etc/nginx/sites-enabled/myapp
```

---

**🎓 You're now ready to deploy production-grade Nginx configurations with custom domains and SSL!**
