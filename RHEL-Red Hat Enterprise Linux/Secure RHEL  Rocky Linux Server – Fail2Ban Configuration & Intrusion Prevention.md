# 🔐 Secure RHEL Server – SSH + 2FA (MFA) + Fail2Ban with Advanced Filters

> **Enterprise-grade SOP for hardened SSH access with selective Multi-Factor Authentication (2FA/MFA) and advanced Fail2Ban intrusion prevention (OTP-only, sudo-only, geo-ban) on RHEL servers.**

---

## 📌 Overview

This document defines a **production-ready Standard Operating Procedure (SOP)** for securing SSH access on **RHEL servers** by combining:

- 🔐 Hardened SSH configuration
- 🔑 Key-based authentication only
- 🔒 Selective Two-Factor Authentication (OTP-based MFA)
- 🚫 Automated intrusion prevention using Fail2Ban
- 🧠 **Advanced Fail2Ban filters** for OTP abuse, sudo misuse, and geographic blocking

It is intentionally written to be:

* 🔁 **Repeatable** — deterministic steps with predictable results  
* 🔍 **Auditable** — explicit controls, logs, and validation points  
* 🛡️ **Security-first** — layered defense, least privilege  
* 🏢 **Enterprise & compliance aligned** — suitable for regulated environments  

---

## 📄 Document Control

| Item | Details |
|---|---|
| Document Name | Secure RHEL Server – SSH + 2FA + Fail2Ban SOP |
| Environment | RHEL 8 / 9 |
| Scope | SSH access, MFA enforcement, intrusion prevention |
| Audience | Cloud Engineers, DevOps Engineers, SysAdmins, Security Teams |
| Owner | Platform / Infrastructure / Security Team |

---

## 🎯 Purpose

The purpose of this SOP is to establish a **secure, repeatable, and auditable process** to:

- Enforce **key-based SSH authentication only**
- Apply **2FA (OTP)** for selected human users
- Detect and block **OTP brute-force attacks**
- Detect and block **sudo misuse**
- Block traffic from **unauthorized geographic regions**
- Preserve automation and service account access
- Meet **enterprise security and compliance** requirements

---

## 🔐 Core Security Principles

- SSH passwords are **never permitted**
- MFA is enforced **only for human users**
- Service and automation accounts are excluded from MFA
- Authentication failures are treated as **security events**
- Intrusion prevention must be **automatic and reversible**
- Break-glass access must always exist

---

## ✅ Prerequisites

- RHEL 8 / 9 server
- Key-based SSH authentication enabled
- Password authentication disabled
- `sudo` access available
- Fail2Ban installed
- Console / cloud recovery access
- Time synchronization enabled (chrony)

---

## 🧠 Security Architecture

```
SSH Login Attempt
   │
   ├─ SSH Key Authentication
   │
   ├─ (If MFA-enabled user)
   │     └─ OTP Verification via PAM
   │
   ├─ Access Granted
   │
   └─ Fail2Ban monitors:
         - SSH failures
         - OTP failures (PAM)
         - sudo misuse
         - Geographic violations
```

---

## 🪜 Step 1: Harden SSH Configuration

```bash
sudo nano /etc/ssh/sshd_config
```

```conf
Include /etc/ssh/sshd_config.d/*.conf

Port 22
AddressFamily any
ListenAddress 0.0.0.0
ListenAddress ::

HostKey /etc/ssh/ssh_host_ed25519_key
HostKey /etc/ssh/ssh_host_rsa_key

PermitRootLogin no
StrictModes yes
UsePAM yes

PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys

PasswordAuthentication no
KbdInteractiveAuthentication no
ChallengeResponseAuthentication no

AllowUsers ubuntu mess-central-admin

MaxAuthTries 3
LoginGraceTime 30
MaxSessions 5

ClientAliveInterval 300
ClientAliveCountMax 2
TCPKeepAlive no

X11Forwarding no
AllowAgentForwarding no
AllowTcpForwarding no
PermitTunnel no
GatewayPorts no
PermitTTY yes

SyslogFacility AUTH
LogLevel VERBOSE

AcceptEnv LANG LC_*
PermitUserEnvironment no

Subsystem sftp /usr/lib/openssh/sftp-server

Match User mess-central-admin
    PubkeyAuthentication yes
    PasswordAuthentication no


```
---

