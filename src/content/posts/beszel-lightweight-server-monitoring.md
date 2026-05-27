---
title: "自架 Beszel：極輕量伺服器監控方案"
pubDate: 2026-05-27
categories: [rolod]
tags: [beszel, monitoring, devops, self-hosted, docker]
---

屋企有幾部 VPS 或者自己㗳 Docker 嘅朋友，搵一個輕身又夠用嘅 monitoring 工具係永恆嘅課題。Netdata 太肥、Prometheus + Grafana 太重型、Uptime Kuma 只 cover uptime。如果你只需要一個簡單、靚仔、食資源少到幾乎 feel 唔到嘅監控方案，**Beszel** 值得一試。

## Beszel 係咩？

Beszel 係一個開源 server monitoring 平台，用 Go 寫，成個 system 得兩個 component：

- **Hub** — Web 儀表板，睇晒所有機嘅數據（基於 PocketBase）
- **Agent** — 裝喺每部要監控嘅機器上，採集數據經 WebSocket 傳返 Hub

GitHub 上目前 22.2k stars，MIT 授權，社群好活躍。

## 有咩賣點？

### 1. 超細 volume

Hub 嘅 Docker image 約 **15MB**，Agent 約 **9MB**。相比 Netdata（~300MB）或者 Grafana（~400MB），呢個係完全另一個量級。RAM 佔用通常喺 20-50MB 左右，行喺 1GB RAM 嘅細 VPS 上都 feel 唔到。

### 2. 一條 docker-compose 搞掂

Hub 嘅部署就係：

```yaml
services:
  beszel:
    image: henrygd/beszel
    container_name: beszel
    restart: unless-stopped
    ports:
      - 8090:8090
    volumes:
      - ./beszel_data:/beszel_data
```

Agent 都係差唔多長度。由 clone 到見到 dashboard，五分鐘唔使。

### 3. 睇到 Docker container 層級

一般 monitoring 只睇到成部機嘅 CPU/RAM，Beszel 仲可以 show 到每個 container 嘅資源佔用，對於自己 host 好多 service 嘅人來講好實用。

### 4. Alert 齊全

支援 CPU、記憶體、磁碟、bandwidth、溫度、load average 嘅 threshold alert，可以經 Discord、Telegram、Slack、Email、Webhook 等送出通知。仲有 S.M.A.R.T. 硬碟健康監測，壞碟前收到預警。

### 5. Multi-user + OAuth

可以開賬號俾人，各自管理自己嘅 system。支援 OAuth2 登入（GitHub、Google、Discord 等），唔想記密碼都得。

### 6. Automatic backup

支援本地 disk 同 S3-compatible storage 自動備份，驚 data loss 嘅話可以 set 好就唔使理。

## 我自己點用

而家用緊嘅 setup 係咁：

```
┌─────────────────┐       WebSocket       ┌──────────────────┐
│  Dokploy 主機    │ ◄────────────────── │    VPS #1        │
│  (Beszel Hub)    │    port 45876        │  (Beszel Agent)  │
│  port 8090       │                      │                   │
│                   │ ◄────────────────── │    VPS #2        │
│                   │    port 45876        │  (Beszel Agent)  │
└─────────────────┘                      └──────────────────┘
```

Hub 行喺 Dokploy（自己 host 嘅 PaaS）上面，每部 VPS 用 Docker 起一個 Agent container，全部經 WebSocket 連返 Hub。Dashboard 一頁睇晒所有機嘅 CPU、RAM、Disk、Network。

Dashboard 大概係咁樣：

![Beszel Dashboard](https://raw.githubusercontent.com/henrygd/beszel/main/screenshots/dashboard.png)

每個 system 點入去仲睇到 historical charts，睇到過去一段時間嘅資源變化趨勢。

## 監控到咩 metric？

- CPU 使用率（每 core + overall）
- Memory（RAM + Swap）
- Disk usage（可以 mount 額外 partition）
- Network traffic（RX/TX）
- Docker container 層級資源
- GPU 使用率（NVIDIA）
- System temperature
- S.M.A.R.T. 硬碟健康
- Load average
- Process count

## 安裝快速回顧

**Hub** — 一行 docker-compose 搞掂，或者 binary 直接行。

**Agent** 有幾種裝法：

| 方法 | 適用場景 |
|------|---------|
| Docker | 有 Docker 嘅機器，最簡單 |
| Binary | 冇 Docker 嘅 VPS／資源好慳嘅機器 |
| Homebrew | macOS |
| WinGet / Scoop | Windows |
| HA Add-on | Home Assistant |

Agent 唔需要 expose 任何 port 俾 public — 佢主動 outbound WebSocket 連去 Hub，安全性上好啲。

## 同其他方案比較

| | Beszel | Netdata | Prometheus + Grafana | Uptime Kuma |
|---|---|---|---|---|
| Image size | ~15MB | ~300MB | ~600MB+ | ~30MB |
| RAM usage | ~30MB | ~200MB | ~300MB+ | ~50MB |
| Docker stats | ✅ | ✅ | 要 setup | ❌ |
| Alert | ✅ | ✅ | ✅ | ✅ |
| Historical data | ✅ | ✅ | ✅ | ❌ |
| Setup complexity | ⭐ | ⭐⭐ | ⭐⭐⭐⭐ | ⭐ |

## 總結

Beszel 適合嘅人：

- 有 2-5 部 VPS 想統一 monitoring
- 想要簡單 setup、唔想㗳 Prometheus + Grafana 嗰套
- 資源有限（1GB RAM VPS）
- 需要 Docker container 層級監控
- 想要 alert 但唔需要太複雜

唔適合嘅人：

- 需要超細粒度嘅 custom metric
- 需要複雜嘅 dashboard 自訂
- 企業級多 team 架構

對我來講，Beszel 係「啱啱好」嘅 monitoring — 唔會太少資訊，又唔會 overload。如果你都係自己 host 幾部機嘅人，值得試下。

---

**Links:**
- [GitHub](https://github.com/henrygd/beszel)
- [Official Docs](https://beszel.dev)
- [Agent Installation Guide](https://beszel.dev/guide/agent-installation)
