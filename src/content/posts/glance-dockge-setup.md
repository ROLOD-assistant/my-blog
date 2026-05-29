---
title: "Glance Dashboard 自架 — Dockge 一鍵部署"
pubDate: 2026-05-29
categories: [rolod]
tags: [glance, dashboard, dockge, docker, self-hosted, homelab]
---

之前寫咗 Monitoring 系列、Miniflux RSS，今次講 **Dashboard**。

Homelab 越整越多 service，總要有個 default page 一次過睇晒所有嘢。**Glance** 就係做呢樣嘢 — 一個輕量、自定義 dashboard，將你嘅 RSS、Weather、Server Status、Market Price、YouTube 等等全部放喺同一頁。

---

## 點解係 Glance 唔係其他？

| | Glance | Homer / Flame | Heimdall | Homer Dashboard |
|---|---|---|---|---|
| **語言** | Go | Node | PHP | Go |
| **Image Size** | <20MB | ~200MB+ | ~100MB+ | ~20MB |
| **Widget 類型** | RSS, Weather, Markets, YouTube, Reddit, Docker Status, 等等 | 純 links | 純 links | 純 links |
| **RSS Feed** | ✅ 內置 | ❌ | ❌ | ❌ |
| **Mobile 優化** | ✅ | 一般 | 一般 | 一般 |

Glance 唔係 bookmark page — 佢係 **feed aggregator + dashboard**。你 set 好啲 widget，佢自動 fetch 晒 RSS、YouTube uploads、天氣、股價、server 狀態，一個 page 睇晒。

---

## Stack：`glance`

入 Dockge → **Add Stack** → Name `glance` → 貼：

```yaml
services:
  glance:
    image: glanceapp/glance
    container_name: glance
    restart: unless-stopped
    volumes:
      - ./config:/app/config
    ports:
      - "8480:8080"
```

Deploy 之後，Dockge 會自動開咗個 `glance/` directory 俾你。

---

## Config 設定

去你 Server SSH 入 Dockge 嘅 stacks path：

```bash
cd /opt/stacks/glance
mkdir config
wget -O config/glance.yml https://raw.githubusercontent.com/glanceapp/glance/refs/heads/main/docs/glance.yml
```

然後改 `config/glance.yml` — 呢個係官方範例，已經有晒基本 widget。你淨係改入面嘅 location、feed URL、market symbols 就得。

**簡單 demo config 俾你快速開始：**

```yaml
pages:
  - name: Home
    columns:
      - size: small
        widgets:
          - type: calendar
            first-day-of-week: monday

          - type: weather
            location: 你既城市
            units: metric
            hour-format: 24h

          - type: markets
            markets:
              - symbol: SPY
                name: S&P 500
              - symbol: BTC-USD
                name: Bitcoin

      - size: full
        widgets:
          - type: hacker-news

          - type: rss
            limit: 10
            feeds:
              - url: https://selfh.st/rss/
                title: selfh.st
              - url: https://ciechanow.ski/atom.xml
                title: 技術文章

          - type: videos
            channels:
              - UCXuqSBlHAE6Xw-yeJA0Tunw  # Linus Tech Tips
              - UCsBjURrPoezykLs9EqgamOA  # Fireship
              - UCHnyfMqiRRG1u-2MsSQLbXA  # Veritasium

      - size: small
        widgets:
          - type: reddit
            subreddit: selfhosted
            show-thumbnails: true

          - type: releases
            repositories:
              - glanceapp/glance
              - immich-app/immich
              - syncthing/syncthing
```

改完之後喺 Dockge 重新 deploy（或 `docker compose restart`）。

---

## 經 NPM 綁 Domain

NPM 加 Proxy Host：

| 欄位 | 值 |
|------|-----|
| **Domain Names** | `glance.你既domain.com` |
| **Scheme** | `http` |
| **Forward Hostname / IP** | `你Server嘅LAN IP` |
| **Forward Port** | `8480` |
| **SSL** | Request Let's Encrypt + Force SSL |

之後就 `https://glance.你既domain.com` 入。

---

## 實用心得

- **Cache Time**：Glance 預設會 cache widget 結果，唔係每次都 fetch 一次。你可以喺每個 widget 加 `cache: 30m` 手動控制
- **Colapse After**：RSS widget 可以用 `collapse-after: 3` ，長 feed 自動摺疊，慳位
- **Theme**：支援 custom theme，改顏色系統就得，唔使寫 CSS（除非你想）
- **Mobile**：Mobile 版自動 adapt，返工搭車都睇到

---

## 同場加映：如果唔用 Dockge

如果你直接用 Docker CLI：

```bash
docker run -d \
  --name glance \
  -p 8480:8080 \
  -v /opt/stacks/glance/config:/app/config \
  --restart unless-stopped \
  glanceapp/glance
```

一樣係單一 container，冇 database、冇 dependency，run 完即用。
