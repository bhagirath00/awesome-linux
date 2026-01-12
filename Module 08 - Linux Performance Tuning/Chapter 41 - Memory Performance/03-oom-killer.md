# Chapter 41: Memory Performance - The Out-Of-Memory (OOM) Killer

## 1. Introduction: Why Does the OOM Killer Exist?

In Linux, the kernel employs a memory management strategy known as **Memory Overcommit**. 

When an application asks for memory using `malloc(1GB)`, the kernel does not immediately allocate 1GB of physical RAM. Instead, it grants a virtual memory allocation promise. The kernel assumes that programs will not immediately use 100% of the memory they request. This allows Linux to run many concurrent processes with high memory density.

However, if multiple applications simultaneously begin writing data to their promised memory spaces, physical RAM and swap space become **100% exhausted**. 

At this point, the Linux kernel faces an existential choice:
- **Crash the entire operating system** via a kernel panic.
- **Select one or more processes and terminate them immediately** using `SIGKILL` (Signal 9) to free physical RAM and save the OS.

The kernel subsystem responsible for making this sacrifice is the **Out-Of-Memory (OOM) Killer**.

```
+--------------------------------------------------------------------------+
|                        OOM Killer Decision Workflow                      |
+--------------------------------------------------------------------------+
|  Total Physical RAM & Swap Reaches 100% Exhaustion                       |
|                                |                                         |
|                                v                                         |
|  Linux Kernel Triggers: out_of_memory() Subsystem                        |
|                                |                                         |
|  Scans all running processes in the system:                              |
|  - Calculates "Badness Score" based on:                                  |
|    1. Percentage of system RAM consumed (Huge consumers scored higher)   |
|    2. Process UID (Non-root penalized higher)                            |
|    3. Runtime adjustment: /proc/<PID>/oom_score_adj                      |
|                                |                                         |
|                                v Identifies Highest badness score:       |
|  Process: java (PID: 14205, Badness: 842)                                |
|                                |                                         |
|                                v Kernel executes:                        |
|  kill -9 14205 (SIGKILL sent! Memory freed instantly! OS saved!)         |
+--------------------------------------------------------------------------+
```

---

## 2. Investigating OOM Kills in Production

When an application suddenly disappears without an error traceback or an exit code, it was almost certainly terminated by the OOM Killer (exit code **137** = `128 + 9`).

### 1. Search Kernel Logs for OOM Invocations
```bash
# Query kernel ring buffer via dmesg
sudo dmesg -T | grep -i -E "oom|out of memory|killed process"

# Or via journalctl
sudo journalctl -k --grep="Out of memory"
```

```
[Wed Oct 14 06:12:45 2026] Out of memory: Killed process 14205 (java) total-vm:8412000kB, anon-rss:7214000kB, file-rss:1200kB, shmem-rss:0kB, UID:1000 pgtables:14200kB oom_score_adj:0
```
This log confirms:
- **Process**: `java` (PID 14205)
- **Memory Consumed**: `7.2 GB` of anonymous RSS memory
- **Victim Reason**: Highest badness score

---

## 3. Controlling OOM Priority: `oom_score_adj`

You can influence the kernel's decision by adjusting the **`oom_score_adj`** value for specific critical processes:

- Value Range: **`-1000` to `+1000`**.
- **`-1000`**: **IMMUNITY**. The process will **NEVER be killed by the OOM killer** under any circumstance.
- **`+1000`**: First in line to be sacrificed the moment memory runs low.

### Protecting a Critical Service (e.g., SSHD or Core Database):
```bash
# Give PID 890 (sshd) complete immunity from OOM termination:
echo -1000 | sudo tee /proc/890/oom_score_adj
```

### In systemd Services:
Configure immunity directly in your service unit file:
```ini
[Service]
# Protect database from OOM killer:
OOMScoreAdjust=-1000
```

---

## 4. Kernel Memory Overcommit Settings (`vm.overcommit_memory`)

You can control how aggressively the kernel promises memory to applications via `sysctl`:

| `vm.overcommit_memory` | Mode | Behavior |
| :--- | :--- | :--- |
| **`0`** | **Heuristic (Default)** | Kernel uses internal heuristic estimates to allow reasonable overcommitment. |
| **`1`** | **Always Overcommit** | Kernel always grants memory requests regardless of physical RAM. **Mandatory for Redis** background saving! |
| **`2`** | **Strict / No Overcommit** | Kernel refuses to allocate memory beyond `Swap + (RAM * overcommit_ratio)`. Prevents OOM kills, but programs receive `NULL` on `malloc()`. |

### Configuring Overcommit for Redis:
Redis requires `vm.overcommit_memory=1` to allow memory forking during background snapshots without failing:
```bash
sudo sysctl vm.overcommit_memory=1
echo "vm.overcommit_memory=1" | sudo tee /etc/sysctl.d/99-redis-overcommit.conf
```

---

## 5. Summary Takeaways

- The OOM Killer terminates memory-hogging processes to save the kernel from crashing.
- Sudden process deaths with exit code `137` are caused by OOM termination.
- Audit OOM events using `dmesg -T | grep -i oom`.
- Protect critical daemons by setting `OOMScoreAdjust=-1000` in systemd.
- Set `vm.overcommit_memory=1` on Redis servers to prevent background snapshot crashes.
