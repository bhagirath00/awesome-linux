# Chapter 22: Process Management - The kill, killall, and pkill Commands

## 1. Introduction: How Process Termination Works

In Linux, the `kill` command does not simply "slay" a program immediately; its actual function is to **send an asynchronous signal** to one or more processes. 

The target process intercepts the signal and executes its designated **signal handler routine**. Depending on the signal sent, the process can gracefully flush databases and close open file descriptors (`SIGTERM`), reload its configuration on the fly (`SIGHUP`), or be forcibly destroyed by the kernel (`SIGKILL`).

```
+-------------------------------------------------------------------------+
|                       Process Signaling Mechanism                       |
+-------------------------------------------------------------------------+
|  User / Shell                Kernel System Call        Target Process   |
|  +--------------------+     +--------------------+     +--------------+ |
|  | kill -15 1420      | ==> | sys_kill(1420, 15) | ==> | Process 1420 | |
|  +--------------------+     +--------------------+     +-------+------+ |
|                                                                |        |
|                                        Catches SIGTERM Signal  v        |
|                                        +------------------------------+ |
|                                        | 1. Stop accepting requests   | |
|                                        | 2. Finish ongoing database tx| |
|                                        | 3. Flush buffers to disk     | |
|                                        | 4. Delete /tmp/app.pid file  | |
|                                        | 5. Call exit(0)              | |
|                                        +------------------------------+ |
+-------------------------------------------------------------------------+
```

---

## 2. The Standard `kill` Command

The `kill` utility operates on **Process IDs (PIDs)**:

**Syntax:**
```bash
kill [signal_option] <PID> [PID...]
```

```bash
# Send default signal (SIGTERM / 15)
kill 1420

# Send SIGTERM explicitly by name
kill -SIGTERM 1420
kill -TERM 1420

# Send SIGKILL (force termination by kernel)
kill -9 1420
kill -SIGKILL 1420
```

> [!WARNING]
> Never jump directly to `kill -9` (`SIGKILL`)! 
> 
> `SIGKILL` cannot be handled or caught by the application. The kernel abruptly frees memory and cuts off execution mid-flight. If a database (PostgreSQL, MySQL, Redis) or application is currently writing to disk, issuing `kill -9` can cause **unrecoverable data corruption** and leave lockfiles orphaned. Always attempt `kill -15` (`SIGTERM`) first, wait a few seconds for graceful shutdown, and use `kill -9` only as a last resort.

---

## 3. Targeted Termination with `pkill`

Remembering and typing numerical PIDs is tedious in production. `pkill` allows you to send signals to processes based on **process name patterns, usernames, or terminal sessions**:

**Syntax:**
```bash
pkill [options] <pattern>
```

```bash
# Gracefully terminate all Nginx worker and master processes
sudo pkill -15 nginx

# Forcibly kill all processes owned by user "baduser"
sudo pkill -9 -u baduser

# Terminate processes matching exact full command line
pkill -f "python3 /opt/worker.py"
```

> [!TIP]
> Before running `pkill`, use its companion tool `pgrep` to preview exactly which PIDs match your pattern:
> ```bash
> pgrep -l nginx
> ```
> ```
> 1420 nginx
> 1421 nginx
> ```

---

## 4. Bulk Termination with `killall`

`killall` sends a signal to all processes matching an **exact executable name**:

```bash
# Gracefully terminate all instances of httpd
sudo killall httpd

# Forcibly terminate all node processes
sudo killall -9 node

# Interactive mode (prompts for confirmation before killing each process)
killall -i firefox
```

---

## 5. Tool Comparison Matrix

| Command | Target Identifier | Matches Regex/Substring? | Best Use Case |
| :--- | :--- | :--- | :--- |
| `kill` | Process ID (`PID`) | No (Exact PID required) | Single targeted process when PID is already known. |
| `pkill` | Name Pattern / User / TTY | Yes (Pattern matching) | Microservices, command-line arguments (`-f`), killing by user. |
| `killall`| Exact Process Name | No (Exact binary name) | Terminating all instances of a daemon by executable name. |

---

## 6. Summary Takeaways

- `kill` sends signals; it does not automatically mean "destroy".
- Default signal is `15` (`SIGTERM`), which gives processes time to save state.
- `SIGKILL` (`-9`) bypasses the application and forces immediate kernel termination—avoid using it on databases without trying `SIGTERM` first.
- Use `pgrep` to safely preview processes before issuing `pkill`.
