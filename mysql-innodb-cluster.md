# MySQL InnoDB Cluster — Complete SOP
> **Purpose:** Self-contained guide for coworkers. Follow top-to-bottom. No prior MySQL Cluster experience needed.  
> **OS Target:** Rocky Linux (RHEL-based). Ubuntu steps included where commands differ.  
> **Author note:** Read Section 1 first. Then jump to your task in later sections.

---

## Table of Contents

1. [Concepts — What Is All This?](#1-concepts--what-is-all-this)
2. [Architecture Overview](#2-architecture-overview)
3. [Prerequisites & Planning](#3-prerequisites--planning)
4. [Installation](#4-installation)
5. [Cluster Setup (N-Node)](#5-cluster-setup-n-node)
6. [MySQL Router Setup](#6-mysql-router-setup)
7. [Failover Testing](#7-failover-testing)
8. [Dummy Data Load](#8-dummy-data-load)
9. [Backup & Restore](#9-backup--restore)
10. [Day-to-Day Operations Cheatsheet](#10-day-to-day-operations-cheatsheet)
11. [Troubleshooting](#11-troubleshooting)

---

## 1. Concepts — What Is All This?

### 1.1 MySQL
Relational database. Stores data in tables. Speaks SQL. You already know this.

### 1.2 InnoDB (Storage Engine)
Default storage engine inside MySQL. Think of it as the internal "filing cabinet" format MySQL uses to store and retrieve rows. Key properties:
- **ACID compliant** — transactions are safe (Atomic, Consistent, Isolated, Durable)
- **Row-level locking** — two users can write different rows simultaneously without blocking each other
- **Crash recovery** — if server dies mid-write, InnoDB auto-repairs on restart using its redo log
- **Foreign keys** — enforces relational integrity between tables

InnoDB is what makes MySQL reliable enough to cluster.

### 1.3 MySQL InnoDB Cluster
A built-in high-availability solution composed of three pieces that work together:

| Piece | What it does |
|---|---|
| **MySQL Group Replication** | Keeps data synchronized across all nodes in real time |
| **MySQL Shell (AdminAPI)** | CLI tool to create, manage, and monitor the cluster |
| **MySQL Router** | Smart proxy — apps connect to Router, Router forwards to the right node |

Together they give you: **automatic failover**, **read scaling**, and **zero-downtime maintenance**.

### 1.4 Group Replication
Think of it as a "group chat" between MySQL nodes. Every write one node accepts gets broadcast to all others before it is confirmed. If a node disagrees or goes silent, the group votes and continues without it. Rules:
- **Minimum nodes to vote:** you need more than half the nodes alive to form a quorum (majority). Example: 3-node cluster needs 2 alive; 5-node needs 3.
- **Why odd numbers are recommended:** avoids split-brain (tie votes). Even numbers work but add risk.
- **Any number of nodes is supported** — 1 node for dev, 3+ for production. More nodes = more read replicas + better fault tolerance.

### 1.5 Single-Primary vs Multi-Primary Mode
- **Single-Primary (default, recommended):** One node accepts writes (PRIMARY). All others are read-only replicas. If primary dies, cluster elects a new primary automatically.
- **Multi-Primary:** All nodes accept writes. Higher complexity, conflict risk. Use only if you have specific parallel-write needs.

### 1.6 MySQL Router
Sits between your application and the cluster. App connects to Router on a fixed port. Router knows the current primary and routes write traffic there. If cluster fails over, Router detects it and redirects — app sees no change.

### 1.7 MySQL Shell
Not the regular `mysql` CLI. A newer tool (`mysqlsh`) with JavaScript/Python scripting and the `dba` object for cluster operations. You will use it for all cluster admin work.

### 1.8 What "Failover" Means Here
When primary node dies:
1. Remaining nodes detect the silence (within seconds)
2. Group elects new primary (fastest/most up-to-date node wins)
3. Router detects topology change
4. New writes go to new primary
5. Dead node, when restarted, rejoins as replica

This happens **automatically**. No human intervention needed for basic failover.

---

## 2. Architecture Overview

```
                          ┌─────────────┐
                          │ Application │
                          └──────┬──────┘
                                 │ connects to Router port
                          ┌──────▼──────┐
                          │ MySQL Router│  (can run on any node or dedicated host)
                          └──┬───────┬──┘
              write port      │       │  read port
                    ┌─────────▼──┐  ┌─▼──────────┐
                    │  PRIMARY   │  │  SECONDARY │ ...N secondaries
                    │  Node 1    │  │  Node 2    │
                    └─────┬──────┘  └─────┬──────┘
                          │  Group Replication   │
                          └──────────┬───────────┘
                                     │
                               Node 3, 4, 5...
                               (more secondaries)
```

**Key ports used:**

| Port | Purpose |
|---|---|
| 3306 | MySQL client connections |
| 33060 | MySQL X Protocol (used by Shell/AdminAPI) |
| 33061 | Group Replication communication (internal) |
| 6446 | Router write port (forwards to primary) |
| 6447 | Router read port (round-robin across secondaries) |

---

## 3. Prerequisites & Planning

### 3.1 Before You Start — Checklist

- [ ] All nodes have static IP addresses (never use DHCP for cluster nodes)
- [ ] All nodes can reach each other on ports 3306, 33060, 33061
- [ ] Hostnames set and resolvable (either via DNS or `/etc/hosts`)
- [ ] Time is synchronized across all nodes (NTP/Chrony) — **critical**
- [ ] SELinux / AppArmor policy planned (or disabled for initial setup)
- [ ] Firewall rules planned
- [ ] You know which node will be the seed (first primary) — call it Node 1

### 3.2 Hostname & /etc/hosts Setup (all nodes)

Set unique hostname on each node:
```bash
hostnamectl set-hostname mysql-node1   # repeat for node2, node3...
```

Add all nodes to `/etc/hosts` on **every** node:
```
192.168.1.101  mysql-node1
192.168.1.102  mysql-node2
192.168.1.103  mysql-node3
# add more lines for more nodes
```

### 3.3 Time Sync (all nodes)

```bash
# RHEL/Rocky
systemctl enable --now chronyd
chronyc tracking

# Ubuntu
systemctl enable --now systemd-timesyncd
timedatectl status
```

Time drift > 5 seconds can break Group Replication. Verify before continuing.

### 3.4 Firewall Rules (all nodes)

**RHEL/Rocky (firewalld):**
```bash
firewall-cmd --permanent --add-port=3306/tcp
firewall-cmd --permanent --add-port=33060/tcp
firewall-cmd --permanent --add-port=33061/tcp
firewall-cmd --reload
```

**Ubuntu (ufw):**
```bash
ufw allow 3306/tcp
ufw allow 33060/tcp
ufw allow 33061/tcp
ufw reload
```

> **Note:** If using MySQL Router on a dedicated host or on Node 1, also open 6446 and 6447 on that host.

### 3.5 SELinux (RHEL/Rocky)

For initial setup you may set to permissive, then harden after:
```bash
setenforce 0
# To make permanent (not recommended for production long-term):
sed -i 's/^ENFORCING/PERMISSIVE/' /etc/selinux/config
```

To keep enforcing and allow MySQL ports:
```bash
semanage port -a -t mysqld_port_t -p tcp 33060
semanage port -a -t mysqld_port_t -p tcp 33061
```

---

## 4. Installation

> **Perform on ALL nodes unless stated otherwise.**

### 4.1 Option A — RHEL-Based (Rocky Linux, CentOS Stream, RHEL, AlmaLinux)

#### Add MySQL Official Repository
```bash
# Download and install MySQL repo RPM (check https://dev.mysql.com/downloads/repo/yum/ for latest)
dnf install -y https://dev.mysql.com/get/mysql84-community-release-el9-1.noarch.rpm

# Verify repo added
dnf repolist | grep mysql
```

#### Install MySQL Server + Shell
```bash
dnf install -y mysql-community-server mysql-shell
```

#### Enable & Start MySQL
```bash
systemctl enable --now mysqld
```

#### Get Temporary Root Password
```bash
grep 'temporary password' /var/log/mysqld.log
```

#### Secure Installation
```bash
mysql_secure_installation
# Follow prompts: change root password, remove anon users, disallow remote root, remove test db
```

#### Install MySQL Router (on router host or Node 1)
```bash
dnf install -y mysql-router-community
```

---

### 4.2 Option B — Ubuntu/Debian-Based

#### Add MySQL Official Repository
```bash
# Download apt config package
wget https://dev.mysql.com/get/mysql-apt-config_0.8.29-1_all.deb
dpkg -i mysql-apt-config_0.8.29-1_all.deb
# Select MySQL 8.x in the dialog, then OK

apt update
```

#### Install MySQL Server + Shell
```bash
apt install -y mysql-server mysql-shell
```

#### Enable & Start MySQL
```bash
systemctl enable --now mysql
```

#### Get Temporary Root Password
```bash
# Ubuntu sets root with auth_socket by default — run:
mysql -u root
# Then set password manually inside MySQL:
# ALTER USER 'root'@'localhost' IDENTIFIED BY 'YourStrongPassword!';
```

#### Secure Installation
```bash
mysql_secure_installation
```

#### Install MySQL Router
```bash
apt install -y mysql-router
```

---

### 4.3 Verify Installation (all nodes, both OS)
```bash
mysqladmin -u root -p version
mysqlsh --version
```

---

## 5. Cluster Setup (N-Node)

> Concepts: Node 1 = seed (first primary). All other nodes join after.

### 5.1 Configure MySQL for Group Replication (all nodes)

Edit `/etc/my.cnf` (RHEL) or `/etc/mysql/mysql.conf.d/mysqld.cnf` (Ubuntu):

```ini
[mysqld]
# --- Basic ---
server_id = 1                        # UNIQUE per node: 1, 2, 3, 4...
bind-address = 0.0.0.0
report_host = mysql-node1            # THIS node's hostname — change per node

# --- Binary Log (required for replication) ---
log_bin = binlog
binlog_format = ROW
gtid_mode = ON
enforce_gtid_consistency = ON

# --- Group Replication Plugin ---
plugin_load_add = group_replication.so
group_replication_group_name = "aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa"   # same UUID on all nodes
group_replication_start_on_boot = OFF                                    # AdminAPI controls this
group_replication_local_address = "mysql-node1:33061"                   # THIS node's address:port
group_replication_group_seeds = "mysql-node1:33061,mysql-node2:33061,mysql-node3:33061"
group_replication_bootstrap_group = OFF                                  # AdminAPI controls this

# --- InnoDB settings (performance, optional but recommended) ---
innodb_buffer_pool_size = 1G         # ~70% of RAM on dedicated DB servers
innodb_flush_log_at_trx_commit = 1
sync_binlog = 1
```

> **Critical:** `server_id` and `report_host` and `group_replication_local_address` must be different per node. Everything else is the same.

Restart MySQL after editing:
```bash
systemctl restart mysqld   # RHEL
systemctl restart mysql    # Ubuntu
```

### 5.2 Create Cluster Admin User (all nodes)

Connect as root on **each node**:
```bash
mysql -u root -p
```
```sql
-- Create replication/admin user (same on all nodes)
CREATE USER 'clusteradmin'@'%' IDENTIFIED BY 'ClusterAdminPass!123';
GRANT ALL PRIVILEGES ON *.* TO 'clusteradmin'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
EXIT;
```

### 5.3 Run Pre-Check (on Node 1 via MySQL Shell)

```bash
mysqlsh
```
Inside Shell:
```javascript
// Connect to Node 1
\connect clusteradmin@mysql-node1:3306

//Go into JS mode
\js

// Check if this instance is ready to be clustered
dba.checkInstanceConfiguration('clusteradmin@mysql-node1:3306')

// Auto-fix configuration issues if any found
dba.configureInstance('clusteradmin@mysql-node1:3306')
// Follow prompts, restart MySQL if asked
```

Repeat `dba.checkInstanceConfiguration` for **every other node** from the same shell session:
```javascript
dba.checkInstanceConfiguration('clusteradmin@mysql-node2:3306')
dba.configureInstance('clusteradmin@mysql-node2:3306')

dba.checkInstanceConfiguration('clusteradmin@mysql-node3:3306')
dba.configureInstance('clusteradmin@mysql-node3:3306')
// ... repeat for all nodes
```

### 5.4 Create the Cluster (on Node 1)

Still in MySQL Shell:
```javascript
// Make sure you're connected to Node 1
\connect clusteradmin@mysql-node1:3306

//Go into JS mode
\js

// Create cluster — Node 1 becomes first primary
var cluster = dba.createCluster('myCluster')
```

You should see: `Cluster successfully created`.

### 5.5 Add Remaining Nodes

```javascript
// Add Node 2
cluster.addInstance('clusteradmin@mysql-node2:3306')
// When asked about recovery method: choose 'Clone' for fresh nodes

// Add Node 3
cluster.addInstance('clusteradmin@mysql-node3:3306')

// Add Node 4, 5... same pattern
cluster.addInstance('clusteradmin@mysql-node4:3306')
```

> **Clone method** copies data from primary to new node automatically. Use this for all new nodes.

### 5.6 Verify Cluster Status

```javascript
cluster.status()
```

Expected output summary:
```
"status": "OK",
"primary": "mysql-node1:3306",
"members": {
  "mysql-node1:3306": { "status": "ONLINE", "role": "PRIMARY" },
  "mysql-node2:3306": { "status": "ONLINE", "role": "SECONDARY" },
  "mysql-node3:3306": { "status": "ONLINE", "role": "SECONDARY" }
}
```

All members must show `ONLINE`. If any show `RECOVERING`, wait — it's still syncing.

---

## 6. MySQL Router Setup

> Router runs on one host (can be Node 1, a dedicated host, or the app server). Bootstraps itself from the cluster metadata.

### 6.1 Bootstrap Router

```bash
# Run as root or mysqlrouter user
mysqlrouter --bootstrap clusteradmin@mysql-node1:3306 \
            --user=mysqlrouter \
            --directory=/etc/mysqlrouter
# Enter clusteradmin password when asked
```

Bootstrap reads cluster topology and writes router config automatically.

### 6.2 Start Router

```bash
systemctl enable --now mysqlrouter
# OR if installed manually:
mysqlrouter &
```

### 6.3 Verify Router

```bash
# Check Router is listening
ss -tlnp | grep 6446
ss -tlnp | grep 6447

# Test write connection through Router
mysql -u clusteradmin -p -h 127.0.0.1 -P 6446 -e "SELECT @@hostname, @@read_only;"

# Test read connection through Router
mysql -u clusteradmin -p -h 127.0.0.1 -P 6447 -e "SELECT @@hostname, @@read_only;"
```

Write port should show primary hostname, `read_only = 0`.  
Read port should show a secondary hostname, `read_only = 1`.

---

## 7. Failover Testing

> Goal: Confirm cluster auto-recovers when primary dies. Do this in a maintenance window before production.

### 7.1 Identify Current Primary

```bash
mysqlsh --uri clusteradmin@mysql-node1:3306 -- cluster status
# OR in Shell:
# var cluster = dba.getCluster(); cluster.status()
```

Note which node is PRIMARY. Call it NodeX.

### 7.2 Kill the Primary (Simulated Crash)

On NodeX:
```bash
systemctl stop mysqld   # RHEL
# OR for hard crash simulation:
systemctl kill -s 9 mysqld
```

### 7.3 Observe Failover (on a surviving node)

```bash
mysqlsh --uri clusteradmin@mysql-node2:3306
```
```javascript
var cluster = dba.getCluster()
cluster.status()
```

Within ~10–30 seconds you should see:
- A different node promoted to PRIMARY
- NodeX shows `UNREACHABLE` or `MISSING`
- Cluster status still `OK` (if quorum maintained)

### 7.4 Verify Writes Still Work (through Router)

```bash
mysql -u clusteradmin -p -h 127.0.0.1 -P 6446 -e "SELECT @@hostname;"
```

Should return the **new** primary hostname — Router has automatically switched.

### 7.5 Rejoin the Failed Node

Restart MySQL on NodeX:
```bash
systemctl start mysqld
```

In MySQL Shell (from any live node):
```javascript
var cluster = dba.getCluster()
cluster.rejoinInstance('clusteradmin@mysql-nodeX:3306')
cluster.status()
```

NodeX should return as SECONDARY `ONLINE`.

### 7.6 Failover Test — Pass Criteria

| Check | Expected |
|---|---|
| Cluster elects new primary | < 30 seconds |
| Router redirects writes | Automatic, app transparent |
| Old primary rejoins as secondary | After restart + rejoin command |
| Final cluster.status() | `OK`, all members `ONLINE` |

---

## 8. Dummy Data Load

> Used for testing replication, backup, restore verification.

### 8.1 Using MySQL's Built-in world_x Database

```bash
# Download sample database (run on primary node)
wget https://downloads.mysql.com/docs/world_x-db.tar.gz
tar xzf world_x-db.tar.gz
cd world_x-db

# Load into cluster via primary
mysql -u clusteradmin -p -h 127.0.0.1 -P 6446 < world_x.sql
```

### 8.2 Create Custom Test Data (quick method)

Connect to primary via Router:
```bash
mysql -u clusteradmin -p -h 127.0.0.1 -P 6446
```
```sql
CREATE DATABASE testdb;
USE testdb;

CREATE TABLE employees (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    department VARCHAR(50),
    salary DECIMAL(10,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert 1000 rows
DELIMITER $$
CREATE PROCEDURE load_test_data()
BEGIN
  DECLARE i INT DEFAULT 1;
  WHILE i <= 1000 DO
    INSERT INTO employees (name, department, salary)
    VALUES (
      CONCAT('Employee_', i),
      ELT(1 + FLOOR(RAND() * 5), 'HR', 'IT', 'Finance', 'Ops', 'Sales'),
      ROUND(30000 + RAND() * 70000, 2)
    );
    SET i = i + 1;
  END WHILE;
END$$
DELIMITER ;

CALL load_test_data();
SELECT COUNT(*) FROM employees;
```

### 8.3 Verify Replication of Data

Connect to a secondary directly (not through Router):
```bash
mysql -u clusteradmin -p -h mysql-node2 -P 3306
```
```sql
USE world_x;
SELECT COUNT(*) FROM city;   -- must match primary count
```

---

## 9. Backup & Restore

### 9.1 Backup Strategy

| Method | Best For | Notes |
|---|---|---|
| `mysqlpump` / `mysqldump` | Logical backup, small-medium DBs | Slow on large DBs, portable SQL |
| MySQL Shell `util.dumpInstance()` | Fast logical backup | Parallel, compressed, recommended |
| MySQL Enterprise Backup / `xtrabackup` | Physical backup, large DBs | Hot backup, fast restore |

This SOP covers **MySQL Shell dump** (recommended) and **mysqldump** (fallback).

---

### 9.2 Backup Using MySQL Shell Dump (Recommended)

#### Take Backup
```bash
mysqlsh --uri clusteradmin@mysql-node1:3306
```
```javascript
// Dump entire instance to directory
util.dumpInstance('/backup/mysql_dump_' + new Date().toISOString().slice(0,10), {
  threads: 4,
  compression: 'zstd'
})
```

Or non-interactive one-liner:
```bash
mysqlsh clusteradmin@mysql-node1:3306 -- util dumpInstance \
  /backup/mysql_full_$(date +%F) \
  --threads=4 --compression=zstd
```

#### Verify Backup
```bash
ls -lh /backup/mysql_full_$(date +%F)/
# Should contain .json metadata files and .zst data files
```

#### Restore from Shell Dump

> **Warning:** Restore wipes existing data in target. Always restore to a test instance first to verify backup integrity before using in disaster recovery.

```bash
mysqlsh clusteradmin@target-host:3306
```
```javascript
util.loadDump('/backup/mysql_full_2024-01-01', {
  threads: 4,
  resetProgress: true
})
```

---

### 9.3 Backup Using mysqldump (Classic/Fallback)

#### Full Backup
```bash
mysqldump \
  -u clusteradmin -p \
  -h 127.0.0.1 -P 6446 \
  --all-databases \
  --single-transaction \
  --routines \
  --events \
  --triggers \
  --set-gtid-purged=OFF \
  --master-data=2 \
  | gzip > /backup/full_backup_$(date +%F_%H%M).sql.gz

echo "Exit code: $?"   # 0 = success
```

#### Single Database Backup
```bash
mysqldump \
  -u clusteradmin -p \
  -h 127.0.0.1 -P 6446 \
  --databases testdb \
  --single-transaction \
  --set-gtid-purged=OFF \
  | gzip > /backup/testdb_$(date +%F_%H%M).sql.gz
```

#### Restore from mysqldump

> **Warning:** This replaces existing data. Confirm you are restoring to the right host.

```bash
# Restore to cluster via primary (Router write port)
zcat /backup/full_backup_2024-01-01_0200.sql.gz | \
  mysql -u clusteradmin -p -h 127.0.0.1 -P 6446

echo "Exit code: $?"   # 0 = success
```

### 9.4 Automate Backups (Cron)

```bash
crontab -e
# Add: daily backup at 2am
0 2 * * * /usr/bin/mysqlsh clusteradmin@mysql-node1:3306 \
  -- util dumpInstance /backup/mysql_$(date +\%F) \
  --threads=4 --compression=zstd >> /var/log/mysql_backup.log 2>&1
```

Retain 7 days:
```bash
# Add after backup line:
0 3 * * * find /backup/mysql_* -maxdepth 0 -mtime +7 -exec rm -rf {} \;
```

### 9.5 Backup Verification Checklist

- [ ] Backup directory not empty, files > 0 bytes
- [ ] No errors in backup log
- [ ] Test restore to a non-production instance monthly
- [ ] Confirm row counts match post-restore

---

## 10. Day-to-Day Operations Cheatsheet

### Check Cluster Health
```bash
mysqlsh clusteradmin@mysql-node1:3306 -- cluster status
```

### Get Cluster Object (in Shell)
```javascript
var cluster = dba.getCluster()
cluster.status()
cluster.describe()
```

### Add a New Node
```javascript
cluster.addInstance('clusteradmin@new-node:3306')
```

### Remove a Node Gracefully
```javascript
cluster.removeInstance('clusteradmin@mysql-node3:3306')
```

### Force Remove an Unreachable Node
```javascript
cluster.removeInstance('clusteradmin@mysql-node3:3306', {force: true})
```

### Manual Switchover (planned maintenance — make different node primary)
```javascript
cluster.setPrimaryInstance('clusteradmin@mysql-node2:3306')
```

### Dissolve Cluster (emergency / rebuild)
```javascript
// WARNING: Destroys cluster metadata, stops group replication
cluster.dissolve({force: true})
```

### Rejoin Node After Crash
```javascript
cluster.rejoinInstance('clusteradmin@mysql-node1:3306')
```

### Reboot Cluster from Complete Outage (all nodes were down)
```bash
# On the node with most recent GTID (usually last known primary):
mysqlsh clusteradmin@mysql-node1:3306
```
```javascript
dba.rebootClusterFromCompleteOutage()
```

---

## 11. Troubleshooting

### Node shows RECOVERING for long time
- Check network between nodes: `ping mysql-node2`
- Check port 33061 open: `nc -zv mysql-node2 33061`
- Check MySQL error log: `journalctl -u mysqld -n 100`
- If clone is stuck: `cluster.removeInstance(..., {force:true})` then re-add

### Cluster shows NO_QUORUM
- Means too many nodes down (minority remaining)
- Force quorum from a surviving node:
```javascript
cluster.forceQuorumUsingPartitionOf('clusteradmin@surviving-node:3306')
```
> **Warning:** Use only when you are certain the other nodes are truly dead. Causes split-brain if other partition is still running.

### Group Replication won't start — GTID mismatch
```sql
RESET MASTER;  -- clears local GTIDs on the problematic node
-- Then rejoin via Shell
```

### Router not connecting after failover
```bash
systemctl restart mysqlrouter
# Router re-reads cluster topology on start
```

### "Transaction size limit exceeded" error
```sql
-- Increase on primary (temporary):
SET GLOBAL group_replication_transaction_size_limit = 150000000;
-- Add to my.cnf for persistence
```

### Check replication lag
```javascript
cluster.status({extended: 1})
// Look for "applierQueuedTransactionSet" per member
```

---

*End of SOP. For questions or updates, contact the original author.*