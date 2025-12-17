# 🔐 Secure Ubuntu Server – SSH Hardening with 2FA (MFA) & Fail2Ban

> **Enterprise-grade SOP for hardened SSH access with selective Multi-Factor Authentication (2FA/MFA) and automated intrusion prevention using Fail2Ban on Ubuntu servers (AWS EC2 & beyond).**

---

## 📌 Overview

This document defines a **production-ready Standard Operating Procedure (SOP)** for securing SSH access on **Ubuntu servers** by combining:

- 🔐 Hardened SSH configuration
- 🔑 Key-based authentication only
- 🔒 Selective Two-Factor Authentication (OTP-based MFA)
- 🚫 Automated intrusion prevention using Fail2Ban

It is intentionally written to be:

* 🔁 **Repeatable** — deterministic steps with predictable results  
* 🔍 **Auditable** — explicit controls, logs, and validation points  
* 🛡️ **Security-first** — layered defense, least privilege  
* 🏢 **Enterprise & compliance aligned** — suitable for regulated environments  

This SOP establishes a **defense-in-depth SSH security baseline** suitable for long-term production operations.

---

## 📄 Document Control

| Item | Details |
|---|---|
| Document Name | Secure Ubuntu Server – SSH + 2FA + Fail2Ban SOP |
| Environment | Ubuntu Server 20.04 / 22.04 / 24.04 |
| Scope | SSH access, MFA enforcement, intrusion prevention |
| Audience | Cloud Engineers, DevOps Engineers, SysAdmins, Security Teams |
| Owner | Platform / Infrastructure / Security Team |

---

## 🎯 Purpose

The purpose of this SOP is to establish a **secure, repeatable, and auditable process** to:

- Enforce **key-based SSH authentication only**
- Apply **2FA (OTP)** for selected human users
- Prevent SSH brute-force and OTP abuse
- Automatically block malicious IP addresses
- Preserve automation and service account access
- Prevent administrative lockout
- Meet **enterprise security and compliance** requirements

---

## 🔐 Core Security Principles

- SSH passwords are **never permitted**
- SSH keys are treated as **privileged credentials**
- MFA is enforced **only for human users**
- Service and automation accounts are excluded from MFA
- All authentication failures are treated as **security signals**
- At least one **break-glass access path** must always exist

---

## ✅ Prerequisites

- Ubuntu Server with SSH hardening capability
- Key-based SSH authentication enabled
- Password authentication disabled
- `sudo` access available
- Fail2Ban installed
- Console or cloud recovery access
- Time synchronization enabled (systemd-timesyncd / chrony)

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
         - PAM / OTP failures
         - Repeat offenders
```

---

## 🪜 Step 1: Harden SSH Configuration

Edit SSH configuration:

```bash
sudo nano /etc/ssh/sshd_config
```

### ✅ Approved Baseline

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
sudo apt update
sudo apt install -y libpam-google-authenticator qrencode
```

---

## 🪜 Step 3: Enroll Selected Users for MFA

Run **as each target user**:

```bash
google-authenticator
```

Secure OTP file:

```bash
chmod 400 ~/.google_authenticator
```

---

## 🪜 Step 4: PAM Configuration (Selective MFA)

Edit PAM SSH configuration:

```bash
sudo nano /etc/pam.d/sshd
```

Add **at the top**:

```ini
auth required pam_google_authenticator.so nullok
```

---

## 🪜 Step 5: Enforce MFA via SSH Match Rules

Create MFA group:

```bash
sudo groupadd ssh-mfa
sudo usermod -aG ssh-mfa adminuser
```

SSH configuration:

```conf
Match Group ssh-mfa
    ChallengeResponseAuthentication yes
    AuthenticationMethods publickey,keyboard-interactive
```

---

## 🪜 Step 6: Install Fail2Ban

```bash
sudo apt install -y fail2ban
```

---

## 🪜 Step 7: Configure Fail2Ban

Edit configuration:

```bash
sudo nano /etc/fail2ban/jail.local
```

### ✅ Approved Configuration

```ini
[DEFAULT]
ignoreip = 127.0.0.1/8 <ADMIN_IP>
bantime  = 12h
findtime = 10m
maxretry = 3
backend  = systemd

[sshd]
enabled  = true
port     = ssh
maxretry = 3
bantime  = 24h

[pam-generic]
enabled  = true
logpath  = /var/log/auth.log
maxretry = 3
bantime  = 12h

[recidive]
enabled  = true
logpath  = /var/log/fail2ban.log
findtime = 1d
bantime  = 7d
maxretry = 5
```

---

## 🪜 Step 8: Apply and Activate

```bash
sudo sshd -t
sudo systemctl restart ssh

sudo systemctl enable fail2ban
sudo systemctl restart fail2ban
```

⚠️ Keep at least one SSH session open during validation.

---

## 🪜 Step 9: Validation

```bash
ssh adminuser@server_ip
```

Expected flow:
1. SSH key accepted
2. OTP prompt (for MFA users)
3. Login successful

Fail2Ban status:

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

| Component | Log File |
|---|---|
| SSH / PAM | `/var/log/auth.log` |
| Fail2Ban | `/var/log/fail2ban.log` |

---

## 🏢 Enterprise Best Practices

- Enforce MFA only for human admins
- Never enable MFA for service accounts
- Rotate SSH keys every 90–180 days
- Rotate OTP secrets annually
- Forward logs to SIEM
- Test break-glass access quarterly

---

## 📜 Compliance Alignment

- CIS Ubuntu Linux Benchmark
- ISO/IEC 27001 – Access Control
- SOC 2 – Logical Access Controls
- NIST SP 800-53 (IA, AC, AU)

---

## ✅ Final Note

SSH security must be **layered, selective, and auditable**.

If SSH allows passwords, MFA is globally enforced without planning, or Fail2Ban bans are not logged — **it is not production-ready**.

This SOP defines a **complete Ubuntu enterprise SSH security baseline**.
