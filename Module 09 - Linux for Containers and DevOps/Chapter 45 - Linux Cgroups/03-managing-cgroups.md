# Managing Linux Cgroups v2: Hands-on Operations & Systemd

## 1. Architectural Concept & Deep Dive

In modern Linux, Control Groups (cgroups v2) can be managed via two primary interfaces:
1. **Direct Filesystem Manipulation**: Creating directories and writing parameters directly inside `/sys/fs/cgroup`.
2. **Systemd Unit & Slice Interface**: Creating transient scopes, services, and slices via `systemd-run` and `systemctl`.

In production environments, **Systemd is the designated cgroup manager**. Direct filesystem writes in `/sys/fs/cgroup` outside of designated delegation trees can conflict with systemd's state machine. However, understanding both layers is essential for debugging container runtimes (`containerd`, `runc`, `crun`, `docker`).

```
                +──────────────────────────────────────+
                |          Systemd Manager             |
                |   (systemd-run, slices, services)    |
                +──────────────────┬───────────────────+
                                   │
                                   ▼
                +──────────────────────────────────────+
                |    VFS Interface (/sys/fs/cgroup)    |
                |   - mkdir /sys/fs/cgroup/mygroup     |
                |   - echo $PID > cgroup.procs         |
                |   - echo "50000 100000" > cpu.max    |
                +──────────────────┬───────────────────+
                                   │
                                   ▼
                +──────────────────────────────────────+
                |         Linux Kernel Subsystems      |
                |     CFS Scheduler, Page Allocator    |
                +──────────────────────────────────────+
```

---

## 2. Low-Level Management: Manual Cgroups v2 via sysfs

Let's build and manage a cgroup from scratch without any external tools.

### Step 1: Create a Sub-Cgroup Directory
In cgroups v2, creating a directory automatically creates all interface files inside it!

```bash
# Must be root
sudo mkdir -p /sys/fs/cgroup/devops_sandbox
ls -la /sys/fs/cgroup/devops_sandbox
```
```
total 0
drwxr-xr-x  2 root root 0 Oct 14 06:20 .
dr-xr-xr-x 11 root root 0 Oct 14 06:15 ..
-r--r--r--  1 root root 0 Oct 14 06:20 cgroup.controllers
-rw-r--r--  1 root root 0 Oct 14 06:20 cgroup.events
-rw-r--r--  1 root root 0 Oct 14 06:20 cgroup.freeze
-rw-r--r--  1 root root 0 Oct 14 06:20 cgroup.kill
-rw-r--r--  1 root root 0 Oct 14 06:20 cgroup.max.depth
-rw-r--r--  1 root root 0 Oct 14 06:20 cgroup.max.descendants
-rw-r--r--  1 root root 0 Oct 14 06:20 cgroup.procs
-r--r--r--  1 root root 0 Oct 14 06:20 cgroup.stat
-rw-r--r--  1 root root 0 Oct 14 06:20 cgroup.subtree_control
-rw-r--r--  1 root root 0 Oct 14 06:20 cgroup.threads
-rw-r--r--  1 root root 0 Oct 14 06:20 cpu.max
-rw-r--r--  1 root root 0 Oct 14 06:20 cpu.weight
-rw-r--r--  1 root root 0 Oct 14 06:20 memory.high
-rw-r--r--  1 root root 0 Oct 14 06:20 memory.max
-rw-r--r--  1 root root 0 Oct 14 06:20 pids.max
```

### Step 2: Configure Hard Resource Ceilings

```bash
# 1. Limit Maximum Memory to 100 Megabytes (104,857,600 bytes)
echo "104857600" | sudo tee /sys/fs/cgroup/devops_sandbox/memory.max

# 2. Limit CPU to 0.5 Cores (50,000 microseconds quota per 100,000 period)
echo "50000 100000" | sudo tee /sys/fs/cgroup/devops_sandbox/cpu.max

# 3. Limit Process count to 10 PIDs (prevents fork-bombs)
echo "10" | sudo tee /sys/fs/cgroup/devops_sandbox/pids.max
```

### Step 3: Attach Processes to the Cgroup
To move a running process into the cgroup, write its PID to `cgroup.procs`:

```bash
# Open a background sleep task
sleep 300 &
TASK_PID=$!

# Move task into sandbox
echo "$TASK_PID" | sudo tee /sys/fs/cgroup/devops_sandbox/cgroup.procs

# Verify membership
cat /proc/$TASK_PID/cgroup
```
```
0::/devops_sandbox
```

### Step 4: Freezing and Killing Processes in a Cgroup
Cgroups v2 provides atomic group freezing and killing primitives:

```bash
# Freeze all processes in the sandbox (SIGSTOP equivalent, but kernel-enforced)
echo 1 | sudo tee /sys/fs/cgroup/devops_sandbox/cgroup.freeze

# Thaw/resume processes
echo 0 | sudo tee /sys/fs/cgroup/devops_sandbox/cgroup.freeze

# Atomically terminate all processes in the sandbox with SIGKILL
echo 1 | sudo tee /sys/fs/cgroup/devops_sandbox/cgroup.kill
```

