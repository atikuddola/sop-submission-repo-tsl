# Redis: Complete SOP
### Installation · Cluster Setup · Sharding · Failover

---

## TABLE OF CONTENTS

1. [What is Redis?](#1-what-is-redis)
2. [Core Concepts (Learn These First)](#2-core-concepts-learn-these-first)
   - 2.1 Data Structures
   - 2.2 Persistence
   - 2.3 Replication
   - 2.4 Clustering
   - 2.5 Sharding & Hash Slots
   - 2.6 Failover
3. [Objectives of This SOP](#3-objectives-of-this-sop)
4. [Installation](#4-installation)
   - 4.1 Ubuntu / Debian
   - 4.2 RHEL / CentOS / Rocky / AlmaLinux
5. [Redis Cluster: Architecture & Setup](#5-redis-cluster-architecture--setup)
   - 5.1 Architecture Overview
   - 5.2 Node Planning
   - 5.3 Configuration File (Every Node)
   - 5.4 Starting Nodes
   - 5.5 Forming the Cluster
   - 5.6 Verifying Cluster Health
6. [Data Sharding Deep Dive](#6-data-sharding-deep-dive)
7. [Failover Testing](#7-failover-testing)
   - 7.1 Automatic Failover Test
   - 7.2 Manual Failover
   - 7.3 Verifying Recovery
8. [Operational Runbook](#8-operational-runbook)
9. [Troubleshooting](#9-troubleshooting)

---

## 1. What is Redis?

Redis = **RE**mote **DI**ctionary **S**erver.

In-memory data store. Stores data in RAM → blazing fast reads/writes. Also optionally writes to disk for durability.

Used for:
- **Caching** — offload DB queries
- **Session storage** — fast user session lookup
- **Message queues** — producer/consumer pipelines
- **Pub/Sub** — real-time event broadcast
- **Rate limiting** — counters per user/IP
- **Leaderboards** — sorted sets for rankings

Key trait: **single-threaded** command execution. No race conditions on data. Scales via clustering, not multi-threading.

---

## 2. Core Concepts (Learn These First)

### 2.1 Data Structures

Redis not just key-value store. Supports rich types:

| Type | What it is | Use case |
|------|-----------|----------|
| **String** | Plain text/binary/number | Cache, counters |
| **Hash** | Map of field→value inside one key | User profile objects |
| **List** | Ordered linked list | Message queues, activity feed |
| **Set** | Unordered unique items | Tags, friend lists |
| **Sorted Set (ZSet)** | Set with score per member | Leaderboards, time-series |
| **Bitmap** | Bit-level operations on strings | Feature flags, analytics |
| **HyperLogLog** | Probabilistic unique count | Approx unique visitor count |
| **Stream** | Append-only log | Event sourcing |

### 2.2 Persistence

Redis data lives in RAM. Two ways to survive restart:

**RDB (Redis Database Snapshot)**
- Dumps full memory snapshot to disk at intervals
- Smaller file, faster restart
- Risk: lose data since last snapshot

**AOF (Append-Only File)**
- Logs every write command to file
- Higher durability — can replay all commands on restart
- Larger file, slower restart

**Best practice:** Use both. RDB for fast restore, AOF for durability.

### 2.3 Replication

Redis uses **primary → replica** model (formerly called master → slave).

- **Primary**: accepts reads and writes
- **Replica**: copies all data from primary, serves reads only
- Replication is **asynchronous** — tiny lag possible
- One primary can have multiple replicas
- If primary dies, a replica can be promoted

### 2.4 Clustering

Single Redis node = limited by one machine's RAM. Cluster = spread data across many nodes.

Redis Cluster is the **built-in horizontal scaling solution**:
- Multiple primary nodes, each holds portion of data
- Each primary has one or more replicas for HA
- Clients connect to any node — cluster redirects as needed
- Requires minimum **3 primary nodes** (Redis enforces this for quorum)

> **Why 3 minimum?** Cluster uses **quorum voting** to decide if a primary is really dead. Need majority agreement. With 2 nodes, a split-brain tie is possible — no majority forms. 3 nodes: 2 can agree, cluster survives 1 failure.

### 2.5 Sharding & Hash Slots

Redis Cluster divides keyspace into **16,384 hash slots**.

When you write key `user:1001`:
1. Redis runs **CRC16** algorithm on the key name
2. Takes result `mod 16384`
3. Gets a slot number (e.g., slot 7638)
4. That slot belongs to a primary node
5. Data goes to that node

Primary nodes split these 16,384 slots among themselves. Example with 3 primaries:
- Node A → slots 0–5460
- Node B → slots 5461–10922
- Node C → slots 10923–16383

Add more primaries? Rebalance = move slots (and their data) between nodes. No downtime needed.

**Hash tags:** Force related keys to same slot:
```
{user:1001}:profile
{user:1001}:cart
```
Curly brace part `{user:1001}` used for hashing → both keys land on same slot → same node → atomic operations work.

### 2.6 Failover

What happens when primary dies?

1. Replicas detect primary gone (miss heartbeats for `cluster-node-timeout` ms)
2. Replicas **request an election** from other primaries
3. Primaries vote — majority must agree
4. Winning replica **promotes itself** to primary
5. Cluster reconfigures, other nodes learn new topology
6. Old primary comes back → rejoins as replica of new primary

This is **automatic failover**. Requires: replica exists for that primary + quorum reachable.

---

## 3. Objectives of This SOP

This SOP achieves three goals:

| # | Goal | What you get |
|---|------|-------------|
| 1 | **Installation** | Redis installed on any Ubuntu or RHEL-based machine |
| 2 | **Cluster Setup** | Multi-node Redis cluster running with replicas for HA |
| 3 | **Sharding + Failover** | Data distributed across nodes, automatic failover tested and verified |

After completing this SOP, cluster will:
- Survive loss of any single primary (if replicas exist)
- Automatically elect new primary without manual intervention
- Distribute data evenly across all primary nodes
- Allow safe manual failover for planned maintenance

---

## 4. Installation

> **Do this on every node in your cluster.**

### Prerequisites (Both Distros)

- Root or sudo access
- Ports open between nodes: **6379** (Redis data), **16379** (cluster bus = data port + 10000)
- Disable transparent huge pages (performance)
- Set overcommit memory

Run on every node:

```bash
# Disable transparent huge pages (survives reboot via rc.local or systemd)
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# Persist across reboots (add to /etc/rc.local before exit 0)
echo 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' >> /etc/rc.local
echo 'echo never > /sys/kernel/mm/transparent_hugepage/defrag' >> /etc/rc.local

# Set vm.overcommit_memory (allows Redis fork for RDB snapshots)
echo "vm.overcommit_memory = 1" >> /etc/sysctl.conf
sysctl -p
```

---

### 4.1 Ubuntu / Debian

**Option A — Official Ubuntu package (simpler, slightly older version)**

```bash
sudo apt update
sudo apt install -y redis-server redis-tools

# Check version
redis-server --version

# Service status
sudo systemctl status redis-server
```

**Option B — Latest stable via official Redis repo (recommended)**

```bash
# Install prerequisites
sudo apt install -y lsb-release curl gpg

# Add Redis official GPG key
curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg

# Add Redis repo
echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/redis.list

# Install
sudo apt update
sudo apt install -y redis

# Verify
redis-server --version
redis-cli --version

# Enable service
sudo systemctl enable redis-server
sudo systemctl start redis-server
```

**Firewall (Ubuntu — ufw)**

```bash
# Allow Redis port from cluster nodes (replace with actual node IPs)
sudo ufw allow from <NODE_IP_1> to any port 6379
sudo ufw allow from <NODE_IP_1> to any port 16379
sudo ufw allow from <NODE_IP_2> to any port 6379
sudo ufw allow from <NODE_IP_2> to any port 16379
# Repeat for all nodes
sudo ufw reload
```

---

### 4.2 RHEL / CentOS / Rocky / AlmaLinux

**Option A — EPEL / AppStream (simpler)**

```bash
# RHEL 8/9, Rocky, Alma — Redis in AppStream
sudo dnf install -y redis

# Enable and start
sudo systemctl enable redis
sudo systemctl start redis

# Check version
redis-server --version
```

**Option B — Official Redis repo (latest stable)**

```bash
# Install dnf-plugins-core if not present
sudo dnf install -y dnf-plugins-core

# Add official Redis repo
sudo dnf config-manager --add-repo https://packages.redis.io/rpm/rhel/$(rpm -E %{rhel})/$(uname -m)/redis.repo

# Install
sudo dnf install -y redis

# Enable and start
sudo systemctl enable redis
sudo systemctl start redis

# Verify
redis-server --version
redis-cli --version
```

**Firewall (RHEL — firewalld)**

```bash
# Allow Redis ports from each cluster node
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<NODE_IP_1>" port port="6379" protocol="tcp" accept'
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<NODE_IP_1>" port port="16379" protocol="tcp" accept'
# Repeat for all other node IPs
sudo firewall-cmd --reload
```

---

### Post-Install Verification (Both Distros)

```bash
# Connect to local Redis
redis-cli ping
# Expected: PONG

# Check server info
redis-cli info server | grep redis_version
```

---

## 5. Redis Cluster: Architecture & Setup

### 5.1 Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                     REDIS CLUSTER                        │
│                                                         │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐          │
│  │ Primary A│    │ Primary B│    │ Primary C│          │
│  │slots 0–N │    │slots N–M │    │slots M–16383│        │
│  └────┬─────┘    └────┬─────┘    └────┬─────┘          │
│       │               │               │                  │
│  ┌────▼─────┐    ┌────▼─────┐    ┌────▼─────┐          │
│  │Replica A1│    │Replica B1│    │Replica C1│          │
│  └──────────┘    └──────────┘    └──────────┘          │
│                                                         │
│  Clients connect to any node → MOVED redirect if needed │
└─────────────────────────────────────────────────────────┘
```

**Key rules:**
- Every primary must have its own replica on a **different physical machine**
- Cluster bus port = data port + 10000 (automatic, do not block)
- All nodes must reach all other nodes on both ports

### 5.2 Node Planning

Before touching any config, plan your cluster on paper.

**Minimum viable cluster:**

| Role | Count | Why |
|------|-------|-----|
| Primaries | ≥ 3 | Quorum requires majority agreement |
| Replicas per primary | ≥ 1 | Survive primary loss |
| **Total nodes minimum** | **≥ 6** | 3 primaries + 3 replicas |

**Scaling rule:** Add primaries in odd increments to maintain quorum. Add replicas to increase read capacity or fault tolerance depth.

**Example node table (fill in your actual IPs):**

| Node | IP Address | Port | Role |
|------|-----------|------|------|
| node-1 | 192.168.1.10 | 6379 | Primary |
| node-2 | 192.168.1.11 | 6379 | Primary |
| node-3 | 192.168.1.12 | 6379 | Primary |
| node-4 | 192.168.1.13 | 6379 | Replica of node-1 |
| node-5 | 192.168.1.14 | 6379 | Replica of node-2 |
| node-6 | 192.168.1.15 | 6379 | Replica of node-3 |

### 5.3 Configuration File (Every Node)

Default config lives at `/etc/redis/redis.conf` (Ubuntu) or `/etc/redis.conf` (RHEL).

Back it up first:
```bash
sudo cp /etc/redis/redis.conf /etc/redis/redis.conf.bak
# RHEL:
# sudo cp /etc/redis.conf /etc/redis.conf.bak
```

Edit config on **every node** (adjust IP per node):

```bash
sudo nano /etc/redis/redis.conf
```

Key settings to change:

```conf
# ==========================================
# Redis Cluster Configuration - Node Template
# ==========================================

# --- NETWORK ---
bind 0.0.0.0
protected-mode no
port 6379
tcp-backlog 511
timeout 0
tcp-keepalive 300

# --- GENERAL ---
daemonize yes
pidfile /var/run/redis/redis_6379.pid
loglevel notice
logfile /var/log/redis/redis-server.log
databases 16
always-show-logo no
set-proc-title yes
proc-title-template "{title} {listen-addr} {server-mode}"

# --- SNAPSHOTTING (RDB) ---
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir /var/lib/redis

# --- REPLICATION ---
replica-serve-stale-data yes
replica-read-only yes
repl-diskless-sync no
repl-diskless-sync-delay 5
repl-disable-tcp-nodelay no
repl-backlog-size 1mb
repl-backlog-ttl 3600
replica-priority 100

# --- SECURITY ---
# ⚠️ Replace with a strong password (generate via: openssl rand -base64 32)
requirepass your_strong_password_here
masterauth your_strong_password_here
acllog-max-len 128

# --- MEMORY MANAGEMENT ---
maxmemory 512mb
maxmemory-policy allkeys-lru
maxmemory-samples 5
lazyfree-lazy-eviction no
lazyfree-lazy-expire no
lazyfree-lazy-server-del no
replica-ignore-maxmemory yes

# --- APPEND ONLY MODE (AOF) ---
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-load-truncated yes
aof-use-rdb-preamble yes

# --- LUA SCRIPTING ---
lua-time-limit 5000

# --- REDIS CLUSTER ---
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
# ⚠️ REPLACE WITH THIS NODE'S ACTUAL IP ADDRESS:
cluster-announce-ip <THIS_NODE_IP>
cluster-announce-port 6379
cluster-announce-bus-port 16379
cluster-replica-validity-factor 10
cluster-migration-barrier 1
cluster-require-full-coverage no
cluster-replica-no-failover no

# --- SLOW LOG ---
slowlog-log-slower-than 10000
slowlog-max-len 128

# --- LATENCY MONITOR ---
latency-monitor-threshold 0

# --- EVENT NOTIFICATION ---
notify-keyspace-events ""

# --- ADVANCED ---
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
stream-node-max-bytes 4096
stream-node-max-entries 100
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10
dynamic-hz yes
aof-rewrite-incremental-fsync yes
rdb-save-incremental-fsync yes
disable-thp yes
```

> **Important:** `cluster-config-file nodes.conf` is auto-managed by Redis. Never edit it manually.

### 5.4 Starting Nodes

On every node:

```bash
# Restart Redis to apply new config
sudo systemctl restart redis-server   # Ubuntu
# sudo systemctl restart redis        # RHEL

# Confirm running
sudo systemctl status redis-server

# Confirm cluster mode active (look for cluster_enabled:1)
redis-cli -a your_strong_password_here info cluster | grep cluster_enabled
```

### 5.5 Forming the Cluster

**Do this step only once, from any single node.**

Redis provides `redis-cli --cluster create` command. List all primary IPs first, then replica IPs. The `--cluster-replicas` flag tells Redis how many replicas per primary.

```bash
redis-cli --cluster create \
  192.168.122.34:6379 \
  192.168.122.35:6379 \
  192.168.122.237:6379 \
  192.168.122.33:6379 \
  192.168.122.225:6379 \
  192.168.122.11:6379 \
  --cluster-replicas 1 \
  -a your_strong_password_here
```

Redis will:
1. Show proposed assignment (which node becomes primary, which replica)
2. Ask for confirmation: type `yes`
3. Create cluster and assign hash slots

**Expected output snippet:**
```
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 192.168.1.13:6379 to 192.168.1.10:6379
Adding replica 192.168.1.14:6379 to 192.168.1.11:6379
Adding replica 192.168.1.15:6379 to 192.168.1.12:6379
...
[OK] All 16384 slots covered.
```

### 5.6 Verifying Cluster Health

```bash
# Check cluster info (from any node)
redis-cli -c -h 192.168.1.10 -p 6379 -a your_strong_password_here cluster info

# Key fields to verify:
# cluster_enabled:1
# cluster_state:ok
# cluster_slots_assigned:16384
# cluster_known_nodes:6
# cluster_size:3

# See all nodes and their roles
redis-cli -c -h 192.168.1.10 -p 6379 -a your_strong_password_here cluster nodes
```

**Cluster nodes output explained:**
```
<node-id> <ip:port@bus-port> <role> <primary-id> <ping-sent> <pong-recv> <epoch> <link-state> <slots>

Example:
abc123... 192.168.1.10:6379@16379 master - 0 1700000000 1 connected 0-5460
def456... 192.168.1.13:6379@16379 slave abc123... 0 1700000000 1 connected
```

Healthy cluster shows:
- All nodes `connected`
- All 16384 slots covered
- `cluster_state:ok`

---

## 6. Data Sharding Deep Dive

### How Keys Get Routed

Every key goes through this path:

```
Key name
   ↓
CRC16(key) % 16384
   ↓
Hash slot number (0–16383)
   ↓
Primary node owning that slot
   ↓
Data written/read there
```

### MOVED vs ASK Redirects

When client connects to wrong node:

- **MOVED**: Permanent redirect. Key belongs to this slot forever (until resharding). Client should cache this mapping.
- **ASK**: Temporary redirect during slot migration. Do not cache.

Use `-c` flag in redis-cli for automatic redirect following:
```bash
redis-cli -c -h 192.168.1.10 -p 6379 -a your_strong_password_here
```

### Test Sharding Distribution

```bash
# Connect with cluster mode (-c)
redis-cli -c -h 192.168.1.10 -p 6379 -a your_strong_password_here

# Write keys — watch MOVED redirects
> SET user:1 "alice"
> SET user:2 "bob"
> SET product:100 "widget"
> SET order:500 "pending"

# Check which slot a key lands on
> CLUSTER KEYSLOT user:1
> CLUSTER KEYSLOT product:100

# Force same slot with hash tags
> SET {session}:user1 "data1"
> SET {session}:user2 "data2"
> CLUSTER KEYSLOT {session}:user1
> CLUSTER KEYSLOT {session}:user2
# Both return same slot number
```

### Check Slot Distribution Per Node

```bash
redis-cli -h 192.168.1.10 -p 6379 -a your_strong_password_here cluster slots
```

Shows each slot range and which primary/replica serves it.

### Adding Nodes (Scale Out)

**Add new primary:**
```bash
# Add node to cluster
redis-cli --cluster add-node \
  <NEW_NODE_IP>:6379 \
  <EXISTING_NODE_IP>:6379 \
  -a your_strong_password_here

# Rebalance slots (interactive — accept defaults or specify slots)
redis-cli --cluster rebalance \
  <EXISTING_NODE_IP>:6379 \
  -a your_strong_password_here
```

**Add new replica:**
```bash
redis-cli --cluster add-node \
  <NEW_REPLICA_IP>:6379 \
  <EXISTING_NODE_IP>:6379 \
  --cluster-slave \
  --cluster-master-id <PRIMARY_NODE_ID> \
  -a your_strong_password_here
```

### Removing Nodes

```bash
# First reshard slots away from node being removed
redis-cli --cluster reshard <ANY_NODE_IP>:6379 -a your_strong_password_here
# Follow interactive prompts: how many slots, which node receives, which node sends

# Then remove node
redis-cli --cluster del-node \
  <ANY_NODE_IP>:6379 \
  <NODE_ID_TO_REMOVE> \
  -a your_strong_password_here
```

---

## 7. Failover Testing

> **Warning:** These tests cause intentional disruption. Always test in staging first. Confirm you have replicas before killing primaries.

### 7.1 Automatic Failover Test

**Step 1: Record current cluster state**

```bash
redis-cli -c -h 192.168.1.10 -p 6379 -a your_strong_password_here cluster nodes
```

Save output. Identify which node is primary for slot range 0-5460 (call it Primary-A).

**Step 2: Write test data**

```bash
redis-cli -c -h 192.168.1.10 -p 6379 -a your_strong_password_here
> SET failover_test "before_kill"
> GET failover_test
```

**Step 3: Kill a primary**

```bash
# SSH to Primary-A node and kill Redis
sudo systemctl stop redis-server   # Ubuntu
# sudo systemctl stop redis        # RHEL

# OR simulate crash (hard kill)
sudo kill -9 $(pgrep redis-server)
```

**Step 4: Watch failover happen (from another node)**

```bash
# Monitor cluster every 1 second
watch -n 1 'redis-cli -h 192.168.1.11 -p 6379 -a your_strong_password_here cluster nodes'
```

Expect within `cluster-node-timeout` (5 seconds default):
- Dead primary shows `disconnected`
- Replica promotes: role changes from `slave` to `master`
- Cluster state returns to `ok`

**Step 5: Verify data survived**

```bash
redis-cli -c -h 192.168.1.11 -p 6379 -a your_strong_password_here GET failover_test
# Expected: "before_kill"
```

**Step 6: Bring old primary back**

```bash
# Back on Primary-A node
sudo systemctl start redis-server

# Old primary rejoins as REPLICA of new primary
watch -n 1 'redis-cli -h 192.168.1.11 -p 6379 -a your_strong_password_here cluster nodes'
# Old Primary-A now shows role: slave
```

### 7.2 Manual Failover (Planned Maintenance)

Use when you want controlled switchover with zero data loss. Run from the **replica** you want to promote:

```bash
redis-cli -h <REPLICA_IP> -p 6379 -a your_strong_password_here CLUSTER FAILOVER
```

Process:
1. Replica stops accepting new data from primary
2. Waits for replication lag to reach zero
3. Promotes itself
4. Old primary demotes to replica

No data loss. Takes ~1-2 seconds.

**Force failover** (if primary unreachable, accept possible small data loss):
```bash
redis-cli -h <REPLICA_IP> -p 6379 -a your_strong_password_here CLUSTER FAILOVER FORCE
```

### 7.3 Verifying Recovery

After any failover, run full health check:

```bash
# Check cluster info
redis-cli -h 192.168.1.11 -p 6379 -a your_strong_password_here cluster info

# Must show:
# cluster_state:ok
# cluster_slots_assigned:16384
# cluster_slots_ok:16384

# Check all nodes connected
redis-cli --cluster check 192.168.1.11:6379 -a your_strong_password_here

# Write + read round trip
redis-cli -c -h 192.168.1.11 -p 6379 -a your_strong_password_here SET post_failover_test "ok"
redis-cli -c -h 192.168.1.11 -p 6379 -a your_strong_password_here GET post_failover_test
```

---

## 8. Operational Runbook

### Daily Health Check

```bash
# Run from any node
redis-cli --cluster check <ANY_NODE_IP>:6379 -a your_strong_password_here

# Quick cluster state
redis-cli -h <ANY_NODE_IP> -p 6379 -a your_strong_password_here cluster info | grep -E "cluster_state|cluster_slots|cluster_known_nodes"
```

### Check Memory Usage Per Node

```bash
redis-cli -h <NODE_IP> -p 6379 -a your_strong_password_here info memory | grep -E "used_memory_human|maxmemory_human"
```

### Monitor Live Operations

```bash
# Real-time command stream (use carefully on production — high output)
redis-cli -h <NODE_IP> -p 6379 -a your_strong_password_here monitor

# Slow log (commands taking > 10ms)
redis-cli -h <NODE_IP> -p 6379 -a your_strong_password_here slowlog get 25
```

### Backup (Manual RDB Snapshot)

```bash
# Trigger RDB save
redis-cli -h <NODE_IP> -p 6379 -a your_strong_password_here BGSAVE

# Check save status
redis-cli -h <NODE_IP> -p 6379 -a your_strong_password_here LASTSAVE
# Returns Unix timestamp of last save

# RDB file location
ls -lh /var/lib/redis/dump.rdb
```

### Flush Cluster Data (DANGER)

> **WARNING: IRREVERSIBLE. Deletes all data. Do NOT run on production without explicit authorization.**

```bash
# Flush all nodes
for ip in 192.168.1.10 192.168.1.11 192.168.1.12; do
  redis-cli -h $ip -p 6379 -a your_strong_password_here FLUSHALL ASYNC
done
```

### Restart Sequence (Planned Downtime)

Safe restart order to avoid data loss:

1. Stop all **replicas** first
2. Stop all **primaries** (Redis saves RDB on clean shutdown)
3. Start all **primaries**
4. Wait for primaries to be ready (`PING` returns `PONG`)
5. Start all **replicas** (they will resync from primaries)

```bash
# Clean shutdown (triggers RDB save)
redis-cli -h <NODE_IP> -p 6379 -a your_strong_password_here SHUTDOWN SAVE
```

---

## 9. Troubleshooting

### Cluster State Not `ok`

```bash
redis-cli --cluster fix <ANY_AVAILABLE_NODE_IP>:6379 -a your_strong_password_here
```

Fixes:
- Orphaned slots
- Incomplete slot migration
- Node disagreements on topology

### Node Showing `fail`

Check:
1. Is Redis process running on that node? `sudo systemctl status redis-server`
2. Network reachable? `ping <NODE_IP>` and `telnet <NODE_IP> 6379`
3. Firewall blocking? Check ufw/firewalld rules
4. Check logs: `sudo journalctl -u redis-server -n 100 --no-pager`

### `CLUSTERDOWN` Error on Writes

Cluster lost quorum — too many primaries down simultaneously.

```bash
# Check which primaries down
redis-cli -h <ALIVE_NODE> -p 6379 -a your_strong_password_here cluster nodes | grep fail
```

Recovery: Bring dead primaries back. If primary dead permanently with no replica, cluster cannot self-heal — must restore from backup.

### `MOVED` Error in Non-Cluster Client

Client not using cluster-aware library. Solutions:
- Use `-c` flag in redis-cli
- Use cluster-aware Redis client in application (most modern clients support this)

### Replication Lag

```bash
# Check lag on replica
redis-cli -h <REPLICA_IP> -p 6379 -a your_strong_password_here info replication | grep master_repl_offset
redis-cli -h <REPLICA_IP> -p 6379 -a your_strong_password_here info replication | grep slave_repl_offset
# Difference = lag in bytes
```

### High Memory Usage / Evictions

```bash
# Check eviction stats
redis-cli -h <NODE_IP> -p 6379 -a your_strong_password_here info stats | grep evicted_keys
```

If evictions happening: increase `maxmemory`, add nodes and rebalance, or review key TTL strategy.

### Slot Migration Stuck

```bash
# Check stuck slots
redis-cli -h <NODE_IP> -p 6379 -a your_strong_password_here cluster nodes | grep -E "MIGRATING|IMPORTING"

# Fix
redis-cli --cluster fix <NODE_IP>:6379 -a your_strong_password_here
```

---

*SOP Version 1.0 | Maintained by: [Your Name] | Last updated: 2026-04*