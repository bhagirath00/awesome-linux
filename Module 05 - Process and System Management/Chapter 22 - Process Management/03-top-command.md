# Chapter 22: Process Management - The top Command

## 1. Introduction: What is top?

The `top` (**Table of Processes**) program provides a real-time, dynamic view of a running Linux system. It displays system summary information (uptime, load averages, CPU utilization, physical and swap memory) followed by a continuously refreshed list of active processes sorted by CPU utilization by default.

Because `top` is part of the `procps` / `procps-ng` package, it is guaranteed to be installed on virtually **every Linux server, minimal container, and recovery environment**, making it an essential diagnostic tool for every systems engineer.

---

## 2. Anatomy of the top Interface

Running `top` launches an interactive dashboard divided into two distinct sections: the **System Summary Area** (the top 5 lines) and the **Task Table**.

```
top - 04:50:12 up 14 days,  3:22,  2 users,  load average: 0.15, 0.08, 0.02
Tasks: 184 total,   1 running, 183 sleeping,   0 stopped,   0 zombie
%Cpu(s):  2.3 us,  0.7 sy,  0.0 ni, 96.8 id,  0.1 wa,  0.0 hi,  0.1 si,  0.0 st
MiB Mem :   7942.5 total,   2140.2 free,   3412.1 used,   2390.2 buff/cache
MiB Swap:   2048.0 total,   2048.0 free,      0.0 used.   4210.8 avail Mem 

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
   1420 devops    20   0  984210 342110  42100 S   4.3   4.2   1:12.45 node
   1120 postgres  20   0  541200 128400  32100 S   1.0   1.6   0:45.10 postgres
    890 root      20   0   84200  12400   8200 S   0.3   0.2   0:05.12 systemd-journal
   2412 devops    20   0   42100   4800   3200 R   0.3   0.1   0:00.08 top
```

---

## 3. Deconstructing the Summary Header Lines

### Line 1: Uptime and Load Averages
```
top - 04:50:12 up 14 days, 3:22, 2 users, load average: 0.15, 0.08, 0.02
```
- Current system time (`04:50:12`).
- System uptime (`14 days, 3:22`).
- Number of logged-in sessions (`2 users`).
- **Load Average** over 1, 5, and 15 minutes. Represents the average number of processes in a **Runnable (`R`)** or **Uninterruptible Disk Sleep (`D`)** state. On a 4-core CPU, a load average of `4.00` indicates 100% capacity utilization.

### Line 2: Process States (Tasks)
```
Tasks: 184 total, 1 running, 183 sleeping, 0 stopped, 0 zombie
```
Keep a close eye on the **zombie** count. A non-zero, increasing zombie count indicates an application defect in process reaping.

### Line 3: CPU State Breakdown (`%Cpu(s)`)
| Metric | Full Name | Diagnostic Significance |
| :--- | :--- | :--- |
| `us` | User Space | Time spent executing un-niced user applications (Node, Nginx, Python). |
| `sy` | System (Kernel) | Time spent in kernel system calls (context switches, network I/O). |
| `ni` | Nice | Time spent executing user processes with positive nice values. |
| `id` | Idle | Percentage of time CPU cores are doing nothing. |
| `wa` | **I/O Wait** | **Critical metric**: Time CPU is stalled waiting for disk or network storage I/O. High `wa` indicates disk bottlenecks! |
| `hi` | Hardware IRQ | Time servicing hardware interrupts. |
| `si` | Software IRQ | Time servicing software interrupts (heavy network packet handling). |
| `st` | **Steal Time** | **Critical in Cloud/VMs**: Percentage of physical CPU cycles stolen by the hypervisor to serve other noisy neighbor VMs. |

### Lines 4 & 5: Physical RAM and Swap Breakdown
- `buff/cache`: RAM used by the Linux page cache to speed up disk reads. This memory is dynamically freed the instant an application demands it.
- `avail Mem`: The **true available memory** for new applications without swapping.

---

## 4. Interactive Keyboard Shortcuts Inside top

While `top` is running, press these single-key shortcuts:

| Key | Action |
| :--- | :--- |
| **`M`** | Sort process list by **Memory Usage** (`%MEM`) descending. |
| **`P`** | Sort process list by **CPU Usage** (`%CPU`) descending (default). |
| **`T`** | Sort by **Cumulative Time** (`TIME+`). |
| **`1`** | Toggle individual CPU core breakdown (reveals per-core thread distribution). |
| **`k`** | Interactive **Kill**: Prompts for PID and signal (default is 15 / `SIGTERM`). |
| **`r`** | Interactive **Renice**: Prompts for PID and new nice value. |
| **`u`** | Filter by a specific username. |
| **`c`** | Toggle displaying the full command path and CLI arguments. |
| **`d`** or **`s`**| Change update refresh interval (default is 3.0 seconds). |
| **`q`** | Quit `top`. |

---

## 5. Non-Interactive / Batch Mode for Scripts (`-b`)

You can run `top` in non-interactive batch mode to capture system snapshots or stream metrics into monitoring logs:

```bash
# Capture 2 iterations of top and exit
top -b -n 2 > /tmp/top-snapshot.txt

# Extract top 5 CPU consumers once
top -b -n 1 | head -n 12
```

---

## 6. Summary Takeaways

- `top` is universally pre-installed across all Linux systems and minimal container images.
- Monitor `wa` (I/O wait) for disk bottlenecks and `st` (steal time) for cloud hypervisor contention.
- Press `M` to sort by RAM, `P` for CPU, `1` for individual CPU cores, and `k` to kill a process directly.
- Use `top -b -n 1` in automation scripts to record point-in-time system metrics.
