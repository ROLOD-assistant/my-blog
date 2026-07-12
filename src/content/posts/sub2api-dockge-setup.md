---
title: "Sub2API 自架 — Dockge 一鍵部署（AI API Gateway）"
pubDate: 2026-07-12
categories: [rolod]
tags: [rolod, sub2api, dockge, docker, self-hosted, homelab, ai-api]
---

> Sub2API — AI API Gateway Platform。將你嘅 AI subscription（ChatGPT、Claude、Midjourney）變成 API，key-based 配額管理、用量追蹤。呢篇係純粹自己 deploy 嘅筆記，用 Dockge + PostgreSQL + Redis，乾淨企理。

## Sub2API 係咩

Sub2API 係一個 AI API Gateway Platform，主要用嚟將 AI 產品嘅 subscription quota 包裝成 API，分發俾唔同 user。

簡單講：你有一張 ChatGPT Plus，想 team 入面 10 個人都用到。正常方法係 share login — 但你唔想俾 password 出去，又想知道邊個用得最多。Sub2API 幫你做晒：

1. 將你嘅 subscription 掛上去（OpenAI key、ChatGPT session、Claude、Midjourney）
2. 生成 API key 俾每個 user
3. 每個 key 有獨立 quota / rate limit
4. Dashboard 睇到用量報表

類似工具比較：

| | Sub2API | OneAPI | NewAPI |
|---|---|---|---|
| **用途** | AI subscription 配額管理 | API 聚合轉發 | API 聚合轉發 |
| **Database** | PostgreSQL + Redis | MySQL + Redis | MySQL + Redis |
| **Image Size** | ~70MB | ~100MB | ~120MB |
| **Subscription 支援** | ✅ ChatGPT Plus / Claude Pro / MJ | ❌ 淨係 API key | ❌ 淨係 API key |
| **Quota 管理** | ✅ built-in | ✅ | ✅ |
| **Dashboard** | ✅ | ✅ | ✅ |

Sub2API 嘅獨特賣點：佢唔只係夾 API key，係識得用 **subcription token** — 即係你可以 Share 緊嘅 ChatGPT Plus 月費 plan，唔係開多個 API paid account。

## Stack YAML（Dockge）

```yaml
services:
  sub2api:
    image: weishaw/sub2api:latest
    container_name: sub2api
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgres://postgres:37d5f75b9d82120cc0044536499c968b@db:5432/sub2api?sslmode=disable
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis
    restart: unless-stopped

  db:
    image: postgres:15-alpine
    container_name: sub2api-db
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=37d5f75b9d82120cc0044536499c968b
      - POSTGRES_DB=sub2api
    volumes:
      - ./postgres-data:/var/lib/postgresql/data
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    container_name: sub2api-redis
    volumes:
      - ./redis-data:/data
    restart: unless-stopped
```

> ⚠️ 上面嘅 `POSTGRES_PASSWORD` 同 `DATABASE_URL` 入面嘅 password 要一致。呢度用 `openssl rand -hex 16` generate 咗一個做 example，記得自己改另一個 secure password，可以用 `openssl rand -hex 32` generate。`DATABASE_URL` 裡面嘅 password 都要同步改。

**部署步驟：**

1. Dockge → Add Stack → 改名 `sub2api`
2. Paste 上面 YAML → Deploy
3. 等 10 秒，`docker compose logs sub2api` confirm 冇 error
4. 開 browser → `http://你ServerIP:8080`
5. 首次開機會見到 setup wizard，Database Configuration 記得 **Host 填 `db`**（唔係 `localhost`），因為 sub2api container 要用 Docker internal DNS 去連 PostgreSQL container：
   - **Host:** `db`
   - **Port:** `5432`
   - **Username:** `postgres`
   - **Password:** 你 compose 入面 set 嘅 `POSTGRES_PASSWORD`
   - **Database Name:** `sub2api`
   - **SSL Mode:** Disable
6. Register admin account → 開始加 subscription 🎉

## 點解用 internal network？

呢個 compose 冇 define custom network，Docker 會自動放佢哋落 default bridge network，service name resolve 到就得。

Postgres 同 Redis 冇 `ports:` mapping，即係唔會佔 host port。就算你 host 本身已經有個 real PostgreSQL 行緊，完全唔會撞 port。Sub2API 透過 Docker internal DNS（service name `db` / `redis`）連過去，外界 ping 唔到個 DB port。

## 環境變數

| Variable | Required | Default | Description |
|---|---|---|---|
| `DATABASE_URL` | ✅ | — | PostgreSQL connection string |
| `REDIS_URL` | ✅ | — | Redis connection string |
| `PORT` | ❌ | `8080` | Server port |
| `GIN_MODE` | ❌ | `release` | Gin framework mode (debug/release) |

## NPM Reverse Proxy

如果你用 Nginx Proxy Manager 擺出去俾人用：

| 欄位 | 值 |
|---|---|
| **Domain Names** | `api.你既domain.com` |
| **Scheme** | `http` |
| **Forward Hostname / IP** | 你 Server LAN IP |
| **Forward Port** | `8080` |
| **SSL** | Request Let's Encrypt + Force SSL |
| **Websocket Support** | ❌ 唔使 |

## 實用心得

1. **首次開 browser** 去 `http://你ServerIP:8080` 會叫你 register account，呢個係 admin
2. **加 subscription** — 支援 OpenAI API key、ChatGPT Plus session token、Claude Pro、Midjourney。每種 provider 嘅拎法唔同，dashboard 有說明
3. **Generate API key** — 每個 key 可以 set 獨立 quota（每日 / 每月 / 總額），俾邊個用就 set 邊個
4. **用量報表** — dashboard 有 chart，睇到每條 key、每個 model 嘅 call count / token usage
5. **Backup** — `./postgres-data` 同 `./redis-data` 係重要 data，backup 呢兩個 folder 就夠
6. **Update** — 去 Dockge stop stack → 改 image tag → deploy，就 pull 最新版

Postgres + Redis 比 SQLite 穩定好多。Production 用一定要行 PostgreSQL，唔好貪方便 skipped。

參考官方 Docker Hub：[weishaw/sub2api](https://hub.docker.com/r/weishaw/sub2api)（500K+ pulls，9 stars）
