# SOP — Linux Permission Management
**Standard Operating Procedure | Linux System Administration**

---

## Overview

| Item | Detail |
|------|--------|
| **Purpose** | Understand, read, and manage file/directory permissions in Linux |
| **Target Systems** | All Linux distributions |
| **Applies To** | Linux Sysadmins, DevOps Engineers, DBAs |

---

## Background: Why Permissions Exist

**What:** Linux is a multi-user OS. Multiple users and processes access the same files simultaneously.

**Why:** Without permissions, any user could read, modify, or delete any file — including system files, passwords, and other users' data. Permissions enforce who can do what to every file and directory.

> Rule of thumb: **Grant the minimum permissions needed.** Nothing more.

---

## Core Concepts

### The Three Permission Categories

Every file has permissions defined for three groups of users:

| Category | Symbol | Who It Applies To |
|----------|--------|-------------------|
| Owner (User) | `u` | The user who created/owns the file |
| Group | `g` | All users who belong to the file's assigned group |
| Others | `o` | Everyone else on the system |
| All | `a` | Shorthand for owner + group + others |

### The Three Permission Types

| Permission | Symbol | Numeric | On a File | On a Directory |
|------------|--------|---------|-----------|----------------|
| Read | `r` | `4` | View file contents | List directory contents (`ls`) |
| Write | `w` | `2` | Modify file | Create or delete files inside |
| Execute | `x` | `1` | Run as a program | Enter the directory (`cd`) |
| None | `-` | `0` | — | — |

---

## Reading Permission Output

**What:** The `ls -l` command shows the full permission string for every file.

```bash
ls -l filename
```

**Example output:**
```
-rwxr-xr-- 1 alice devs 1024 Mar 11 10:00 script.sh
```

**Breaking it down:**

```
- rwx r-x r--
│ │   │   │
│ │   │   └─ Others:  r-- = read only
│ │   └───── Group:   r-x = read + execute
│ └───────── Owner:   rwx = read + write + execute
└─────────── File type: - = file  |  d = directory  |  l = symlink
```

---

## `chmod` — Change File Permissions

**What:** Modifies the read, write, and execute permissions of a file or directory.
**Why:** Controls who can access or modify your files. Essential for securing scripts, config files, and web directories.

---

### Method 1 — Symbolic Mode

**What:** Use human-readable letters to add, remove, or set permissions.
**Why:** Easier to read and less error-prone when making targeted changes (e.g. "just add execute for owner").

**Syntax:** `chmod [who][operator][permission] file`

| Operator | Meaning |
|----------|---------|
| `+` | Add permission |
| `-` | Remove permission |
| `=` | Set exact permission (replaces existing) |

```bash
# Add execute permission for owner
chmod u+x script.sh

# Remove write permission from group
chmod g-w file.txt

# Add read & write for owner, remove execute from others
chmod u+rw,o-x file.txt

# Set exact permissions (owner=rwx, group=rx, others=r)
chmod u=rwx,g=rx,o=r file.txt

# Add execute for everyone
chmod a+x script.sh

# Remove all permissions for others
chmod o= file.txt
```

---

### Method 2 — Numeric (Octal) Mode

**What:** Use numbers to set all permissions at once.
**Why:** Faster and precise — sets all three categories in one value. Preferred in scripts and automation.

**How to calculate:**

```
r = 4,  w = 2,  x = 1  →  add the values for each category

rwx = 4+2+1 = 7
r-x = 4+0+1 = 5
r-- = 4+0+0 = 4
--- = 0
```

**Common permission sets:**

| Octal | Permission String | Typical Use |
|-------|-------------------|-------------|
| `777` | `rwxrwxrwx` | Full access for all — **avoid in production** |
| `755` | `rwxr-xr-x` | Owner full control, others read+execute (web dirs, scripts) |
| `644` | `rw-r--r--` | Owner read+write, others read only (config files, web files) |
| `600` | `rw-------` | Owner only (SSH keys, private files) |
| `700` | `rwx------` | Owner only, executable (private scripts) |

```bash
chmod 777 file.txt      # Full access for all (AVOID in production)
chmod 755 script.sh     # Standard for scripts and public directories
chmod 644 file.txt      # Standard for regular files
chmod 600 ~/.ssh/id_rsa # Private key — owner only
```

---

### `chmod` Flags

| Flag | Example | Meaning |
|------|---------|---------|
| `-R` | `chmod -R 755 /var/www/html/` | Apply recursively to directory and all contents |
| `-v` | `chmod -v 644 file.txt` | Verbose — print what was changed |

---

## `chown` — Change File Owner and Group

