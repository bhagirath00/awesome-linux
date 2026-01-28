# Library Call Tracing with ltrace

## 1. Architectural Concept & Deep Dive

While `strace` monitors the boundary between **userspace and the Linux kernel** (system calls), **`ltrace`** monitors the boundary between **the application program and dynamic shared libraries** (`.so` files, such as `libc.so.6`, `libssl.so`, `libcrypto.so`, or `libm.so`).

Most compiled programs (C, C++, Rust, Python/Ruby runtimes) do not invoke system calls directly. Instead, they call high-level C library wrapper functions:
- A program calls `printf("Hello %s\n", name)` in `libc.so`.
- `libc.so` performs string formatting and internal buffering, and eventually calls the kernel system call `write(1, "Hello Alice\n", 12)`.

`ltrace` intercepts these calls before they reach the library implementation using breakpoint injection into the Procedure Linkage Table (PLT):

```
+─────────────────────────────────────────────────────────────+
|                    WHERE LTRACE INTERCEPTS                  |
|                                                             |
|   Application Binary (main)                                 |
|          │                                                  |
|          ├── strcmp(user_input, "supersecret")              |
|          │                                                  |
|          ▼ (Breakpoint Trap in PLT)                         |
|   +─────────────────────────────────────────────────────+   |
|   |                      LTRACE                         |   |
|   |   Intercepts dynamic library symbol call            |   |
|   |   Reads arguments from registers / stack            |   |
|   +─────────────────────────────────────────────────────+   |
|          │                                                  |
|          ▼                                                  |
|   Shared Library (libc.so.6 / libssl.so)                    |
|          │                                                  |
|          ├── write(1, ...)  ──► Kernel System Call (strace) |
|          │                                                  |
|          ▼                                                  |
|   Linux Kernel (Ring 0)                                     |
+─────────────────────────────────────────────────────────────+
```

---

## 2. Comparison Matrix: strace vs ltrace

| Dimension | `strace` | `ltrace` |
| :--- | :--- | :--- |
| **Layer Intercepted** | Userspace to Kernel (Ring 3 -> Ring 0) | Binary to Shared Dynamic Library (Ring 3 -> Ring 3) |
| **Mechanism** | Kernel `ptrace(PTRACE_SYSCALL)` | Breakpoint injection (`int 3`) into the PLT |
| **Library Calls** | Invisible (only shows resulting syscalls) | Captured (`strcmp`, `malloc`, `SSL_write`, `getenv`) |
| **Statically Linked Binaries**| Works perfectly | **Fails completely** (no PLT/shared dynamic links) |
| **Go Binaries** | Works (Go compiles direct syscalls) | Fails on pure Go (no dynamic C library dependencies) |
| **Performance Overhead**| Significant (stops on every syscall) | Very High (stops on every matched function call) |

---

## 3. Command-Line Options Reference

| Flag | Description | Production Use Case |
| :--- | :--- | :--- |
| **`-p <PID>`** | Attach to a running process PID. | Inspect library calls of a running daemon. |
| **`-e <filter>`** | Filter by library symbol name (e.g. `-e malloc+free`, `-e getenv`). | Focus on specific operations (e.g., memory leaks, auth checks). |
| **`-l <lib.so>`** | Trace only symbols originating from a specific library. | Focus exclusively on `libssl.so` or `libcrypto.so`. |
| **`-s <size>`** | Maximum length of displayed strings (default 32). | View complete passwords, tokens, or URL paths passed to functions. |
| **`-c`** | Summarize call counts and time spent in each library function. | Benchmark application library bottlenecks. |
| **`-S`** | Display **both** library calls AND system calls together! | Complete unified trace across user libraries and kernel. |

---

## 4. Hands-on Terminal Lab: Uncovering Secrets & Logic via ltrace

Let's examine how `ltrace` exposes internal application logic and authentication strings without requiring source code.

