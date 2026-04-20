# MongoDB Replica Set — Complete SOP
> **Purpose:** Failproof guide for installation, replica set setup, and failover testing.
> **Audience:** Coworker with basic Linux skills. No prior MongoDB cluster experience needed.
> **OS Coverage:** Ubuntu / Debian AND RHEL / CentOS / Rocky / AlmaLinux

---

## Part 1 — Concepts (Read Before Touching Anything)

### 1.1 What Is MongoDB?

MongoDB is a **document database** — data stored as JSON-like documents (called BSON), not rows/columns. Collections replace tables. No fixed schema required.

---

### 1.2 What Is a Replica Set?

A **Replica Set** is a group of MongoDB processes (called **members**) that all hold the same data. They constantly sync with each other. If one dies, the others take over automatically.

Think of it as: **one leader, rest are followers.** Leader handles all writes. Followers replicate data from the leader and can serve reads.

```
┌─────────────┐     replication     ┌─────────────┐
│   PRIMARY   │ ──────────────────► │  SECONDARY  │
│  (writes)   │                     │  (replica)  │
└─────────────┘                     └─────────────┘
       │                                   │
       │          replication              │
       ▼                                   ▼
┌─────────────┐                     ┌─────────────┐
│  SECONDARY  │                     │   ARBITER   │
│  (replica)  │                     │ (vote only) │
└─────────────┘                     └─────────────┘
```

---

### 1.3 Member Roles

| Role | Holds Data? | Can Vote? | Can Become Primary? |
|------|-------------|-----------|----------------------|
| **Primary** | Yes | Yes | Already is |
| **Secondary** | Yes | Yes | Yes |
| **Arbiter** | No | Yes | No |

**Primary** — the one active leader. All writes go here.

**Secondary** — syncs data from primary. Promotes itself to primary if primary fails.

**Arbiter** — lightweight member. Holds no data. Only participates in elections to break ties. Use when you have an even number of data-bearing members and need a tiebreaker.

---

### 1.4 Elections

When primary goes down, remaining members **vote** to elect a new primary. Election needs a **majority** of all configured votes to succeed.

> **Critical rule:** Always configure an **odd total vote count** (or use Arbiter to make it odd). Even vote count risks a tie = no election = cluster stuck.

Examples:
- 2 data nodes → add 1 Arbiter → 3 votes total ✓
- 3 data nodes → 3 votes total ✓
- 4 data nodes → add 1 Arbiter → 5 votes total ✓

---

### 1.5 Write Concern & Read Preference

**Write Concern** — how many members must acknowledge a write before MongoDB confirms it to the app.
- `w:1` → only Primary confirms (fast, less safe)
- `w:majority` → majority of members confirm (slower, safer)

**Read Preference** — which member the app reads from.
- `primary` → always read from primary (default, consistent)
- `secondaryPreferred` → read from secondary if available (lower load on primary)

---

### 1.6 Oplog

The **oplog** (operations log) is a special capped collection on each member that records every write operation. Secondaries tail the primary's oplog to stay in sync. If a secondary falls too far behind and the oplog rolls over, it goes into **RECOVERING** state and needs manual resync.

---

### 1.7 Key Terms Cheat Sheet

| Term | Meaning |
|------|---------|
| `mongod` | MongoDB server process |
| `mongosh` | MongoDB shell (CLI to interact) |
| `rs` | Replica Set object in shell |
| `rs.status()` | Shows current cluster health |
| `rs.conf()` | Shows replica set configuration |
| `rs.initiate()` | Bootstraps a new replica set |
| `rs.add()` | Adds a member |
| `rs.stepDown()` | Forces primary to step down (triggers election) |
| Keyfile | Shared secret file — members authenticate with each other using this |

---

## Part 2 — Architecture for This Setup

### 2.1 Our Objective

Deploy a **MongoDB Replica Set** with:
- Multiple data-bearing members for redundancy
- Automatic failover when any member goes down
- Internal authentication between members via keyfile
- Ability to verify failover works before going to production

