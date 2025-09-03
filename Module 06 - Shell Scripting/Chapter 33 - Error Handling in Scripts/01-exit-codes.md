# Chapter 33: Error Handling in Scripts - Exit Codes and Status Propagation

## 1. Introduction: The Linux Exit Status Protocol

In Linux and Unix-like operating systems, every process that completes execution must return an integer status code back to its parent process via the kernel `exit(2)` system call. 

This numerical value is called the **Exit Status Code** (or return code). It is strictly an 8-bit unsigned integer ranging from **`0` to `255`**:

- **`0`**: **SUCCESS**. The command executed normally with zero errors.
- **`1` to `255`**: **FAILURE**. The command encountered an error, misconfiguration, or was terminated by an external signal.

In CI/CD pipelines (GitHub Actions, GitLab CI, Jenkins), automated runners inspect the script's exit status. If the script exits with `0`, the pipeline step turns green; if it exits with any non-zero value, the pipeline step turns red and stops downstream deployments.

---

## 2. Inspecting the Last Exit Code: `$?`

The special parameter **`$?`** holds the exit status code of the **most recently executed foreground command**:

```bash
# Successful command
ls /etc > /dev/null
echo "Exit code: $?"
# Output: Exit code: 0

# Failing command
cat /nonexistent/file.txt 2>/dev/null
echo "Exit code: $?"
# Output: Exit code: 1
```

> [!IMPORTANT]
> The value of `$?` is volatile! It is overwritten by **every subsequent command** you execute, including `echo`! 
> 
> If you need to evaluate an exit code across multiple logic branches, save it to a variable immediately:
> ```bash
> curl -s https://api.company.com/health
> RET_CODE="$?"
> 
> # Now $RET_CODE is preserved:
> if [ "$RET_CODE" -ne 0 ]; then
>     echo "Health check failed with code: ${RET_CODE}"
> fi
> ```

---

## 3. Standard Reserved Exit Codes

While any non-zero integer represents failure, POSIX and GNU conventions reserve specific exit code ranges:

| Exit Code | Meaning / Cause | Example Scenario |
| :--- | :--- | :--- |
| **`0`** | **Success** | Successful program execution. |
| **`1`** | **General / Catch-all Error** | Miscellaneous error (`let "var = 1/0"`). |
| **`2`** | **Misuse of Shell Built-in** | Missing keyword, invalid argument syntax in built-in. |
| **`126`** | **Command Cannot Execute** | Permission problem; file is not executable (`chmod -x`). |
| **`127`** | **Command Not Found** | Binary is missing from `$PATH` or typo in command name. |
| **`128`** | **Invalid Exit Argument** | Executing `exit 3.14` (non-integer). |
| **`128 + N`**| **Fatal Signal Termination** | Process killed by OS signal `N` (e.g., `128 + 9 = 137`). |
| **`130`** | **Terminated by Ctrl+C** | Signal 2 (`SIGINT`) -> `128 + 2 = 130`. |
| **`137`** | **Killed by SIGKILL / OOM** | Signal 9 (`SIGKILL`) -> `128 + 9 = 137` (Docker/K8s Out-Of-Memory). |
| **`143`** | **Terminated by SIGTERM** | Signal 15 (`SIGTERM`) -> `128 + 15 = 143` (Kubernetes pod eviction). |

---

## 4. Setting Exit Codes with `exit`

In shell scripts, use the `exit [N]` command to terminate script execution immediately with an explicit status code:

```bash
#!/usr/bin/env bash

validate_config() {
    if [[ ! -f "config.yaml" ]]; then
        echo "[ERROR] Missing config.yaml!" >&2
        exit 1  # Exit script with error code 1
    fi
}

validate_config
echo "Config validated."
exit 0  # Explicit success
```

---

## 5. Summary Takeaways

- Exit codes range from `0` to `255`; `0` indicates success, non-zero indicates failure.
- Always check exit status via `$?`.
- Exit codes `126` (cannot execute) and `127` (command not found) indicate environment issues.
- Exit code `137` (`128 + 9`) signifies process termination by OOM killer or `SIGKILL`.
- Always save `$?` to a variable immediately if needed for multiple evaluations.
