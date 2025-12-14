# Chapter 24: Service Management - The journalctl Command (System Logging)

## 1. Introduction: What is systemd-journald?

`systemd-journald` is the unified system logging daemon that collects, indexes, and stores logging data from the entire operating system:
- Kernel log ring buffers (`dmesg` / `/dev/kmsg`)
- Standard system log calls via POSIX `syslog(3)`
- Standard output (`stdout`) and standard error (`stderr`) of all systemd-supervised services
- Early-boot messages from initial RAM disks (initramfs)
- Audit records from the Linux audit framework

Unlike legacy syslog daemons (like `rsyslog` or `syslog-ng`) that write plain, unstructured text files into `/var/log/messages`, `journald` stores logs in **indexed binary format**. This allows instantaneous filtering by service unit, PID, UID, kernel subsystem, priority level, and precise timestamps via the **`journalctl`** query tool.

---

## 2. Basic Command Execution

Running `journalctl` without arguments opens the entire system journal starting from the oldest available record, paged via `less`:

```bash
journalctl
```

To jump directly to the most recent entries:
```bash
journalctl -e
```

---

## 3. The Essential Diagnostic Flags

### 1. Real-Time Following (`-f`) - Equivalent to `tail -f`
Stream new incoming log lines in real-time as they are written:
```bash
# Follow all system logs
journalctl -f

# Follow logs strictly for a specific service
journalctl -u nginx.service -f
```

### 2. Filtering by Service Unit (`-u`)
The most frequently used flag in daily development and operations:
```bash
journalctl -u docker.service
```

### 3. Filtering by Time Windows (`--since` and `--until`)
Query logs using human-readable relative or absolute timestamps:
```bash
# View logs from the last 30 minutes
journalctl --since "30 min ago"

# View logs between specific hours yesterday
journalctl --since "2026-10-13 14:00:00" --until "2026-10-13 15:30:00"

# View logs starting from today's midnight
journalctl --since today
```

### 4. Filtering by Current Boot (`-b`)
Ignore historical logs and show only messages generated since the last machine reboot:
```bash
# Current boot
journalctl -b

# Previous boot (useful for investigating why a server crashed/rebooted)
journalctl -b -1
```

### 5. Filtering by Log Severity / Priority (`-p`)
Filter by standard syslog priority levels (0 through 7):
`0: emerg`, `1: alert`, `2: crit`, `3: err`, `4: warning`, `5: notice`, `6: info`, `7: debug`.

```bash
# Show only errors, critical events, and emergencies
journalctl -p err..emerg

# Show only errors for SSH daemon
journalctl -u sshd -p err
```

### 6. Showing Kernel Messages Only (`-k`)
Displays Linux kernel hardware and driver ring buffers (identical to `dmesg`):
```bash
journalctl -k
```

---

## 4. Making Journal Logs Persistent Across Reboots

By default on some minimal distributions, `systemd-journald` is configured to store logs in memory (`/run/log/journal/`), which are erased when the server reboots!

To ensure system logs persist across reboots:

```bash
# 1. Create persistent directory
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal

# 2. Configure persistence in /etc/systemd/journald.conf
sudo sed -i 's/^#Storage=.*/Storage=persistent/' /etc/systemd/journald.conf

# 3. Restart journald daemon
sudo systemctl restart systemd-journald
```

---

## 5. Controlling Journal Disk Space Usage

Binary journals can grow over time. Check and prune disk consumption:

```bash
# Check current disk space consumed by journal files
journalctl --disk-usage
```

```
Archived and active journals take up 2.4G in the file system.
```

### Vacuuming / Pruning Old Logs:
```bash
# Retain at most 500MB of journal logs (deletes oldest archives)
sudo journalctl --vacuum-size=500M

# Retain at most 14 days of logs
sudo journalctl --vacuum-time=14d
```

---

## 6. Journalctl Command Cheat Sheet

| Command Syntax | Diagnostic Operation |
| :--- | :--- |
| `journalctl -u <unit> -f` | Real-time live log stream for a specific service. |
| `journalctl -u <unit> -n 50 --no-pager` | Print last 50 lines for a service without terminal paging. |
| `journalctl -b -p err` | Inspect all system errors during current boot. |
| `journalctl -b -1` | Inspect logs from the previous boot cycle (post-mortem). |
| `journalctl --since "1 hour ago"` | Slice logs by time window. |
| `journalctl -k` | Inspect kernel and hardware events (`dmesg`). |
| `journalctl --vacuum-size=1G` | Enforce disk usage cap on stored logs. |

---

## 7. Summary Takeaways

- `journalctl` queries the binary, indexed logs collected by `systemd-journald`.
- Use `-u <unit>` to isolate application microservices and daemons.
- Combine `-u <unit> -f` to tail logs during deployments.
- Inspect prior crash cycles using `journalctl -b -1`.
- Ensure `Storage=persistent` is enabled to retain logs across reboots.
