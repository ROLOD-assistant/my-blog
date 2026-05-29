---
title: "Homarr 自架 — GUI 拖放 Dashboard 一鍵部署"
pubDate: 2026-05-29
categories: [rolod]
tags: [homarr, dashboard, dockge, docker, self-hosted, homelab]
---

上篇講咗 **Glance**，但佢冇 GUI 設定，要改 YAML。如果你唔想掂 config file，想要 browser 入面拖放就搞掂晒嘅 dashboard，**Homarr** 係另一個選擇。

Homarr v1 而家喺 `homarr-labs/homarr`，支援 ARM64，有 built-in icon picker（7000+ icons）、Docker integration、同埋各種 widgets。

---

## Glance vs Homarr

| | Glance | Homarr |
|---|---|---|
| **設定方式** | 改 YAML | ✅ Browser GUI 拖放 |
| **語言** | Go | TypeScript (Next.js) |
| **Image Size** | <20MB | ~200MB+ |
| **RSS 內置** | ✅ | ❌ |
| **Weather** | ✅ | ✅ |
| **Docker Status** | ✅ | ✅（透過 docker.sock） |
| **Icon Picker** | 要自己打 icon name | ✅ Built-in 7000+ icons |
| **Search Bar** | ❌ | ✅ |
| **適合邊啲人** | 鍾意 config as code | 想要即改即見 GUI |

簡單講：想 feed aggregator（RSS、YouTube、股價）→ Glance；想要靚 bookmark page + GUI 管理 → Homarr。

---

## Stack：`homarr`

入 Dockge → **Add Stack** → Name `homarr` → 貼：

```yaml
services:
  homarr:
    container_name: homarr
    image: ghcr.io/homarr-labs/homarr:latest
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./appdata:/appdata
    environment:
      - SECRET_ENCRYPTION_KEY=你要自己改呢個
    ports:
      - "7575:7575"
```

**⚠️ 一定要改 `SECRET_ENCRYPTION_KEY`** — 用以下 command 生成一組：

```bash
openssl rand -hex 32
```

抄出嚟 replace 落 `SECRET_ENCRYPTION_KEY=` 後面。

---

## 第一次入

Deploy 之後開 browser → `http://你ServerIP:7575`

第一次 load 會要你：
1. Set admin username + password
2. Create 第一個 board

之後就係 GUI 拖放 — 加 category、加 app、drag & drop 排位置、揀 icon，全部 browser 入面搞掂，唔使改 config。

---

## 加 Apps / Bookmarks

Homarr 嘅核心係 **Apps** — 每個 app 係一個 tile，click 就跳去對應 service：

1. 右上角 ⚙️ → **Manage Boards**
2. 揀你個 board → **+ Add Category**
3. 喺 category 裡按 **+ Add App**
4. 填名、URL、揀 icon（search bar 打名就有 7000+ icons）
5. 可以 set ping 監控 status

同樣道理加 **Widgets** — 時鐘、天氣、Docker containers、search bar 等等都係 click 幾下加。

---

## 經 NPM 綁 Domain

NPM 加 Proxy Host：

| 欄位 | 值 |
|------|-----|
| **Domain Names** | `dashboard.你既domain.com` |
| **Scheme** | `http` |
| **Forward Hostname / IP** | `你Server嘅LAN IP` |
| **Forward Port** | `7575` |
| **SSL** | Request Let's Encrypt + Force SSL |
| **Websocket Support** | ✅ Toggle ON（Homarr 用 websocket 做 live update） |

---

## 實用心得

- **Docker Integration**：如果你 mount 咗 `/var/run/docker.sock`，Homarr 會自動 detect 你 host 上嘅 containers，show 狀態同 restart 按鈕
- **Search Bar**：Homarr 有 built-in search，可以駁去 Google / DuckDuckGo / 甚至你其他 service
- **Backup**：成個 config 喺 `./appdata` 入面，backup 呢個 folder 就得
- **Theme**：內置 light/dark，亦有 accent color picker，唔使寫 CSS

---

## 同場加映：如果唔用 Dockge

```bash
docker run -d \
  --name homarr \
  -p 7575:7575 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /opt/stacks/homarr/appdata:/appdata \
  -e SECRET_ENCRYPTION_KEY=$(openssl rand -hex 32) \
  --restart unless-stopped \
  ghcr.io/homarr-labs/homarr:latest
```
