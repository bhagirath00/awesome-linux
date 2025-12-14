# Chapter 25: Scheduling Tasks - Scheduling Best Practices and Pitfalls

## 1. Introduction: The Realities of Automated Task Execution

While task scheduling via `cron` appears straightforward, unattended automated background execution is one of the most common sources of subtle production outages. 

Jobs that run cleanly when manually tested by an engineer in an interactive shell often fail silently when triggered by `cron` due to stripped environment variables, missing search paths, overlapping processes, or unhandled disk output.

---

## 2. Pitfall 1: The Stripped-Down cron Environment

When you log in interactively via SSH, your shell reads `~/.bashrc`, `~/.profile`, and `/etc/profile`, populating environment variables such as `PATH`, `JAVA_HOME`, `NODE_ENV`, and terminal settings.

**Cron does NOT load these files!**

By default, cron runs commands in a minimal `/bin/sh` shell with a severely truncated `PATH` (typically only `/usr/bin:/bin`). If your script relies on binaries located in `/usr/local/bin`, `/opt/bin`, or custom virtual environments (`python`, `docker`, `aws`, `node`), the script fails with:
```
command not found
```

### The Solutions:
1. **Always use absolute paths**:
   ```crontab
   # Insecure / Fragile:
   0 2 * * * backup.sh
   
   # Robust / Production-Ready:
   0 2 * * * /usr/local/bin/backup.sh
   ```

2. **Explicitly define PATH and SHELL at the top of the crontab**:
   ```crontab
   SHELL=/bin/bash
   PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
   MAILTO=devops-alerts@company.com
   
   30 2 * * * /opt/scripts/export-data.sh
   ```

---

## 3. Pitfall 2: Overlapping Executions (Job Stacking)

Consider a backup script scheduled to run every 5 minutes:
```crontab
*/5 * * * * /opt/scripts/sync-data.sh
```

Under normal circumstances, the script completes in 60 seconds. However, if network latency spikes or database volume increases, the job might take 15 minutes. 

Without protection, **cron will mindlessly launch a new instance of the script every 5 minutes**, resulting in multiple concurrent processes fighting for disk I/O, deadlocking databases, and causing a catastrophic system freeze!

### The Solution: Use `flock` (File Locking)
The `flock` utility prevents a command from running if a lock file is already held by a prior running instance:

```crontab
# -n: Non-blocking (fail and exit immediately if lock is already held)
*/5 * * * * /usr/bin/flock -n /var/lock/sync-data.lock /opt/scripts/sync-data.sh
```

---

## 4. Pitfall 3: Capturing Standard Output and Error

By default, `cron` attempts to email any output generated on `stdout` or `stderr` to the local user account using the local mail transfer agent (`sendmail` / `postfix`). If a mail server is not configured, the output is discarded into system logs or fills `/var/mail/`.

### The Rule: Always Explicitly Redirect Output
```crontab
# Redirect stdout and stderr to a dedicated log file
0 3 * * * /opt/bin/db-backup.sh >> /var/log/db-backup.log 2>&1

# Discard output completely only for non-critical, spammy jobs
* * * * * /usr/bin/ping-worker.sh > /dev/null 2>&1
```

---

## 5. Pitfall 4: Percent Signs (`%`) in Crontab Commands

In crontab lines, the percent sign (`%`) has a special meaning: unless escaped with a backslash (`\`), **the first `%` is translated into a newline**, and all subsequent data is passed to standard input!

### Broken:
```crontab
# FAILS: cron interprets %Y as a newline!
0 2 * * * tar -czf /backup/db_$(date +%Y-%m-%d).tar.gz /data
```

### Fixed:
```crontab
# Escape with backslashes:
0 2 * * * tar -czf /backup/db_$(date +\%Y-\%m-\%d).tar.gz /data
```
*(Alternatively, place the complex command inside a dedicated `.sh` shell script and invoke the script).*

---

## 6. Enterprise Scheduling Checklist

| Best Practice Rule | Production Recommendation |
| :--- | :--- |
| **Absolute Paths** | Always use `/usr/bin/python3`, `/bin/tar`, `/usr/local/bin/app`. |
| **Locking** | Wrap frequent tasks (`*/5`) with `flock -n` to prevent pileups. |
| **Logging** | Always redirect output with `>> /var/log/app.log 2>&1`. |
| **Log Rotation** | Add your custom cron log files to `/etc/logrotate.d/`. |
| **Time Jitter** | Add random sleep delays (`sleep $((RANDOM \% 60))`) to prevent fleet-wide spikes. |
| **Timezone Awareness** | Ensure system timezone (`timedatectl`) aligns with business scheduling expectations. |

---

## 7. Summary Takeaways

- Cron runs in an isolated, minimal `/bin/sh` shell without your interactive `PATH`.
- Use `flock` to guarantee jobs never overlap or stack during high load.
- Escape percent signs (`\%`) when using the `date` command in crontabs.
- Redirect `stdout` and `stderr` to persistent log files for auditing and debugging.
- Consider modern **systemd timers** for critical workloads requiring deep observability.
