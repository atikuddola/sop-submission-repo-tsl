# Basic Commands for Linux 🐧

---

## 1. `ls` — List Directory Contents

| Flag | Example | Explanation |
|------|---------|-------------|
| `-l` | `ls -l /var/lib/mysql` | Long format — shows permissions, owner, size, and date |
| `-a` | `ls -a /etc/mysql` | Show hidden files (dotfiles like `.my.cnf`) |
| `-h` | `ls -lh /var/lib/mysql` | Human-readable file sizes (KB, MB, GB) |
| `-lah` | `ls -lah /var/lib/mysql` | Combined: detailed list with hidden files and readable sizes |
| `-t` | `ls -lt /backup` | Sort by modification time (newest first) |
| `-r` | `ls -ltr /backup` | Reverse sort order (oldest first — useful for log rotation checks) |

---

## 2. `cd` — Change Directory

| Flag/Usage | Example | Explanation |
|------------|---------|-------------|
| `cd <path>` | `cd /var/log/postgresql` | Navigate to an absolute path |
| `cd ..` | `cd ..` | Go up one directory level |
| `cd -` | `cd -` | Return to the previous directory |
| `cd ~` | `cd ~` | Go to the home directory of current user |

---

## 3. `pwd` — Print Working Directory

| Flag | Example | Explanation |
|------|---------|-------------|
| *(no flags)* | `pwd` | Print the full absolute path of current directory |
| `-P` | `pwd -P` | Print physical path, resolving any symlinks |
| `-L` | `pwd -L` | Print logical path, including symlinks (default behavior) |

---

## 4. `mkdir` — Make Directory

| Flag | Example | Explanation |
|------|---------|-------------|
| `-p` | `mkdir -p /backup/daily/2026-03-08` | Create parent directories as needed; no error if exists |
| `-m` | `mkdir -m 750 /backup/secure` | Set directory permissions at creation time |
| `-v` | `mkdir -v /backup/new` | Verbose — print a message for each created directory |

---

## 5. `touch` — Create or Update File Timestamps

| Flag | Example | Explanation |
|------|---------|-------------|
| *(no flags)* | `touch maintenance.flag` | Create an empty file or update its timestamp |
| `-t` | `touch -t 202603080200 file` | Set a specific timestamp (YYYYMMDDhhmm) |
| `-a` | `touch -a logfile` | Update only the access time |
| `-m` | `touch -m logfile` | Update only the modification time |

---

## 6. `cat` — Concatenate and Display Files

| Flag | Example | Explanation |
|------|---------|-------------|
| *(no flags)* | `cat /etc/my.cnf` | Print entire file contents to screen |
| `-n` | `cat -n my.cnf` | Show line numbers (useful for referencing config line issues) |
| `-A` | `cat -A my.cnf` | Show hidden characters like `^M` (Windows line endings) |
| *(concat)* | `cat file1.sql file2.sql > merged.sql` | Merge multiple SQL files into one |

---

## 7. `cp` — Copy Files and Directories

| Flag | Example | Explanation |
|------|---------|-------------|
| *(no flags)* | `cp my.cnf my.cnf.bak` | Copy a file to a new location |
| `-r` | `cp -r /var/lib/mysql /backup/mysql-copy` | Recursively copy entire directories |
| `-p` | `cp -p my.cnf my.cnf.bak` | Preserve permissions, ownership, and timestamps |
| `-i` | `cp -i dump.sql /backup/` | Prompt before overwriting existing files |
| `-v` | `cp -v *.sql /backup/` | Verbose — show each file being copied |

---

## 8. `mv` — Move or Rename Files

| Flag | Example | Explanation |
|------|---------|-------------|
| *(no flags)* | `mv old_backup.sql new_backup.sql` | Rename a file in place |
| *(move)* | `mv backup.sql /backup/archive/` | Move file to a different directory |
| `-i` | `mv -i dump.sql /backup/` | Prompt before overwriting destination |
| `-v` | `mv -v *.sql /archive/` | Verbose — print what is being moved |
| `-n` | `mv -n source.sql dest.sql` | No-clobber — never overwrite existing file |

