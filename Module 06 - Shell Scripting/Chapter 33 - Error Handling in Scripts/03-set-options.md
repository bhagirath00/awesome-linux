# Chapter 33: Error Handling in Scripts - Strict Mode with set -euo pipefail

## 1. Introduction: The Unofficial Bash Strict Mode

The single most impactful line of code you can add to any production Bash script is:

```bash
set -euo pipefail
```

Popularized by Aaron Maxwell as the **"Unofficial Bash Strict Mode"**, this configuration transforms Bash from a permissive, failure-ignoring interpreter into a strict, fail-fast programming language.

```
+--------------------------------------------------------------------------+
|                  The "set -euo pipefail" Safety Suite                    |
+--------------------------------------------------------------------------+
|  -e (errexit)   ===> Exit immediately if any command returns non-zero.   |
|  -u (nounset)   ===> Exit immediately if any variable is uninitialized.  |
|  -o pipefail    ===> Fail pipeline if ANY command in the pipe fails.     |
+--------------------------------------------------------------------------+
```

---

## 2. Deconstructing Each Option

### 1. `set -e` (`errexit`)
Instructs Bash to exit immediately if any simple command fails (returns a non-zero exit status).

```bash
#!/usr/bin/env bash
set -e

cp /nonexistent/file.txt /tmp/
# SCRIPT HALTS HERE! The following destructive command is NEVER executed:
echo "Deployment continuing..."
```

### 2. `set -u` (`nounset`)
Treats uninitialized/unset variables as fatal errors and exits immediately:

```bash
#!/usr/bin/env bash
set -u

# Typo in variable name:
TARGET_DIR="/opt/app"
rm -rf "${TARGT_DIR}"/*
```
```
line 5: TARGT_DIR: unbound variable
```
The script aborts instantly, saving the server from disastrous deletions caused by typos!

#### Handling Intentionally Optional Variables under `-u`:
If a variable is optional, use parameter expansion to provide a default fallback so `-u` does not trigger:
```bash
# Fallback to empty string if unset:
OPTIONAL_FLAG="${DEBUG_MODE:-}"
```

### 3. `set -o pipefail`
By default, the exit status of a pipeline (`cmd1 | cmd2 | cmd3`) is determined **strictly by the last command** (`cmd3`). If `cmd1` crashes, but `cmd3` succeeds, Bash considers the whole pipeline a success!

```bash
# Without pipefail:
failing_command | grep "pattern"
echo "Exit code: $?"  # Returns 0! The failure of cmd1 was completely masked!

# With set -o pipefail:
set -o pipefail
failing_command | grep "pattern"
echo "Exit code: $?"  # Returns non-zero! Catches failures in any stage of the pipe!
```

---

## 4. Other Useful `set` Options

| Flag | Meaning | Use Case |
| :--- | :--- | :--- |
| **`set -x`** (`xtrace`) | Prints every command to `stderr` with a `+` prefix before running. | Debugging and execution tracing. |
| **`set -v`** (`verbose`) | Prints input lines as they are read. | Script parsing diagnostics. |
| **`set -f`** (`noglob`) | Disables filename globbing expansion (`*`, `?`). | Processing user input containing literal asterisks. |

---

## 5. Summary Takeaways

- Every production script should start with `set -euo pipefail`.
- `-e` halts execution on command errors.
- `-u` treats typos in variable names as fatal errors.
- `-o pipefail` catches failures hidden within piped commands.
- Provide fallbacks (`"${VAR:-}"`) for variables intended to be optional under `-u`.
