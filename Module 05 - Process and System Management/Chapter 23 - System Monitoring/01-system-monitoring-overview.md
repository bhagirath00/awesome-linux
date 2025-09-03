# Chapter 23: System Monitoring - System Monitoring Overview

## 1. Introduction: The USE and RED Methods

System monitoring is the disciplined practice of measuring, observing, and analyzing system resources and application metrics to detect performance degradations, capacity bottlenecks, and hardware failures before they cause production outages.

In enterprise systems engineering, monitoring methodology is guided by two industry-standard operational frameworks:

### 1. The USE Method (Resource-Centric - Brendan Gregg)
For every hardware and software resource (CPUs, Disks, Memory, Network Busses), analyze:
- **Utilization**: The percentage of time the resource was busy performing work (e.g., CPU % busy).
- **Saturation**: The degree to which extra work is queued waiting for the resource (e.g., Runqueue length, I/O queue depth).
- **Errors**: Hardware and filesystem error counts (e.g., dropped network packets, SMART disk reallocated sectors).

### 2. The RED Method (Service-Centric - Tom Wilkie)
For microservices, APIs, and request-driven applications, measure:
- **Rate**: Number of requests processed per second.
- **Errors**: Number of failed requests per second (e.g., HTTP 5xx responses).
- **Duration**: Latency or time taken to complete each request (P95, P99 percentiles).

```
+--------------------------------------------------------------------------+
|                       Linux Subsystem Monitoring Map                     |
+--------------------------------------------------------------------------+
|  CPU & Scheduler      Memory & Paging        Block I/O / Storage         |
|  - uptime / top       - free -h              - iostat -xz 1              |
|  - vmstat 1           - vmstat 1 (si/so)     - df -hT / du -sh           |
|  - mpstat -P ALL 1    - /proc/meminfo        - iotop                     |
|                                                                          |
|  Networking           Operating System Files Historical Performance      |
|  - ss -tulpn          - lsof                 - sar -A                    |
|  - ip -s link         - /proc/sys/fs/file-nr - atop / node_exporter      |
+--------------------------------------------------------------------------+
```

---

## 2. Core Linux Subsystems to Monitor

| System Subsystem | Primary Metric | Primary CLI Tools | Bottleneck Warning Sign |
| :--- | :--- | :--- | :--- |
| **CPU** | Runqueue depth & Steal time | `uptime`, `vmstat`, `top`, `mpstat` | Load average significantly exceeds available CPU cores. |
| **Memory** | Available RAM & Swap thrashing | `free -m`, `vmstat` (`si`/`so`), `/proc/meminfo` | Non-zero `so` (swap out) activity combined with low `avail Mem`. |
| **Storage / Disk** | Await latency & %util | `iostat -xz`, `iotop`, `df -h` | `%util` > 80% or `await` > 10ms on solid-state drives. |
| **Network** | Throughput & packet drops | `ip -s link`, `ss -s`, `sar -n DEV` | Dropped frames (`rx_dropped`), socket queue overflow. |
| **Kernel File Descriptors**| Allocated vs max handles | `cat /proc/sys/fs/file-nr`, `lsof` | Open file count approaches system limit (`file-max`). |

---

## 3. Real-Time vs. Historical Monitoring

1. **Point-in-Time Diagnostics**:
   Interactive tools (`top`, `vmstat`, `iostat`, `htop`, `lsof`) used by an engineer actively logged into an SSH terminal to triage an active incident in real time.

2. **Continuous Historical Telemetry**:
   System agents (`sar` via `sysstat`, `node_exporter` for Prometheus, Datadog Agent, Telegraf) that scrape OS metrics continuously every 10–60 seconds, store them in a Time-Series Database (TSDB), and visualize them in Grafana dashboards with alert thresholds.

---

## 4. The Linux Virtual Filesystems: `/proc` and `/sys`

Virtually all Linux monitoring tools are simply formatted wrappers around the kernel's virtual runtime filesystems:

```bash
# Direct CPU information
cat /proc/cpuinfo

# Detailed memory pages and dirty caches
cat /proc/meminfo

# Global system load average (1m, 5m, 15m, runnable/total, last PID)
cat /proc/loadavg
```

```
0.42 0.38 0.25 2/342 28412
```

---

## 5. Summary Takeaways

- Apply the **USE method** (Utilization, Saturation, Errors) for hardware resources.
- High load average does not automatically mean high CPU usage; check I/O wait (`wa`) and disk sleep (`D`).
- Understand that Linux caching (`buff/cache`) is healthy and reclaimed dynamically.
- Modern observability pipelines bridge local CLI tools with Prometheus and Grafana for long-term telemetry.
