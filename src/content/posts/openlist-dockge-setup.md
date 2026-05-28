---
title: "OpenList 自架 — Dockge 部署 + NPM 綁 Domain"
pubDate: 2026-05-28
categories: [rolod]
tags: [openlist, alist, file-management, docker, dockge, self-hosted]
---

OpenList 係 AList 嘅 fork，一個支援多種雲端儲存嘅檔案列表程式。你可以將阿里雲盤、OneDrive、Google Drive、百度網盤、115 等掛載埋一齊，統一用一個 Web UI 瀏覽同管理。

GitHub 上 22.7k stars，AGPL-3.0 授權。

---

## 用 Dockge 部署

### Stack：`openlist`

入 Dockge → **「Add Stack」** → 填 Name `openlist` → 貼：

```yaml
services:
  openlist:
    image: 'openlistteam/openlist:latest'
    container_name: openlist
    user: '0:0'
    volumes:
      - './openlist_data:/opt/openlist/data'
    ports:
      - '5244:5244'
    environment:
      - UMASK=022
    restart: unless-stopped
```

按 **Deploy**。

---

## 初始設定

開 browser → `http://你ServerIP:5244`

第一次啓動會自動生成 admin 密碼，睇 container log：

```bash
docker logs openlist | grep "initial password"
```

你會見到類似：

```
Successfully created the admin user and the initial password is: xLUKOZTm
```

Login：

| 欄位 | 值 |
|------|-----|
| **Username** | `admin` |
| **Password** | 上面 log 嗰個 |

Login 後記得去 **管理 → 使用者** 改密碼。

---

## 權限注意

OpenList v4.1.0+ 改用 user 1001:1001 執行，唔再用 `PUID`/`PGID` 環境變數。所以 docker-compose 要加 `user: '0:0'` 或者直接俾返你 host user 嘅 UID:GID。

If you skip `user: '0:0'`，會出權限 error：

```
Error: Current user does not have write and/or execute permissions for the ./data directory
```

---

## UFW Firewall 注意

如果你部機開咗 ufw，要 allow port 5244：

```bash
sudo ufw allow 5244
```

Docker publish port 唔一定會 bypass ufw，尤其是新版 Ubuntu + Docker 組合。

---

## 經 NPM 綁 Domain

如果你用 Nginx Proxy Manager，加返 Proxy Host：

| 欄位 | 值 |
|------|-----|
| **Domain Names** | `openlist.你既domain.com` |
| **Scheme** | `http` |
| **Forward Hostname / IP** | `你Server嘅LAN IP` |
| **Forward Port** | `5244` |
| **SSL** | Request Let's Encrypt + Force SSL |

之後就可以 `https://openlist.你既domain.com` 入。

---

## 支援嘅儲存

OpenList 支援超過 30 種儲存後端，包括：

- 本地儲存
- 阿里雲盤 / 123盤 / 天翼雲盤 / 百度網盤 / 115
- OneDrive / Google Drive / Dropbox
- S3 / WebDAV / FTP / SFTP
- PikPak / Mega.nz / Terabox
- GitHub / Cloudreve / Azure Blob

每種 storage 嘅設定係 OpenList Web UI 入面直接填，唔使改 config file。

---

## 同 Beszel 嘅關係

OpenList 係 file management，Beszel 係 monitoring，兩個各自做唔同嘢。但佢哋可以行喺同一個 server，共用 NPM 做 reverse proxy：

```
openlist.homelab.deven.tw  → NPM → 192.168.x.x:5244
beszel.homelab.deven.tw    → NPM → 192.168.x.x:8090
uptime.homelab.deven.tw    → NPM → 192.168.x.x:3001
```

全部一個 domain structure，同一部 server，唔使記 port number。
