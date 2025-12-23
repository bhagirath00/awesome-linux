# Chapter 31: Input and Output - Output Formatting with echo and printf

## 1. Introduction: echo vs. printf

Printing text to standard output is the most fundamental operation in shell scripting. While almost every beginner reaches for **`echo`**, enterprise-grade scripting heavily favors **`printf`**.

### The Problem with `echo`: Inconsistency
The behavior of `echo` differs wildly between different shells and Unix operating systems:
- In GNU Bash, `echo` supports flags like `-e` (enable escape characters) and `-n` (omit trailing newline).
- On macOS, BSD, Dash, or POSIX `sh`, `echo -e` may literally print the characters `-e` on the screen!
- Backslash escape sequences (like `\n` or `\t`) may or may not be evaluated depending on whether the shell has `xpg_echo` enabled.

For consistent, robust, and portable script output across all Linux and Unix distributions, **`printf` is the industry gold standard**.

```
+--------------------------------------------------------------------------+
|                            echo vs. printf                               |
+--------------------------------------------------------------------------+
|  Feature                      echo                    printf             |
+--------------------------------------------------------------------------+
|  POSIX Portability            Inconsistent            100% Consistent    |
|  Trailing Newline             Automatic               Manual via \n      |
|  Column / Width Formatting    None                    Full padding (%-20s|
|  Data Type Specifiers         None                    %s, %d, %f, %x     |
|  Color Escapes                Requires -e             Native support     |
+--------------------------------------------------------------------------+
```

---

## 2. Using echo in Bash

When you do use `echo` in modern Bash:

```bash
# Basic string output (automatically appends a newline)
echo "Operation completed successfully."

# Suppress trailing newline with -n (ideal for interactive input prompts)
echo -n "Enter deployment environment: "

# Enable backslash interpretation with -e
echo -e "Tier 1: Web\nTier 2: API\nTier 3: Database"

# Outputting tabs (\t)
echo -e "STATUS\tSERVER\tUPTIME"
```

---

## 3. Mastering printf: Syntax and Format Specifiers

The `printf` built-in models the classic C `printf()` library function:

**Syntax:**
```bash
printf <format_string> [arguments...]
```

> [!IMPORTANT]
> `printf` does **NOT** append a trailing newline automatically! You must explicitly append **`\n`** wherever you want a line break.

### Common Format Specifiers:
| Specifier | Data Interpretation | Example |
| :--- | :--- | :--- |
| **`%s`** | String of characters | `printf "User: %s\n" "alice"` |
| **`%d`** or **`%i`**| Signed Decimal Integer | `printf "Replicas: %d\n" 5` |
| **`%f`** | Floating-point number | `printf "Ratio: %.2f%%\n" 98.452` |
| **`%x`** / **`%X`** | Hexadecimal representation | `printf "Hex: 0x%X\n" 255` (0xFF) |
| **`%%`** | Literal percent sign | `printf "Discount: %d%%\n" 10` |

---

## 4. Aligning Columns and Building Professional Tables

`printf` allows you to specify exact field widths, padding, and alignments:

- **`%20s`**: Right-align string within a 20-character wide field.
- **`%-20s`**: **Left-align** string within a 20-character wide field (standard for tables).

```bash
#!/usr/bin/env bash

# Print Table Header
printf "%-15s %-12s %-10s %-8s\n" "SERVICE" "STATUS" "PORT" "UPTIME"
printf "%s\n" "--------------------------------------------------"

# Print Table Rows
printf "%-15s %-12s %-10s %-8s\n" "nginx" "RUNNING" "80,443" "14d"
printf "%-15s %-12s %-10s %-8s\n" "postgresql" "RUNNING" "5432" "42d"
printf "%-15s %-12s %-10s %-8s\n" "redis-cache" "DEGRADED" "6379" "2h"
```

### Execution Output:
```
SERVICE         STATUS       PORT       UPTIME  
--------------------------------------------------
nginx           RUNNING      80,443     14d     
postgresql      RUNNING      5432       42d     
redis-cache     DEGRADED     6379       2h      
```

---

## 5. Adding Colors and Styles in Terminal Output

Use standard ANSI escape sequences with `printf` to provide visual feedback:

```bash
# Define ANSI color constants
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[0;33m'
BLUE='\033[0;34m'
BOLD='\033[1m'
NC='\033[0m' # No Color / Reset

printf "${GREEN}[SUCCESS]${NC} Server provisioned in ${BOLD}14.2s${NC}\n"
printf "${YELLOW}[WARNING]${NC} Disk space above 85%%\n"
printf "${RED}[FAILED]${NC}  Connection refused to database!\n"
```

---

## 6. Assigning Formatted Output to a Variable (`-v`)

Instead of spawning a slow subshell `VAR=$(printf ...)`, modern Bash allows `printf` to populate a variable directly using the **`-v`** flag:

```bash
# Stores output directly into $FORMATTED_DATE without subshell overhead:
printf -v FORMATTED_DATE "%(%Y-%m-%d %H:%M:%S)T" -1

echo "Current Timestamp: ${FORMATTED_DATE}"
```

---

## 7. Summary Takeaways

- Use `printf` instead of `echo` for portable, reliable script output.
- Remember to include explicit `\n` linebreaks with `printf`.
- Align table columns with `%-20s` (left-aligned) or `%20s` (right-aligned).
- Use `printf -v <variable>` to assign formatted text directly without subshell overhead.
