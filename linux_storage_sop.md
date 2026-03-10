# Linux Storage Management
**Standard Operating Procedure | Linux System Administration**

---

## Overview

| Item | Detail |
|------|--------|
| **Purpose** | Extend existing LVM storage and mount a new disk without LVM |
| **Target Systems** | RHEL / AlmaLinux / Ubuntu Servers |
| **Applies To** | Linux Sysadmins, DevOps Engineers, DBAs |

---

## Background: How Linux Storage Works

Before touching any command, understand the storage stack:

```
Disk → Partition → Filesystem → Mount Point → Files
```

A raw disk (e.g. `/dev/sda`) cannot store files directly. It must be:
1. **Partitioned** — divided into usable sections
2. **Formatted** — given a filesystem (ext4, xfs)
3. **Mounted** — attached to a directory so the OS can access it

---

## Core Concepts

### What is a Partition?
A disk divided into isolated sections. Each partition acts like its own mini-disk.
Partition info is stored in either **MBR** (old, max 2TB, 4 partitions) or **GPT** (modern, large disks, 128 partitions). Use GPT for all modern systems.

### What is a Filesystem?
The structure that organises files inside a partition. Without it, the partition is just raw empty space.

| Filesystem | Use Case |
|------------|----------|
| `ext4` | General purpose, widely supported |
| `xfs` | High performance, default on RHEL/AlmaLinux |
| `btrfs` | Advanced features, snapshots |

### What is Mounting?
Attaching a formatted partition to a directory so files can be read/written through it.

```
/dev/sda1  →  mounted at  →  /mnt/data
```

---

## What is LVM and Why Do We Need It?

**LVM (Logical Volume Manager)** solves a fundamental problem with traditional storage:

> **Problem:** You have `/` at 10GB and `/home` at 5GB. `/` fills up but `/home` still has space. With traditional partitions, you **cannot** move that space without backup → repartition → restore.

**LVM creates a flexible storage pool** that you can resize freely without downtime.

### LVM Storage Stack

```
Physical Disk
     ↓
Physical Volume (PV)   ← raw disk or partition registered with LVM
     ↓
Volume Group (VG)      ← storage pool made from one or more PVs
     ↓
Logical Volume (LV)    ← virtual partitions carved from the VG
     ↓
Filesystem + Mount
```

### LVM vs Traditional Storage

| Feature | Without LVM | With LVM |
|---------|-------------|----------|
| Resize volumes | ❌ Painful | ✅ Online, live |
| Combine multiple disks | ❌ | ✅ |
| Snapshots | ❌ | ✅ |
| Setup complexity | Simple | More steps |

---

## Lab Environment

```
vda (20GB) → OS disk
 ├─ vda1         BIOS boot
 ├─ vda2         /boot
 └─ vda3         LVM PV
       ├─ root   LV (filesystem /)
       └─ swap   LV

sda (10GB) → New empty practice disk
```

---

## Part 1 — Extend Storage Using LVM

**Goal:** Add `/dev/sda` into the existing LVM pool and expand the root filesystem.

---

### Step 1 — Verify Disks

**What:** Confirm which disks are available before making any changes.

```bash
sudo lsblk
```

**Expected output:**
```
sda   10G   disk          ← new disk (no partitions)
vda   20G   disk
├─vda1
├─vda2      /boot
└─vda3
  ├─almalinux-root   /
  └─almalinux-swap
```

> Only work on `/dev/sda`. Never touch `vda` (OS disk).

---

### Step 2 — Partition the New Disk

**What:** Create a partition on the raw disk, typed for LVM use.
**Why:** LVM needs a partition (or whole disk) to register as a Physical Volume.

```bash
sudo fdisk /dev/sda
```

Inside `fdisk`, type these commands in order:

| Input | Action |
|-------|--------|
| `n` | New partition |
| `p` | Primary type |
| `1` | Partition number |
| `ENTER` | Accept default start sector |
| `ENTER` | Accept default end sector (uses full disk) |
| `t` | Change partition type |
| `8e` | Set type to Linux LVM |
| `w` | Write changes and exit |

