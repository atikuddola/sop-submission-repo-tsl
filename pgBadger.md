# SOP 2: pgBadger (PostgreSQL Log Analyzer)

**Goal:** Analyze PostgreSQL logs to visualize query performance, slow queries, connection activity, and database behavior — generating a human-readable HTML report.

---

## Step 1: Install pgBadger

### Option A — APT (Debian/Ubuntu, recommended)
```bash
sudo apt install pgbadger
```

### Option B — From CPAN (latest version)
```bash
sudo apt install perl cpanminus
sudo cpanm App::pgBadger
```

### Option C — From source (GitHub)
```bash
git clone https://github.com/darold/pgbadger.git
cd pgbadger
perl Makefile.PL
make && sudo make install
```

### Option D — pip (Python wrapper, if available in your distro)
```bash
pip install pgbadger
```

### Verify installation
```bash
pgbadger --version
```

---

## Step 2: Configure PostgreSQL for pgBadger

### 2a. Edit `postgresql.conf`

**Required settings:**
```ini
logging_collector = on
log_destination = 'stderr'
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
log_min_duration_statement = 0     # log ALL queries (use higher threshold in production)
```

**Recommended additional settings:**
```ini
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0                 # log all temp files (set to size threshold in MB, e.g. 10MB = 10240)
log_autovacuum_min_duration = 0    # log all autovacuum runs
log_checkpoints = on               # log checkpoint activity
log_duration = off                 # avoid duplicate timing info when log_min_duration_statement is set
```

**`log_min_duration_statement` options:**
```ini
log_min_duration_statement = 0       # log ALL queries (development/testing)
log_min_duration_statement = 100     # log queries slower than 100ms
log_min_duration_statement = 1000    # log queries slower than 1 second (production default)
log_min_duration_statement = -1      # disable query logging entirely
```

**`log_line_prefix` — pgBadger-compatible formats:**
```ini
# Standard pgBadger format (recommended):
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '

# Alternate with millisecond timestamps:
log_line_prefix = '%m [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '

# Minimal format (also supported by pgBadger):
log_line_prefix = '%t [%p] %u@%d '

# With session ID (for fine-grained session tracking):
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h,session=%c '
```

> **Important:** pgBadger parses `log_line_prefix` strictly. Always pass the same prefix string to `pgbadger` using `--log-prefix` if it differs from the default.

**Alternative `log_destination` options:**

| Destination | Notes |
|---|---|
| `stderr` | Default; collected into files by `logging_collector` |
| `csvlog` | Structured; use `pgbadger --format csv` |
| `jsonlog` | PostgreSQL 15+; use `pgbadger --format json` |
| `syslog` | System-level log aggregation; use `pgbadger --format syslog` |

### 2b. Restart PostgreSQL
```bash
# systemd
sudo systemctl restart postgresql

# Specific cluster
sudo systemctl restart postgresql@18-main

# pg_ctlcluster (Debian/Ubuntu)
sudo pg_ctlcluster 18 main restart

# Manual
sudo -u postgres pg_ctl restart -D /var/lib/postgresql/18/main
```

---

## Step 3: Clear Old Logs (Start Fresh)

```bash
# Truncate without breaking the open file handle (recommended)
sudo -u postgres truncate -s 0 /var/log/postgresql/postgresql-18-main.log

# Alternative: rotate logs via pg_ctl
sudo -u postgres pg_ctl logrotate -D /var/lib/postgresql/18/main

# Alternative: send SIGHUP to rotate (if log_rotation_size/age is configured)
sudo kill -HUP $(cat /var/run/postgresql/18-main.pid)
```

> **Why clear logs first?** pgBadger aggregates everything in the log file. Old entries from before your test will skew the report.

---

## Step 4: Run Workload

Execute representative queries, scripts, or application traffic to populate the log with meaningful data.

```sql
-- Example workload
SELECT * FROM users WHERE id = 1;
INSERT INTO orders (user_id, total) VALUES (1, 99.99);
UPDATE products SET stock = stock - 1 WHERE id = 42;
DELETE FROM sessions WHERE expires_at < NOW();
```

Or run your application normally, then proceed to report generation.

---

## Step 5: Generate pgBadger Report

### Option A — Basic HTML report (default)
```bash
pgbadger /var/log/postgresql/postgresql-18-main.log -o report.html
```

### Option B — Process all rotated log files
```bash
pgbadger /var/log/postgresql/postgresql-18-main.log* -o report.html
```

