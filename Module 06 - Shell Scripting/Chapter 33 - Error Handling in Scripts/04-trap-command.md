# Chapter 33: Error Handling in Scripts - Signal Trapping and Cleanup with trap

## 1. Introduction: What is the trap Built-in?

When a script is abruptly terminated—whether due to a developer pressing `Ctrl + C` (`SIGINT`), a deployment timeout issuing `SIGTERM`, an unhandled error under `set -e`, or normal exit—it often leaves behind:
- Temporary lock files (`/var/lock/deploy.lock`) that prevent future runs.
- Dangling scratch directories (`/tmp/build.XXXXXX`) filling disk storage.
- Orphaned background child processes or active SSH tunnels.

The **`trap`** built-in allows you to register a **cleanup function** that the kernel and shell **guarantee to execute upon exit or signal receipt**, providing the equivalent of `try...finally` in modern programming languages.

```
+--------------------------------------------------------------------------+
|                     Signal Trapping Architecture                         |
+--------------------------------------------------------------------------+
|  Script Execution                                                        |
|  Creates /tmp/scratch.4120                                               |
|  Creates /var/run/deploy.pid                                             |
|                                                                          |
|  Interruption Occurs: User presses Ctrl+C (SIGINT) OR Error occurs       |
|                                                                          |
|  Kernel notifies Bash ---> Intercepted by TRAP!                          |
|                                                                          |
|  Executes registered cleanup() routine:                                  |
|  1. rm -rf /tmp/scratch.4120                                             |
|  2. rm -f /var/run/deploy.pid                                            |
|  3. Kills child workers                                                  |
|                                                                          |
|  Script exits cleanly with zero resource leaks!                          |
+--------------------------------------------------------------------------+
```

---

## 2. Command Syntax and Trappable Signals

**Syntax:**
```bash
trap <command_or_function> <SIGNAL_OR_EVENT...>
```

### Essential Trappable Events:
| Event / Signal | Trigger Condition |
| :--- | :--- |
| **`EXIT`** (Pseudo-signal 0)| **The universal catch-all**. Triggers whenever the script terminates for **ANY** reason (normal finish, `exit`, or crash under `set -e`). |
| **`SIGINT`** (Signal 2) | User interrupts script from terminal via **`Ctrl + C`**. |
| **`SIGTERM`** (Signal 15)| Standard termination request (sent by `systemctl stop`, Docker, or Kubernetes). |
| **`SIGHUP`** (Signal 1) | Terminal window closed or SSH session dropped. |
| **`ERR`** | Special Bash hook: triggers whenever any command returns a non-zero exit code. |

---

## 3. The Gold Standard Cleanup Pattern

In production DevOps scripts, register a unified `cleanup` function on `EXIT, SIGINT, SIGTERM`:

```bash
#!/usr/bin/env bash
set -euo pipefail

# 1. Create temporary directory securely
SCRATCH_DIR=$(mktemp -d /tmp/deployment.XXXXXX)
LOCK_FILE="/var/run/app-deploy.lock"

# 2. Acquire lock
exec 200>"${LOCK_FILE}"
flock -n 200 || { echo "[ERROR] Another deployment is currently running!" >&2; exit 1; }

# 3. Define the guaranteed cleanup function
cleanup() {
    local exit_code="$?"
    echo -e "\n[CLEANUP] Cleaning up temporary resources (Exit code: ${exit_code})..."

    # Remove temporary scratch directories
    if [[ -d "${SCRATCH_DIR}" ]]; then
        rm -rf "${SCRATCH_DIR}"
        echo "[CLEANUP] Removed: ${SCRATCH_DIR}"
    fi

    # Release lockfile
    flock -u 200 2>/dev/null || true
    rm -f "${LOCK_FILE}" 2>/dev/null || true

    # Preserve the original failure exit code if script failed
    exit "${exit_code}"
}

# 4. Register the trap on EXIT and signals
trap cleanup EXIT INT TERM

# 5. Main Script Logic
echo "Deployment workspace initialized at ${SCRATCH_DIR}"
echo "Downloading build artifacts..."
sleep 2
echo "Simulating workload..."
sleep 2

echo "Deployment finished successfully."
```

If you interrupt this script with `Ctrl+C` while it is running:
```
^C
[CLEANUP] Cleaning up temporary resources (Exit code: 130)...
[CLEANUP] Removed: /tmp/deployment.v71A49
```
The temporary files and locks are guaranteed to be cleaned up!

---

## 4. Advanced: Line Number Error Tracing with `trap ... ERR`

You can use the `ERR` trap to print the exact filename and line number where an unhandled failure occurred:

```bash
#!/usr/bin/env bash
set -euo pipefail

trap 'echo "[CRASH] Error on line ${LINENO} while executing: ${BASH_COMMAND}" >&2' ERR

echo "Step 1: Running health check..."
ls /etc/hosts > /dev/null

echo "Step 2: Intentionally failing command..."
cat /nonexistent/file.conf

echo "Step 3: This line will never execute."
```

```
Step 1: Running health check...
Step 2: Intentionally failing command...
cat: /nonexistent/file.conf: No such file or directory
[CRASH] Error on line 10 while executing: cat /nonexistent/file.conf
```

---

## 5. Summary Takeaways

- Use `trap cleanup EXIT INT TERM` to ensure scratch files and locks are always removed.
- `EXIT` triggers regardless of whether the script succeeds, fails, or exits early.
- Always preserve the original exit code inside the cleanup handler (`exit "$?"`).
- Use `trap '...' ERR` to instantly pinpoint the line number of unexpected crashes.
