<div align="center">

# 🛡️ Wazuh SIEM Setup on Ubuntu (VirtualBox)

### A hands-on guide to deploying a single-node Wazuh stack — with real errors, real fixes.

![Wazuh](https://img.shields.io/badge/Wazuh-4.7.x-1A73E8?style=for-the-badge&logo=wazuh&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-Server-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![VirtualBox](https://img.shields.io/badge/VirtualBox-VM-183A61?style=for-the-badge&logo=virtualbox&logoColor=white)
![Status](https://img.shields.io/badge/Status-Working-2EA44F?style=for-the-badge)
![License](https://img.shields.io/badge/License-MIT-yellow?style=for-the-badge)

</div>

---

A step-by-step guide for installing the Wazuh SIEM stack (🧠 Manager, 🗂️ Indexer, 📊 Dashboard) on a single Ubuntu VM running inside VirtualBox — including the real problems hit during setup and exactly how they were fixed.

## 📋 Environment

| 🔧 Spec         | 💻 Value                       |
|-----------------|---------------------------------|
| Virtualization  | VirtualBox                     |
| OS              | Ubuntu                         |
| Disk            | 60 GB                          |
| Wazuh Version   | 4.7.x                          |
| Components      | Manager + Indexer + Dashboard (all-in-one) |

## 📑 Table of Contents

- [🚀 Installation Steps](#-installation-steps)
- [🐛 Problems Faced & Fixes](#-problems-faced--fixes)
- [🌐 Accessing the Dashboard](#-accessing-the-dashboard)
- [🔑 Login](#-login)
- [💡 Lessons Learned](#-lessons-learned)

---

## 🚀 Installation Steps

### 1️⃣ System Update

```bash
sudo apt update && sudo apt upgrade -y
```

### 2️⃣ Install Required Packages

```bash
sudo apt install curl
sudo apt install curl unzip apt-transport-https gnupg -y
```

### 3️⃣ Add Swap Space ⚠️ Important

> Wazuh's Indexer is memory-hungry — adding swap up front prevents crashes during install.

```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

### 4️⃣ Install Wazuh

```bash
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
chmod +x wazuh-install.sh
sudo bash wazuh-install.sh -a
```

This installs:
- ✅ Wazuh Manager
- ✅ Wazuh Indexer
- ✅ Wazuh Dashboard

⏱️ Takes roughly **15–20 minutes**.

### 5️⃣ Save Login Info

At the end of the install, Wazuh prints the dashboard credentials. **Save these immediately** — they're also stored in `wazuh-install-files.tar` (in the directory you ran the script from) if you need them later.

```
User: admin
Password: <generated password>
```

> 🚫 **Don't commit your real password to GitHub.** Keep credentials out of version control.

---

## 🐛 Problems Faced & Fixes

### 🔴 Problem 1: Indexer fails to start

**Error:**
```
Failed to start wazuh-indexer. Start operation timed out
```

**🔍 Root cause:** Default JVM heap settings were too aggressive for available RAM.

**✅ Fix** — reduce JVM memory allocation:

```bash
sudo nano /etc/wazuh-indexer/jvm.options
```

| Before | After |
|--------|-------|
| `-Xms256m` | `-Xms1g` |
| `-Xmx2456m` | `-Xmx1g` |

Restart and verify:
```bash
sudo systemctl restart wazuh-indexer
sudo systemctl status wazuh-indexer
```

✅ Expected output: `active (running)`

---

### 🔴 Problem 2: Browser can't reach the dashboard

**Error:**
```
localhost refused to connect
ERR_CONNECTION_ABORTED
```

**🔍 Root cause:** The dashboard service was running, but not bound to an accessible network interface.

**✅ Fix:**

```bash
sudo nano /etc/wazuh-dashboard/opensearch_dashboards.yml
```

Add/update:
```yaml
server.host: "0.0.0.0"
server.port: 5601
```

**Also fix the SSL setting** (was blocking plain HTTP access):

| Before | After |
|--------|-------|
| `server.ssl.enabled: true` | `server.ssl.enabled: false` |

Restart and verify:
```bash
sudo systemctl restart wazuh-dashboard
sudo ss -tulnp | grep 5601
```

✅ Expected output: `0.0.0.0:5601`

> ⚠️ Disabling SSL is fine for local/lab testing, but **re-enable it for production**.

---

### 🔴 Problem 3: VirtualBox network access issue

**Symptom:** `localhost` on the host machine couldn't reach the VM's dashboard.

**✅ Fix:** Set up port forwarding in VirtualBox.

```
VirtualBox → Settings → Network → NAT → Port Forwarding
Host Port: 5601
Guest Port: 5601
```

---

## 🌐 Accessing the Dashboard

**🥇 Method 1 — via localhost (after port forwarding):**
```
http://localhost:5601
```

**🥈 Method 2 — via the VM's direct IP:**
```bash
ip a
```
```
http://10.0.2.15:5601
```

---

## 🔑 Login

```
Username: admin
Password: <your generated password>
```

---

## 💡 Lessons Learned

- 🧠 Always provision swap **before** installing Wazuh — the indexer fails silently otherwise on low-RAM VMs.
- ⚙️ JVM heap size (`jvm.options`) should be tuned to match available system memory, not left at defaults.
- 🌐 For VM-based setups, dashboard binding (`server.host`) and VirtualBox port forwarding are two **separate** things — missing either one breaks browser access.

---

<div align="center">

Made with ☕ and a lot of debugging.

</div>
