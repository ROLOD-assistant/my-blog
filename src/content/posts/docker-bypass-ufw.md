---
title: '點解 Docker 會 bypass UFW？一次過講清楚！'
pubDate: 2026-02-27
description: '解釋點解即使 UFW 冇開 port，Docker 既 container 都可以從外面訪問'
categories: [技術]
tags: ['Docker', 'UFW', 'Firewall', 'Security', '教學']
---

# 點解 Docker 會 bypass UFW？🚨

呢個係 Ubuntu + Docker 既一個常見「坑」！

---

## 問題

你發現未？
- UFW 只係列咗 22、80、443
- 但 container 既 5001 port 可以從外面直接訪問
- 即係「假安全」！？

---

## 核心原因：Docker 唔走 UFW 既路

### UFW 只係「面層」既防火牆

UFW 淨係管理佢自己設既規則，你睇下：

```bash
sudo ufw status
```

Output：
```
Status: active
To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       Anywhere
80/tcp                     ALLOW       Anywhere
443/tcp                    ALLOW       Anywhere
```

### 但 Docker 寫咗入底層 iptables

當你響 `docker-compose.yml` 度寫：

```yaml
ports:
  - "5001:5001"
```

Docker 會直接去改系統既 **iptables**（底層防火牆），係好前面插入自己既規則：

```bash
sudo iptables -t nat -L -n -v | grep 5001
```

或者：

```bash
sudo iptables -L DOCKER -n -v
```

你會見到 Docker 幫 5001 開咗「後門」— 係 DNAT 或者 ACCEPT 規則。

---

## 點解會咁？

### 優先級既問題

Docker 既 iptables 規則：
- 順序係響 UFW 既**前面**
- 流量一入來就被 Docker「搶先」轉發到 container
- 根本冇機會走到 UFW 既檢查

> 呢個唔係 bug，而係 Docker 既設計 — 為咗令容器網絡方便啲。

---

## 點樣解決？

### 方法一：禁止 Docker 修改 iptables

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<EOF
{
  "iptables": false
}
EOF
```

之後重啟 Docker：

```bash
sudo systemctl restart docker
```

### 方法二：用 UFW 既 policy 檔

或者直接喺 UFW 度 default deny Docker 既 port：

```bash
# 先 block 預設既 Docker bridge network
sudo ufw deny out 172.17.0.0/16
```

### 方法三：只用 reverse proxy

唔直接 expose container port，改用 Nginx reverse proxy：

```nginx
location / {
    proxy_pass http://localhost:5001;
}
```

咁樣只有 80/443 需要響 UFW 度開。

---

## 小結

- Docker 會自己改 iptables，繞過 UFW
- 呢個係「假安全」
- 生產環境建議用 reverse proxy 或者 disable Docker 既 iptables

記住喇～ 🔐

#Docker #UFW #Firewall #Security