---

## 9. `rm` — Remove Files and Directories

| Flag | Example | Explanation |
|------|---------|-------------|
| `-i` | `rm -i old_dump.sql` | Interactive — ask for confirmation before each deletion |
| `-r` | `rm -r /backup/old/` | Recursively remove a directory and its contents |
| `-f` | `rm -f lock.file` | Force removal — no error if file doesn't exist |
| `-v` | `rm -v *.bak` | Verbose — show what is being removed |
| *(caution)* | `rm -rf /old_backup/` | Force-recursive delete — **use with extreme caution** |

---

## 10. `sudo` — Execute Commands as Superuser

| Flag | Example | Explanation |
|------|---------|-------------|
| *(no flags)* | `sudo systemctl restart mysql` | Run a single command as root |
| `-u` | `sudo -u postgres psql` | Run command as a specific user (e.g., postgres) |
| `-i` | `sudo -i` | Open a root login shell with root's environment |
| `-l` | `sudo -l` | List commands the current user is allowed to run |
| `-k` | `sudo -k` | Invalidate cached sudo credentials immediately |

---

## 11. `head` — Display Beginning of a File

| Flag | Example | Explanation |
|------|---------|-------------|
| `-n` | `head -20 error.log` | Show first N lines (default is 10) |
| `-c` | `head -c 500 dump.sql` | Show first N bytes of a file |
| *(multi-file)* | `head -5 *.log` | Preview the first 5 lines of each log file |

---

## 12. `tail` — Display End of a File

| Flag | Example | Explanation |
|------|---------|-------------|
| `-n` | `tail -50 slow-query.log` | Show last N lines |
| `-f` | `tail -f /var/log/mysql/error.log` | Follow the file live — updates as new lines are written ⭐ |
| `-F` | `tail -F /var/log/mysql/error.log` | Like `-f` but handles log rotation (re-opens file if renamed) |
| `-c` | `tail -c 1000 error.log` | Show last N bytes |

---

## 13. `less` — Page Through Large Files

| Flag | Example | Explanation |
|------|---------|-------------|
| *(no flags)* | `less huge_backup.log` | Open file for scrolling without loading it into memory |
| `-N` | `less -N my.cnf` | Show line numbers while paging |
| `-S` | `less -S wide_log.log` | Don't wrap long lines — scroll horizontally instead |
| `+F` | `less +F error.log` | Start in follow mode (similar to `tail -f`) |
| `/pattern` | *(within less)* | Search forward for a pattern |

---

## 14. `grep` — Search Text Using Patterns

| Flag | Example | Explanation |
|------|---------|-------------|
| `-i` | `grep -i "timeout" app.log` | Case-insensitive search |
| `-r` | `grep -r "max_connections" /etc/mysql/` | Recursively search all files in a directory |
| `-n` | `grep -n "ERROR" error.log` | Show matching line numbers |
| `-c` | `grep -c "ERROR" error.log` | Count the number of matching lines |
| `-v` | `grep -v "INFO" app.log` | Invert — show lines that do NOT match |
| `-A` | `grep -A 3 "FATAL" error.log` | Show N lines after each match (context) |
| `-B` | `grep -B 3 "FATAL" error.log` | Show N lines before each match |
| `-E` | `grep -E "ERROR \| WARN" app.log` | Use extended regex (match multiple patterns) |

---

## 15. `find` — Search for Files and Directories

| Flag | Example | Explanation |
|------|---------|-------------|
| `-name` | `find /backup -name "*.sql"` | Find files by name pattern |
| `-mtime` | `find /backup -mtime -7` | Files modified in the last 7 days |
| `-size` | `find /var/lib -size +1G` | Find files larger than 1 GB |
| `-type` | `find /etc -type f -name "*.cnf"` | Limit to files (`f`) or directories (`d`) |
| `-exec` | `find /backup -name "*.sql" -exec ls -lh {} \;` | Execute a command on each result |
| `-delete` | `find /backup -mtime +30 -delete` | Delete files older than 30 days (**be careful**) |

