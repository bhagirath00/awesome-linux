# Chapter 22: Process Management - Linux Signals Overview

## 1. What are POSIX Signals?

A **signal** is an asynchronous notification sent by the Linux kernel (or by another process) to inform a target process that a specific event has occurred. 

When a signal arrives, the process's normal execution stream is interrupted. The process then takes one of three actions:
1. **Catches the signal**: Executes a custom signal handler function (e.g., saving data or closing network sockets).
2. **Ignores the signal**: Discards the signal completely (if permitted by the kernel).
3. **Executes default kernel action**: The default behavior defined by POSIX (such as terminating, dumping core, or stopping execution).

---

## 2. Listing All Available Signals

To inspect the full catalog of signals supported by your Linux kernel, run:

```bash
kill -l
```

```
 1) SIGHUP       2) SIGINT       3) SIGQUIT      4) SIGILL       5) SIGTRAP
 6) SIGABRT      7) SIGBUS       8) SIGFPE       9) SIGKILL     10) SIGUSR1
11) SIGSEGV     12) SIGUSR2     13) SIGPIPE     14) SIGALRM     15) SIGTERM
16) SIGSTKFLT   17) SIGCHLD     18) SIGCONT     19) SIGSTOP     20) SIGTSTP
21) SIGTTIN     22) SIGTTOU     23) SIGURG      24) SIGXCPU     25) SIGXFSZ
26) SIGVTALRM   27) SIGPROF     28) SIGWINCH    29) SIGIO       30) SIGPWR
31) SIGSYS      34) SIGRTMIN    ...             64) SIGRTMAX
```

Standard POSIX signals run from 1 to 31; real-time signals run from 34 to 64.

---

## 3. The 10 Most Critical Signals in DevOps

| Signal Number | Signal Name | Keyboard Shortcut | Can Be Caught/Ignored? | Default Action & Common DevOps Usage |
| :--- | :--- | :--- | :--- | :--- |
| **1** | `SIGHUP` | None | **Yes** | **Hangup / Reload**: Traditionally sent when terminal closes. Modern daemons (Nginx, Prometheus) interpret `SIGHUP` as a directive to reload configuration files without dropping connections. |
| **2** | `SIGINT` | `Ctrl + C` | **Yes** | **Interrupt**: Sent from controlling terminal to gracefully interrupt a running foreground command. |
| **3** | `SIGQUIT`| `Ctrl + \` | **Yes** | **Quit**: Like `SIGINT`, but instructs the kernel to produce a process core dump (`core.<PID>`) for debugging. |
| **9** | `SIGKILL`| None | **NO (Uncatchable)**| **Forced Kill**: Kernel immediately terminates process and frees memory. Cannot be intercepted, blocked, or ignored. |
| **10** | `SIGUSR1`| None | **Yes** | **User-Defined 1**: Custom application-specific hook. Nginx uses `SIGUSR1` to rotate log files without restarting. |
| **12** | `SIGUSR2`| None | **Yes** | **User-Defined 2**: Nginx uses `SIGUSR2` for zero-downtime binary upgrades. |
| **15** | `SIGTERM`| None | **Yes** | **Terminate (Default)**: Politely asks the process to exit cleanly, execute cleanup routines, and release ports. |
| **18** | `SIGCONT`| None | **Yes** | **Continue**: Resumes a previously suspended or paused process. |
| **19** | `SIGSTOP`| None | **NO (Uncatchable)**| **Forced Pause**: Kernel immediately freezes process execution. Cannot be caught or blocked. |
| **20** | `SIGTSTP`| `Ctrl + Z` | **Yes** | **Terminal Stop**: Requests process to suspend and move to background. |

---

## 4. The Two Uncatchable Signals: SIGKILL (9) vs. SIGSTOP (19)

In the Linux kernel signal architecture, two signals are hardcoded to **bypass process signal handlers entirely**:

```
+--------------------------------------------------------------------------+
|                        Catchable vs. Uncatchable                         |
+--------------------------------------------------------------------------+
|  Normal Signal (SIGTERM 15):                                             |
|  [ Kernel ] ---> [ Process Signal Table ] ---> [ Custom Handler Code ]   |
|                                                (Can delay, save, ignore) |
+--------------------------------------------------------------------------+
|  Uncatchable Signal (SIGKILL 9 / SIGSTOP 19):                            |
|  [ Kernel ] ---> [ Kernel Scheduler Directly Frees / Freezes Task ]      |
|                  (Bypasses process memory completely! Cannot be trapped) |
+--------------------------------------------------------------------------+
```

Because `SIGKILL` cannot be trapped, processes cannot clean up temporary lock files (such as `/var/run/app.pid`), finish in-flight HTTP requests, or commit database write-ahead logs (WAL).

---

## 5. Graceful Reloading in Production Daemons

Enterprise web servers and microservices leverage signals to perform administrative maintenance without incurring downtime:

```bash
# Reload Nginx configuration without dropping active client connections
sudo kill -HUP $(cat /var/run/nginx.pid)

# Re-open log files after logrotate runs
sudo kill -USR1 $(cat /var/run/nginx.pid)

# Initiate zero-downtime binary upgrade in Nginx
sudo kill -USR2 $(cat /var/run/nginx.pid)
```

---

## 6. Trapping Signals in Shell Scripts

In DevOps automation, shell scripts should capture termination signals to clean up temporary scratch files upon unexpected interruption:

```bash
#!/bin/bash
TEMP_DIR=$(mktemp -d /tmp/deploy.XXXXXX)

# Trap SIGINT (Ctrl+C) and SIGTERM (15)
cleanup() {
    echo -e "\n[!] Signal received. Removing temporary directory: ${TEMP_DIR}..."
    rm -rf "${TEMP_DIR}"
    exit 1
}

trap cleanup SIGINT SIGTERM

echo "Deployment running in ${TEMP_DIR}..."
sleep 60
```

---

## 7. Summary Takeaways

- Signals are asynchronous kernel software interrupts.
- `SIGTERM` (`15`) is the default and polite shutdown signal; always use it first.
- `SIGKILL` (`9`) and `SIGSTOP` (`19`) cannot be caught, blocked, or ignored.
- Use `SIGHUP` (`1`) to reload daemon configurations on the fly without downtime.
- Use `trap` in bash scripts to ensure resources are cleaned up on interruption.
