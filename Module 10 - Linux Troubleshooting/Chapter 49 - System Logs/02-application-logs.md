# Application Logs: Architecture, Collection, and Triage

## 1. Architectural Concept & Deep Dive

In Linux production systems, **application logs** capture internal business logic, database queries, API transactions, error traces, and runtime exceptions. 

Historically, applications ran as daemons and wrote directly to dedicated log files in `/var/log/<service>/` (e.g., `/var/log/nginx/error.log`, `/var/log/mysql/error.log`). In modern containerized and cloud-native architectures (12-Factor App design), applications log directly to **Standard Output (`stdout`)** and **Standard Error (`stderr`)**, delegating log capture, buffering, rotation, and persistence entirely to the Linux host runtime (`systemd`, `containerd`, or `docker`).

```
+─────────────────────────────────────────────────────────────────────────+
|                    APPLICATION LOG ARCHITECTURE PATTERNS                |
|                                                                         |
|  Pattern A: Legacy Host Daemon          Pattern B: 12-Factor Container  |
|  +─────────────────────────────+        +─────────────────────────────+ |
|  |       App Process           |        |       App Process           | |
|  |             │               |        |             │               | |
|  |             ▼               |        |             ▼               | |
|  |     /var/log/app.log        |        |       stdout / stderr       | |
|  |   (Direct disk write)       |        |    (Non-blocking pipes)     | |
|  +─────────────┬───────────────+        +─────────────┬───────────────+ |
|                │                                      │                 |
|                ▼                                      ▼                 |
|       Rotated by logrotate                   Captured by containerd     |
|                                              /var/log/pods/...          |
|                │                                      │                 |
|                └──────────────────┬───────────────────┘                 |
|                                   ▼                                     |
|                      Shipper (Promtail / Fluentbit)                     |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Common Application Log Locations on Linux

| Application / Service | Default Linux Paths | Key Purpose |
| :--- | :--- | :--- |
| **Nginx** | `/var/log/nginx/access.log`<br>`/var/log/nginx/error.log` | HTTP request telemetry and reverse proxy upstream connection errors. |
| **Apache HTTPD** | `/var/log/apache2/` (Debian)<br>`/var/log/httpd/` (RHEL) | Web server access and VirtualHost errors. |
| **PostgreSQL** | `/var/log/postgresql/postgresql-*.log` | Slow query logs, deadlocks, connection resets, replication status. |
| **MySQL / MariaDB** | `/var/log/mysql/error.log`<br>`/var/log/mysql/slow.log` | Query execution latency and storage engine crashes. |
| **Docker / Containerd** | `/var/log/pods/<POD_UID>/`<br>`/var/lib/docker/containers/<ID>/` | Raw containerized stdout/stderr JSON or CRI streams. |

---

## 3. Formatting Best Practices: Structured JSON vs Unstructured

Unstructured plaintext logs require fragile regex patterns that break whenever developers alter spacing or wording. **Structured JSON logging** is the industry standard for production services.

### Unstructured (Fragile):
```
2026-10-14 07:22:15 ERROR [Worker-3] payment_gateway: Charge failed for user 45012: Card declined (Bank timeout)
```

### Structured JSON (Machine-Parseable):
```json
{
  "timestamp": "2026-10-14T07:22:15.102Z",
  "level": "error",
  "service": "billing-api",
  "trace_id": "c4b12f80-2a9c-4b44-85cb-299a3c7f6a94",
  "span_id": "a9c4b4485cb299a3",
  "thread": "Worker-3",
  "user_id": 45012,
  "error": "Card declined",
  "reason": "Bank timeout",
  "http_status": 504
}
```

### Triage JSON Logs from the Terminal with `jq`
```bash
# Query all failed transactions with trace_ids for the billing service
cat /var/log/app/production.log | jq -r 'select(.level=="error" and .service=="billing-api") | "\(.timestamp) [\(.trace_id)] User \(.user_id) failed: \(.reason)"'
```

---

## 4. The Linux I/O Buffering Trap & `stdbuf`

When processes write to an interactive terminal, the standard C library (`glibc`) operates in **line-buffered mode** (flushing after every `\n`). 

However, when standard output is redirected to a file or piped to another process (`app | grep ERROR`), glibc automatically switches to **block-buffered mode** (typically 4 KB or 8 KB buffer). 

### The Symptom:
A developer tails a pipe (`python app.py | grep error`), and nothing appears for minutes, then suddenly 100 lines arrive in a single burst.

### The Fix: `stdbuf`
Force unbuffered or line-buffered I/O using `stdbuf`:
```bash
# Force stdout to be line-buffered (L) and stderr unbuffered (0)
stdbuf -oL -e0 python app.py | grep --line-buffered "error"
```

---

## 5. Correlating Logs Across Microservices: Trace IDs

In distributed environments, a single user click traverses multiple microservices:
`Client -> Ingress -> API Gateway -> Auth -> Order Service -> Postgres`.

If the Order Service fails, searching for generic error strings is futile. Systems inject a unique **Trace ID / Request ID** (e.g. `X-Request-ID` or W3C `traceparent` header) at the edge, propagating it across all downstream calls.

```bash
# Grep across ALL application logs on a node for a specific transaction ID
grep -r "c4b12f80-2a9c-4b44-85cb-299a3c7f6a94" /var/log/app/
```

---

## 6. Verification & Troubleshooting Runbook

### Application Logging Health Checklist
```bash
# Verify process has open log file descriptors
lsof -p <PID> | grep "\.log"

# Measure log write throughput per second
sudo pidstat -d 1 5 -p <PID>

# Check if application logging is saturating disk I/O
iotop -o -b -n 2
```

### Common Pitfalls & Solutions
1. **Disk I/O latency spikes caused by logging**:
   - Cause: Synchronous file logging on high-concurrency services. Every `log.Info()` blocks the thread until disk write succeeds.
   - Solution: Use asynchronous non-blocking loggers (e.g., Zap in Go, Log4j AsyncAppender, Winston async).
2. **Container logs consume entire disk in `/var/lib/docker`**:
   - Cause: Default Docker json-file driver has no max size configured.
   - Solution: Configure global rotation in `/etc/docker/daemon.json`:
     ```json
     {
       "log-driver": "json-file",
       "log-opts": {
         "max-size": "100m",
         "max-file": "5"
       }
     }
     ```
3. **Multi-line stack traces split into separate lines**:
   - Cause: Log shippers (Fluentbit, Promtail) treat each `\n` as an independent event.
   - Solution: Configure multi-line parser rules (regex matching leading timestamp `^\d{4}-\d{2}-\d{2}`) or emit single-line JSON logs with escaped `\n` inside the error field.

---

## 7. Key Takeaways
- Modern 12-factor applications log to `stdout`/`stderr`, leaving persistence to host container runtimes.
- Structured JSON logging enables machine querying and automated metrics extraction via tools like `jq`.
- `stdbuf` resolves stdout pipe buffering delays during live terminal debugging.
- Trace IDs (`trace_id`) are indispensable for correlating logs across distributed microservices.