## 🪜 Step 2: Install 2FA (OTP) Components

```bash
sudo dnf install -y epel-release
sudo dnf install -y google-authenticator qrencode
```

---

## 🪜 Step 3: Enroll Selected Users for MFA

```bash
google-authenticator
chmod 400 ~/.google_authenticator
```

---

## 🪜 Step 4: PAM Configuration (Selective MFA)

```bash
sudo nano /etc/pam.d/sshd
```

Add at top:

```ini
auth required pam_google_authenticator.so nullok
```

---

## 🪜 Step 5: Enforce MFA via SSH Group

```bash
sudo groupadd ssh-mfa
sudo usermod -aG ssh-mfa adminuser
```

```conf
Match Group ssh-mfa
    ChallengeResponseAuthentication yes
    AuthenticationMethods publickey,keyboard-interactive
```

---

## 🪜 Step 6: Install Fail2Ban

```bash
sudo dnf install -y fail2ban
sudo systemctl enable fail2ban
```

---

## 🪜 Step 7: Advanced Fail2Ban Configuration

```bash
sudo nano /etc/fail2ban/jail.local
```

### ✅ Core Settings

```ini
[DEFAULT]
ignoreip = 127.0.0.1/8 <ADMIN_IP>
bantime  = 12h
findtime = 10m
maxretry = 3
backend  = systemd
```

---

### 🔒 SSH Brute-Force Protection

```ini
[sshd]
enabled = true
port = ssh
bantime = 24h
```

---

### 🔐 OTP-Only Abuse Detection (PAM)

```ini
[pam-generic]
enabled = true
logpath = /var/log/secure
maxretry = 3
bantime = 12h
```

---

### 🚨 Sudo Abuse Detection

```ini
[sudo]
enabled = true
logpath = /var/log/secure
maxretry = 3
bantime = 6h
```

---

### 🌍 Geo-IP Blocking (Country-Level)

> ⚠️ Requires firewall-cmd and ipset

```ini
[geo-ban]
enabled = true
filter = sshd
action = firewallcmd-ipset[name=geo-ban, port=ssh, protocol=tcp]
logpath = /var/log/secure
maxretry = 1
bantime = 7d
```

Use firewall rules to allow **approved countries only**.

---

## 🪜 Step 8: Apply and Validate

```bash
sudo sshd -t
sudo systemctl restart sshd
sudo systemctl restart fail2ban
```

Check status:

```bash
sudo fail2ban-client status
```

---

## 🧯 Emergency Access (Break-Glass)

```conf
Match User breakglass
    AuthenticationMethods publickey
```

Controls:
- IP allowlisting
- Separate SSH key
- Mandatory audit review

---

## 🛠️ Monitoring & Logs

| Component | Log |
|---|---|
| SSH / PAM / sudo | `/var/log/secure` |
| Fail2Ban | `/var/log/fail2ban.log` |

---

## 🏢 Enterprise Best Practices

- Separate jails per attack surface
- Geo-ban only after VPN / IP allowlist
- Forward logs to SIEM
- Review bans weekly
- Test break-glass quarterly

---

## 📜 Compliance Alignment

- CIS RHEL Benchmark
- ISO/IEC 27001
- SOC 2
- NIST SP 800-53

---

## ✅ Final Note

Fail2Ban must evolve beyond SSH brute-force protection.

If OTP abuse, sudo misuse, or geographic anomalies are not detected — **your intrusion prevention is incomplete**.

This SOP defines a **full enterprise-grade RHEL SSH security baseline**.
