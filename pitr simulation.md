# PostgreSQL PITR (Point-in-Time Recovery) Simulation SOP
## With Patroni + pgBackRest on a 4-Node HA Cluster

> **Version:** 1.0 | **Date:** March 2026
> **Stack:** PostgreSQL 16 · Patroni · pgBackRest
> **Cluster:** pgnode1 (98) · pgnode2 (147) · pgnode3 (105) · pgnode4 (252) · pgproxy (181)

---

## Table of Contents

- [1. What is PITR and Why It Matters](#1-what-is-pitr-and-why-it-matters)
- [2. The Golden Rule of PITR](#2-the-golden-rule-of-pitr)
- [3. Phase 1 — Create Test Data & Safe Point](#3-phase-1--create-test-data--safe-point)
- [4. Phase 2 — Simulate the Accident](#4-phase-2--simulate-the-accident)
- [5. Phase 3 — Perform PITR Recovery](#5-phase-3--perform-pitr-recovery)
- [6. Phase 4 — Restore the Cluster](#6-phase-4--restore-the-cluster)
- [7. Verification](#7-verification)
- [8. Lessons Learned & Common Mistakes](#8-lessons-learned--common-mistakes)
- [9. Quick Reference](#9-quick-reference)

---

## 1. What is PITR and Why It Matters

PITR (Point-in-Time Recovery) lets you restore a PostgreSQL database to any exact moment in the past using a base backup plus WAL (Write-Ahead Log) archives.

### Why streaming replication alone is not enough

Streaming replication copies every change — including mistakes — to all replicas **instantly**. If you run `DROP TABLE` on the primary, all replicas reflect that drop within seconds. There is nowhere to recover from within the cluster itself.

pgBackRest solves this by:
- Storing physical base backups independently of the cluster
- Continuously archiving WAL files to the repository
- Allowing replay of WAL up to any specific timestamp

### PITR flow

```
Base Backup (pgBackRest)
        +
WAL Archives (continuous)
        │
        ▼
Restore backup → Replay WAL → Stop at target time → Promote
```

---

## 2. The Golden Rule of PITR

> **ALL nodes must be stopped before starting the restore. If any node is running and holds the etcd leader lock, the restored node will sync to it and lose the recovery.**

The restored node must be the **first and only** node to start. It will win the leader race, promote itself, and only then should replicas be brought back.

---

## 3. Phase 1 — Create Test Data & Safe Point

### Step 1 — Create test table and insert safe data

Connect to the primary (port 5000):

```bash
psql -h 192.168.122.112 -p 5000 -U postgres -d dvdrental
```

```sql
CREATE TABLE pitr_test (
    id         SERIAL PRIMARY KEY,
    message    TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

INSERT INTO pitr_test (message) VALUES
    ('row 1 - safe data'),
    ('row 2 - safe data'),
    ('row 3 - safe data');

SELECT * FROM pitr_test;
```

### Step 2 — Take a full backup checkpoint

**Run on pgproxy:**

```bash
sudo -u postgres pgbackrest --stanza=postgres --log-level-console=info backup --type=full
```

Wait for completion:

```
INFO: backup command end: completed successfully
```

### Step 3 — Note the safe timestamp

```bash
psql -h 192.168.122.112 -p 5000 -U postgres -d dvdrental \
  -c "SELECT NOW();"
```

**Write this timestamp down.** This is your recovery target. Example:
```
2026-03-31 11:06:03
```

> Wait at least 60 seconds after noting the time — gives WAL time to archive to pgproxy.

---

## 4. Phase 2 — Simulate the Accident

### Step 1 — Insert bad data after the safe point

```bash
psql -h 192.168.122.112 -p 5000 -U postgres -d dvdrental
```

```sql
INSERT INTO pitr_test (message) VALUES
    ('row 4 - this should be LOST after recovery'),
    ('row 5 - this should be LOST after recovery');
```

### Step 2 — Drop the table (the accident)

```sql
DROP TABLE pitr_test;

-- Confirm it is gone:
SELECT * FROM pitr_test;
-- ERROR: relation "pitr_test" does not exist
```

### Step 3 — Confirm damage replicated everywhere

```bash
# Replica also has no table — replication propagated the DROP:
psql -h 192.168.122.112 -p 5001 -U postgres -d dvdrental \
  -c "SELECT * FROM pitr_test;"
# ERROR: relation "pitr_test" does not exist
```

This confirms why replication alone cannot save you.

---

## 5. Phase 3 — Perform PITR Recovery

### Step 1 — Pause Patroni auto-failover

**From any DB node:**

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml pause
```

### Step 2 — Stop Patroni on ALL nodes

```bash
# On pgnode1:
sudo systemctl stop patroni

# On pgnode2:
sudo systemctl stop patroni

# On pgnode3:
sudo systemctl stop patroni

# On pgnode4:
sudo systemctl stop patroni
```

Verify all stopped:

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml list
# All nodes should show: stopped
```

> If any node is still running and holds the leader lock, the restore will fail — the restored node will sync to it instead of promoting.

### Step 3 — Clear data directory on the restore target (pgnode1)

```bash
# On pgnode1:
sudo find /data/pgsql/16/data/ -mindepth 1 -delete

# Verify completely empty:
sudo ls /data/pgsql/16/data/
# Must show nothing
```

> Use `find -mindepth 1 -delete` not `rm -rf *` — the glob expansion misses hidden files and protected subdirectories.

### Step 4 — Run the restore on pgnode1

```bash
# On pgnode1:
sudo -u postgres pgbackrest --stanza=postgres restore \
    --type=time \
    --target="2026-03-31 11:06:03" \
    --target-action=promote \
    --log-level-console=info
```

> Replace the timestamp with your actual safe point from Phase 1 Step 3.

Wait for:
```
INFO: repo1: restore backup set 20260331-XXXXXXF
INFO: restore size = XXmb, file total = XXXX
INFO: restore command end: completed successfully
```

### Step 5 — Verify recovery.signal and postgresql.auto.conf

```bash
# On pgnode1:
sudo ls /data/pgsql/16/data/recovery.signal
sudo cat /data/pgsql/16/data/postgresql.auto.conf
```

`postgresql.auto.conf` must contain:

```ini
restore_command = 'pgbackrest --stanza=postgres archive-get %f %p'
recovery_target_time = '2026-03-31 11:06:03'
recovery_target_action = 'promote'
```

If `recovery.signal` is missing, create it:

```bash
sudo -u postgres touch /data/pgsql/16/data/recovery.signal
```

If `postgresql.auto.conf` is missing the recovery settings, add them:

```bash
sudo -u postgres tee -a /data/pgsql/16/data/postgresql.auto.conf << 'EOF'
restore_command = 'pgbackrest --stanza=postgres archive-get %f %p'
recovery_target_time = '2026-03-31 11:06:03'
recovery_target_action = 'promote'
EOF
```

> Do NOT add `recovery_target_exclusive` — it is not a valid PostgreSQL parameter and will prevent startup.

### Step 6 — Fix ownership

```bash
# On pgnode1:
sudo chown -R postgres:postgres /data/pgsql/16/data
sudo chmod 700 /data/pgsql/16/data
```

### Step 7 — Start PostgreSQL directly to watch WAL replay

```bash
# On pgnode1:
sudo -u postgres /usr/pgsql-16/bin/pg_ctl \
    -D /data/pgsql/16/data \
    -l /var/lib/pgsql/pg_restore.log \
    start
```

Watch the log:

```bash
sudo tail -f /data/pgsql/16/data/log/$(sudo ls -t /data/pgsql/16/data/log/ | head -1)
```

Wait for:

```
LOG: starting point-in-time recovery to 2026-03-31 11:06:03
LOG: restored log file "0000000X..." from archive
LOG: archive recovery complete
LOG: database system is ready to accept connections
```

### Step 8 — Verify recovered data BEFORE handing to Patroni

```bash
psql -h 192.168.122.98 -p 5432 -U postgres -d dvdrental \
  -c "SELECT * FROM pitr_test;"
```

Expected — only the 3 safe rows, no rows 4 or 5, table exists:

```
 id |      message      |         created_at
----+-------------------+----------------------------
  1 | row 1 - safe data | 2026-03-31 11:03:09
  2 | row 2 - safe data | 2026-03-31 11:03:09
  3 | row 3 - safe data | 2026-03-31 11:03:09
```

> Confirm the data here before proceeding. Once you hand control to Patroni there is no going back without running the restore again.

### Step 9 — Stop PostgreSQL and hand to Patroni

```bash
# On pgnode1:
sudo -u postgres /usr/pgsql-16/bin/pg_ctl \
    -D /data/pgsql/16/data \
    stop
```

---

## 6. Phase 4 — Restore the Cluster

### Step 1 — Clear stale etcd leader lock

```bash
# On pgnode1:
etcdctl del /db/ --prefix --endpoints=$ENDPOINTS
```

### Step 2 — Resume Patroni auto-failover

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml resume
```

### Step 3 — Start Patroni on pgnode1 ONLY

```bash
# On pgnode1:
sudo systemctl start patroni
sudo journalctl -u patroni -f
```

Watch for:

```
INFO: Lock owner: None; I am pgnode1
INFO: promoted self to leader by acquiring session lock
INFO: no action. I am (pgnode1), the leader with the lock
```

### Step 4 — Confirm pgnode1 is Leader before anything else

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml list
```

pgnode1 must show `Leader` and `running`. Do not start replicas until this is confirmed.

### Step 5 — Reinitialize all replicas

Use `patronictl reinit` — it wipes the data dir and clones from the leader cleanly, handling ownership and pg_hba correctly:

```bash
# From pgnode1 — reinit each replica:
sudo -u postgres patronictl -c /etc/patroni/patroni.yml reinit postgres pgnode2 --force
```

Wait for pgnode2 to show `streaming`, then:

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml reinit postgres pgnode3 --force
```

Wait for pgnode3 to show `streaming`, then:

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml reinit postgres pgnode4 --force
```

If `reinit` fails due to leftover files, do a manual wipe on the failing node:

```bash
# On the failing replica node:
sudo systemctl stop patroni
sudo find /data/pgsql/16/data/ -mindepth 1 -delete
sudo chown -R postgres:postgres /data/pgsql/16
sudo chmod 700 /data/pgsql/16/data
sudo systemctl start patroni
sudo journalctl -u patroni -f
```

---

## 7. Verification

### Check full cluster status

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml list
```

Expected — all nodes on same timeline, all replicas streaming, lag 0:

```
+---------+-----------------+---------+-----------+----+-----------+
| Member  | Host            | Role    | State     | TL | Lag in MB |
+---------+-----------------+---------+-----------+----+-----------+
| pgnode1 | 192.168.122.98  | Leader  | running   | 18 |           |
| pgnode2 | 192.168.122.147 | Replica | streaming | 18 |         0 |
| pgnode3 | 192.168.122.105 | Replica | streaming | 18 |         0 |
| pgnode4 | 192.168.122.252 | Replica | streaming | 18 |         0 |
+---------+-----------------+---------+-----------+----+-----------+
```

> Timeline number will be higher than before PITR — this is correct. Each recovery creates a new timeline.

### Confirm recovered data on primary

```bash
psql -h 192.168.122.112 -p 5000 -U postgres -d dvdrental \
  -c "SELECT * FROM pitr_test;"
```

### Confirm recovered data on replicas

```bash
psql -h 192.168.122.112 -p 5001 -U postgres -d dvdrental \
  -c "SELECT * FROM pitr_test;"
```

### Confirm recovered data on pgnode4 directly

```bash
psql -h 192.168.122.252 -p 5432 -U postgres -d dvdrental \
  -c "SELECT * FROM pitr_test;"
```

All three should return only the 3 safe rows.

### Insert new data and verify replication is live

```bash
psql -h 192.168.122.112 -p 5000 -U postgres -d dvdrental
```

```sql
INSERT INTO pitr_test (message) VALUES
    ('row 4 - inserted after PITR recovery'),
    ('row 5 - replication check');

SELECT * FROM pitr_test;
```

Then verify on pgnode4:

```bash
psql -h 192.168.122.252 -p 5432 -U postgres -d dvdrental \
  -c "SELECT * FROM pitr_test;"
```

All 5 rows should appear — confirming live replication is working post-recovery.

---

## 8. Lessons Learned & Common Mistakes

### Mistake 1 — Not stopping all nodes before restore

If any node is running when you start the restored node, Patroni will make the restored node sync to the running leader — wiping the recovery. **All nodes must be stopped.**

### Mistake 2 — Running restore from pgproxy

```
ERROR [072]: restore command must be run on the PostgreSQL host
```

The `restore` command must run **on the DB node itself** (pgnode1), not on pgproxy. pgproxy is only the repository host — it can read backups but cannot write to a remote filesystem.

| Command | Where to run |
|---------|-------------|
| `backup` | pgproxy |
| `restore` | DB node (pgnode1) |
| `check` | pgproxy |
| `info` | pgproxy |

### Mistake 3 — Using recovery_target_exclusive parameter

```
FATAL: unrecognized configuration parameter "recovery_target_exclusive"
```

This is not a valid PostgreSQL parameter. Do not add it to `postgresql.auto.conf`. The `--target-exclusive` flag is handled by pgBackRest during restore, not by PostgreSQL directly.

### Mistake 4 — Using rm -rf * to wipe data dir

The glob `*` misses hidden files and some protected subdirectories. Always use:

```bash
sudo find /data/pgsql/16/data/ -mindepth 1 -delete
```

### Mistake 5 — Editing local patroni.yml for pg_hba changes

The local `/etc/patroni/patroni.yml` is only used during initial bootstrap. After that, pg_hba is managed via:

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml edit-config postgres
```

Editing the local file has no effect on a running cluster.

### Mistake 6 — Panicking about Patroni log errors

Errors like `Can not fetch local timeline from replication connection` and `no pg_hba.conf entry for 127.0.0.1` look scary but are Patroni's self health check failing — not the actual replication stream. Always verify actual state with:

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml list
```

If it shows `streaming` with `Lag: 0` — data is replicating correctly regardless of log noise.

---

## 9. Quick Reference

### PITR recovery command

```bash
# On the target DB node (not pgproxy):
sudo -u postgres pgbackrest --stanza=postgres restore \
    --type=time \
    --target="YYYY-MM-DD HH:MM:SS" \
    --target-action=promote \
    --log-level-console=info
```

### Full PITR checklist

```
[ ] 1.  Safe timestamp noted and written down
[ ] 2.  Full backup taken before or after safe point
[ ] 3.  Accident simulated (DROP TABLE)
[ ] 4.  patronictl pause — auto-failover frozen
[ ] 5.  Patroni stopped on ALL nodes
[ ] 6.  Data dir wiped on restore target (find -mindepth 1 -delete)
[ ] 7.  pgbackrest restore command run on DB node (not pgproxy)
[ ] 8.  recovery.signal exists in data dir
[ ] 9.  postgresql.auto.conf has restore_command and recovery_target_time
[ ] 10. Ownership fixed (chown -R postgres)
[ ] 11. PostgreSQL started directly (pg_ctl) to watch WAL replay
[ ] 12. Data verified BEFORE handing to Patroni
[ ] 13. PostgreSQL stopped (pg_ctl stop)
[ ] 14. etcd leader lock cleared (etcdctl del /db/ --prefix)
[ ] 15. patronictl resume
[ ] 16. Patroni started on restore node ONLY — wait for Leader
[ ] 17. Replicas reinit'd one at a time (patronictl reinit --force)
[ ] 18. Final: all nodes streaming, same TL, lag 0
[ ] 19. New data inserted on primary, verified on all replicas
```

### Key commands

```bash
# Check backup availability
sudo -u postgres pgbackrest --stanza=postgres info

# Take a full backup before PITR test
sudo -u postgres pgbackrest --stanza=postgres backup --type=full

# Restore to specific time (run on DB node)
sudo -u postgres pgbackrest --stanza=postgres restore \
    --type=time --target="YYYY-MM-DD HH:MM:SS" \
    --target-action=promote --log-level-console=info

# Clear etcd leader lock
etcdctl del /db/ --prefix --endpoints=$ENDPOINTS

# Reinit a replica
sudo -u postgres patronictl -c /etc/patroni/patroni.yml reinit postgres <nodename> --force

# Force wipe data dir
sudo find /data/pgsql/16/data/ -mindepth 1 -delete
```

---

*PITR SOP — PostgreSQL 16 HA Cluster · pgnode1 · pgnode2 · pgnode3 · pgnode4 · repo: pgproxy*
