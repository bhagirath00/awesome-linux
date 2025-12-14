# Chapter 25: Scheduling Tasks - Cron Overview and Architecture

## 1. Introduction: What is Cron?

**`cron`** is the traditional, time-based job scheduler in Unix-like computer operating systems. First introduced in Version 7 Unix in 1979 by Brian Kernighan, cron enables system administrators and automated accounts to schedule jobs (commands, maintenance shell scripts, database backups, log archiving) to run periodically at fixed times, dates, or intervals.

The execution engine is the background daemon **`crond`** (or **`cron`** on Debian/Ubuntu). It starts during system boot and remains sleeping in the background, waking up **once every minute** to inspect system and user schedule files (known as **crontabs**). If any scheduled entry matches the current minute, hour, day, and month, `cron` forks a subshell and executes the designated command.

```
+--------------------------------------------------------------------------+
|                         cron Daemon Architecture                         |
+--------------------------------------------------------------------------+
|  crond Daemon (Wakes up at the 00 second mark of every minute)           |
|                                |                                         |
|                                v Scans Schedule Sources                  |
|  +-----------------------------+---------------------------------------+ |
|  | System-Wide Crontabs:       | User Crontabs:                        | |
|  | - /etc/crontab              | - /var/spool/cron/crontabs/<username> | |
|  | - /etc/cron.d/*             |   (Managed strictly via `crontab -e`) | |
|  | - /etc/cron.{hourly,daily,  |                                       | |
|  |          weekly,monthly}/   |                                       | |
|  +-----------------------------+---------------------------------------+ |
|                                |                                         |
|                                v Matches current time: 02:00:00 AM?      |
|                                |                                         |
|                                v Fork Subshell                           |
|        [ /bin/sh -c "/opt/scripts/database-backup.sh" ]                  |
|                                |                                         |
|                                v Captures stdout / stderr                |
|  +-----------------------------+---------------------------------------+ |
|  | If output produced: Send email via local MTA (sendmail / postfix)  | |
|  | OR discard if redirected to /dev/null                               | |
+--------------------------------------------------------------------------+
```

---

## 2. Managing the cron Daemon

The cron daemon is managed via systemd under the name `cron.service` (Debian/Ubuntu) or `crond.service` (RHEL/CentOS/Rocky/Fedora):

```bash
# Verify daemon operational status
sudo systemctl status cron
# On RHEL/CentOS:
# sudo systemctl status crond

# Enable and start daemon on boot
sudo systemctl enable --now cron
```

---

## 3. The Hierarchy of Cron Schedules

Linux systems organize cron jobs into two distinct categories: **System-Wide Crontabs** and **User-Specific Crontabs**.

| Storage Location | Managed By | User Specification Field? | Primary Use Case |
| :--- | :--- | :--- | :--- |
| **`/var/spool/cron/`** | Individual users via `crontab -e` | **No** (runs as file owner) | Developer scripts, personal syncs, application tasks. |
| **`/etc/crontab`** | Root superuser directly | **Yes** (6th column specifies username) | Legacy system-wide tasks. |
| **`/etc/cron.d/`** | Packages & Root administrators | **Yes** (6th column specifies username) | Modular application jobs (e.g., certbot renewal, sysstat). |
| **`/etc/cron.hourly/`** | Drop-in executable scripts | **No** (runs as root) | Hourly system maintenance scripts. |
| **`/etc/cron.daily/`** | Drop-in executable scripts | **No** (runs as root) | Daily log rotation, package update checks. |
| **`/etc/cron.weekly/`** | Drop-in executable scripts | **No** (runs as root) | Weekly filesystem checks, database vacuuming. |
| **`/etc/cron.monthly/`**| Drop-in executable scripts | **No** (runs as root) | Monthly compliance reports, archive purges. |

---

## 4. The Drop-In Directories (`/etc/cron.daily/`, etc.)

Rather than editing text crontabs with complex time expressions, system administrators can simply place standard executable shell scripts directly inside `/etc/cron.daily/`, `/etc/cron.hourly/`, etc.:

```bash
# Create a daily backup script in /etc/cron.daily/
sudo tee /etc/cron.daily/backup-web << 'EOF'
#!/bin/bash
tar -czf /backup/web_$(date +%F).tar.gz /var/www/html/
EOF

# Ensure script is executable (CRITICAL!)
sudo chmod +x /etc/cron.daily/backup-web
```

> [!WARNING]
> Scripts placed inside `/etc/cron.daily/` and `/etc/cron.hourly/` are executed via `run-parts`. Scripts **must be executable (`chmod +x`)** and must **not contain dots (`.`) or file extensions** in Debian/Ubuntu (e.g., name it `backup-web`, NOT `backup-web.sh`), otherwise `run-parts` will silently skip execution!

---

## 5. Summary Takeaways

- `cron` executes scheduled jobs periodically by checking crontab files once every minute.
- User-specific schedules live in `/var/spool/cron/` and are managed via `crontab -e`.
- Modular system jobs live in `/etc/cron.d/` and include an explicit username column.
- Directory drop-ins (`/etc/cron.daily/`) must have executable permissions (`+x`) to run.
