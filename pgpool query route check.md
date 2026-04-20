# SOP: Verify Pgpool Query Routing via Logs

---

## Prerequisites — Enable Logging

Edit `/etc/pgpool-II/pgpool.conf`:
```ini
# Logging
logging_collector = on
log_destination = 'stderr'
log_directory = '/var/log/pgpool'
log_filename = 'pgpool-%Y-%m-%d_%H%M%S.log'
log_statement = on
log_per_node_statement = on
log_connections = on
log_disconnections = on

# Load Balancing
load_balance_mode = on
statement_level_load_balance = on
```

Apply:
```bash
systemctl restart pgpool-II.service
```

---

## Step 1 — Start Log Monitor

**Option A** — Full view (client + routing):
```bash
tail -f /var/log/pgpool/pgpool-*.log | \
grep -E "statement: (SELECT|INSERT|UPDATE|DELETE)" | \
sed -E '
s/DB node id: 0/\x1b[32m[MASTER]\x1b[0m/;
s/DB node id: 1/\x1b[34m[REPLICA]\x1b[0m/;
s/statement: SELECT/\x1b[36m[READ]\x1b[0m SELECT/;
s/statement: INSERT/\x1b[31m[WRITE]\x1b[0m INSERT/;
s/statement: UPDATE/\x1b[31m[WRITE]\x1b[0m UPDATE/;
s/statement: DELETE/\x1b[31m[WRITE]\x1b[0m DELETE/;
'
```

**Option B** — Clean execution view (recommended):
```bash
tail -f /var/log/pgpool/pgpool-*.log | \
grep "backend pid" | \
grep -E "SELECT|INSERT|UPDATE|DELETE" | \
sed -E '
s/DB node id: 0/\x1b[32m[MASTER]\x1b[0m/;
s/DB node id: 1/\x1b[34m[REPLICA]\x1b[0m/;
s/SELECT/\x1b[36m[READ] SELECT/;
s/INSERT|UPDATE|DELETE/\x1b[31m[WRITE] &/g;
'
```

---

## Step 2 — Run Verification Queries

**READ test** (run via Pgpool port, e.g. 9999):
```sql
SELECT * FROM customer LIMIT 1;
```

Expected log output: `[READ] [REPLICA] SELECT ...`

**WRITE test:**
```sql
UPDATE customer SET active = 1 WHERE customer_id = 1;
```

Expected log output: `[WRITE] [MASTER] UPDATE ...`

---

## Routing Reference

| Query | Expected Node |
|---|---|
| `SELECT` (outside transaction) | REPLICA |
| `SELECT` (inside transaction) | MASTER |
| `INSERT` / `UPDATE` / `DELETE` | MASTER |
| `BEGIN` / `COMMIT` | Ignore |

---

## Troubleshooting

| Symptom | Check |
|---|---|
| `SELECT` always → MASTER | `load_balance_mode = on`, not inside transaction, `statement_level_load_balance = on` |
| REPLICA never used | Session stickiness after write, `disable_load_balance_on_write` value, query complexity |
| WRITE hitting REPLICA | Wrong `backend_flag` or replication misconfiguration |
| No logs generated | `logging_collector = on`, `/var/log/pgpool` exists and owned by `postgres` |

Fix log directory ownership if needed:
```bash
chown -R postgres:postgres /var/log/pgpool
```

---

## Success Criteria

- [ ] `SELECT` → REPLICA
- [ ] `INSERT` / `UPDATE` / `DELETE` → MASTER
- [ ] Logs clearly show node mapping
- [ ] No unexpected routing behavior