### Option C — Specify log prefix explicitly
```bash
pgbadger /var/log/postgresql/postgresql-18-main.log \
  --log-prefix '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h ' \
  -o report.html
```

### Option D — CSV log format
```bash
pgbadger /var/log/postgresql/postgresql-18-main.csv \
  --format csv \
  -o report.html
```

### Option E — JSON log format (PostgreSQL 15+)
```bash
pgbadger /var/log/postgresql/postgresql-18-main.json \
  --format json \
  -o report.html
```

### Option F — Syslog input
```bash
pgbadger /var/log/syslog --format syslog -o report.html
```

### Option G — Filter by time range
```bash
pgbadger /var/log/postgresql/postgresql-18-main.log \
  --begin "2025-01-01 08:00:00" \
  --end "2025-01-01 18:00:00" \
  -o report.html
```

### Option H — Filter by database or user
```bash
pgbadger /var/log/postgresql/postgresql-18-main.log \
  --dbname myapp_db \
  --username appuser \
  -o report.html
```

### Option I — Incremental reports (for large or ongoing logs)
```bash
# First run — create state file
pgbadger /var/log/postgresql/postgresql-18-main.log \
  --incremental \
  --outdir /var/reports/pgbadger/ \
  --last-parsed /var/reports/pgbadger/.pgbadger_last_state

# Subsequent runs — only process new log entries
pgbadger /var/log/postgresql/postgresql-18-main.log \
  --incremental \
  --outdir /var/reports/pgbadger/ \
  --last-parsed /var/reports/pgbadger/.pgbadger_last_state
```

### Option J — Parallel processing (large logs)
```bash
pgbadger /var/log/postgresql/postgresql-18-main.log \
  --jobs 4 \
  -o report.html
```

### Option K — JSON output (for programmatic use)
```bash
pgbadger /var/log/postgresql/postgresql-18-main.log -o report.json --format-output json
```

---

## Step 6: Open the Report

```bash
# Desktop GUI (Linux)
xdg-open report.html

# macOS
open report.html

# Remote server — copy to local machine
scp user@server:/path/to/report.html ./report.html

# Serve via HTTP temporarily
python3 -m http.server 8080  # then open http://localhost:8080/report.html
```

---

## Step 7: Analyze the Report

| Section | What to Look For |
|---|---|
| **Slow Queries** | Queries exceeding `log_min_duration_statement`; candidates for index optimization or query rewrite |
| **Query Frequency** | Most-called queries; consider caching or connection pooling |
| **Connection Activity** | Spike patterns, max connections reached, idle connections |
| **Locks & Waits** | Long lock waits indicate contention; investigate conflicting transactions |
| **Temporary File Usage** | Frequent temp files suggest `work_mem` is too low |
| **Autovacuum Activity** | Tables with heavy autovacuum load may need storage tuning |
| **Checkpoints** | Frequent checkpoints may indicate `max_wal_size` is too small |
| **Error Distribution** | High error rates by query or user can reveal application bugs |

---

## Step 8: Best Practices

| Practice | Detail |
|---|---|
| **Don't use `log_min_duration_statement = 0` in production** | Logging all queries creates very high I/O. Use `100`–`1000` ms in production. |
| **Use incremental mode for continuous monitoring** | Avoids reprocessing the entire log on each run. |
| **Rotate logs before generating reports** | Keeps report scope clean and avoids memory issues with huge log files. |
| **Combine with pgAudit** | pgAudit handles *who did what*; pgBadger handles *how slowly and how often*. |
| **Schedule reports via cron** | Run nightly and archive HTML reports for trend analysis. |
| **Protect the report** | HTML reports may contain query text with sensitive data — restrict access accordingly. |
| **Use `--jobs` on multi-core servers** | Parallel processing significantly reduces report generation time for large logs. |

---

## Quick Reference: Common pgBadger Flags

| Flag | Description |
|---|---|
| `-o / --outfile` | Output file path (`.html`, `.json`, `.txt`) |
| `--format` | Input log format: `syslog`, `csv`, `json`, `stderr` (default) |
| `--log-prefix` | Specify `log_line_prefix` if non-standard |
| `--begin` / `--end` | Filter by time range |
| `--dbname` | Filter by database name |
| `--username` | Filter by PostgreSQL user |
| `--jobs` | Number of parallel workers |
| `--incremental` | Process only new log entries since last run |
| `--last-parsed` | State file path for incremental mode |
| `--outdir` | Output directory (used with `--incremental`) |
| `--quiet` | Suppress progress output |
| `--debug` | Verbose debug output |
