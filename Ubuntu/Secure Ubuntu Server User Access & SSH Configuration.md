# 🔐 Secure Ubuntu Server User Access & SSH Configuration

> **Enterprise-grade SOP for safe, auditable, and repeatable SSH access on Ubuntu servers (AWS EC2 & beyond).**

---

## 📌 Overview

This document provides a **production-ready Standard Operating Procedure (SOP)** for managing **user access and SSH configuration** on **Ubuntu servers**, with first‑class support for **AWS EC2**.

It is intentionally written to be:

* 🔁 **Repeatable** — clear steps that work the same way every time
* 🔍 **Auditable** — explicit controls, validations, and logs
* 🛡️ **Security‑first** — least privilege, no shortcuts
* 🏢 **Enterprise & compliance aligned** — suitable for regulated environments

The SOP minimizes operational risk, **prevents administrative lockout**, and establishes a hardened SSH baseline suitable for long‑term operations.

---

## 📄 Document Control

| Item          | Details                                                  |
| ------------- | -------------------------------------------------------- |
| Document Name | Secure Ubuntu Server User Access & SSH Configuration     |
| Environment   | Ubuntu Server (AWS EC2 / On‑prem compatible)             |
| Scope         | User access, SSH hardening, key management               |
| Audience      | Cloud Engineers, DevOps Engineers, System Administrators |
| Owner         | Platform / Infrastructure / DevOps Team                  |

---

## 🎯 Purpose

The purpose of this SOP is to establish a **secure, repeatable, and auditable process** to:

* Create a dedicated administrative user on Ubuntu
* Enforce **key‑based SSH authentication only**
* Apply correct ownership and filesystem permissions
* Deploy a **hardened and future‑proof SSH configuration**
* Ensure continuous administrative access (break‑glass readiness)
* Meet **enterprise security and compliance** expectations

---

## 🔐 Core Security Principles

* SSH private keys are treated as **privileged credentials**
* Password‑based SSH authentication is **prohibited**
* Direct root login over SSH is **never allowed**
* Access must be **explicitly granted and attributable** to a user
* At least one **break‑glass access path** must always be maintained

---

## ✅ Prerequisites

* Ubuntu Server running on AWS EC2 (or equivalent environment)
* SSH access using the default `ubuntu` user
* Local SSH private key (`.pem`) available
* User has `sudo` privileges
* AWS Security Group restricts **port 22** to approved source IPs only

---

## 🧠 Default Ubuntu EC2 Access Model

Understanding the default model is critical to avoid lockout:

* Default SSH user: `ubuntu`
* SSH keys are injected **only for `ubuntu`** at instance launch
* Root account exists but is **disabled for SSH**
* Any additional user **must be explicitly configured** with SSH keys

---

## 🪜 Step 1: Verify Administrative Access

```bash
ssh -i yourkey.pem ubuntu@<EC2_PUBLIC_IP>
sudo whoami
```

**Expected output:**

```text
root
```

This confirms administrative access before making changes.

---

## 🪜 Step 2: Create a Dedicated Administrative User

```bash
sudo adduser mess-central-admin
sudo usermod -aG sudo mess-central-admin
```

**Outcome:**

* Administrative user created
* Full privileged access granted via `sudo` (no root login required)

---

## 🪜 Step 3: Configure SSH Key Access

### Create SSH Directory

```bash
sudo mkdir -p /home/mess-central-admin/.ssh
sudo chmod 700 /home/mess-central-admin/.ssh
```

### Copy Existing Authorized Keys

```bash
sudo cp /home/ubuntu/.ssh/authorized_keys \
        /home/mess-central-admin/.ssh/authorized_keys
```

### Apply Ownership (Explicit)

```bash
sudo chown mess-central-admin:mess-central-admin /home/mess-central-admin
sudo chown mess-central-admin:mess-central-admin /home/mess-central-admin/.ssh
sudo chown mess-central-admin:mess-central-admin \
     /home/mess-central-admin/.ssh/authorized_keys
```

### Apply Permissions (Critical for SSH)

```bash
sudo chmod 755 /home/mess-central-admin
sudo chmod 700 /home/mess-central-admin/.ssh
sudo chmod 600 /home/mess-central-admin/.ssh/authorized_keys
```

### Verify

```bash
ls -ld /home/mess-central-admin \
       /home/mess-central-admin/.ssh \
       /home/mess-central-admin/.ssh/authorized_keys
```

---

## 🪜 Step 4: Harden the SSH Server Configuration

Edit the SSH daemon configuration:

```bash
sudo nano /etc/ssh/sshd_config
```

### ✅ Approved & Hardened Configuration

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

## 🪜 Step 5: Validate and Apply SSH Configuration

```bash
sudo sshd -t
sudo systemctl restart ssh
sudo systemctl status ssh
```

> ⚠️ **Keep the existing SSH session open** until validation is complete to avoid lockout.

---

## 🪜 Step 6: Verify New User Login

From your local machine:

```bash
ssh -i yourkey.pem mess-central-admin@<EC2_PUBLIC_IP>
```

**Expected outcome:**

* Successful login
* No password prompt
* Full `sudo` privileges available

---

## 🛠️ Troubleshooting

### Monitor Authentication Logs

```bash
sudo tail -f /var/log/auth.log
```

**Common causes of failure:**

* Incorrect permissions on `.ssh` or `authorized_keys`
* Incorrect file ownership
* Missing or malformed public key
* User not listed in `AllowUsers`

---

## 🔄 SSH Key Loss or Compromise Response

### Immediate Containment

```bash
sudo usermod -L mess-central-admin
```

**Or disable SSH access only:**

```bash
sudo mv /home/mess-central-admin/.ssh/authorized_keys \
        /home/mess-central-admin/.ssh/authorized_keys.disabled
```

This action immediately blocks SSH access while preserving the user account.

---

## 🔁 SSH Key Rotation Procedure

### Generate a New Key (User Machine)

```bash
ssh-keygen -t ed25519 -a 100 -C "mess-central-admin@company"
```

### Install the New Public Key

```bash
sudo nano /home/mess-central-admin/.ssh/authorized_keys
```

Remove all old keys and add **only** the new public key.

### Restore Permissions

```bash
sudo chmod 700 /home/mess-central-admin/.ssh
sudo chmod 600 /home/mess-central-admin/.ssh/authorized_keys
sudo chown mess-central-admin:mess-central-admin \
     /home/mess-central-admin/.ssh/authorized_keys
```

### Re‑enable User

```bash
sudo usermod -U mess-central-admin
```

---

## 🏢 Enterprise Best Practices

* One user = one SSH key (no sharing)
* Rotate SSH keys every **90–180 days**
* Keep password‑based SSH disabled permanently
* Prefer **AWS SSM Session Manager** where available
* Maintain a documented and audited **break‑glass account**

---

## 📜 Compliance Alignment

* CIS Linux & AWS Benchmarks
* ISO/IEC 27001 — Access Control
* SOC 2 — Logical Access Controls
* NIST SP 800‑53

---

## ✅ Final Note

This README defines a **production‑approved, enterprise‑ready SSH access baseline** for Ubuntu servers. It is intended to be reused across environments and adapted only where organizational policy requires.

*If a step is not explicit, repeatable, and reversible—it does not belong in production.*