---

## 16. `file` — Determine File Type

| Flag | Example | Explanation |
|------|---------|-------------|
| *(no flags)* | `file backup_2026.sql.gz` | Detect file type (compressed, text, binary, etc.) |
| `-i` | `file -i backup.sql` | Show MIME type instead of human-readable description |
| `-z` | `file -z archive.gz` | Look inside compressed files to identify content |

---

## 17. `diff` — Compare Files Line by Line

| Flag | Example | Explanation |
|------|---------|-------------|
| *(no flags)* | `diff my.cnf.prod my.cnf.staging` | Show differences between two files |
| `-u` | `diff -u my.cnf.prod my.cnf.staging` | Unified format — easier to read with context lines |
| `-i` | `diff -i config1 config2` | Ignore case differences |
| `-w` | `diff -w config1 config2` | Ignore all whitespace differences |
| `-r` | `diff -r /etc/mysql-prod/ /etc/mysql-staging/` | Recursively compare two directories |

---

## 18. `wc` — Word/Line/Character Count

| Flag | Example | Explanation |
|------|---------|-------------|
| `-l` | `wc -l slow_queries.log` | Count lines |
| `-w` | `wc -w report.txt` | Count words |
| `-c` | `wc -c dump.sql` | Count bytes |
| `-m` | `wc -m config.cnf` | Count characters |

---

## 19. `ps` — Display Running Processes

| Flag | Example | Explanation |
|------|---------|-------------|
| `aux` | `ps aux \| grep postgres` | Show all processes with CPU/MEM usage |
| `-ef` | `ps -ef \| grep mysql` | Full-format listing with PPID |
| `-u` | `ps -u mysql` | Show processes owned by a specific user |
| `--sort` | `ps aux --sort=-%cpu` | Sort by CPU usage descending |

---

## 20. `top` / `htop` — Real-Time Process Monitor

| Flag | Example | Explanation |
|------|---------|-------------|
| *(no flags)* | `top` | Real-time process view (CPU, memory, load) |
| `-u` | `top -u mysql` | Filter processes by user |
| `-p` | `top -p 12345` | Monitor a specific PID |
| `htop` | `htop` | Enhanced interactive version with color and mouse support ⭐ |

---

## 21. `free` — Display Memory Usage

| Flag | Example | Explanation |
|------|---------|-------------|
| `-h` | `free -h` | Human-readable sizes (MB/GB) |
| `-s` | `free -h -s 5` | Continuously update every N seconds |
| `-t` | `free -ht` | Show total row at the bottom |

---

## 22. `df` — Disk Space Usage

| Flag | Example | Explanation |
|------|---------|-------------|
| `-h` | `df -h /var/lib/mysql` | Human-readable sizes |
| `-T` | `df -T` | Show filesystem type (ext4, xfs, etc.) |
| `-i` | `df -i` | Show inode usage instead of block usage |
| `--total` | `df -h --total` | Add a grand total row at the bottom |

---

## 23. `du` — Disk Usage of Files/Directories

| Flag | Example | Explanation |
|------|---------|-------------|
| `-sh` | `du -sh /var/lib/postgresql/*/` | Summary with human-readable size per directory |
| `-a` | `du -ah /var/lib/mysql` | Show size of every file, not just directories |
| `--max-depth` | `du -h --max-depth=1 /var/lib` | Limit depth of directory traversal |
| `--exclude` | `du -sh --exclude="*.log" /data` | Exclude matching files from totals |

---

## 24. `kill` — Send Signals to Processes

| Flag | Example | Explanation |
|------|---------|-------------|
| `-15` | `kill -15 12345` | SIGTERM — graceful shutdown request |
| `-9` | `kill -9 12345` | SIGKILL — force terminate (use as last resort) |
| `-l` | `kill -l` | List all available signal names |
| `killall` | `killall -15 postgres` | Send signal to all processes by name |

---

## 25. `systemctl` — Control systemd Services