**Verify:**
```bash
lsblk
```

Expected: `sda └─ sda1` now visible.

---

### Step 3 — Create Physical Volume (PV)

**What:** Register the partition with LVM as a Physical Volume.
**Why:** This is the entry point into the LVM stack. Without this, LVM cannot see the disk.

```bash
sudo pvcreate /dev/sda1
```

**Verify:**
```bash
sudo pvdisplay
```

Expected: `/dev/sda1` listed as a Physical Volume.

---

### Step 4 — Extend the Volume Group (VG)

**What:** Add the new PV into the existing Volume Group to increase the pool size.
**Why:** The VG is the storage pool. Adding a PV makes more space available to allocate.

```bash
sudo vgextend almalinux /dev/sda1
```

**Verify:**
```bash
sudo vgdisplay
```

Check that `VG Size` and `Free PE / Size` have increased.

---

### Step 5 — Extend the Logical Volume (LV)

**What:** Allocate the newly available free space to the root Logical Volume.
**Why:** The VG has free space but it hasn't been assigned to any LV yet.

```bash
sudo lvextend -l +100%FREE /dev/almalinux/root
```

**Flag explained:**

| Flag | Meaning |
|------|---------|
| `-l` | Extend by number of logical extents (blocks) |
| `+100%FREE` | Use 100% of all remaining free space in the VG |

> To extend by a fixed size instead: `lvextend -L +5G /dev/almalinux/root`

---

### Step 6 — Grow the Filesystem

**What:** Tell the filesystem it now has more space to use.
**Why:** The LV is larger but the filesystem inside it still thinks the old size is the limit. This step syncs them.

First, confirm your filesystem type:
```bash
df -Th /
```

**If XFS** (default on AlmaLinux/RHEL):
```bash
sudo xfs_growfs /
```

**If ext4:**
```bash
sudo resize2fs /dev/almalinux/root
```

---

### Step 7 — Verify the Result

```bash
df -h /
```

**Before:** `/ = ~17G`
**After:** `/ = ~27G`

✅ Root filesystem expanded live — no reboot, no data loss.

---

## Part 2 — Mount a New Disk Without LVM (Traditional)

**Goal:** Format `/dev/sda` with ext4 and mount it permanently at `/new-drive/test-drive`.

> **Pre-requisite:** Reset `/dev/sda` to a clean state (remove from LVM if previously used). Easiest in a lab: shut down VM, detach disk, attach a fresh 10GB disk, boot.

---

### Step 1 — Verify Disk

```bash
lsblk
```

Confirm `sda` appears clean with no partitions.

---

### Step 2 — Partition the Disk

```bash
sudo fdisk /dev/sda
```

Inside `fdisk`:

| Input | Action |
|-------|--------|
| `n` | New partition |
| `p` | Primary |
| `1` | Partition number |
| `ENTER` | Default start |
| `ENTER` | Default end |
| `w` | Write and exit |

> No need to change partition type to LVM this time. Default Linux type is fine.

**Verify:**
```bash
lsblk
```

Expected: `sda └─ sda1`

---

### Step 3 — Create Filesystem

**What:** Format the partition so Linux can store files on it.
**Why:** A raw partition has no structure. `mkfs` writes the filesystem metadata that makes it usable.

```bash
sudo mkfs.ext4 /dev/sda1
```

**Command explained:**

| Part | Meaning |
|------|---------|
| `mkfs` | Make filesystem |
| `.ext4` | Filesystem type to use |
| `/dev/sda1` | Target partition |

---

### Step 4 — Create Mount Point

**What:** Create the directory that will act as the access point for the disk.
**Why:** Linux accesses all storage through the directory tree. The disk must be attached to a directory.

```bash
sudo mkdir -p /new-drive/test-drive
```

**Flag explained:**

| Flag | Meaning |
|------|---------|
| `-p` | Create parent directories as needed; no error if they already exist |

