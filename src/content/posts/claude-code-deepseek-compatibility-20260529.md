---
title: "Claude Code + DeepSeek API 相容性筆記"
pubDate: 2026-05-29
categories: [rolod]
tags: [claude-code, deepseek, troubleshooting]
---

2026-05-28 Claude Code 更新版本 2.1.154+ 後，與 DeepSeek API 出現相容性問題，記錄解法如下。

## 問題

新版將 system prompt 放入 messages 陣列中，DeepSeek API 不支援此格式：

```
API Error: 400 Failed to deserialize the JSON body into the target type:
messages[1].role: unknown variant `system`, expected `user` or `assistant`
```

DeepSeek 的 API 只接受 `user` 同 `assistant` 兩種 role，唔接受 `system` role。

## 解法

降級到最後正常版本 **2.1.153**，並關閉自動更新。

## 步驟

### 切換版本

Claude Code 使用自己的版本管理，本機已有多個版本，直接切換 symlink：

```bash
rm ~/.local/bin/claude && \
  ln -s ~/.local/share/claude/versions/2.1.153 ~/.local/bin/claude
```

若未來需要特定版本：

```bash
# 查已安裝版本
ls ~/.local/share/claude/versions/

# 查 npm 可用版本
npm view @anthropic-ai/claude-code versions --json | grep "2\\.1\\."
```

### settings.json 關鍵配置

`~/.claude/settings.json`：

```json
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "sk-********************************",
    "ANTHROPIC_BASE_URL": "https://api.deepseek.com/anthropic",
    "ANTHROPIC_MODEL": "deepseek-v4-pro[1m]",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "Deepseek-v4-flash",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL_NAME": "Deepseek-v4-flash",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "deepseek-v4-pro[1m]",
    "ANTHROPIC_DEFAULT_OPUS_MODEL_NAME": "deepseek-v4-pro",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "deepseek-v4-pro[1m]",
    "ANTHROPIC_DEFAULT_SONNET_MODEL_NAME": "deepseek-v4-pro",
    "CLAUDE_CODE_DISABLE_AUTOUPDATER": "1"
  }
}
```

### 版本時間線

| 版本 | 日期 | 狀態 |
|------|------|------|
| 2.1.152 | 5/26 | 正常 |
| 2.1.153 | 5/27 | 最後正常版 |
| 2.1.154 | 5/28 | 開始有問題 |
| 2.1.156 | 5/28 | 有問題 |

## 注意

- Model 名稱 `[1m]` 無空格，`m` 小寫
- `CLAUDE_CODE_DISABLE_AUTOUPDATER=1` 防止自動更新
- 若有本機 proxy，`ANTHROPIC_BASE_URL` 改為 `http://127.0.0.1:8765`
