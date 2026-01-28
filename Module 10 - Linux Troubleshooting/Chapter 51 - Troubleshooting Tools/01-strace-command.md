# System Call Tracing with strace

## 1. Architectural Concept & Deep Dive

Whenever a userspace application needs to interact with the outside world—read a configuration file, open a network socket, allocate memory, or fork a process—it must transition from CPU Ring 3 (Userspace) to CPU Ring 0 (Kernel Space) via a **System Call (syscall)**.

**`strace`** is the premier diagnostic tool for capturing, intercepting, and recording every system call made by a process and the signals it receives. It operates via the Linux **`ptrace(2)`** kernel debugging subsystem:

```
+─────────────────────────────────────────────────────────────+
|                    HOW STRACE USES PTRACE                   |
|                                                             |
|   Target Process (e.g. Python / Go / Nginx)                 |
|          │                                                  |
|          ├── openat(AT_FDCWD, "/etc/app.conf", O_RDONLY)    |
|          │                                                  |
|          ▼ (Syscall Trap Triggered)                         |
|   +─────────────────────────────────────────────────────+   |
|   |             Linux Kernel ptrace Engine              |   |
|   |   1. Halts target process execution (SIGTRAP)       |   |
|   |   2. Context-switches to strace process             |   |
|   |   3. strace reads CPU registers (rax, rdi, rsi...)  |   |
|   |   4. Kernel resumes target process                  |   |
|   |   5. Syscall completes; Kernel halts target AGAIN   |   |
|   |   6. strace reads return value and latency          |   |
|   +─────────────────────────────────────────────────────+   |
|          │                                                  |
|          ▼                                                  |
|   strace outputs: openat(...) = 3 <0.000045>                |
+─────────────────────────────────────────────────────────────+
```

> **Production Warning (The ptrace Overhead)**:
> Because `ptrace` stops the target process twice for every single system call, `strace` can slow down high-throughput production applications by **5x to 100x**. Use targeted filters (`-e trace=...`) and avoid running unrestricted `strace` on high-concurrency database engines.

---

## 2. Essential Command-Line Flags Reference

| Flag | Description | Production Use Case |
| :--- | :--- | :--- |
| **`-p <PID>`** | Attach to an already running process PID. | Debug a stuck or hung live service. |
| **`-f`** | Follow forks (trace all child processes and threads). | **Mandatory** for multi-threaded/multi-process daemons. |
| **`-e trace=<set>`** | Filter by syscall type (e.g. `file`, `network`, `process`, `desc`). | Drastically reduces overhead and output noise. |
| **`-s <size>`** | Maximum string length to display (default is only 32 chars). | Set `-s 2048` to view full SQL queries or HTTP payloads. |
| **`-T`** | Record the time spent inside each system call. | Pinpoint slow disk reads or blocking database network calls. |
| **`-tt`** | Prefix each line with microsecond timestamp (`HH:MM:SS.uuuuuu`). | Correlate syscall events with external application logs. |
| **`-c`** | Profiling mode: summarize counts, errors, and total time per syscall. | Instant performance overview without terminal spam. |
| **`-o <file>`** | Write output to a dedicated file instead of stderr. | Essential when tracing services producing thousands of calls. |

---

## 3. Real-World Triage Scenarios

### Scenario A: Finding Which Missing File Is Causing a Crash
An application crashes on startup with a vague `Error: Configuration missing`:

```bash
strace -f -e trace=openat,access myapp 2>&1 | grep -E "ENOENT|EACCES"
```
```
openat(AT_FDCWD, "/etc/myapp/config.yaml", O_RDONLY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/home/deployer/.myapp.json", O_RDONLY) = -1 EACCES (Permission denied)
```
- **Instant Discovery**: The application checks `/etc/myapp/config.yaml` (missing), falls back to `~/.myapp.json`, and crashes due to a permission failure!

### Scenario B: Diagnosing Why a Process Is Hanging
A background script has been hanging for 45 minutes without outputting anything:

