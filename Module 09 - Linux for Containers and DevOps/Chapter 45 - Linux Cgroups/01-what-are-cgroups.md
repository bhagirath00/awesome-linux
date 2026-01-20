# Linux Control Groups (Cgroups) Fundamentals

## 1. Architectural Concept & Deep Dive

**Control Groups (cgroups)** are a fundamental Linux kernel feature that allows hierarchical organization of processes into named groups, with fine-grained metering, limiting, accounting, and isolation of hardware resources.

Originally designed by Paul Menage and Rohit Seth at Google in 2006 under the name "Process Containers," the feature was merged into Linux 2.6.24 (2008) and renamed **cgroups** to avoid confusion with operating system virtualization containers.

### Namespaces vs. Cgroups: The Core Difference
While Linux **Namespaces** determine *what a process can see* (visibility: file paths, PIDs, network cards, IPC channels), **Cgroups** determine *what a process can use* (consumption: CPU cores, RAM bytes, disk I/O bandwidth, thread count).

```
+─────────────────────────────────────────────────────────────+
|                         CONTAINER                           |
|                                                             |
|   NAMESPACES (Visibility)          CGROUPS (Resource Limits)|
|   ───────────────────────          ─────────────────────────|
|   • Mount: Isolated rootfs         • Memory: Max 512 MB     |
|   • PID: Isolated PID tree         • CPU: Max 1.5 Cores     |
|   • Net: Isolated eth0             • IO: 50 MB/s Read Limit |
|   • IPC: Isolated message queue    • PIDs: Max 256 threads  |
|                                                             |
+─────────────────────────────────────────────────────────────+
```

Without cgroups, a single compromised or runaway container could trigger an Out-Of-Memory (OOM) storm or starve the host CPU cores, crashing every neighboring workload on the bare-metal node.

---

## 2. Core Functional Pillars of Cgroups

The cgroup kernel subsystem enforces four distinct controls over process groups:

| Pillar | Description | Example Implementation |
| :--- | :--- | :--- |
| **1. Resource Limiting** | Enforces hard ceilings on memory, CPU usage, or file descriptor consumption. | Terminate process when memory exceeds `512MB` via OOM-killer. |
| **2. Prioritization** | Weighs relative resource allocation when host resources are contested. | Grant critical payment service 70% of CPU time and logging service 30%. |
| **3. Accounting** | Tracks exact resource consumption per group for billing, observability, and metrics. | Measure cumulative CPU runtime (`cpu.stat`) and I/O byte counts (`io.stat`). |
| **4. Control & Freezing** | Enables collective signals and state management across all processes in the group. | Pause/freeze all container processes instantly using `cgroup.freeze`. |

```
              Root Cgroup (/sys/fs/cgroup)
                      │
       ┌──────────────┴──────────────┐
       ▼                             ▼
system.slice                   kubepods.slice
(System Daemons)               (Workload Containers)
       │                             │
       ▼                      ┌──────┴──────┐
sshd.service                  ▼             ▼
(Limit: 100M RAM)          pod-1234      pod-5678
                         (Limit: 2G)   (Limit: 4G)
```

---

## 3. Major Resource Controllers

Cgroups utilize modular kernel drivers known as **controllers** (or subsystems):

### 1. `memory` Controller
Tracks physical memory, page cache, anonymous memory, kernel memory (e.g. socket buffers, page tables), and swap usage.
- Enforces soft limits (throttling/reclaiming pages) and hard limits.
- Triggers the Linux **OOM Killer** specifically within the offending cgroup when `memory.max` is breached, protecting the host system.

### 2. `cpu` Controller
Regulates CPU bandwidth using the Completely Fair Scheduler (CFS).
- **Proportional Weight**: Relative CPU shares (`cpu.weight`).
- **Bandwidth Quotas**: Absolute execution time limits within a period (e.g. 50ms quota every 100ms period = 0.5 CPU cores).

