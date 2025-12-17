# 🔐 Secure RHEL / Rocky Linux Server – Fail2Ban Configuration & Intrusion Prevention

> **Enterprise-grade SOP for automated intrusion prevention, SSH protection, OTP/2FA abuse detection, and security alerting on RHEL & Rocky Linux servers (AWS, Azure, On‑Prem).**

---

## 📌 Overview

This document defines a **production-ready Standard Operating Procedure (SOP)** for implementing **Fail2Ban** on **RHEL / Rocky Linux servers** to protect against:

- SSH brute-force attacks
- OTP / 2FA abuse
- PAM authentication failures
- Sudo misuse
- Repeat malicious offenders

It establishes a **host-based intrusion prevention system (HIPS)** that complements firewalls, SELinux, IAM controls, and SSH hardening.

---

## 📄 Document Control

| Item | Details |
|---|---|
| Document Name | Secure RHEL / Rocky Linux Server – Fail2Ban SOP |
| Environment | RHEL 8 / 9, Rocky Linux 8 / 9 |
| Scope | Intrusion prevention, SSH protection, abuse detection |
| Audience | Cloud Engineers, DevOps, SysAdmins, Security Teams |
| Owner | Platform / Infrastructure / Security Team |

---

## 🎯 Purpose

The purpose of this SOP is to establish a **secure, repeatable, and auditable process** to:

- Detect and block malicious authentication attempts
- Automatically ban abusive IP addresses
- Protect SSH, PAM, sudo, and OTP/MFA workflows
- Generate actionable **email alerts for security events**
- Reduce risk from brute-force and credential attacks
- Meet **enterprise security and compliance** requirements

---

## 🔐 Core Security Principles

- Fail2Ban acts as a **host-based intrusion prevention system**
- Bans are enforced at the **firewall level (firewalld)**
- Authentication failures are treated as **security signals**
- Admin IPs are **explicitly whitelisted**
- All bans are **logged, reversible, and auditable**

---

## ✅ Prerequisites

- RHEL / Rocky Linux server fully patched
- SSH hardening completed
- Key-based SSH authentication enabled
- `sudo` access available
- (Optional) PAM-based OTP / MFA enabled
- Outbound SMTP access for alerts
- firewalld enabled and running
- SELinux in Enforcing or Permissive mode

---

## 🧠 Fail2Ban Operational Model

Fail2Ban operates by:

- Monitoring authentication logs via systemd / journald
- Matching failures using regex-based filters
- Applying bans using **firewalld**
- Automatically expiring bans after a defined duration
- Detecting repeat offenders via the `recidive` jail

Fail2Ban **augments but does not replace** firewalls, SELinux, or IAM.

---

## 🪜 Step 1: Enable EPEL Repository

Fail2Ban is available via EPEL.

```bash
sudo dnf install -y epel-release
sudo dnf update -y
```

---

## 🪜 Step 2: Install Fail2Ban

```bash
sudo dnf install -y fail2ban fail2ban-firewalld
```

Verify installation:

```bash
fail2ban-client --version
```

---

## 🪜 Step 3: Install Email Dependencies

```bash
sudo dnf install -y sendmail mailx
```

Validate email delivery:

```bash
echo "Fail2Ban test" | mail -s "Fail2Ban Test" security@yourdomain.com
```

---

## 🪜 Step 4: Configuration Strategy

| File | Purpose |
|---|---|
| `/etc/fail2ban/jail.conf` | Default configuration (**DO NOT EDIT**) |
| `/etc/fail2ban/jail.local` | Enterprise overrides |
| `/etc/fail2ban/filter.d/` | Custom filters |

All production configuration **MUST be placed in `jail.local`**.

---

## 🪜 Step 5: Enterprise Fail2Ban Configuration

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

banaction = firewallcmd-ipset
logtarget = /var/log/fail2ban.log
loglevel  = INFO


[sshd]
enabled  = true
port     = ssh
maxretry = 3
bantime  = 24h


[recidive]
enabled   = true
logpath   = /var/log/fail2ban.log
findtime  = 1d
bantime   = 7d
maxretry  = 5


[pam-generic]
enabled  = true
logpath  = /var/log/secure
maxretry = 3
bantime  = 12h


[sudo]
enabled  = true
logpath  = /var/log/secure
maxretry = 3
bantime  = 6h
```

---

## 🪜 Step 6: Apply and Activate Fail2Ban

```bash
sudo systemctl stop fail2ban
sudo rm -f /var/run/fail2ban/fail2ban.sock
sudo systemctl daemon-reexec
sudo systemctl start fail2ban
sudo systemctl enable fail2ban
```

Ensure firewalld is active:

```bash
sudo systemctl enable firewalld --now
```

---

## 🪜 Step 7: Validation & Health Checks

```bash
sudo systemctl status fail2ban --no-pager
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

Expected active jails:

- `sshd`
- `recidive`
- `pam-generic`
- `sudo`

---

## 📧 Email Alert Validation

```bash
sudo fail2ban-client set sshd banip 8.8.8.8
sudo fail2ban-client set sshd unbanip 8.8.8.8
```

---

## 🛠️ Operational Commands

| Task | Command |
|---|---|
| View all jails | `fail2ban-client status` |
| View SSH jail | `fail2ban-client status sshd` |
| Unban IP | `fail2ban-client set sshd unbanip <IP>` |
| View logs | `tail -f /var/log/fail2ban.log` |

---

## 🧯 Emergency Rollback (Break-Glass)

```bash
sudo systemctl stop fail2ban
```

All active bans are immediately removed.

---

## 🏢 Enterprise Best Practices

- Whitelist only static admin IPs or VPN ranges
- Always enable the `recidive` jail in production
- Combine Fail2Ban with SSH hardening and MFA
- Forward logs to a SIEM (ELK, Splunk, Sentinel)
- Monitor SELinux AVC denials if SSH behavior changes

---

## 📜 Compliance Alignment

- CIS RHEL Benchmark
- ISO/IEC 27001 – Monitoring & Access Control
- SOC 2 – Security & Availability
- NIST SP 800-53 (AC, AU, IR)

---

## ✅ Final Note

Fail2Ban is a **mandatory host-level security control** for RHEL / Rocky Linux production servers.

If bans are not enforced, alerts are not delivered, or rollback is not immediate — **the system is not production-ready**.
