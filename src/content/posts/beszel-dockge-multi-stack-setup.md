---
title: "用 Dockge 管理 Beszel Monitoring — Stack 分拆實戰"
pubDate: 2026-05-27
categories: [rolod]
tags: [beszel, dockge, monitoring, docker, devops, self-hosted]
---

之前介紹咗 Beszel 做輕量 server monitoring，有人問到實際部署嘅問題：**用 Dockge 嘅話，Hub 同 Agent 應該放一個 stack 定分開？**

答案係：**分開，效果更好。**

---

## 點解 Dockge + Beszel 係好組合？

Dockge 係一個 Docker Compose 管理器，每個 stack 對應一個 docker-compose.yml，有獨立嘅 restart / update / log。

Beszel 嘅 architecture 係 Hub + 多個 Agent，先天就係多個獨立 service 嘅結構。Dockge 嘅 multi-stack 設計正好 match 呢個 pattern：

```
Dockge Dashboard
├── beszel-hub        ← Hub (port 8090)
├── beszel-agent-vps1 ← Agent for 主機自己
├── beszel-agent-vps2 ← Agent for 另一部 VPS
└── beszel-agent-vps3 ← Agent for 再另一部 VPS
```

每個 stack 獨立管理，互不影響。

---

## 實戰步驟

我會假設你已經有 Dockge 行緊。未有的話可以先睇返之前嘅 Dockge 安裝文。

### Step 1：喺 Dockge 建立 Hub stack

入 Dockge → **「Add Stack」** → 填：

| 欄位 | 值 |
|------|-----|
| **Name** | `beszel-hub` |
| **Docker Compose** | 貼下面內容 |

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

按 **Deploy**，等幾秒就見到 Hub 著咗。

去 `http://你部機IP:8090` 建立 admin 帳號。

### Step 2：喺 Hub 加系統、拎 KEY

Login Hub → 右上角 **「+ Add System」**：

| 欄位 | 填乜 |
|------|------|
| **Name** | `主機 (local)` |
| **Host / Port** | 因為 agent 同 hub 同一部機，可以揀 **local socket** |
| **Type** | Agent |

按 Add 之後，畫面會俾你一組 `KEY` 同指令。**Copy 個 KEY**。

如果係用嘅版本比較新，可以喺 Hub → Settings → Tokens 度 Generate 一個 universal token，之後每加一部機都用同一個 token，唔使逐次 copy。

### Step 3：建立 Agent stack（主機自己）

Dockge → **「Add Stack」** → 填：

| 欄位 | 值 |
|------|-----|
| **Name** | `beszel-agent-vps1` |
| **Docker Compose** | 貼下面 |

```yaml
services:
  beszel-agent:
    image: henrygd/beszel-agent
    container_name: beszel-agent
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./beszel_agent_data:/var/lib/beszel-agent
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      LISTEN: :45876
      KEY: "<你嘅 public key>"
      HUB_URL: "http://localhost:8090"
      TOKEN: "<你嘅 token>"
```

留意 `HUB_URL` 係 `http://localhost:8090` — 因為同一部機，直接 localhost 互通。

### Step 4：驗證

返去 Hub Dashboard → 正常情況下幾秒內就會見到 agent 連線，status 變綠色 Connected。

如果未見，可以喺 Dockge 睇 Agent stack 嘅 log：

```
Dockge → beszel-agent-vps1 → Logs
```

常見 error 係 KEY / TOKEN 填錯，或者 Hub URL 唔啱。

---

## 再加多部 VPS

新 VPS 嘅 Agent 步驟一模一樣，只係改幾個位：

### 新 VPS 上嘅 docker-compose.yml

```yaml
services:
  beszel-agent:
    image: henrygd/beszel-agent
    container_name: beszel-agent
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./beszel_agent_data:/var/lib/beszel-agent
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      LISTEN: :45876
      KEY: "<同一個 key 或者 universal token>"
      HUB_URL: "https://beszel.yourdomain.com"
      TOKEN: "<universal token>"
```

呢次 **唔係用 localhost** — Agent 唔同機，要經 internet 連返 Hub，所以 `HUB_URL` 要填 Hub 嘅公開網址。

喺 Dockge 上面就多一個 stack：

```
beszel-hub          ← Hub
beszel-agent-vps1   ← 主機自己
beszel-agent-vps2   ← 新加嘅 VPS
```

---

## 日常操作

### Update Hub

```
Dockge → beszel-hub → Update → Deploy
```

得一個 container，快過閃電。

### Update Agent

```
Dockge → beszel-agent-vps1 → Update → Deploy
```

其他 Agent 唔受影響。

### 睇某一部機嘅 log

```
Dockge → beszel-agent-vps2 → Logs
```

唔使喺 terminal 慢慢 grep。

### 熄某部機嘅 monitoring

```
Dockge → beszel-agent-vps3 → Stop
```

其他機繼續行。

---

## 一個 stack  vs  多個 stack 對比 recap

| | 一個 stack | 多個 stack（推薦） |
|---|---|---|
| Dockge UI 整潔度 | 一個 entry 入面有 N 個 container | 每個 stack 一個 entry，一目了然 |
| Restart Hub | 全部 agent 一齊 restart | 淨係 hub restart |
| 加新 VPS | 要入去 edit compose file | 直接開新 stack |
| 單一 agent crash | 成個 stack restart flag | 其他 stack 正常 |
| 每部機版本管理 | 全部一齊 update | 可以逐部 update |

Dockge 嘅強項就係 multi-stack 管理，Beszel 嘅 Hub + 多 Agent 結構天生就適合咁用。分開之後唔止乾淨，日常操作都快好多。
