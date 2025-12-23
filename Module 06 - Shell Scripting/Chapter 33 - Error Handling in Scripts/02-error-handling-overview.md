# Chapter 33: Error Handling in Scripts - Error Handling Overview and Strategies

## 1. The Default Behavior of Bash: The Silent Killer

By default in standard Bash: **SCRIPTS DO NOT STOP WHEN A COMMAND FAILS!**

If command 1 fails on line 10, Bash prints an error message to `stderr`, completely ignores the failure, and blindly continues executing line 11, line 12, and line 13.

In DevOps automation, this default behavior can lead to catastrophic data loss.

### The Classic Catastrophe:
```bash
#!/bin/bash
# Suppose BUILD_DIR variable is empty due to a typo or failed previous command:
cd "$BUILD_DIR"
rm -rf *
```

Because `BUILD_DIR` was empty, `cd ""` failed! But Bash ignored the error and executed the next command:
```bash
rm -rf *
```
The script wiped out the **current working directory** (or the server's entire root filesystem if run as root)!

---

## 2. Core Strategies for Robust Error Handling

To make shell scripts production-ready, implement three layers of error defense:

```
+--------------------------------------------------------------------------+
|                     Three Pillars of Shell Error Defense                 |
+--------------------------------------------------------------------------+
|  Pillar 1: Fail-Fast Shell Flags (set -euo pipefail)                     |
|  - Halts execution instantly if ANY command fails                        |
|  - Treats unset variables as fatal errors                                |
|  - Captures failed commands hidden inside pipelines                      |
|                                                                          |
|  Pillar 2: Defensive Scripting Logic (Short-circuiting & Checks)         |
|  - Explicit exit code verification: command || exit 1                    |
|  - Pre-flight precondition assertions (test -f, command -v)              |
|                                                                          |
|  Pillar 3: Automated Resource Cleanup (Signals & Traps)                  |
|  - trap cleanup EXIT INT TERM                                            |
|  - Guarantees lock files, scratch dirs, and temp containers are wiped    |
+--------------------------------------------------------------------------+
```

---

## 3. Short-Circuiting with Logical Operators (`&&` and `||`)

In shell scripting, logical AND (`&&`) and logical OR (`||`) provide elegant, single-line error guards:

### 1. Guarding Preconditions (`&&`)
Executes the second command **only if the first command succeeds (exit code 0)**:
```bash
# Safe Directory Change:
cd "/opt/app/build" && rm -rf ./*
```
If `cd` fails, the `rm` command is **never executed**!

### 2. Fast Exit on Failure (`||`)
Executes the second command **only if the first command fails (non-zero exit code)**:
```bash
# Verify directory creation
mkdir -p "/var/log/myapp" || { echo "Failed to create log directory!" >&2; exit 1; }

# Download artifact or abort
curl -sSf "https://downloads.company.com/app.tar.gz" -o app.tar.gz || {
    echo "Download failed! Aborting." >&2
    exit 2
}
```

---

## 4. Intentional Error Handling Under Strict Mode

When writing scripts with fail-fast flags (`set -e`), some commands are **expected to return a non-zero exit code** under normal circumstances (e.g., `grep` returns 1 when no matches are found, or `diff` returns 1 when files differ).

If a command fails under `set -e`, the script aborts immediately. To allow an expected non-zero code without terminating the script:

```bash
# Allow grep to return 1 without killing the script:
MATCH=$(grep "search_term" file.txt || true)

# Explicitly test the command directly in an 'if' statement:
if grep -q "database_ready" /var/log/db.log; then
    echo "Database is ready."
else
    echo "Not ready yet."
fi
```
*(Commands evaluated as the condition of an `if`, `while`, or `until` statement are exempted from `set -e` termination).*

---

## 5. Summary Takeaways

- Bash by default continues running after errors, leading to cascading failures.
- Always guard destructive operations with `cd /path && rm -rf *`.
- Use `|| exit 1` for immediate short-circuiting on critical steps.
- Exclude expected failures under strict mode using `|| true` or `if` blocks.