| Flag/Subcommand | Example | Explanation |
|-----------------|---------|-------------|
| `status` | `systemctl status mysql` | Show service state, recent logs, and PID ⭐ |
| `start` | `sudo systemctl start postgresql` | Start a stopped service |
| `stop` | `sudo systemctl stop mysql` | Gracefully stop a service |
| `restart` | `sudo systemctl restart postgresql` | Stop then start the service |
| `reload` | `sudo systemctl reload mysql` | Reload config without full restart |
| `enable` | `sudo systemctl enable mysql` | Auto-start service on system boot |
| `disable` | `sudo systemctl disable mysql` | Disable auto-start on boot |

---

## 26. `journalctl` — Query systemd Journal Logs

| Flag | Example | Explanation |
|------|---------|-------------|
| `-u` | `journalctl -u mysql` | Filter logs for a specific service unit |
| `-n` | `journalctl -u mysql -n 100` | Show last N log lines |
| `-f` | `journalctl -f -u mysql` | Follow log output in real time |
| `--since` | `journalctl -u mysql --since "1 hour ago"` | Filter by time range |
| `-p` | `journalctl -p err -u mysql` | Filter by priority (err, warning, info, etc.) |

---

## 27. `uptime` — System Uptime and Load

| Flag | Example | Explanation |
|------|---------|-------------|
| *(no flags)* | `uptime` | Shows current time, uptime duration, users, and load averages |
| `-p` | `uptime -p` | Human-friendly uptime (e.g., "up 3 days, 5 hours") |
| `-s` | `uptime -s` | Show the date/time the system was last booted |

---

## 28. `chmod` — Change File Permissions

| Flag | Example | Explanation |
|------|---------|-------------|
| *(numeric)* | `chmod 600 /etc/mysql/debian.cnf` | Set exact permissions via octal (owner rw only) |
| *(symbolic)* | `chmod u+x backup.sh` | Add execute permission for the owner |
| `-R` | `chmod -R 750 /backup/` | Recursively apply permissions to all contents |
| *(remove)* | `chmod o-rwx secret.cnf` | Remove all permissions from others |

---

## 29. `chown` — Change File Ownership

| Flag | Example | Explanation |
|------|---------|-------------|
| *(no flags)* | `chown mysql:mysql /var/lib/mysql` | Set user and group owner of a file |
| `-R` | `chown -R mysql:mysql /var/lib/mysql` | Recursively change ownership ⭐ |
| `-v` | `chown -v postgres /data/pgdata` | Verbose — show each file being changed |
| `--from` | `chown --from=root mysql file` | Only change ownership if current owner matches |

---

## 30. `ss` / `netstat` — Socket and Network Stats

| Flag | Example | Explanation |
|------|---------|-------------|
| `-tuln` | `ss -tuln \| grep :3306` | Show listening TCP/UDP ports (no DNS resolution) |
| `-p` | `ss -tulnp` | Include the process name/PID for each socket |
| `-a` | `ss -a` | Show all sockets (listening + established) |
| `netstat -an` | `netstat -an \| grep :5432` | Legacy equivalent — show all connections |

---

## 31. `ping` / `curl` — Test Connectivity

| Flag | Example | Explanation |
|------|---------|-------------|
| `ping -c` | `ping -c 4 replica-host` | Send exactly N ICMP packets and stop |
| `ping -i` | `ping -i 0.5 db-host` | Set interval between pings (seconds) |
| `curl -I` | `curl -I http://db-monitor:9104` | Fetch only HTTP headers (check if service responds) |
| `curl -o` | `curl -o output.json http://api/health` | Save response body to a file |
| `curl -v` | `curl -v http://service/endpoint` | Verbose — show full request/response details |

---

## 32. `scp` — Secure Remote File Copy

| Flag | Example | Explanation |
|------|---------|-------------|
| *(no flags)* | `scp backup.sql user@dr-site:/backups/` | Copy file to remote server securely ⭐ |
| `-r` | `scp -r /backup/ user@remote:/backup/` | Recursively copy a directory |
| `-P` | `scp -P 2222 file user@host:/path/` | Specify a non-default SSH port |
| `-i` | `scp -i ~/.ssh/key.pem file user@host:/path/` | Use a specific SSH identity/key file |
| `-C` | `scp -C dump.sql user@host:/tmp/` | Enable compression during transfer |

