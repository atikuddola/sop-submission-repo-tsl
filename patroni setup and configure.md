# PostgreSQL 16 High Availability with Patroni
## Complete Implementation & Scaling Guide (SOP)

> **Version:** 1.0 | **Date:** March 2026
> **Stack:** PostgreSQL 16 · Patroni · etcd · HAProxy · Keepalived
> **OS:** RHEL 9 / Rocky Linux 9

---
<img width="972" height="712" alt="Untitled Diagram drawio" src="https://github.com/user-attachments/assets/59787df7-211a-4f42-8fad-4e0cc4d18a88" />

---

## Table of Contents
 
- [1. Infrastructure Overview](#1-infrastructure-overview)
- [2. Phase 1 — Initial Cluster Setup](#2-phase-1--initial-cluster-setup)
  - [2.1 System Preparation](#21-system-preparation-all-nodes)
  - [2.2 etcd Cluster Configuration](#22-etcd-cluster-configuration-db-nodes-only)
  - [2.3 Proxy Layer — Keepalived & HAProxy](#23-proxy-layer-pgproxy-only)
  - [2.4 Patroni & PostgreSQL Installation](#24-patroni--postgresql-installation-db-nodes-only)
  - [2.5 Starting the Cluster](#25-starting-the-cluster)
- [3. Phase 2 — Adding a New Node (Scaling)](#3-phase-2--adding-a-new-node-scaling)
  - [3.1 Adding etcd Member](#31-adding-etcd-member)
  - [3.2 Updating Cluster Configuration (Critical Step)](#32-updating-cluster-configuration-critical-step)
  - [3.3 Bootstrapping the New Node](#33-bootstrapping-the-new-node)
  - [3.4 Why This Process Matters](#34-why-this-process-matters)
- [4. Phase 3 — Operations & Verification](#4-phase-3--operations--verification)
  - [4.1 Daily Operations](#41-daily-operations)
  - [4.2 Failover Testing](#42-failover-testing)
  - [4.3 Troubleshooting](#43-troubleshooting)
- [5. Appendix — Configuration References](#5-appendix--configuration-references)
 
---
 
## 1. Infrastructure Overview
 
This guide establishes a highly available PostgreSQL cluster where applications connect to a **Virtual IP (VIP)**. The VIP automatically routes traffic to the **primary node** for writes and **replica nodes** for reads.
 
### How It Works — The Flow
 
```
Client Application
       │
       ▼
  VIP :5000 (write) / :5001 (read)
       │
       ▼
   HAProxy (on pgproxy)
  ┌────┴──────────────────────────────┐
  │  Checks Patroni REST API (:8008)  │
  │  /master → 200 OK = Primary       │
  │  /replica → 200 OK = Replica      │
  └────┬──────────────────────────────┘
       │
  ┌────▼────────────────────────┐
  │  pgnode1 / pgnode2 / pgnode3│
  │   PostgreSQL 16 + Patroni   │
  │   Patroni ←→ etcd (DCS)    │
  └─────────────────────────────┘
```
 
**Automatic Failover Flow** (when primary dies):
1. Primary node goes down → Patroni stops
2. etcd leader lock expires (TTL: 30 seconds)
3. Remaining Patroni nodes race to acquire the lock
4. Winner promotes itself via `pg_promote()`
5. HAProxy detects new primary via health checks and reroutes traffic
6. **Total failover time: ~30 seconds**
 
---
 
### 1.1 Node Map
 
| Hostname | IP Address      | Role       | Components                        |
|----------|-----------------|------------|-----------------------------------|
| pgnode1  | 192.168.122.98  | Database   | etcd, Patroni, PostgreSQL         |
| pgnode2  | 192.168.122.147 | Database   | etcd, Patroni, PostgreSQL         |
| pgnode3  | 192.168.122.105 | Database   | etcd, Patroni, PostgreSQL         |
| pgproxy  | 192.168.122.181 | Proxy      | HAProxy, Keepalived               |
| ha-vip   | 192.168.122.112 | Virtual IP | Floating IP for Clients (not a VM)|
 
> **Note:** `ha-vip` is **not a separate VM**. It is a virtual IP address that floats between nodes using the VRRP protocol via Keepalived.
 
### 1.2 Port Map
 
| Port | Service         | Purpose                     |
|------|-----------------|-----------------------------|
| 2379 | etcd            | Client connections          |
| 2380 | etcd            | Peer-to-peer communication  |
| 5432 | PostgreSQL      | Database connections        |
| 5000 | HAProxy         | Write endpoint (Primary)    |
| 5001 | HAProxy         | Read endpoint (Replicas)    |
| 7000 | HAProxy         | Stats dashboard             |
| 8008 | Patroni REST API| Health checks by HAProxy    |
 
### 1.3 Component Roles
 
| Component   | Role                                                                                    |
|-------------|-----------------------------------------------------------------------------------------|
| PostgreSQL  | The actual database engine. Patroni manages it — do NOT start it manually.              |
| Patroni     | HA manager. Monitors PG, holds leader lock in etcd, triggers automatic failover.        |
| etcd        | Distributed key-value store. Stores cluster state & leader lock. Acts as the "brain".  |
| HAProxy     | Load balancer. Routes writes to primary, reads to replicas via Patroni REST API checks. |
| Keepalived  | Manages the VIP. VIP floats to whichever node has active HAProxy (VRRP protocol).      |
 
---
 
## 2. Phase 1 — Initial Cluster Setup
 
### 2.1 System Preparation (All Nodes)
 
> Run on **pgnode1, pgnode2, pgnode3, and pgproxy** unless otherwise stated.
 
#### Set Hostnames
 
Run on each respective node (change hostname per node):
 
```bash
sudo hostnamectl set-hostname pgnode1
# Change to pgnode2, pgnode3, pgproxy on each respective machine
reboot
```
 
#### Configure Hosts File
 
Edit `/etc/hosts` on **all nodes** (same content everywhere):
 
```bash
sudo nano /etc/hosts
```
 
Add:
 
```
192.168.122.98   pgnode1
192.168.122.147  pgnode2
192.168.122.105  pgnode3
192.168.122.181  pgproxy
192.168.122.112  ha-vip
```
 
#### Disable SELinux
 
```bash
sudo nano /etc/selinux/config
```
 
Change:
```
SELINUX=enforcing
```
To:
```
SELINUX=disabled
```
 
```bash
reboot
```
 
#### Disable Firewalld
 
```bash
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```
 
#### Create postgres User (DB Nodes Only — pgnode1, pgnode2, pgnode3)
 
```bash
sudo useradd postgres
sudo passwd postgres
echo 'postgres ALL=(ALL) NOPASSWD:ALL' | sudo tee /etc/sudoers.d/postgres
```
 
#### Install PostgreSQL 16 (DB Nodes Only — pgnode1, pgnode2, pgnode3)
 
```bash
# Add official PostgreSQL YUM repository
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
 
# Disable built-in PostgreSQL module to avoid conflicts
sudo dnf -qy module disable postgresql
 
# Install PostgreSQL 16
sudo dnf install -y postgresql16-server postgresql16-contrib
```
 
> **⚠️ Critical:** Do **NOT** run `postgresql-16-setup initdb`. Do **NOT** enable or start `postgresql-16` service. Patroni manages initialization and startup.
 
---
 
### 2.2 etcd Cluster Configuration (DB Nodes Only)
 
**What is etcd?**
etcd is a distributed key-value store that acts as the "brain" of the cluster. Patroni uses it as a Distributed Configuration Store (DCS) to store cluster state and maintain the leader lock. A 3-node etcd cluster requires 2 nodes to be available for quorum (majority).
 
#### Install etcd
 
```bash
sudo dnf install 'dnf-command(config-manager)' -y
sudo dnf config-manager --enable pgdg-rhel9-extras
sudo dnf -y install etcd
```
 
#### Create Data Directories
 
```bash
# On pgnode1:
sudo mkdir -p /var/lib/etcd/pgnode1
 
# On pgnode2:
sudo mkdir -p /var/lib/etcd/pgnode2
 
# On pgnode3:
sudo mkdir -p /var/lib/etcd/pgnode3
 
# Set ownership (run on all db nodes):
sudo chown -R etcd:etcd /var/lib/etcd/
```
 
#### Configure etcd — pgnode1
 
```bash
sudo mv /etc/etcd/etcd.conf /etc/etcd/etcd.conf.orig
sudo nano /etc/etcd/etcd.conf
```
 
```ini
ETCD_NAME=pgnode1
ETCD_DATA_DIR="/var/lib/etcd/pgnode1"
 
ETCD_LISTEN_PEER_URLS="http://192.168.122.98:2380,http://127.0.0.1:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.122.98:2379,http://127.0.0.1:2379"
 
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.122.98:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.122.98:2379"
 
ETCD_INITIAL_CLUSTER="pgnode1=http://192.168.122.98:2380,pgnode2=http://192.168.122.147:2380,pgnode3=http://192.168.122.105:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ENABLE_V2="true"
```
 
#### Configure etcd — pgnode2
 
```ini
ETCD_NAME=pgnode2
ETCD_DATA_DIR="/var/lib/etcd/pgnode2"
 
ETCD_LISTEN_PEER_URLS="http://192.168.122.147:2380,http://127.0.0.1:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.122.147:2379,http://127.0.0.1:2379"
 
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.122.147:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.122.147:2379"
 
ETCD_INITIAL_CLUSTER="pgnode1=http://192.168.122.98:2380,pgnode2=http://192.168.122.147:2380,pgnode3=http://192.168.122.105:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ENABLE_V2="true"
```
 
#### Configure etcd — pgnode3
 
```ini
ETCD_NAME=pgnode3
ETCD_DATA_DIR="/var/lib/etcd/pgnode3"
 
ETCD_LISTEN_PEER_URLS="http://192.168.122.105:2380,http://127.0.0.1:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.122.105:2379,http://127.0.0.1:2379"
 
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.122.105:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.122.105:2379"
 
ETCD_INITIAL_CLUSTER="pgnode1=http://192.168.122.98:2380,pgnode2=http://192.168.122.147:2380,pgnode3=http://192.168.122.105:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ENABLE_V2="true"
```
 
#### Start etcd (All DB Nodes)
 
```bash
sudo systemctl enable etcd
sudo systemctl start etcd
sudo systemctl status etcd
```
 
#### Configure bash_profile (All DB Nodes)
 
Add to `~/.bash_profile`:
 
```bash
export ENDPOINTS="http://192.168.122.98:2379,http://192.168.122.147:2379,http://192.168.122.105:2379"
source ~/.bash_profile
```
 
#### Verify etcd Cluster Health
 
```bash
etcdctl endpoint health --write-out=table --endpoints=$ENDPOINTS
etcdctl endpoint status --write-out=table --endpoints=$ENDPOINTS
```
 
Expected output — all three nodes healthy:
 
```
+-------------------------------+--------+
| ENDPOINT                      | HEALTH |
+-------------------------------+--------+
| http://192.168.122.98:2379    | true   |
| http://192.168.122.147:2379   | true   |
| http://192.168.122.105:2379   | true   |
+-------------------------------+--------+
```
 
---
 
### 2.3 Proxy Layer (pgproxy Only)
 
**What is HAProxy + Keepalived doing here?**
HAProxy load-balances client connections — port 5000 always routes to the current PostgreSQL primary, port 5001 round-robins across replicas. It determines who is primary by calling Patroni's REST API (`/master` returns HTTP 200 only on the primary node). Keepalived manages the VIP using the VRRP protocol, so if `pgproxy` itself goes down, another node could take over the VIP (in multi-proxy setups).
 
#### Enable IP Non-Local Bind
 
```bash
sudo nano /etc/sysctl.conf
```
 
Add:
```
net.ipv4.ip_nonlocal_bind = 1
net.ipv4.ip_forward = 1
```
 
Apply:
```bash
sudo sysctl --system
```
 
#### Install Keepalived
 
```bash
sudo dnf -y install keepalived
```
 
#### Configure Keepalived
 
```bash
sudo mv /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.orig
sudo nano /etc/keepalived/keepalived.conf
```
 
> **Note:** Verify your network interface name with `ip link show` — replace `enp1s0` if different.
 
```
vrrp_script check_haproxy {
    script "pkill -0 haproxy"
    interval 2
    weight 2
}
 
vrrp_instance VI_1 {
    state MASTER
    interface enp1s0
    virtual_router_id 51
    priority 101
    advert_int 1
    virtual_ipaddress {
        192.168.122.112
    }
    track_script {
        check_haproxy
    }
}
```
 
#### Start Keepalived
 
```bash
sudo systemctl enable keepalived
sudo systemctl start keepalived
```
 
Verify VIP is assigned:
 
```bash
ip addr show enp1s0 | grep 192.168.122.112
```
 
#### Install HAProxy
 
```bash
sudo dnf -y install haproxy
```
 
#### Configure HAProxy
 
```bash
sudo mv /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.orig
sudo nano /etc/haproxy/haproxy.cfg
```
 
```
global
    maxconn 1000
 
defaults
    mode                tcp
    log                 global
    option              tcplog
    retries             3
    timeout queue       1m
    timeout connect     4s
    timeout client      60m
    timeout server      60m
    timeout check       5s
    maxconn             900
 
listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /
 
listen primary
    bind 192.168.122.112:5000
    option httpchk
    http-check send meth GET uri /master
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server pgnode1 192.168.122.98:5432  maxconn 100 check port 8008
    server pgnode2 192.168.122.147:5432 maxconn 100 check port 8008
    server pgnode3 192.168.122.105:5432 maxconn 100 check port 8008
 
listen standby
    bind 192.168.122.112:5001
    balance roundrobin
    option httpchk
    http-check send meth GET uri /replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server pgnode1 192.168.122.98:5432  maxconn 100 check port 8008
    server pgnode2 192.168.122.147:5432 maxconn 100 check port 8008
    server pgnode3 192.168.122.105:5432 maxconn 100 check port 8008
```
 
#### Start HAProxy
 
```bash
sudo systemctl enable haproxy
sudo systemctl start haproxy
sudo systemctl status haproxy
```
 
#### Configure Firewall on pgproxy (if not disabled)
 
Allow traffic to VIP ports using nftables:
 
```bash
sudo nft add table inet filter
sudo nft add chain inet filter input { type filter hook input priority 0\; }
sudo nft add rule inet filter input ip daddr 192.168.122.112 tcp dport { 5000, 5001, 7000 } accept
 
# Save rules persistently
sudo nft list ruleset | sudo tee /etc/nftables.conf
```
 
---
 
### 2.4 Patroni & PostgreSQL Installation (DB Nodes Only)
 
**What is Patroni?**
Patroni is a Python-based HA template for PostgreSQL. It wraps PostgreSQL's built-in replication and uses etcd as an external consensus store to elect a single primary. Only the node holding the etcd leader lock may act as primary — all others are replicas. When the primary fails, the lock expires and a replica acquires it and promotes itself.
 
#### Install Patroni
 
```bash
sudo dnf -y install patroni patroni-etcd watchdog
```
 
#### Create Data Directories
 
```bash
sudo mkdir -p /data/pgsql/16/data
sudo chown -R postgres:postgres /data/pgsql/16
sudo chmod 700 /data/pgsql/16/data
 
sudo mkdir -p /run/postgresql
sudo chown postgres:postgres /run/postgresql
 
# Persist socket directory across reboots
echo 'd /run/postgresql 0775 postgres postgres -' | sudo tee /etc/tmpfiles.d/postgresql.conf
```
 
#### Create Patroni Configuration
 
Create the configuration directory:
 
```bash
sudo mkdir -p /etc/patroni
```
 
Create the configuration file:
 
```bash
sudo nano /etc/patroni/patroni.yml
```
 
Paste the following configuration. The full config is shown once for **pgnode1** — for other nodes, only 5 values change (listed after the config).
 
> **Critical YAML Rules:**
> - Use **spaces only** — no tabs anywhere in the file
> - `use_pg_rewind: true` must be indented under `postgresql:` inside `dcs:`
> - The `initdb:` section is **required** — Patroni will never bootstrap without it
> - Use `/32` (not `/0`) in `pg_hba` replication entries
 
##### `patroni.yml` — pgnode1 (full config)
 
```yaml
scope: postgres
namespace: /db/
name: pgnode1
 
restapi:
    listen: 192.168.122.98:8008
    connect_address: 192.168.122.98:8008
 
etcd3:
    hosts: 192.168.122.98:2379,192.168.122.147:2379,192.168.122.105:2379
 
bootstrap:
    dcs:
        ttl: 30
        loop_wait: 10
        retry_timeout: 10
        maximum_lag_on_failover: 1048576
        postgresql:
            use_pg_rewind: true
 
    initdb:
    - encoding: UTF8
    - data-checksums
 
    pg_hba:
    - host replication replicator 192.168.122.98/32 scram-sha-256
    - host replication replicator 192.168.122.147/32 scram-sha-256
    - host replication replicator 192.168.122.105/32 scram-sha-256
    - host all all 0.0.0.0/0 scram-sha-256
 
postgresql:
    listen: 192.168.122.98:5432
    connect_address: 192.168.122.98:5432
    data_dir: /data/pgsql/16/data
    bin_dir: /usr/pgsql-16/bin
 
    authentication:
        replication:
            username: replicator
            password: replicator
        superuser:
            username: postgres
            password: postgres
 
    parameters:
        unix_socket_directories: '/run/postgresql'
 
watchdog:
    mode: off
    device: /dev/watchdog
    safety_margin: 5
 
tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
```
 
##### For pgnode2 and pgnode3 — change only these 4 lines
 
Everything else stays identical. On each node, replace the pgnode1 values with the node's own hostname and IP:
 
| Line in config              | pgnode2                  | pgnode3                  |
|-----------------------------|--------------------------|--------------------------|
| `name:`                     | `pgnode2`                | `pgnode3`                |
| `restapi.listen:`           | `192.168.122.147:8008`   | `192.168.122.105:8008`   |
| `restapi.connect_address:`  | `192.168.122.147:8008`   | `192.168.122.105:8008`   |
| `postgresql.listen:`        | `192.168.122.147:5432`   | `192.168.122.105:5432`   |
| `postgresql.connect_address:` | `192.168.122.147:5432` | `192.168.122.105:5432`   |
 
#### Disable Standalone PostgreSQL
 
```bash
sudo systemctl stop postgresql-16
sudo systemctl disable postgresql-16
```
 
---
 
### 2.5 Starting the Cluster
 
> **⚠️ Order is Critical:** Always start pgnode1 first. Wait until it becomes Leader before starting other nodes.
 
#### Step 1 — Start Patroni on pgnode1 ONLY
 
```bash
sudo systemctl enable patroni
sudo systemctl start patroni
 
# Watch logs — wait for these messages:
sudo journalctl -u patroni -f
```
 
Wait for:
```
INFO: trying to bootstrap a new cluster
INFO: initialized a new cluster
INFO: no action. I am (pgnode1), the leader with the lock
```
 
#### Step 2 — Verify pgnode1 is Leader
 
```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml list
```
 
Confirm `pgnode1` shows `Role: Leader` before proceeding.
 
#### Step 3 — Start Patroni on pgnode2 and pgnode3
 
```bash
sudo systemctl enable patroni
sudo systemctl start patroni
 
# Watch logs:
sudo journalctl -u patroni -f
```
 
Expected log output:
```
INFO: trying to bootstrap from leader 'pgnode1'
INFO: clone from 'pgnode1' in progress
INFO: bootstrap succeeded
```
 
#### Step 4 — Final Verification
 
```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml list
```
 
Expected output:
```
+----------+------------------+---------+-----------+----+-----------+
| Member   | Host             | Role    | State     | TL | Lag in MB |
+----------+------------------+---------+-----------+----+-----------+
| pgnode1  | 192.168.122.98   | Leader  | running   |  1 |           |
| pgnode2  | 192.168.122.147  | Replica | streaming |  1 |         0 |
| pgnode3  | 192.168.122.105  | Replica | streaming |  1 |         0 |
+----------+------------------+---------+-----------+----+-----------+
```
 
---
 
## 3. Phase 2 — Adding a New Node (Scaling)
 
This section details how to add a new node (e.g., pgnode4) to an **already running** cluster.
 
### 3.1 Adding etcd Member
 
Before starting Patroni on the new node, it must join the etcd cluster so it can participate in distributed consensus.
 
#### On an Existing Node (e.g., pgnode1) — Register New Member
 
```bash
etcdctl --endpoints=$ENDPOINTS member add pgnode4 \
    --peer-urls=http://<new-node-ip>:2380
```
 
This command outputs a configuration snippet — **save it**, you will use it in the next step.
 
#### Configure etcd on the New Node
 
Create `/etc/etcd/etcd.conf` using the output from the command above.
 
> **Critical:** Set `ETCD_INITIAL_CLUSTER_STATE="existing"` (not `"new"`)
 
```ini
ETCD_NAME=pgnode4
ETCD_DATA_DIR="/var/lib/etcd/pgnode4"
ETCD_INITIAL_CLUSTER_STATE="existing"   # <-- Must be "existing", not "new"
# ... rest of config from member add output
```
 
#### Start etcd on New Node
 
```bash
sudo systemctl enable etcd
sudo systemctl start etcd
```
 
Verify health from any existing node:
 
```bash
etcdctl endpoint health --write-out=table --endpoints=$ENDPOINTS
```
 
---
 
### 3.2 Updating Cluster Configuration (Critical Step)
 
> This is the **most important step** that is often missed and causes bootstrap failure.
 
**Why this is required — Theory:**
 
Patroni stores the active cluster configuration in etcd (the DCS), **not just the local file**. When a new node tries to replicate, the Leader checks its active `pg_hba.conf`. This file is **generated by Patroni** based on the configuration stored in etcd. If the new node's IP is not in the etcd-stored config, the Leader will reject the replication connection with a `no pg_hba.conf entry` error, causing bootstrap failure.
 
| Config Location          | When Used                                         |
|--------------------------|---------------------------------------------------|
| `/etc/patroni/patroni.yml` | Node identity and **initial bootstrap only**    |
| etcd DCS config          | **Active runtime config** — source of truth after bootstrap |
 
#### Edit Configuration in etcd
 
Run from **any existing node** with `patronictl` installed:
 
```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml edit-config postgres
```
 
This opens a text editor. Locate the `postgresql` section and add the new node's IP to `pg_hba`:
 
```yaml
postgresql:
  pg_hba:
  - host replication replicator 192.168.122.98/32 scram-sha-256
  - host replication replicator 192.168.122.147/32 scram-sha-256
  - host replication replicator 192.168.122.105/32 scram-sha-256
  - host replication replicator <new-node-ip>/32 scram-sha-256   # Add this line
  - host all all 0.0.0.0/0 scram-sha-256
```
 
Save and exit the editor.
 
#### Verify the Update
 
```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml show-config postgres
```
 
Confirm the new IP appears in the output.
 
#### Reload Leader Configuration
 
Force the leader to apply the new `pg_hba` rules immediately, **without restarting** PostgreSQL:
 
```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml reload postgres <leader-node-name>
```
 
> Using `patronictl reload` applies changes dynamically without restarting the PostgreSQL service, maintaining high availability during scaling operations.
 
---
 
### 3.3 Bootstrapping the New Node
 
#### Prepare Patroni Config on New Node
 
Create `/etc/patroni/patroni.yml` on the new node:
 
```bash
sudo mkdir -p /etc/patroni
sudo nano /etc/patroni/patroni.yml
```
 
Ensure:
- `name` matches the etcd member name (e.g., `pgnode4`)
- `restapi.connect_address` matches the new node's IP
- `postgresql.connect_address` matches the new node's IP
 
#### Create Required Directories
 
```bash
sudo mkdir -p /data/pgsql/16/data
sudo chown -R postgres:postgres /data/pgsql/16
sudo chmod 700 /data/pgsql/16/data
 
sudo mkdir -p /run/postgresql
sudo chown postgres:postgres /run/postgresql
```
 
#### Start Patroni on the New Node
 
```bash
sudo systemctl enable patroni
sudo systemctl start patroni
```
 
#### Monitor Logs
 
```bash
sudo journalctl -u patroni -f
```
 
Look for:
```
INFO: trying to bootstrap from leader 'pgnode1'
INFO: clone from 'pgnode1' in progress
INFO: bootstrap succeeded
```
 
> If you see `no pg_hba.conf entry` → return to [Step 3.2](#32-updating-cluster-configuration-critical-step) and add the IP.
 
#### Verify Cluster Status
 
```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml list
```
 
The new node should appear with `Role: Replica` and `State: streaming`.
 
---
 
### 3.4 Why This Process Matters
 
#### Local vs. Distributed Config
 
The local file `/etc/patroni/patroni.yml` is used **only** for:
- Node identity (`name`, IP addresses)
- Initial cluster bootstrap parameters
 
Once the cluster is running, the **source of truth is etcd**. Changing the local file on one node does **not** update authentication rules on the Leader.
 
#### Authentication Security
 
PostgreSQL requires explicit permission in `pg_hba.conf` for replication users. By managing this via `patronictl edit-config`, you ensure:
 
- Consistency across the entire cluster
- If the Leader restarts, it pulls the correct `pg_hba` rules from etcd automatically
- The new node remains authorized permanently
 
#### Avoiding Downtime
 
Using `patronictl reload` applies changes dynamically without restarting the PostgreSQL service on the leader, maintaining high availability during scaling operations.
 
---
 
## 4. Phase 3 — Operations & Verification
 
### 4.1 Daily Operations
 
#### Check Cluster Health
 
```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml list
```
 
#### Connect for Writes (Primary — always routed to leader)
 
```bash
psql -h 192.168.122.112 -p 5000 -U postgres
```
 
#### Connect for Reads (Replicas — round-robin load balanced)
 
```bash
psql -h 192.168.122.112 -p 5001 -U postgres
```
 
#### Verify Which Node Is Handling a Connection
 
```bash
# Check write connection (should show is_replica = f)
psql -h 192.168.122.112 -p 5000 -U postgres \
  -c "SELECT inet_server_addr() AS node, pg_is_in_recovery() AS is_replica;"
 
# Check read connection (should show is_replica = t)
psql -h 192.168.122.112 -p 5001 -U postgres \
  -c "SELECT inet_server_addr() AS node, pg_is_in_recovery() AS is_replica;"
```
 
#### Test Round-Robin Load Balancing
 
```bash
for i in {1..6}; do
  psql -h 192.168.122.112 -p 5001 -U postgres -c "SELECT inet_server_addr();" -t
done
```
 
#### Check HAProxy Stats Dashboard
 
Open in browser:
```
http://192.168.122.112:7000/
```
 
#### Check Patroni REST API
 
```bash
curl -s http://192.168.122.98:8008/patroni | python3 -m json.tool
```
 
#### Check Replication Lag
 
```bash
psql -h 192.168.122.112 -p 5000 -U postgres \
  -c "SELECT client_addr, state, pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes FROM pg_stat_replication;"
```
 
#### Pause / Resume Auto-Failover (for maintenance)
 
```bash
# Pause — prevents automatic failover during planned maintenance
sudo -u postgres patronictl -c /etc/patroni/patroni.yml pause
 
# Resume when maintenance is complete
sudo -u postgres patronictl -c /etc/patroni/patroni.yml resume
```
 
#### Reinitialize a Replica
 
```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml reinit postgres pgnode2
```
 
---
 
### 4.2 Failover Testing
 
#### Manual Switchover (controlled, zero-downtime)
 
```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml switchover postgres
```
 
Follow prompts to select the new leader.
 
#### Simulate Node Failure (automatic failover)
 
```bash
# On the current leader node — stop Patroni:
sudo systemctl stop patroni
 
# Wait ~30 seconds, then check from another node:
sudo -u postgres patronictl -c /etc/patroni/patroni.yml list
```
 
A new leader should be elected automatically. When you restart the stopped node, it will rejoin as a replica:
 
```bash
sudo systemctl start patroni
sudo journalctl -u patroni -f
```
 
---
 
### 4.3 Troubleshooting
 
#### VIP Not Reachable
 
```bash
# Check Keepalived status
sudo systemctl status keepalived
 
# Check VIP assignment
ip addr show
 
# Verify HAProxy is running
sudo systemctl status haproxy
 
# Check ports are listening on VIP
ss -tlnp | grep -E '5000|5001|7000'
```
 
Make sure firewall on pgproxy allows traffic on ports 5000, 5001, 7000 for the VIP IP.
 
#### Empty Cluster List / Cluster Not Forming
 
```bash
# 1. Check etcd is healthy
etcdctl endpoint health --endpoints=$ENDPOINTS
 
# 2. If stale data exists, clean it
etcdctl del /db/ --prefix --endpoints=$ENDPOINTS
 
# 3. Stop Patroni on ALL nodes
sudo systemctl stop patroni   # run on all nodes
 
# 4. Start pgnode1 FIRST, wait for Leader, then start others
sudo systemctl start patroni  # pgnode1 only first
```
 
#### "waiting for leader to bootstrap"
 
| Cause | Fix |
|-------|-----|
| Missing `initdb:` in `patroni.yml` | Add `initdb:` section under `bootstrap:` |
| Wrong YAML indentation | `use_pg_rewind` must be under `postgresql:` inside `dcs:` |
| Stale etcd key | `etcdctl del /db/ --prefix --endpoints=$ENDPOINTS` |
| Data dir not empty | `rm -rf /data/pgsql/16/data/*` |
 
#### Replication Error — "no pg_hba.conf entry"
 
This means the new node's IP is not in the cluster's active pg_hba configuration stored in etcd.
 
```bash
# Fix: add the new node's IP via edit-config
sudo -u postgres patronictl -c /etc/patroni/patroni.yml edit-config postgres
 
# Then reload the leader
sudo -u postgres patronictl -c /etc/patroni/patroni.yml reload postgres <leader-name>
```
 
#### PostgreSQL Connection Refused
 
```bash
# Check if standalone postgresql-16 service is interfering (must be disabled)
sudo systemctl status postgresql-16
sudo systemctl stop postgresql-16
sudo systemctl disable postgresql-16
 
# Fix data dir ownership
sudo chown -R postgres:postgres /data/pgsql/16
sudo chmod 700 /data/pgsql/16/data
```
 
#### etcd Health Issues
 
```bash
# Check all endpoints
etcdctl endpoint health --endpoints=$ENDPOINTS
etcdctl endpoint status --write-out=table --endpoints=$ENDPOINTS
 
# Check etcd keys
etcdctl get /db/ --prefix --endpoints=$ENDPOINTS
```
 
If unhealthy, check network connectivity on ports 2379 and 2380 between all DB nodes.
 
#### $ENDPOINTS Variable Empty
 
```bash
# Must include http:// prefix
export ENDPOINTS="http://192.168.122.98:2379,http://192.168.122.147:2379,http://192.168.122.105:2379"
```
 
#### Useful Diagnostic Commands
 
```bash
# Live Patroni logs
sudo journalctl -u patroni -f
 
# Check all cluster-related ports are listening
ss -tlnp | grep -E '5432|8008|2379|2380'
 
# Check replication lag
psql -h 192.168.122.112 -p 5000 -U postgres \
  -c "SELECT client_addr, state, sent_lsn, replay_lsn FROM pg_stat_replication;"
 
# View etcd keys stored by Patroni
etcdctl get /db/ --prefix --endpoints=$ENDPOINTS
```
 
---
 
## 5. Appendix — Configuration References
 
### 5.1 patroni.yml — Key Settings Explained
 
The full working config is in **Section 2.4**. This table explains what each important setting does.
 
| Setting | Value | Purpose |
|---------|-------|---------|
| `scope` | `postgres` | Cluster name — must be identical on all nodes |
| `namespace` | `/db/` | etcd key prefix — all cluster data stored under this path |
| `name` | `pgnode1` | **Unique per node** — identifies this node in the cluster |
| `restapi.listen` | `<node-ip>:8008` | HAProxy calls this to check if node is primary or replica |
| `restapi.connect_address` | `<node-ip>:8008` | Address other nodes use to reach this REST API |
| `etcd3.hosts` | all 3 node IPs | Same on all nodes — Patroni connects to any available etcd |
| `bootstrap.dcs.ttl` | `30` | Seconds before leader lock expires if primary goes silent |
| `use_pg_rewind` | `true` | Allows a former primary to re-sync as replica without full rebuild |
| `initdb` | UTF8 + checksums | **Required** — tells Patroni how to initialize the data directory |
| `pg_hba` | `/32` entries | Replication whitelist — use `/32` (exact IP), not `/0` |
| `postgresql.listen` | `<node-ip>:5432` | PostgreSQL listens on this IP |
| `postgresql.connect_address` | `<node-ip>:5432` | Address other nodes use to reach PostgreSQL |
| `data_dir` | `/data/pgsql/16/data` | PostgreSQL data directory — must be empty before first start |
| `bin_dir` | `/usr/pgsql-16/bin` | Location of pg_ctl, pg_basebackup, etc. |
| `watchdog.mode` | `off` | Watchdog disabled — automatic failover still works via etcd |
 
### 5.2 All Per-Node Differences — One Place
 
| Setting | pgnode1 | pgnode2 | pgnode3 |
|---------|---------|---------|---------|
| Patroni `name` | `pgnode1` | `pgnode2` | `pgnode3` |
| `restapi.listen` / `connect_address` | `192.168.122.98:8008` | `192.168.122.147:8008` | `192.168.122.105:8008` |
| `postgresql.listen` / `connect_address` | `192.168.122.98:5432` | `192.168.122.147:5432` | `192.168.122.105:5432` |
| etcd `ETCD_NAME` | `pgnode1` | `pgnode2` | `pgnode3` |
| etcd `ETCD_DATA_DIR` | `/var/lib/etcd/pgnode1` | `/var/lib/etcd/pgnode2` | `/var/lib/etcd/pgnode3` |
| etcd peer/client URL IP | `192.168.122.98` | `192.168.122.147` | `192.168.122.105` |
 
### 5.3 Startup Order Checklist
 
```
[ ] 1. etcd started and healthy on all 3 DB nodes
[ ] 2. Keepalived started on pgproxy (VIP assigned)
[ ] 3. HAProxy started on pgproxy
[ ] 4. postgresql-16 service stopped and disabled on all DB nodes
[ ] 5. Patroni started on pgnode1 ONLY → wait for "leader with the lock"
[ ] 6. Verify pgnode1 is Leader via: patronictl list
[ ] 7. Patroni started on pgnode2
[ ] 8. Patroni started on pgnode3
[ ] 9. Final: patronictl list shows 1 Leader + 2 streaming Replicas
```
 
### 5.4 Quick Reference — Key Commands
 
```bash
# Cluster status
sudo -u postgres patronictl -c /etc/patroni/patroni.yml list
 
# Edit runtime cluster config (pg_hba, parameters)
sudo -u postgres patronictl -c /etc/patroni/patroni.yml edit-config postgres
 
# Show current runtime config
sudo -u postgres patronictl -c /etc/patroni/patroni.yml show-config postgres
 
# Reload config on a node (no restart)
sudo -u postgres patronictl -c /etc/patroni/patroni.yml reload postgres <node-name>
 
# Controlled switchover
sudo -u postgres patronictl -c /etc/patroni/patroni.yml switchover postgres
 
# Reinitialize a replica from scratch
sudo -u postgres patronictl -c /etc/patroni/patroni.yml reinit postgres <node-name>
 
# Pause / resume automatic failover
sudo -u postgres patronictl -c /etc/patroni/patroni.yml pause
sudo -u postgres patronictl -c /etc/patroni/patroni.yml resume
 
# etcd health check
etcdctl endpoint health --write-out=table --endpoints=$ENDPOINTS
 
# Clean stale etcd Patroni data (only if cluster is fully stopped)
etcdctl del /db/ --prefix --endpoints=$ENDPOINTS
```
 
---
 