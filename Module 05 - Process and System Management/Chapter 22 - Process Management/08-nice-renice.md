# Chapter 22: Process Management - Process Priority with nice and renice

## 1. Introduction: Process Scheduling and Priority

The Linux kernel uses the **Completely Fair Scheduler (CFS)** (and recently EEVDF in newer kernels) to allocate CPU execution time slices fairly among all competing runnable tasks.

However, not all tasks are created equal:
- An interactive database query or SSH shell requires immediate CPU responsiveness to maintain low latency.
- A nightly batch report, backup archive, or video transcoding job can take longer without impacting user experience, provided it doesn't starve critical workloads.

To control this behavior, Linux provides the **Niceness** scale.

---

## 2. The Niceness Scale (-20 to +19)

Niceness represents how "nice" or polite a process is to other processes on the system:

```
HIGHEST PRIORITY                                                 LOWEST PRIORITY
(Consumes maximum CPU)                                    (Yields CPU generously)
       |                                                             |
      -20 <-------------------- 0 --------------------> +19
                                ^
                         Default Niceness
```

| Nice Value | Priority Level | Behavioral Meaning | Permissions Required |
| :--- | :--- | :--- | :--- |
| **`-20`** | **Highest Priority** | Extremely aggressive; demands maximum CPU time; rarely yields. | **Root (`sudo`) only** |
| **`-1` to `-19`** | High Priority | Prioritized over normal background tasks. | **Root (`sudo`) only** |
| **`0`** | **Standard (Default)** | Default baseline assigned to newly spawned processes. | Any standard user |
| **`1` to `+19`** | Low Priority | Polite; graciously yields CPU cycles to other processes. | Any standard user |
| **`+19`** | **Lowest Priority** | Background idle processing; runs only when CPU has idle cycles. | Any standard user |

> [!NOTE]
> In `top` and `htop`:
> - `PR` or `PRI` represents the **Kernel Dynamic Priority** (calculated roughly as `20 + NI`).
> - `NI` represents the user-adjustable **Nice Value** (`-20` to `+19`).

---

## 3. Starting a Process with `nice`

Use the `nice` command to launch a brand-new process with a custom nice value:

**Syntax:**
```bash
nice -n <nice_value> <command> [args...]
```

### Examples:
```bash
# Run a CPU-heavy backup with lowest possible priority (+19)
nice -n 19 tar -czf /backup/archive.tar.gz /data

# Launch a real-time audio/database process with high priority (-10)
sudo nice -n -10 /opt/database/bin/engine
```

If you execute `nice` without any flags, it prints the current shell's default niceness (typically `0`):
```bash
nice
```
```
0
```

---

## 4. Modifying Running Processes with `renice`

To change the priority of an **already running** process, use `renice`:

**Syntax:**
```bash
renice -n <new_nice_value> -p <PID>
```

### Examples:
```bash
# Identify runaway PID
pgrep -f "ffmpeg"
# Output: 14205

# De-prioritize the runaway encoding job to +15 so web traffic is not impacted
renice -n 15 -p 14205
```

```
14205 (process ID) old priority 0, new priority 15
```

### Batch Renicing:
```bash
# Renice ALL processes owned by user "developer" to +10
sudo renice -n 10 -u developer

# Renice all processes belonging to a process group (PGRP)
sudo renice -n -5 -g 3200
```

---

## 5. Security and Permission Constraints

To prevent regular users from monopolizing system CPU cores and starving critical system daemons:
- **Standard (unprivileged) users** can **only increase** the niceness of their own processes (i.e., make them nicer / lower priority).
- **Only the `root` superuser** can lower niceness into negative values (higher priority) or lower the niceness of existing tasks.

---

## 6. Summary Takeaways

- Nice values range from **`-20` (highest priority)** to **`+19` (lowest priority)**; default is `0`.
- Remember: **"More nice = Less CPU priority"**.
- Use `nice` when launching new commands; use `renice` to adjust already active PIDs.
- Unprivileged users can only deprioritize their own processes; only root can assign negative nice values.
