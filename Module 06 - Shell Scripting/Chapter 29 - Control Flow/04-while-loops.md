# Chapter 29: Control Flow - While Loops in Bash

## 1. Introduction: Condition-Driven Iteration

While a `for` loop executes a predetermined number of times across a fixed list, a **`while` loop** continues iterating as long as a specified condition or command continues to evaluate as **True (exit code 0)**.

In production DevOps engineering, `while` loops are used for:
- **Polling and Readiness Probes**: Waiting for a database, cloud VM, or Kubernetes pod to initialize.
- **Reading Files Line-by-Line**: Parsing CSV files, log dumps, or system tables safely.
- **Retry Backoff Loops**: Retrying failed network requests with delay timeouts.

---

## 2. Basic Syntax and Condition Polling

**Syntax:**
```bash
while [ condition_or_command ]; do
    # Commands to execute while condition evaluates as success (0)
done
```

### Example: Polling for Database Readiness
When spinning up a PostgreSQL container in a CI/CD pipeline, the application must wait until the database socket is ready before running migrations:

```bash
#!/usr/bin/env bash
DB_HOST="127.0.0.1"
DB_PORT="5432"
TIMEOUT_SECS=30
ELAPSED=0

echo "Waiting for PostgreSQL on ${DB_HOST}:${DB_PORT} to accept connections..."

# While nc fails to connect, sleep and retry:
while ! nc -z "${DB_HOST}" "${DB_PORT}" 2>/dev/null; do
    if [ "${ELAPSED}" -ge "${TIMEOUT_SECS}" ]; then
        echo "[ERROR] Database connection timed out after ${TIMEOUT_SECS}s!" >&2
        exit 1
    fi
    echo "Database not ready yet. Retrying in 2 seconds... (${ELAPSED}s elapsed)"
    sleep 2
    ELAPSED=$((ELAPSED + 2))
done

echo "[SUCCESS] Database is ready! Proceeding with migrations."
```

---

## 3. Reading Files Line-by-Line (The Standard Pattern)

Reading files line-by-line is one of the most common tasks in systems administration. 

The universal, robust, safe way to read a file line-by-line in Bash is using **`while IFS= read -r line`**:

```bash
#!/usr/bin/env bash
INPUT_FILE="/etc/hosts"

while IFS= read -r line || [[ -n "${line}" ]]; do
    # Skip empty lines and comment lines
    [[ -z "${line}" || "${line}" =~ ^# ]] && continue

    echo "Host record: ${line}"
done < "${INPUT_FILE}"
```

### Why Every Part of this Pattern is Mandatory:
1. **`IFS=`**: Setting the Internal Field Separator to empty prevents leading and trailing whitespace from being stripped from the line.
2. **`read -r`**: The `-r` flag prevents backslashes from acting as escape characters (preserves raw paths like `C:\path` or regexes).
3. **`|| [[ -n "${line}" ]]`**: Ensures the very last line of the file is processed even if it does not end with a newline character (`\n`).
4. **`< "${INPUT_FILE}"`**: Direct input redirection keeps the loop inside the **parent shell** (avoiding subshell pipe traps).

---

## 4. Infinite Loops with `while true`

In daemon monitoring agents or background worker loops, scripts run indefinitely until interrupted:

```bash
while true; do
    echo "Heartbeat check at $(date '+%Y-%m-%d %H:%M:%S')"
    # Do monitoring work...
    sleep 60
done
```
*(Tip: `while :; do` is a POSIX shorthand for `while true; do`, where `:` is the built-in null command that always returns 0).*

---

## 5. Summary Takeaways

- `while` loops execute as long as the test condition or command returns exit code `0`.
- Essential for polling readiness checks and retry backoff loops in CI/CD.
- Always use `while IFS= read -r line; do ... done < file` to parse files line-by-line safely.
- Avoid piping files into while loops (`cat file | while`) to prevent subshell variable loss.
