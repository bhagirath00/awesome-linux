# Production Log Analysis & Forensic Triage

## 1. Architectural Concept & Deep Dive

Logs are the historical and real-time black box flight recorder of the Linux operating system and its deployed applications. In production incidents, log analysis provides the deterministic proof required to transition from hypothesis to root-cause identification.

In modern Linux distributions (systemd-based), logging is bifurcated into:
1. **The Systemd Journal (`systemd-journald`)**: Binary, structured, indexed logging daemon capturing kernel ring buffer events, standard out/error from systemd units, and syslog messages.
2. **Traditional Syslog (`rsyslog` / `syslog-ng`)**: Plaintext files traditionally organized under `/var/log/` (`syslog`, `auth.log`, `messages`).
3. **Application Logs**: Plaintext or JSON structured files written to dedicated directories (e.g. `/var/log/nginx/access.log`, `/var/log/postgresql/`).

```
+─────────────────────────────────────────────────────────────────────────+
|                           LOG INGESTION FLOW                            |
|                                                                         |
|  Kernel Messages (/dev/kmsg) ───┐                                       |
|  Service stdout/stderr ─────────┼──► systemd-journald ──► /run/log      |
|  Syslog API (/dev/log) ─────────┘    (Indexed Binary)     /var/log/journal
|                                             │                           |
|                                             ├──► Forward to rsyslog     |
|                                             │    (Text in /var/log/*)   |
|                                             ▼                           |
|                             journalctl Query Engine / Promtail / Fluentbit
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Systemd Journal Mastery (`journalctl`)

The `journalctl` utility queries `systemd-journald`. Because journal entries are stored with indexed binary metadata (UID, PID, Unit Name, Priority, Boot ID, Timestamp), queries execute orders of magnitude faster than parsing text files.

### 1. Essential Journalctl Filter Flags

| Command / Flag | Purpose | Production Use Case |
| :--- | :--- | :--- |
| `journalctl -u nginx.service` | Filter by systemd service unit | Triage specific crashing daemon |
| `journalctl -b` | Current boot logs | Isolate logs since last reboot |
| `journalctl -b -1` | Previous boot logs | Investigate why node rebooted unexpectedly |
| `journalctl -p err..emerg` | Filter by priority level | Discard info/debug noise; show only errors/panics |
| `journalctl --since "15 min ago"` | Time-bounded window | Focus on exact alert trigger window |
| `journalctl -k` | Kernel ring buffer (`dmesg`) | Hardware failures, OOM kills, NVMe drops |
| `journalctl -f` | Real-time follow | Watch live log stream during deployment |
| `journalctl -u docker -o json-pretty` | Output in JSON format | Export structured events to external tools |

### 2. Syslog Priority Levels (RFC 5424)
Journalctl uses standard syslog severity levels:
- `0`: Emergency (`emerg`) - System is unusable
- `1`: Alert (`alert`) - Action must be taken immediately
- `2`: Critical (`crit`) - Critical conditions (hardware/subsystem)
- `3`: Error (`err`) - Error conditions (daemon failures)
- `4`: Warning (`warning`) - Warning conditions
- `5`: Notice (`notice`) - Normal but significant condition
- `6`: Informational (`info`) - Informational messages
- `7`: Debug (`debug`) - Debug-level messages

---

## 3. Command-Line Log Analysis Pipelines (`grep`, `awk`, `sed`, `jq`)

When analyzing multi-gigabyte production log files on a server, fast text manipulation using standard POSIX utilities is mandatory.

### Scenario A: Top 10 IP Addresses Flooding Nginx
```bash
# Nginx access log format: $remote_addr - $remote_user [$time_local] "$request" $status
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -n 10
```
```
 145021 203.0.113.45
  84120 198.51.100.12
  41200 192.0.2.89
```

### Scenario B: Calculate Distribution of HTTP Status Codes
```bash
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn
```
```
 241092 200
  12041 301
   4502 404
   1240 500
    814 502
