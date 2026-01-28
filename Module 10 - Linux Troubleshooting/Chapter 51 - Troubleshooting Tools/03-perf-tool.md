# Linux Profiling & Analysis with the perf Tool

## 1. Architectural Concept & Deep Dive

**`perf`** (sometimes called Performance Counters for Linux or PCL) is the official performance profiling subsystem built directly into the Linux kernel (under `tools/perf/`).

Unlike `strace` and `ltrace`, which rely on invasive `ptrace` breakpoints that slow down applications, `perf` is designed for **low-overhead, non-disruptive production profiling**. It operates via **statistical sampling**:
- Every $N$ CPU clock cycles or hardware events, the Linux kernel triggers a Performance Monitoring Unit (PMU) hardware interrupt.
- The kernel inspects the current Instruction Pointer (IP) and call stack of the active CPU core, records the sample in an in-memory ring buffer, and resumes execution in nanoseconds.
- By collecting thousands of samples per second, `perf` reconstructs a statistically accurate profile of where CPU time is spent with **less than 1% performance overhead**.

```
+─────────────────────────────────────────────────────────────+
|                     PERF SAMPLING ARCHITECTURE              |
|                                                             |
|   Hardware CPU Cores (PMU: Performance Monitoring Unit)     |
|          │                                                  |
|          ├── Generates Hardware Counter Interrupts          |
|          │   (e.g., Every 100,000 CPU cycles or cache miss) |
|          ▼                                                  |
|   Linux Kernel perf Subsystem                               |
|          │                                                  |
|          ├── Captures stack trace: [User IP] -> [Kernel IP] |
|          ├── Writes sample to lockless circular ring buffer |
|          ▼                                                  |
|   Userspace perf CLI (perf record / perf top)               |
|          │                                                  |
|          ▼                                                  |
|   Interactive Hotspot Report & FlameGraph Generation        |
+─────────────────────────────────────────────────────────────+
```

---

## 2. Event Types Measured by `perf`

`perf` can measure three distinct categories of performance events:

| Event Type | Mechanism | Examples | Production Diagnosis |
| :--- | :--- | :--- | :--- |
| **Hardware Events** | CPU Silicon PMU registers | `cpu-cycles`, `instructions`, `cache-misses`, `branch-misses` | CPU instruction efficiency (IPC), L1/L2/L3 cache misses, memory latency. |
| **Software Events** | Kernel software tracepoints | `page-faults`, `context-switches`, `cpu-migrations` | Virtual memory page allocation overhead, lock contention. |
| **Tracepoint Events**| Static kernel instrumentation hooks | `sched:sched_switch`, `syscalls:sys_enter_read` | Scheduler behavior, exact syscall latencies. |

```bash
# List all performance events supported by your hardware and kernel
perf list
```

---

## 3. Real-Time Hotspot Analysis (`perf top`)

`perf top` is the system profiler's equivalent of `top`. Instead of showing processes, it displays **the exact functions and assembly symbols consuming CPU cycles right now across the entire operating system**:

```bash
sudo perf top
```
```
Samples: 42K of event 'cycles', 4000 Hz, Event count (approx.): 24105021000
Overhead  Shared Object           Symbol
  32.10%  my_crypto_service       [.] sha256_transform_avx2
  18.40%  [kernel]                [k] native_queued_spin_lock_slowpath
  12.05%  libc.so.6               [.] __memmove_avx_unaligned_erms
   8.50%  [kernel]                [k] clear_page_erms
```

- **`[.]`**: Symbol in **userspace** library or application binary.
- **`[k]`**: Symbol inside the **Linux kernel**.
- **Analysis**:
  - `sha256_transform_avx2` at 32%: Application is CPU-bound in cryptographic hashing.
  - `native_queued_spin_lock_slowpath` at 18%: Severe kernel lock contention; threads are fighting over a shared resource.

---

## 4. Recording & Analyzing Profiles (`perf record` and `perf report`)

To record a profile during a latency spike or benchmark run for offline analysis:

### Step 1: Record CPU Profile with Call Graphs (`-g`)
```bash
# Record all CPUs (-a) with call stacks (-g) at 99Hz for 10 seconds (-F 99)
sudo perf record -F 99 -a -g -- sleep 10
```
- **Why 99 Hz?** Sampling at 100 Hz or 1000 Hz can accidentally synchronize with a periodic timer interrupt. Sampling at 99 Hz avoids harmonic resonance lock.

### Step 2: Interactive Drill-Down (`perf report`)
```bash
sudo perf report --stdio
```
You can expand individual call stacks to navigate down from `main()` -> `http_handle()` -> `parse_json()` -> `malloc()` to discover the exact line of code causing bottlenecks.

