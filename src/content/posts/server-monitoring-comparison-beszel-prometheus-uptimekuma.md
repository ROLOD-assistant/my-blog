---
title: "伺服器監控三選一：Beszel vs Prometheus+Grafana vs Uptime Kuma"
pubDate: 2026-05-27
categories: [rolod]
tags: [beszel, prometheus, grafana, uptime-kuma, monitoring, devops, self-hosted]
---

之前寫咗篇 Beszel 介紹，有人問同其他方案點比。今次直接將三條主流路線攤出嚟講。

先講結論：**揀咩取決於你需要幾多嘢**。唔係愈強大愈好，啱自己用量先係最好。

---

## 三條路線概覽

### Beszel — 輕量 all-in-one

一條 docker-compose 搞掂 Hub + Dashboard，Agent 都係一行 command。目標好清晰：用最少資源、最快 setup，拎到你需要嘅系統監控數據。

- Image size: Hub ~15MB, Agent ~9MB
- RAM usage: ~30-50MB
- Language: Go
- 數據儲存: SQLite（PocketBase）

### Prometheus + Grafana — 工業級監控 stack

Prometheus 做 metrics 採集 + 儲存，Grafana 做可視化儀表板。中間仲可能要加 node_exporter、cAdvisor、Alertmanager 等 component。每一個都係獨立 service，各自食資源。

- Image size: Prometheus ~200MB, Grafana ~400MB
- RAM usage: Prometheus ~150-300MB, Grafana ~100-200MB
- Language: Go + TypeScript
- 數據儲存: TSDB（Prometheus 自帶）

### Uptime Kuma — 純 uptime 監控

唔係系統監控工具，而係 **網站/服務可用性監控**。定期 ping/HTTP 你嘅 service，如果 down 咗就 alert 你。唔會睇 CPU、RAM、Disk 呢啲。

- Image size: ~30MB
- RAM usage: ~50-80MB
- Language: Node.js（Vue.js frontend）
- 數據儲存: SQLite

---

## 逐項比較

### 1. Setup 難度

| | Beszel | Prometheus+Grafana | Uptime Kuma |
|---|---|---|---|
| Docker Compose 行數 | ~10 行 | 50+ 行（起碼 4 個 service） | ~10 行 |
| 學習曲線 | ⭐ | ⭐⭐⭐⭐ | ⭐ |
| 由零到見到 dashboard | 3 分鐘 | 30 分鐘+ | 3 分鐘 |
| 文件品質 | 清楚簡潔 | 多但散亂 | 清楚 |

Beszel 嘅 setup 體驗接近 Uptime Kuma 嘅簡單程度，但做到嘅嘢多好多。Prometheus + Grafana 嘅門檻唔係技術上難，而係你要搞清楚 prometheus.yml 點寫、點樣 discover target、Grafana dashboard 點 import、datasource 點連。

### 2. 系統資源監控能力

| | Beszel | Prometheus+Grafana | Uptime Kuma |
|---|---|---|---|
| CPU 使用率 | ✅ 每 core + overall | ✅ | ❌ |
| Memory | ✅ RAM + Swap | ✅ | ❌ |
| Disk usage | ✅ 可掛 multiple partition | ✅ | ❌ |
| Network traffic | ✅ RX/TX | ✅ | ❌ |
| Docker container | ✅ 內建 | 要加 cAdvisor | ❌ |
| GPU (NVIDIA) | ✅ 內建 | 要加 DCGM exporter | ❌ |
| Temperature | ✅ | 要加 sensor exporter | ❌ |
| S.M.A.R.T. | ✅ 內建 | 要加 smartctl exporter | ❌ |
| Process / Load | ✅ | ✅ | ❌ |

呢個係最明顯嘅分野：Beszel 同 Prometheus+Grafana 係同一個類別（系統監控），Uptime Kuma 係完全唔同嘅嘢（服務可用性監控）。

### 3. Alert 功能

| | Beszel | Prometheus+Grafana | Uptime Kuma |
|---|---|---|---|
| 內建 alert | ✅ CPU/Mem/Disk/BW/Temp/Load | 要加 Alertmanager | ✅ HTTP/Ping/Port |
| Notification channels | Discord, Telegram, Slack, Email, Webhook | 多但逐個 setup | Discord, Telegram, Slack, Email, Webhook 等 |
| Alert 條件自訂 | Threshold based | PromQL + 條件 | 簡單 condition |

Beszel 嘅 alert 係「夠用」— set threshold，過咗就通知。Prometheus 嘅 alert 強大好多，可以用 PromQL 寫複雜條件（例如「過去 5 分鐘 CPU 平均 > 80% 而且 memory 都 > 90% 先 alert」）。但代價係你要學 PromQL 同維護多一個 Alertmanager service。

