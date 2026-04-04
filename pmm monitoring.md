# SOP: PMM Monitoring Setup for Patroni PostgreSQL Cluster

**Version:** 1.0  
**Date:** April 2026  
**Environment:** Rocky Linux VMs, Ubuntu Host (PMM Server)

---

## Infrastructure Overview

| Hostname | IP Address | Role | Components |
|----------|------------|------|------------|
| pgnode1 | 192.168.122.98 | Database | etcd, Patroni, PostgreSQL, pgBackRest client |
| pgnode2 | 192.168.122.147 | Database | etcd, Patroni, PostgreSQL, pgBackRest client |
| pgnode3 | 192.168.122.105 | Database | etcd, Patroni, PostgreSQL, pgBackRest client |
| pgnode4 | 192.168.122.252 | Database | etcd, Patroni, PostgreSQL, pgBackRest client |
| pgproxy | 192.168.122.181 | Proxy + Backup | HAProxy, Keepalived, pgBackRest repository |
| Ubuntu Host | 192.168.122.1 | Monitoring | PMM Server (Docker) |

---

## Step 1 — Run PMM Server on Ubuntu Host (Docker)

```bash
docker pull percona/pmm-server:3

docker volume create pmm-data

docker run -d \
  --name pmm-server \
  --restart always \
  -p 443:443 \
  -p 8443:8443 \
  -p 80:80 \
  -v pmm-data:/srv \
  percona/pmm-server:3
```

Access the UI at `https://192.168.122.1:8443` — default login: `admin / admin`

> Allow port 8443 through firewall: `sudo ufw allow 8443/tcp`

---

## Step 2 — Install PMM Client on All VMs (Rocky Linux)

Run on **each VM** (pgnode1–4 and pgproxy):

```bash
sudo dnf install -y https://repo.percona.com/yum/percona-release-latest.noarch.rpm
sudo percona-release enable pmm3-client
sudo dnf install -y pmm-client
```

> **Note:** Always use `dnf` on Rocky Linux. Never use `apt`.

---

## Step 3 — Create PostgreSQL Monitoring User

Run **once on the primary node** only:

```bash
sudo -u postgres psql
```

```sql
CREATE USER pmm_monitor WITH PASSWORD 'your_password' CONNECTION LIMIT 10;
GRANT pg_monitor TO pmm_monitor;
\q
```

---

## Step 4 — Register and Add Services Per VM

### DB Nodes (pgnode1–4)

After installing the PMM client, run these **2 commands** on each DB node:

```bash
# Command 1 — Register node with PMM Server
sudo pmm-admin config \
  --server-insecure-tls \
  --server-url=https://admin:admin@192.168.122.1:8443 \
  --force \
  <NODE_IP> generic <NODE_NAME>

# Command 2 — Add PostgreSQL service
pmm-admin add postgresql \
  --query-source=perfschema \
  --username=pmm_monitor \
  --password='your_password' \
  --service-name=<NODE_NAME> \
  --host=<NODE_IP> \
  --port=5432 \
  --environment=OP-DB
```

#### pgnode1
```bash
sudo pmm-admin config --server-insecure-tls --server-url=https://admin:admin@192.168.122.1:8443 --force 192.168.122.98 generic pgnode1

pmm-admin add postgresql --query-source=perfschema --username=pmm_monitor --password='your_password' --service-name=pgnode1 --host=192.168.122.98 --port=5432 --environment=OP-DB
```

#### pgnode2
```bash
sudo pmm-admin config --server-insecure-tls --server-url=https://admin:admin@192.168.122.1:8443 --force 192.168.122.147 generic pgnode2

pmm-admin add postgresql --query-source=perfschema --username=pmm_monitor --password='your_password' --service-name=pgnode2 --host=192.168.122.147 --port=5432 --environment=OP-DB
```

#### pgnode3
```bash
sudo pmm-admin config --server-insecure-tls --server-url=https://admin:admin@192.168.122.1:8443 --force 192.168.122.105 generic pgnode3

pmm-admin add postgresql --query-source=perfschema --username=pmm_monitor --password='your_password' --service-name=pgnode3 --host=192.168.122.105 --port=5432 --environment=OP-DB
```

#### pgnode4
```bash
sudo pmm-admin config --server-insecure-tls --server-url=https://admin:admin@192.168.122.1:8443 --force 192.168.122.252 generic pgnode4

pmm-admin add postgresql --query-source=perfschema --username=pmm_monitor --password='your_password' --service-name=pgnode4 --host=192.168.122.252 --port=5432 --environment=OP-DB
```

---

### pgproxy — OS Monitoring Only

Only **1 command** needed. No PostgreSQL runs here — `node_exporter` activates automatically on registration and collects all OS metrics (CPU, memory, disk, network):

```bash
sudo pmm-admin config --server-insecure-tls --server-url=https://admin:admin@192.168.122.1:8443 --force 192.168.122.181 generic pgproxy
```

---

## Step 5 — Verify Each Node

After setup on each node:

```bash
sudo pmm-admin list
```

**Expected output on DB nodes:**

```
Service type    Service name    Address and port
PostgreSQL      pgnode1         192.168.122.98:5432

Agent type                    Status      Metrics Mode
pmm_agent                     Connected
node_exporter                 Running     push
postgres_exporter             Running     push
postgresql_pgstatements_agent Running
vmagent                       Running     push
```

**Expected output on pgproxy:**

```
Agent type       Status       Metrics Mode
pmm_agent        Connected
node_exporter    Running      push
vmagent          Running      push
```

---

## Step 6 — Access Dashboards in PMM UI

Navigate to `https://192.168.122.1:8443`

| What to monitor | Dashboard path |
|-----------------|---------------|
| PostgreSQL performance | Dashboards > PostgreSQL > Query Details |
| OS metrics (all nodes) | Dashboards > Operating System > Node Summary |
| Patroni cluster health | Dashboards > Experimental > PostgreSQL Patroni Details |
| Query analytics | PMM > Query Analytics (QAN) |

---

## Notes

- PMM 2 reached end-of-life October 2025 — this SOP uses **PMM 3**
- PostgreSQL on these nodes listens on the node's IP, not `127.0.0.1` — always use the node IP in `--host`
- The `--force` flag is required if the node was previously registered to avoid "node already exists" errors
- OS-level metrics require no extra command — `node_exporter` starts automatically on `pmm-admin config`
- The warning `"Could not detect pg_stat_monitor, falling back to pg_stat_statements"` is harmless
