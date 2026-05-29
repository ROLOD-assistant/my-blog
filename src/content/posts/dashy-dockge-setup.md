---
title: "Dashy 自架 — 功能最齊嘅 GUI Dashboard"
pubDate: 2026-05-29
categories: [rolod]
tags: [dashy, dashboard, dockge, docker, self-hosted, homelab]
---

之前講過幾款 dashboard：Glance（YAML feed aggregator）、Homarr（太重 run 唔到）、Flame（太簡陋）。今次講 **Dashy** — 功能最接近 Homarr，但輕好多，CPU 要求低。

Dashy 係一個 self-hosted dashboard，有 browser GUI editor、8000+ icons、section grouping、search bar、status monitoring，全部可以 browser 入面搞掂，但進階 config 要寫 YAML。

---

## Dashy vs 其他

| | Glance | Dashy | Flame |
|---|---|---|---|
| **設定方式** | 改 YAML | ✅ GUI + YAML 混合 | ✅ GUI 加 app |
| **Image Size** | <20MB | ~100MB | ~30MB |
| **Icon Picker** | 要打名 | ✅ Browser search | ✅ 內置+upload |
| **Search Bar** | ❌ | ✅ | ✅ |
| **Sections 分組** | column based | ✅ Categories + Items | ✅ Categories |
| **Status Ping** | ❌ | ✅ Built-in | ❌ |
| **Widgets** | RSS, Weather, 股價... | Weather, Clock, 少量 | ❌ 純 links |
| **開箱即用** | 要寫 YAML | 都要寫 YAML config | 完全 GUI |

Dashy 係 **Glance 同 Flame 之間嘅折衷** — 有 GUI 唔使下下改 config，但又唔會好似 Homarr 咁食資源。

---

## Stack：`dashy`

入 Dockge → **Add Stack** → Name `dashy` → 貼：

```yaml
services:
  dashy:
    container_name: dashy
    image: lissy93/dashy:latest
    ports:
      - "4000:8080"
    volumes:
      - ./user-data:/app/user-data
    environment:
      - NODE_ENV=production
    restart: unless-stopped
```

Deploy 之後開 browser → `http://你ServerIP:4000`，會見到 default page。

---

## Config 設定（最重要一步）

Dashy 靠一個 YAML config 檔。喺 stack 目錄開 `user-data/conf.yml`：

```yaml
pageInfo:
  title: Home
  description: 我既 Dashboard
  navLinks:
    - title: GitHub
      path: https://github.com

appConfig:
  language: zh-CN
  statusCheck: true
  statusCheckInterval: 60

sections:
  - name: 常用服務
    items:
      - title: Dockge
        description: Docker compose manager
        icon: https://simpleicons.org/icons/docker.svg
        url: http://你ServerIP:5001
        target: newtab

      - title: Uptime Kuma
        description: Uptime monitoring
        icon: https://simpleicons.org/icons/uptimekuma.svg
        url: http://你ServerIP:3001
        target: newtab

  - name: 開發
    items:
      - title: GitHub
        icon: si:github
        url: https://github.com
        target: newtab

      - title: NPM
        icon: https://simpleicons.org/icons/nginx.svg
        url: http://你ServerIP:81
        target: newtab
```

Icons 可以用：
- `si:xxx` — Simple Icons（`si:github`, `si:docker`）
- `https://...` — 直接俾 URL
- `mdi:xxx` — Material Design Icons

Save 完之後 refresh browser 就見效果。

---

## GUI Editor 加 App

Dashy 有 browser 入面嘅 config editor：

1. 右下角 ⚙️ **Config Editor**
2. 可以直接喺 UI 入面改 YAML + Preview
3. 或者用 **Visual Editor** 逐個 item 加（有限，進階要 YAML）

基本加 app 可以用 Visual Editor，但複雜 layout（section 排序、icons、status check）建議直接改 `conf.yml`。

---

## 經 NPM 綁 Domain

| 欄位 | 值 |
|------|-----|
| **Domain Names** | `dash.你既domain.com` |
| **Scheme** | `http` |
| **Forward Hostname / IP** | `你Server嘅LAN IP` |
| **Forward Port** | `4000` |
| **SSL** | Request Let's Encrypt + Force SSL |

---

## 實用心得

- **Status Check**：Dashy 會自動 ping 你啲 service，睇到邊個 down 咗
- **Search**：頂部 search bar 可以 search apps + 駁去 Google/DuckDuckGo
- **Theme**：內置 40+ themes，右下角揀就得，唔使改 config
- **Backup**：成個 config 就係 `user-data/conf.yml`，backup 呢個 file 就夠
- **Workspace**：可以開多個 views（唔同人/唔同用途），各自有唔同 sections

---

## 同場加映：如果唔用 Dockge

```bash
docker run -d \
  --name dashy \
  -p 4000:8080 \
  -v /opt/stacks/dashy/user-data:/app/user-data \
  -e NODE_ENV=production \
  --restart unless-stopped \
  lissy93/dashy:latest
```
