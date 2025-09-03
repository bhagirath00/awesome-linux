# Chapter 25: Scheduling Tasks - One-Time Tasks with at and batch

## 1. Introduction: What is the at Command?

While `cron` is designed for **recurring, cyclical tasks** (e.g., every night at 2 AM), **`at`** is designed for **one-time, future task execution**.

Instead of creating a cron job and having to remember to delete it after it runs, you queue a job with `at`. The job executes once at the specified time, and its entry is automatically purged from the scheduling spool.

---

## 2. Managing the atd Daemon

The `at` command requires the **`atd`** daemon to be running in the background:

```bash
# Ubuntu / Debian
sudo apt install -y at
sudo systemctl enable --now atd

# RHEL / Rocky Linux / AlmaLinux
sudo dnf install -y at
sudo systemctl enable --now atd
```

---

## 3. Scheduling Jobs with at

`at` features a flexible natural-language time parsing parser:

### Method 1: Interactive Terminal Input
Run `at` followed by the target execution time, type your shell commands, and press **`Ctrl + D`** (EOF) to save and queue:

```bash
at 17:30
```

```
warning: commands will be executed using /bin/sh
at> /usr/local/bin/deploy-release.sh > /tmp/deploy.log 2>&1
at> echo "Deployment complete!" | mail -s "Release Status" admin@company.com
at> <EOT>
job 4 at Wed Oct 14 17:30:00 2026
```

### Method 2: Piped / Scripted Input (DevOps Automation)
Pipe commands directly into `at` without interactive prompts:

```bash
# Schedule a server reboot in 30 minutes
echo "sudo reboot" | at now + 30 minutes

# Schedule a backup at 2:00 AM tomorrow
echo "/opt/scripts/backup.sh" | at 2:00 AM tomorrow

# Schedule on a specific calendar date
echo "/opt/cleanup.sh" | at 04:00 11/15/2026
```

---

## 4. Natural Language Time Formats

The `at` parser accepts a wide variety of human-readable time specifications:

| Expression | Execution Time |
| :--- | :--- |
| `at 14:00` | At 2:00 PM today (or tomorrow if 2 PM has already passed). |
| `at 8:30 AM tomorrow` | At 8:30 AM the next day. |
| `at now + 1 hour` | Exactly 60 minutes from current system time. |
| `at now + 3 days` | Exactly 72 hours from current system time. |
| `at midnight` | At 00:00:00 tonight. |
| `at noon` | At 12:00:00 PM today. |
| `at teatime` | Traditional British afternoon tea time (16:00 / 4:00 PM). |
| `at 10:00 PM next week` | At 22:00 exactly 7 days from today. |

---

## 5. Managing Queued Jobs: atq and atrm

```bash
# List all currently pending one-time jobs (at queue)
atq
```

```
4	Wed Oct 14 17:30:00 2026 a devops
5	Wed Oct 14 18:00:00 2026 a devops
```
The columns display: **Job ID**, **Execution Timestamp**, **Queue Identifier** (`a`), and **Owner**.

### Inspecting Job Contents (`at -c`)
View the exact environment variables and command commands stored for job 4:
```bash
at -c 4
```

### Removing / Canceling a Queued Job (`atrm`)
```bash
# Cancel job ID 4
atrm 4
```

---

## 6. Load-Sensitive Scheduling with `batch`

`batch` (part of the `at` package) queues a task that runs **only when the system load average falls below a threshold** (typically below `1.5` or `0.8` times CPU count).

This is ideal for resource-intensive compilation, video processing, or database indexing jobs that should not disrupt active daytime traffic:

```bash
echo "/opt/heavy-analytics-job.sh" | batch
```

---

## 7. Summary Takeaways

- Use `at` for one-time future jobs; use `cron` for cyclical repeating jobs.
- Input commands interactively and save with `Ctrl + D`, or pipe via `echo "cmd" | at <time>`.
- Use `atq` to list queued jobs and `atrm <job_id>` to delete them.
- Use `batch` to execute resource-heavy tasks only when system load average is low.
