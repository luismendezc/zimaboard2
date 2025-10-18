# 🌐 ZimaBoard 2 Cloudflare Tunnel Setup (with Squarespace Domain)

## Overview

This guide explains how to expose a **ZimaBoard 2** web dashboard (ZimaOS Gateway) to the internet securely using a **Cloudflare Tunnel**, even when your public IP changes or ports are closed by your ISP.  
It uses a **Squarespace domain (`oceloti.com`)** with Cloudflare DNS to serve the tunnel under  
👉 `https://mencan.oceloti.com`.

---

## 🧰 Requirements

| Component | Purpose |
|------------|----------|
| **ZimaBoard 2** running ZimaOS | Host server |
| **Docker** installed on ZimaBoard | To run `cloudflared` |
| **Cloudflare account** | For tunnel + DNS proxy |
| **Squarespace domain (`oceloti.com`)** | Your registered domain |

---

## ⚙️ Step 1 – Cloudflare Account and Tunnel

1. Go to [https://dash.cloudflare.com](https://dash.cloudflare.com) and **sign in** or create an account.
2. In the dashboard, click **Zero Trust → Access → Tunnels**.
3. Click **Create a tunnel → Cloudflared**.
4. Name the tunnel (e.g. `MENCAN`).
5. Copy the generated **token** — you’ll use it inside Docker.
6. Under **Public Hostname**, set:
   ```
   Hostname: mencan.oceloti.com
   Path: *
   Service: http://127.0.0.1:80
   ```
   This forwards all requests from your domain to your local ZimaOS web UI.

---

## 🐳 Step 2 – Run Cloudflared in Docker on ZimaBoard

1. SSH into your ZimaBoard terminal.
2. Remove old containers if any:
   ```bash
   sudo docker rm -f cloudflare-tunnel || true
   ```
3. Run the tunnel container (replace `<YOUR_TOKEN>` with your real token):
   ```bash
   sudo docker run -d --restart=always --network host --name cloudflare-tunnel      cloudflare/cloudflared:latest tunnel run --no-autoupdate --token <YOUR_TOKEN>
   ```
4. Verify it’s running:
   ```bash
   sudo docker ps
   sudo docker logs -n 20 cloudflare-tunnel
   ```
   You should see:
   ```
   Registered tunnel connection ... protocol=http2
   Updated to new configuration ... service="http://127.0.0.1:80"
   ```

---

## 🌍 Step 3 – Configure DNS on Cloudflare (Squarespace Domain)

1. Log into your **Cloudflare dashboard**.
2. Add your **domain `oceloti.com`** (if not already there).
3. In **DNS → Records**, add or edit these:

   | Type | Name | Value | Proxy | TTL |
   |------|------|--------|-------|-----|
   | `CNAME` | `mencan` | `3b32dff1-1b01-4f0d-af1f-844181a752d2.cfargotunnel.com` | 🟧 **Proxied** | Auto |
   | `A` | `oceloti.com` | `151.101.1.195` and `151.101.65.195` | 🟧 Proxied | Auto |
   | `A` | `www` | same as above | 🟧 Proxied | Auto |

   Make sure the **orange cloud icon = Proxied** on all tunnel-related records.

4. If Cloudflare isn’t the authoritative DNS yet, update the **Nameservers** in Squarespace:
   - Replace Squarespace’s NS entries with those Cloudflare assigns (e.g. `lila.ns.cloudflare.com`, `weston.ns.cloudflare.com`).

5. Wait until propagation completes (≈ 15–30 min).

---

## 🔍 Step 4 – Verification

From your ZimaBoard or any device:

```bash
nslookup mencan.oceloti.com
```
✅ Expected output:
```
Name: mencan.oceloti.com
Addresses: 104.21.x.x, 172.67.x.x
```

Then:
```bash
curl -I https://mencan.oceloti.com
```
✅ Expected:
```
HTTP/1.1 200 OK
Server: cloudflare
Via: ZimaOS-Gateway
```

If you see that, the tunnel is active and publicly accessible.

---

## 🧠 Step 5 – (Recommended) Container Management Commands

| Action | Command |
|--------|----------|
| View tunnel logs | `sudo docker logs -f cloudflare-tunnel` |
| Restart container | `sudo docker restart cloudflare-tunnel` |
| Remove container | `sudo docker rm -f cloudflare-tunnel` |
| Pull latest image | `sudo docker pull cloudflare/cloudflared:latest` |
| Verify uptime | `sudo docker ps` |

---

## ✅ Step 6 – Test from External Network

1. Disconnect Wi-Fi on your phone and use mobile data.  
2. Visit [https://mencan.oceloti.com](https://mencan.oceloti.com).  
3. You should see your **ZimaOS Gateway dashboard**.  

---

## 💡 Notes and Troubleshooting

| Issue | Fix |
|-------|-----|
| `ERR_CONNECTION_TIMED_OUT` | Check that the DNS CNAME is **Proxied** and tunnel container is running. |
| `fd10::` private address from nslookup | Domain still on Squarespace DNS → move nameservers to Cloudflare. |
| `UDP blocked` | Normal – Cloudflared falls back to HTTPS (443). |
| `No such container` | Use the correct container name from `docker ps`. |

---

## 🧾 Final Configuration Summary

| Item | Value |
|------|-------|
| **Tunnel ID** | `3b32dff1-1b01-4f0d-af1f-844181a752d2` |
| **Connector ID** | `cde82002-23d0-40f1-923c-ae42c4a03d64` |
| **Hostname** | `mencan.oceloti.com` |
| **Service** | `http://127.0.0.1:80` |
| **Cloudflared Protocol** | HTTP/2 over TCP 443 |
| **DNS Proxy Mode** | Proxied (orange cloud) |
| **Firewall Ports** | Outbound 443 (open), UDP 7844 (optional) |

---

## 🎉 Result

✅ `https://mencan.oceloti.com` now securely serves the **ZimaOS Gateway** interface from your **ZimaBoard 2**, without port forwarding, static IP, or VPN exposure.

---

_Authored by Luis Méndez | Last updated: 2025-10-18_
