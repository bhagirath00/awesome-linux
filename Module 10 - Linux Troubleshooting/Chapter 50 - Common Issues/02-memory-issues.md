# Troubleshooting Memory Issues, Swap Thrashing, and the OOM Killer

## 1. Architectural Concept & Deep Dive

Linux manages physical memory through a complex Virtual Memory System (VMS) governed by demand paging, page caches, and swap spaces.

A common misunderstanding among junior engineers is the assumption that high "used" memory is a problem. In Linux:
> **Free RAM is wasted RAM.**

The Linux kernel aggressively utilizes all idle physical RAM to cache disk reads and writes (**Page Cache** and **Dentries/Inodes**). If an application subsequently requests memory, the kernel evicts clean cache pages instantaneously. Therefore, true memory exhaustion is determined not by `free`, but by **`available`** memory and **swap thrashing**.

```
+─────────────────────────────────────────────────────────────────────────+
|                       LINUX PHYSICAL MEMORY POOL                        |
|                                                                         |
|  +──────────────────+ +──────────────────────────────────────────────+  |
|  | Anonymous Memory | | Page Cache & Buffers (Reclaimable)           |  |
|  | (Process heaps,  | | • Disk file reads cached in RAM              |  |
|  |  stack, mmap)    | | • Dirty write buffers waiting on flush       |  |
|  +─────────┬────────+ +──────────────────────┬───────────────────────+  |
|            │                                 │                          |
|            ▼                                 ▼                          |
|    Under Extreme Pressure:            Instant Reclaim                   |
|    Must Swap to Disk                  (Evicted to make room for app)    |
|    (Expensive Disk I/O)                                                 |
|            │                                                            |
|            ▼                                                            |
|    If Swap Full / Exhausted:                                            |
|    Kernel OOM Killer invoked (SIGKILL)                                  |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Dissecting `free -h` & `/proc/meminfo`

### Interpreting `free -h`
```bash
free -h
```
```
               total        used        free      shared  buff/cache   available
Mem:            31Gi        18Gi       2.1Gi       1.2Gi        11Gi        12Gi
Swap:          8.0Gi       1.4Gi       6.6Gi
```

| Field | Meaning | Production Diagnosis |
| :--- | :--- | :--- |
| **`total`** | Total usable physical RAM (excluding kernel reserved bytes). | Hardware capacity. |
| **`used`** | Memory allocated to processes (`total - free - buff/cache`). | Active application footprint. |
| **`free`** | Memory completely untouched and zeroed out. | Expect this to be low on any healthy, long-running server. |
| **`shared`** | Memory used by tmpfs mounts and POSIX shared memory (`shm`). | High values indicate large Docker shm or inter-process tables. |
| **`buff/cache`** | Disk buffers + page cache + filesystem slab. | Reclaimable space. |
| **`available`** | **The most critical metric.** Estimated amount of memory available for starting new applications without swapping. | If `available` approaches 0, an OOM kill is imminent. |

---

## 3. Detecting Swap Thrashing (`vmstat 1`)

Swapping memory to disk (NVMe/SSD/HDD) is several orders of magnitude slower than accessing physical RAM. When available memory drops and active processes are continuously paged out to disk and immediately paged back in, the system enters **Swap Thrashing**.

```bash
vmstat 1 5
```
```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 6  8 1845120  42100   8200  95100  4200 8910  5100 9200 4500 8100 12 45  0 43  0
```

- **`si` (Swap-In)**: Kilobytes per second read from swap disk into RAM.
- **`so` (Swap-Out)**: Kilobytes per second written from RAM to swap disk.
- **`wa` (I/O Wait)**: CPU percentage waiting on disk transactions.
- **Diagnosis**: High `si`/`so` concurrent with high `wa` and near-zero `id` indicates catastrophic swap thrashing. The host is functionally frozen.

---

## 4. The Linux OOM Killer Mechanics

When the kernel experiences an allocation failure (`alloc_pages` fails) and cannot free any memory via page cache reclamation or swapping, the **Out of Memory (OOM) Killer** is invoked to prevent total kernel panic.

### 1. How the Kernel Selects a Victim (`oom_score`)
The kernel inspects every task and calculates a score between 0 and 1000 stored in `/proc/<PID>/oom_score`.
The calculation:
- Factors in the percentage of physical RAM consumed by the process.
- Multiplies based on process age (prefers killing newer short-lived workers over systemd or init).
- Adds the user-configured **`oom_score_adj`** (-1000 to +1000).

```bash
# View the calculated OOM score of a process
cat /proc/<PID>/oom_score

