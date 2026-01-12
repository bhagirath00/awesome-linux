# Chapter 40: CPU Performance - CPU Profiling and Flame Graphs with perf

## 1. Introduction: What is CPU Profiling?

Standard monitoring tools (`top`, `ps`, `mpstat`) tell you *that* a process is consuming 100% of a CPU core. However, they cannot tell you **why** or **where** inside the application's code those cycles are being burned. 

Is the application stuck in an unoptimized regex evaluation, a cryptographic hashing loop, a slow string concatenation, or a kernel lock contention?

**CPU Profiling** uses hardware performance counters and kernel timers to sample the CPU's instruction pointer and stack trace thousands of times per second, building a statistical distribution of which functions consume the most execution time.

In Linux, the premier tool for this is **`perf`** (the Linux Performance Events subsystem).

```
+--------------------------------------------------------------------------+
|                      CPU Profiling Sampling Process                      |
+--------------------------------------------------------------------------+
|  Linux Kernel Timer / Hardware PMU Interrupt (e.g., 99 Hertz / 99 Hz)    |
|                                |                                         |
|  At each tick, perf records:   v                                         |
|  - Process ID & Thread ID                                                |
|  - Exact Instruction Pointer (RIP register)                              |
|  - Full Call Stack: main() -> handle_req() -> parse_json() -> malloc()   |
|                                |                                         |
|                                v                                         |
|  Recorded into perf.data       v Aggregated into Flame Graph             |
|  [ Identifies parse_json() accounts for 78% of all CPU execution! ]     |
+--------------------------------------------------------------------------+
```

---

## 2. Installing the perf Subsystem

`perf` is maintained directly in the upstream Linux kernel source tree. Its package name matches the exact kernel version currently running on the server:

```bash
# Ubuntu / Debian
sudo apt install -y linux-tools-common linux-tools-generic linux-tools-$(uname -r)

# RHEL / Rocky / AlmaLinux
sudo dnf install -y perf
```

---

## 3. Real-Time Top Profiling with `perf top`

`perf top` acts like `top`, but displays **functions and symbols** instead of process names:

```bash
sudo perf top
```

```
Samples: 42K of event 'cycles:P', 4000 Hz, Event count (approx.): 241981249
Overhead  Shared Object       Symbol
  34.12%  node                [.] v8::internal::Scavenger::Scavenge
  18.40%  libc.so.6           [.] __memmove_avx_unaligned_erms
  12.05%  [kernel]            [k] copy_user_enhanced_fast_string
   8.20%  node                [.] node::crypto::PBKDF2
   4.10%  [kernel]            [k] finish_task_switch.isra.0
```

### Overhead Interpretation:
- `[.]`: User-space application symbol.
- `[k]`: Linux kernel function symbol.
- The top entry (`v8::internal::Scavenger`) immediately reveals that Node.js is spending 34% of its entire CPU budget on garbage collection!

---

## 4. Recording and Analyzing Profiles with `perf record`

To profile an active production application (e.g., Nginx, Envoy, or Python) for 30 seconds:

```bash
# -F 99: Sample at 99 Hertz (avoids lockstep resonance with 100Hz timers)
# -p 14205: Target PID
# -g: Capture call stack traces (call-graph)
# -- sleep 30: Run recording for 30 seconds
sudo perf record -F 99 -p 14205 -g -- sleep 30
```
This writes the samples into `perf.data`.

### Inspecting the Report:
```bash
sudo perf report -n --stdio
```

---

## 5. Visualizing CPU Profiles with Flame Graphs

Invented by Brendan Gregg, **Flame Graphs** are the universal visual standard for analyzing CPU profiles. 

In a Flame Graph:
- The **X-axis** represents the percentage of CPU time spent in each function (wider boxes consume more CPU).
- The **Y-axis** represents the call stack depth (functions on top are called by functions underneath them).

```
+--------------------------------------------------------------------------+
|                      Conceptual CPU Flame Graph                          |
+--------------------------------------------------------------------------+
|  [ parse_json() (60% width) ]            [ compute_hash() (25% width) ]  |
|  [------------------ process_http_request() --------------------------]  |
|  [-------------------------- worker_thread() --------------------------]  |
|  [------------------------------ main() -------------------------------]  |
+--------------------------------------------------------------------------+
```

### Generating a Flame Graph SVG:
```bash
# Clone the open-source FlameGraph scripts
git clone https://github.com/brendangregg/FlameGraph.git /tmp/FlameGraph

# 1. Unpack stack traces from perf.data
sudo perf script > /tmp/out.perf

# 2. Collapse stacks into single lines
/tmp/FlameGraph/stackcollapse-perf.pl /tmp/out.perf > /tmp/out.folded

# 3. Generate interactive vector SVG Flame Graph
/tmp/FlameGraph/flamegraph.pl /tmp/out.folded > /var/www/html/flamegraph.svg
```
Open `flamegraph.svg` in any web browser to zoom, search, and click through execution stacks!

---

## 6. Summary Takeaways

- `perf` profiles the exact functions and kernel routines consuming CPU cycles.
- Sample at 99Hz (`-F 99`) to prevent lockstep sampling artifacts.
- Always include `-g` to capture complete call graphs and stack traces.
- Convert raw `perf.data` into Flame Graphs to visually identify performance bottlenecks in seconds.
