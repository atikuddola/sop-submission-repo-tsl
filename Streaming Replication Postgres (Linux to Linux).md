# Standard Operating Procedure (SOP)
# PostgreSQL 18 Streaming Replication
Primary (Master) → Standby (Slave)

OS: AlmaLinux OS

Version: PostgreSQL 18  
Replication Type: Physical Streaming Replication  
Primary IP: 192.168.122.107  
Standby IP: 192.168.122.237  

---
<img width="1413" height="825" alt="Postgres Streaming Replication" src="https://github.com/user-attachments/assets/d9ec4bd6-4588-4ab3-aa04-69a7192148a1" />

# 1. Overview

This document describes the step-by-step procedure to configure physical streaming replication between:

- Primary (Master) Server
- Standby (Slave) Server

The standby will continuously replicate WAL records from the primary.

---

# 2. Prerequisites

## 2.1 System Requirements

- PostgreSQL 18 installed on both servers
- Network connectivity between servers (port 5432 open)
- Same PostgreSQL major version on both servers
- Sufficient disk space for base backup
- Time synchronized (recommended: chrony or ntpd)

## 2.2 Firewall Requirement

Ensure port 5432 is open on the Primary:

Example (firewalld):
```
firewall-cmd --add-port=5432/tcp --permanent
firewall-cmd --reload
```

---

####################################
# 3. On the Primary (Master) Server
####################################

Switch to root:

```
sudo su -
```

Check PostgreSQL service:

```
systemctl status postgresql-18.service
```

---

## 3.1 Configure postgresql.conf

Edit configuration file:

```
sudo nano /var/lib/pgsql/18/data/postgresql.conf
```

Ensure the following parameters are set:

```
listen_addresses = '*'
wal_level = replica
max_wal_senders = 10
wal_keep_size = 64MB
archive_mode = on
archive_command = 'rsync -a %p /log/archive/%f'
hot_standby = on

# Optional alternatives:
# archive_command = 'true'
# archive_command = 'cp -a %p /log/archive/%f'
```

Important Notes:
- Ensure `/log/archive` directory exists.
- Proper permissions must be set:
```
mkdir -p /log/archive
chown -R postgres:postgres /log/archive
chmod 700 /log/archive
```

---

## 3.2 Configure pg_hba.conf

Edit:

```
sudo nano /var/lib/pgsql/18/data/pg_hba.conf
```

Add replication entry:

```
host    replication     slave     192.168.122.237/32     md5
```

Reload is NOT sufficient — restart required after replication parameter changes.

---

## 3.3 Create Replication User

```
sudo -u postgres psql
```

Inside psql:

```
CREATE USER slave REPLICATION LOGIN PASSWORD 'slave';
```

Exit psql:

```
\q
```

---

## 3.4 Restart PostgreSQL

```
systemctl restart postgresql-18
systemctl status postgresql-18
```

Verify listening:

```
ss -lntp | grep 5432
```

Primary configuration complete.

---

#######################################
# 4. On the Standby (Slave) Server
#######################################

IMPORTANT:
Existing PGDATA must be moved or removed before base backup.

---

## 4.1 Stop PostgreSQL

```
systemctl status postgresql-18
systemctl stop postgresql-18
```

---

## 4.2 Prepare Data Directory

```
mv /var/lib/pgsql/18/data /var/lib/pgsql/18/data_old
mkdir /var/lib/pgsql/18/data
chown -R postgres:postgres /var/lib/pgsql/18/data
chmod 700 /var/lib/pgsql/18/data
```

---

## 4.3 Take Base Backup from Primary

Switch to postgres user:

```
sudo -i -u postgres
```

Run base backup:

```
pg_basebackup -h 192.168.122.107 \
-D /var/lib/pgsql/18/data \
-U slave \
-Fp -Xs -P -R
```

Explanation of options:

- -Fp → plain format
- -Xs → stream WAL while backup runs
- -P → show progress
- -R → automatically creates standby configuration

If prompted, enter replication user password.

---

## 4.4 Start Standby Server

```
systemctl start postgresql-18
systemctl status postgresql-18
```

---

#######################################
# 5. Verification
#######################################

## 5.1 Verify on Primary

```
sudo -u postgres psql
```

Run:

```
SELECT application_name,
       client_addr,
       state,
       sync_state,
       write_lag,
       flush_lag,
       replay_lag
FROM pg_stat_replication;
```

Expected:
- state = streaming
- client_addr = 192.168.122.237

---

## 5.2 Verify on Standby

```
sudo -u postgres psql
```

Run:

```
SELECT pg_is_in_recovery();
```

Expected output:

```
t
```

If result is `t`, standby is correctly in recovery mode.

---

#######################################
# 6. Troubleshooting
#######################################

## 6.1 Connection Refused

- Verify primary is listening on 5432
- Check firewall
- Verify pg_hba.conf entry
- Confirm correct IP address

## 6.2 Authentication Failed

- Check replication user password
- Confirm pg_hba.conf authentication method
- Reload/restart after changes

## 6.3 Replication Not Streaming

Check on primary:

```
SELECT * FROM pg_stat_replication;
```

Check standby logs:

```
journalctl -u postgresql-18 -f
```

---

#######################################
# 7. Promotion (Failover)
#######################################

To promote standby to primary:

```
sudo -u postgres pg_ctl promote -D /var/lib/pgsql/18/data
```

OR

```
touch /var/lib/pgsql/18/data/promote
```

Verify:

```
SELECT pg_is_in_recovery();
```

Expected:
```
f
```

---
# Standard Operating Procedure (SOP)
# PostgreSQL 18 Streaming Replication
Primary (Master) → Standby (Slave)

