---
title: "Docker Port 仲 hold 住？Ghost docker-proxy 處理"
pubDate: 2026-05-28
categories: [rolod]
tags: [docker, troubleshooting, port, proxy, dockge, devops]
---

Dockge delete 咗 stack，container 唔見咗，但 port 仲係 `address already in use` 起唔到新 container。

原因係 **docker-proxy ghost process** — container 刪咗，但 Docker 嘅 port mapping subprocess 仲留低，hold 住個 port。

---

## 點 Check

```bash
ss -tlnp | grep 5244
```

如果你見到類似：

```
LISTEN 0  4096  0.0.0.0:5244  0.0.0.0:*  users:(("docker-proxy",pid=3917,fd=8))
```

但 `docker ps` 又冇 container 行緊，就係 **ghost docker-proxy**。

```
docker ps | grep 5244
→ 冇
docker ps -a | grep openlist
→ 冇
ss -tlnp | grep 5244
→ docker-proxy pid=3917  ❌ 仲喺度
```

---

## 點解會咁

正常流程：

```
Docker delete container
    ↓
Remove container ✅
Remove docker-proxy ✅
Release port ✅
```

但某啲情況下：

- Container crash loop 得太快（不停 restart exit restart）
- 手動 `docker rm` 但冇等 Docker daemon cleanup
- Docker daemon 嘅 cleanup 機制唔夠快
- Dockge 或者其他管理工具 delete stack 時漏咗

就會出現：

```
Dockge delete stack
    ↓
Remove container ✅
Remove network ✅
Remove docker-proxy ❌ ← 留低咗
    ↓
Port 仲 hold 住
    ↓
新 container 起唔到
```

---

## 解決

直接用 kill 释放 port：

```bash
kill <PID>
```

例如：

```bash
kill 3917 3923
```

再 check：

```bash
ss -tlnp | grep 5244
→ 冇
```

Port free 咗，新 container 可以正常 Deploy。

---

## 如果 kill 唔到（Permission denied）

用 force kill：

```bash
kill -9 3917 3923
```

或者直接用 root：

```bash
sudo kill 3917
```

---

## 另一個方法：Restart Docker daemon

如果 kill 咗都仲喺度，直接 restart Docker：

```bash
systemctl restart docker
```

呢個會清晒所有 ghost proxy，但缺點係所有 running container 都會 restart 一次。

---

## 預防

- 刪 stack 前先 `docker compose down` 而唔係直接 delete
- 如果係 Dockge，先 Stop 再 Remove，而唔係直接 Remove
- 避免手動 `docker rm` 正在 crash loop 嘅 container

但有時候真係避無可避，知道點 kill ghost docker-proxy 就夠。