```

### Scenario C: Filtering Structured JSON Logs with `jq`
Modern microservices log JSON payloads directly to file or stdout:

```json
{"level":"error","ts":"2026-10-14T06:12:01Z","caller":"db/pool.go:88","msg":"connection timed out","db_host":"10.0.1.20","latency_ms":5002}
```

Extract all unique database error messages and host targets:
```bash
cat /var/log/app/service.json.log | jq -r 'select(.level=="error") | "\(.ts) [\(.db_host)] \(.msg)"'
```

---

## 4. Hands-on Terminal Lab: Diagnosing a Crashed Service via Logs

### Step 1: Discover When the Service Crashed
```bash
systemctl status postgresql
```
```
● postgresql.service - PostgreSQL RDBMS
     Loaded: loaded (/lib/systemd/system/postgresql.service; enabled)
     Active: failed (Result: exit-code) since Wed 2026-10-14 06:14:22 UTC; 4min ago
    Process: 18451 ExecStart=/usr/lib/postgresql/15/bin/postgres (code=exited, status=1/FAILURE)
```

### Step 2: Query the Service Journal for the Exact Failure Window
```bash
journalctl -u postgresql --since "10 min ago" --no-pager
```
```
Oct 14 06:14:20 node1 postgres[18451]: 2026-10-14 06:14:20.102 UTC [18451] LOG:  starting PostgreSQL 15.3
Oct 14 06:14:21 node1 postgres[18451]: 2026-10-14 06:14:21.410 UTC [18451] FATAL:  could not access private key file "/etc/ssl/private/ssl-cert-snakeoil.key": Permission denied
Oct 14 06:14:22 node1 systemd[1]: postgresql.service: Main process exited, code=exited, status=1/FAILURE
Oct 14 06:14:22 node1 systemd[1]: postgresql.service: Failed with result 'exit-code'.
```
- **Root Cause Identified**: Clear filesystem permission issue on TLS private key!

---

## 5. Log Aggregation & Centralization Architecture

In multi-node production clusters, logs must never reside solely on the originating node. If a cloud instance terminates or an EC2 spot instance is interrupted, uncollected local logs are lost forever.

```
+───────────────────────────+       +───────────────────────────+
|        Node 01            |       |        Node 02            |
|  systemd-journal / files  |       |  systemd-journal / files  |
|             │             |       |             │             |
|             ▼             |       |             ▼             |
|    Promtail / Fluentbit   |       |    Promtail / Fluentbit   |
+─────────────┬─────────────+       +─────────────┬─────────────+
              │                                   │
              └─────────────────┬─────────────────┘
                                ▼
              +───────────────────────────────────+
              |   Central Log Cluster (Loki / ES) |
              |   - Persistent Storage (S3 / GCS) |
              |   - Search & Alerting (Grafana)   |
              +───────────────────────────────────+
```

---

## 6. Verification & Troubleshooting Runbook

### Log System Health Checklist
```bash
# Verify disk usage of systemd-journald
journalctl --disk-usage

# Verify journal integrity
journalctl --verify

# Vacuum journal logs older than 7 days or larger than 1GB
sudo journalctl --vacuum-time=7d
sudo journalctl --vacuum-size=1G
```

### Common Pitfalls & Solutions
1. **Journal logs vanish after reboot**:
   - Cause: Journal storage is set to `volatile` (stored in `/run/log/journal` tmpfs).
   - Solution: In `/etc/systemd/journald.conf`, set `Storage=persistent` and run `sudo systemctl restart systemd-journald`.
2. **Log file fills 100% of `/var` partition**:
   - Cause: Misconfigured application writing debug logs, or missing `logrotate` configuration.
   - Solution: Find huge logs with `du -ah /var/log | sort -rh | head -n 10`. Truncate without locking via `> /var/log/huge.log`.
3. **Application logging stops after manual file deletion (`rm logfile.log`)**:
   - Cause: Process still holds open file descriptor pointing to the deleted inode. Disk space is not freed and no new file is written.
   - Solution: Truncate instead of deleting (`truncate -s 0 file.log`), or signal the process to reopen file descriptors (`kill -HUP <PID>`).

---

## 7. Key Takeaways
- `journalctl` provides fast, indexed, metadata-rich queries across all system units and kernel logs.
- Use priority filtering (`-p err..emerg`) and time boundaries (`--since`) to quickly pinpoint errors during active incidents.
- Master POSIX text processing pipelines (`awk | sort | uniq -c | sort -nr`) and `jq` for high-speed terminal log triage.
- Never delete open log files directly with `rm`; truncate them to avoid inode leaks.
