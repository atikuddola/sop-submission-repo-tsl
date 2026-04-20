# pgBackRest Integration with Patroni HA Cluster
## Setup & Operations SOP

> **Version:** 1.0 | **Date:** March 2026
> **Stack:** PostgreSQL 16 · Patroni · pgBackRest
> **OS:** RHEL 9 / Rocky Linux 9

---

## Table of Contents

- [1. What is pgBackRest & Why You Need It](#1-what-is-pgbackrest--why-you-need-it)
- [2. How It Fits Into the Existing Setup](#2-how-it-fits-into-the-existing-setup)
- [3. Phase 1 — Prerequisites](#3-phase-1--prerequisites)
  - [3.1 Install pgBackRest](#31-install-pgbackrest)
  - [3.2 Create Repository Directory](#32-create-repository-directory)
  - [3.3 Set Up postgres User SSH](#33-set-up-postgres-user-ssh)
- [4. Phase 2 — Configuration](#4-phase-2--configuration)
  - [4.1 Configure pgBackRest](#41-configure-pgbackrest)
  - [4.2 Enable WAL Archiving via Patroni](#42-enable-wal-archiving-via-patroni)
- [5. Phase 3 — Initialize & First Backup](#5-phase-3--initialize--first-backup)
  - [5.1 Create the Stanza](#51-create-the-stanza)
  - [5.2 Verify Everything is Wired Up](#52-verify-everything-is-wired-up)
  - [5.3 Take the First Full Backup](#53-take-the-first-full-backup)
- [6. Phase 4 — Schedule Automatic Backups](#6-phase-4--schedule-automatic-backups)
- [7. Restore Operations](#7-restore-operations)
- [8. Quick Reference](#8-quick-reference)

---

## 1. What is pgBackRest & Why You Need It

pgBackRest is a dedicated PostgreSQL backup and restore tool. It replaces basic `pg_dump` for production use.

### pg_dump vs pgBackRest

| | pg_dump | pgBackRest |
|---|---|---|
| Type | Logical dump (SQL) | Physical backup (file-level) |
| Speed on large DBs | Slow | Fast |
| Point-in-time recovery | ❌ No | ✅ Yes |
| Incremental backups | ❌ No | ✅ Yes |
| WAL archiving | ❌ No | ✅ Yes |
| Works with Patroni | Manual | Native integration |

### Why streaming replication alone is not enough

Streaming replication is **HA, not backup**. It replicates everything — including accidental `DROP DATABASE`, data corruption, and bad migrations — to all replicas instantly. If data is destroyed on the primary, all replicas reflect that destruction within seconds.

pgBackRest gives you:
- **Full backups** — complete snapshot of the database cluster
- **Incremental backups** — only changed blocks since last backup
- **WAL archiving** — continuous shipping of WAL files to the repository
- **Point-in-time recovery (PITR)** — restore to any exact second in the past
- **Replica-based backups** — backup runs from a replica, zero load on the primary

---

## 2. How It Fits Into the Existing Setup

```
                    pgBackRest Repository
                    /var/lib/pgbackrest
                    (stored on pgproxy)
                           │
              ┌────────────┼────────────┐
              │            │            │
           pgnode1      pgnode2      pgnode3
           (primary)   (replica)   (replica)
                            │
                    backup runs from
                    replica — no load
                    on primary
```

### Updated Node Roles

| Hostname | IP              | Role              | Components                                      |
|----------|-----------------|-------------------|-------------------------------------------------|
| pgnode1  | 192.168.122.98  | Database          | etcd, Patroni, PostgreSQL, pgBackRest client    |
| pgnode2  | 192.168.122.147 | Database          | etcd, Patroni, PostgreSQL, pgBackRest client    |
| pgnode3  | 192.168.122.105 | Database          | etcd, Patroni, PostgreSQL, pgBackRest client    |
| pgproxy  | 192.168.122.181 | Proxy + Backup    | HAProxy, Keepalived, pgBackRest repository      |

### Updated Port Map Addition

| Port | Service | Purpose |
|------|---------|---------|
| 22   | SSH     | pgBackRest file transfer between nodes and repository |

---

## 3. Phase 1 — Prerequisites

### 3.1 Install pgBackRest

**On ALL DB nodes (pgnode1, pgnode2, pgnode3) and pgproxy:**

```bash
sudo dnf install -y pgbackrest
```

Verify installation:

```bash
pgbackrest version
```

---

### 3.2 Create Repository Directory

**On pgproxy only:**

```bash
sudo mkdir -p /var/lib/pgbackrest
sudo chmod 750 /var/lib/pgbackrest
sudo chown postgres:postgres /var/lib/pgbackrest
```

**Create log directory on ALL DB nodes and pgproxy:**

```bash
sudo mkdir -p /var/log/pgbackrest
sudo chown postgres:postgres /var/log/pgbackrest
sudo chmod 750 /var/log/pgbackrest
```

---

### 3.3 Set Up postgres User SSH

pgBackRest transfers files over SSH as the `postgres` user. The `postgres` user exists on all nodes (created automatically by the PostgreSQL package) but needs SSH keys set up.

#### Verify postgres user exists

```bash
getent passwd postgres
# Expected: postgres:x:26:26:PostgreSQL Server:/var/lib/pgsql:/bin/bash
```

#### Set a password for postgres (ALL DB nodes + pgproxy)

```bash
sudo passwd postgres
```

#### Create .ssh directory (ALL DB nodes + pgproxy)

```bash
sudo mkdir -p /var/lib/pgsql/.ssh
sudo chown postgres:postgres /var/lib/pgsql/.ssh
sudo chmod 700 /var/lib/pgsql/.ssh
```

#### Generate SSH key as postgres (ALL DB nodes + pgproxy)

Run this on pgnode1, pgnode2, pgnode3, and pgproxy — one by one:

```bash
sudo -u postgres ssh-keygen -t rsa -b 4096 -N "" -f /var/lib/pgsql/.ssh/id_rsa
```

#### Copy DB node keys to pgproxy

```bash
# On pgnode1:
sudo -u postgres ssh-copy-id postgres@192.168.122.181

# On pgnode2:
sudo -u postgres ssh-copy-id postgres@192.168.122.181

# On pgnode3:
sudo -u postgres ssh-copy-id postgres@192.168.122.181
```

#### Copy pgproxy key to all DB nodes

```bash
# On pgproxy:
sudo -u postgres ssh-copy-id postgres@192.168.122.98
sudo -u postgres ssh-copy-id postgres@192.168.122.147
sudo -u postgres ssh-copy-id postgres@192.168.122.105
```

#### Test all SSH connections

```bash
# From pgproxy — must return hostname with no password prompt:
sudo -u postgres ssh postgres@192.168.122.98  "hostname"
sudo -u postgres ssh postgres@192.168.122.147 "hostname"
sudo -u postgres ssh postgres@192.168.122.105 "hostname"

# From pgnode1:
sudo -u postgres ssh postgres@192.168.122.181 "hostname"
```

> If any connection asks for a password, SSH is not set up correctly. Do not proceed until all connections work passwordlessly.

---

## 4. Phase 2 — Configuration

### 4.1 Configure pgBackRest

#### On ALL DB nodes (pgnode1, pgnode2, pgnode3) — same config on all three

```bash
sudo mkdir -p /etc/pgbackrest
sudo nano /etc/pgbackrest/pgbackrest.conf
```

```ini
[global]
repo1-host=192.168.122.181
repo1-host-user=postgres
repo1-path=/var/lib/pgbackrest
repo1-retention-full=2
repo1-retention-diff=7
log-level-console=info
log-level-file=detail
start-fast=y
delta=y

[postgres]
pg1-path=/data/pgsql/16/data
pg1-port=5432
```

#### On pgproxy ONLY — different config (pgproxy IS the repo host)

```bash
sudo mkdir -p /etc/pgbackrest
sudo nano /etc/pgbackrest/pgbackrest.conf
```

```ini
[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=2
repo1-retention-diff=7
log-level-console=info
log-level-file=detail
start-fast=y
delta=y

[postgres]
pg1-path=/data/pgsql/16/data
pg1-host=192.168.122.98
pg1-host-user=postgres
pg1-port=5432

pg2-path=/data/pgsql/16/data
pg2-host=192.168.122.147
pg2-host-user=postgres
pg2-port=5432

pg3-path=/data/pgsql/16/data
pg3-host=192.168.122.105
pg3-host-user=postgres
pg3-port=5432
```

---

### 4.2 Enable WAL Archiving via Patroni

WAL archiving must be enabled through `patronictl edit-config` so it applies to the live cluster. Editing the local `patroni.yml` file alone is not enough.

#### Edit the live cluster config (run once from any DB node)

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml edit-config postgres
```

Add the `parameters:` block inside the `postgresql:` section. Your full config should look like this after editing:

```yaml
loop_wait: 10
maximum_lag_on_failover: 1048576
postgresql:
  pg_hba:
  - host replication replicator 192.168.122.98/32 scram-sha-256
  - host replication replicator 192.168.122.147/32 scram-sha-256
  - host replication replicator 192.168.122.105/32 scram-sha-256
  - host replication replicator 192.168.122.252/32 scram-sha-256
  - host all all 0.0.0.0/0 scram-sha-256
  - local all postgres trust
  
  parameters:
    archive_command: pgbackrest --stanza=postgres archive-push %p
    archive_mode: 'on'
    archive_timeout: 60
    max_wal_senders: 10
    wal_level: replica
  use_pg_rewind: true
retry_timeout: 10
ttl: 30
```

> The `parameters:` block sits between `pg_hba:` and `use_pg_rewind:`. Indentation is critical — 2 spaces for `parameters:`, 4 spaces for each parameter under it.

Save and exit the editor.

#### Verify the config saved

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml show-config postgres
```

Confirm `archive_command`, `archive_mode`, and `wal_level` appear in the output.

#### Reload all nodes to apply config

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml reload postgres pgnode1
sudo -u postgres patronictl -c /etc/patroni/patroni.yml reload postgres pgnode2
sudo -u postgres patronictl -c /etc/patroni/patroni.yml reload postgres pgnode3
```

#### Restart all nodes to activate archive_mode

`archive_mode` requires a full PostgreSQL restart — reload alone is not sufficient. Restart replicas first, leader last to avoid unnecessary failover:

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml restart postgres pgnode2
sudo -u postgres patronictl -c /etc/patroni/patroni.yml restart postgres pgnode3
sudo -u postgres patronictl -c /etc/patroni/patroni.yml restart postgres pgnode1
```

#### Confirm archive_mode is active

```bash
psql -h 192.168.122.112 -p 5000 -U postgres \
  -c "SELECT name, setting FROM pg_settings WHERE name IN ('archive_mode','archive_command','wal_level');"
```

Expected:

```
    name          |                        setting
------------------+------------------------------------------------
 archive_command  | pgbackrest --stanza=postgres archive-push %p
 archive_mode     | on
 wal_level        | replica
```

---

## 5. Phase 3 — Initialize & First Backup

### 5.1 Create the Stanza

A stanza is pgBackRest's configuration unit for a PostgreSQL cluster. Created once.

**Run on pgproxy only:**

```bash
sudo -u postgres pgbackrest --stanza=postgres --log-level-console=info stanza-create
```

Expected output:

```
INFO: stanza-create command begin
INFO: stanza-create for stanza 'postgres' on repo1
INFO: stanza-create command end: completed successfully
```

> If this fails, it is almost always an SSH connectivity issue. Go back to Step 3.3 and re-test all SSH connections.

---

### 5.2 Verify Everything is Wired Up

**Run on pgproxy:**

```bash
sudo -u postgres pgbackrest --stanza=postgres check
```

Expected output:

```
INFO: check command begin
INFO: WAL segment 000000010000000000000001 successfully archived
INFO: check command end: completed successfully
```

This confirms:
- pgBackRest can reach all DB nodes over SSH ✅
- WAL archiving is working — a test WAL file was sent to the repo ✅
- The stanza config is correct ✅

---

### 5.3 Take the First Full Backup

**Run on pgproxy:**

```bash
sudo -u postgres pgbackrest --stanza=postgres --log-level-console=info backup --type=full
```

Check the result:

```bash
sudo -u postgres pgbackrest --stanza=postgres info
```

Expected output:

```
stanza: postgres
    status: ok
    cipher: none

    db (current)
        wal archive min/max (16): 000000010000000000000001/000000010000000000000005

        full backup: 20260329-120000F
            timestamp start/stop: 2026-03-29 12:00:00 / 2026-03-29 12:04:30
            database size: 2.4GB, database backup size: 2.4GB
            repo1: backup set size: 800MB, backup size: 800MB
```

---

## 6. Phase 4 — Schedule Automatic Backups

**On pgproxy — edit the postgres user crontab:**

```bash
sudo -u postgres crontab -e
```

Add:

```cron
# Full backup every Sunday at 1:00 AM
0 1 * * 0 pgbackrest --stanza=postgres backup --type=full

# Incremental backup every Mon-Sat at 1:00 AM
0 1 * * 1-6 pgbackrest --stanza=postgres backup --type=incr

# Differential backup at 7:00 AM and 7:00 PM daily
0 7,19 * * * pgbackrest --stanza=postgres backup --type=diff
```

### Backup type explanation

| Type | What it backs up | Depends on |
|------|-----------------|------------|
| `full` | Everything | Nothing — self-contained |
| `diff` | Changes since last full | Last full backup |
| `incr` | Changes since last backup of any type | Last full, diff, or incr |

---

## 7. Restore Operations

> **Always stop Patroni on ALL nodes before restoring.** Restoring to a running cluster will cause conflicts.

### Restore to latest backup

```bash
# Step 1 — Stop Patroni on all DB nodes:
sudo systemctl stop patroni   # run on pgnode1, pgnode2, pgnode3

# Step 2 — Clear data dir on the target node (e.g. pgnode1):
sudo rm -rf /data/pgsql/16/data/*

# Step 3 — Restore from pgproxy:
sudo -u postgres pgbackrest --stanza=postgres restore

# Step 4 — Start Patroni on pgnode1 first, then others:
sudo systemctl start patroni
```

### Restore to a specific point in time (PITR)

```bash
# Step 1 — Stop Patroni on all DB nodes:
sudo systemctl stop patroni   # run on pgnode1, pgnode2, pgnode3

# Step 2 — Clear data dir:
sudo rm -rf /data/pgsql/16/data/*

# Step 3 — Restore to a specific timestamp (run on pgproxy):
sudo -u postgres pgbackrest --stanza=postgres restore \
    --type=time \
    --target="2026-03-29 14:30:00" \
    --target-action=promote

# Step 4 — Start Patroni on pgnode1 first:
sudo systemctl start patroni
```

---

## 8. Quick Reference

### Which node runs what

| Task | Run on |
|------|--------|
| Install pgBackRest | All DB nodes + pgproxy |
| Create repo directory | pgproxy only |
| SSH key setup | All DB nodes + pgproxy |
| pgbackrest.conf (client config) | All DB nodes |
| pgbackrest.conf (repo config) | pgproxy only |
| edit-config for WAL archiving | Any one DB node (applies cluster-wide) |
| stanza-create | pgproxy only |
| check | pgproxy only |
| backup | pgproxy only |
| Cron schedule | pgproxy only |
| restore | pgproxy only |

### Key commands

```bash
# Check backup status and WAL archive range
sudo -u postgres pgbackrest --stanza=postgres info

# Verify archiving is working
sudo -u postgres pgbackrest --stanza=postgres check

# Full backup
sudo -u postgres pgbackrest --stanza=postgres backup --type=full

# Incremental backup
sudo -u postgres pgbackrest --stanza=postgres backup --type=incr

# Differential backup
sudo -u postgres pgbackrest --stanza=postgres backup --type=diff

# Restore latest
sudo -u postgres pgbackrest --stanza=postgres restore

# Restore to point in time
sudo -u postgres pgbackrest --stanza=postgres restore \
    --type=time \
    --target="YYYY-MM-DD HH:MM:SS" \
    --target-action=promote

# List backup details as JSON
sudo -u postgres pgbackrest --stanza=postgres info --output=json
```

### Config file locations

| File | Location | Node |
|------|----------|------|
| pgBackRest config | `/etc/pgbackrest/pgbackrest.conf` | All nodes (different content) |
| Backup repository | `/var/lib/pgbackrest/` | pgproxy only |
| pgBackRest logs | `/var/log/pgbackrest/` | All nodes |
| postgres SSH keys | `/var/lib/pgsql/.ssh/` | All nodes |

---

*pgBackRest SOP — PostgreSQL 16 HA Cluster · pgnode1 · pgnode2 · pgnode3 · repo: pgproxy*
