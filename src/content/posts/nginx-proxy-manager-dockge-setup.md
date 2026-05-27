---
title: "Nginx Proxy Manager 自架 — Dockge 一鍵部署"
pubDate: 2026-05-27
categories: [rolod]
tags: [nginx, proxy-manager, reverse-proxy, docker, dockge, self-hosted, ssl]
---

自己 host 咗幾個 service 之後，唔通下下用 `http://192.168.x.x:8090` 入？SSL 又冇、domain 又唔靚、每次要記 port number。

**Nginx Proxy Manager（NPM）** 就係為咗呢個問題而存在 — 一個 Web UI 嘅 reverse proxy，唔使寫 nginx config，mouse click 幾下就加好 domain + SSL。

---

## NPM 係咩？

一個網頁界面俾你管理 Nginx reverse proxy：

- 加 domain 名 → 指去你某個 internal service
- Let's Encrypt SSL 自動申請 + renew
- Access list、404 page、redirection 全部 UI 搞掂
- 一個 port 443 可以 forward 去幾十個 service

```
Browser ──443──▶ NPM
                 ├── beszel.homelab.tw  → localhost:8090
                 ├── dockge.homelab.tw  → localhost:5001
                 ├── uptime.homelab.tw  → localhost:3001
                 └── grafana.homelab.tw → localhost:3000
```

唔使逐個 service expose port，全部經 NPM 一個 entry point。

---

## 用 Dockge 部署

如果你仲未有 Dockge，可以睇返之前嘅 Dockge 安裝文。

### Stack：`nginx-proxy-manager`

入 Dockge → **「Add Stack」** → 填 Name `nginx-proxy-manager` → 貼：

```yaml
services:
  npm:
    image: jc21/nginx-proxy-manager:latest
    container_name: npm
    restart: unless-stopped
    ports:
      - "80:80"       # HTTP（Let's Encrypt 驗證用）
      - "443:443"     # HTTPS
      - "81:81"       # Admin UI
    volumes:
      - ./npm_data:/data
      - ./npm_letsencrypt:/etc/letsencrypt
```

按 **Deploy**。

三個 port 嘅用途：

| Port | 用途 |
|------|------|
| 80 | Let's Encrypt HTTP challenge（整 SSL cert 時用） |
| 443 | HTTPS traffic — 所有 service 經呢個 port 入 |
| 81 | NPM Admin Panel 自己 |

---

## 初始設定

開 browser → `http://192.168.31.188:81`

Default login：

| 欄位 | 值 |
|------|-----|
| Email | `admin@example.com` |
| Password | `changeme` |

第一次 login 會強制你改 email 同 password。

---

## 加第一個 Proxy Host

Login 之後 → **Proxy Hosts** → **Add Proxy Host**：

假設你已經有個 Beszel Hub 行緊 port 8090：

| 欄位 | 值 |
|------|-----|
| **Domain Names** | `beszel.homelab.deven.tw` |
| **Scheme** | `http` |
| **Forward Hostname / IP** | `192.168.31.188` |
| **Forward Port** | `8090` |
| **Cache Assets** | ❌ 唔使 |
| **Block Common Exploits** | ✅ 可以開 |

### SSL 設定

去 **SSL** tab：

| 欄位 | 值 |
|------|-----|
| **SSL Certificate** | 揀 `Request a new SSL Certificate` |
| **Force SSL** | ✅ |
| **Email** | 填你嘅 email（for Let's Encrypt notification） |
| **Agree to Let's Encrypt TOS** | ✅ |

按 **Save**。等 10-20 秒，cert 就會自動申請成功。

之後你開 `https://beszel.homelab.deven.tw` 就見到有鎖仔嘅 HTTPS。

---

## 加多幾個 service

同一個步驟，只係改 domain 同 port：

| Domain | Service | Internal Port |
|--------|---------|--------------|
| `dockge.homelab.deven.tw` | Dockge | 5001 |
| `uptime.homelab.deven.tw` | Uptime Kuma | 3001 |
| `portainer.homelab.deven.tw` | Portainer | 9000 |
| `grafana.homelab.deven.tw` | Grafana | 3000 |

每個加完之後，NPM 會自動 renew SSL cert（每 60 日），唔使你理。

---

## 同 Cloudflare 一齊用

如果你嘅 domain 用緊 Cloudflare DNS，有兩個做法：

### 灰色雲（DNS only）— NPM 直接 handle SSL

```
Cloudflare DNS: beszel.homelab.deven.tw → 你Router Public IP（灰色雲）
    ↓
Router Port Forward 443 → 192.168.31.188:443
    ↓
NPM handle SSL + routing
```

好處：最快，直連冇中轉
壞處：真 IP expose 咗

### 橙色雲（Proxied）— Cloudflare handle SSL

```
Cloudflare DNS: beszel.homelab.deven.tw → Cloudflare IP（橙色雲）
    ↓
Cloudflare → 你Router Public IP → NPM
```

好處：隱藏真 IP，可以 set 防火牆 rules
壞處：經 Cloudflare 中轉，多少少 latency

我自己用 **灰色雲 + NPM SSL**，因為 home server 嘅 traffic 唔想經多一層。

---

## Troubleshooting

### 502 Bad Gateway

NPM 連唔到 upstream service。通常原因：

- Forward Hostname 填錯（check IP 同 port）
- Service 未起好
- Docker network 問題 — 如果 NPM 同 service 唔同 network，用 `192.168.x.x` 而唔好用 `localhost`

### SSL cert 申請失敗

- Port 80 要開通俾 Let's Encrypt 驗證
- Domain DNS 要正確指向你嘅 public IP
- 如果係 Cloudflare 橙色雲，要開 **SSL/TLS → Full (strict)**

### NPM 個 Admin UI 上唔到

- Check `docker logs npm` 睇 error
- 有時 81 port 同其他 service 撞，可以改 `"81:81"` 做 `"8081:81"`

---

## 總結

NPM 係 self-hoster 必備嘅工具。裝咗之後，所有 service 統一經一個 port 443 出街，SSL 自動 renew，domain 管理一目了然。

下一步可以考慮裝 **Uptime Kuma** 幫你睇住啲 service 係咪 alive — 如果 NPM 死咗或者某個 service down 咗，會第一時間收到通知。
