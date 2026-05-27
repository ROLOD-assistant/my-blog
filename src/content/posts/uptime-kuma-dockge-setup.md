---
title: "Uptime Kuma 自架 — Dockge 部署 + NPM 綁 Domain"
pubDate: 2026-05-27
categories: [rolod]
tags: [uptime-kuma, monitoring, docker, dockge, self-hosted, ssl]
---

之前講過 Beszel 用嚟 monitor 系統資源（CPU、RAM、Disk），但要 monitor **service 層面** — 個 website 仲行唔行到、SSL cert 幾時到期、response time 有冇慢到 — 就係 **Uptime Kuma** 嘅範疇。

兩個加埋先係完整嘅 monitoring。

---

## Uptime Kuma 係咩？

一個開源嘅 service uptime monitoring 工具，Node.js 寫，Vue.js frontend。

佢唔係裝喺你要 monitor 嗰部機上面，而係從你自己部 server 主動 probe 出去：

```
Uptime Kuma
  ├── HTTP GET → https://beszel.homelab.deven.tw    ← 自己 host 嘅 service
  ├── HTTP GET → https://github.com                  ← 第三方都得
  ├── PING     → 8.8.8.8
  ├── TCP      → 192.168.31.188:3306                 ← internal port check
  └── DNS      → google.com
```

任何 timeout / 非預期 status code / SSL cert 就到期，都會 alert 你。

---

## 用 Dockge 部署

### Stack：`uptime-kuma`

入 Dockge → **「Add Stack」** → 填 Name `uptime-kuma` → 貼：

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:latest
    container_name: uptime-kuma
    restart: unless-stopped
    ports:
      - "3001:3001"    # Web UI
    volumes:
      - ./uptime-kuma_data:/app/data
```

按 **Deploy**。

Port 3001 係 Web UI，data 用 Docker volume 持久化。

---

## 初始設定

開 browser → `http://192.168.31.188:3001`

第一次會叫你建立 Admin account：

| 欄位 | 填乜 |
|------|------|
| **Username** | 你鍾意，例如 `admin` |
| **Password** | 自己 set |

Login 之後見到空空如也嘅 dashboard。

---

## 加第一個 Monitor

按 **「Add Monitor」**：

| 欄位 | 值 |
|------|-----|
| **Monitor Type** | `HTTP(s)` — 最常用 |
| **Friendly Name** | `Beszel Hub` |
| **URL** | `https://beszel.homelab.deven.tw` |
| **Monitor Interval** | `60` 秒（預設） |
| **Check Certificate** | ✅ 開 — check SSL cert 到期日 |
| **Retry** | 3 次先當 down（避免短暫網絡問題誤報） |

按 **Save**。

等幾秒就會見到第一次 check 結果，綠色 = UP。

---

## 常用 Monitor 類型

| Type | 用喺邊 |
|------|--------|
| HTTP(s) | 任何 website / API endpoint |
| Ping | 部機係咪 online（ICMP） |
| TCP Port | 某個 port 仲開唔開到，例如 3306 (MySQL)、5432 (Postgres) |
| DNS | Domain resolve 到幾多個 record |
| Docker | 一個 container 仲行唔行到 |

我自己 set 咗呢啲：

| Monitor | Type | Interval |
|---------|------|----------|
| Beszel Hub | HTTPS | 60s |
| Dockge | HTTPS | 60s |
| NPM | HTTPS | 60s |
| Blog | HTTPS | 60s |
| Router (Ping) | Ping | 120s |
| Server alive | TCP :22 | 120s |

全部一個 dashboard 睇晒。

---

## 加 Notification

**Settings → Notifications → 「Set up Notification」**

支援超多平台：

| 平台 | 設定 |
|------|------|
| Telegram | Bot Token + Chat ID |
| Discord | Webhook URL |
| Slack | Webhook URL |
| Email (SMTP) | 自己 email server |
| Webhook | 任何自訂 endpoint |
| Gotify / Pushover / Pushbullet | 手機 push |
| Line / WeChat | 都有支援 |

揀你最常用嘅，跟住佢嘅指引 fill 一次，之後所有 monitor down 咗都會經呢個 channel 通知你。

---

## 經 NPM 綁 Domain

如果你跟上一篇裝咗 Nginx Proxy Manager，可以將 Uptime Kuma 嘅 UI 綁返個靚 domain：

### NPM 加 Proxy Host

| 欄位 | 值 |
|------|-----|
| **Domain Names** | `uptime.homelab.deven.tw` |
| **Scheme** | `http` |
| **Forward Hostname / IP** | `192.168.31.188` |
| **Forward Port** | `3001` |
| **SSL** | Request Let's Encrypt + Force SSL |

之後就唔使記 port number，直接 `https://uptime.homelab.deven.tw` 入。

---

## Status Page 功能

Uptime Kuma 有個好實用嘅功能 — **Status Page**。整一個公開頁面，show 晒你啲 service 係 up 定 down：

```
Settings → Status Page → Add Status Page
```

選你想 show 出嚟嘅 monitor，可以加 group（例如「Core Services」、「Databases」），然後 Uptime Kuma 會俾你一個 public URL。

呢個可以俾其他人睇（或者俾你自己喺街外快速 check 一切正常）。

---

## 同 Beszel 嘅分工

```
Beszel:    「部機嘅 CPU 爆咗，RAM 就滿，disk 得返 5%」
Uptime:   「https://beszel.homelab.deven.tw 仲係 return 200 ✅」
           「SSL cert 14 日後到期 ⚠️」
           「Blog load time 由 200ms 升到 3s，可能有問題」
```

Beszel 睇內部健康，Uptime Kuma 睇外部可用性，兩個加埋先係全面 monitoring。

---

## Troubleshooting

### Monitor 成日 False Positive

某啲 website load 得慢少少就 trigger down？加 **Retry** 或者加長 **Interval**。

### 收唔到 Notification

喺 notification setting 㩒 **「Test」** 制，會 send 一個 test message。收唔到就代表設定錯咗。

### Docker container 入面監控 host service

Uptime Kuma 喺 Docker 入面，用 `http://localhost:xxx` 係連唔到 host 嘅 service 㗎，要用 host 嘅 LAN IP（例如 `http://192.168.31.188:8090`）。
