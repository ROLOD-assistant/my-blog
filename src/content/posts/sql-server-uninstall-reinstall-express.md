---
title: 卸載 SQL Server Developer 版 → 安裝 Express 版（完整流程）
pubDate: 2026-02-24
description: 教你點樣完整卸載SQL Server Developer版並安裝Express版
categories: [技術]
tags: [SQL Server, 教學, 數據庫]
---

# 卸載 SQL Server Developer 版 → 安裝 Express 版（完整流程）🔄

## 卸載 Developer 版

### 1. 前置準備
- **備份資料庫**（.bak 或 .mDF/.ldf）
- 注意：Express 單庫上限 50 GB
- 打開 `services.msc` → 停止 SQL Server、Agent、Browser
- 以管理員身份執行之後既步驟

### 2. 移除 SQL Server
- 去設定 → 已安裝應用程式 → 搜「sql」
- 點 **Microsoft SQL Server 2025 (64-bit)** → 解除安裝
- 選擇 **移除** → 選實例 → 全選功能 → 移除
- 等完成

### 3. 清理剩餘項目
拎走以下組件（如果存在）：
- SQL Server Setup
- SQL Server Browser
- SQL Server VSS Writer
- ODBC/OLE DB 驅動
- SSMS（可選）

### 4. 手動刪除資料夾（備份後）
- `C:\Program Files\Microsoft SQL Server\`
- `C:\Program Files (x86)\Microsoft SQL Server\`
- `C:\ProgramData\Microsoft SQL Server\`
- 實例資料夾（如 `MSSQL16.*`）

### 5. 重開機
若失敗：查日誌 `C:\Program Files\Microsoft SQL Server\170\Setup Bootstrap\Log\`

---

## 安裝 Express 版

### 1. 下載
去 [Microsoft SQL Server downloads](https://www.microsoft.com/zh-cn/sql-server/sql-server-downloads) → Express → **立即下載**

### 2. 安裝
- 選擇：**基本**（推薦）或 **自訂**
- 實例名建議：**SQLEXPRESS**
- 驗證選：Windows 或 Mixed（設 sa 密碼）

### 3. 連線
- `.\SQLEXPRESS` 或 `localhost\SQLEXPRESS`

### 4. SSMS
若移除咗既話，去同一頁面重新下載 **SQL Server Management Studio 22**

---

## 小提示 💡

安裝後若連不上，確認：
- 防火牆允許 1433 埠
- 重開 SQL Browser 服務

---

#SQLServer #數據庫 #教學
