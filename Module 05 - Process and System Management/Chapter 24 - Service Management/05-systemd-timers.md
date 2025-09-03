# Chapter 24: Service Management - systemd Timers (Modern Cron Alternative)

## 1. Introduction: Why systemd Timers?

For decades, the standard tool for executing scheduled recurring tasks in Linux was **`cron`**. However, in modern Linux environments, **systemd Timers** (`.timer` units) provide a vastly superior, observable, and enterprise-grade alternative.

```
+--------------------------------------------------------------------------+
|                        cron vs. systemd Timers                           |
+--------------------------------------------------------------------------+
|  Feature                      Cron                    systemd Timer      |
+--------------------------------------------------------------------------+
|  Logging                      Silent (Needs mail/syslog)| Full journalctl logs|
|  Resource Limits              None (Can starve CPU)   | Memory/CPU Cgroups |
|  Missed Run Catch-up          Needs anacron           | Persistent=true    |
|  Failure Alerts               None / Hard to monitor  | OnFailure= hook    |
|  Execution Independence       Runs inside cron shell  | Runs in clean unit |
|  Time Resolution              1 Minute                | 1 Millisecond      |
+--------------------------------------------------------------------------+
```

---

## 2. The Two-File Architecture

A systemd timer setup always consists of **two companion files** with matching base names in `/etc/systemd/system/`:

1. **The Service File (`app-backup.service`)**: Defines **WHAT** task to execute (`Type=oneshot`).
2. **The Timer File (`app-backup.timer`)**: Defines **WHEN** to trigger the matching service.

```
+-----------------------------+           Triggers
|     app-backup.timer        | ---------------------------> +------------------------------+
| [Timer]                     |                              |      app-backup.service      |
| OnCalendar=*-*-* 02:00:00   |                              | [Service]                    |
| Persistent=true             |                              | Type=oneshot                 |
+-----------------------------+                              | ExecStart=/opt/bin/backup.sh |
                                                             +------------------------------+
```

---

## 3. Step-by-Step Implementation Example

Let's build a scheduled database backup that executes every night at 02:00 AM.

### Step 1: Create the Companion Service (`/etc/systemd/system/db-backup.service`)
```ini
[Unit]
Description=Nightly PostgreSQL Database Backup Service

[Service]
Type=oneshot
User=postgres
ExecStart=/usr/local/bin/backup-postgres.sh
```

### Step 2: Create the Timer Unit (`/etc/systemd/system/db-backup.timer`)
```ini
[Unit]
Description=Trigger Nightly PostgreSQL Database Backup
Requires=db-backup.service

[Timer]
# Run every day at 02:00:00 AM UTC
OnCalendar=*-*-* 02:00:00

# Add random jitter (up to 15m) to avoid simultaneous disk spikes across 100 VMs
RandomizedDelaySec=15m

# Catch up missed runs if server was powered off at 02:00 AM
Persistent=true

[Install]
WantedBy=timers.target
```

### Step 3: Enable and Start the Timer
```bash
# Reload systemd
sudo systemctl daemon-reload

# Enable and start the TIMER (not the service!)
sudo systemctl enable --now db-backup.timer
```

---

## 4. Timer Types: Monotonic vs. Real-Time (Calendar)

### 1. Monotonic Timers (Relative to an Event)
Trigger tasks relative to boot time or service activation:
- `OnBootSec=15min`: Triggers 15 minutes after system boots up.
- `OnUnitActiveSec=1h`: Triggers 1 hour after the associated service was last activated.

```ini
[Timer]
OnBootSec=10min
OnUnitActiveSec=2h
```

### 2. Real-Time / Calendar Timers (`OnCalendar=`)
Trigger tasks at specific calendar dates and times (equivalent to cron expressions):
- `OnCalendar=hourly`: Run every hour at minute 0.
- `OnCalendar=daily`: Run every day at midnight (00:00:00).
- `OnCalendar=weekly`: Run every Monday at 00:00:00.
- `OnCalendar=Mon..Fri *-*-* 09:00:00`: Run Monday through Friday at 9:00 AM.
- `OnCalendar=*-*-01 04:00:00`: Run on the 1st of every month at 04:00 AM.

> [!TIP]
> Test calendar syntax before deploying using `systemd-analyze calendar`:
> ```bash
> systemd-analyze calendar "Mon..Fri *-*-* 09:00:00"
> ```
> ```
>   Original form: Mon..Fri *-*-* 09:00:00
> Normalized form: Mon..Fri *-*-* 09:00:00
>     Next elapse: Mon 2026-10-19 09:00:00 UTC
>        (in 4d 22h)
> ```

---

## 5. Monitoring and Managing Active Timers

### Listing All Active Timers (`list-timers`)
```bash
systemctl list-timers
```

```
NEXT                         LEFT          LAST                         PASSED       UNIT                         ACTIVATES
Wed 2026-10-14 06:00:00 UTC  45min left    Wed 2026-10-14 05:00:00 UTC  14min ago    sysstat-collect.timer        sysstat-collect.service
Thu 2026-10-15 02:00:00 UTC  20h left      Wed 2026-10-14 02:00:00 UTC  3h 14min ago db-backup.timer              db-backup.service
```

### Checking Execution Logs
```bash
journalctl -u db-backup.service
```

---

## 6. Summary Takeaways

- systemd Timers replace `cron` with full journal logging, cgroup resource limits, and failure handling.
- Requires two matching files: `.service` (task logic) and `.timer` (schedule trigger).
- Always enable the **`.timer`** unit, not the service unit.
- Use `Persistent=true` to automatically execute missed runs after downtime.
- Verify schedule formatting with `systemd-analyze calendar`.
