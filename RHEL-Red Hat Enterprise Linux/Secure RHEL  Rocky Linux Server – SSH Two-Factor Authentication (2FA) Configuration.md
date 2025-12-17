# 🔐 Secure RHEL / Rocky Linux Server – SSH Two-Factor Authentication (2FA) Configuration

> **Enterprise-grade SOP for safe, auditable, and selective SSH Multi-Factor Authentication (MFA) on RHEL & Rocky Linux servers (AWS EC2 & on‑prem).**

---

## 📌 Overview

This document provides a **production-ready Standard Operating Procedure (SOP)** for implementing **SSH Two-Factor Authentication (2FA / MFA)** on **RHEL and Rocky Linux servers**, with **selective user enforcement**.

It is intentionally written to be:

* 🔁 **Repeatable** — deterministic steps with consistent outcomes
* 🔍 **Auditable** — explicit controls, logs, and validation points
* 🛡️ **Security-first** — layered defense, no global enforcement shortcuts
* 🏢 **Enterprise & compliance aligned** — suitable for regulated environments

This SOP enhances SSH security while **preserving automation stability**, preventing administrative lockout, and maintaining compatibility with Fail2Ban.

---

## 📄 Document Control

| Item          | Details                                                  |
| ------------- | -------------------------------------------------------- |
| Document Name | Secure RHEL / Rocky Linux Server – SSH 2FA Configuration |
| Environment   | RHEL 8 / 9, Rocky Linux 8 / 9                            |
| Scope         | SSH authentication, MFA enforcement                     |
| Audience      | Cloud Engineers, DevOps Engineers, System Administrators |
| Owner         | Platform / Infrastructure / Security Team                |

---

## 🎯 Purpose

The purpose of this SOP is to establish a **secure, repeatable, and auditable process** to:

* Enforce **multi-factor authentication** for SSH access
* Apply MFA **only to selected human users**
* Protect against compromised SSH keys
* Prevent OTP / MFA brute-force attempts
* Maintain uninterrupted automation and service access
* Meet **enterprise security and compliance** expectations

---

## 🔐 Core Security Principles

* SSH keys are the **primary authentication factor**
* OTP (TOTP) is a **secondary verification layer**
* MFA is enforced **only for human users**
* Service and automation accounts are excluded
* Password-based SSH authentication is prohibited
* At least one **break-glass access path** must always exist

---

## ✅ Prerequisites

* RHEL / Rocky Linux server with SSH hardening completed
* Key-based SSH authentication enabled
* Password authentication disabled
* `sudo` access available
* Fail2Ban installed and running
* Console or cloud recovery access available
* Time synchronization (chrony) enabled

---

## 🧠 SSH MFA Access Model

* SSH validates the public key
* PAM invokes OTP verification
* OpenSSH `Match` rules determine MFA enforcement
* Authentication failures are logged via PAM
* Fail2Ban monitors SSH and OTP abuse

MFA **augments but does not replace** SSH hardening or network firewalls.

---

## 🪜 Step 1: Install MFA Dependencies

Enable EPEL repository:

```bash
sudo dnf install -y epel-release
```

Install required packages:

```bash
sudo dnf install -y google-authenticator qrencode
```

---

## 🪜 Step 2: Enroll Selected Users for MFA

Run the following **as the target user (not root)**:

```bash
google-authenticator
```

Secure the OTP file:

```bash
chmod 400 ~/.google_authenticator
```

---

## 🪜 Step 3: PAM Configuration (Selective MFA)

Edit PAM SSH configuration:

```bash
sudo nano /etc/pam.d/sshd
```

Add the following line **at the top**:

```ini
auth required pam_google_authenticator.so nullok
```

---

## 🪜 Step 4: SSH Global Configuration

Edit SSH daemon configuration:

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

## 🪜 Step 5: Enforce MFA for Selected Users

```conf
Match Group ssh-mfa
    ChallengeResponseAuthentication yes
    AuthenticationMethods publickey,keyboard-interactive
```

Add users to MFA group:

```bash
sudo groupadd ssh-mfa
sudo usermod -aG ssh-mfa mess-central-admin
```

---

## 🪜 Step 6: SELinux Considerations

Ensure SELinux allows PAM authentication:

```bash
sudo setsebool -P authlogin_nsswitch_use_ldap on
```

> ℹ️ Default SELinux policies usually allow Google Authenticator.  
> If issues occur, review audit logs:
```bash
sudo ausearch -m avc -ts recent
```

---

## 🪜 Step 7: Validate and Apply

```bash
sudo sshd -t
sudo systemctl restart sshd
sudo systemctl status sshd
```

> ⚠️ Keep at least one active SSH session open during validation.

---

## 🧯 Emergency Access (Break-Glass)

```conf
Match User breakglass
    ChallengeResponseAuthentication no
    AuthenticationMethods publickey
```

Recommended controls:
* IP allowlisting
* Short-lived SSH keys
* Mandatory audit review

---

## 🛠️ Monitoring & Troubleshooting

Authentication logs:

```bash
sudo tail -f /var/log/secure
```

Common failure causes:
* Incorrect OTP file permissions
* Missing `.google_authenticator`
* Time drift (chrony)
* User not matching SSH `Match` rules
* SELinux denials

---

## 🏢 Enterprise Best Practices

* Enforce MFA only for **human administrators**
* Never enable MFA for service accounts
* Use group-based SSH `Match` rules
* Rotate OTP secrets annually
* Forward logs to SIEM (ELK / Splunk / Sentinel)
* Document break-glass usage

---

## 📜 Compliance Alignment

* CIS RHEL & Rocky Linux Benchmarks
* ISO/IEC 27001 — Access Control
* SOC 2 — Logical Access Controls
* NIST SP 800-53 (IA, AC, AU)

---

## ✅ Final Note

This SOP defines a **production-approved, enterprise-ready SSH MFA baseline** for RHEL and Rocky Linux servers.

Selective enforcement, explicit recovery paths, and full auditability are **mandatory for production environments**.