OS: AlmaLinux OS
Version: PostgreSQL 18  
Replication Type: Physical Streaming Replication  
Primary IP: 192.168.122.107  
Standby IP: 192.168.122.237  

---

# 1. Overview

This document describes the step-by-step procedure to configure physical streaming replication between:

- Primary (Master) Server
- Standby (Slave) Server

The standby will continuously replicate WAL records from the primary.

---

# 2. Prerequisites

## 2.1 System Requirements

- PostgreSQL 18 installed on both servers
- Network connectivity between servers (port 5432 open)
- Same PostgreSQL major version on both servers
- Sufficient disk space for base backup
- Time synchronized (recommended: chrony or ntpd)

## 2.2 Firewall Requirement

Ensure port 5432 is open on the Primary:

Example (firewalld):
```
firewall-cmd --add-port=5432/tcp --permanent
firewall-cmd --reload
```

---

####################################
# 3. On the Primary (Master) Server
####################################

Switch to root:

```
sudo su -
```

Check PostgreSQL service:

```
systemctl status postgresql-18.service
```

---

## 3.1 Configure postgresql.conf

Edit configuration file:

```
sudo nano /var/lib/pgsql/18/data/postgresql.conf
```

Ensure the following parameters are set:

```
listen_addresses = '*'
wal_level = replica
max_wal_senders = 10
wal_keep_size = 64MB
archive_mode = on
archive_command = 'rsync -a %p /log/archive/%f'
hot_standby = on

# Optional alternatives:
# archive_command = 'true'
# archive_command = 'cp -a %p /log/archive/%f'
```

Important Notes:
- Ensure `/log/archive` directory exists.
- Proper permissions must be set:
```
mkdir -p /log/archive
chown -R postgres:postgres /log/archive
chmod 700 /log/archive
```

---

## 3.2 Configure pg_hba.conf

Edit:

```
sudo nano /var/lib/pgsql/18/data/pg_hba.conf
```

Add replication entry:

```
host    replication     slave     192.168.122.237/32     md5
```

Reload is NOT sufficient — restart required after replication parameter changes.

---

## 3.3 Create Replication User

```
sudo -u postgres psql
```

Inside psql:

```
CREATE USER slave REPLICATION LOGIN PASSWORD 'slave';
```

Exit psql:

```
\q
```

---

## 3.4 Restart PostgreSQL

```
systemctl restart postgresql-18
systemctl status postgresql-18
```

Verify listening:

```
ss -lntp | grep 5432
```

Primary configuration complete.

---

#######################################
# 4. On the Standby (Slave) Server
#######################################

IMPORTANT:
Existing PGDATA must be moved or removed before base backup.

---

## 4.1 Stop PostgreSQL

```
systemctl status postgresql-18
systemctl stop postgresql-18
```

---

## 4.2 Prepare Data Directory

```
mv /var/lib/pgsql/18/data /var/lib/pgsql/18/data_old
mkdir /var/lib/pgsql/18/data
chown -R postgres:postgres /var/lib/pgsql/18/data
chmod 700 /var/lib/pgsql/18/data
```

---

## 4.3 Take Base Backup from Primary

Switch to postgres user:

```
sudo -i -u postgres
```

Run base backup:

```
pg_basebackup -h 192.168.122.107 \
-D /var/lib/pgsql/18/data \
-U slave \
-Fp -Xs -P -R
```

Explanation of options:

- -Fp → plain format
- -Xs → stream WAL while backup runs
- -P → show progress
- -R → automatically creates standby configuration

If prompted, enter replication user password.

---

## 4.4 Start Standby Server

```
systemctl start postgresql-18
systemctl status postgresql-18
```

---

#######################################
# 5. Verification
#######################################

## 5.1 Verify on Primary

```
sudo -u postgres psql
```

Run:

```
SELECT application_name,
       client_addr,
       state,
       sync_state,
       write_lag,
       flush_lag,
       replay_lag
FROM pg_stat_replication;
```

Expected:
- state = streaming
- client_addr = 192.168.122.237

---

## 5.2 Verify on Standby

```
sudo -u postgres psql
```

Run:

```
SELECT pg_is_in_recovery();
```

Expected output:

```
t
```

If result is `t`, standby is correctly in recovery mode.

---

#######################################
# 6. Troubleshooting
#######################################

## 6.1 Connection Refused

- Verify primary is listening on 5432
- Check firewall
- Verify pg_hba.conf entry
- Confirm correct IP address

## 6.2 Authentication Failed

- Check replication user password
- Confirm pg_hba.conf authentication method
- Reload/restart after changes

## 6.3 Replication Not Streaming

Check on primary:

```
SELECT * FROM pg_stat_replication;
```

Check standby logs:

```
journalctl -u postgresql-18 -f
```

---

#######################################
# 7. Promotion (Failover)
#######################################

To promote standby to primary:

```
sudo -u postgres pg_ctl promote -D /var/lib/pgsql/18/data
```

OR

```
touch /var/lib/pgsql/18/data/promote
```

Verify:

```
SELECT pg_is_in_recovery();
```

Expected:
```
f
```

---

#######################################
# 8. Best Practices
#######################################

- Use strong passwords
- Use SSL for replication in production
- Monitor replication lag regularly
- Use synchronous replication if zero data loss is required
- Keep WAL archive on separate disk
- Monitor disk usage of archive directory
- Implement regular backups in addition to replication
- Document failover procedure clearly

---


Streaming Replication Postgres (Linux to Linux)
#######################################
# 8. Best Practices
#######################################

- Use strong passwords
- Use SSL for replication in production
- Monitor replication lag regularly
- Use synchronous replication if zero data loss is required
- Keep WAL archive on separate disk
- Monitor disk usage of archive directory
- Implement regular backups in addition to replication
- Document failover procedure clearly

---
