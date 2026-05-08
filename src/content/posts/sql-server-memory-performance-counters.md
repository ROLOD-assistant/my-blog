---
title: SQL Server 效能監控 - 記憶體相關 Performance Counters
pubDate: 2026-02-24
description: 教你用SQL Server既Performance Counters檢查記憶體使用情況
categories: [技術]
tags: [SQL Server, 教學, 數據庫, 效能]
---

# SQL Server 效能監控 - 記憶體相關 Performance Counters 💾

今日教你幾個有用既 SQL Server 效能監控語句，等你可以瞭解 SQL Server 既記憶體狀況！

---

## 1. Page Life Expectancy (PLE)

```sql
SELECT * FROM sys.dm_os_performance_counters
WHERE counter_name = N'Page life expectancy';
```

### 呢個係咩？

**Page Life Expectancy (PLE)** 係指一個 data page 係記憶體緩衝池入面既「預期壽命」—— 單位係**秒**。

### 點解咁緊要？

- 如果 PLE 低過 300 秒（5分鐘），表示 SQL Server 記憶體不足
- 數值越高越好，表示數據可以係 RAM 度keep耐啲，唔洗成日去硬碟攞
- **典型值**：正常應該有幾百秒甚至幾千秒

### 幾時用？

- 當你發覺query慢咗既時候
- 懷疑記憶體唔夠既時候
- 檢查 SQL Server 係唔係有記憶體 pressure

---

## 2. 緩衝池命中率 (Buffer Cache Hit Ratio)

```sql
SELECT * FROM sys.dm_os_performance_counters
WHERE counter_name LIKE N'%Buffer cache hit ratio%';
```

### 呢個係咩？

**Buffer Cache Hit Ratio** 顯示 SQL Server 係緩衝池入面搵數據既命中率。

### 點解咁緊要？

- **越高越好**：理想值應該 > 95%
- 如果低過 90%，表示 SQL Server 經常要去硬碟攞數據，而不是從記憶體
- 呢個直接影響query既速度

### 幾時用？

- 效能優化既時候
- 懷疑緩衝池大細唔夠既時候
- 同 PLE 一齊睇可以全面瞭解記憶體健康

---

## 點樣解讀？

| 指標 | 正常範圍 | 點算？ |
|------|------|------|
| Page Life Expectancy | > 300秒 | 如果低咗，可能要加 RAM 或優化 query |
| Buffer Cache Hit Ratio | > 95% | 如果低咗，可能要增加 max server memory |

---

## 小提示 💡

- 記得睇埋 `SQL Server: Buffer Manager` 既其他 counters
- 如果發現長期低數值，可能要考慮：
  - 加 RAM
  - 調整 `max server memory`
  - 優化經常用到既 queries
  - 檢查有無memory leak

---

#SQLServer #數據庫 #效能 #教學