```bash
# Attach to stuck PID, track thread forks (-f), timestamps (-tt), and syscall duration (-T)
sudo strace -p 18451 -f -tt -T
```
```
[pid 18452] 08:20:15.102451 connect(4, {sa_family=AF_INET, sin_port=htons(5432), sin_addr=inet_addr("10.0.1.50")}, 16) = -1 EINPROGRESS (Operation now in progress) <0.000021>
[pid 18452] 08:20:15.102510 poll([{fd=4, events=POLLOUT}], 1, 10000
```
- **Instant Discovery**: The process is blocked inside `poll()` waiting on a TCP connection to PostgreSQL (`10.0.1.50:5432`) that is being silently dropped by a security group or firewall!

### Scenario C: High-Level Performance Profiling (`-c`)
Determine where a slow batch processor is spending its runtime:

```bash
strace -c -p 1420
# (Let it run for 10 seconds, then hit Ctrl+C)
```
```
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 84.12    5.120400         512     10000           futex
 12.45    0.758000          15     50000           read
  3.12    0.190000          19     10000           write
  0.31    0.018900           9      2100       120 openat
------ ----------- ----------- --------- --------- ----------------
100.00    6.087300                 72100       120 total
```
- **Analysis**: 84% of time is consumed in `futex` (userspace mutex lock contention). The bottleneck is thread synchronization, not disk I/O.

---

## 4. Hands-on Terminal Lab: Advanced Tracing Techniques

### Step 1: Trace Only Network Activity
```bash
strace -e trace=network -s 512 curl -s https://example.com > /dev/null
```
```
socket(AF_INET, SOCK_DGRAM|SOCK_CLOEXEC, IPPROTO_IP) = 3
connect(3, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("127.0.0.53")}, 16) = 0
sendto(3, "\241\20...\0\0\1\0\0\0\0\0\0\7example\3com\0\0\1\0\1", 40, MSG_NOSIGNAL, NULL, 0) = 40
recvfrom(3, "\241\20...\0\1\0\1\0\0\0\0\7example\3com\0\0\1\0\1...", 2048, 0, ...) = 56
socket(AF_INET, SOCK_STREAM|SOCK_CLOEXEC, IPPROTO_TCP) = 3
connect(3, {sa_family=AF_INET, sin_port=htons(443), sin_addr=inet_addr("93.184.216.34")}, 16) = 0
```
*(Shows DNS query to 127.0.0.53 followed by TCP connection to port 443).*

### Step 2: Tampering with Syscalls (Fault Injection)
Did you know `strace` can deliberately inject failures to test application resilience?
```bash
# Inject EIO disk error into all read calls for testing error handling
strace -e read -e fault=read:error=EIO ls
```
```
ls: reading directory '.': Input/output error
```

---

## 5. Verification & Troubleshooting Runbook

### Health & Diagnostic Tips
```bash
# Verify if ptrace is permitted on the system (0 = classic, 1 = restricted to children)
cat /proc/sys/kernel/yama/ptrace_scope
```
- If `ptrace_scope` is `1` (standard in Ubuntu/Debian), you cannot attach to processes owned by your user unless run with `sudo`.

### Common Pitfalls & Solutions
1. **`strace: attach: ptrace(PTRACE_SEIZE, ...): Operation not permitted`**:
   - Cause: Target process runs as another UID, Yama ptrace restrictions are active, or the process is already being traced by another debugger/gdb.
   - Solution: Use `sudo strace`. Verify no other gdb/strace session is attached (`grep TracerPid /proc/<PID>/status`).
2. **`strace` produces hundreds of megabytes of output per second**:
   - Cause: Tracing memory allocation (`mmap`, `brk`) or busy loop calls.
   - Solution: Filter specifically for desired subsystem: `-e trace=file` or `-e trace=network`.
3. **Containerized process cannot be traced**:
   - Cause: Container lacks `CAP_SYS_PTRACE`.
   - Solution: Add capability to container: `docker run --cap-add=SYS_PTRACE`.

---

## 6. Key Takeaways
- `strace` captures the boundary between userspace and the Linux kernel via `ptrace(2)`.
- Use `-p <PID> -f` to attach to multi-threaded running services, and `-T` to pinpoint syscall latency.
- Filter outputs (`-e trace=file,network`) to reduce overhead and focus on relevant interactions.
- Be cautious of production performance impacts; use `-c` for low-overhead summary profiling.