**What:** Changes who owns a file and/or which group it belongs to.
**Why:** File permissions only work correctly when ownership is assigned correctly. If a process runs as user `mysql`, it needs to own its data files — otherwise it cannot read or write them even with correct permissions.

**Syntax:** `chown [owner][:group] file`

```bash
# Change owner only
chown alice file.txt

# Change owner and group
chown alice:devs file.txt

# Change group only (note the colon prefix)
chown :devs file.txt

# Only change if current owner matches a specific user
chown --from=bob alice file.txt
```

### `chown` Flags

| Flag | Example | Meaning |
|------|---------|---------|
| `-R` | `chown -R alice /home/alice/` | Recursively change ownership of directory and contents |
| `-v` | `chown -v alice:devs file.txt` | Verbose — print each file changed |
| `--from` | `chown --from=bob alice file.txt` | Only change if current owner matches the given user |

---

## `chgrp` — Change Group Ownership

**What:** Changes only the group ownership of a file or directory.
**Why:** A shortcut when you only need to reassign the group without touching the owner. Useful when sharing files between teams.

```bash
# Change group of a single file
chgrp devs file.txt

# Change group recursively
chgrp -R devs /var/www/html/
```

### `chgrp` Flags

| Flag | Example | Meaning |
|------|---------|---------|
| `-R` | `chgrp -R devs /var/www/html/` | Apply recursively to all files and subdirectories |
| `-v` | `chgrp -v devs file.txt` | Verbose — print each file changed |

---

## `umask` — Default Permission Mask

**What:** A value that automatically restricts permissions when new files and directories are created.
**Why:** Without umask, new files might be created with overly permissive defaults. umask ensures new files never start with more access than intended.

### How umask Works

Linux starts with a maximum:
- Files: `666` (no execute by default)
- Directories: `777`

Then it **subtracts** the umask:

```
Files:       666 - umask = actual file permissions
Directories: 777 - umask = actual directory permissions
```

**Examples:**

| umask | New Files | New Dirs | Use Case |
|-------|-----------|----------|----------|
| `022` | `644` (rw-r--r--) | `755` (rwxr-xr-x) | Standard — others can read |
| `027` | `640` (rw-r-----) | `750` (rwxr-x---) | Tighter — others have no access |
| `077` | `600` (rw-------) | `700` (rwx------) | Private — owner only |

```bash
# View current umask (numeric)
umask

# View current umask (symbolic, human-readable)
umask -S

# Set umask for current session only
umask 022
umask 027       # More restrictive: no permissions for others

# Set umask permanently (survives reboot)
echo "umask 022" >> ~/.bashrc       # For current user
# or
echo "umask 022" >> /etc/profile    # For all users system-wide
```

### `umask` Flags

| Flag | Example | Meaning |
|------|---------|---------|
| *(no flag)* | `umask` | Display current umask as numeric value |
| `-S` | `umask -S` | Display current umask in symbolic format |

---

## Quick Reference — Key Commands

```bash
# ── View Permissions ──────────────────────────────
ls -l file             # Long listing with permissions
ls -la                 # Include hidden files
stat file              # Detailed file info including all timestamps

# ── Change Permissions ────────────────────────────
chmod 755 file         # Numeric mode — set all at once
chmod u+x file         # Symbolic mode — targeted change
chmod -R 644 dir/      # Recursive — apply to all contents

# ── Change Ownership ──────────────────────────────
chown user file        # Change owner only
chown user:group file  # Change owner and group
chown :group file      # Change group only
chgrp group file       # Change group only (alternative)
chown -R user:group /path/  # Recursive ownership change

# ── Default Permissions ───────────────────────────
umask                  # View current umask (numeric)
umask -S               # View current umask (symbolic)
umask 022              # Set umask for current session
```

---

## Common Permission Patterns (Production Reference)

| File Type | Recommended Permission | Reason |
|-----------|------------------------|--------|
| Web files (`/var/www`) | `644` / `755` | Readable by web server, not writable |
| Shell scripts | `755` | Executable by all, writable only by owner |
| Config files | `600` or `640` | Sensitive — restrict to owner or group |
| SSH private keys | `600` | Required by SSH — will refuse to use if too open |
| Database data dirs | `700` (owned by db user) | Only the DB process should access |
| Shared group dirs | `775` | Owner and group can write, others read only |

---

## Summary

| Command | Purpose | Key Flag |
|---------|---------|----------|
| `chmod` | Change permissions on a file/dir | `-R` recursive |
| `chown` | Change owner and/or group | `-R` recursive |
| `chgrp` | Change group only | `-R` recursive |
| `umask` | Set default permissions for new files | `-S` symbolic view |

---