### 2.2 Port & Firewall Requirements

MongoDB listens on **port 27017** by default.

Every member must be able to reach every other member on port 27017. Open this port between all cluster nodes. Do NOT expose 27017 to the internet.

### 2.3 Hostname Planning

Before starting, decide hostnames/IPs for all members. Example layout (replace with your actual values):

| Member | Hostname (example) | Role |
|--------|--------------------|------|
| Node 1 | `mongo1.internal` | Primary (initially) |
| Node 2 | `mongo2.internal` | Secondary |
| Node 3 | `mongo3.internal` | Secondary |
| Node N | `mongoN.internal` | Secondary or Arbiter |

> Add all hostnames to `/etc/hosts` on every node, OR use DNS. Members reference each other by hostname — this must resolve correctly from every node.

---

## Part 3 — Installation

> **Run Section 3.1 OR 3.2 depending on your OS. Then all nodes do Part 4 onward.**

---

### 3.1 Ubuntu / Debian

Run on **every node**:

```bash
# 1. Import MongoDB GPG key
curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | \
  sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor

# 2. Add MongoDB repo (Ubuntu 22.04 example — change "jammy" for other versions)
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] \
  https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | \
  sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list

# 3. Update and install
sudo apt-get update
sudo apt-get install -y mongodb-org

# 4. Enable and start service
sudo systemctl daemon-reload
sudo systemctl enable mongod
sudo systemctl start mongod

# 5. Verify running
sudo systemctl status mongod
```

> **Ubuntu version → codename mapping:**
> - 20.04 → `focal`
> - 22.04 → `jammy`
> - 24.04 → `noble`

---

### 3.2 RHEL / CentOS / Rocky / AlmaLinux

Run on **every node**:

```bash
# 1. Create MongoDB repo file
sudo tee /etc/yum.repos.d/mongodb-org-7.0.repo << 'EOF'
[mongodb-org-7.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/7.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-7.0.asc
EOF

# 2. Install
sudo dnf install -y mongodb-org
# (use yum instead of dnf on older systems)

# 3. Enable and start
sudo systemctl daemon-reload
sudo systemctl enable mongod
sudo systemctl start mongod

# 4. Verify
sudo systemctl status mongod

# 5. SELinux — if enforcing, allow MongoDB port
sudo semanage port -a -t mongod_port_t -p tcp 27017
# If semanage not found: sudo dnf install -y policycoreutils-python-utils

# 6. Firewall (if firewalld active)
sudo firewall-cmd --zone=public --add-port=27017/tcp --permanent
sudo firewall-cmd --reload
```

---

### 3.3 Post-Install: Verify on All Nodes

```bash
mongod --version
mongosh --version
```

Both commands must return version strings. If not, installation failed — recheck repo setup.

---

## Part 4 — Configure Each Node for Replica Set

Do this on **every node** before initiating the replica set.

### 4.1 Generate Keyfile (On ONE node, copy to all)

The keyfile is a shared secret. All members authenticate each other using it.

```bash
# Run on Node 1 only
openssl rand -base64 756 > /tmp/mongodb-keyfile
sudo mkdir -p /etc/mongodb
sudo cp /tmp/mongodb-keyfile /etc/mongodb/keyfile
sudo chown mongod:mongod /etc/mongodb/keyfile
sudo chmod 400 /etc/mongodb/keyfile
```

```
# Run on rest of the nodes
sudo mkdir -p /etc/mongodb

```
**Copy keyfile to all other nodes:**

```bash
# Run on Node 1 — replace with actual IPs/hostnames
scp /tmp/mongodb-keyfile user@mongo2.internal:/tmp/mongodb-keyfile
scp /tmp/mongodb-keyfile user@mongo3.internal:/tmp/mongodb-keyfile
```


**On Node 2, Node 3, ... Node N — place keyfile:**

