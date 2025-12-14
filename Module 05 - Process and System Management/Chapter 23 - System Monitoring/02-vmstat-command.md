# Chapter 23: System Monitoring - The vmstat Command

## 1. Introduction: What is vmstat?

`vmstat` (**Virtual Memory Statistics**) is one of the most powerful, lightweight, and informative performance monitoring tools in Linux. It reports instantaneous summary metrics across **processes**, **virtual memory**, **paging/swapping**, **block I/O**, **interrupts**, and **CPU activity**.

Because `vmstat` outputs concise single-line tabular records at regular intervals with minimal CPU overhead, it is the first command senior systems architects run when diagnosing an unresponsive server.

---

## 2. Basic Command Syntax and Execution

**Syntax:**
```bash
vmstat [options] [delay_seconds] [count]
```

```bash
# Display metrics every 1 second, for 5 iterations:
vmstat 1 5
```

```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 2  0      0 214028 142100 2489200    0    0    12    45  840 1420  4  2 93  1  0
 1  0      0 213940 142100 2489240    0    0     0     0  912 1530  3  1 96  0  0
 4  1      0 212800 142100 2489260    0    0   450  1200 1450 2890 45 12 25 18  0
 2  0      0 212500 142100 2489260    0    0     0    80 1100 1900 12  4 84  0  0
 0  0      0 212480 142100 2489280    0    0     0     0  750 1250  1  1 98  0  0
```

> [!IMPORTANT]
> The **first line** of output from `vmstat` represents averages **since the last system reboot**. The **second line onward** represents real-time performance over the specified interval (e.g., the last 1 second). Always ignore the first line when troubleshooting live performance issues!

---

## 3. Detailed Column Explanations

### 1. Process Columns (`procs`)
| Column | Meaning | Diagnostic Significance |
| :--- | :--- | :--- |
| **`r`** | **Runqueue** | Number of runnable processes waiting for CPU time. If `r` consistently exceeds the number of physical CPU cores, your CPU is saturated. |
| **`b`** | **Blocked** | Number of processes placed in uninterruptible sleep (`D` state), typically waiting on disk or network storage I/O. |

### 2. Memory Columns (`memory` in KB)
- `swpd`: Amount of virtual memory used (pages moved to swap space).
- `free`: Completely idle, unallocated memory.
- `buff`: Memory used as temporary buffers for raw disk blocks.
- `cache`: Memory used by the Linux kernel page cache.

### 3. Swap Columns (`swap` in KB/s)
| Column | Meaning | Diagnostic Significance |
| :--- | :--- | :--- |
| **`si`** | **Swap In** | Amount of memory swapped **in** from disk per second. |
| **`so`** | **Swap Out** | Amount of memory swapped **out** to disk per second. |

> [!WARNING]
> Having non-zero `swpd` is not necessarily harmful if those memory pages belong to dormant processes. However, if **`si` and `so` are continuously non-zero and growing**, the system is experiencing **Swap Thrashing**. The kernel is desperately paging memory to disk, leading to massive performance degradation!

### 4. Block I/O Columns (`io` in blocks/s)
- `bi`: Blocks received from a block device (disk reads).
- `bo`: Blocks sent to a block device (disk writes).

### 5. System Columns (`system`)
- `in`: Number of system interrupts per second (including timer ticks and hardware devices).
- `cs`: Number of **context switches** per second. A spike in context switching indicates heavy thread contention or excessive locking.

### 6. CPU Columns (`cpu` percentage)
- `us`: Percentage of time spent in user space (applications).
- `sy`: Percentage of time spent in kernel system calls.
- `id`: Percentage of time CPU is idle.
- `wa`: Percentage of time spent stalled waiting for I/O (`iowait`).
- `st`: Time stolen by hypervisor from virtual machine.

---

## 4. Useful Flags

### Human-Readable Units (`-S M` or `-w`)
By default, memory columns are in kilobytes. Display them in Megabytes or wide layout:
```bash
# Display memory in Megabytes (-S M) with wide formatting (-w)
vmstat -S M -w 1 5
```

### Memory Slab Breakdown (`-m`)
Displays kernel dynamic data structure memory allocations:
```bash
sudo vmstat -m
```

### Disk Statistics Summary (`-d`)
Outputs detailed read/write operations per disk partition:
```bash
vmstat -d
```

---

## 5. Summary Takeaways

- `vmstat 1` is the fastest way to get a holistic view of system bottlenecks in real time.
- Always disregard line 1 (boot averages); inspect line 2 onwards.
- Watch `r` for CPU saturation, `b` for blocked disk I/O, and `wa` for storage stalls.
- Non-zero `si` and `so` indicate active swap thrashing.
