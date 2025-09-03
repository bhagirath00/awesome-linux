# Cgroups v1 vs Cgroups v2: Architecture & Migration

## 1. Architectural Concept & Deep Dive

The original implementation of Control Groups (**cgroups v1**), merged in Linux 2.6.24, introduced independent multi-hierarchies for every resource controller. Over a decade of production scale exposed deep structural flaws, race conditions, and an inability for controllers to coordinate with one another.

Kernel engineer Tejun Heo led a ground-up redesign resulting in **cgroups v2** (unified hierarchy), stabilized in Linux 4.5 (2016) and adopted as the default standard in modern Linux distributions (Fedora 31+, Ubuntu 21.10+, Debian 11+, RHEL 9+).

```
  CGROUPS v1 (Multiple Independent Trees)
  /sys/fs/cgroup/
  ├── cpu/           -> /sys/fs/cgroup/cpu/docker/container_A
  ├── memory/        -> /sys/fs/cgroup/memory/docker/container_A
  ├── blkio/         -> /sys/fs/cgroup/blkio/docker/container_A
  └── pids/          -> /sys/fs/cgroup/pids/docker/container_A
  (A process could be at path X in memory, but path Y in blkio!)

  VS

  CGROUPS v2 (Single Unified Hierarchy)
  /sys/fs/cgroup/
  └── system.slice/
      └── docker-container_A.scope/
          ├── cgroup.controllers (cpu memory io pids)
          ├── cpu.max
          ├── memory.max
          ├── io.max
          └── pids.max
  (All controllers bind to the EXACT same unified process tree!)
```

---

## 2. Why Cgroups v1 Failed at Scale

### 1. The Multi-Hierarchy Disaster
In v1, each controller (`cpu`, `memory`, `blkio`, `net_cls`) was mounted separately. A process could belong to `groupA` in memory, but `groupB` in cpu, and `groupC` in blkio. This caused intractable locking issues and rendered cross-resource algorithms mathematically impossible.

### 2. Page Cache Writeback Deadlock (The Buffered I/O Bug)
In v1, asynchronous file writes (dirty page cache) could not be throttled properly by `blkio`. When a process wrote to disk, pages were dirtied in memory. The kernel writeback thread (`kworker`), which belonged to the root cgroup, flushed those pages to disk. Consequently:
- `blkio` limits were completely bypassed because `kworker` did the write, not the container!
- In v2, memory and I/O controllers coordinate: dirty pages are tagged with their originating cgroup, allowing exact writeback throttling.

### 3. Lack of Safe Delegation
In v1, delegating sub-cgroups to unprivileged containers (e.g. rootless containers) introduced critical privilege escalation vulnerabilities. In v2, delegation is safe and native.

---

## 3. Comparison Matrix: v1 vs v2

| Feature / Metric | Cgroups v1 | Cgroups v2 |
| :--- | :--- | :--- |
| **Hierarchy** | Multiple distinct trees per controller | Single unified tree for all controllers |
| **Filesystem Mount** | Multiple mounts under `/sys/fs/cgroup/<controller>/` | Single mount: `cgroup2 on /sys/fs/cgroup` |
| **Process Placement** | Any node in tree can contain processes | **"No internal processes" rule**: only leaf nodes hold processes |
| **Buffered I/O Throttling**| Broken (flushed by kernel root thread) | Full writeback attribution and throttling |
| **Thread Management** | Arbitrary per-thread migration | Thread mode (`cgroup.type = threaded`) explicitly scoped |
| **Resource Pressure** | Crude thresholds / notifications | **PSI (Pressure Stall Information)** built-in |
| **eBPF Integration** | Limited / Hacky socket filters | Native eBPF hooks for network, device, and cgroup policy |
| **OOM Handling** | OOM killer kills single offending process | Optional `memory.oom.group`: kills entire container atomically |

---

## 4. The "No Internal Processes" Rule in v2

A core architectural invariant of cgroups v2:
> **Except for the root cgroup, a parent cgroup cannot distribute resources to child cgroups if it contains processes.**

```
       [Parent Cgroup]
       cgroup.subtree_control = "+cpu +memory"
             │
     ┌───────┴───────┐
     ▼               ▼
[Child Cgroup 1] [Child Cgroup 2]
(Has processes)   (Has processes)

* If Parent Cgroup also has a process, writing "+cpu" to
  subtree_control will return -EBUSY!
```
This guarantees that resource distribution among children is mathematically sound, avoiding the ambiguity of parent processes competing with entire child groups.

---

## 5. Pressure Stall Information (PSI)

Cgroups v2 introduced **PSI** (`/proc/pressure/` and `cgroup.pressure`), the gold standard for measuring resource starvation in production systems.

Instead of looking only at utilization metrics (which don't tell you if a system is actually hurting), PSI measures **lost time (stall time)**:

```bash
# Inspect CPU pressure on root or within a container
cat /sys/fs/cgroup/cpu.pressure
```
```
some avg10=0.04 avg60=0.02 avg300=0.00 total=1240822
```

- **`some`**: Percentage of wall-clock time where at least one task was stalled waiting for that resource (CPU, Memory, or I/O).
- **`full`**: Percentage of wall-clock time where *all* non-idle tasks were completely stalled (total system lockup).

Kubernetes node eviction agents and Meta's `oomd` rely on PSI metrics to proactively shed traffic or restart pods before a hard kernel lockup occurs.

---

## 6. How to Identify and Verify Cgroup Version

Run this command on your Linux host:
```bash
# Check filesystem type of /sys/fs/cgroup
stat -fc %T /sys/fs/cgroup/
```
- If output is **`cgroup2fs`**: Host is running pure **Cgroups v2**.
- If output is **`tmpfs`**: Host is running legacy **Cgroups v1** (hybrid or pure).

You can also check `/proc/filesystems`:
```bash
grep cgroup /proc/filesystems
```
```
nodev   cgroup
nodev   cgroup2
```

---

## 7. Migration Checklist for DevOps Engineers

When upgrading Linux host nodes (e.g. Ubuntu 20.04 -> 22.04+ or Debian 10 -> 11+):

1. **Kernel Boot Parameters**:
   - To force v2 on older kernels: `systemd.unified_cgroup_hierarchy=1`.
   - To revert to v1 if legacy software breaks: `systemd.unified_cgroup_hierarchy=0`.
2. **Container Engine Configuration**:
   - Ensure Docker 20.10+ or containerd 1.4+ is installed.
   - For Kubernetes, verify `/etc/containerd/config.toml` has:
     ```toml
     [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
       SystemdCgroup = true
     ```
3. **Java / JVM Compatibility**:
   - Ensure JVM applications use Java 8u372+, Java 11.0.16+, or Java 17+. Older JVMs misread `/sys/fs/cgroup/memory` on cgroups v2 hosts and calculate incorrect heap limits, crashing immediately.

---

## 8. Key Takeaways
- Cgroups v1 used messy multiple hierarchies with broken I/O accounting and race conditions.
- Cgroups v2 unifies all controllers into a single tree, strictly enforces the "no internal processes" rule, and fixes buffered I/O writeback.
- Pressure Stall Information (PSI) provides real-time measurement of CPU, memory, and I/O starvation.
- Modern container runtimes (Docker, containerd, Kubernetes) require Cgroups v2 for rootless operation and advanced resource QoS.
