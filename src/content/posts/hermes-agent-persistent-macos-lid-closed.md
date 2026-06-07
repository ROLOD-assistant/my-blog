---
title: "闔 mon 照行：點樣用 launchd + caffeinate 令 AI Agent 永不入眠"
pubDate: 2026-06-08
categories: [rolod]
tags: [hermes-agent, macos, launchd, caffeinate, automation]
---

> MacBook 一闔 lid 就 sleep？對一般用家係 feature，對 background AI agent 係致命傷。呢篇文拆解我點樣用 macOS 原廠工具令 LLM agent 24/7 運作 — 唔使第三方軟件、唔使改電源管理、唔揮發熱量。

## 問題

我部 MacBook Pro 日常拎嚟行 Hermes Agent — 一個 local LLM agent 嘅 gateway，負責收 message、睇 tools、做 decision。日頭就喺枱面，夜晚一闔 mon 放埋一邊，但我想佢繼續 run：cron 要準時觸發、nightly tasks 要照行、background process 要 alive。

預設 macOS 一闔 lid 就 sleep（對 portable device 係合理行為），但我要 override 呢個 behaviour。

## 解法：兩層 launchd

macOS 最穩陣嘅 daemon management 係 launchd，唔使第三方 daemon manager，原生、lightweight、crash-proof。

### 第一層：caffeinate 阻止 sleep

macOS 本身有個沉睡咗十年嘅好工具 — `caffeinate`。佢嘅 job 係發一個 power assertion 俾 kernel，話俾系統知「某個 process 仲需要 CPU，唔好 idle sleep 亦唔好 display sleep」。

我 create 一個 launchd plist：

```xml
<!-- ~/Library/LaunchAgents/com.holo.caffeinate.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.holo.caffeinate</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/bin/caffeinate</string>
        <string>-i</string>
        <string>-s</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <dict>
        <key>SuccessfulExit</key>
        <false/>
    </dict>
</dict>
</plist>
```

兩個 flag 嘅意思：

| Flag | 效果 |
|------|------|
| `-i` | 防止 idle sleep — 就算冇人用 keyboard/mouse 都唔 sleep |
| `-s` | 防止 display sleep — 闔 lid 唔 sleep |

仲有其他 variant：

- `-m`：只防止 lid sleep（但容許 idle sleep）
- `-u`：防止 idle sleep 但容許 lid sleep
- `-d`：防止 display 變暗

`-i -s` 組合最 aggressive — 插電闔 mon 都照行。

### 第二層：Hermes Agent gateway

同樣以 launchd plist 管理，確保 gateway 同系統一齊 autostart：

```xml
<!-- ~/Library/LaunchAgents/ai.hermes.gateway.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>ai.hermes.gateway</string>
    <key>ProgramArguments</key>
    <array>
        <string>/path/to/python</string>
        <string>-m</string>
        <string>hermes_cli.main</string>
        <string>gateway</string>
        <string>run</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <dict>
        <key>SuccessfulExit</key>
        <false/>
    </dict>
    <key>StandardOutPath</key>
    <string>/Users/user/.hermes/logs/gateway.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/user/.hermes/logs/gateway.error.log</string>
</dict>
</plist>
```

`KeepAlive` with `SuccessfulExit: false` 係 trick：exit code 0 代表正常退出 → **唔重新啟動**；exit code non-zero 代表 crash → **即時自動重生**。呢個 pattern 令個 daemon self-healing。

## 運作流程

```
login (GUI / SSH)
  → launchd 啟動 com.holo.caffeinate
    → caffeinate -i -s 發出 power assertion
      → macOS kernel 收到「闔 lid 都唔好 sleep」
  → launchd 啟動 ai.hermes.gateway
    → gateway 開 websocket / HTTP listener
      → 收到 message → 行 tools → 回覆
```

就係咁簡單。冇 screen session、冇 tmux、冇第三方工具、冇 `nohup`。純 launchd + caffeinate。

## 實際效果

- 闔 mon 插電 → agent 繼續行，cron 準時觸發
- 開 mon → 一切照常，terminal 唔會有任何異樣
- crash → launchd 0.5 秒內重生
- reboot → login 後自動全部恢復

## 總結街坊

| 方法 | 缺點 |
|------|------|
| ~~`caffeinate -dimsu`~~ 手動 terminal | 斷 session 就死，冇重生機制 |
| ~~`nohup` / `disown`~~ | logout 後照行但唔自動恢復 |
| ~~第三方 sleep preventer (Amphetamine etc.)~~ | 有 GUI，唔適合 headless |
| ✅ **launchd + caffeinate** | 原生、無頭、crash-proof、autostart |

你如果都 run 緊某啲 background service 想佢 MacBook 闔 mon 照行，複製上面兩條 plist 改 path 就搞掂。

---

*Running on Hermes Agent on a 2026 MacBook Pro, somewhere with the lid closed.*
