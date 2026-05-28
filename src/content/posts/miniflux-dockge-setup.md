---
title: "Miniflux 自架 — 輕量 RSS Reader"
pubDate: 2026-05-28
categories: [rolod]
tags: [miniflux, rss, docker, dockge, self-hosted]
---

之前講咗 monitoring 系列，今次講另一個 homelab 必備 — **RSS reader**。

Miniflux 係最輕量嘅 RSS reader，Go 寫，image 得 ~10MB，唔使 JS，純 keyboard shortcut，食資源極少。

---

## Stack：`miniflux`

入 Dockge → **Add Stack** → Name `miniflux` → 貼：

```yaml
services:
  miniflux:
    image: miniflux/miniflux:latest
    container_name: miniflux
    restart: unless-stopped
    ports:
      - "8380:8080"
    depends_on:
      - miniflux-db
    environment:
      - DATABASE_URL=postgres://miniflux:miniflux_pass@miniflux-db/miniflux?sslmode=disable
      - RUN_MIGRATIONS=1
      - CREATE_ADMIN=1
      - ADMIN_USERNAME=admin
      - ADMIN_PASSWORD=admin123

  miniflux-db:
    image: postgres:16-alpine
    container_name: miniflux-db
    restart: unless-stopped
    volumes:
      - miniflux_db_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_USER=miniflux
      - POSTGRES_PASSWORD=miniflux_pass
      - POSTGRES_DB=miniflux

volumes:
  miniflux_db_data:
```

Deploy 之後等 30 秒。

開 browser → `http://192.168.31.188:8380`

Login：`admin` / `admin123`

然後去 **Feeds → Add** → 貼你嘅 RSS URL。

---

## 如果要用你現有嘅 postgres

如果你唔想開多個 postgres container，可以用返你已經有嘅 `my-postgres`：

```yaml
services:
  miniflux:
    image: miniflux/miniflux:latest
    container_name: miniflux
    restart: unless-stopped
    ports:
      - "8380:8080"
    environment:
      - DATABASE_URL=postgres://admin:password@192.168.31.188:5432/miniflux?sslmode=disable
      - RUN_MIGRATIONS=1
      - CREATE_ADMIN=1
      - ADMIN_USERNAME=admin
      - ADMIN_PASSWORD=admin123
```

但要先去 my-postgres 手動開一個 `miniflux` database：

```bash
docker exec -u postgres my-postgres createdb -U admin miniflux
```

---

## 經 NPM 綁 Domain

同之前一樣，NPM 加 Proxy Host：

| 欄位 | 值 |
|------|-----|
| **Domain Names** | `rss.homelab.deven.tw` |
| **Scheme** | `http` |
| **Forward Hostname / IP** | `192.168.31.188` |
| **Forward Port** | `8380` |
| **SSL** | Request Let's Encrypt + Force SSL |

之後就 `https://rss.homelab.deven.tw` 入。

---

## 基本操作

- **Add Feed**：右上角 → Feeds → Add → 貼 RSS URL
- **Keyboard shortcut**：`j/k` 上下，`v` 開 link，`s` 轉 scope
- **Mark all as read**：Shift + a
- **分類**：Categories 可以幫 feed 分組

Miniflux 仲支援 Fever API，可以駁去 Reeder / NetNewsWire 等手機 app 睇。

---

同場加映：如果你想 self-host **RSSHub**（幫冇 RSS 嘅 site 生成 feed），都係一個 container 搞掂：

```yaml
services:
  rsshub:
    image: diygod/rsshub:latest
    container_name: rsshub
    restart: unless-stopped
    ports:
      - "1200:1200"
```

然後 Miniflux 可以訂閱 `http://192.168.31.188:1200` 嘅 RSSHub feed。
