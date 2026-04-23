# Apache Cassandra — Complete SOP
**Target OS:** Rocky Linux 9 (RHEL-based) | 3 VMs  
**Audience:** Coworkers operating without author present  
**Goal:** Production-grade cluster — install, configure, load data, backup, restore, failover

---

## Table of Contents
1. [What Is Cassandra?](#1-what-is-cassandra)
2. [Core Concepts — Learn These First](#2-core-concepts--learn-these-first)
3. [How Sharding Works in Cassandra](#3-how-sharding-works-in-cassandra)
4. [How Replication Works](#4-how-replication-works)
5. [Sharding vs Replication — Which Comes First?](#5-sharding-vs-replication--which-comes-first)
6. [Cluster Architecture for 3 VMs](#6-cluster-architecture-for-3-vms)
7. [Pre-Installation Checklist](#7-pre-installation-checklist)
8. [Installation — RHEL/Rocky Linux](#8-installation--rhelrocky-linux)
9. [Installation — Ubuntu/Debian](#9-installation--ubuntudebian)
10. [Cluster Configuration](#10-cluster-configuration)
11. [Starting & Verifying the Cluster](#11-starting--verifying-the-cluster)
12. [Loading Dummy Data](#12-loading-dummy-data)
13. [Failover Testing](#13-failover-testing)
14. [Backup and Restore](#14-backup-and-restore)
15. [Troubleshooting Reference](#15-troubleshooting-reference)

---

## 1. What Is Cassandra?

Apache Cassandra is a **distributed NoSQL database** built for high availability and massive scale. No single master. No single point of failure. Every node is equal — any node can serve any request.

### Why Use Cassandra?
- Need never-down write performance (IoT, logs, time-series)
- Data too large for one server
- Need auto-recovery if a node dies
- Need data spread across locations (multi-datacenter)

### What Cassandra Is NOT
- Not a relational DB (no JOINs, no foreign keys)
- Not best for analytics on arbitrary columns
- Not ideal for small, simple apps (overhead not worth it)

---

## 2. Core Concepts — Learn These First

### Node
Single Cassandra process running on one machine. Each node owns a portion of the data. Nodes gossip with each other — they constantly share state information so every node knows the health and data ownership of every other node.

### Cluster
Collection of nodes that all belong to the same logical database. All nodes in a cluster share the same cluster name configured in `cassandra.yaml`.

### Datacenter (DC)
Logical grouping of nodes — usually maps to a physical location or rack. You can have multiple DCs in one cluster. Replication can be controlled per-DC.

### Rack
Subdivision within a datacenter. Cassandra tries to place replicas on different racks so a rack failure doesn't lose all copies of data.

### Keyspace
Equivalent to a **database** in SQL. Contains tables. You set replication policy at keyspace level — this is where you say "keep N copies across which DCs."

### Table
Standard rows and columns, but schema is defined up front. Queries must use the **primary key** — Cassandra is designed around known access patterns, not ad-hoc queries.

### Primary Key
Two parts:
- **Partition Key** — determines which node(s) store this row (hashed to find node)
- **Clustering Columns** — sort order within a partition

### Token Ring
Cassandra assigns each node a range of tokens (hash values). When data arrives, its partition key is hashed to a token. That token falls in a node's range — that node is the **coordinator** and primary owner.

### Gossip Protocol
Nodes exchange state (is this node up? what token range does it own?) every second with a few random peers. Within seconds, all nodes know the full cluster state. No central registry needed.

### Coordinator Node
When a client sends a query, it hits any node — that node becomes the coordinator. It routes the query to the correct nodes that own the data. Client doesn't need to know where data lives.

### Consistency Level
Per-query setting. Controls how many replicas must confirm a read or write before success is returned to the client.

| Level | Meaning |
|-------|---------|
| ONE | 1 replica responds — fastest, least safe |
| QUORUM | Majority of replicas respond — balanced |
| ALL | Every replica responds — safest, slowest |
| LOCAL_QUORUM | Quorum within local DC only |

### Compaction
Background process that merges SSTables (on-disk data files) to reclaim space and improve read performance. Cassandra never updates in place — old versions pile up until compaction.

### Tombstone
Deletion marker. Cassandra can't delete in place across distributed nodes. It writes a tombstone — a marker saying "this data is gone." Compaction eventually removes tombstones after the `gc_grace_seconds` window.

---

## 3. How Sharding Works in Cassandra

**Sharding** = splitting data across nodes so no single node holds everything.

Cassandra shards automatically using **consistent hashing**:

1. Partition key of each row gets hashed → produces a **token** (a large integer)
2. The token ring is divided into ranges
3. Each node owns one or more token ranges
4. Data lands on whichever node owns the token for its partition key

### What this means practically
- You don't manually assign data to nodes
- Adding a node → Cassandra redistributes token ranges automatically
- Removing a node → its token range moves to neighbors
- The **partition key design** is critical — bad partition keys cause "hot spots" (one node getting all traffic)

### Hot Spot Warning
If all your queries hit the same partition key value (e.g., `date = '2024-01-01'` for all rows), one node gets hammered. Design partition keys for even distribution.

---

## 4. How Replication Works

**Replication** = storing copies of data on multiple nodes so losing one node doesn't lose data.

### Replication Factor (RF)
Number of copies of each piece of data. `RF=3` means every row exists on 3 different nodes.

Set per keyspace:
```sql
CREATE KEYSPACE my_app
WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'dc1': 3
};
```

### Replication Strategies

**SimpleStrategy** — for single datacenter or testing only. Picks the next N nodes clockwise on the token ring.

**NetworkTopologyStrategy** — for production. Specify RF per datacenter. Cassandra tries to place replicas on different racks within each DC.

### How Replication Happens
1. Client writes row → hits coordinator node
2. Coordinator hashes partition key → finds owner node
3. Coordinator writes to that node AND to RF-1 additional replica nodes
4. Returns success based on consistency level (not waiting for ALL unless you set ALL)

### Hinted Handoff
If a replica node is down during a write, the coordinator saves a "hint" — a local copy of the write. When the dead node comes back, the coordinator replays the hint. This is why brief failures don't lose data.

### Read Repair
During reads, if replicas return different versions of data, Cassandra detects the inconsistency and fixes the stale replica in the background.

---

## 5. Sharding vs Replication — Which Comes First?

**Sharding comes first. Replication is layered on top.**

Here's the sequence:
1. Partition key is hashed → token determined → **primary node found** (sharding)
2. The RF setting then says: "also copy this to N-1 more nodes" (replication)

**Default RF = 1** in Cassandra (with SimpleStrategy). RF=1 means no replication — one copy, one node. If that node dies, data is unavailable.

> ⚠️ **Always set RF ≥ 3 in production.** RF=1 is for local dev only.

Think of it as:
- **Sharding** decides WHERE data primarily lives
- **Replication** decides HOW MANY copies exist

Both happen automatically once configured. You configure RF at keyspace creation. You don't configure sharding manually — it's always on.

---

## 6. Cluster Architecture for 3 VMs

With 3 VMs, the optimal layout achieves **both sharding and replication simultaneously**:

```
┌────────────────────────────────────────────────────────────┐
│                  CLUSTER: prod-cluster                     │
│             Datacenter: dc1  |  Single Rack: rack1        │
│                                                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │   node1      │  │   node2      │  │   node3      │    │
│  │   (seed)     │  │   (seed)     │  │              │    │
│  │              │  │              │  │              │    │
│  │  Token       │  │  Token       │  │  Token       │    │
│  │  Range: A    │  │  Range: B    │  │  Range: C    │    │
│  └──────────────┘  └──────────────┘  └──────────────┘    │
│                                                            │
│  RF = 3  →  EVERY row exists on ALL 3 nodes               │
│  Sharding: token ring splits data across 3 nodes          │
│  Replication: each shard copied to the other 2 nodes      │
└────────────────────────────────────────────────────────────┘
```

### How Sharding + Replication Work Together on 3 Nodes

With RF=3 and 3 nodes, every row is on every node. This might seem redundant — why shard at all? Here's why it still matters:

- **Sharding** divides the token ring — node1 is *primary owner* of range A, node2 of range B, node3 of range C. Writes go to the primary owner first (coordinator routes there).
- **Replication** copies each shard to all other nodes. So range A data lives on node1 (primary) AND node2 AND node3 (replicas).
- Net effect: full dataset on every node. Any node can answer any query without coordination. This is maximum redundancy for a 3-node cluster.
- As you add more nodes (4, 5, 6...) with RF=3, sharding starts splitting data — not every node holds everything. With 3 nodes + RF=3, sharding and replication perfectly overlap.

> **Tolerate:** 1 node failure with full read/write capability (QUORUM = 2 of 3).  
> **Lose 2 nodes:** cluster goes read-only or unavailable depending on consistency level.

### VM Role Assignment

| VM | Hostname | IP (example) | Role | Rack |
|----|----------|--------------|------|------|
| 1 | cass-node1 | 192.168.1.11 | Seed node | rack1 |
| 2 | cass-node2 | 192.168.1.12 | Seed node | rack1 |
| 3 | cass-node3 | 192.168.1.13 | Data node | rack1 |

> All 3 nodes in the same rack — single rack is fine with 3 nodes since RF=3 already puts data on all nodes.  
> Adjust IPs to match your actual network.

### What Are Seed Nodes?
Seeds are the **bootstrap contact points**. New or restarting nodes contact seeds first to learn the cluster topology. Seeds are not special after bootstrap — they have no extra responsibility. With 3 nodes, use 2 as seeds (node1 + node2). Never make ALL nodes seeds — if all seeds are down simultaneously, new nodes can't bootstrap.

---

## 7. Pre-Installation Checklist

Run on **ALL 3 nodes** before installing Cassandra.

### System Requirements

```bash
# Check RAM (minimum 8GB per node recommended for production)
free -h

# Check disk space (SSD preferred, /var/lib/cassandra needs space)
df -h

# Check CPU count
nproc
```

### Set Hostnames

On each node, set its hostname:
```bash
# On node1:
hostnamectl set-hostname cass-node1

# On node2:
hostnamectl set-hostname cass-node2

# On node3:
hostnamectl set-hostname cass-node3
```

### Configure /etc/hosts on ALL Nodes

Add entries for all cluster nodes so hostnames resolve:
```bash
cat >> /etc/hosts << 'EOF'
192.168.1.11  cass-node1
192.168.1.12  cass-node2
192.168.1.13  cass-node3
EOF
```

### Disable Swap (Critical for Cassandra)

Cassandra JVM pauses if OS swaps memory — this causes timeouts and node failures.

```bash
# Disable immediately
swapoff -a

# Disable permanently
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Verify
free -h  # Swap row should show 0
```

### Set vm.max_map_count

```bash
echo "vm.max_map_count=1048575" >> /etc/sysctl.conf
sysctl -p
```

### Configure Firewall

Open required Cassandra ports on ALL nodes:
```bash
# Cassandra ports:
# 7000 - inter-node communication (non-TLS)
# 7001 - inter-node communication (TLS)
# 9042 - native client port (CQL)
# 7199 - JMX monitoring

firewall-cmd --permanent --add-port=7000/tcp
firewall-cmd --permanent --add-port=7001/tcp
firewall-cmd --permanent --add-port=9042/tcp
firewall-cmd --permanent --add-port=7199/tcp
firewall-cmd --reload
firewall-cmd --list-ports  # verify
```

### Install Java (Required by Cassandra)

```bash
# Rocky Linux / RHEL:
dnf install -y java-11-openjdk java-11-openjdk-devel

# Verify
java -version
# Expected: openjdk version "11.x.x"
```

---

## 8. Installation — RHEL/Rocky Linux

Run on **ALL 3 nodes**.

### Add Cassandra Repository

```bash
cat > /etc/yum.repos.d/cassandra.repo << 'EOF'
[cassandra]
name=Apache Cassandra
baseurl=https://redhat.cassandra.apache.org/41x/
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://downloads.apache.org/cassandra/KEYS
EOF
```

> The repo URL `41x` = Cassandra 4.1.x. Replace with `50x` if targeting Cassandra 5.0.

### Install Cassandra

```bash
dnf install -y cassandra

# Verify install
cassandra -version
```


### Create Service

```bash
vi /etc/systemd/system/cassandra.service
```
```
[Unit]
Description=Apache Cassandra
After=network.target

[Service]
Type=forking

User=cassandra
Group=cassandra

ExecStart=/usr/sbin/cassandra -p /var/run/cassandra.pid
ExecStop=/bin/kill -TERM $MAINPID

LimitNOFILE=100000
LimitMEMLOCK=infinity
LimitNPROC=32768

Restart=always

[Install]
WantedBy=multi-user.target
```

### Key File Locations (RHEL/Rocky)

| Item | Path |
|------|------|
| Main config | `/etc/cassandra/conf/cassandra.yaml` |
| Rack/DC config | `/etc/cassandra/conf/cassandra-rackdc.properties` |
| Log file | `/var/log/cassandra/system.log` |
| Data directory | `/var/lib/cassandra/data/` |
| Commit log | `/var/lib/cassandra/commitlog/` |
| JVM settings | `/etc/cassandra/conf/jvm.options` |

---

## 9. Installation — Ubuntu/Debian

(Use this section if your environment has Ubuntu nodes instead of Rocky Linux)

### Add Repository

```bash
# Install prerequisites
apt-get install -y curl gnupg apt-transport-https

# Add Cassandra GPG key
curl https://downloads.apache.org/cassandra/KEYS | gpg --dearmor -o /usr/share/keyrings/cassandra.gpg

# Add repository
echo "deb [signed-by=/usr/share/keyrings/cassandra.gpg] https://debian.cassandra.apache.org 41x main" \
  > /etc/apt/sources.list.d/cassandra.list

# Update and install
apt-get update
apt-get install -y cassandra
```

### Create Service

```bash
vi /etc/systemd/system/cassandra.service
```
```
[Unit]
Description=Apache Cassandra
After=network.target

[Service]
Type=forking

User=cassandra
Group=cassandra

ExecStart=/usr/sbin/cassandra -p /var/run/cassandra.pid
ExecStop=/bin/kill -TERM $MAINPID

LimitNOFILE=100000
LimitMEMLOCK=infinity
LimitNPROC=32768

Restart=always

[Install]
WantedBy=multi-user.target
```

### Key File Locations (Ubuntu/Debian)

| Item | Path |
|------|------|
| Main config | `/etc/cassandra/cassandra.yaml` |
| Rack/DC config | `/etc/cassandra/cassandra-rackdc.properties` |
| Log file | `/var/log/cassandra/system.log` |
| Data directory | `/var/lib/cassandra/data/` |
| Commit log | `/var/lib/cassandra/commitlog/` |
| JVM settings | `/etc/cassandra/jvm.options` |

> All configuration steps below use RHEL paths. Ubuntu paths differ only by directory — substitute accordingly.

---

## 10. Cluster Configuration

This is the most important section. **Wrong config = cluster won't form.**

### Step 1: Edit cassandra.yaml on EVERY Node

Each node gets its own IP in `listen_address` and `rpc_address`. Everything else is the same.

```bash
vi /etc/cassandra/conf/cassandra.yaml
```

Find and set these parameters (leave all others at defaults unless specified):

#### cluster_name
Same value on ALL nodes. Nodes with different cluster names reject each other.
```yaml
cluster_name: 'prod-cluster'
```

#### listen_address
**Set to THIS node's IP.** Different on each node.
```yaml
# On node1:
listen_address: 192.168.1.11
# On node2:
listen_address: 192.168.1.12
# On node3:
listen_address: 192.168.1.13
```

#### rpc_address
IP that clients connect to. Usually same as listen_address.
```yaml
# On node1:
rpc_address: 192.168.1.11
```

#### seeds
Contact points for new/restarting nodes. List your 2 seed nodes here. **Same on all nodes.**
```yaml
seed_provider:
  - class_name: org.apache.cassandra.locator.SimpleSeedProvider
    parameters:
      - seeds: "192.168.1.11,192.168.1.12"
```

#### endpoint_snitch
Tells Cassandra how nodes are organized into DCs and racks. Use `GossipingPropertyFileSnitch` for any multi-node or multi-DC setup.
```yaml
endpoint_snitch: GossipingPropertyFileSnitch
```

#### Data and Log Directories
```yaml
data_file_directories:
  - /var/lib/cassandra/data

commitlog_directory: /var/lib/cassandra/commitlog
```

#### num_tokens
Virtual nodes per physical node. Higher = more even data distribution. Default 16 is fine for most setups.
```yaml
num_tokens: 16
```

### Step 2: Edit cassandra-rackdc.properties on EVERY Node

Each node declares which DC and rack it belongs to.

```bash
vi /etc/cassandra/conf/cassandra-rackdc.properties
```

All 3 nodes go in the same DC and same rack (single-rack setup):

```properties
# Same on ALL 3 nodes:
dc=dc1
rack=rack1
```

> With RF=3 across 3 nodes, every node holds every row regardless of rack assignment. Single rack is acceptable here.

### Step 3: Tune JVM Memory

Edit `/etc/cassandra/conf/jvm-server.options`:
```bash
vi /etc/cassandra/conf/jvm-server.options
```

Set heap size. Rule of thumb: half of system RAM, max 8GB.
```
# If node has 16GB RAM:
-Xms8G
-Xmx8G

# If node has 8GB RAM:
-Xms4G
-Xmx4G
```


### Step 4: Import `cqlsh` if not done

Import cqlsh path 
```bash
export PYTHONPATH=/usr/lib/python3.6/site-packages:$PYTHONPATH
```

Try running cqlsh to check it can be accessed
```
cqlsh <node's ip>
```

---

## 11. Starting & Verifying the Cluster

### Start Order Matters

**Always start seed nodes first.**

```bash
# On node1 (seed):
systemctl start cassandra
sleep 30  # wait for node1 to fully start

# On node2 (seed):
systemctl start cassandra
sleep 30

# On node3:
systemctl start cassandra
```

### Verify Node Status

Run on any node:
```bash
nodetool status
```

**Expected output:**
```
Datacenter: dc1
===============
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address        Load       Tokens  Owns    Host ID   Rack
UN  192.168.1.11   50 KiB     16      33.3%   <uuid>    rack1
UN  192.168.1.12   50 KiB     16      33.3%   <uuid>    rack1
UN  192.168.1.13   50 KiB     16      33.3%   <uuid>    rack1
```

> With RF=3 on 3 nodes, Owns shows 100% per node (every node holds all data). The `33.3%` above reflects token ownership before accounting for RF — after running `nodetool status demo`, each node shows ~100% effective ownership.

| Code | Meaning |
|------|---------|
| UN | Up, Normal — healthy |
| DN | Down, Normal — node unreachable |
| UJ | Up, Joining — node bootstrapping |
| UL | Up, Leaving — node decommissioning |

> If Owns shows `?`, run `nodetool status <keyspace>` after creating a keyspace.

### Check Gossip and Ring

```bash
# Detailed gossip state
nodetool gossipinfo

# Show token ring
nodetool ring

# Check logs if node won't join
tail -f /var/log/cassandra/system.log
```

### Test CQL Connection

```bash
# Open CQL shell (on any node):
cqlsh 192.168.1.11 9042

# Inside cqlsh, verify:
DESCRIBE CLUSTER;
SELECT * FROM system.peers;
```

---

## 12. Loading Dummy Data

### Create Keyspace with RF=3

```sql
-- Connect to CQL shell:
cqlsh 192.168.1.11

-- Create keyspace:
CREATE KEYSPACE IF NOT EXISTS demo
WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'dc1': 3
};

-- Use keyspace:
USE demo;
```

### Create Table

```sql
CREATE TABLE IF NOT EXISTS employees (
  dept     TEXT,
  emp_id   UUID,
  name     TEXT,
  email    TEXT,
  joined   TIMESTAMP,
  salary   DECIMAL,
  PRIMARY KEY ((dept), emp_id)
);
```

> `dept` is the partition key — data groups by department across nodes.
> `emp_id` is the clustering column — rows sorted by emp_id within a partition.

### Insert Dummy Data

```sql
-- Insert sample rows:
INSERT INTO employees (dept, emp_id, name, email, joined, salary)
VALUES ('Engineering', uuid(), 'Alice Rahman', 'alice@company.com', toTimestamp(now()), 95000.00);

INSERT INTO employees (dept, emp_id, name, email, joined, salary)
VALUES ('Engineering', uuid(), 'Bob Hossain', 'bob@company.com', toTimestamp(now()), 88000.00);

INSERT INTO employees (dept, emp_id, name, email, joined, salary)
VALUES ('Marketing', uuid(), 'Carol Ahmed', 'carol@company.com', toTimestamp(now()), 75000.00);

INSERT INTO employees (dept, emp_id, name, email, joined, salary)
VALUES ('Marketing', uuid(), 'David Islam', 'david@company.com', toTimestamp(now()), 72000.00);

INSERT INTO employees (dept, emp_id, name, email, joined, salary)
VALUES ('HR', uuid(), 'Eva Khatun', 'eva@company.com', toTimestamp(now()), 68000.00);
```

### Bulk Load with Python (Optional — for large datasets)

```bash
# Install driver:
pip install cassandra-driver

# Create script:
cat > /tmp/bulk_load.py << 'EOF'
from cassandra.cluster import Cluster
from cassandra.query import SimpleStatement
import uuid
from datetime import datetime
import random

cluster = Cluster(['192.168.1.11', '192.168.1.12'])
session = cluster.connect('demo')

depts = ['Engineering', 'Marketing', 'HR', 'Finance', 'Ops']
names = ['Ahmed', 'Rahman', 'Hossain', 'Islam', 'Khatun', 'Begum', 'Khan']

insert_stmt = session.prepare("""
  INSERT INTO employees (dept, emp_id, name, email, joined, salary)
  VALUES (?, ?, ?, ?, ?, ?)
""")

for i in range(1000):
    dept = random.choice(depts)
    first = f"User{i}"
    last = random.choice(names)
    session.execute(insert_stmt, [
        dept,
        uuid.uuid4(),
        f"{first} {last}",
        f"user{i}@company.com",
        datetime.now(),
        round(random.uniform(50000, 120000), 2)
    ])
    if i % 100 == 0:
        print(f"Inserted {i} rows...")

print("Done. 1000 rows loaded.")
cluster.shutdown()
EOF

python3 /tmp/bulk_load.py
```

### Verify Data

```sql
-- In cqlsh:
SELECT COUNT(*) FROM demo.employees;
SELECT * FROM demo.employees WHERE dept = 'Engineering';

-- Check data distribution across nodes:
-- Exit cqlsh first, then:
```
```bash
nodetool tablestats demo.employees
```

---

## 13. Failover Testing

**Goal:** Prove that cluster survives node loss without data loss.

### Preparation

Open two terminals:
- **Terminal A:** Running continuous queries
- **Terminal B:** Killing nodes

### Terminal A — Start Continuous Reads

```bash
# Create a simple loop that queries every 2 seconds:
cat > /tmp/test_read.py << 'EOF'
from cassandra.cluster import Cluster
import time

# Connect via multiple nodes — driver handles failover automatically
cluster = Cluster(['192.168.1.11', '192.168.1.12', '192.168.1.13'])
session = cluster.connect('demo')

count = 0
while True:
    try:
        result = session.execute("SELECT COUNT(*) FROM employees")
        rows = result.one()[0]
        print(f"[{time.strftime('%H:%M:%S')}] Read OK — {rows} rows")
    except Exception as e:
        print(f"[{time.strftime('%H:%M:%S')}] ERROR: {e}")
    count += 1
    time.sleep(2)
EOF

python3 /tmp/test_read.py
```

### Terminal B — Kill a Node

```bash
# Stop node3 (simulates crash — keep both seeds alive):
ssh root@192.168.1.13
systemctl stop cassandra
exit

# Back on node1, check cluster status:
nodetool status
# node3 should show DN (Down, Normal)
```

**Expected behavior in Terminal A:** Reads continue without error. With RF=3 and 2 nodes still up, QUORUM (2 of 3) is satisfied — cluster stays fully operational.

### Test Write During Failure

```sql
-- In cqlsh (on node1 or node2):
INSERT INTO demo.employees (dept, emp_id, name, email, joined, salary)
VALUES ('Engineering', uuid(), 'Test Failover', 'test@company.com', toTimestamp(now()), 60000.00);

SELECT COUNT(*) FROM demo.employees;
-- Count should include the new row
```

> Cassandra stores a **hint** for node3 — when node3 returns, the coordinator replays any writes it missed.

### Bring Node Back

```bash
# On node3:
ssh root@192.168.1.13
systemctl start cassandra

# Monitor rejoining (watch system.log):
tail -f /var/log/cassandra/system.log
# Look for: "JOINING" → "NORMAL"

# From node1, watch status:
watch -n 5 nodetool status
# node3 should return to UN
```

### Verify Data Consistency After Rejoin

```bash
nodetool repair demo
```

```sql
-- Verify all data present including write made during outage:
SELECT COUNT(*) FROM demo.employees;
SELECT * FROM demo.employees WHERE dept = 'Engineering';
```

### ⚠️ 3-Node Limit — Do NOT Kill 2 Nodes

With 3 nodes and RF=3, QUORUM = 2. Killing 2 nodes simultaneously drops available nodes to 1 — below QUORUM. Writes and reads **fail**.

```bash
# This is a destructive test — read carefully before running.
# Only do this to understand failure behavior, not in production.

# Kill node2 AND node3:
ssh root@192.168.1.12 "systemctl stop cassandra" &
ssh root@192.168.1.13 "systemctl stop cassandra" &
wait

# Check:
nodetool status
# 2 nodes show DN

# In cqlsh on node1:
CONSISTENCY QUORUM;
SELECT COUNT(*) FROM demo.employees;
# FAILS — cannot achieve QUORUM with 1 node

CONSISTENCY ONE;
SELECT COUNT(*) FROM demo.employees;
# SUCCEEDS — but risky, may read stale data
```

**Recovery from 2-node failure:**
```bash
# Restart both nodes:
ssh root@192.168.1.12 "systemctl start cassandra"
ssh root@192.168.1.13 "systemctl start cassandra"

# After all 3 show UN:
nodetool repair demo
```

---

## 14. Backup and Restore

Cassandra backup uses **snapshots** — point-in-time copies of SSTable files on disk.

### Understanding Snapshots

- A snapshot is a hard-linked copy of SSTable files
- No service interruption — snapshot while cluster is running
- Each node creates its own snapshot — must collect from all nodes for full backup
- Snapshots live under: `/var/lib/cassandra/data/<keyspace>/<table>/snapshots/<snapshot_name>/`

### Backup — Single Node Snapshot

```bash
# Create snapshot named "backup-2024-01-15" for keyspace "demo":
nodetool snapshot -t backup-2024-01-15 demo

# Find snapshot files:
find /var/lib/cassandra/data/demo -name "*.db" -path "*/snapshots/backup-2024-01-15/*"

# Snapshot directory structure:
ls /var/lib/cassandra/data/demo/employees-*/snapshots/backup-2024-01-15/
```

### Backup — Full Cluster (Run on ALL Nodes)

Create a script to snapshot and archive on every node:

```bash
cat > /usr/local/bin/cassandra_backup.sh << 'EOF'
#!/bin/bash
# Cassandra Backup Script
# Run on EVERY node

KEYSPACE="${1:-demo}"
DATE=$(date +%Y%m%d-%H%M%S)
SNAPSHOT_NAME="snapshot-${DATE}"
BACKUP_DIR="/backup/cassandra/${DATE}"
HOSTNAME=$(hostname)
LOG="/var/log/cassandra/backup.log"

mkdir -p "${BACKUP_DIR}"
echo "[$(date)] Starting backup on ${HOSTNAME}" >> ${LOG}

# Step 1: Clear old snapshots to reclaim space
nodetool clearsnapshot -t "" ${KEYSPACE}

# Step 2: Create new snapshot
nodetool snapshot -t ${SNAPSHOT_NAME} ${KEYSPACE}
if [ $? -ne 0 ]; then
    echo "[$(date)] ERROR: Snapshot failed on ${HOSTNAME}" >> ${LOG}
    exit 1
fi

# Step 3: Copy snapshot files to backup directory
for TABLE_DIR in /var/lib/cassandra/data/${KEYSPACE}/*/; do
    TABLE=$(basename ${TABLE_DIR})
    SNAP_DIR="${TABLE_DIR}snapshots/${SNAPSHOT_NAME}"
    if [ -d "${SNAP_DIR}" ]; then
        DEST="${BACKUP_DIR}/${TABLE}"
        mkdir -p "${DEST}"
        cp -a "${SNAP_DIR}/." "${DEST}/"
    fi
done

# Step 4: Save schema (important for restore)
cqlsh $(hostname -I | awk '{print $1}') -e "DESCRIBE KEYSPACE ${KEYSPACE};" > "${BACKUP_DIR}/schema.cql"

# Step 5: Compress
tar -czf "${BACKUP_DIR}.tar.gz" -C "$(dirname ${BACKUP_DIR})" "$(basename ${BACKUP_DIR})"
rm -rf "${BACKUP_DIR}"

echo "[$(date)] Backup complete: ${BACKUP_DIR}.tar.gz" >> ${LOG}
echo "${BACKUP_DIR}.tar.gz"
EOF

chmod +x /usr/local/bin/cassandra_backup.sh
```

Run backup on all nodes:
```bash
# Execute on all nodes (from management node or via loop):
for NODE in 192.168.1.11 192.168.1.12 192.168.1.13; do
    ssh root@${NODE} "/usr/local/bin/cassandra_backup.sh demo" &
done
wait
echo "All node backups completed"
```

### Schedule Automatic Backups (Cron)

```bash
# On ALL nodes, run daily at 2 AM:
echo "0 2 * * * root /usr/local/bin/cassandra_backup.sh demo >> /var/log/cassandra/backup.log 2>&1" \
  > /etc/cron.d/cassandra-backup
```

### List and Manage Snapshots

```bash
# List existing snapshots:
nodetool listsnapshots

# Delete specific snapshot (free disk space):
nodetool clearsnapshot -t backup-2024-01-15 demo

# Delete ALL snapshots for a keyspace:
nodetool clearsnapshot demo

# Check snapshot disk usage:
du -sh /var/lib/cassandra/data/demo/*/snapshots/
```

---

### Restore Procedure

> ⚠️ **WARNING: Restore overwrites existing data. Verify backup integrity before proceeding. Stop application traffic before restore.**

#### Scenario A: Restore Single Node (Node Rejoining After Disk Failure)

Fastest path — rebuild from other replicas using streaming.

```bash
# On the recovered/replaced node:

# 1. Stop Cassandra
systemctl stop cassandra

# 2. Clear existing data
rm -rf /var/lib/cassandra/data/*
rm -rf /var/lib/cassandra/commitlog/*
rm -rf /var/lib/cassandra/hints/*

# 3. Start Cassandra fresh — it will stream data from peers automatically
systemctl start cassandra

# 4. Monitor bootstrap (may take minutes to hours depending on data size):
tail -f /var/log/cassandra/system.log | grep -E "JOINING|NORMAL|bootstrap"

# 5. Verify node joined:
nodetool status
# Node should show UN after bootstrap completes
```

#### Scenario B: Restore Full Cluster from Snapshot

Use when cluster is fully lost or data corruption affects all nodes.

```bash
# ---- On ALL nodes ----

# 1. Stop Cassandra
systemctl stop cassandra

# 2. Clear data directories
rm -rf /var/lib/cassandra/data/*
rm -rf /var/lib/cassandra/commitlog/*

# 3. Extract backup archive
BACKUP_DATE="20240115-020000"  # change to your backup date
mkdir -p /restore
tar -xzf /backup/cassandra/${BACKUP_DATE}.tar.gz -C /restore/
```

```bash
# ---- On ONE node (node1 first) ----

# 4. Restore schema first
cqlsh 192.168.1.11 -f /restore/${BACKUP_DATE}/schema.cql

# 5. Copy SSTable files back to data directory
for TABLE in /restore/${BACKUP_DATE}/*/; do
    TABLE_NAME=$(basename ${TABLE})
    # Find matching table directory (includes UUID suffix in path)
    DEST=$(find /var/lib/cassandra/data/demo -maxdepth 1 -type d -name "${TABLE_NAME%%-*}*" | head -1)
    if [ -n "${DEST}" ]; then
        cp ${TABLE}/*.db ${DEST}/
    fi
done

# 6. Load SSTables (tells Cassandra about the files):
nodetool refresh demo employees
```

```bash
# ---- Repeat steps 5-6 on all other nodes ----

# 7. After all nodes loaded, run repair:
nodetool repair demo
```

#### Verify Restore

```sql
-- In cqlsh, check data:
USE demo;
SELECT COUNT(*) FROM employees;
SELECT * FROM employees LIMIT 10;
```

---

## 15. Troubleshooting Reference

### Node Won't Start

```bash
# Check log for error:
tail -100 /var/log/cassandra/system.log | grep ERROR

# Common causes:
# - listen_address wrong (check cassandra.yaml)
# - Port 9042 already in use: lsof -i :9042
# - Java not found: java -version
# - Directory permissions: ls -la /var/lib/cassandra/
```

### Node Can't Join Cluster

```bash
# Check seed nodes reachable:
ping 192.168.1.11
telnet 192.168.1.11 7000  # should connect

# Check cluster_name matches:
grep cluster_name /etc/cassandra/conf/cassandra.yaml
# Must be identical on all nodes

# Check snitch config:
cat /etc/cassandra/conf/cassandra-rackdc.properties
```

### nodetool Says DN (Node Down)

```bash
# SSH to that node, check service:
systemctl status cassandra
journalctl -u cassandra --no-pager -n 50

# If service running but showing DN, check firewall:
firewall-cmd --list-ports
# 7000/tcp must be open
```

### CQL Connection Refused

```bash
# Verify rpc_address in cassandra.yaml:
grep rpc_address /etc/cassandra/conf/cassandra.yaml
# Must match node's IP, not 127.0.0.1 (unless testing locally)

# Verify port open:
ss -tlnp | grep 9042
```

### Disk Space Full

```bash
# Check usage:
df -h /var/lib/cassandra/

# Clear old snapshots (safe to delete):
nodetool clearsnapshot demo

# Run compaction to merge SSTables:
nodetool compact demo
```

### High Latency / Slow Queries

```bash
# Check GC pressure (long GC = heap too small):
grep "GCInspector" /var/log/cassandra/system.log | tail -20

# Check tombstone warnings:
grep "tombstone" /var/log/cassandra/system.log | tail -20

# Show table statistics:
nodetool tablestats demo.employees

# Check thread pool stats:
nodetool tpstats
```

### Repair Failures

```bash
# Run repair with verbose output:
nodetool repair -v demo

# Repair single table:
nodetool repair demo employees

# If repair hangs, check:
nodetool compactionstats
nodetool tpstats | grep Repair
```

---

## Quick Reference Card

### Daily Commands

```bash
# Check cluster health
nodetool status

# Check one node's health
nodetool info

# Check ring token assignment
nodetool ring

# View table stats
nodetool tablestats <keyspace>.<table>

# Connect to CQL
cqlsh <node-ip> 9042
```

### Service Management

```bash
systemctl start cassandra
systemctl stop cassandra
systemctl restart cassandra
systemctl status cassandra
journalctl -u cassandra -f   # live logs
```

### Backup Quick Commands

```bash
# Create snapshot
nodetool snapshot -t <name> <keyspace>

# List snapshots
nodetool listsnapshots

# Delete snapshot
nodetool clearsnapshot -t <name> <keyspace>
```

### Emergency Contacts

| Situation | Action |
|-----------|--------|
| 1 node down | Cluster healthy — QUORUM (2 of 3) still works. Restore ASAP |
| 2 nodes down | **Critical** — only 1 node up, below QUORUM. Reads with ONE only. Restore immediately |
| All 3 nodes down | Emergency full restore procedure |
| Data loss suspected | Stop writes, contact senior team |

---

*SOP Version: 1.1 | OS: Rocky Linux 9 | Cassandra: 4.1.x | Cluster: 3-node, RF=3 | Last Updated: 2024*