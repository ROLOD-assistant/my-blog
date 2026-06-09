---
title: "n8n 自架 — Dockge 一鍵部署（PostgreSQL backend）"
pubDate: 2026-06-08
categories: [rolod]
tags: [n8n, dockge, docker, self-hosted, homelab, postgresql, automation]
---

> n8n — 自架版 Zapier / Make。呢篇係純粹自己 deploy 嘅筆記，用 Dockge + PostgreSQL backend，internal network 唔開 port，乾淨企理。

## n8n 係咩

n8n 係一個 workflow automation 平台。你可以用 visual node editor 砌 automation：收到 email → 寫入 Google Sheets → send Slack message、或者 cron trigger → fetch API → transform data → write to DB。全部 self-hosted，data 唔經第三方。

類似工具比較：

| | n8n | Node-RED | Huginn | Activepieces |
|---|---|---|---|---|
| **Language** | TypeScript | Node.js | Ruby | TypeScript |
| **Image Size** | ~200MB | ~300MB | ~500MB | ~300MB |
| **UI** | Node editor | Flow editor | Web UI | Node editor |
| **REST API** | ✅ | ❌ | ✅ | ✅ |
| **Auth / Users** | ✅ Built-in | ❌ | ❌ | ✅ |
| **Community Nodes** | 400+ | 5000+ | 少 | 少 |

n8n 適合：想寫 automation 但唔想寫 code 的人；Node-RED 適合 IoT / MQTT hardware；Huginn 適合純 RSS/ scraping agent。

## Stack YAML

Dockge-ready compose，用 **internal network**，postgres 唔 expose port：

```yaml
services:
  postgres:
    image: postgres:16-alpine
    container_name: n8n-db
    restart: unless-stopped
    environment:
      - POSTGRES_USER=n8n
      - POSTGRES_PASSWORD=***
      - POSTGRES_DB=n8n
    volumes:
      - ./postgres-data:/var/lib/postgresql/data
    networks:
      - n8n-net

  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=n8n.你既domain.com
      - N8N_PROTOCOL=https
      - N8N_PORT=5678
      - NODE_ENV=production
      - WEBHOOK_URL=https://n8n.你既domain.com
      - GENERIC_TIMEZONE=Asia/Hong_Kong
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=***
      - DB_POSTGRESDB_DATABASE=n8n
    volumes:
      - ./data:/home/node/.n8n
    depends_on:
      - postgres
    networks:
      - n8n-net

networks:
  n8n-net:
    driver: bridge
```

> ⚠️ `POSTGRES_PASSWORD=***` — 自己改一個 secure password，可以用 `openssl rand -hex 32` generate。
>
> 參考官方文檔：https://docs.n8n.io/hosting/installation/docker/

### 點解用 internal network？

postgres 冇 `ports:` mapping，即係唔會佔 host 既 5432。就算你 host 本身已經有個 real PostgreSQL 行緊，完全唔會撞 port。
n8n 透過 Docker internal DNS（service name `postgres`）連過去，外界 ping 唔到個 DB port。

## NPM Reverse Proxy

| 欄位 | 值 |
|------|-----|
| **Domain Names** | `n8n.你既domain.com` |
| **Scheme** | `http` |
| **Forward Hostname / IP** | 你 Server LAN IP |
| **Forward Port** | `5678` |
| **SSL** | Request Let's Encrypt + Force SSL |
| **Websocket Support** | ✅ Toggle ON |

## 實用心得

1. **首次開 browser** 去 `n8n.你既domain.com` 會叫你 set owner account，呢個係 global admin
2. **Credentials** 係 encrypted at rest，encryption key auto-generated — backup `/home/node/.n8n` 既話記得 keep 埋
3. 如果用 **node with webhook**（Telegram、Slack 等），NPM 個 Websocket Support 要開
4. Postgres 比 default SQLite 穩定好多 — n8n 官方都 recommend production 用 Postgres

## 同場加映：直接用現有 host DB

如果你本身已經有 PostgreSQL 行緊，可以 skip 個 postgres container：

```yaml
services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=n8n.你既domain.com
      - N8N_PROTOCOL=https
      - N8N_PORT=5678
      - NODE_ENV=production
      - WEBHOOK_URL=https://n8n.你既domain.com
      - GENERIC_TIMEZONE=Asia/Hong_Kong
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=host.docker.internal
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_USER=xxx
      - DB_POSTGRESDB_PASSWORD=***
      - DB_POSTGRESDB_DATABASE=n8n
    volumes:
      - ./data:/home/node/.n8n
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

不過 homelab 環境用上面 internal network + dedicated postgres 其實最乾淨 — 搬走去另一部機都係一個 compose 搞掂。

---

*Deployed via Hermes Agent on a 2026 MacBook Pro.*
