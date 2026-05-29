---
title: "VPS 基礎 Security Hardening — SSH、防火牆、自動更新"
pubDate: 2026-05-29
categories: [rolod]
tags: [vps, security, ssh, firewall, linux, self-hosted, devops]
---

新開一部 VPS，幾分鐘內就會有 SSH login attempts。Hackers 24/7 用自動化 script 掃緊所有 public IP，搵 vulnerable 嘅 server。

以下係最基本嘅 VPS 安全設定，跟住做就可以將一部全新 VPS 鎖好。

---

## 1. 第一次 SSH

```bash
ssh root@你既VPS_IP
```

第一次連會見到 **fingerprint warning** — 打 `yes` 存檔。之後再見到就唔正常，可能 server 被 compromise。

**做完第一步，第一件事：更新系統。**

```bash
apt update && apt upgrade -y
```

- Kernel 有 upgrade 的話要 reboot
- Check 需唔需要 reboot：`ls /var/run/reboot-required`
- Reboot 後再做一次 `apt upgrade` 確保冇 kept back packages

成日 keep 住最新 packages 係最基本嘅 security — hackers 通常 exploit 已知嘅舊漏洞。

---

## 2. 改 Root Password

```bash
passwd
```

VPS provider 俾嘅 default password 一定要改。`passwd` 唔會顯示你打嘅字。

---

## 3. 建立新 User（Least Privilege）

唔好用 root 做日常操作。建立一個普通 user，要用 super user 權限時先用 `sudo`：

```bash
adduser 你既username
usermod -aG sudo 你既username
groups 你既username    # 確認有 sudo group
```

- `id` 睇 UID：0 = root
- 之後 SSH 用新 user：`ssh 你既username@你既VPS_IP`
- 需要用 root 權限時：`sudo apt update`

---

## 4. SSH Key Authentication — 停用 Password Login

**喺你部 local machine generate SSH key（如果未有）：**

```bash
ssh-keygen -t ed25519 -C "你既email"
```

**睇 public key：**
```bash
cat ~/.ssh/id_ed25519.pub
```

**加落 VPS：**
```bash
mkdir -p ~/.ssh
nano ~/.ssh/authorized_keys
# paste public key → Ctrl+X → Y → Enter
```

之後 SSH 就唔使打 password。

---

## 5. Disable Password Login

```bash
sudo nano /etc/ssh/sshd_config
```

搵到：
```
PasswordAuthentication yes
→ 改做 no
```

Check 埋 override config：
```bash
ls /etc/ssh/sshd_config.d/
sudo nano /etc/ssh/sshd_config.d/50-cloud-init.conf  # 如有
```

改完 restart：
```bash
sudo service ssh restart
```

**測試：** `ssh root@你既VPS_IP` → 應該出 `Permission denied (publickey)`

---

## 6. Disable Root Login Over SSH

```bash
sudo nano /etc/ssh/sshd_config
```

搵到：
```
PermitRootLogin yes
→ 改做 no
```

```bash
sudo service ssh restart
```

之後 root 完全唔可以 SSH login，一定要用普通 user 入再用 `sudo`。

---

## 7. Firewall — Port 控制

**方法 A：VPS Provider Dashboard（推薦）**
- 搵 Firewall / Port / Network section
- Close 晒唔需要嘅 ports
- Default 開咗：22（SSH）、80（HTTP）、443（HTTPS）
- 如果未用到 web server，close 80 同 443

**方法 B：UFW（Application Firewall）**
```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

---

## 8. 改 SSH Port（Optional）

```bash
sudo nano /etc/ssh/sshd_config
# 改 Port 22 → 2222 或你揀嘅 port
sudo service ssh restart
```

可以避開 automated attacks（default 掃 port 22），但 savvy hacker 仍然 scan 到你用緊咩 port。

---

## 9. IP Whitelisting（Optional）

如果你有 static IP，可以限制 SSH 只俾你個 IP 入，firewall rule 用你嘅 IP 代替 `0.0.0.0/0`。

---

## 10. Unattended Upgrades — 自動安全性更新

```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure unattended-upgrades
# → 選 Yes
```

**Customize config：**
```bash
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```

可以開嘅 options：
- 唔只 security updates，連 regular updates 都自動
- 自動 reboot + 揀時間
- Send email notification

**Check service：**
```bash
sudo systemctl status unattended-upgrades
# 見到 green dot = active
```

---

## 做齊 Checklist

1. ✅ System fully upgraded
2. ✅ Root password changed
3. ✅ New sudo user created
4. ✅ SSH key configured
5. ✅ Password login disabled
6. ✅ Root login disabled
7. ✅ Firewall locked down
8. ✅ Auto security updates enabled

之後就可以放心開 ports 裝 services、上 reverse proxy + SSL。

---

參考資料：VPS Setup and Security Hardening by CJ@Sentense
https://www.youtube.com/watch?v=Q1Y_g0wMwww
