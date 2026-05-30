---
title: "FileBrowser 自架 — Dockge 一鍵部署"
pubDate: 2026-05-30
categories: [rolod]
tags: [rolod, filebrowser, dockge, docker, self-hosted, homelab]
---

自架 homelab，成日要 share file 俾人 download。APK、config backup、firmware update — 次次都 scp 或者開 Nginx autoindex 好麻煩。

FileBrowser 就係一個輕量 web UI，俾你直接 upload / download 任何 file，開個 public link 俾人 download 就搞掂。

## 比較

| | FileBrowser | OpenList / AList | Nexus3 |
|---|---|---|---|
| **用途** | 一個 folder 嘅 web UI | 聚合雲端硬碟 | 完整 artifact repository |
| **Image Size** | **~20MB** | ~100MB | ~1.5GB |
| **RAM** | ~15MB | ~50MB | ~1GB+ |
| **Docker image?** | ❌ | ❌ | ✅ |
| **APK / raw file** | ✅ direct upload | ⚠️ 要經 backend | ✅ |
| **Setup** | 一行 docker | 要逐個 backend 配 | 複雜 |
| **適合同你的 homelab** | **✅ 最啱** | overkill | overkill |

FileBrowser 嘅優點：**簡單到不得了的簡單。** 一個 image 20MB，開機即用，upload download 就係咁簡單。

## Stack YAML (Dockge)

```yaml
services:
  filebrowser:
    image: filebrowser/filebrowser:latest
    container_name: filebrowser
    ports:
      - "8480:80"
    volumes:
      - ./data:/srv
      - ./config:/config
    restart: unless-stopped
```

**部署步驟：**

1. Dockge → Add Stack → 改名 `filebrowser`
2. Paste 上面 YAML → Deploy
3. 開 browser → `http://你ServerIP:8480`
4. Default login: `admin` / `admin`
5. Upload 你啲 file 🎉

參考官方文檔：[filebrowser.org](https://filebrowser.org/)

> ⚠️ 第一次 login 記得去 Settings 改 password。Default admin/admin 係公開嘅。

## 進階設定

FileBrowser 支援環境變數自訂：

```yaml
services:
  filebrowser:
    image: filebrowser/filebrowser:latest
    container_name: filebrowser
    ports:
      - "8480:80"
    volumes:
      - ./data:/srv
      - ./config:/config
    environment:
      - FB_BASEURL=/files
      - FB_PORT=80
    restart: unless-stopped
```

常用 env vars：

| Variable | 用途 | Default |
|----------|------|---------|
| `FB_BASEURL` | 自訂 URL prefix | `/` |
| `FB_PORT` | Container port | `80` |
| `FB_ADMIN` | Admin username | `admin` |
| `FB_NOAUTH` | Disable login | false |

## NPM Reverse Proxy

如果經 Nginx Proxy Manager 公開俾人 download：

| 欄位 | 值 |
|------|-----|
| **Domain Names** | `files.你既domain.com` |
| **Scheme** | `http` |
| **Forward Hostname / IP** | `你Server嘅LAN IP` |
| **Forward Port** | `8480` |
| **SSL** | Request Let's Encrypt + Force SSL |

## Tips

1. **Public share link**：Right click file → Share → Generate link。俾條 link 人哋 direct download，唔使 login。
2. **Folder structure**：建議開 folder 分好類 — `apks/`、`configs/`、`firmware/`
3. **Auto cleanup**：冇內置 expiry，耐唔耐入去 delete 舊 file 就得

## 同場加映 — Docker CLI One-liner

```bash
docker run -d \
  --name filebrowser \
  -p 8480:80 \
  -v ./data:/srv \
  -v ./config:/config \
  filebrowser/filebrowser:latest
```

同上面 docker-compose 一樣，只係一行搞掂。
