---
title: SQL Server 安裝失敗？教你解決磁碟磁區大小問題
pubDate: 2026-02-24
description: 解決SQL Server係新式NVMe儲存上無法安裝既問題
categories: [技術]
tags: [SQL Server, 教學, 數據庫, 疑難解決]
---

# SQL Server 安裝失敗？教你解決磁碟磁區大小問題 💾

## 問題係咩？

如果你既電腦使用**新式儲存平臺**（例如 NVMe SSD），佢既**磁區大小**可能大於 4 KB。

呢個時候安裝 SQL Server 通常會fail，出現奇怪既error但又唔知咩事。

## 點解會咁？

SQL Server 設計既時候假定磁碟既物理磁區大小係 4 KB。

但係新既 NVMe  SSD 通常用 4 KB 以上既磁區（例如 8 KB、16 KB 甚至更大）。

當 SQL Server 發現磁區大小唔啱既時候，就會拒絕安裝——因為佢擔心數據會寫唔正確。

## 點樣解決？

你需要加入一個 Windows 登錄機碼，等 SQL Server 可以將磁區大小**模擬為 4 KB**。

### 方法一：PowerShell（推薦）

以**系統管理員**身份執行 PowerShell，然後執行：

```powershell
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\stornvme\Parameters\Device" -Name "ForcedPhysicalSectorSizeInBytes" -PropertyType MultiString -Force -Value "* 4095"
```

### 驗證

執行以下命令確認機碼已經成功加入：

```powershell
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\stornvme\Parameters\Device" -Name "ForcedPhysicalSectorSizeInBytes"
```

### 方法二：註冊表編輯器

1. 按 `Win + R`，輸入 `regedit`
2. 去到：`HKLM\SYSTEM\CurrentControlSet\Services\stornvme\Parameters\Device`
3. 新增字串值：`ForcedPhysicalSectorSizeInBytes`
4. 數值設為：`* 4095`

## 為咗咩要咁做？

| 設定 | 意思 |
|------|------|
| `*` | 適用於所有磁碟 |
| `4095` | 強制模擬為 4 KB |

呢個設定會話俾 Windows 聽：「無論實際磁區幾大，都模擬做 4 KB」，SQL Server 就可以正常運作。

## 小提示 💡

- 呢個**唔需要**加追蹤旗標 1800
- 如果你用緊舊式 SATA SSD 或者 HDD，應該唔會遇到呢個問題
- 安裝完 SQL Server 之後，如果唔放心可以刪除呢個機碼（但留低都唔會有害）

---

## 參考 🔗

- [Microsoft 官方文檔：SQL Server 中磁碟磁區大小錯誤的解決步驟](https://learn.microsoft.com/zh-tw/troubleshoot/sql/database-engine/database-file-operations/troubleshoot-os-4kb-disk-sector-size?tabs=PowerShell#resolution-steps-for-disk-sector-size-errors-in-sql-server)

#SQLServer #數據庫 #教學 #疑難解決
