---
title: "ESC/POS Network Printer — Half-Print Paper Out Handling"
pubDate: 2026-05-30
categories: [rolod]
tags: [rolod, escpos, printer, pos, architecture, flow]
---

上一篇講咗 [換紙後印唔到嘅問題同 ESC @ 解法](/posts/escpos-printer-paper-recovery/)，今次講埋 **print 到一半冇紙嗰張單點處理**。

## 問題

TCP :9100 嘅特性：

```
你 send data ──TCP──► Printer buffer
                        │
                        ├── 有紙 → 印出嚟 ✅
                        └── 冇紙 → buffer 照收
                                   但唔印 ❌
                                   socket 唔斷，冇 error
```

**TCP 層面唔會話俾你知失敗。** Printer 收晒你啲 bytes，只係印唔出。

## 點樣 detect？

靠 **print 前後 check paper sensor** 對比：

```python
def print_with_detect(ip, port, esc_pos_data):
    s = socket.socket()
    s.settimeout(5)
    s.connect((ip, port))

    # 1. Check paper BEFORE
    s.send(bytes([0x1D, 0x72, 0x01]))
    before = ord(s.recv(1))
    if before & 0x60 == 0x60:
        s.close()
        return {"success": False, "reason": "no_paper_before"}

    # 2. Initialize + Print
    s.send(bytes([0x1B, 0x40]))  # ESC @
    s.send(esc_pos_data)

    # 3. Check paper AFTER
    s.send(bytes([0x1D, 0x72, 0x01]))
    after = ord(s.recv(1))
    s.close()

    # 4. Compare
    if after & 0x60 == 0x60:
        return {"success": False, "reason": "paper_empty_during_print"}

    return {"success": True}
```

| Check 前 | Check 後 | 結果 |
|---------|---------|------|
| ✅ OK | ✅ OK | ✅ 正常印完 |
| ❌ EMPTY | ❌ EMPTY | ❌ Skip，唔印 |
| ✅ OK | ❌ EMPTY | ❌ **Half-print，要 retry** |

第三個 case：Check 前 OK、Check 後 EMPTY → 中間一定出咗事。

## 成個 Flow

因為 tablet 係 competing consumer 嘅 architecture，呢個 flow 係 thread-safe 嘅：

```
┌──────────────┐
│  Server       │  print_jobs table (FOR UPDATE SKIP LOCKED)
│  Queue        │
└──────┬───────┘
       │
       │ Tablet poll GET /print-queue
       ▼
┌──────────────┐
│  Tablet A     │  ── 攞到 job #241
│  (做緊 print) │
│               │
│  1. TCP :9100 │
│  2. ESC @     │
│  3. Check 前  │
│  4. Send data │
│  5. Check 後  │  ← 冇紙！
│  6. POST ack  │
│     { success: false, reason: "paper_empty" }
└──────┬───────┘
       │
       ▼ Server
       mark failed → job 留返喺 queue
       │
       ▼
┌──────────────┐
│  Tablet B     │  ── 下一輪 poll 到同一個 job #241
│               │     佢唔知之前 fail 過
│               │     照 print 一次
│               │     今次有紙 → ✅ Done
└──────────────┘
```

**關鍵規則：**
- Server 唔 delete job，只 mark `failed` / `pending`
- Tablet 唔 delete job，只 POST ack result
- Half-print 嘅單由下一個 polling tablet retry
- SKIP LOCKED 保證同一時間得一部機做 TCP print

## Job Lifecycle

```
status: pending     → 未印，等緊 tablet 攞
status: assigned    → 已被 tablet 拎咗去印
status: failed      → print 失敗 (冇紙/network error)
status: done        → 成功印完
```

```sql
UPDATE print_jobs SET status = 'pending'
WHERE status = 'failed'
  AND created_at > NOW() - INTERVAL '1 hour'
```

**Failed 嘅 job 會自動放返入 queue，等下次 polling 再印。**

## 總結

| 問題 | 解法 |
|------|------|
| TCP 唔知 print 失敗 | Print 前後 check paper sensor |
| Half-print 張單冇咗 | Retry — 留返喺 queue 等下次 |
| 點 guarantee 唔 double print | DB SKIP LOCKED |
| 店員想手動 reprint | KDS 加 Reprint button → POST /reprint |
