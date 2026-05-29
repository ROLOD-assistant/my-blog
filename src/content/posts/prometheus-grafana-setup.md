---
title: "Prometheus + Grafana 自架 — Long-term Metrics 與可視化 Dashboard"
pubDate: 2026-05-29
categories: [rolod]
tags: [prometheus, grafana, monitoring, docker, dockge, self-hosted, homelab]
---

之前講咗 Beszel（輕量即時 monitoring）、Uptime Kuma（Uptime 監控），今次講 **Long-term metrics** — Prometheus 負責儲數據，Grafana 負責畫靚 dashboard。

Beszel 睇到而家部機嘅 CPU / RAM / Disk，但過咗就冇。Prometheus 儲起晒所有 data points，你可以睇返 **上星期、上個月、上年** 同一時間嘅資源走勢。

---

## Prometheus vs Grafana 分工

| | Prometheus | Grafana |
|---|---|---|
| **角色** | TSDB — 收 metrics、儲數據 | Visualization — 畫 dashboard |
| **點拎數據** | Pull mode（定期 scrape 各 exporter） | 查 Prometheus 嘅數據 |
| **儲存** | 本地 TSDB（可 set retention） | 唔儲數據，只係 query |
| **UI** | 簡單 query browser | 靚 dashboard + alert |
| **Port** | 9090 | 3000 |

簡單講：Prometheus 係個 database，Grafana 係個 frontend 俾你睇數據。

---

## Stack：`prometheus`

入 Dockge → **Add Stack** → Name `prometheus` → 貼：

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./data:/prometheus
    ports:
      - "9090:9090"
```

同一個 stack 加 Grafana：

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./data:/prometheus
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    volumes:
      - ./grafana-data:/var/lib/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=你自己set密碼
```

---

## Prometheus Config

喺同一個 stack 目錄開 `prometheus.yml`：

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
```

呢個 config 只係 scrape Prometheus 自己。要監控其他服務，要加對應嘅 **exporter**（例如 node_exporter 俾 server stats）。

---

## Add Node Exporter（監控 Server）

喺同一部機開多個 container：

```yaml
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
    ports:
      - "9100:9100"
```

然後喺 `prometheus.yml` 加：

```yaml
  - job_name: "node"
    static_configs:
      - targets: ["你ServerIP:9100"]
```

改完 `prometheus.yml` 要 restart Prometheus container。

---

## 第一次入 Grafana

1. 開 browser → `http://你ServerIP:3000`
2. Login：`admin` / 你 set 嘅 password（第一次會叫你改 password）
3. **Connections → Add data source** → 揀 **Prometheus**
4. URL 填：`http://prometheus:9090`（Docker network 內部）
5. **Save & Test** → green

之後 **Dashboards → Import** → 搜 `node exporter full`（ID 1860）→ Load → 揀 Prometheus data source → Import。

你就會見到一個完整嘅 server monitoring dashboard：CPU、RAM、Disk、Network、Uptime，全部有歷史走勢。

---

## 經 NPM 綁 Domain

Grafana：

| 欄位 | 值 |
|------|-----|
| **Domain Names** | `grafana.你既domain.com` |
| **Scheme** | `http` |
| **Forward Hostname / IP** | `你Server嘅LAN IP` |
| **Forward Port** | `3000` |
| **SSL** | Request Let's Encrypt + Force SSL |

Prometheus 通常唔放 public，如果你想放都得，同上面一樣但 port 9090。

---

## 實用心得

- **Retention**：Prometheus default 保留 15 日數據。想改耐啲，喺 `prometheus.yml` 加：
  ```yaml
  global:
    scrape_interval: 15s
    evaluation_interval: 15s
  storage:
    tsdb:
      retention.time: 90d
  ```

- **Alert**：Grafana 內置 alerting，可以 set threshold 例如 CPU > 90% 就 send notification（Telegram / Email / Slack）

- **Exporters**：想 monitor 其他嘢，例如：
  - `blackbox_exporter` — HTTP / TCP / ICMP 外部監控
  - `cadvisor` — Docker container 層級嘅 metrics
  - `postgres_exporter` — PostgreSQL 狀態

- **Backup**：Prometheus data 喺 `./data`，Grafana data 喺 `./grafana-data`，定期 backup 呢兩個 folder

---

## 同場加映：如果唔用 Dockge

```bash
# Prometheus
docker run -d \
  --name prometheus \
  -p 9090:9090 \
  -v /opt/stacks/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml \
  -v /opt/stacks/prometheus/data:/prometheus \
  --restart unless-stopped \
  prom/prometheus:latest

# Grafana
docker run -d \
  --name grafana \
  -p 3000:3000 \
  -v /opt/stacks/grafana/grafana-data:/var/lib/grafana \
  -e GF_SECURITY_ADMIN_PASSWORD=你既密碼 \
  --restart unless-stopped \
  grafana/grafana:latest
```