---

### Step 5 — Mount the Partition

**What:** Attach the formatted partition to the mount point directory.

```bash
sudo mount /dev/sda1 /new-drive/test-drive
```

**Verify:**
```bash
df -h /new-drive/test-drive
```

Expected:
```
/dev/sda1   10G   0   10G   0%   /new-drive/test-drive
```

> This mount is **temporary** — it will not survive a reboot. Steps 6–8 make it permanent.

---

### Step 6 — Get the Disk UUID

**What:** Retrieve the universally unique identifier for the partition.
**Why:** Device names like `/dev/sda1` can change after a reboot (e.g. become `/dev/sdb1`). UUIDs are stable and permanent, so `/etc/fstab` should always use them.

```bash
sudo blkid /dev/sda1
```

**Example output:**
```
/dev/sda1: UUID="f7a27d32-5bdf-484c-b211-60b3675a59f6" TYPE="ext4"
```

Copy the UUID value.

---

### Step 7 — Add to `/etc/fstab`

**What:** Register the mount in the filesystem table so it auto-mounts on every boot.
**Why:** `/etc/fstab` is read at startup. Any disk listed here is mounted automatically.

```bash
sudo vi /etc/fstab
```

Add this line at the bottom (replace UUID with your actual value):

```
UUID=f7a27d32-5bdf-484c-b211-60b3675a59f6 /new-drive/test-drive ext4 defaults 0 0
```

**fstab field explanation:**

| Field | Value | Meaning |
|-------|-------|---------|
| 1 | `UUID=...` | Device identifier (stable across reboots) |
| 2 | `/new-drive/test-drive` | Mount point directory |
| 3 | `ext4` | Filesystem type |
| 4 | `defaults` | Standard mount options (rw, suid, exec, auto, nouser, async) |
| 5 | `0` | Dump backup — `0` = disabled |
| 6 | `0` | fsck order — `0` = skip filesystem check at boot |

---

### Step 8 — Test fstab Without Rebooting

**What:** Apply all entries in `/etc/fstab` immediately.
**Why:** Catches errors in your fstab entry before a reboot leaves the system unable to mount drives.

```bash
sudo mount -a
```

If no errors appear, the configuration is correct.

**Final verify:**
```bash
df -h /new-drive/test-drive
lsblk
```

✅ Disk is mounted and will persist across reboots.

---

## Quick Reference — Key Commands

```bash
# ── Disk Inspection ─────────────────────────────
lsblk                          # list all block devices and mount points
blkid /dev/sda1                # get UUID and filesystem type
df -h                          # disk usage (human readable)
df -Th                         # disk usage with filesystem type

# ── Partitioning ────────────────────────────────
sudo fdisk /dev/sda            # interactive partition tool

# ── LVM ─────────────────────────────────────────
sudo pvcreate /dev/sda1                        # create Physical Volume
sudo pvdisplay                                 # view PV info
sudo vgextend almalinux /dev/sda1              # add PV to Volume Group
sudo vgdisplay                                 # view VG info
sudo lvextend -l +100%FREE /dev/almalinux/root # extend Logical Volume
sudo xfs_growfs /                              # grow XFS filesystem
sudo resize2fs /dev/almalinux/root             # grow ext4 filesystem

# ── Traditional Mount ────────────────────────────
sudo mkfs.ext4 /dev/sda1                       # format partition
sudo mkdir -p /mountpoint                      # create mount directory
sudo mount /dev/sda1 /mountpoint               # mount partition
sudo mount -a                                  # apply fstab entries
```

---

## Summary

| Method | Steps | Flexible? | Use When |
|--------|-------|-----------|----------|
| **LVM** | PV → VG → LV → FS → Mount | ✅ Yes | Servers, production, needs resizing |
| **Traditional** | Partition → FS → Mount → fstab | ❌ Fixed | Simple extra disks, backups, containers |

---

*SOP Version 1.0 — Linux Storage Management*
