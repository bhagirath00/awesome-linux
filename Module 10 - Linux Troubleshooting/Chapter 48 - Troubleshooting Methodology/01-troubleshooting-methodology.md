# Production Linux Troubleshooting Methodology

## 1. Architectural Concept & Deep Dive

When a production incident strikes—whether it is latency degradation, cascading service timeouts, or total node unresponsiveness—haphazardly executing arbitrary commands causes diagnostic noise, wastes critical MTTR (Mean Time to Resolution), and can exacerbate the failure.

Elite Systems and SRE engineers employ formal performance and troubleshooting methodologies to systematically narrow down the root cause from the application layer down to the silicon:

```
+─────────────────────────────────────────────────────────────+
|               INCIDENT TRIAGE DECISION TREE                 |
|                                                             |
|   1. Observability Ingestion (Alert / Metrics / Customer)   |
|                               │                             |
|                               ▼                             |
|   2. The 60-Second Linux Health Check (Systemic State)      |
|                               │                             |
|          ┌────────────────────┴────────────────────┐        |
|          ▼                                         ▼        |
|   Resource Exhausted?                      Errors Detected? |
|   (USE Method: Saturation)                 (Dmesg, Syslog)  |
|          │                                         │        |
|          ▼                                         ▼        |
|   Subsystem Profiling                      Log Triage       |
|   (CPU, Memory, Disk I/O, Net)             (Auth, App, OOM) |
|          │                                         │        |
|          └────────────────────┬────────────────────┘        |
|                               ▼                             |
|   3. Process Deep Dive: strace / perf / lsof / gdb          |
|                               ▼                             |
|   4. Mitigation & Hypothesis Validation                     |
+─────────────────────────────────────────────────────────────+
```

---

## 2. Core Methodologies: USE vs RED

### The USE Method (Resource-Oriented)
Formulated by Brendan Gregg, the **USE Method** targets **every physical and virtual hardware resource** (CPUs, memory buses, disks, network interfaces, cgroups):
- **U - Utilization**: The average time that the resource was busy servicing work (e.g., 90% CPU busy, 100% disk utilization).
- **S - Saturation**: The degree to which extra work is queued waiting for the resource (e.g., CPU run-queue length, disk queue depth, memory swapping).
- **E - Errors**: The count of error events (e.g., dropped packets, PCIe bus errors, disk I/O retries).

### The RED Method (Request-Oriented)
Formulated by Tom Wilkie, the **RED Method** targets **application endpoints and microservices**:
- **R - Rate**: The number of requests serviced per second (throughput).
- **E - Errors**: The number of failed requests per second (HTTP 5xx, TCP resets).
- **D - Duration**: The amount of time requests take to complete (latency percentiles: p50, p95, p99).

| Methodology | Best Used For | Primary Metrics | Primary Tools |
| :--- | :--- | :--- | :--- |
| **USE** | Host OS, Kernel, Bare-metal hardware, Cloud instances | CPU wait%, RAM swap, Disk `await`, Net drops | `top`, `vmstat`, `iostat`, `sar`, `mpstat` |
| **RED** | Microservices, API gateways, Databases, Ingress | RPS, HTTP status codes, p99 latency | Prometheus, Grafana, Jaeger, Access logs |

---

## 3. The 60-Second Linux Triage Checklist

In the first minute of connecting to an impacted Linux server, execute this standardized 10-command sequence:

```bash
# 1. Check system uptime and 1, 5, 15 min load averages
uptime

# 2. Inspect kernel ring buffer for hardware/kernel errors (OOM kills, disk resets)
dmesg -T | tail -n 50

# 3. Overall system view: run queues, memory swapping, context switches
vmstat 1 5

# 4. Per-CPU core breakdown (detect single-core saturation or interrupt storm)
mpstat -P ALL 1 3

# 5. Process-level CPU breakdown
pidstat 1 3

# 6. Disk I/O throughput, IOPS, and wait latencies
iostat -xz 1 3

# 7. Physical memory and swap consumption
free -h

# 8. Network interface throughput and packet drop rates
sar -n DEV 1 3

# 9. TCP connection state and retransmissions
sar -n TCP,ETCP 1 3

# 10. Real-time interactive inspection
top -c
```

