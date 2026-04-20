# SOP 1: pgAudit (PostgreSQL Audit Extension)

**Goal:** Log and track who did what in PostgreSQL — capturing write operations and DDL events with session-level detail.

---

## Step 1: Install pgAudit

### Option A — APT (Debian/Ubuntu, recommended)
```bash
sudo apt install postgresql-18-pgaudit
```

### Option B — From source (any Linux)
```bash
git clone https://github.com/pgaudit/pgaudit.git
cd pgaudit
make install USE_PGXS=1
```

### Option C — Via PGDG repo (if default apt doesn't have the package)
```bash
sudo apt install -y postgresql-common
sudo sh /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
sudo apt install postgresql-18-pgaudit
```

### Verify installation
```sql
SELECT * FROM pg_extension WHERE extname = 'pgaudit';
-- Or check available extensions:
SELECT * FROM pg_available_extensions WHERE name = 'pgaudit';
```

---

## Step 2: Enable pgAudit in PostgreSQL

### 2a. Edit `postgresql.conf`

**Minimum required settings:**
```ini
shared_preload_libraries = 'pgaudit'
pgaudit.log = 'write, ddl'
```

**Recommended production settings:**
```ini
shared_preload_libraries = 'pgaudit'

# Audit scope — choose one or combine:
pgaudit.log = 'write, ddl'          # logs INSERT/UPDATE/DELETE + CREATE/ALTER/DROP
# pgaudit.log = 'all'               # logs everything (verbose)
# pgaudit.log = 'read, write, ddl'  # adds SELECT logging
# pgaudit.log = 'role'              # logs GRANT/REVOKE

# Log formatting
pgaudit.log_catalog = on            # include system catalog queries (default: on)
pgaudit.log_level = log             # log level: log, notice, warning, error
pgaudit.log_parameter = on          # include bind parameters in log (useful for debugging)
pgaudit.log_relation = on           # log each relation (table/view) separately per statement
pgaudit.log_statement_once = off    # log statement text only on first entry per statement

# Log output
logging_collector = on
log_destination = 'stderr'          # alternatives: 'csvlog', 'syslog', 'jsonlog'
log_line_prefix = '%t [%p] %u@%d '  # timestamp, PID, user@database
```

**Alternative `log_destination` options:**
```ini
log_destination = 'csvlog'    # structured CSV — easier to parse programmatically
log_destination = 'syslog'    # send to OS syslog (good for centralized logging)
log_destination = 'jsonlog'   # JSON format (PostgreSQL 15+)
log_destination = 'stderr, csvlog'  # multiple destinations supported
```

**Alternative `log_line_prefix` formats:**
```ini
log_line_prefix = '%t [%p] %u@%d '             # basic: time, pid, user@db
log_line_prefix = '%m [%p] %q%u@%d '           # millisecond timestamp
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '  # pgBadger-compatible
log_line_prefix = '%t [%p] %u@%d %r '          # includes remote host:port
```

### 2b. Object-level auditing (per role)

To audit only specific users or objects instead of globally:
```sql
-- Enable audit for a specific role
ALTER ROLE myuser SET pgaudit.log = 'write';

-- Enable audit on a specific table (requires pgaudit.log_relation = on)
ALTER ROLE myuser SET pgaudit.log_relation = on;
```

### 2c. Restart PostgreSQL
```bash
# systemd (most systems)
sudo systemctl restart postgresql

# Specific cluster version
sudo systemctl restart postgresql@18-main

# pg_ctlcluster (Debian/Ubuntu)
sudo pg_ctlcluster 18 main restart

# Manual (if not using systemd)
sudo -u postgres pg_ctl restart -D /var/lib/postgresql/18/main
```

---

## Step 3: Enable the Extension in the Database

```sql
-- Connect to target database and run:
CREATE EXTENSION IF NOT EXISTS pgaudit;

-- Verify:
\dx pgaudit
```

> **Note:** `shared_preload_libraries` loads the module at server start; `CREATE EXTENSION` registers it per-database. Both are required.

---

## Step 4: Test pgAudit

```sql
-- Run test DDL and DML statements:
CREATE TABLE audit_test (id INT);
INSERT INTO audit_test VALUES (1);
UPDATE audit_test SET id = 2;
DELETE FROM audit_test WHERE id = 2;
DROP TABLE audit_test;
```

---

## Step 5: Check Logs

### Option A — Follow logs live (stderr/file)
```bash
tail -f /var/log/postgresql/postgresql-18-main.log | grep AUDIT
```

### Option B — journalctl (if using systemd logging)
```bash
journalctl -u postgresql -f | grep AUDIT
```

### Option C — csvlog (if `log_destination = 'csvlog'`)
```bash
tail -f /var/log/postgresql/postgresql-18-main.csv
# Or query via foreign table:
# See pg_log_csv foreign table setup in PostgreSQL docs
```

### Option D — syslog
```bash
grep AUDIT /var/log/syslog
grep AUDIT /var/log/messages  # RHEL/CentOS
```

### Expected output (SESSION audit)
```
AUDIT: SESSION,1,1,DDL,CREATE TABLE,TABLE,public.audit_test,CREATE TABLE audit_test(id int),<none>
AUDIT: SESSION,2,1,WRITE,INSERT,TABLE,public.audit_test,INSERT INTO audit_test VALUES (1),<none>
AUDIT: SESSION,3,1,WRITE,UPDATE,TABLE,public.audit_test,UPDATE audit_test SET id = 2,<none>
```

---

## Step 6: Best Practices

| Practice | Detail |
|---|---|
| **Scope audit log carefully** | `pgaudit.log = 'all'` is thorough but expensive. Start with `write, ddl`. |
| **Enable log rotation** | Set `log_rotation_age` and `log_rotation_size` in `postgresql.conf` to avoid unbounded log files. |
| **Separate audit logs from error logs** | Use `log_destination = 'csvlog'` or `syslog` for easy separation. |
| **Use object auditing for fine-grained control** | Apply `pgaudit.log` per role for targeted coverage. |
| **Protect log files** | Logs may contain sensitive data — restrict read access appropriately. |
| **Avoid logging SELECT in production without cause** | `read` logging generates very high volume. |
| **Truncate logs safely** | Use `sudo -u postgres truncate -s 0 /var/log/postgresql/postgresql-18-main.log` to clear without breaking the file handle. |

---

## Quick Reference: `pgaudit.log` Values

| Value | Logs |
|---|---|
| `read` | SELECT, COPY (source) |
| `write` | INSERT, UPDATE, DELETE, TRUNCATE, COPY (destination) |
| `function` | Function calls and DO blocks |
| `role` | GRANT, REVOKE, CREATE/ALTER/DROP ROLE |
| `ddl` | CREATE, ALTER, DROP (non-role) |
| `misc` | DISCARD, FETCH, CHECKPOINT, VACUUM, SET |
| `misc_set` | SET statements only |
| `all` | All of the above |

Combine with commas: `pgaudit.log = 'read, write, ddl'`