---

## 3. Production Management: Systemd Interface

Systemd manages cgroups through **Units**:
- **Slices** (`.slice`): Hierarchical tree nodes partitioning resources (e.g. `system.slice`, `user.slice`).
- **Scopes** (`.scope`): Wrap externally created process groups (e.g. user sessions, container runtimes).
- **Services** (`.service`): Managed daemons.

### Running Ad-Hoc Commands with Resource Limits (`systemd-run`)

Launch commands with strict resource boundaries without modifying configuration files:

```bash
# Run a memory-capped stress-test session
sudo systemd-run --unit=mem-stress-test \
  --property=MemoryMax=128M \
  --property=CPUQuota=20% \
  --property=TasksMax=50 \
  python3 -c "import time; [bytearray(10**7) for _ in range(50)]; time.sleep(10)"
```

### Modifying Running Services Dynamically
You can modify resource limits of existing services in real-time without restarting them:

```bash
# Set runtime memory limit on nginx to 512MB
sudo systemctl set-property nginx.service MemoryMax=512M

# Set persistent limit (writes to /etc/systemd/system/nginx.service.d/50-MemoryMax.conf)
sudo systemctl set-property --runtime=false nginx.service MemoryMax=512M CPUQuota=150%
```

---

## 4. Parameter Mapping: Cgroups v2 vs Systemd vs Docker vs K8s

| Resource Control | Cgroups v2 File | Systemd Property | Docker CLI Flag | Kubernetes Pod Spec |
| :--- | :--- | :--- | :--- | :--- |
| **CPU Limit** | `cpu.max` | `CPUQuota=` | `--cpus="1.5"` | `resources.limits.cpu: "1500m"` |
| **CPU Weight** | `cpu.weight` | `CPUWeight=` | `--cpu-shares=1024` | `resources.requests.cpu` |
| **Memory Limit** | `memory.max` | `MemoryMax=` | `--memory="512m"` | `resources.limits.memory: "512Mi"` |
| **Memory Throttling** | `memory.high` | `MemoryHigh=` | N/A | Node Memory Pressure Eviction |
| **Max Processes** | `pids.max` | `TasksMax=` | `--pids-limit=100` | N/A (configured via kubelet `--pod-pids-limit`) |
| **Block I/O Read** | `io.max` (rbps) | `IOReadBandwidthMax=`| `--device-read-bps` | N/A |

---

## 5. Real-Time Monitoring & Inspection

### Real-Time Cgroup Top (`systemd-cgtop`)
```bash
# Monitor cgroups sorted by memory consumption
sudo systemd-cgtop -m
```
```
Path                                         Tasks   %CPU   Memory  Input/s Output/s
/                                              312    4.2     2.1G        -        -
/system.slice                                  142    1.8     1.2G        -        -
/system.slice/containerd.service                18    0.8   452.1M        -        -
/system.slice/kubelet.service                   24    0.6   210.4M        -        -
/user.slice                                     38    0.2   340.5M        -        -
```

### Inspecting Specific Cgroup Accounting
```bash
# View detailed memory breakdown of a cgroup
cat /sys/fs/cgroup/system.slice/containerd.service/memory.stat

# View CPU throttle stats (detect CFS starvation!)
cat /sys/fs/cgroup/system.slice/containerd.service/cpu.stat
```
Output:
```
usage_usec 49204921
user_usec 24102911
system_usec 25102010
nr_periods 1240
nr_throttled 12
throttled_usec 450120
```
- If `nr_throttled` is high, your container or service is constantly hitting its CPU quota and experiencing high response latencies.

---

## 6. Verification & Troubleshooting Runbook

### Common Issues & Remedies
1. **`write error: Device or resource busy (EBUSY)` when enabling subtree control**:
   - Cause: Attempting to enable controllers via `echo "+cpu" > cgroup.subtree_control` on a directory that already contains processes.
   - Solution: Move all processes to a child cgroup first (satisfy "no internal processes" rule).
2. **Application randomly hangs or becomes unresponsive**:
   - Cause: Process exceeded `memory.high` causing kernel direct reclaim throttling, or CPU throttling due to too low `CPUQuota`.
   - Solution: Check `cpu.stat` for `throttled_usec > 0`. Check `memory.events` for `high > 0`.
3. **Cannot delete cgroup directory (`rmdir: Device or resource busy`)**:
   - Cause: Processes or threads still reside inside `cgroup.procs`.
   - Solution: Kill all tasks via `echo 1 > cgroup.kill`, verify `cgroup.procs` is empty, then `rmdir`.

---

## 7. Key Takeaways
- Direct creation of cgroups is done by creating directories under `/sys/fs/cgroup`.
- Production servers should always use the systemd interface (`systemd-run`, `systemctl set-property`) to maintain state synchronization.
- `cpu.max` enforces hard CFS bandwidth limits, while `memory.max` triggers local OOM termination.
- Monitoring `cpu.stat` (for `nr_throttled`) and `systemd-cgtop` reveals container bottlenecks before service outages occur.