---

## 4. Hands-on Terminal Lab: Interpreting Triage Outputs

### Step 1: Interpreting `uptime` Load Average
```bash
uptime
```
```
 14:10:05 up 45 days,  2:12,  3 users,  load average: 18.42, 12.10, 4.50
```
- **Analysis**:
  - Determine total CPU cores: `nproc` (e.g. 4 cores).
  - Load average is 18.42 on a 4-core box! This indicates heavy saturation (14.42 processes are waiting in queue).
  - In Linux, load average includes both **runnable tasks (`R`)** AND **uninterruptible disk sleep tasks (`D`)**. Load alone does not distinguish between CPU saturation and disk I/O deadlock!

### Step 2: Dissecting `vmstat 1 5`
```bash
vmstat 1 5
```
```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
14  2      0 421050  82104 1450200   0    0  2400   450 1420 5400 85 12  0  3  0
```
- **`r` (14)**: 14 runnable processes competing for CPU (matches load average).
- **`b` (2)**: 2 processes blocked in uninterruptible sleep (`D`), waiting on disk/NFS.
- **`si`/`so` (0/0)**: No swap-in or swap-out. Memory is not thrashing.
- **`us` (85)**: 85% CPU consumed in userspace (application code).
- **`sy` (12)**: 12% CPU spent in kernel space (syscall overhead).
- **`id` (0)**: 0% idle; CPU is 100% saturated.
- **`wa` (3)**: Low disk wait. Conclusion: pure CPU exhaustion in userspace.

---

## 5. Top-Down vs Bottom-Up Investigation

```
           TOP-DOWN APPROACH                      BOTTOM-UP APPROACH
      (Alert at App Layer)                    (Hardware/Kernel Trigger)

   [ Application Code / Logs ]                [ Hardware / Kernel ]
              │                                        ▲
              ▼                                        │
     [ HTTP / RPC Latency ]                   [ Network Drops / Disk Errors ]
              │                                        ▲
              ▼                                        │
    [ Syscalls (strace) ]                     [ OS Metrics (USE Method) ]
              │                                        ▲
              ▼                                        │
  [ Kernel Subsystems / Hardware ]            [ Impacted Application ]
```

- **Top-Down**: Best when users report slow queries or timeouts. Start at application metrics (RED), profile syscalls (`strace`), then inspect OS resource limits.
- **Bottom-Up**: Best when alerting detects high node CPU, OOM kills, or disk saturation. Start at kernel metrics (USE), identify the offending PID via `top`/`pidstat`, then inspect application thread dumps.

---

## 6. The Scientific Method of Incident Resolution

1. **Observe**: Collect factual telemetry without preconceived bias (`uptime`, `dmesg`, `vmstat`).
2. **Hypothesize**: Formulate a falsifiable theory (e.g., "Thread contention on mutex in service X is driving 100% CPU in kernel mode").
3. **Predict**: If the hypothesis is true, what specific metric must follow? (e.g., "Context switch rate `cs` in `vmstat` must be > 100,000/s").
4. **Test & Verify**: Run `perf top` or inspect `/proc/<PID>/status`. If verified, apply mitigation. If disproved, reject and formulate a new hypothesis.
5. **Document**: Record exact timeline, root cause, and remediation steps in a Blameless Post-Mortem.

---

## 7. Key Takeaways
- Never guess or reboot blindly; follow a structured diagnostic workflow.
- The USE Method (Utilization, Saturation, Errors) guides hardware and OS layer triage.
- The RED Method (Rate, Errors, Duration) guides microservice endpoint triage.
- Linux load average includes both runnable (`R`) and uninterruptible sleep (`D`) tasks; use `vmstat` to identify whether CPU or I/O is the bottleneck.
- Master the 60-second triage checklist to quickly isolate root causes during active production incidents.
