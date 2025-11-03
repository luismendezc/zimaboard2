# 🚀 ZimaBoard 2 Complete Setup Guide

**System:** ZimaOS v1.4.3
**Hardware:** ZimaBoard 2 with Kingston 1 TB SSD (`/dev/jbod0`)
**Network:** `192.168.1.11` (WiFi: `192.168.1.12`)
**Hostname:** `zimaboard2.local`
**Internet Access:** `https://mencan.oceloti.com` (via Cloudflare Tunnel)
**Author:** Luis Méndez
**Last Updated:** 2025-11-03

---

## 📋 Table of Contents

1. [System Overview](#system-overview)
2. [Storage Configuration](#storage-configuration)
3. [Network Architecture](#network-architecture)
4. [Reverse Proxy Setup (Critical)](#reverse-proxy-setup-critical)
5. [Installed Applications](#installed-applications)
6. [Adding New Applications](#adding-new-applications)
7. [Cloudflare Tunnel](#cloudflare-tunnel)
8. [Maintenance & Troubleshooting](#maintenance--troubleshooting)

---

## 🖥️ System Overview

### Hardware Specifications
- **Device:** ZimaBoard 2
- **OS:** ZimaOS v1.4.3 (based on CasaOS)
- **Storage:**
  - Internal eMMC: 45 GB (`/DATA`)
  - External SSD: 895 GB (`/dev/jbod0`) - Kingston

### Storage Layout

| Mount Point | Device | Size | Usage | Description |
|------------|---------|------|-------|-------------|
| `/` | `/dev/root` | 1.2 GB | System | Read-only system partition |
| `/DATA` | `/dev/mmcblk0p8` | 45 GB | Minimal | Internal eMMC storage |
| `/dev/jbod0` | `/dev/sda` | 895 GB | **Primary** | External SSD for all data |

### Network Configuration

| Interface | IP Address | Purpose |
|-----------|------------|---------|
| Ethernet | `192.168.1.11` | Primary network access |
| WiFi Dongle | `192.168.1.12` | Backup/secondary access |
| mDNS | `zimaboard2.local` | Local hostname resolution |

---

## 💾 Storage Configuration

### Docker Data Migration to SSD

**All Docker data lives on the SSD** (`/dev/jbod0/docker`) to preserve internal storage.

#### Configuration File: `/etc/docker/daemon.json`
```json
{
  "data-root": "/dev/jbod0/docker"
}
```

#### Verify Docker Root Location
```bash
sudo docker info | grep "Docker Root Dir"
# Expected: Docker Root Dir: /dev/jbod0/docker
```

#### SSD Directory Structure
```
/dev/jbod0/
├── docker/              # All Docker data (images, containers, volumes)
├── nginx/               # Nginx reverse proxy configuration
│   └── nginx.conf
├── nexus-data/          # Nexus Repository data
├── planka/              # Planka project management data
│   ├── db/              # PostgreSQL database
│   ├── avatars/         # User avatars
│   └── attachments/     # Project attachments
├── onlyoffice/          # OnlyOffice document server data
│   ├── logs/            # Application logs
│   ├── data/            # Document data
│   ├── lib/             # Library files
│   └── db/              # Database files
└── docs/                # System documentation backup
```

---

## 🌐 Network Architecture

### **CRITICAL: Reverse Proxy Setup**

This is the **foundation** of our multi-application architecture. All services are accessible through a single entry point (port 80) using path-based routing.

#### Architecture Diagram

```
Internet/Local Network
         ↓
    Port 80 (Nginx Reverse Proxy)
         ↓
    ┌────────────────────────────────────┐
    │  Path-Based Routing                │
    ├────────────────────────────────────┤
    │  /              → ZimaOS (8080)    │
    │  /nexus/        → Nexus (8081)     │
    │  /planka/       → Planka (8082)    │
    │  /onlyoffice/   → OnlyOffice (8083)│
    │  /future-app    → Future (8084+)   │
    └────────────────────────────────────┘
```

#### Why This Matters

✅ **Single Port Access:** Everything accessible via port 80
✅ **Clean URLs:** Use paths instead of ports (`/nexus` vs `:8081`)
✅ **Cloudflare Compatible:** Works perfectly with the tunnel
✅ **Scalable:** Easy to add new services
✅ **Professional:** Standard enterprise pattern

---

## 🔧 Reverse Proxy Setup (Critical)

### Port Assignments

| Service | Port | Access |
|---------|------|--------|
| **Nginx (Reverse Proxy)** | 80 | Public entry point |
| ZimaOS Gateway | 8080 | Proxied via `/` |
| Nexus Repository | 8081 | Proxied via `/nexus/` |
| Planka (Project Management) | 8082 | Proxied via `/planka/` |
| OnlyOffice (Document Server) | 8083 | Proxied via `/onlyoffice/` |
| PostgreSQL (Planka DB) | 5432 | Internal only (no proxy) |
| Future Apps | 8084+ | Add to Nginx config |

### Nginx Configuration

**Location:** `/dev/jbod0/nginx/nginx.conf`

```nginx
events {
    worker_connections 1024;
}

http {
    # Increase buffer sizes for large file uploads
    client_max_body_size 2G;
    proxy_buffering off;

    server {
        listen 80;
        server_name _;

        # Root path - ZimaOS Dashboard
        location / {
            proxy_pass http://host.docker.internal:8080;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # WebSocket support for ZimaOS
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }

        # Nexus Repository Manager
        location /nexus/ {
            proxy_pass http://host.docker.internal:8081/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # Planka - Project Management Board
        location /planka/ {
            proxy_pass http://host.docker.internal:8082/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # WebSocket support for real-time updates
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }

        # OnlyOffice Document Server
        location /onlyoffice/ {
            proxy_pass http://host.docker.internal:8083/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Forwarded-Host $host/onlyoffice;

            # Required for OnlyOffice
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }

        # Future applications - Add more location blocks here
        # Example:
        # location /portainer/ {
        #     proxy_pass http://host.docker.internal:9000/;
        #     ...
        # }
    }
}
```

### Nginx Docker Container

**Deployment Command:**
```bash
sudo docker run -d \
  --name nginx-reverse-proxy \
  --restart unless-stopped \
  -p 80:80 \
  -v /dev/jbod0/nginx/nginx.conf:/etc/nginx/nginx.conf:ro \
  --add-host host.docker.internal:host-gateway \
  nginx:alpine
```

**Management Commands:**

| Action | Command |
|--------|---------|
| View logs | `sudo docker logs nginx-reverse-proxy` |
| Reload config | `sudo docker exec nginx-reverse-proxy nginx -s reload` |
| Restart | `sudo docker restart nginx-reverse-proxy` |
| Stop | `sudo docker stop nginx-reverse-proxy` |
| Remove | `sudo docker rm -f nginx-reverse-proxy` |

### ZimaOS Gateway Configuration

**Original port:** 80 → **New port:** 8080

**Configuration file:** `/etc/casaos/gateway.ini`

```ini
[common]
runtimepath = /var/run/casaos

[gateway]
port        = 8080  # Changed from 80
logpath     = /var/log/casaos
logsavename = gateway
logfileext  = log

[ssl]
enabled   = false
port      = 443
cert_path = /var/lib/casaos/ssl/zimaos.local.fullchain.pem
key_path  = /var/lib/casaos/ssl/zimaos.local.key.pem
domain    = zimaos.local
type      = local
```

**Restart after changes:**
```bash
sudo systemctl restart zimaos-gateway.service
```

---

## 📦 Installed Applications

### 1. Nginx Reverse Proxy

**Purpose:** Central gateway for all services
**Status:** ✅ Running
**Port:** 80 (public), proxies to internal services
**Data:** `/dev/jbod0/nginx/`

**Access URLs:**
- Local: `http://192.168.1.11/` or `http://zimaboard2.local/`
- Internet: `https://mencan.oceloti.com/`

---

### 2. Nexus Repository Manager (OSS)

**Purpose:** Artifact repository for APKs, IPAs, EXEs, ZIPs, and more
**Status:** ✅ Running
**Version:** 3.85.0-03 COMMUNITY
**Port:** 8081 (internal), accessed via `/nexus/`
**Data:** `/dev/jbod0/nexus-data/`

**Access URLs:**
- Local: `http://192.168.1.11/nexus` or `http://zimaboard2.local/nexus`
- Internet: `https://mencan.oceloti.com/nexus`

**Initial Credentials:**
- **Username:** `admin`
- **Password:** `0ce47c66-4eda-46c1-bfd2-232e90807ef2`
- ⚠️ **Change password on first login**

**Deployment Command:**
```bash
sudo docker run -d \
  --name nexus \
  --restart unless-stopped \
  -p 8081:8081 \
  -v /dev/jbod0/nexus-data:/nexus-data \
  -e INSTALL4J_ADD_VM_PARAMS="-Xms512m -Xmx1024m -XX:MaxDirectMemorySize=1024m" \
  sonatype/nexus3:latest
```

**Management Commands:**

| Action | Command |
|--------|---------|
| View logs | `sudo docker logs -f nexus` |
| Restart | `sudo docker restart nexus` |
| Stop | `sudo docker stop nexus` |
| Access admin password | `sudo cat /dev/jbod0/nexus-data/admin.password` |

**Creating a Raw Repository (for APKs, IPAs, etc):**
1. Login to Nexus web UI
2. Settings (⚙️) → Repositories → Create repository
3. Select `raw (hosted)`
4. Name: `mobile-artifacts` (or any name)
5. Blob store: `default`
6. Deployment policy: `Allow redeploy` (for development)
7. Click Create

**Upload via API:**
```bash
# Upload file
curl -u admin:password \
  --upload-file myapp.apk \
  http://192.168.1.11/nexus/repository/mobile-artifacts/myapp-v1.0.apk

# Or via internet
curl -u admin:password \
  --upload-file myapp.apk \
  https://mencan.oceloti.com/nexus/repository/mobile-artifacts/myapp-v1.0.apk
```

**Download URL:**
```
http://192.168.1.11/nexus/repository/mobile-artifacts/myapp-v1.0.apk
https://mencan.oceloti.com/nexus/repository/mobile-artifacts/myapp-v1.0.apk
```

---

### 3. Planka - Project Management (Trello Alternative)

**Purpose:** Kanban-style project management board for tracking tasks and projects
**Status:** ✅ Running
**Port:** 8082 (internal), accessed via `/planka/`
**Data:** `/dev/jbod0/planka/`
**Database:** PostgreSQL (port 5432, internal only)

**Access URLs:**
- ⚠️ **Local:** Not working (use Cloudflare URL instead)
- Internet: `https://mencan.oceloti.com/planka`

**Default Admin Credentials:**
- **Username:** `admin` or **Email:** `admin@planka.local`
- **Password:** `admin123`
- ⚠️ **IMPORTANT:** Change password immediately after first login!

**Initial Setup:**
1. Visit `https://mencan.oceloti.com/planka`
2. Login with the default credentials above
3. Change your password in user settings
4. Create boards, lists, and cards like Trello

**Deployment Commands:**

**Step 1 - Create Docker Network:**
```bash
sudo docker network create planka-network
```

**Step 2 - Deploy PostgreSQL Database:**
```bash
sudo docker run -d \
  --name planka-db \
  --restart unless-stopped \
  --network planka-network \
  -v /dev/jbod0/planka/db:/var/lib/postgresql/data \
  -e POSTGRES_DB=planka \
  -e POSTGRES_USER=planka \
  -e POSTGRES_PASSWORD=planka123secure \
  postgres:14-alpine
```

**Step 3 - Deploy Planka:**
```bash
sudo docker run -d \
  --name planka \
  --restart unless-stopped \
  --network planka-network \
  -p 8082:1337 \
  -v /dev/jbod0/planka/avatars:/app/public/user-avatars \
  -v /dev/jbod0/planka/attachments:/app/public/project-background-images \
  -v /dev/jbod0/planka/attachments:/app/private/attachments \
  -e BASE_URL=https://mencan.oceloti.com/planka \
  -e DATABASE_URL=postgresql://planka:planka123secure@planka-db:5432/planka \
  -e SECRET_KEY=notverysecretkey12345678901234567890 \
  -e DEFAULT_ADMIN_EMAIL=admin@planka.local \
  -e DEFAULT_ADMIN_PASSWORD=admin123 \
  -e DEFAULT_ADMIN_NAME="Admin User" \
  -e DEFAULT_ADMIN_USERNAME=admin \
  ghcr.io/plankanban/planka:latest
```

**Note about DEFAULT_ADMIN_* variables:**
- These create the initial admin user on first startup
- Username: `admin` / Password: `admin123`
- After first login, change your password!
- You can optionally remove these variables later (they lock the admin account if left in place)

**Management Commands:**

| Action | Command |
|--------|---------|
| View Planka logs | `sudo docker logs -f planka` |
| View DB logs | `sudo docker logs -f planka-db` |
| Restart Planka | `sudo docker restart planka` |
| Restart DB | `sudo docker restart planka-db` |
| Stop all | `sudo docker stop planka planka-db` |

**Features:**
- ✅ Kanban boards with drag-and-drop
- ✅ Multiple projects and boards
- ✅ Task cards with descriptions, labels, due dates
- ✅ File attachments
- ✅ User avatars and assignments
- ✅ Real-time updates (via WebSocket)

**Important Notes:**
- Planka requires the Cloudflare tunnel URL to work properly
- Local IP access (`http://192.168.1.11/planka`) will show a white screen
- Always use `https://mencan.oceloti.com/planka` (works from local network too!)
- First user registered becomes the admin

---

### 4. OnlyOffice Document Server

**Purpose:** Online document editor (Google Docs/Sheets alternative)
**Status:** ✅ Running
**Version:** Latest Community Edition
**Port:** 8083 (internal), accessed via `/onlyoffice/`
**Data:** `/dev/jbod0/onlyoffice/`

**Access URLs:**
- Local: `http://192.168.1.11/onlyoffice`
- Internet: `https://mencan.oceloti.com/onlyoffice`

**Deployment Command:**
```bash
sudo docker run -d \
  --name onlyoffice \
  --restart unless-stopped \
  -p 8083:80 \
  -v /dev/jbod0/onlyoffice/logs:/var/log/onlyoffice \
  -v /dev/jbod0/onlyoffice/data:/var/www/onlyoffice/Data \
  -v /dev/jbod0/onlyoffice/lib:/var/lib/onlyoffice \
  -v /dev/jbod0/onlyoffice/db:/var/lib/postgresql \
  -e JWT_ENABLED=false \
  onlyoffice/documentserver:latest
```

**Management Commands:**

| Action | Command |
|--------|---------|
| View logs | `sudo docker logs -f onlyoffice` |
| Restart | `sudo docker restart onlyoffice` |
| Stop | `sudo docker stop onlyoffice` |

**Features:**
- ✅ Document editor (DOCX, ODT, TXT, RTF, etc.)
- ✅ Spreadsheet editor (XLSX, ODS, CSV, etc.)
- ✅ Presentation editor (PPTX, ODP, etc.)
- ✅ Real-time collaborative editing
- ✅ Formula support in spreadsheets
- ✅ Comments and track changes

**Usage:**
- OnlyOffice is a **document server** that needs to be integrated with a file storage system
- For standalone use, you can integrate it with:
  - Nextcloud (file storage + OnlyOffice integration)
  - Seafile
  - ownCloud
  - Or use the API to open/edit documents programmatically

**API Example:**
Visit the welcome page at: `https://mencan.oceloti.com/onlyoffice/welcome/`

**Security Note:**
- Currently running with `JWT_ENABLED=false` for development
- For production, enable JWT tokens to secure the document server

---

### 5. Cloudflare Tunnel

**Purpose:** Secure internet access without port forwarding
**Status:** ✅ Running
**Tunnel ID:** `3b32dff1-1b01-4f0d-af1f-844181a752d2`
**Connector ID:** `cde82002-23d0-40f1-923c-ae42c4a03d64`
**Public URL:** `https://mencan.oceloti.com`
**Target:** `http://127.0.0.1:80` (Nginx)

**Deployment:**
```bash
sudo docker run -d \
  --restart=always \
  --network host \
  --name cloudflare-tunnel \
  cloudflare/cloudflared:latest \
  tunnel run --no-autoupdate --token <YOUR_TOKEN>
```

**DNS Configuration (Cloudflare):**

| Type | Name | Value | Proxy |
|------|------|-------|-------|
| `CNAME` | `mencan` | `3b32dff1-1b01-4f0d-af1f-844181a752d2.cfargotunnel.com` | 🟧 Proxied |

**How it Works:**
```
Internet User
    ↓
https://mencan.oceloti.com
    ↓
Cloudflare Edge (SSL termination)
    ↓
Cloudflare Tunnel (encrypted)
    ↓
ZimaBoard2:80 (Nginx)
    ↓
Proxied to appropriate service
```

---

## ➕ Adding New Applications

This is the beauty of our reverse proxy setup. Adding a new application is **easy and consistent**.

### Step-by-Step Process

#### 1. Deploy Your Application Container

Example: Adding Portainer (Docker management UI)

```bash
sudo docker run -d \
  --name portainer \
  --restart unless-stopped \
  -p 9000:9000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /dev/jbod0/portainer-data:/data \
  portainer/portainer-ce:latest
```

#### 2. Update Nginx Configuration

Edit `/dev/jbod0/nginx/nginx.conf` and add a new `location` block:

```nginx
# Portainer Docker Management
location /portainer/ {
    proxy_pass http://host.docker.internal:9000/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # WebSocket support (if needed)
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
}
```

#### 3. Reload Nginx

```bash
sudo docker exec nginx-reverse-proxy nginx -s reload
```

#### 4. Test Access

```bash
# Local
curl -I http://192.168.1.11/portainer/

# Internet (via Cloudflare)
curl -I https://mencan.oceloti.com/portainer/
```

### Port Assignment Strategy

| Port Range | Purpose | Examples |
|------------|---------|----------|
| 80 | Nginx (reverse proxy) | Public entry |
| 8080 | ZimaOS Gateway | Dashboard |
| 8081 | Nexus Repository | Artifacts |
| 8082 | Planka | Project Management |
| 8083 | OnlyOffice | Document Server |
| 8084-8099 | Future applications | Portainer, Jellyfin, etc |
| 5432 | PostgreSQL (Planka) | Database |
| 5433-5999 | Future databases | MySQL, MongoDB, Redis, etc |

### Template for New Apps

```nginx
# <APP_NAME> - <DESCRIPTION>
location /<path>/ {
    proxy_pass http://host.docker.internal:<PORT>/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # Add WebSocket support if needed
    # proxy_http_version 1.1;
    # proxy_set_header Upgrade $http_upgrade;
    # proxy_set_header Connection "upgrade";
}
```

---

## ☁️ Cloudflare Tunnel Configuration

### Requirements

| Component | Purpose |
|-----------|---------|
| Cloudflare account | Tunnel + DNS management |
| Squarespace domain `oceloti.com` | Your domain |
| Docker on ZimaBoard | Run `cloudflared` |

### Setup Steps

1. **Create Tunnel in Cloudflare Dashboard**
   - Go to Zero Trust → Access → Tunnels
   - Create tunnel named `MENCAN`
   - Copy the token

2. **Configure Public Hostname**
   ```
   Hostname: mencan.oceloti.com
   Path: *
   Service: http://127.0.0.1:80
   ```

3. **Deploy Container** (already done)
   ```bash
   sudo docker run -d --restart=always --network host --name cloudflare-tunnel \
     cloudflare/cloudflared:latest tunnel run --no-autoupdate --token <TOKEN>
   ```

4. **Verify**
   ```bash
   sudo docker logs cloudflare-tunnel
   # Look for: "Registered tunnel connection ... protocol=http2"
   ```

### DNS Records (Cloudflare)

All records must have **Proxy status ON** (orange cloud):

| Type | Name | Value | Proxy |
|------|------|-------|-------|
| `CNAME` | `mencan` | `3b32dff1-1b01-4f0d-af1f-844181a752d2.cfargotunnel.com` | 🟧 |
| `A` | `oceloti.com` | `151.101.1.195`, `151.101.65.195` | 🟧 |
| `A` | `www` | Same as above | 🟧 |

### How Traffic Flows

```
User requests: https://mencan.oceloti.com/nexus

1. DNS lookup → Cloudflare IPs (104.21.x.x)
2. Cloudflare Edge receives HTTPS request
3. Cloudflare routes to Tunnel (encrypted)
4. Tunnel forwards to localhost:80 on ZimaBoard
5. Nginx receives request on port 80
6. Nginx routes /nexus/ → localhost:8081
7. Nexus serves response
8. Response travels back through tunnel to user
```

**Benefits:**
- ✅ No port forwarding needed
- ✅ No static IP required
- ✅ SSL/TLS handled by Cloudflare
- ✅ DDoS protection
- ✅ Works behind NAT/CGNAT

---

## 🛠️ Maintenance & Troubleshooting

### System Health Checks

```bash
# Check all running containers
sudo docker ps

# Check disk usage
df -h

# Check Docker root directory
sudo docker info | grep "Docker Root Dir"

# Check ZimaOS Gateway
sudo systemctl status zimaos-gateway.service

# View Nginx logs
sudo docker logs nginx-reverse-proxy

# View Nexus logs
sudo docker logs -f nexus
```

### Common Issues

#### Issue: Can't access services via domain

**Check:**
1. Cloudflare tunnel is running: `sudo docker ps | grep cloudflare`
2. Tunnel logs show connection: `sudo docker logs cloudflare-tunnel`
3. Nginx is running: `sudo docker ps | grep nginx`
4. Test locally first: `curl -I http://localhost/nexus/`

#### Issue: Nginx shows 502 Bad Gateway

**Possible causes:**
- Service is down (check with `sudo docker ps`)
- Wrong port in nginx config
- Service hasn't finished starting (wait 30s)

**Fix:**
```bash
# Restart the backend service
sudo docker restart nexus

# Check service logs
sudo docker logs nexus
```

#### Issue: Changes to nginx.conf not taking effect

**Fix:**
```bash
# Test config syntax
sudo docker exec nginx-reverse-proxy nginx -t

# Reload nginx
sudo docker exec nginx-reverse-proxy nginx -s reload

# If reload doesn't work, restart (REQUIRED for new routes!)
sudo docker restart nginx-reverse-proxy
```

**⚠️ Important:** When adding new location blocks to nginx.conf, you MUST restart the container (not just reload) for the changes to take effect!

#### Issue: Planka shows white screen or asset loading errors

**Symptoms:**
- White screen when accessing Planka
- Console errors like `ERR_CONNECTION_REFUSED` for CSS/JS files
- Assets trying to load from `https://192.168.1.11/...`

**Cause:**
Planka's `BASE_URL` is hardcoded and must match how you access it.

**Fix:**
Always use the Cloudflare tunnel URL: `https://mencan.oceloti.com/planka`
- This works from both inside and outside your local network
- Do NOT use `http://192.168.1.11/planka` (it won't work)

If you need to change BASE_URL:
```bash
sudo docker stop planka
sudo docker rm planka
# Redeploy with new BASE_URL (see Planka deployment section)
```

#### Issue: Forgot Planka admin password or can't login

**Symptom:**
Can't remember the admin password or need to reset Planka completely.

**Solution - Reset Planka and recreate admin user:**
```bash
# Stop and remove containers
sudo docker stop planka planka-db
sudo docker rm planka planka-db

# Delete the database to start fresh
sudo rm -rf /dev/jbod0/planka/db/*

# Redeploy database
sudo docker run -d \
  --name planka-db \
  --restart unless-stopped \
  --network planka-network \
  -v /dev/jbod0/planka/db:/var/lib/postgresql/data \
  -e POSTGRES_DB=planka \
  -e POSTGRES_USER=planka \
  -e POSTGRES_PASSWORD=planka123secure \
  postgres:14-alpine

# Wait 10 seconds, then redeploy Planka (see deployment section for full command)
# Make sure to include DEFAULT_ADMIN_* environment variables
```

**Default credentials after reset:**
- Username: `admin`
- Password: `admin123`

#### Issue: Docker containers not persisting after reboot

**Check:**
- Containers have `--restart unless-stopped` flag
- Verify with: `sudo docker inspect <container> | grep -A5 RestartPolicy`

### Backup Strategy

**Critical Files/Directories:**
```bash
# Nginx configuration
/dev/jbod0/nginx/nginx.conf

# Nexus data
/dev/jbod0/nexus-data/

# Planka data (includes PostgreSQL database!)
/dev/jbod0/planka/

# OnlyOffice data
/dev/jbod0/onlyoffice/

# Docker daemon config
/etc/docker/daemon.json

# ZimaOS gateway config
/etc/casaos/gateway.ini
```

**Backup command:**
```bash
# Create backup directory
sudo mkdir -p /dev/jbod0/backups

# Backup Nginx config
sudo cp /dev/jbod0/nginx/nginx.conf /dev/jbod0/backups/nginx.conf.$(date +%Y%m%d)

# Backup Docker config
sudo cp /etc/docker/daemon.json /dev/jbod0/backups/daemon.json.$(date +%Y%m%d)

# Backup ZimaOS gateway config
sudo cp /etc/casaos/gateway.ini /dev/jbod0/backups/gateway.ini.$(date +%Y%m%d)
```

### Update Procedures

#### Updating Nexus

```bash
# Stop current container
sudo docker stop nexus

# Backup data (optional but recommended)
sudo cp -r /dev/jbod0/nexus-data /dev/jbod0/nexus-data.backup

# Remove old container
sudo docker rm nexus

# Pull latest image
sudo docker pull sonatype/nexus3:latest

# Redeploy with same command
sudo docker run -d \
  --name nexus \
  --restart unless-stopped \
  -p 8081:8081 \
  -v /dev/jbod0/nexus-data:/nexus-data \
  -e INSTALL4J_ADD_VM_PARAMS="-Xms512m -Xmx1024m -XX:MaxDirectMemorySize=1024m" \
  sonatype/nexus3:latest
```

#### Updating Nginx

```bash
# Pull latest image
sudo docker pull nginx:alpine

# Stop and remove old container
sudo docker stop nginx-reverse-proxy
sudo docker rm nginx-reverse-proxy

# Redeploy
sudo docker run -d \
  --name nginx-reverse-proxy \
  --restart unless-stopped \
  -p 80:80 \
  -v /dev/jbod0/nginx/nginx.conf:/etc/nginx/nginx.conf:ro \
  --add-host host.docker.internal:host-gateway \
  nginx:alpine
```

---

## 📊 Quick Reference

### Access URLs

| Service | Local | Internet |
|---------|-------|----------|
| ZimaOS Dashboard | `http://192.168.1.11/` | `https://mencan.oceloti.com/` |
| Nexus Repository | `http://192.168.1.11/nexus` | `https://mencan.oceloti.com/nexus` |
| Planka (Project Mgmt) | ⚠️ Use internet URL | `https://mencan.oceloti.com/planka` |
| OnlyOffice (Docs) | `http://192.168.1.11/onlyoffice` | `https://mencan.oceloti.com/onlyoffice` |

### Default Credentials

| Service | Username/Email | Password | Notes |
|---------|---------------|----------|-------|
| Planka | `admin` or `admin@planka.local` | `admin123` | ⚠️ Change immediately after first login! |
| Nexus | `admin` | See: `/dev/jbod0/nexus-data/admin.password` | First-time password, change on login |

### Important Paths

| Purpose | Path |
|---------|------|
| Docker data | `/dev/jbod0/docker/` |
| Nginx config | `/dev/jbod0/nginx/nginx.conf` |
| Nexus data | `/dev/jbod0/nexus-data/` |
| Planka data | `/dev/jbod0/planka/` |
| OnlyOffice data | `/dev/jbod0/onlyoffice/` |
| Documentation | `/dev/jbod0/docs/` |
| Docker config | `/etc/docker/daemon.json` |
| ZimaOS config | `/etc/casaos/gateway.ini` |

### Essential Commands

```bash
# View all containers
sudo docker ps -a

# View all running services
sudo systemctl list-units | grep -i zimaos

# Check disk usage
df -h | grep jbod0

# Reload Nginx without downtime
sudo docker exec nginx-reverse-proxy nginx -s reload

# View live logs
sudo docker logs -f <container_name>

# Restart ZimaOS Gateway
sudo systemctl restart zimaos-gateway.service
```

---

## 🎯 Future Expansion Ideas

With our reverse proxy setup, you can easily add:

- **Portainer** (`/portainer`) - Docker management UI
- **Jellyfin** (`/jellyfin`) - Media server
- **Home Assistant** (`/homeassistant`) - Home automation
- **GitLab** (`/gitlab`) - Git repository hosting
- **Jenkins** (`/jenkins`) - CI/CD pipeline
- **Grafana** (`/grafana`) - Monitoring dashboards
- **PostgreSQL/MongoDB** - Databases (internal ports only)

Each follows the same pattern:
1. Deploy container on unique port
2. Add location block to Nginx
3. Reload Nginx
4. Access via `/path`

---

## ✅ Summary

This ZimaBoard 2 setup provides:

✅ **Scalable Architecture** - Easy to add new services
✅ **Professional Setup** - Path-based routing like enterprise systems
✅ **Internet Access** - Secure Cloudflare tunnel
✅ **Fast Storage** - Everything on SSD
✅ **Single Entry Point** - All services via port 80
✅ **Clean URLs** - No ugly port numbers
✅ **Documented** - Comprehensive guides for all components

---

**Author:** Luis Méndez
**Contact:** lemendezc
**Device:** ZimaBoard 2
**Documentation Version:** 2.2
**Last Updated:** November 3, 2025

---

## 📚 Related Documentation Files

- [Docker Migration to SSD](./zimaos_docker_migration.md)
- [Cloudflare Tunnel Setup](./zimaboard-cloudflare-tunnel-setup.md)
- [Utility Instructions](./UtilInstructionsZimaboard2.docx)

---

_This document is maintained in `/Users/lemendezc/Documents/Personal/zima/` and backed up to `/dev/jbod0/docs/`_
