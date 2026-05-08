---
title: SQL Server - 啟用 TCP/IP 協議並設定靜態端口 1433
pubDate: 2026-02-24
description: 教你點樣係SQL Server度啟用TCP/IP協議同設定靜態端口1433
categories: [技術]
tags: [SQL Server, 教學, 數據庫, 網絡]
---

# SQL Server - 啟用 TCP/IP 協議並設定靜態端口 1433 🌐

## 重點要點

- 預設情況下，**TCP/IP** 協議係 SQL Server 度通常**係停用既**（尤其係 Developer/Express 版本）
- 即使啟用咗，都可能用緊 **dynamic ports** → 客戶端無SQL Browser就連接唔到
- 對於遠程連接（例如由另一部PC連到 `192.168.60.155`），**TCP/IP必須啟用** + 建議用靜態端口 **1433** + 防火牆要開
- 所有改動都需要**重新啟動** SQL Server service

---

## 逐步教學：啟用 TCP/IP 同設定端口 1433

### 1. 打開 SQL Server Configuration Manager

- Windows 搜尋：**SQL Server Configuration Manager**
- 或者運行：`C:\Windows\SysWOW64\SQLServerManagerXX.msc`（XX = version，例如 2019 就係 15，2022 就係 16）

### 2. 啟用 TCP/IP 協議

- 展開 **SQL Server Network Configuration**
- 展開 **Protocols for [你既Instance]**
  - Default instance → **Protocols for MSSQLSERVER**
  - Named instance → **Protocols for SQLEXPRESS**（或你既instance名）
- 右邊搵 **TCP/IP**
- 右鍵 **TCP/IP** → **Enable**
  （Icon會由紅色X變成綠色箭嘴）

### 3. 設定靜態端口 1433（推薦）

- 右鍵 **TCP/IP** → **Properties**
- 去 **IP Addresses** tab
- 去最底既 **IPAll** 部分
- **TCP Dynamic Ports** → 清空任何數值（刪除「0」或數字）→ 留**空白**
- **TCP Port** → 輸入 **1433**
- 點 **OK**

### 4. 重新啟動 SQL Server Service

- 左邊：展開 **SQL Server Services**
- 右鍵 **SQL Server (MSSQLSERVER)**（或你既instance名）→ **Restart**
-（可選但推薦）都 restart **SQL Server Browser** service

### 5. 驗證監聽端口 1433

- 係 server 度打開 **Command Prompt**（管理員）
- 運行：
  ```cmd
  netstat -ano | find "1433"
  ```

- 期望輸出（類似）：
```
TCP    0.0.0.0:1433           0.0.0.0:0              LISTENING       1234
```

- → 表示 SQL 係所有 IP (0.0.0.0) 監聽 port 1433

### 6. 防火牆：允許 TCP 1433 入站

- 打開 **Windows Defender Firewall** → **Advanced Settings** → **Inbound Rules**
- **New Rule**:
    - Rule Type: **Port**
    - Protocol: **TCP**
    - Specific local ports: **1433**
    - Action: **Allow the connection**
    - Profile: **Domain + Private + public**（全部剔！）
    - Name: "SQL Server TCP 1433"

### 7. 測試遠程連接

- 由 client PC 打開 **SSMS**
- Server name: `192.168.60.155`（default instance）或者 `192.168.60.155,1433`（指定端口）
- Authentication: **Windows** 或 **SQL Server** (sa + password)

---

## 快速參考表

| 步驟 | 位置 | 做咩 | 點解 |
|------|------|------|------|
| 啟用 TCP/IP | SQL Server Network Config → Protocols... | 右鍵 TCP/IP → Enable | 允許遠程 TCP 連接 |
| 設定靜態端口 | TCP/IP Properties → IP Addresses → IPAll | TCP Dynamic Ports = 空白, TCP Port = 1433 | 可靠連接，唔洗 Browser |
| 重啟 Service | SQL Server Services | Restart SQL Server (instance) | 應用所有改動 |
| 防火牆規則 | Windows Firewall → Inbound → New Rule | TCP 1433 → Allow → All Profiles | 防止「Connect timed out」error |
| 用 netstat 驗證 | Command Prompt | `netstat -ano \| find "1433"` | 確認 SQL 係監聽緊 |

---

## 常見錯誤同解決方法

| 錯誤信息 / 症狀 | 可能原因 | 解決方法 |
|------|------|------|
| Connect timed out (port 1433) | TCP/IP 停用或端口唔係 1433 | 啟用 TCP/IP + set 1433 + restart |
| No such host / wrong port | 錯誤既 IP 或 instance 名 | 檢查 server name / 用 IP,1433 |
| Login failed after connect | Auth mode 唔係 Mixed 或 sa 停用 | 啟用 Mixed Mode + enable sa + password |
| 本地得但遠程唔得 | 防火牆 block 咗 | 加 TCP 1433 入站規則 |
| Named instance → "Instance not found" | SQL Browser 未運行或 UDP 1434 被 block | 啟動 SQL Browser + 開 UDP 1434 |

---

## 小提示 💡

- 用 **靜態端口 1433** 簡單啲（唔洗dynamic ports）
- 安全性：唔好直接將 SQL 暴露係互聯網度 → 用 VPN 或者 Azure SQL 代替
- 官方文檔：[Enable TCP/IP](https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/enable-or-disable-a-server-network-protocol)

搞掂！做完呢啲步驟，遠程連接到 192.168.60.155 既 port 1433 應該完全正常 🚀

---

#SQLServer #數據庫 #教學 #網絡
