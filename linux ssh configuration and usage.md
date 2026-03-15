# SOP — SSH Configuration on Rocky Linux
**Standard Operating Procedure | Linux System Administration**

---

## Overview

| Item | Detail |
|------|--------|
| **Purpose** | Install, configure, and secure SSH on Rocky Linux |
| **Target Systems** | Rocky Linux 8 / 9 |
| **Applies To** | Linux Sysadmins, DevOps Engineers, DBAs |

---

## Background

**What:** SSH (Secure Shell) is the standard protocol for remote access to Linux servers. All communication is encrypted.

**Two authentication methods:**

| Method | Security | Use Case |
|--------|----------|----------|
| Password | Moderate | Quick setup, testing |
| Key-based | High | Production, automation |

> Key-based authentication is always preferred in production. Passwords are brute-forceable; keys are not.

---

## Prerequisites

| Requirement | Detail |
|-------------|--------|
| OS | Rocky Linux 8 / 9 |
| User | Root or sudo privileges |
| Network | Port 22 accessible |
| Client | Linux/macOS terminal, Windows PuTTY/WSL |

---

## 1. Install & Enable SSH Server

```bash
sudo dnf update -y
sudo dnf install -y openssh-server openssh-clients

sudo systemctl start sshd
sudo systemctl enable sshd

# Verify
sudo systemctl status sshd
```

Expected: `Active: active (running)`

---

## 2. Password Authentication

### Server Side — Enable Password Auth

Edit the SSH config:

```bash
sudo nano /etc/ssh/sshd_config
```

Set these values:

```
PasswordAuthentication yes
PermitRootLogin no
```

```bash
sudo systemctl restart sshd
```

### Create a User (if needed)

```bash
sudo useradd -m samrat
sudo passwd samrat
```

### Connect from Client

```bash
ssh samrat@<SERVER_IP>
# Example:
ssh samrat@192.168.226.131
```

You will be prompted for the user's password.

---

## 3. Key-Based Authentication

### Step 1 — Generate Key Pair (Client Side)

Run on your **local machine**, not the server:

```bash
# Preferred (modern)
ssh-keygen -t ed25519 -C "your-comment"

# Legacy systems
ssh-keygen -t rsa -b 4096 -C "your-comment"
```

**Flag explained:**

| Flag | Meaning |
|------|---------|
| `-t` | Key type (`ed25519` or `rsa`) |
| `-b` | Key size in bits (RSA only) |
| `-C` | Comment label for the key |

Press `Enter` for default path. Always set a **passphrase** when prompted.

**Generated files:**

| File | Purpose |
|------|---------|
| `~/.ssh/id_ed25519` | Private key — **never share this** |
| `~/.ssh/id_ed25519.pub` | Public key — copy this to the server |

### Step 2 — Copy Public Key to Server

```bash
ssh-copy-id samrat@192.168.226.131

# Specify key explicitly
ssh-copy-id -i ~/.ssh/id_ed25519.pub samrat@192.168.226.131
```

This appends your public key to `~/.ssh/authorized_keys` on the server.

### Step 3 — Set Correct Permissions on Server

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

> SSH silently rejects keys if permissions are too open. This step is critical.

### Step 4 — Enable Key Auth in SSH Config (Server)

```bash
sudo vi /etc/ssh/sshd_config
```

Ensure these are set:

```
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
```

```bash
sudo systemctl restart sshd
```

### Step 5 — Connect Using Key

```bash
ssh samrat@192.168.226.131

# Specify key explicitly
ssh -i ~/.ssh/id_ed25519 samrat@192.168.226.131
```

---

## 4. Harden SSH Configuration

Edit `/etc/ssh/sshd_config` and apply:

```
PermitRootLogin no           # Disable direct root login
PasswordAuthentication no    # Force key-only login
MaxAuthTries 3               # Limit brute-force attempts
```

**Validate and restart:**

```bash
sudo sshd -t                 # Test config for syntax errors — run BEFORE restart
sudo systemctl restart sshd  # Apply only if no errors
```

> Always run `sshd -t` before restarting. A bad config can lock you out of the server.

---

## 5. Firewall Configuration

```bash
sudo firewall-cmd --permanent --add-service=ssh     # Allow port 22
sudo firewall-cmd --permanent --add-port=2222/tcp   # Allow custom port (if changed)
sudo firewall-cmd --reload
```

---

## 6. Troubleshooting

**Fix permission issues (client side):**

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
chmod 600 ~/.ssh/authorized_keys
```

**Check service and logs:**

```bash
sudo systemctl status sshd
sudo journalctl -u sshd -n 50 --no-pager
sudo tail -f /var/log/secure
```

**Verify SSH port is listening:**

```bash
sudo ss -tlnp | grep sshd
```

**Common errors:**

| Error | Cause | Fix |
|-------|-------|-----|
| `Connection refused` | sshd not running or port blocked | Start sshd, check firewall |
| `Permission denied (publickey)` | Wrong key or bad permissions | Check `authorized_keys` and file permissions |
| `Host key verification failed` | Server key changed | Run `ssh-keygen -R <IP>` |
| `Too many authentication failures` | Multiple keys tried | Use `-i` to specify exact key |
| `UNPROTECTED PRIVATE KEY FILE` | Key permissions too open | `chmod 600 ~/.ssh/id_ed25519` |

---

## Quick Reference

```bash
# ── Install & Service ─────────────────────────────
sudo dnf install -y openssh-server
sudo systemctl enable --now sshd
sudo systemctl restart sshd
sudo sshd -t                                        # Validate config

# ── Key Generation (Client) ───────────────────────
ssh-keygen -t ed25519 -C "comment"
ssh-copy-id user@server

# ── Connect ───────────────────────────────────────
ssh user@192.168.1.100
ssh -p 2222 user@192.168.1.100
ssh -i ~/.ssh/id_ed25519 user@192.168.1.100

# ── Firewall ──────────────────────────────────────
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
```

---

## Security Checklist

| Setting | Recommended Value |
|---------|-------------------|
| Root login | `PermitRootLogin no` |
| Password auth | `PasswordAuthentication no` |
| Key type | Ed25519 (preferred) or RSA 4096-bit |
| Key passphrase | Always set one |
| Auth attempts | `MaxAuthTries 3` |
| Default port | Change from 22 to a high port (e.g., 2222) |

---

*SOP Version 1.0 — SSH Configuration on Rocky Linux*
