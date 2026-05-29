---
title: "WG-Easy 自架 — 5 分鐘搞掂 WireGuard VPN"
pubDate: 2026-05-29
categories: [rolod]
tags: [wireguard, vpn, wg-easy, dockge, docker, self-hosted, homelab]
---

之前講 monitoring、dashboard，今次講 **VPN** — WireGuard + wg-easy，一個 container 搞掂晒，仲有 web UI 管理 client。

WireGuard 比 OpenVPN 快 10 倍、kernel level、codebase 得 4000 行。wg-easy 再加個 web UI 俾你 generate config / QR code，手機 scan 就連到。

---

## Stack：`wg-easy`

入 Dockge → **Add Stack** → Name `wg-easy` → 貼：

```yaml
services:
  wg-easy:
    image: ghcr.io/wg-easy/wg-easy:latest
    container_name: wg-easy
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
    ports:
      - 51820:51820/udp
      - 51821:51821/tcp
    environment:
      - TZ=Asia/Hong_Kong
      - WG_HOST=你既VPS_IP或Domain
      - PASSWORD=你自己set密碼
    volumes:
      - ./wg-easy-data:/etc/wireguard
```

---

## 設定重點

**`WG_HOST`** — 你 VPS 嘅 public IP 或者 domain。Client 會用呢個地址連入嚟。

```
WG_HOST=你既VPS_IP或Domain  # 用 public IP
# 或
WG_HOST=vpn.你既domain.com  # 用 domain（建議 + DDNS）
```

**`PASSWORD`** — Web UI 嘅登入密碼。就咁打明文就得。

**Ports：**
- `51820/udp` — WireGuard VPN 連接埠（一定要 UDP）
- `51821/tcp` — Web 管理界面

---

## 第一次入

Deploy 之後開 browser → `http://你ServerIP:51821`

Login 用你 set 嘅 password。

之後你會見到一個好簡單嘅界面：
- **Create** — 加新 client
- 每個 client 可以有 Download config 或者 **QR code**（用手機 WG app scan）
- 可以改名、睇 traffic stats、enable/disable

---

## 用手機連

1. 手機裝 **WireGuard** app（iOS / Android 都有）
2. wg-easy web UI → 揀 client → **QR Code**
3. 手機 WG app → 右上角 + → **Scan from QR code**
4. 畀個名 → Save → Toggle ON

咁就 connect 咗。Check 下 IP 係咪已經轉咗。

---

## 實用心得

- **防火牆**：記得開 VPS firewall 放行 `51820/udp`，唔係 client 連唔入
- **DNS**：預設用 `1.1.1.1`，想轉可以加：
  ```yaml
  - WG_DEFAULT_DNS=8.8.8.8
  ```
- **全流量 vs 內網**：預設係全流量（0.0.0.0/0），亦即係全部 traffic 經 VPN。如果只係想 access 內網 resources：
  ```yaml
  - WG_ALLOWED_IPS=192.168.0.0/16, 10.0.0.0/8
  ```
- **Backup**：`./wg-easy-data` 入面有晒私鑰同 config，一定要 backup

---

## 同場加映：如果唔用 Dockge

```bash
docker run -d \
  --name wg-easy \
  --cap-add NET_ADMIN \
  --cap-add SYS_MODULE \
  --sysctl net.ipv4.ip_forward=1 \
  --sysctl net.ipv4.conf.all.src_valid_mark=1 \
  -p 51820:51820/udp \
  -p 51821:51821/tcp \
  -e TZ=Asia/Hong_Kong \
  -e WG_HOST=你既VPS_IP \
  -e PASSWORD=你自己set密碼 \
  -v /opt/stacks/wg-easy/wg-easy-data:/etc/wireguard \
  --restart unless-stopped \
  ghcr.io/wg-easy/wg-easy:latest
```
