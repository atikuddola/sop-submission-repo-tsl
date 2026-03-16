# Patroni — Node Detach, Resize & Reattach SOP

> **Scope:** How to cleanly remove a replica node from a running Patroni cluster, resize its VM specs, and reattach it properly.
> **Tested on:** Rocky Linux 9, QEMU/KVM (virt-manager), Patroni + etcd + HAProxy

---

## Table of Contents

- [Phase 1 — Detach the Node](#phase-1--detach-the-node)
- [Phase 2 — Resize the VM](#phase-2--resize-the-vm)
- [Phase 3 — Reattach the Node](#phase-3--reattach-the-node)
- [Verification](#verification)

---

## Phase 1 — Detach the Node

> Run Step 1 and 2 from **any existing cluster node**. Run Step 3 on the **node being detached**.

### Step 1 — Pause auto-failover (safety)

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml pause
```

### Step 2 — Confirm the node is a Replica, not Leader

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml list
```

If the node is currently Leader, switchover first:

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml switchover postgres \
    --master pgnode4 --candidate pgnode1
```

### Step 3 — Stop Patroni on the node being detached

```bash
# On pgnode4:
sudo systemctl stop patroni
```

### Step 4 — Resume auto-failover

```bash
# From any existing node:
sudo -u postgres patronictl -c /etc/patroni/patroni.yml resume
```

### Step 5 — Remove pgnode4 from HAProxy config

**File:** `/etc/haproxy/haproxy.cfg` on **pgproxy**

Remove (or comment out) the pgnode4 server line from **both** listen blocks:

```
listen primary
    # server pgnode4 192.168.122.252:5432 maxconn 100 check port 8008  ← remove

listen standby
    # server pgnode4 192.168.122.252:5432 maxconn 100 check port 8008  ← remove
```

Reload HAProxy:

```bash
sudo systemctl reload haproxy
```

### Step 6 — Remove pgnode4 IP from pg_hba in etcd

**From any existing node:**

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml edit-config postgres
```

Remove the pgnode4 line from `pg_hba`:

```yaml
pg_hba:
- host replication replicator 192.168.122.98/32 scram-sha-256
- host replication replicator 192.168.122.147/32 scram-sha-256
- host replication replicator 192.168.122.105/32 scram-sha-256
# remove this line ↓
# - host replication replicator 192.168.122.252/32 scram-sha-256
- host all all 0.0.0.0/0 scram-sha-256
```

Save and exit. Reload the current leader:

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml reload postgres pgnode1
```

### Step 7 — Verify cluster is healthy on remaining nodes

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml list
```

pgnode1, pgnode2, pgnode3 should all show `streaming` or `running`. pgnode4 absent — expected.

---

## Phase 2 — Resize the VM

> Run all commands on the **hypervisor host** unless stated otherwise.

### Step 1 — Check current VM name

```bash
virsh list --all
```

### Step 2 — Check current specs

```bash
virsh dominfo pgnode4
virsh dumpxml pgnode4 | grep -E 'memory|vcpu'
virsh domblklist pgnode4
```

### Step 3 — CPU

**Hot-add (if maxvcpus > current):**

```bash
virsh setvcpus pgnode4 4 --live
```

**If maxvcpus equals current — requires shutdown:**

```bash
virsh shutdown pgnode4
virsh setvcpus pgnode4 4 --config --maximum
virsh setvcpus pgnode4 4 --config
virsh start pgnode4
```

### Step 4 — RAM

**Hot-add (if max memory > current, value in KiB):**

```bash
virsh setmem pgnode4 4194304 --live
```

**If max equals current — requires shutdown:**

```bash
virsh shutdown pgnode4
virsh setmaxmem pgnode4 8388608 --config
virsh setmem pgnode4 4194304 --config
virsh start pgnode4
```

### Step 5 — Disk expansion (fully live)

**On the hypervisor host** — resize the image:

```bash
# Check current image:
sudo qemu-img info /var/lib/libvirt/images/pgnode4.qcow2

# Add space (use G not T):
sudo qemu-img resize /var/lib/libvirt/images/pgnode4.qcow2 +10G

# Verify:
sudo qemu-img info /var/lib/libvirt/images/pgnode4.qcow2 | grep 'virtual size'
```

> ⚠️ Always use `+10G` not `+10T` — qemu-img accepts both silently. Double-check `virtual size` after.

Notify the running VM of the new disk size:

```bash
virsh blockresize pgnode4 /var/lib/libvirt/images/pgnode4.qcow2 <new-size-in-bytes>
```

**On pgnode4 guest** — expand partition and filesystem:

```bash
# Check current layout first:
lsblk

# Grow the partition (change 2 to your actual partition number):
sudo growpart /dev/vda 2

# Resize LVM physical volume:
sudo pvresize /dev/vda2

# Extend logical volume:
sudo lvextend -l +100%FREE /dev/rlm/root

# Grow the filesystem (XFS — Rocky Linux default):
sudo xfs_growfs /

# Verify:
df -h /
lsblk
```

---

## Phase 3 — Reattach the Node

> Run Steps 1–2 on **pgnode4**. Run Steps 3–5 from **any existing cluster node**.

### Step 1 — Add pgnode4 IP back to pg_hba in etcd

**From any existing node:**

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml edit-config postgres
```

Add the line back:

```yaml
pg_hba:
- host replication replicator 192.168.122.98/32 scram-sha-256
- host replication replicator 192.168.122.147/32 scram-sha-256
- host replication replicator 192.168.122.105/32 scram-sha-256
- host replication replicator 192.168.122.252/32 scram-sha-256   # ← add back
- host all all 0.0.0.0/0 scram-sha-256
```

Save and exit. Reload the current leader:

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml reload postgres pgnode1
```

Verify:

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml show-config postgres
```

### Step 2 — Clear stale data directory on pgnode4

Patroni will clone fresh data from the leader — the old data dir must be empty:

```bash
# On pgnode4:
sudo rm -rf /data/pgsql/16/data/*
sudo chown -R postgres:postgres /data/pgsql/16
sudo chmod 700 /data/pgsql/16/data
```

### Step 3 — Start Patroni on pgnode4

```bash
# On pgnode4:
sudo systemctl start patroni
sudo journalctl -u patroni -f
```

Wait for:

```
INFO: trying to bootstrap from leader 'pgnode1'
INFO: clone from 'pgnode1' in progress
INFO: bootstrap succeeded
```

> If you see `no pg_hba.conf entry` → go back to Step 1 and confirm the IP was saved.

### Step 4 — Add pgnode4 back to HAProxy config

**File:** `/etc/haproxy/haproxy.cfg` on **pgproxy**

Add the server line back to **both** listen blocks:

```
listen primary
    server pgnode4 192.168.122.252:5432 maxconn 100 check port 8008

listen standby
    server pgnode4 192.168.122.252:5432 maxconn 100 check port 8008
```

Reload HAProxy:

```bash
sudo systemctl reload haproxy
```

---

## Verification

From any node — confirm all 4 members healthy:

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml list
```

Expected:

```
+----------+------------------+---------+-----------+----+-----------+
| Member   | Host             | Role    | State     | TL | Lag in MB |
+----------+------------------+---------+-----------+----+-----------+
| pgnode1  | 192.168.122.98   | Leader  | running   |  1 |           |
| pgnode2  | 192.168.122.147  | Replica | streaming |  1 |         0 |
| pgnode3  | 192.168.122.105  | Replica | streaming |  1 |         0 |
| pgnode4  | 192.168.122.252  | Replica | streaming |  1 |         0 |
+----------+------------------+---------+-----------+----+-----------+
```

Check HAProxy stats to confirm pgnode4 is green:

```
http://192.168.122.112:7000/
```

---

## Summary — What Changes Where

| Step | File / Command | Where to Run |
|------|---------------|--------------|
| Pause/resume failover | `patronictl pause/resume` | any existing node |
| Stop/start Patroni | `systemctl stop/start patroni` | pgnode4 |
| Remove/add pg_hba entry | `patronictl edit-config` | any existing node |
| Reload leader config | `patronictl reload` | any existing node |
| Remove/add HAProxy server line | `/etc/haproxy/haproxy.cfg` | pgproxy |
| Reload HAProxy | `systemctl reload haproxy` | pgproxy |
| VM resize (CPU/RAM/Disk) | `virsh` commands | hypervisor host |
| Expand partition + filesystem | `growpart`, `lvextend`, `xfs_growfs` | pgnode4 guest |
| Clear data dir before rejoin | `rm -rf /data/pgsql/16/data/*` | pgnode4 |

---

*The main cluster (pgnode1/2/3) stays fully operational and serving traffic throughout this entire process.*
