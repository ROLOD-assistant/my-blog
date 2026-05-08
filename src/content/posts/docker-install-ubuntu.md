---
title: 'Docker Engine 安裝教學（Ubuntu Convenience Script）'
pubDate: 2026-02-27
description: '教你點樣用 convenience script 快速安裝 Docker Engine'
categories: [技術]
tags: ['Docker', 'Ubuntu', '教學', 'DevOps']
---

# Docker Engine 安裝教學（Ubuntu）🐳

呢篇教你點樣響 Ubuntu 上用官方既 convenience script 快速安裝 Docker Engine～

> ⚠️ **注意：** 呢個方法只推薦用於測試同開發環境，生產環境建議用 apt repository 方式安裝。

---

## 前置要求

### 支援既 Ubuntu 版本

- Ubuntu Questing 25.10
- Ubuntu Noble 24.04 (LTS)
- Ubuntu Jammy 22.04 (LTS)

### 支援既架構

- x86_64 (amd64)
- armhf
- arm64
- s390x
- ppc64le

---

## 預先卸載舊版本

如果之前有裝過 Docker，要先卸載：

```bash
sudo apt remove docker.io docker-compose docker-compose-v2 docker-doc podman-docker containerd runc
```

---

## 安裝步驟

### 1. Download 同執行 convenience script

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

### 2. 啟動 Docker service

```bash
sudo service docker start
```

### 3. 驗證安裝

```bash
docker --version
docker run hello-world
```

---

## （可選）以非 root 用戶使用 Docker

如果你唔想每次都用 `sudo`：

```bash
sudo usermod -aG docker $USER
```

> 之後要 logout 再 login 先會生效～

---

## 注意事項

- **Firewall：** 如果你用 ufw 或 firewalld，要留意 Docker 會 bypass 你既 firewall rules
- **升級：** 用 script 安裝既話，升級要重新執行 script 或者手動處理
- **生產環境：** 建議用官方 apt repository 既方式安裝，方便日後升級

---

## 卸載 Docker

如果唔想要喇：

```bash
sudo apt remove docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

同時可以刪除相關既 data：

```bash
sudo rm -rf /var/lib/docker
```

---

快啲試下啦！🐳

#Docker #Ubuntu #DevOps