---

## 5. Visualizing Bottlenecks: FlameGraph Generation

Pioneered by Brendan Gregg, **FlameGraphs** convert massive, unreadable stack trace text dumps into intuitive, interactive visual graphs:
- **X-axis**: Alphabetical sorting of stack traces (width = percentage of CPU time consumed).
- **Y-axis**: Stack depth from bottom (`main`) to top (leaf function executing on CPU).

```
                      +──────────────────────────+
                      |   sha256_transform       |  <-- Top of stack (Bottleneck!)
               +──────┴──────────────────────────┴──────+
               |             hash_payload               |
        +──────┴────────────────────────────────────────┴──────+
        |                 process_request                      |
+───────┴──────────────────────────────────────────────────────┴───────+
|                                 main                                 |
+──────────────────────────────────────────────────────────────────────+
```

### Steps to Generate an SVG FlameGraph:
```bash
# 1. Clone FlameGraph utility scripts
git clone https://github.com/brendangregg/FlameGraph /tmp/FlameGraph

# 2. Record samples with stack traces
sudo perf record -F 99 -p <PID> -g -- sleep 30

# 3. Collapse stacks and render vector SVG
sudo perf script | /tmp/FlameGraph/stackcollapse-perf.pl > /tmp/out.folded
/tmp/FlameGraph/flamegraph.pl /tmp/out.folded > /tmp/cpu_profile.svg
```
Open `/tmp/cpu_profile.svg` in any web browser to interactively click and zoom into functions.

---

## 6. Measuring Hardware Efficiency: IPC & Cache Misses

Run `perf stat` to evaluate hardware-level compute efficiency:

```bash
sudo perf stat -p <PID> -- sleep 5
```
```
 Performance counter stats for process id '1420':

     18,452.12 msec task-clock                       #    3.690 CPUs utilized
         1,240      context-switches                 #   67.201 /sec
            12      cpu-migrations                   #    0.650 /sec
           412      page-faults                      #   22.328 /sec
52,102,401,920      cycles                           #    2.824 GHz
28,451,200,100      instructions                     #    0.55  insn per cycle
 4,510,200,120      branches                         #  244.427 M/sec
   142,500,200      branch-misses                    #    3.16% of all branches
   412,050,100      cache-misses                     #   18.42% of all L1D hits

       5.000841201 seconds time elapsed
```

### Key Ratios:
- **Instructions Per Cycle (IPC)**: `instructions / cycles = 0.55`.
  - IPC > 1.0: Good compute efficiency.
  - IPC < 0.7: CPU is spending most of its time stalled waiting for memory fetches from RAM!
- **Cache Misses**: High cache-miss percentages indicate non-contiguous memory access patterns (e.g. linked lists vs contiguous arrays).

---

## 7. Verification & Troubleshooting Runbook

### Prerequisites Checklist
```bash
# Install perf on Debian/Ubuntu
sudo apt-get install -y linux-tools-common linux-tools-$(uname -r)

# Check perf event paranoid setting (controls unprivileged access)
cat /proc/sys/kernel/perf_event_paranoid
```
- `perf_event_paranoid = 2`: Unprivileged access disallowed (standard).
- To profile as non-root user during local development: `sudo sysctl -w kernel.perf_event_paranoid=1`.

### Common Pitfalls & Solutions
1. **Symbols appear as hex addresses (`0x00007f...`) without function names**:
   - Cause: Binary is stripped of debugging symbols or compiled without `-g` / `-fno-omit-frame-pointer`.
   - Solution: Compile with `-fno-omit-frame-pointer -g`. For JVM/Node.js, install `perf-map-agent` to map JIT symbols.
2. **`perf: command not found` or package mismatch**:
   - Cause: `perf` version must match running kernel version.
   - Solution: Install kernel-specific tools: `linux-tools-$(uname -r)`.
3. **Missing kernel symbols (`[unknown]`)**:
   - Cause: `kptr_restrict` hides kernel memory addresses from non-root users.
   - Solution: Run `perf` with `sudo` or set `sysctl kernel.kptr_restrict=0`.

---

## 8. Key Takeaways
- `perf` uses low-overhead statistical sampling (< 1% impact), making it safe for production performance profiling.
- `perf top` provides instantaneous, system-wide visibility into user and kernel hotspots.
- Use `perf record -F 99 -a -g` coupled with FlameGraphs to visually pinpoint CPU bottlenecks.
- IPC (Instructions Per Cycle) from `perf stat` reveals whether a process is compute-bound or memory-stalled.