# Check its adjustment factor (-1000 disables OOM killer for this process!)
cat /proc/<PID>/oom_score_adj
```

### 2. Identifying OOM Kills in Production
```bash
# Search kernel ring buffer for OOM events
sudo dmesg -T | grep -E -i "oom[-_]killer|out of memory" -A 10
```
```
[Wed Oct 14 07:45:12 2026] Out of memory: Killed process 24151 (mysqld) total-vm:22104520kB, anon-rss:18451204kB, file-rss:0kB, shmem-rss:0kB, UID:105 pgtables:38104kB oom_score_adj:0
[Wed Oct 14 07:45:12 2026] oom_reaper: reaped process 24151 (mysqld), now anon-rss:0kB, file-rss:0kB, shmem-rss:0kB
```

---

## 5. Kernel Slab Memory Leaks (`slabtop`)

Not all memory leaks occur in user applications. Kernel drivers, network sockets, or filesystem dentries can leak kernel memory inside the **Slab Allocator**.

```bash
# Check slab usage in /proc/meminfo
grep -E "(Slab|SReclaimable|SUnreclaim)" /proc/meminfo
```
```
Slab:            4512040 kB
SReclaimable:    4120100 kB
SUnreclaim:       391940 kB
```

If `SUnreclaim` (unreclaimable slab) consumes gigabytes of memory:
```bash
# Interactive real-time top for kernel slab caches
sudo slabtop -s c
```
Look for runaway object counts in caches such as `dentry`, `inode_cache`, or `skbuff_head_cache`.

---

## 6. Hands-on Tuning & Mitigation Runbook

### 1. Tuning Swappiness (`vm.swappiness`)
`vm.swappiness` (range 0 to 200, default 60) defines the kernel's relative preference between reclaiming page cache vs swapping anonymous memory:
- `vm.swappiness = 10`: Recommended for databases and Kubernetes worker nodes (avoids premature swapping).
- `vm.swappiness = 0`: Kernel will avoid swapping until memory pressure is absolute.

```bash
# Set swappiness dynamically
sudo sysctl -w vm.swappiness=10

# Persist in /etc/sysctl.d/99-memory.conf
echo "vm.swappiness=10" | sudo tee /etc/sysctl.d/99-memory.conf
```

### 2. Protecting Critical Daemons from the OOM Killer
To ensure critical daemons (e.g. `sshd`, monitoring agents) are never terminated by the OOM killer:

```bash
# In systemd unit file (/etc/systemd/system/sshd.service.d/override.conf)
[Service]
OOMScoreAdjust=-1000
```

### 3. Emergency Drop Caches (Temporary Relief)
If memory is tied up in clean page cache and unneeded dentries:
```bash
# Sync dirty buffers to disk first!
sync

# Drop page cache (1), dentries/inodes (2), or both (3)
echo 3 | sudo tee /proc/sys/vm/drop_caches
```

---

## 7. Key Takeaways
- High "used" RAM in Linux is normal due to aggressive kernel page caching; prioritize `available` memory and `si`/`so` metrics.
- Swap thrashing (`vmstat 1` showing sustained `si`/`so` with high `wa`) freezes the host; tune `vm.swappiness=10`.
- The OOM killer terminates processes based on `/proc/<PID>/oom_score` to protect the host kernel.
- Protect critical management daemons by setting `OOMScoreAdjust=-1000` in systemd.
