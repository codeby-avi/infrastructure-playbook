
# 🔐 Secure Ubuntu Server – Fail2Ban Configuration & Intrusion Prevention

> **Enterprise-grade SOP for automated intrusion prevention, SSH protection, OTP/2FA abuse detection, and security alerting on Ubuntu servers (AWS EC2 & beyond).**

---

## 📌 Overview

This document defines a **production-ready Standard Operating Procedure (SOP)** for implementing **Fail2Ban** on **Ubuntu servers** to protect against:

- SSH brute-force attacks
- OTP / 2FA abuse
- PAM authentication failures
- Sudo misuse
- Repeat malicious offenders

It establishes a **host-based intrusion prevention system (HIPS)** that complements cloud firewalls, IAM controls, and SSH hardening.

---

## 📄 Document Control

| Item | Details |
|---|---|
| Document Name | Secure Ubuntu Server – Fail2Ban SOP |
| Environment | Ubuntu Server 20.04 / 22.04 / 24.04 |
| Scope | Intrusion prevention, SSH protection, abuse detection |
| Audience | Cloud Engineers, DevOps, SysAdmins, Security Teams |
| Owner | Platform / Infrastructure / Security Team |

---

## 🎯 Purpose

The purpose of this SOP is to establish a **secure, repeatable, and auditable process** to:

- Detect and block malicious authentication attempts
- Automatically ban abusive IP addresses
- Protect SSH, PAM, sudo, and OTP workflows
- Generate actionable **email alerts for security events**
- Reduce risk from brute-force and credential attacks
- Meet **enterprise security and compliance** requirements

---

## 🔐 Core Security Principles

- Fail2Ban acts as a **host-based intrusion prevention system**
- Bans are enforced at the **firewall level**
- Authentication failures are treated as **security signals**
- Admin IPs are **explicitly whitelisted**
- All bans are **logged, reversible, and auditable**

---

## ✅ Prerequisites

- Ubuntu Server with SSH hardening completed
- Key-based SSH authentication enabled
- `sudo` access available
- (Optional) PAM-based OTP / MFA enabled
- Outbound SMTP access for alerts
- Cloud firewall restricting SSH access

---

## 🧠 Fail2Ban Operational Model

Fail2Ban operates by:

- Monitoring authentication logs (systemd / log files)
- Matching failures using regex-based filters
- Applying bans using iptables / nftables
- Automatically expiring bans after a defined duration
- Detecting repeat offenders via the `recidive` jail

Fail2Ban **augments but does not replace** firewalls or IAM.

---

## 🪜 Step 1: Install Fail2Ban

```bash
sudo apt update
sudo apt install -y fail2ban
```

Verify installation:

```bash
fail2ban-client --version
```

---

## 🪜 Step 2: Install Email Dependencies

Fail2Ban uses an MTA for alerting.

```bash
sudo apt install -y sendmail mailutils
```

Validate email:

```bash
echo "Fail2Ban test" | mail -s "Fail2Ban Test" security@yourdomain.com
```

---

## 🪜 Step 3: Configuration Strategy

| File | Purpose |
|---|---|
| `/etc/fail2ban/jail.conf` | Default config (DO NOT EDIT) |
| `/etc/fail2ban/jail.local` | Enterprise overrides |

All production configuration **must be in `jail.local`**.

---

## 🪜 Step 4: Enterprise Fail2Ban Configuration

Edit configuration:

```bash
sudo nano /etc/fail2ban/jail.local
```

### ✅ Approved & Hardened Configuration

```ini
[DEFAULT]
ignoreip = 127.0.0.1/8 YOUR_ADMIN_PUBLIC_IP
bantime  = 12h
findtime = 10m
maxretry = 3
backend  = systemd

destemail = security@yourdomain.com
sender    = fail2ban@yourdomain.com
mta       = sendmail
action    = %(action_mwl)s

banaction = iptables-multiport
logtarget = /var/log/fail2ban.log
loglevel  = INFO

[sshd]
enabled  = true
port     = 22
maxretry = 3
bantime  = 24h

[recidive]
enabled  = true
logpath  = /var/log/fail2ban.log
findtime = 1d
bantime  = 7d
maxretry = 5

[pam-generic]
enabled  = true
logpath  = /var/log/auth.log
maxretry = 3
bantime  = 12h

[sudo]
enabled  = true
logpath  = /var/log/auth.log
maxretry = 3
bantime  = 6h
```

---

## 🪜 Step 5: Apply and Activate

```bash
sudo systemctl stop fail2ban
sudo rm -f /var/run/fail2ban/fail2ban.sock
sudo systemctl daemon-reexec
sudo systemctl start fail2ban
sudo systemctl enable fail2ban
```

---

## 🪜 Step 6: Validation & Health Checks

```bash
sudo systemctl status fail2ban --no-pager
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

Expected jails:

- sshd
- recidive
- pam-generic
- sudo

---

## 📧 Email Alert Validation

Trigger test ban:

```bash
sudo fail2ban-client set sshd banip 8.8.8.8
```

Remove test ban:

```bash
sudo fail2ban-client set sshd unbanip 8.8.8.8
```

---

## 🛠️ Operational Commands

| Task | Command |
|---|---|
| View jails | `fail2ban-client status` |
| View SSH bans | `fail2ban-client status sshd` |
| Unban IP | `fail2ban-client set sshd unbanip <IP>` |
| View logs | `tail -f /var/log/fail2ban.log` |

---

## 🧯 Emergency Rollback (Break-Glass)

```bash
sudo systemctl stop fail2ban
```

All bans are immediately removed.

---

## 🏢 Enterprise Best Practices

- Whitelist only static admin IPs or VPN ranges
- Use `recidive` jail in production
- Combine with SSH hardening and MFA
- Forward logs to SIEM (ELK, Splunk, Sentinel)
- Review bans periodically

---

## 📜 Compliance Alignment

- CIS Ubuntu Linux Benchmark
- ISO/IEC 27001 – Monitoring & Access Control
- SOC 2 – Security & Availability
- NIST SP 800-53 (AC, AU, IR)

---

## ✅ Final Note

Fail2Ban should be treated as a **mandatory host-level security control**, not optional hardening.

If detection is not logged, bans are not auditable, or rollback is not immediate — **it is not production-ready**.