### 3. `io` Controller
Monitors and throttles Block I/O operations to storage devices.
- Sets IOPS caps (`io.max` read/write operations per second) and byte throughput caps (`rbps`/`wbps`).
- Prevents batch log processing or database backups from saturating NVMe/SSD buses.

### 4. `pids` Controller
Restricts the total number of processes and threads (`clone` or `fork` calls) that can exist within the cgroup.
- Protects the system against **fork bombs** (`:(){ :|:& };:`).

### 5. `cpuset` Controller
Pins a group of processes to dedicated physical CPU cores and NUMA memory nodes.
- Essential for low-latency financial trading platforms, Telco 5G DPDK stacks, and high-performance databases.

---

## 4. Hands-on Terminal Lab: Exploring the Cgroup Filesystem

Cgroups are presented to userspace through a pseudo-filesystem mounted at `/sys/fs/cgroup`.

### Step 1: Inspect the Cgroup Mount
```bash
mount | grep cgroup
```
```
cgroup2 on /sys/fs/cgroup type cgroup2 (rw,nosuid,nodev,noexec,relatime,nsdelegate)
```

### Step 2: View Controllers Enabled in the Kernel
```bash
cat /sys/fs/cgroup/cgroup.controllers
```
```
cpuset cpu io memory hugetlb pids rdma misc
```

### Step 3: Inspect Your Current Shell's Cgroup
Every process in Linux belongs to exactly one cgroup. Inspect your shell:
```bash
cat /proc/self/cgroup
```
```
0::/user.slice/user-1000.slice/session-3.scope
```

---

## 5. Cgroups in Containers and Kubernetes

In modern Kubernetes production environments:
- **Kubelet** interacts with cgroups via the **systemd cgroup driver** (`cgroupDriver: systemd`).
- Workload QoS classes map directly to cgroup hierarchies:
  - `Guaranteed`: `cpu.weight` prioritized, strict memory limits (`request == limit`).
  - `Burstable`: Baseline allocation with bursting allowance (`request < limit`).
  - `BestEffort`: No limits; first to be evicted when host faces memory pressure.

```
/sys/fs/cgroup/kubepods.slice/
├── kubepods-burstable.slice/
│   └── kubepods-burstable-pod<UID>.slice/
│       ├── memory.max (Limit)
│       ├── cpu.max (CFS Quota)
│       └── pids.max
└── kubepods-besteffort.slice/
```

---

## 6. Verification & Troubleshooting Runbook

### Health & Diagnostic Commands
```bash
# View active resource usage tree in real-time
systemd-cgtop

# Display full systemd cgroup tree
systemd-cgls

# Check OOM kills triggered inside cgroups
journalctl -k --grep="oom-kill|invoked oom-killer"
```

### Common Pitfalls & Solutions
1. **Container silently exits with exit code 137**:
   - Cause: Process exceeded cgroup memory limit (`128 + 9 = 137` = SIGKILL by OOM Killer).
   - Solution: Inspect `dmesg -T` for `oom-kill:constraint=CONSTRAINT_MEMCG`. Increase container memory limit or profile for memory leaks.
2. **Kubelet warns: "cgroup driver mismatch"**:
   - Cause: Docker or containerd was configured with `cgroupfs` driver while kubelet expects `systemd`.
   - Solution: In `/etc/containerd/config.toml`, set `SystemdCgroup = true` and restart containerd.
3. **Fork bomb crashes host node**:
   - Cause: `pids` controller is disabled or unset.
   - Solution: Ensure `pids.max` is configured on container slices (default in modern Docker is `--pids-limit 100`).

---

## 7. Key Takeaways
- Cgroups meter and restrict physical resource consumption (CPU, Memory, I/O, PIDs), whereas namespaces restrict visibility.
- All cgroup operations are managed through the virtual filesystem at `/sys/fs/cgroup`.
- Four primary controllers power modern containerization: `memory` (OOM isolation), `cpu` (CFS quotas & shares), `io` (bandwidth/IOPS), and `pids` (fork-bomb prevention).
- Kubernetes leverages systemd slices to enforce Pod QoS guarantees across production clusters.