### Step 1: Trace Standard String Comparisons
Consider a compiled binary checking an authorization license:

```bash
# Run a binary under ltrace with extended string display
ltrace -s 128 ./license_validator "my_test_key"
```
```
getenv("LICENSE_PATH")                                 = "/etc/license.key"
fopen("/etc/license.key", "r")                         = 0
strcmp("my_test_key", "PROD-ENTERPRISE-KEY-99881")    = -1
puts("License Validation Failed!")                     = 28
+++ exited (status 1) +++
```
- **Analysis**: `ltrace` instantly revealed:
  1. The program looked up environment variable `LICENSE_PATH`.
  2. It attempted to open `/etc/license.key`.
  3. It executed `strcmp` against the hardcoded string `"PROD-ENTERPRISE-KEY-99881"`, exposing the exact expected key!

### Step 2: Tracking Memory Leaks (`malloc` and `free`)
Inspect whether an application cleans up heap allocations properly:

```bash
ltrace -e malloc+free -c ./parser_tool
```
```
% time     seconds  usecs/call     calls      function
------ ----------- ----------- --------- --------------------
 58.12    0.045000           9      5000 malloc
 41.88    0.032400           8      3200 free
------ ----------- ----------- --------- --------------------
100.00    0.077400                  8200 total
```
- **Leak Detected**: 5,000 `malloc` calls vs only 3,200 `free` calls. 1,800 heap blocks were leaked!

### Step 3: Intercepting Encrypted Payloads Before Encryption (`libssl`)
Because `ltrace` operates in userspace before the data is encrypted and passed to the network card:

```bash
# Trace plaintext payloads sent through OpenSSL SSL_write
sudo ltrace -l libssl.so.3 -e SSL_write -s 2048 -p <PID>
```

---

## 5. Limitations & Caveats

1. **Statically Linked Binaries**:
   If an application is compiled with `gcc -static` or built in Go with `CGO_ENABLED=0`, all code is inlined into a single monolithic binary. There is no Dynamic Linker (`ld-linux.so`) and no PLT; `ltrace` cannot hook anything.
2. **Stripped Binaries & Compiler Inlining**:
   Modern compilers with `-O2` or `-O3` inline small library functions directly into machine code instructions, bypassing dynamic symbol dispatch.
3. **High Overhead in Tight Loops**:
   Tracing frequently called functions like `memcpy` or `strlen` inside a tight processing loop will freeze the application. Always filter specifically with `-e`!

---

## 6. Verification & Troubleshooting Runbook

### Diagnostic Checklist
```bash
# Verify if a target binary is dynamically linked (prerequisite for ltrace!)
file /usr/bin/curl
# Must report: "dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2"

# View all shared libraries dynamically linked to a binary
ldd /usr/bin/curl
```

### Common Pitfalls & Solutions
1. **`ltrace: no dynamic symbols found`**:
   - Cause: The target binary is statically linked or does not use shared libraries.
   - Solution: Use `strace` for system-level visibility, or use `perf` / `gdb` for symbol inspection.
2. **Output is truncated with `...`**:
   - Cause: Default string buffer limit is 32 characters.
   - Solution: Increase string limit via `-s 512`.
3. **Application crashes with SIGSEGV under ltrace**:
   - Cause: Breakpoint injection conflicts with internal memory protection (ASLR/DEP) or thread timing in proprietary binaries.
   - Solution: Trace specific functions only (`-e function_name`) or switch to eBPF dynamic probes (`bpftrace`).

---

## 7. Key Takeaways
- `ltrace` traces calls to dynamic shared libraries (`libc`, `openssl`), revealing function parameters, string comparisons, and return values.
- Unlike `strace`, `ltrace` requires dynamically linked binaries and hooks into the Procedure Linkage Table (PLT).
- Use `ltrace -c` to profile function bottlenecks and detect memory leaks (`malloc` vs `free`).
- Use `-S` to trace shared library calls and kernel system calls simultaneously.
