---
title: "ESC/POS Printer 換紙後印唔到 — 原因同解法"
pubDate: 2026-05-30
categories: [rolod]
tags: [rolod, escpos, printer, thermal, pos, troubleshooting]
---

用 network 熱敏打印機 (TCP :9100) 成日遇到一個問題：店員換完紙之後，啲 order 就唔再印出嚟。

唔係 printer 壞，係 ESC/POS 嘅設計問題。

## 點解會咁？

```
1. Printer 冇紙 → 繼續收緊 TCP data，但印唔出
                   buffer 塞滿晒 pending data

2. 店員換紙 → Printer 仲係 error state（未 reset）
              佢收到 data，但唔郁

3. 新 order → Code send data 過去
              Printer 收到，但 error state 未清
              → data 塞埋一齊，永不打印
```

**即係：Printer 記住咗之前「冇紙」嘅 error，換完紙都冇 forget。**

## ESC/POS 嘅特性

ESC/POS printer 要收到 **Initialize command** 先會 reset error state。你換完紙，佢仲覺得自己 error，你 send 乜 data 都唔印。

## 解法

每次 print 前 send 兩個 byte：

```python
s.send(bytes([0x1B, 0x40]))  # ESC @ — Initialize Printer
```

呢句嘅意思係：「唔理你之前咩狀態，清 buffer，reset error state，準備好印新嘢。」

## 完整 Safe Print Function

```python
import socket, time

def safe_print(ip, port, esc_pos_data, max_retries=3):
    for attempt in range(max_retries):
        try:
            s = socket.socket()
            s.settimeout(5)
            s.connect((ip, port))

            # 1. Check paper status
            s.send(bytes([0x1D, 0x72, 0x01]))
            status = ord(s.recv(1))
            paper = (status >> 4) & 0x03

            if paper == 0b10:  # No paper
                s.close()
                print(f"Attempt {attempt+1}: No paper, retrying...")
                time.sleep(2)
                continue

            # 2. Initialize — 清 buffer + reset error state
            s.send(bytes([0x1B, 0x40]))  # ESC @
            time.sleep(0.1)

            # 3. Print
            s.send(esc_pos_data)
            s.close()
            return True

        except Exception as e:
            print(f"Print error: {e}")
            time.sleep(1)

    return False
```

## 簡化版

如果你唔需要 check paper，就咁每次 print 前加一句就得：

```python
# 每次 print 都 reset printer state
s.send(bytes([0x1B, 0x40]))  # ESC @
# 然後先 send data
s.send(esc_pos_data)
```

呢個 pattern 就算 buffer 塞死，`ESC @` 都會清空晒，然後重新開始，唔會漏單。

## 另一個常見問題 — Beeper 唔響

Xprinter 同好多中國牌子熱敏 printer，個 beeper 預設係 **關** 嘅。要用 Windows 調試工具 或者 DIP switch 開返：

- **Xprinter Q200/N200**：芯烨調試工具 → 基礎設置 → DIP 設置 → 蜂鳴器 set YES
- 淘寶買嘅時候可以叫賣家幫手開咗先

開咗之後可以用 ESC B command 控制：

```python
# Beeper pattern: ESC B n t
s.send(bytes([0x1B, 0x42, 0x03, 0x02]))  # 3次中響 — 新order提示
s.send(bytes([0x1B, 0x42, 0x01, 0x01]))  # 1次短響 — 輕微提示
s.send(bytes([0x1B, 0x42, 0x05, 0x03]))  # 5次長響 — 緊急催單
```

## 總結

| 問題 | 原因 | 解法 |
|------|------|------|
| 換紙後印唔到 | Printer error state 未 reset | Print 前加 `ESC @` |
| Beeper 唔響 | 預設關閉 | DIP switch 開 beeper |
| 中途冇紙 | Buffer 塞死 | `ESC @` 清 buffer + check paper status |
