# Logrotate: Automated Log Lifecycle & Rotation Management

## 1. Architectural Concept & Deep Dive

In 24/7 Linux server operations, unmanaged log files grow monotonically until they consume 100% of the available filesystem blocks or inodes, leading to catastrophic disk-full outages (`ENOSPC`).

**Logrotate** is the standard system utility designed to automate the archiving, compression, rotation, and deletion of log files. Executed daily via systemd timer (`logrotate.timer`) or cron (`/etc/cron.daily/logrotate`), it reads declarative configuration files, evaluates file sizes or aging thresholds, and executes configured lifecycle policies.

```
+─────────────────────────────────────────────────────────────────────────+
|                        LOGROTATE LIFECYCLE FLOW                         |
|                                                                         |
|  Active Log: /var/log/nginx/access.log (Current writes)                 |
|                               │                                         |
|  [Daily Timer or Size Limit]  ▼                                         |
|  Step 1: Move active log:     access.log ──► access.log.1               |
|  Step 2: Signal service:      nginx sends USR1 to reopen access.log     |
|  Step 3: Compress previous:   access.log.2 ──► access.log.2.gz          |
|  Step 4: Prune stale archive: access.log.14.gz deleted (retention: 14)  |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Configuration Architecture

Logrotate configuration follows a modular inheritance model:
- **Global Configuration**: `/etc/logrotate.conf` sets baseline defaults (e.g., weekly rotation, gzip compression, root ownership).
- **Service-Specific Drop-ins**: `/etc/logrotate.d/` contains isolated configuration files for specific services (e.g. `/etc/logrotate.d/nginx`, `/etc/logrotate.d/rsyslog`, `/etc/logrotate.d/dpkg`).

### Anatomy of a Production Configuration
```ini
# /etc/logrotate.d/nginx
/var/log/nginx/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 www-data adm
    sharedscripts
    prerotate
        if [ -d /etc/logrotate.d/httpd-prerotate ]; then \
            run-parts /etc/logrotate.d/httpd-prerotate; \
        fi \
    endscript
    postrotate
        [ -s /run/nginx.pid ] && kill -USR1 `cat /run/nginx.pid`
    endscript
}
```

---

## 3. Core Directives Reference

| Directive | Function | Production Rationale |
| :--- | :--- | :--- |
| `daily` / `weekly` / `monthly` | Rotation frequency interval. | Defines baseline time-based cadence. |
| `size 100M` / `maxsize 500M` | Rotates if log exceeds size, even if interval has not elapsed. | Critical safeguard against rogue runaway log floods. |
| `rotate 14` | Retains 14 rotated archives before deleting the oldest. | Enforces compliance retention policies without unbounded disk consumption. |
| `compress` | Compresses rotated logs using gzip (`.gz`). | Reduces log storage footprint by 80% to 90%. |
| `delaycompress` | Delays compression of the most recent rotated file (`.1`) until the next cycle. | Prevents file corruption when daemons take time to close file descriptors. |
| `missingok` | Do not emit an error if the log file is missing. | Prevents alert spam if a service is temporarily uninstalled or disabled. |
| `notifempty` | Skip rotation if the file is 0 bytes. | Avoids creating empty archive files (`access.log.1.gz`). |
| `create <mode> <owner> <group>`| Creates a new empty log file with specified permissions immediately after rotation. | Ensures the service has write access to the newly created log target. |
| `sharedscripts` | Runs `prerotate`/`postrotate` scripts only once per cycle, not once per matched file. | Ensures daemons like Nginx are signaled once instead of for each VirtualHost log. |

---

## 4. `create` vs `copytruncate`: The Critical Architectural Distinction

Choosing between `create` and `copytruncate` is the most important decision in logrotate design:

```
               METHOD 1: create (Standard)
  1. rename(/var/log/app.log -> /var/log/app.log.1)
  2. touch(/var/log/app.log)
  3. signal(app_pid, SIGHUP/SIGUSR1) to reopen file descriptor
  * Zero log loss! Requires application support for signaling.

               METHOD 2: copytruncate (Fallback)
  1. copy(/var/log/app.log -> /var/log/app.log.1)
  2. truncate(/var/log/app.log to 0 bytes)
  * No service restart or signal needed!
  * RISK: Data written between copy and truncate is permanently lost.
