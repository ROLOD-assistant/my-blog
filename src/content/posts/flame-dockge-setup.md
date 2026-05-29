---
title: "Flame 自架 — 最輕量 GUI Dashboard"
pubDate: 2026-05-29
categories: [rolod]
tags: [flame, dashboard, dockge, docker, self-hosted, homelab]
---

前幾篇講咗 Glance（YAML config）同 Homarr（功能多但食資源），今次講最輕、最簡單嘅選擇 — **Flame**。

Flame 係一個純 bookmark dashboard：Python base，image 得 ~30MB，有 browser GUI 加 app，唔使掂 config file，CPU 極低要求，任何機都行到。

---

## 定位

| | Glance | Homarr | Flame |
|---|---|---|---|
| **設定方式** | 改 YAML | GUI 拖放 | ✅ GUI 加 app |
| **Image Size** | <20MB | ~200MB | **~30MB** |
| **CPU 要求** | 極低 | 高（Node 16 + Redis 8） | **極低（Python）** |
| **Widgets / Feed** | RSS, Weather, 股價... | 大量 integrations | ❌ 純 links |
| **Search Bar** | ❌ | ✅ | ✅ |
| **適合** | config as code | 功能齊全 | 最簡單 bookmark page |

Flame 唔會取代 Glance —— 佢哋定位唔同。Glance 係 feed aggregator，Flame 係靚嘅 bookmark page。

---

## Stack：`flame`

入 Dockge → **Add Stack** → Name `flame` → 貼：

```yaml
services:
  flame:
    image: pawelmalak/flame:latest
    container_name: flame
    restart: unless-stopped
    volumes:
      - ./config:/app/data
    ports:
      - "7573:5005"
```

Deploy 就得。

---

## 第一次入

開 browser → `http://你ServerIP:7573`

會見到 default page，右上角 ⚙️ **Settings**：

1. Set 一個 password（optional）
2. 加返你個 dashboard 名
3. 開 **Add App** 開始加 service links

每個 app 可以 set：
- 名
- URL
- Icon（內置圖示或者自己 upload）
- 分類（Categories）

Drag & drop 排次序都得。

---

## 經 NPM 綁 Domain

NPM 加 Proxy Host：

| 欄位 | 值 |
|------|-----|
| **Domain Names** | `dashboard.你既domain.com` |
| **Scheme** | `http` |
| **Forward Hostname / IP** | `你Server嘅LAN IP` |
| **Forward Port** | `7573` |
| **SSL** | Request Let's Encrypt + Force SSL |

---

## 實用心得

- **Password protect**：Settings 入面 set 密碼，開咗之後每次入 dashboard 要 login
- **Search**：頂部 search bar 可以 set default search engine（Google / DuckDuckGo / 自訂）
- **Backup**：成個 config 喺 `./config` 嘅 database file，backup 呢個 folder 就夠
- **External link**：可以加 external link 開新 tab，或者 internal link 用 iframe 嵌入

---

## 同場加映：如果唔用 Dockge

```bash
docker run -d \
  --name flame \
  -p 7573:5005 \
  -v /opt/stacks/flame/config:/app/data \
  --restart unless-stopped \
  pawelmalak/flame:latest
```
