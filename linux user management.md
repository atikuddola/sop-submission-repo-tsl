# SOP — Linux User Management
**Standard Operating Procedure | Linux System Administration**

---

## Overview

| Item | Detail |
|------|--------|
| **Purpose** | Create, modify, delete, and manage users and groups on Linux |
| **Target Systems** | All Linux distributions |
| **Applies To** | Linux Sysadmins, DevOps Engineers, DBAs |

---

## Background

Linux is a multi-user OS. Every file, process, and service is tied to a user account. Proper user management controls who can log in, what they can access, and what privileges they hold.

**Key system files:**

| File | Purpose |
|------|---------|
| `/etc/passwd` | Stores user account info (name, UID, home dir, shell) |
| `/etc/shadow` | Stores encrypted passwords and expiry settings |
| `/etc/group` | Stores group definitions and memberships |

---

## 1. View Users

```bash
cat /etc/passwd      # List all system users
who                  # Currently logged-in users
w                    # Logged-in users with activity detail
whoami               # Your current username
```

---

## 2. Create Users

```bash
sudo useradd username                              # Basic user (no home dir)
sudo useradd -m username                           # Create with home directory
sudo useradd -m -u 1500 username                   # Set specific UID
sudo useradd -m -g groupname username              # Assign primary group
sudo useradd -m -G group1,group2 username          # Add to supplementary groups
sudo useradd -m -d /custom/home username           # Custom home directory path
sudo useradd -m -e 2025-12-31 username             # Set account expiry date
sudo useradd --system --no-create-home username    # System user (no login)
```

**Key flags:**

| Flag | Meaning |
|------|---------|
| `-m` | Create home directory |
| `-u` | Set UID manually |
| `-g` | Set primary group |
| `-G` | Add to supplementary groups |
| `-d` | Custom home directory path |
| `-e` | Account expiry date (YYYY-MM-DD) |
| `--system` | Create a system account (for services, not humans) |

---

## 3. Modify Users

```bash
sudo usermod -l newname oldname               # Rename user
sudo usermod -d /new/home -m username         # Change home dir and move files
sudo usermod -s /bin/bash username            # Change default shell
sudo usermod -g newgroup username             # Change primary group
sudo usermod -aG groupname username           # Add to supplementary group
sudo usermod -u 2000 username                 # Change UID
sudo gpasswd -d username groupname            # Remove from a group
```

> Always use `-aG` (not just `-G`) when adding to groups — `-G` alone **replaces** all existing groups.

---

## 4. Delete Users

```bash
sudo userdel username       # Delete user, keep home directory
sudo userdel -r username    # Delete user AND home directory
```

---

## 5. Password Management

```bash
sudo passwd username         # Set or change a user's password
passwd                       # Change your own password
sudo passwd -e username      # Force password change on next login
sudo passwd -d username      # Remove password (disables password login)
sudo passwd -l username      # Lock account
sudo passwd -u username      # Unlock account
sudo passwd -S username      # Check account status (locked/active)
```

**Password aging with `chage`:**

```bash
sudo chage -l username       # View all expiry/aging info
sudo chage -M 90 username    # Password expires after 90 days
sudo chage -m 7 username     # Min 7 days between password changes
sudo chage -W 14 username    # Warn user 14 days before expiry
sudo chage -M -1 username    # Password never expires
```

---

## 6. Group Management

```bash
cat /etc/group               # List all groups
groups username              # List groups a user belongs to
id username                  # Show UID, GID, and all groups

sudo groupadd groupname           # Create a group
sudo groupadd -g 1500 groupname   # Create group with specific GID
sudo groupmod -n newname oldname  # Rename a group
sudo groupmod -g 2000 groupname   # Change GID
sudo groupdel groupname           # Delete a group
getent group groupname            # List members of a group
```

---

## 7. Sudo & Privileges

**What:** Grants users the ability to run commands as root.
**Why:** Avoids logging in as root directly — safer and auditable.

```bash
sudo usermod -aG wheel username                                       # Add to wheel group (RHEL/AlmaLinux)
sudo visudo                                                           # Safely edit sudoers file

# Grant full sudo access
echo "username ALL=(ALL:ALL) ALL" | sudo tee /etc/sudoers.d/username

# Grant passwordless sudo
echo "username ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/username

# Allow only a specific command
echo "username ALL=(ALL) NOPASSWD:/usr/bin/systemctl" | sudo tee /etc/sudoers.d/username

# Check what sudo rights a user has
sudo -l -U username

# Remove sudo (RHEL/AlmaLinux)
sudo gpasswd -d username wheel
```

> Always use `visudo` to edit the sudoers file directly — it validates syntax before saving and prevents lockouts.

---

## 8. Lock, Unlock & Account Expiry

```bash
# Lock / Unlock
sudo passwd -l username                      # Lock (passwd method)
sudo passwd -u username                      # Unlock (passwd method)
sudo usermod -L username                     # Lock (usermod method)
sudo usermod -U username                     # Unlock (usermod method)
sudo usermod -s /sbin/nologin username       # Disable login via shell
sudo usermod -s /bin/bash username           # Re-enable login

# Account Expiry
sudo usermod -e 2025-12-31 username          # Set expiry date
sudo chage -E 2025-12-31 username            # Set expiry (chage method)
sudo chage -E -1 username                    # Remove expiry (never expires)
sudo chage -E 0 username                     # Expire account immediately
sudo chage -l username                       # View all expiry details
```

---

## 9. User Information & Login History

```bash
id username              # UID, GID, group memberships
getent passwd username   # Full account record from /etc/passwd
last username            # Login history for a user
lastlog                  # Last login time for all users
sudo lastb               # Failed login attempts
```

---

## Quick Reference

```bash
# Users
useradd -m username           # Create user with home dir
usermod -aG group username    # Add to group
userdel -r username           # Delete user + home dir

# Passwords
passwd username               # Set password
chage -l username             # View expiry info
passwd -l / -u username       # Lock / unlock

# Groups
groupadd groupname            # Create group
gpasswd -d username group     # Remove from group
groups username               # List user's groups

# Sudo
usermod -aG wheel username    # Grant sudo (RHEL)
sudo -l -U username           # Check sudo rights
```

---
