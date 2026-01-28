# Troubleshooting CPU Bottlenecks, Steal Time, and CFS Throttling

## 1. Architectural Concept & Deep Dive

CPU performance issues manifest in two distinct forms:
1. **Saturation / Starvation**: Tasks spend an excessive proportion of wall-clock time queued in the kernel run-queues waiting for a free CPU core.
2. **Pathological Overhead**: CPU cycles are consumed not by application logic, but by kernel syscall overhead, software interrupt storms, lock contention, or hypervisor CPU stealing.

Diagnosing CPU issues requires dissecting the eight distinct CPU utilization states reported by the Linux kernel scheduler:

```
+─────────────────────────────────────────────────────────────────────────+
|                           CPU TIME STATES (top)                         |
|                                                                         |
|  %us (User):        Application code execution                          |
|  %sy (System):      Kernel mode execution (syscalls, page faults)       |
|  %ni (Nice):        Low-priority user processes                         |
|  %id (Idle):        Waiting for work (healthy baseline)                 |
|  %wa (I/O Wait):    Idle while waiting on disk / NFS I/O                |
|  %hi (Hardware IRQ):Servicing physical device interrupts                |
|  %si (Software IRQ):Servicing network packet receive (softirqs)         |
|  %st (Steal Time):  Cycles stolen by underlying cloud hypervisor        |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Deciphering CPU Utilization Metrics

| CPU Metric | Danger Threshold | Root Cause Pattern | Primary Investigation Tool |
| :--- | :--- | :--- | :--- |
| **`%us`** | > 85% | Application computational bottleneck (infinite loop, heavy hashing, unindexed queries). | `perf top`, `pidstat -u 1` |
| **`%sy`** | > 20% | Excessive syscall frequency, high context switching (`cs`), memory allocation thrashing. | `perf record -g`, `strace -c` |
| **`%wa`** | > 10% | CPU is stalled waiting on slow disk or remote NFS storage. | `iostat -xz 1`, `iotop` |
| **`%si`** | > 15% | Network packet flood (DDoS, un-threaded network processing, NIC backlog). | `cat /proc/softirqs`, `ethtool -S` |
| **`%st`** | > 5% | Cloud hypervisor CPU contention (noisy neighbor, AWS T-series credit exhaustion). | Cloud console, `vmstat 1` |

---

## 3. Cloud Hypervisor Steal Time (`%st`) Deep Dive

In cloud environments (AWS EC2, GCP Compute Engine, Azure VMs), instances run on shared bare-metal hypervisors (KVM, Nitro). 

**Steal Time (`%st`)** occurs when the virtual CPU (vCPU) is ready to execute instructions, but the hypervisor refuses to allocate physical CPU cycles to the guest VM because:
1. **Burstable Credit Exhaustion**: On AWS `t3`/`t4g` instances, you have exhausted your CPU credits and the hypervisor has throttled your vCPU to baseline (e.g. 10-20% capacity).
2. **Host Oversubscription (Noisy Neighbor)**: The cloud provider has overcommitted physical cores and another virtual machine on the same hardware is consuming excessive cycles.

```bash
top -bn1 | grep "Cpu(s)"
```
```
%Cpu(s): 12.4 us,  3.2 sy,  0.0 ni, 42.1 id,  0.0 wa,  0.0 hi,  0.2 si, 42.1 st
```
- If `%st` is > 10-20%:
  - On AWS burstable instances: Check CloudWatch metric `CPUCreditBalance`. Upgrade to a non-burstable instance type (e.g., `c6i.xlarge`).
  - On standard instances: Stop and restart the instance to force AWS to schedule the VM on a different bare-metal host.

---

## 4. Kubernetes & Cgroups CFS Quota Throttling

A container in Kubernetes can experience severe CPU throttling **even when the node has 90% idle CPU capacity!**

### The Mechanism: Completely Fair Scheduler (CFS) Quotas
When you set `resources.limits.cpu: "1000m"`, the Linux kernel enforces this via `cpu.max` in cgroups v2:
- Period: 100,000 microseconds (100 ms).
- Quota: 100,000 microseconds (100 ms).
- If a multi-threaded application (e.g. Node.js or Java with 8 worker threads) bursts and consumes 100 ms of CPU time in the first 15 ms of the window, **the kernel pauses and freezes all threads of the container for the remaining 85 ms!**

### How to Detect CFS Throttling
```bash
# Check cgroups v2 CPU throttle stats for a container
cat /sys/fs/cgroup/kubepods.slice/.../cpu.stat
```
```
usage_usec 14502910
nr_periods 12000
nr_throttled 4800
throttled_usec 241029000
```
- **Analysis**: `nr_throttled / nr_periods = 40%`. 40% of all scheduling periods are being throttled, introducing massive p99 latency spikes into the microservice.
- **Fix**: Remove or increase CPU limits, or tune `cpu.cfs_quota_us` and period parameters.

---

## 5. Identifying Pathological Kernel Time (`%sy`)

When `%sy` exceeds 20-30%, the system is spending more time entering and exiting the kernel than doing real work.

### Step 1: Check Context Switch Rates with `vmstat`
```bash
vmstat 1 3
```
```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 2  0      0 450120  12040 241020    0    0     0     0 8500 241020 15 75 10  0  0
```
- Context switches (`cs`) at **241,020 per second**!
- Cause: Thousands of threads competing for synchronization mutexes or busy-waiting.

### Step 2: Sample Kernel Stack Traces with `perf top`
```bash
# Real-time inspection of active functions (kernel and userspace)
sudo perf top
```
```
Overhead  Shared Object       Symbol
  45.12%  [kernel]            [k] native_queued_spin_lock_slowpath
  18.40%  [kernel]            [k] futex_wait_queue_me
   8.10%  libc.so.6           [.] pthread_mutex_lock
```
- **Diagnosis**: 45% of total system CPU is wasted waiting on spinlocks (`native_queued_spin_lock_slowpath`). Application has thread contention on a shared memory lock.

---

## 6. Real-Time CPU Profiling CLI Runbook

### 1. View Per-Core CPU Distribution (`mpstat`)
```bash
# Verify if load is balanced across cores or if core 0 is 100% saturated
mpstat -P ALL 1 2
```
```
07:50:01 CPU  %usr  %nice  %sys  %iowait  %irq  %soft  %steal  %idle
07:50:02 all  25.0    0.0   2.0      0.0   0.0    0.5     0.0   72.5
07:50:02   0  98.0    0.0   2.0      0.0   0.0    0.0     0.0    0.0
07:50:02   1   2.0    0.0   1.0      0.0   0.0    0.0     0.0   97.0
```
- Core 0 is locked at 98% `%usr` while Core 1 is 97% idle. Single-threaded application bottleneck!

### 2. Identify the Culprit Process (`pidstat`)
```bash
# Display top CPU-consuming tasks with thread breakdown (-t)
pidstat -u -t 1 3 | sort -k8 -nr | head -n 10
```

---

## 7. Key Takeaways
- High `%us` indicates application optimization or scaling needs; high `%sy` indicates kernel lock contention or excessive syscalls.
- Steal time (`%st`) exposes cloud hypervisor oversubscription or burst credit depletion.
- Kubernetes CPU limits trigger kernel CFS throttling, introducing latent micro-stalls; verify via `cpu.stat`.
- Context switch rates exceeding 100,000/s indicate thread pool bloat or mutex thrashing.
