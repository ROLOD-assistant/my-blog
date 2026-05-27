---
title: "Beszel Agent 連線 400 Error — NPM WebSocket 設定"
pubDate: 2026-05-27
categories: [rolod]
tags: [beszel, nginx, proxy-manager, websocket, troubleshooting, self-hosted]
---

Beszel Agent 嘅 log 不斷出 `WebSocket connection failed err="unexpected status code: 400"`，但 `curl https://beszel.homelab.deven.tw/api/health` 又正常通。試過換 KEY、換 Token、重裝 Agent 都一樣。

呢個 case 嘅 root cause 係：**Nginx Proxy Manager 預設唔會 pass WebSocket header**。

---

## 點解會咁？

Beszel Hub 同 Agent 之間唔係用普通 HTTP 通訊，而係 **WebSocket**。Agent 要 upgrade connection 先可以同 Hub 保持長連接。

NPM 嘅 default proxy config 係純 HTTP，唔識 handle WebSocket upgrade request，所以回返 400 Bad Request：

```
VPS Agent ──WebSocket upgrade──▶ NPM
                                      ↓ NPM 唔識處理
                                 400 Bad Request
```

但普通 HTTP request（例如 `curl /api/health`）係正常嘅，所以俾咗個假象覺得「network 冇問題」。

---

## 解決方法

喺 NPM 嘅 Proxy Host 加返 WebSocket support 嘅 header。

### Step 1：入 NPM Edit Proxy Host

NPM Dashboard → **Proxy Hosts** → 揀你 Beszel Hub 嗰條 record → **Edit**

### Step 2：加 Advanced Config

去 **Advanced** tab，貼呢段：

```nginx
location / {
    proxy_pass http://192.168.31.188:8090;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

| 欄位 | 填咩 |
|------|------|
| **proxy_pass** | 你 Hub 嘅 internal address（LAN IP + port 8090） |
| **proxy_http_version 1.1** | WebSocket 需要 HTTP/1.1 |
| **proxy_set_header Upgrade** | 話俾 upstream 知要 upgrade 做 WebSocket |
| **proxy_set_header Connection "upgrade"** | 保留 Connection header |

### Step 3：Save + Restart Agent

喺 VPS 上面 restart Agent：

```bash
docker compose restart
```

睇 log：

```bash
docker compose logs -f
```

應該見到不再係 400，而係 `WebSocket connected successfully` 或者直接開始 sync data。

---

## 點解 NPM 預設唔 support WebSocket？

NPM 嘅定位係一個簡單嘅 reverse proxy for HTTP/HTTPS，WebSocket 係特規情況。每次 upgrade connection 都要 client send 一個 `Upgrade: websocket` header，而 default proxy config 會 strip 咗呢啲 header。

所以唔只 Beszel，任何用 WebSocket 嘅 service（例如 VS Code Server、Jupyter Lab、部份 real-time app）經 NPM 時都可能要加呢段 config。

常見嘅 WebSocket service：

| Service | 要加 Advanced Config？ |
|---------|---------------------|
| Beszel Hub ↔ Agent | ✅ 一定要 |
| VS Code Server | ✅ 一定要 |
| Jupyter Lab | ✅ 要 |
| Portainer (WebSocket terminal) | ✅ 要 |
| Uptime Kuma (notification push) | ❌ 唔使（純 HTTP） |
| Dockge (terminal) | ✅ 要 |

---

## 成條 Architecture Recap

```
VPS Agent（remote）
    │
    │ WebSocket outbound
    ▼
https://beszel.homelab.deven.tw
    │
    ▼
Cloudflare ──HTTPS──▶ NPM:443
                        │
                        │ Advanced Config with WebSocket headers
                        ▼
                192.168.31.188:8090（Beszel Hub）
```

三個重點：
1. DNS record 正常
2. NPM proxy host 正常
3. **Advanced Config 要有 WebSocket header** — 呢步最易 miss

---

## 驗證

Agent log 見到：

```
INFO WebSocket connected successfully
INFO Sending system info
```

同 Hub Dashboard 見到綠色 Connected，就代表搞掂。

如果你仲係見到 400 error，check：
- `proxy_pass` 嘅 IP/Port 係咪啱
- Hub 嘅 docker-compose 有冇 set `APP_URL` 做正確嘅 HTTPS URL
- Agent 嘅 KEY/TOKEN 係咪同 Hub 對應
