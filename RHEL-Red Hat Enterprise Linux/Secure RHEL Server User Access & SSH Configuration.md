# 🔐 Secure RHEL Server User Access & SSH Configuration

> **Enterprise‑grade SOP for safe, auditable, and repeatable SSH access on Red Hat Enterprise Linux (RHEL) servers (AWS EC2 & on‑prem).**

---

## 📌 Overview

This document provides a **production‑ready Standard Operating Procedure (SOP)** for managing **user access and SSH configuration** on **Red Hat Enterprise Linux (RHEL 8/9)** systems, with support for **AWS EC2 and on‑prem deployments**.

It is intentionally written to be:

* 🔁 **Repeatable** — deterministic steps that behave consistently
* 🔍 **Auditable** — explicit controls, logs, and validation points
* 🛡️ **Security‑first** — least privilege, no shortcuts
* 🏢 **Enterprise & compliance aligned** — suitable for regulated environments

This SOP minimizes operational risk, **prevents administrative lockout**, and establishes a hardened SSH baseline aligned with Red Hat and CIS benchmarks.

---

## 📄 Document Control

| Item          | Details                                            |
| ------------- | -------------------------------------------------- |
| Document Name | Secure RHEL Server User Access & SSH Configuration |
| Environment   | RHEL 8 / RHEL 9 (AWS EC2 & On‑prem)                |
| Scope         | User access, SSH hardening, key management         |
| Audience      | Linux Admins, Cloud Engineers, DevOps Engineers    |
| Owner         | Platform / Infrastructure / DevOps Team            |

---

## 🎯 Purpose

The purpose of this SOP is to establish a **secure, repeatable, and auditable process** to:

* Create a dedicated administrative user on RHEL
* Enforce **key‑based SSH authentication only**
* Apply correct ownership and filesystem permissions
* Deploy a **hardened and future‑proof SSH configuration**
* Ensure continuous administrative access (break‑glass readiness)
* Meet **enterprise security and compliance** requirements

---

## 🔐 Core Security Principles

* SSH private keys are treated as **privileged credentials**
* Password‑based SSH authentication is **prohibited**
* Direct root login over SSH is **never allowed**
* Access must be **explicitly granted and attributable**
* At least one **break‑glass access path** must always exist

---

## ✅ Prerequisites

* RHEL 8 or RHEL 9 system
* SSH access using an existing administrative user (e.g. `ec2-user`)
* Local SSH private key available
* User has `sudo` access via the **wheel** group
* AWS Security Group / firewall allows **port 22** from approved IPs only
* SELinux enabled (recommended)

---

## 🧠 Default RHEL Access Model

Understanding the default RHEL model is critical:

* Root account exists but is **disabled for SSH login**
* Administrative access is provided via users in the **wheel** group
* SSH keys must be configured **per user**
* Authentication logs are written to `/var/log/secure`

---

## 🪜 Step 1: Verify Administrative Access

```bash
ssh -i yourkey.pem ec2-user@<SERVER_IP>
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
sudo useradd -m -c "RHEL Admin User" mess-central-admin
sudo passwd mess-central-admin
sudo usermod -aG wheel mess-central-admin
```

**Outcome:**

* Administrative user created
* Privileged access granted via `sudo` (wheel group)

---

## 🪜 Step 3: Configure SSH Key Access

### Create SSH Directory

```bash
sudo mkdir -p /home/mess-central-admin/.ssh
sudo chmod 700 /home/mess-central-admin/.ssh
```

### Copy Existing Authorized Keys

```bash
sudo cp /home/ec2-user/.ssh/authorized_keys \
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
sudo vi /etc/ssh/sshd_config
```

### ✅ Approved & Hardened Configuration (RHEL)

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

AllowUsers ec2-user mess-central-admin

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

SyslogFacility AUTHPRIV
LogLevel VERBOSE

AcceptEnv LANG LC_*
PermitUserEnvironment no

Subsystem sftp /usr/libexec/openssh/sftp-server

Match User mess-central-admin
    PubkeyAuthentication yes
    PasswordAuthentication no
```

---

## 🪜 Step 5: Validate and Apply SSH Configuration

```bash
sudo sshd -t
sudo systemctl restart sshd
sudo systemctl status sshd
```

> ⚠️ **Keep the existing SSH session open** until validation is complete.

---

## 🪜 Step 6: Verify New User Login

From your local machine:

```bash
ssh -i yourkey.pem mess-central-admin@<SERVER_IP>
```

**Expected outcome:**

* Successful login
* No password prompt
* Full `sudo` privileges available

---

## 🛠️ Troubleshooting

### Monitor Authentication Logs

```bash
sudo tail -f /var/log/secure
```

**Common causes of failure:**

* Incorrect permissions on `.ssh` or `authorized_keys`
* Incorrect ownership
* Missing or malformed public key
* User not listed in `AllowUsers`
* SELinux context issues

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

---

## 🔁 SSH Key Rotation Procedure

### Generate a New Key (User Machine)

```bash
ssh-keygen -t ed25519 -a 100 -C "mess-central-admin@company"
```

### Install the New Public Key

```bash
sudo vi /home/mess-central-admin/.ssh/authorized_keys
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
* Prefer **AWS SSM Session Manager** or Red Hat Identity Management where available
* Maintain a documented and audited **break‑glass account**
* Do not disable SELinux; fix contexts instead

---

## 📜 Compliance Alignment

* CIS Red Hat Enterprise Linux Benchmark
* CIS AWS Foundations Benchmark
* ISO/IEC 27001 — Access Control
* SOC 2 — Logical Access Controls
* NIST SP 800‑53

---

## ✅ Final Note

This README defines a **production‑approved, enterprise‑ready SSH access baseline** for RHEL systems. It is intended to be reused across environments and adapted only where organizational policy requires.

*If a step is not explicit, repeatable, and reversible—it does not belong in production.*
