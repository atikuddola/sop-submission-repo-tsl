# pgpool-II Setup & Configuration Guide

<img width="691" height="321" alt="pgpool drawio" src="https://github.com/user-attachments/assets/f7c13027-f677-4a8c-a784-5800830298c7" />


## Prerequisites

### Enable CRB Repository & Install Dependencies

```bash
sudo dnf config-manager --set-enabled crb
sudo dnf repolist | grep crb
sudo dnf makecache
sudo dnf install -y libmemcached-awesome
```

---

## Installation

### Install pgpool-II Repository & Packages

```bash
# Add pgpool-II YUM repository
sudo dnf install -y https://www.pgpool.net/yum/rpms/4.5/redhat/rhel-9-x86_64/pgpool-II-release-4.5-1.noarch.rpm

# Install pgpool-II for PostgreSQL 16
sudo dnf install -y pgpool-II-pg16

# Install pgpool-II extensions
sudo dnf install -y pgpool-II-pg16-extensions

# Verify repositories
sudo dnf repolist
```

### Enable & Start pgpool Service

```bash
sudo systemctl enable pgpool
sudo systemctl start pgpool
sudo systemctl status pgpool
```

---

## PostgreSQL Configuration

### Create pgpool Role on PostgreSQL

```sql
CREATE ROLE pgpool WITH LOGIN PASSWORD 'strong_password';
```

> **Note:** Grant this user access in both `pg_hba.conf` files (on master and replica nodes).

---

## pgpool-II Main Configuration (`pgpool.conf`)

### Clustering Mode

```ini
backend_clustering_mode = 'streaming_replication'
```

### Connection Settings

```ini
listen_addresses = '*'
port = 9999
unix_socket_directories = '/tmp'
reserved_connections = 2
```

### PCP (Admin Interface) Settings

```ini
pcp_listen_addresses = '*'
pcp_port = 9898
pcp_socket_dir = '/tmp'
listen_backlog_multiplier = 2
serialize_accept = off
```

### Backend Node Configuration

```ini
# Master Node
backend_hostname0 = '192.168.226.128'
backend_port0 = 5432
backend_weight0 = 0
backend_data_directory0 = '/data/pgsql/16/data'
backend_flag0 = 'ALLOW_TO_FAILOVER'
backend_application_name0 = 'master'

# Replica Node
backend_hostname1 = '192.168.226.130'
backend_port1 = 5432
backend_weight1 = 1
backend_data_directory1 = '/data/pgsql/16/data'
backend_flag1 = 'ALLOW_TO_FAILOVER'
backend_application_name1 = 'replica1'
```

> **Weight Note:** Master has `weight = 0` so read queries are not sent to it. Replica has `weight = 1` to receive load-balanced SELECT queries.

### Health Check & Streaming Replication Check

```ini
sr_check_user = 'pgpool'
sr_check_password = 'strong_password'
sr_check_database = 'postgres'

health_check_period = 10
health_check_user = 'pgpool'
health_check_password = ''
health_check_database = 'postgres'

pool_passwd = 'pool_passwd'
```

---

## Access Control

### Configure `pool_hba.conf`

Add entries to allow client connections through pgpool:

```
host    all    all    192.168.109.128/32    scram-sha-256
host    all    all    192.168.109.129/32    scram-sha-256
```

---

## PCP Admin User Setup

### Add a PCP User

Run as the `postgres` user:

```bash
echo "atik:$(pg_md5 asd)" >> /etc/pgpool-II/pcp.conf
```

---

## Password File Setup (`pool_passwd`)

### Step 1 — Create the Encryption Key File

```bash
# Run as postgres user
echo "strong_password" > ~/.pgpoolkey
chmod 600 ~/.pgpoolkey
```

### Step 2 — Register pgpool User Password

```bash
pg_enc -m -k ~/.pgpoolkey -u pgpool -p
```

### Step 3 — (Optional) Register Additional Users

```bash
pg_enc -m -k ~/.pgpoolkey -u samrat -p
```

### Step 4 — Verify the Password File

```bash
cat /etc/pgpool-II/pool_passwd
```

---

## Fixing / Rebuilding the Password File

Use this section if the password file becomes corrupted or authentication fails.

### Step 1 — Remove the Old Password File

```bash
rm /etc/pgpool-II/pool_passwd
```

### Step 2 — Verify the Key File Location & Permissions

pgpool expects the key file at `/var/lib/pgsql/.pgpoolkey`:

```bash
ls -l /var/lib/pgsql/.pgpoolkey
```

The file **must** be owned by `postgres` with `600` permissions:

```bash
chmod 600 /var/lib/pgsql/.pgpoolkey
chown postgres:postgres /var/lib/pgsql/.pgpoolkey
```

### Step 3 — Recreate `pool_passwd`

```bash
pg_enc -m -k ~/.pgpoolkey -u pgpool -p
```

> Enter the **real** PostgreSQL password for the `pgpool` user when prompted.

### Step 4 — Fix File Permissions (Critical on AlmaLinux / SELinux)

Exit the `postgres` user and run as root:

```bash
sudo chown postgres:postgres /etc/pgpool-II/pool_passwd
sudo chmod 600 /etc/pgpool-II/pool_passwd
sudo restorecon -Rv /etc/pgpool-II
```

> `restorecon` is required to apply correct SELinux labels on AlmaLinux/RHEL systems.

### Step 5 — Restart pgpool

```bash
sudo systemctl restart pgpool-II
```

---

## Verification & Testing

### Check Pool Nodes Status

```bash
sudo -i -u postgres
psql -h 192.168.122.247 -p 9999 -U pgpool -d postgres -c "SHOW pool_nodes;"
```

A healthy output shows both nodes with `status = up`.

### Connect to PostgreSQL via pgpool

```bash
psql -h 192.168.122.247 -p 9999 -U pgpool -d postgres
```

---

## Troubleshooting

### Reattach a Down Node

If `SHOW pool_nodes` shows a node as `down`, reattach it using the PCP interface:

```bash
# As postgres user
pcp_attach_node -U samrat -W -h 192.168.109.130 -p 9898 -n 1
```

Then restart pgpool if needed:

```bash
sudo systemctl restart pgpool
```