Uptime Kuma 嘅 alert 係圍繞 downtime 設計嘅 — 「呢個 website down 咗 60 秒」呢類。

### 4. Custom metrics

| | Beszel | Prometheus+Grafana | Uptime Kuma |
|---|---|---|---|
| 可自訂 metric | ❌ 固定 set | ✅ 無限可能性 | ❌ 固定 HTTP/Ping |
| 支援 exporter 生態 | ❌ | ✅ 幾百個 exporter | ❌ |
| PromQL query | ❌ | ✅ | ❌ |
| Dashboard 自訂 | ❌ 固定 UI | ✅ Grafana 完全自訂 | ❌ |

呢度係 Prometheus+Grafana 嘅最大強項。你想監控 MySQL query latency、Nginx request rate、Let's Encrypt cert expiry、WiFi signal strength... 只要有人寫咗 exporter，Prometheus 就食得到。Grafana 嘅 dashboard 仲可以畫到好靚好複雜嘅圖。

Beszel 嘅 philosophy 係「俾你夠用嘅 metric，唔使你諗」。你想要更仔細嘅嘢？Beszel 就俾唔到。

### 5. 資源消耗

呢個係實際數據，唔係吹水：

**100 部機嘅監控場景：**

- **Beszel** — Hub ~50MB RAM + 每部 Agent ~9MB image，total RAM ~100-200MB
- **Prometheus+Grafana** — Prometheus ~300MB + Grafana ~200MB + Alertmanager ~50MB + 每部機 node_exporter ~20MB = total ~600-800MB+，未計 cAdvisor
- **Uptime Kuma** — 一個 service ~80MB，但佢做唔到系統監控

**一部 1GB RAM VPS 自己用：**

Beszel 可以同其他 service 共存而幾乎 feel 唔到。Prometheus+Grafana 食咗你半部機。

### 6. 維護成本

Beszel 係一個 binary，update 就 pull 新 image restart。Prometheus+Grafana 每個 component 各自要 update，config 要逐個 check 兼容性。試過一次 Prometheus upgrade 之後 promQL 語法改咗，dashboard 爛晒，要逐個 fix。

---

## 咩情況下揀邊個

### 揀 Beszel 如果你：

- 有 1-10 部 VPS / server 想監控
- 想要 quick setup、唔想花時間㗳 config
- 資源有限（1-2GB RAM VPS）
- 需要 Docker container 層級監控
- 想要基本 alert 就夠
- 唔需要 custom metric

### 揀 Prometheus + Grafana 如果你：

- 管理 20+ 部機 / 多個 cluster
- 需要 highly customisable dashboard
- 需要監控特定 application metric（DB query time、API latency、request rate）
- 有專門嘅 infra team 維護
- 唔介意花時間 setup 同學習

### 揀 Uptime Kuma 如果你：

- *淨係*想知 website / service 夠唔夠期
- 已經有其他系統監控方案
- 想要一個簡單嘅 status page 俾人睇

---

## 其實可以一齊用

呢三個工具唔係 mutually exclusive。我自己嘅 setup 就係：

```
Beszel Hub ── 系統監控（CPU/RAM/Disk/Docker）
Uptime Kuma ── 服務可用性（website ping / port check）
```

Beszel 睇「部機有冇滿載」，Uptime Kuma 睇「個 service 仲行唔行到」。兩個加埋先係完整 picture。

Prometheus+Grafana 我就暫時唔使 — 冇咁多機、冇咁多 custom metric 要追。將來如果有需要，可以喺某啲 specific service 加 Prometheus exporter，而唔係成個 stack 換過去。

---

## Summary

```
                    Beszel     Prom+Graf     Uptime Kuma
───────────────   ───────   ───────────   ───────────
Setup 易度          ⭐⭐⭐⭐⭐     ⭐⭐          ⭐⭐⭐⭐⭐
系統監控             ✅✅✅       ✅✅✅         ❌
服務可用性            ❌          ❌            ✅✅✅
Docker detail       ✅✅       ✅(需cAdvisor)  ❌
Custom metric       ❌         ✅✅✅          ❌
Alert               ✅✅       ✅✅✅          ✅✅
資源消耗             ⭐⭐⭐⭐⭐     ⭐⭐          ⭐⭐⭐⭐
維護成本             ⭐⭐⭐⭐⭐     ⭐⭐          ⭐⭐⭐⭐⭐
```

冇絕對最好嘅工具，只有最適合你用量同時間嘅工具。