```

### When to use `copytruncate`:
Use `copytruncate` **only** for applications that cannot be signaled to reopen their log files (e.g. proprietary binary blobs, Java daemons without reload hooks, or legacy Python scripts). For all other daemons, use `create` with a proper `postrotate` signal.

---

## 5. Hands-on Terminal Lab: Testing & Debugging Logrotate

### Step 1: Dry-Run a Rotation Configuration (`-d`)
The `-d` (debug) flag tests your configuration without modifying any files or executing shell scripts:
```bash
sudo logrotate -d /etc/logrotate.d/nginx
```
```
reading config file /etc/logrotate.d/nginx
Reading state from file: /var/lib/logrotate/status
Allocating hash table for state file, entries: 87

Handling 1 logs

rotating pattern: /var/log/nginx/*.log  after 1 days (14 rotations)
empty log files are not rotated, old logs are removed
considering log /var/log/nginx/access.log
  log does not need rotating (log is empty)
considering log /var/log/nginx/error.log
  log needs rotating
rotating log /var/log/nginx/error.log, log->rotateCount is 14
dateext suffix '-20261014'
glob pattern '-[0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9]'
renaming /var/log/nginx/error.log to /var/log/nginx/error.log.1
creating new /var/log/nginx/error.log mode = 0640 uid = 33 gid = 4
running postrotate script
```

### Step 2: Force Immediate Execution (`-f`)
Force logrotate to execute regardless of age or size:
```bash
sudo logrotate -f /etc/logrotate.d/nginx
```

### Step 3: Inspect the Logrotate State File
Logrotate records the timestamp of the last rotation in `/var/lib/logrotate/status`:
```bash
head -n 10 /var/lib/logrotate/status
```
```
logrotate state -- version 2
"/var/log/nginx/access.log" 2026-10-14-7:0:0
"/var/log/nginx/error.log" 2026-10-14-7:30:1
```

---

## 6. Verification & Troubleshooting Runbook

### Service & Timer Verification
```bash
# Check status of systemd logrotate timer
systemctl status logrotate.timer

# View execution history of logrotate service
journalctl -u logrotate.service --since yesterday
```

### Common Pitfalls & Solutions
1. **`error: skipping "/var/log/app/app.log" because parent directory has insecure permissions`**:
   - Cause: Directory is world-writable or owned by an unprivileged user (CVE-2011-1015 protection).
   - Solution: Set secure ownership: `sudo chmod 0755 /var/log/app && sudo chown root:root /var/log/app` or add `su <user> <group>` directive inside the logrotate config block.
2. **Log file stops receiving new logs after rotation**:
   - Cause: Process is still writing to the renamed file (`app.log.1`) because `postrotate` failed to signal the daemon, or `copytruncate` was required.
   - Solution: Verify `lsof /var/log/app/app.log.1`. If process holds open descriptor, reload daemon (`systemctl reload app`) and fix `postrotate` script.
3. **Log files rotate multiple times a day unexpectedly**:
   - Cause: `size` directive configured too low on a high-throughput logging server.
   - Solution: Tune size threshold or adjust application logging verbosity from `DEBUG` to `INFO`/`WARN`.

---

## 7. Key Takeaways
- Logrotate manages automated rotation, compression, retention, and truncation to prevent disk full outages.
- Modular configurations live in `/etc/logrotate.d/`.
- Use `create` + `postrotate` signaling whenever possible to avoid the data loss inherent in `copytruncate`.
- Always validate configurations with `logrotate -d` (dry-run) before committing changes to production.
