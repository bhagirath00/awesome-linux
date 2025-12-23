# Chapter 29: Control Flow - If-Else Conditional Statements

## 1. Introduction: Conditional Branching in Bash

Conditional branching allows shell scripts to make decisions dynamically based on:
- Command exit status codes (`$? == 0`).
- File existence, permissions, and types (`-f`, `-d`, `-r`).
- String comparisons (empty, equal, regex matching).
- Integer evaluations (greater than, equal, less than).

In Bash, the `if` statement evaluates the **exit status of a command**. If the command returns an exit status of `0` (Success / True), the `then` block executes. If it returns any non-zero code (`1-255` / False), execution branches to `elif` or `else`.

---

## 2. Test Syntax: `[` (test) vs. `[[` (Bash Extended Test)

Bash supports two testing constructs:

| Feature | Legacy `[` (POSIX test) | Modern `[[` (Bash Keyword) |
| :--- | :--- | :--- |
| **Command Type** | External binary or built-in (`/bin/[`) | Shell language keyword |
| **Word Splitting** | Vulnerable (Unquoted variables crash with syntax errors) | **Safe** (Automatic variable quoting) |
| **Pattern / Regex Matching**| No | **Yes** (`=~` regex, `*` wildcards) |
| **Logical Operators** | `-a` (AND), `-o` (OR) | `&&` (AND), `||` (OR) |
| **Recommendation** | Use only for strict POSIX `#!/bin/sh` | **Universal gold standard for `#!/bin/bash`** |

> [!TIP]
> In all modern Bash scripts (`#!/bin/bash`), **always use `[[ ... ]]`**. It eliminates word-splitting bugs and provides regex support.

---

## 3. Test Operators Reference Table

### 1. File Test Operators
| Operator | True Condition |
| :--- | :--- |
| **`-f "$FILE"`** | Exists and is a **regular file** (not a directory or socket). |
| **`-d "$DIR"`** | Exists and is a **directory**. |
| **`-e "$PATH"`** | Path **exists** (any type: file, dir, symlink). |
| **`-s "$FILE"`** | Exists and has a size **greater than zero bytes** (not empty). |
| **`-r "$FILE"`** | Exists and is **readable** by current user. |
| **`-w "$FILE"`** | Exists and is **writable** by current user. |
| **`-x "$FILE"`** | Exists and is **executable** by current user. |
| **`-L "$FILE"`** | Exists and is a **symbolic link**. |

### 2. String Test Operators
| Operator | True Condition |
| :--- | :--- |
| **`-z "$STR"`** | String is **empty / zero length** (`null` or `""`). |
| **`-n "$STR"`** | String is **not empty / non-zero length**. |
| **`"$A" == "$B"`** | String A is identical to String B. |
| **`"$A" != "$B"`** | String A is not equal to String B. |
| **`"$A" =~ REGEX`**| String A matches the regular expression. |

### 3. Integer Comparison Operators
| Operator | Meaning | Example |
| :--- | :--- | :--- |
| **`-eq`** | Equal to | `[[ "$x" -eq 10 ]]` |
| **`-ne`** | Not equal to | `[[ "$x" -ne 0 ]]` |
| **`-lt`** | Less than | `[[ "$x" -lt 100 ]]` |
| **`-le`** | Less than or equal to | `[[ "$x" -le 5 ]]` |
| **`-gt`** | Greater than | `[[ "$x" -gt 0 ]]` |
| **`-ge`** | Greater than or equal to| `[[ "$x" -ge 1 ]]` |

*(Note: `<` and `>` inside `[[ ]]` perform ASCII alphabetical string sorting, NOT numeric comparison! For numeric math, always use `-gt`, `-lt` or `(( x > 10 ))`).*

---

## 4. Production Script Example: Automated Deployment Health Check

```bash
#!/usr/bin/env bash
set -euo pipefail

CONFIG_FILE="/etc/app/config.json"
PORT="8080"
HEALTH_URL="http://127.0.0.1:${PORT}/health"

# 1. File and Directory Validation
if [[ ! -f "${CONFIG_FILE}" ]]; then
    echo "[ERROR] Configuration file '${CONFIG_FILE}' is missing!" >&2
    exit 1
elif [[ ! -r "${CONFIG_FILE}" ]]; then
    echo "[ERROR] Configuration file exists but is not readable!" >&2
    exit 1
else
    echo "[OK] Configuration file validated."
fi

# 2. String / Regex Validation (Validate PORT is an integer)
if [[ ! "${PORT}" =~ ^[0-9]+$ ]] || [[ "${PORT}" -lt 1024 ]] || [[ "${PORT}" -gt 65535 ]]; then
    echo "[ERROR] Invalid unprivileged port: ${PORT}" >&2
    exit 1
fi

# 3. Direct Command Execution in 'if'
# The 'if' directly evaluates the exit status of curl without needing [[ ]]!
if curl --silent --fail "${HEALTH_URL}" > /dev/null; then
    echo "[SUCCESS] Service healthy at ${HEALTH_URL}."
    exit 0
else
    echo "[CRITICAL] Service failed health check!" >&2
    exit 2
fi
```

---

## 5. Summary Takeaways

- In Bash scripts, prefer `[[ ... ]]` over legacy `[ ... ]`.
- Always check if variables are set with `-z` (empty) or `-n` (not empty).
- Validate file states with `-f` (regular file), `-d` (directory), and `-s` (non-empty).
- Use `-eq`, `-ne`, `-lt`, `-gt` for integer comparisons.
- You can place commands directly in `if` statements (e.g., `if curl ...; then`).
