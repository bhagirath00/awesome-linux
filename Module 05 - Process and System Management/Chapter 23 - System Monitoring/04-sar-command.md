# Chapter 23: System Monitoring - Historical Telemetry with sar

## 1. Introduction: What is sar?

`sar` (**System Activity Report**) is the historical telemetry backbone of the Linux `sysstat` toolkit. 

While commands like `top`, `vmstat`, and `iostat` only reveal what is happening **right now**, incidents often happen in the middle of the night or hours before an engineer investigates. `sar` solves this by collecting system performance counters continuously in the background at regular intervals (typically every 10 minutes), storing binary log files in `/var/log/sysstat/` (or `/var/log/sa/`), and allowing retrospective querying of past CPU, memory, disk, network, and load metrics.

```
+-------------------------------------------------------------------------+
|                         sar System Architecture                         |
+-------------------------------------------------------------------------+
|  Continuous Background Collector                                        |
|  [ sysstat.service / cron / systemd timer ] (Every 10 mins)             |
|                                |                                        |
|                                v Writes daily binary logs               |
|            /var/log/sysstat/sa14 (Binary data for 14th)                 |
|                                |                                        |
|  Engineer Incident Investigation (e.g., at 09:00 AM)                    |
|  "Why did database crash at 03:30 AM last night?"                       |
|                                |                                        |
|  sar -u -f /var/log/sysstat/sa14 -s 03:00:00 -e 04:00:00                |
|                                v                                        |
|  [ Formatted historical CPU report showing 100% saturation at 03:28 AM ]|
+-------------------------------------------------------------------------+
```

---

## 2. Enabling and Configuring the sysstat Service

On many Linux distributions, `sysstat` is installed with background data collection disabled by default.

### On Ubuntu / Debian:
```bash
# 1. Install package
sudo apt install -y sysstat

# 2. Enable collection in /etc/default/sysstat
sudo sed -i 's/ENABLED="false"/ENABLED="true"/' /etc/default/sysstat

# 3. Start and enable systemd timer
sudo systemctl enable --now sysstat
```

### On RHEL / Rocky Linux / AlmaLinux:
```bash
sudo dnf install -y sysstat
sudo systemctl enable --now sysstat
```

---

## 3. Real-Time Diagnostics with sar

In addition to querying past logs, `sar` can be run dynamically like `vmstat`:

```bash
# CPU utilization every 2 seconds, 3 times
sar -u 2 3
```

```
10:14:02 AM     CPU     %user     %nice   %system   %iowait    %steal     %idle
10:14:04 AM     all      5.12      0.00      1.84      0.21      0.00     92.83
10:14:06 AM     all      6.40      0.00      2.10      0.15      0.00     91.35
10:14:08 AM     all      4.88      0.00      1.65      0.10      0.00     93.37
Average:        all      5.47      0.00      1.86      0.15      0.00     92.52
```

---

## 4. Querying Historical Metrics with sar Flags

| Flag | Subsystem Monitored | Example Metric Columns |
| :--- | :--- | :--- |
| **`-u`** | CPU Utilization (Default) | `%user`, `%system`, `%iowait`, `%idle` |
| **`-r`** | Memory Utilization | `kbmemfree`, `kbmemused`, `%memused`, `kbbuffers`, `kbcached` |
| **`-S`** | Swap Utilization | `kbswpfree`, `kbswpused`, `%swpused` |
| **`-b`** | I/O and Transfer Rates | `tps`, `rtps`, `wtps`, `bread/s`, `bwrtn/s` |
| **`-d`** | Disk Device Performance | `tps`, `rkB/s`, `wkB/s`, `await`, `%util` |
| **`-n DEV`**| Network Interface Traffic | `rxpck/s`, `txpck/s`, `rxkB/s`, `txkB/s` |
| **`-q`** | Load Average & Queue Length | `runq-sz`, `plist-sz`, `ldavg-1`, `ldavg-5`, `ldavg-15` |
| **`-w`** | Context Switching & Task Creation | `proc/s`, `cswch/s` |
| **`-A`** | All Metrics Combined | Comprehensive system dump |

---

## 5. Post-Mortem Incident Analysis Examples

### 1. Inspecting Memory Exhaustion During Last Night's Incident
Query memory metrics between 02:00 AM and 04:00 AM from today's log:
```bash
sar -r -s 02:00:00 -e 04:00:00
```

### 2. Inspecting Yesterday's Network Interface Traffic
Binary files are named after the day of the month (e.g., `sa13` for the 13th):
```bash
sar -n DEV -f /var/log/sysstat/sa13 | grep eth0
```

### 3. Checking Historical Load Averages
```bash
sar -q -f /var/log/sysstat/sa14
```

---

## 6. Summary Takeaways

- `sar` provides historical performance analysis without needing external agent software.
- Ensure the `sysstat` daemon or systemd timer is enabled and active.
- Daily binary records are saved in `/var/log/sysstat/sa<day>`.
- Use time window flags (`-s HH:MM:SS` and `-e HH:MM:SS`) to isolate exact incident windows.
- Key flags: `-u` (CPU), `-r` (RAM), `-d` (Disk), `-n DEV` (Network), `-q` (Load).