```bash

sudo cp /tmp/mongodb-keyfile /etc/mongodb/keyfile
sudo chown mongod:mongod /etc/mongodb/keyfile
sudo chmod 400 /etc/mongodb/keyfile
```

---

### 4.2 Edit mongod.conf on Every Node

```bash
sudo nano /etc/mongod.conf
```

Set or confirm these values (replace hostnames with your actual ones):

```yaml
# Network
net:
  port: 27017
  bindIp: 0.0.0.0          # listen on all interfaces (restrict via firewall instead)

# Storage
storage:
  dbPath: /var/lib/mongodb

# Logging
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

# Replica Set — THIS IS THE KEY PART
replication:
  replSetName: "rs0"        # Must be IDENTICAL on every node

# Security — enable after replica set is initiated
#security:
  #keyFile: /etc/mongodb/keyfile
  #authorization: enabled
```
> **Make sure to comment the `security` section due to no prior admin user.**

> **`replSetName` must match exactly on every node — same string, same case.**

Restart after editing:

```bash
sudo systemctl restart mongod
sudo systemctl status mongod
```

---

## Part 5 — Initiate the Replica Set

### 5.1 Connect to Node 1

```bash
mongosh --host mongo1.internal --port 27017
```

### 5.2 Initiate

```javascript
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo1.internal:27017" },
    { _id: 1, host: "mongo2.internal:27017" },
    { _id: 2, host: "mongo3.internal:27017" }
    // Add more members here if needed:
    // { _id: 3, host: "mongo4.internal:27017" }
  ]
})
```

Expected response: `{ ok: 1 }`

If error → check that `replSetName` in mongod.conf matches `_id` here.

### 5.3 Verify

```javascript
rs.status()
```

Look for:
- One member with `"stateStr": "PRIMARY"`
- All others with `"stateStr": "SECONDARY"`
- No members in `STARTUP` state after ~30 seconds

```javascript
rs.conf()
```

Shows full configuration. Confirm all members listed.

---

### 5.4 Adding an Arbiter (Optional — for even data node counts)

Connect to primary:

```javascript
rs.addArb("mongoArbiter.internal:27017")
```

Arbiter must also have mongod installed and running with same `replSetName` in mongod.conf. Arbiter does NOT need the same storage capacity — it holds no data.

---

## Part 6 — Create Admin User

> Do this immediately after initiating replica set. Required before enabling auth.

Connect to primary:

```bash
mongosh --host mongo1.internal --port 27017
```

```javascript
use admin
db.createUser({
  user: "admin",
  pwd: "CHANGE_THIS_STRONG_PASSWORD",
  roles: [ { role: "root", db: "admin" } ]
})
```

> **Save this password somewhere safe. Losing it = locked out.**




Now verify auth works:

```bash
mongosh --host mongo1.internal --port 27017 -u admin -p --authenticationDatabase admin
```
> **With `mongosh` / `mongosh --host host-name` can be acceesed but eveything is secured if anyone try to login withount admin profile**
---

## Part 7 — Ongoing: Useful Commands

### Check Cluster Health

```javascript
rs.status()          // full health view
rs.hello()           // quick: who is primary?
rs.printReplicationInfo()     // oplog size + time coverage
rs.printSecondaryReplicationInfo()  // how far behind each secondary is
```

### Check Who Is Primary Right Now

```javascript
db.hello().primary
```

### View All Members + States

```javascript
rs.status().members.forEach(m => print(m.name, m.stateStr, m.health))
```

---

## Part 8 — Failover Test

> **Goal:** Prove automatic failover works. Safe to run in staging. In production, schedule a maintenance window.

### 8.1 Before Test — Record Current Primary

```bash
mongosh --host mongo1.internal -u admin -p --authenticationDatabase admin --eval "db.hello().primary"
```

Note the primary hostname. Call it `CURRENT_PRIMARY`.

### 8.2 Trigger Failover

Connect to the current primary and step it down:

```bash
mongosh --host <CURRENT_PRIMARY> -u admin -p --authenticationDatabase admin
```

```javascript
rs.stepDown(60)    // steps down for 60 seconds, triggers election
```

Shell will disconnect — that is expected. Primary stepped down.

### 8.3 Observe Election

Wait 10–15 seconds, then connect to any node:

```bash
mongosh --host mongo2.internal -u admin -p --authenticationDatabase admin --eval "rs.status()"
```

Confirm:
- A **different** node is now `PRIMARY`
- Old primary is now `SECONDARY`
- All members show `health: 1`

### 8.4 Test Write on New Primary

```javascript
use testdb
db.failovertest.insertOne({ test: "failover", ts: new Date() })
db.failovertest.findOne()
```

Write must succeed.

### 8.5 Verify Data Replicated to All Secondaries

Connect to a secondary:

```bash
mongosh --host mongo3.internal -u admin -p --authenticationDatabase admin
```

```javascript
rs.secondaryOk()     // allow reads on secondary
use testdb
db.failovertest.findOne()
```

Same document must appear.

### 8.6 Simulate Hard Node Failure

Stop mongod on one node (simulates crash):

```bash
sudo systemctl stop mongod
```

Check cluster from another node:

```javascript
rs.status()
```

Down node shows `health: 0` and state `(not reachable/healthy)`. Other members still `health: 1`. New primary elected if the stopped node was primary.

**Bring it back:**

```bash
sudo systemctl start mongod
```

Node rejoins automatically. Watch it go through `STARTUP2` → `SECONDARY`.

---

## Part 9 — Troubleshooting

### mongod won't start

```bash
sudo journalctl -u mongod -n 50 --no-pager
cat /var/log/mongodb/mongod.log | tail -50
```

Common causes:
- Wrong keyfile permissions → must be `400`, owned by `mongodb`
- Port 27017 already in use → `ss -tlnp | grep 27017`
- Data dir permissions → `sudo chown -R mongodb:mongodb /var/lib/mongodb`

### Member stuck in STARTUP or RECOVERING

```javascript
rs.status()    // read the errmsg field of the stuck member
```

Usually means oplog gap. Fix:
1. Stop mongod on stuck node
2. Delete data dir contents: `sudo rm -rf /var/lib/mongodb/*`
3. Start mongod — it will do initial sync from primary

> **Warning:** Deleting data dir is destructive. Only do on a non-primary member.

### No Primary Elected

Possible causes:
- Majority of members down — need more than half of total votes alive
- Network partition — nodes can't reach each other
- Keyfile mismatch — members reject each other

Check logs on each node for `"not enough members for election"` or authentication errors.

### Can't Connect with Auth

```bash
mongosh --host mongo1.internal -u admin -p --authenticationDatabase admin
```

If rejected: confirm user created in `admin` db, and `authorization: enabled` is in mongod.conf.

---

## Part 10 — Quick Reference Card

| Task | Command |
|------|---------|
| Check who is primary | `db.hello().primary` |
| Full cluster status | `rs.status()` |
| Cluster config | `rs.conf()` |
| Add a new member | `rs.add("newhostname:27017")` |
| Add arbiter | `rs.addArb("arbiterhost:27017")` |
| Remove a member | `rs.remove("hostname:27017")` |
| Force step down primary | `rs.stepDown(60)` |
| Allow reads on secondary | `rs.secondaryOk()` |
| Check replication lag | `rs.printSecondaryReplicationInfo()` |
| Restart mongod | `sudo systemctl restart mongod` |
| View mongod logs | `sudo journalctl -u mongod -f` |

---

## Appendix — File Locations

| File | Path |
|------|------|
| Main config | `/etc/mongod.conf` |
| Keyfile | `/etc/mongodb/keyfile` |
| Data directory | `/var/lib/mongodb/` |
| Log file | `/var/log/mongodb/mongod.log` |
| Service | `mongod` (systemctl) |

---

*SOP End. Last reviewed: April 2026.*