---

## 33. `rsync` — Efficient File Synchronization

| Flag | Example | Explanation |
|------|---------|-------------|
| `-a` | `rsync -a /backup/ user@remote:/backup/` | Archive mode — preserves permissions, timestamps, symlinks |
| `-v` | `rsync -av /backup/ /archive/` | Verbose — show files being transferred |
| `-z` | `rsync -avz /backup/ user@remote:/backup/` | Compress data during transfer |
| `--delete` | `rsync -av --delete /src/ /dst/` | Remove files in destination not in source (mirror) |
| `--dry-run` | `rsync -av --dry-run /src/ /dst/` | Simulate the transfer — no files actually moved |
| `--exclude` | `rsync -av --exclude="*.tmp" /data/ /backup/` | Skip files matching a pattern |

---

## 34. `tar` — Archive and Compress Files

| Flag | Example | Explanation |
|------|---------|-------------|
| `-c` | `tar -c /var/lib/mysql` | Create a new archive |
| `-x` | `tar -xzvf backup.tar.gz` | Extract an archive |
| `-z` | `tar -czvf backup.tar.gz /data` | Use gzip compression |
| `-j` | `tar -cjvf backup.tar.bz2 /data` | Use bzip2 compression (slower, smaller) |
| `-v` | `tar -czvf backup.tar.gz /data` | Verbose — list files as they are archived |
| `-f` | `tar -czvf /backup/db_$(date +%F).tar.gz /var/lib/mysql` | Specify output filename ⭐ |
| `-t` | `tar -tzvf backup.tar.gz` | List archive contents without extracting |

---

## 35. `crontab` — Schedule Automated Jobs

| Flag | Example | Explanation |
|------|---------|-------------|
| `-e` | `crontab -e` | Open crontab file in editor to add/edit jobs |
| `-l` | `crontab -l` | List all scheduled cron jobs for current user |
| `-r` | `crontab -r` | Remove all cron jobs (**be careful!**) |
| `-u` | `sudo crontab -u mysql -l` | Manage cron jobs for a specific user |

**Cron Schedule Format:**
```
*  *  *  *  *  command
│  │  │  │  └─ Day of week (0=Sun)
│  │  │  └──── Month (1-12)
│  │  └─────── Day of month (1-31)
│  └────────── Hour (0-23)
└───────────── Minute (0-59)
```

**Example:**
```bash
0 2 * * * /usr/local/bin/backup.sh   # Run backup every day at 2:00 AM
```

---

## 36. `sed` — Stream Editor for Text Transformation

| Flag | Example | Explanation |
|------|---------|-------------|
| `-i` | `sed -i 's/100/200/' my.cnf` | Edit file in place (no backup) |
| `-i.bak` | `sed -i.bak 's/old/new/' my.cnf` | Edit in place and keep `.bak` backup ⭐ |
| `-n` | `sed -n '10,20p' my.cnf` | Print only lines 10–20 |
| `-e` | `sed -e 's/foo/bar/' -e 's/baz/qux/' file` | Apply multiple expressions |
| `g` flag | `sed 's/ERROR/WARN/g' app.log` | Replace all occurrences on each line (not just first) |

---

## 37. `tcpdump` — Capture Network Traffic

| Flag | Example | Explanation |
|------|---------|-------------|
| `-i` | `tcpdump -i eth0 port 3306` | Listen on a specific network interface |
| `port` | `tcpdump port 5432` | Filter by port number |
| `-w` | `tcpdump -i any port 5432 -w db_traffic.pcap` | Write captured packets to a file for analysis ⭐ |
| `-r` | `tcpdump -r db_traffic.pcap` | Read and analyze a previously saved capture file |
| `-n` | `tcpdump -n port 3306` | Don't resolve hostnames (faster output) |
| `-c` | `tcpdump -c 100 port 3306` | Capture only N packets then stop |

---


