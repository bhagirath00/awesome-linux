# Chapter 41: Memory Performance - Advanced Linux Memory Monitoring (/proc/meminfo)

## 1. Introduction: Beyond the `free` Command

While `free -m` provides a convenient high-level summary of total, used, and available RAM, it obscures critical low-level memory states that dictate system responsiveness:
- How much memory is tied up in **Dirty Pages** waiting to be written to disk?
- How much kernel memory is consumed by the **Slab Cache** (inodes and dentry objects)?
- How much memory is allocated as **Anonymous Memory** by application heaps?

In Linux, the single authoritative source of all memory performance metrics is the virtual file **`/proc/meminfo`**.

---

## 2. Key Metrics in /proc/meminfo

```bash
cat /proc/meminfo | head -n 25
```

```
MemTotal:        8133120 kB
MemFree:         1420512 kB
MemAvailable:    5124800 kB
Buffers:          184200 kB
Cached:          3412580 kB
SwapCached:        12480 kB
Active:          3981240 kB
Inactive:        1840120 kB
Active(anon):    2120480 kB
Inactive(anon):   412000 kB
Active(file):    1860760 kB
Inactive(file):  1428120 kB
Unevictable:           0 kB
Mlocked:               0 kB
SwapTotal:       2097148 kB
SwapFree:        2048000 kB
Dirty:              4512 kB
Writeback:             0 kB
AnonPages:       2532480 kB
Mapped:           412050 kB
Shmem:            210400 kB
KReclaimable:     312000 kB
Slab:             485200 kB
SReclaimable:     312000 kB
SUnreclaim:       173200 kB
```

---

## 3. Deep Dive: Memory Metrics Explained

### 1. Anonymous vs. Page Cache Memory
- **`AnonPages` (Anonymous Memory)**: Memory allocated dynamically by applications (`malloc()`) for stacks, heaps, and runtime data. This memory has **no backing file on disk**. The kernel can **only** reclaim this memory by writing it to Swap!
- **`Cached` (File-backed Memory)**: Disk blocks cached in RAM. If RAM runs low, the kernel can instantly discard clean cached pages without swapping because it can re-read them from disk at any time.

### 2. The Dirty Pages Subsystem (`Dirty` and `Writeback`)
- **`Dirty`**: Memory pages that have been modified by applications in RAM, but have **not yet been flushed to physical disk storage**.
- **`Writeback`**: Pages currently in the process of being written out to disk.

> [!WARNING]
> If `Dirty` memory climbs to multiple gigabytes and stays elevated, the system has a severe disk I/O bottleneck. When the kernel dirty memory thresholds are reached (`vm.dirty_ratio`), the kernel will **freeze all application processes attempting to write to disk** until the dirty buffer is flushed!

### 3. The Kernel Slab Cache (`Slab`, `SReclaimable`, `SUnreclaim`)
The kernel allocates its own internal data structures (directory entries `dentry` and file inodes) inside the **Slab Cache**.
- **`SReclaimable`**: Part of the slab cache that can be reclaimed when memory is needed (e.g., cached directory entries).
- **`SUnreclaim`**: Kernel structures that cannot be freed.

---

## 4. Querying Memory Per-Process with `smem`

Standard tools like `top` report **`RSS`** (Resident Set Size). However, `RSS` counts shared libraries (`libc.so`) multiple times across every process that loads them, producing grossly exaggerated memory totals.

The **`smem`** tool calculates **`PSS`** (**Proportional Set Size**), dividing shared libraries proportionally across sharing processes to report true memory usage:

```bash
# Install smem
sudo apt install -y smem    # Ubuntu
sudo dnf install -y smem    # RHEL (EPEL)

# Report memory sorted by PSS
sudo smem -r -k -s pss | head -n 10
```

```
  PID User     Command                         Swap      USS      PSS      RSS 
 1420 devops   /usr/bin/node server.js           0B   312.4M   324.8M   342.1M 
 1120 postgres /usr/lib/postgresql/bin/pos       0B   112.0M   128.5M   180.2M 
 2412 www-data nginx: worker process             0B    18.4M    24.1M    38.5M 
```
- **`USS` (Unique Set Size)**: Private memory held strictly by this process alone (freed if process dies).
- **`PSS` (Proportional Set Size)**: The true, realistic memory footprint of the application.

---

## 5. Summary Takeaways

- `/proc/meminfo` is the definitive source for all kernel memory metrics.
- Anonymous memory (`AnonPages`) can only be reclaimed via swap space.
- Monitor `Dirty` memory to catch storage write bottlenecks before processes stall.
- Use `smem` and **`PSS`** instead of `RSS` to accurately evaluate multi-process memory consumption.
