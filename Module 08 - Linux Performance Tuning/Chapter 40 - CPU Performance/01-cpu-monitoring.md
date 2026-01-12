# Chapter 40: CPU Performance - CPU Utilization and Bottleneck Monitoring

## 1. Introduction: Deconstructing CPU Utilization

In modern multi-core Linux systems, high CPU utilization does not necessarily indicate that your CPU hardware is the bottleneck. 

A multi-threaded web server might report 90% CPU usage while processing requests efficiently with low latency. Conversely, a single-threaded Node.js or Python application might saturate a single CPU core to 100%, causing massive request queueing while an overall 16-core system reports an average CPU utilization of only **6.25%**!

Effective CPU monitoring requires analyzing **per-core distributions**, **scheduler runqueues**, and **CPU state breakdowns**.

```
+--------------------------------------------------------------------------+
|                     Single Core Saturation Bottleneck                    |
+--------------------------------------------------------------------------+
|  CPU 0: [||||||||||||||||||||||||||||||||||||||||||||||||||||] 100% SATURATED!
|  CPU 1: [|                                                   ]   2% Idle  |
|  CPU 2: [|                                                   ]   1% Idle  |
|  CPU 3: [|                                                   ]   2% Idle  |
|                                                                          |
|  Global System Average CPU Utilization: ~26% (Appears "healthy" on top)   |
|  Reality: Single-threaded Node/Python event loop is bottlenecked on CPU 0!|
+--------------------------------------------------------------------------+
```

---

## 2. Per-Core CPU Monitoring with `mpstat`

The `mpstat` utility (part of `sysstat`) is the definitive tool for observing how workload is distributed across physical and virtual CPU cores:

```bash
# Report metrics for ALL individual cores every 1 second, 3 times
mpstat -P ALL 1 3
```

```
11:15:01 AM  CPU    %usr   %nice    %sys %iowait   %irq   %soft  %steal  %guest  %gnice   %idle
11:15:02 AM  all    8.24    0.00    2.15    0.10   0.00    0.20    0.00    0.00    0.00   89.31
11:15:02 AM    0   98.50    0.00    1.50    0.00   0.00    0.00    0.00    0.00    0.00    0.00  <-- HOT CORE!
11:15:02 AM    1    1.20    0.00    0.80    0.00   0.00    0.10    0.00    0.00    0.00   97.90
11:15:02 AM    2    0.90    0.00    0.50    0.10   0.00    0.00    0.00    0.00    0.00   98.50
11:15:02 AM    3    2.10    0.00    1.10    0.00   0.00    0.20    0.00    0.00    0.00   96.60
```
Looking at core `0` reveals that a single process is choking the core at 98.5% user time (`%usr`).

---

## 3. Per-Process CPU Breakdown with `pidstat`

Once an imbalance is detected, identify exactly **which Process ID and thread** is consuming the cycles:

```bash
# Display CPU usage per process every 1 second:
pidstat 1 3
```

```
11:15:10 AM   UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
11:15:11 AM  1000     14205   98.00    1.50    0.00    0.50   99.50     0  node
11:15:11 AM   105       890    1.20    0.80    0.00    0.00    2.00     3  postgres
```
PID `14205` (`node`) is pinned to CPU core `0` consuming 99.5% CPU!

### Tracking Threads inside a Process (`-t`):
For multi-threaded languages (Java, Go, C++):
```bash
pidstat -t -p 14205 1
```
This isolates the specific thread ID (TID) causing CPU spikes inside the JVM or Go runtime.

---

## 4. Context Switching: Voluntary vs. Involuntary (`pidstat -w`)

High CPU usage combined with sluggish performance often stems from excessive **Context Switching** (the CPU stopping one thread to save registers and load another thread):

```bash
pidstat -w 1 3
```

| Context Switch Type | Column | Meaning & Problem Indicator |
| :--- | :--- | :--- |
| **Voluntary (`cswch/s`)** | Process yields CPU voluntarily | Waiting for I/O, sleep timers, or database responses. Normal for healthy applications. |
| **Involuntary (`nvcswch/s`)**| Process is forcibly preempted by kernel scheduler | **Problem Indicator**: Too many threads competing for CPU time slices or lock contention. |

If `nvcswch/s` is high across worker processes, you have spawned more threads than the physical CPU cores can handle, causing the kernel to waste massive CPU cycles merely switching thread contexts!

---

## 5. Summary Takeaways

- Average system CPU metrics can mask single-threaded core bottlenecks.
- Always inspect per-core CPU breakdown using `mpstat -P ALL 1`.
- Use `pidstat 1` to link CPU consumption directly to specific PIDs.
- Use `pidstat -t` to drill down into multi-threaded Java or Go threads.
- High involuntary context switches (`nvcswch/s`) indicate excessive thread contention or thread pool oversizing.
