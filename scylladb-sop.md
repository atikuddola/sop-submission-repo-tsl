
# ScyllaDB 3-Node Cluster — SOP

**OS:** Rocky Linux | **Version:** ScyllaDB 6.2.3
**Nodes:** `.34` · `.35` · `.237`
---
<img width="748" height="902" alt="svgviewer-png-output" src="https://github.com/user-attachments/assets/3a171bf2-f421-4dd8-873e-2f22853fb508" />

## 1. Pre-installation

- Verify CPU supports SSE4.2 + PCLMUL on all nodes:
  ```bash
  lscpu | grep -E 'sse4_2|pclmul'
  ```
- Update system:
  ```bash
  sudo dnf update -y
  sudo dnf install -y curl gnupg2 net-tools
  ```

---

## 2. Install ScyllaDB

Run on **all 3 nodes**:

```bash
curl -sSf get.scylladb.com/server | sudo bash -s -- --scylla-version 6.2.3
```

---

## 3. Configure `/etc/scylla/scylla.yaml`

Edit on **each node** — change `listen_address` and `rpc_address` per node:

```yaml
cluster_name: 'Scylla-Cluster'

data_file_directories:
    - /var/lib/scylla/data
commitlog_directory: /var/lib/scylla/commitlog

seed_provider:
    - class_name: org.apache.cassandra.locator.SimpleSeedProvider
      parameters:
          - seeds: "192.168.122.34,192.168.122.35,192.168.122.237"

listen_address: 192.168.122.XX      # this node's IP
rpc_address: 192.168.122.XX         # this node's IP

endpoint_snitch: GossipingPropertyFileSnitch
authenticator: PasswordAuthenticator
authorizer: CassandraAuthorizer
```

| Node   | `listen_address` / `rpc_address` |
|--------|----------------------------------|
| Node 1 | `192.168.122.34`                 |
| Node 2 | `192.168.122.35`                 |
| Node 3 | `192.168.122.237`                |

---

## 4. Configure `/etc/scylla/cassandra-rackdc.properties`

Same on all 3 nodes:

```properties
dc=scylla_data_center
rack=scylla_rack
```

---

## 5. Open firewall ports (Rocky Linux — all nodes)

```bash
sudo firewall-cmd --permanent --add-port=7000/tcp   # inter-node
sudo firewall-cmd --permanent --add-port=9042/tcp   # CQL clients
sudo firewall-cmd --permanent --add-port=10000/tcp  # REST API
sudo firewall-cmd --reload
```

---

## 6. SELinux (Rocky Linux)

```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
```

---

## 7. Run system optimization

```bash
sudo scylla_setup
```

Key responses:

| Prompt                     | Answer |
|----------------------------|--------|
| Check kernel version?      | yes    |
| Auto-start on boot?        | yes    |
| Setup NTP?                 | yes    |
| Setup RAID and XFS?        | no     |
| Run iotune?                | yes    |
| CPU scaling governor?      | yes    |
| Enable fstrim service?     | yes    |
| Scylla only service?       | yes    |
| Tune LimitNOFILES?         | yes    |

---

## 8. Start service (all nodes)

```bash
sudo systemctl start scylla-server
sudo systemctl enable scylla-server
sudo systemctl status scylla-server
```

Wait ~2 min, then verify:

```bash
nodetool status
```

Expected — all 3 rows show `UN`:

```
Datacenter: scylla_data_center
================================
-- Address           Tokens  Status
UN 192.168.122.34    256     Up/Normal
UN 192.168.122.35    256     Up/Normal
UN 192.168.122.237   256     Up/Normal
```

---

## 9. Security setup

Connect:

```bash
cqlsh -u cassandra -p cassandra 192.168.122.34 9042
```

```sql
-- Change default password immediately
ALTER ROLE cassandra WITH PASSWORD = 'your-secure-password';

-- Create admin user
CREATE ROLE dbadmin WITH SUPERUSER = true AND LOGIN = true AND PASSWORD = 'admin-password';

-- Verify
LIST ROLES;
```

---

## 10. Create keyspace & table

```sql
CREATE KEYSPACE test_keyspace
WITH replication = {
  'class': 'SimpleStrategy',
  'replication_factor': 3
} AND durable_writes = true;

USE test_keyspace;

CREATE TABLE employees (
    id UUID PRIMARY KEY,
    name TEXT,
    department TEXT,
    salary INT,
    joined_date DATE
);
```

---

## 11. How data flows

### INSERT

```
Client
  |
  | INSERT INTO employees VALUES (uuid(), 'Alice', ...)
  v
Murmur3 hash(uuid)  -->  token = -6,234,891,203,...
  |
  v
Token ring lookup
  [Node1: 0→1/3] [Node2: 1/3→2/3 <-- token lands here] [Node3: 2/3→1]
  |
  v
Node2 = coordinator (owns the token)
  |
  |-- write --> Node1 (.34)  [replica]
  |-- write --> Node2 (.35)  [primary]
  |-- write --> Node3 (.237) [replica]
  |
  v
All 3 write to commitlog + memtable --> ACK to client
```

### SELECT

```
Client
  |
  | SELECT * FROM employees WHERE id = <uuid>
  v
Murmur3 hash(same uuid)  -->  same token --> Node2 = coordinator
  |
  |-- read --> Node1 (.34)   responds with row + timestamp
  |-- read --> Node2 (.35)   responds with row + timestamp
  |            Node3 (.237)  NOT contacted (QUORUM = 2 of 3)
  v
Coordinator picks newest timestamp --> returns row to client
```

> If a node is down, coordinator reads from the remaining 2. Data is safe.

### vnode explained

```
Token space:  |----0-----------------------------------------2^63----|

Node1 owns 256 small scattered segments across the ring:
  |--N1--|--N2--|--N3--|--N1--|--N3--|--N2--|--N1--|--N2--|--N3--|

Not one big chunk — 256 small ones = balanced load
```

---

## 12. Verify replication

After inserting data on `.34`, query from `.35` and `.237`:

```bash
cqlsh -u cassandra -p cassandra 192.168.122.35 9042 \
  -e "SELECT * FROM test_keyspace.employees;"

cqlsh -u cassandra -p cassandra 192.168.122.237 9042 \
  -e "SELECT * FROM test_keyspace.employees;"
```

All 3 must return identical rows.

---

## 13. Common maintenance commands

```bash
nodetool status          # cluster health
nodetool info            # node details
nodetool repair          # fix data consistency
nodetool cleanup         # after adding/removing nodes
nodetool flush           # flush memtables to disk
nodetool snapshot <ks>   # backup keyspace
nodetool listsnapshots   # list backups
```

---

## 14. Troubleshooting

| Symptom | Check |
|---------|-------|
| Node missing from `nodetool status` | `journalctl -u scylla-server -n 100` |
| Port 9042 refused | `firewall-cmd --list-ports` |
| Node stuck joining | wipe `/var/lib/scylla/data/*` and restart |
| Auth error on ALTER ROLE | verify `authenticator` + `authorizer` in yaml, restart all nodes |
| SELinux blocking | `sudo setenforce 0` then restart |

### Node not joining — clean bootstrap

```bash
sudo systemctl stop scylla-server
sudo rm -rf /var/lib/scylla/data/*
sudo rm -rf /var/lib/scylla/commitlog/*
sudo rm -rf /var/lib/scylla/hints/*
sudo rm -rf /var/lib/scylla/view_hints/*
sudo systemctl start scylla-server
sudo journalctl -u scylla-server -f   # watch for "entering NORMAL mode"
```
