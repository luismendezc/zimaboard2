# 🧱 ZimaBoard 2 – Docker Data Migration to External SSD

**System:** ZimaOS v1.4.3  
**Hardware:** ZimaBoard 2 with Kingston 1 TB SSD (`/dev/jbod0`)  
**Goal:** Move Docker data (`/var/lib/docker`) from internal flash (`/DATA`) to the external SSD (`/dev/jbod0`).

---

## 📖 1 Overview

By default, ZimaOS keeps all Docker images, containers, and volumes under:

```
/var/lib/docker
```

That path lives on the 45 GB internal eMMC partition (`/DATA`).  
Heavy workloads can fill it quickly, so we redirect Docker’s **data-root** to the SSD.

---

## ⚙️ 2 Current Storage Layout (before migration)

| Mount Point | Device | Size | Description |
|--------------|---------|------|-------------|
| `/` | `/dev/root` | 1.2 GB | Read-only system partition |
| `/DATA` | `/dev/mmcblk0p8` | 45 GB | Writable internal storage |
| `/dev/jbod0` | `/dev/sda` | 895 GB | External Kingston SSD |

Check anytime:
```bash
df -h
```

---

## 🚀 3 Migration Procedure

### Step 1 – Stop Docker
```bash
sudo systemctl stop docker.socket
sudo systemctl stop docker.service
```

Confirm it’s stopped:
```bash
ps aux | grep dockerd
```

---

### Step 2 – Prepare SSD directory
```bash
sudo mkdir -p /dev/jbod0/docker
```

---

### Step 3 – Copy existing Docker data (if any)
```bash
sudo rsync -aP /var/lib/docker/ /dev/jbod0/docker/
```

---

### Step 4 – Unmount old location and backup
```bash
sudo umount -l /var/lib/docker
sudo mv /var/lib/docker /var/lib/docker.bak
```

---

### Step 5 – Fix nested folders (if present)
```bash
sudo mv /dev/jbod0/docker/docker /dev/jbod0/dockerdata
sudo rm -rf /dev/jbod0/docker
sudo mv /dev/jbod0/dockerdata /dev/jbod0/docker
```

---

### Step 6 – Update Docker configuration
Create or edit:
```bash
sudo mkdir -p /etc/docker
sudo nano /etc/docker/daemon.json
```
Add:
```json
{
  "data-root": "/dev/jbod0/docker"
}
```

Save → Ctrl + O → Enter → Ctrl + X

---

### Step 7 – Restart Docker
```bash
sudo systemctl start docker.socket
sudo systemctl start docker.service
```

Verify:
```bash
sudo docker info | grep "Docker Root Dir"
# → Docker Root Dir: /dev/jbod0/docker
```

---

## 🧪 4 Testing the Migration

1. **Baseline SSD usage**
   ```bash
   df -h | grep jbod0
   ```
   Note the *Used* column.

2. **Pull a large image**
   ```bash
   sudo docker pull node:lts
   ```

3. **Re-check usage**
   ```bash
   df -h | grep jbod0
   ```
   → Used space increases ≈ 1 GB ✅  
   → `df -h | grep DATA` should stay almost unchanged.

4. **Clean up (optional)**
   ```bash
   sudo docker ps -a
   sudo docker rm <container_id>
   sudo docker rmi -f node:lts
   ```

---

## 🧰 5 Optional Developer Workspace

Create a persistent folder for Node or Claude SDK projects:
```bash
sudo mkdir -p /dev/jbod0/projects
sudo chown -R $USER:$USER /dev/jbod0/projects
```

Run a Node.js container that uses the SSD workspace:
```bash
sudo docker run -it   --name node-dev   -v /dev/jbod0/projects:/usr/src/app   -w /usr/src/app   node:lts bash
```

All code and npm packages reside on the SSD.

---

## ♻️ 6 Rollback Procedure (Revert to Internal Storage)

1. Stop Docker:
```bash
sudo systemctl stop docker.socket
sudo systemctl stop docker.service
```

2. Edit config:
```bash
sudo nano /etc/docker/daemon.json
```
Change to:
```json
{
  "data-root": "/var/lib/docker"
}
```

3. Restore backup:
```bash
sudo rm -rf /var/lib/docker
sudo mv /var/lib/docker.bak /var/lib/docker
```

4. Restart Docker:
```bash
sudo systemctl start docker.socket
sudo systemctl start docker.service
```

5. Verify:
```bash
sudo docker info | grep "Docker Root Dir"
# → Docker Root Dir: /var/lib/docker
```

---

## 🧹 7 Cleanup (After Successful Migration)

Once everything is working and containers start normally:
```bash
sudo rm -rf /var/lib/docker.bak
```

---

## ✅ 8 Summary

| Component | Path | Status |
|------------|------|---------|
| Docker Root Dir | `/dev/jbod0/docker` | Now on SSD |
| Old Internal Location | `/var/lib/docker.bak` | Safe backup |
| Developer Workspace | `/dev/jbod0/projects` | Persistent storage |
| Config File | `/etc/docker/daemon.json` | Controls data location |

---

## 🧠 Notes

- After ZimaOS updates, verify that `/etc/docker/daemon.json` still points to `/dev/jbod0/docker`.  
- Do not delete `/var/lib/docker.bak` until the system has been rebooted and Docker runs correctly.  
- The SSD mount (`/dev/jbod0`) is persistent across reboots.  
- This layout keeps the tiny 1.2 GB system partition and 45 GB `/DATA` area clean.

---

**Author:** Luis Méndez – ZimaBoard 2 Docker SSD Migration  
**Date:** 2025-10-17  
**Version:** 1.0
