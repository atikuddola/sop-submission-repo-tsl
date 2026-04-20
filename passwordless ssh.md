# SOP — Passwordless SSH Setup to Multiple VMs

---

## Step 1 — Generate SSH Key (on host machine)

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

- Press `Enter` for default file location
- Leave passphrase empty for zero prompts

**Generated files:**

| File | Purpose |
|------|---------|
| `~/.ssh/id_ed25519` | Private key — never share |
| `~/.ssh/id_ed25519.pub` | Public key — copied to VMs |

---

## Step 2 — Copy Public Key to Each VM

```bash
ssh-copy-id user@192.168.122.105
ssh-copy-id user@192.168.122.98
ssh-copy-id user@192.168.122.147
ssh-copy-id user@192.168.122.252
ssh-copy-id user@192.168.122.181
```

Enter the VM password once per machine. Not needed again after this.

---

## Step 3 — Test Connection

```bash
ssh user@192.168.122.105
```

Should log in without a password prompt.

---

## Step 4 — Simplify with SSH Config (Recommended)

Edit `~/.ssh/config`:

```
Host vm105
    HostName 192.168.122.105
    User user

Host vm98
    HostName 192.168.122.98
    User user

Host vm147
    HostName 192.168.122.147
    User user

Host vm252
    HostName 192.168.122.252
    User user

Host vm181
    HostName 192.168.122.181
    User user
```

Now connect using just:

```bash
ssh vm105
```

---

## Optional — VM-to-VM Passwordless SSH

If VMs need to SSH into each other:

```bash
# On the source VM — generate a key
ssh-keygen -t ed25519

# Copy to target VMs
ssh-copy-id user@192.168.122.98
```

Or copy the same `~/.ssh/id_ed25519.pub` to `~/.ssh/authorized_keys` on all target VMs.

---

*SOP Version 1.0 — Passwordless SSH Setup*
