# Chapter 28: Variables and Data Types - Variables in Bash

## 1. Introduction: Bash Variable Fundamentals

In Bash, a **variable** is a named memory location used to store character data (strings, numbers, file paths, or command outputs). 

Unlike strongly typed compiled programming languages (such as Go, Java, or C++), **Bash is untyped**. Internally, Bash treats all variables as plain ASCII strings of characters. When arithmetic operations are evaluated (`$((a + b))`), Bash dynamically converts the strings to integers, performs the calculation, and converts the result back to a string.

---

## 2. Variable Assignment Rules and Syntax

The golden rule of variable assignment in Bash: **NO SPACES AROUND THE EQUALS SIGN (`=`)!**

```bash
# CORRECT:
SERVER_NAME="web-prod-01"
PORT=8080

# WRONG (Syntax Error):
SERVER_NAME = "web-prod-01"  # Bash thinks 'SERVER_NAME' is a command, and '=' is an argument!
SERVER_NAME= "web-prod-01"   # Bash sets SERVER_NAME to empty and runs "web-prod-01" as a command!
```

### Naming Conventions:
- Variable names can contain letters (`a-z`, `A-Z`), digits (`0-9`), and underscores (`_`).
- Variable names **cannot begin with a digit**.
- Variable names are **case-sensitive** (`$NAME` and `$name` are two distinct variables).
- **DevOps Convention**: Use `UPPERCASE` for global constants and environment variables (`DEPLOY_ENV`, `DB_PORT`), and `lowercase_with_underscores` for internal local script variables (`target_file`, `retry_count`).

---

## 3. Referencing and Expanding Variables

To read or expand the value stored in a variable, prefix its name with a dollar sign (`$`):

```bash
APP_USER="deployer"

# Simple expansion:
echo $APP_USER

# Curly-brace expansion (MANDATORY BEST PRACTICE):
echo "${APP_USER}"
```

### Why Always Use Curly Braces `${VAR}`?
Curly braces clearly delineate the boundary of the variable name from adjacent text:

```bash
FILE_NAME="backup"

# FAILS: Bash searches for a variable named $FILE_NAME_archive
tar -czf "$FILE_NAME_archive.tar.gz" /data

# SUCCEEDS: Clearly isolates $FILE_NAME
tar -czf "${FILE_NAME}_archive.tar.gz" /data
```

---

## 4. Quoting Rules: Single vs. Double vs. Backticks

Understanding quoting in Bash is critical to preventing **word-splitting bugs** and command injection security vulnerabilities:

```
+--------------------------------------------------------------------------+
|                        Bash Quoting Hierarchy                            |
+--------------------------------------------------------------------------+
|  Double Quotes ("...")     Single Quotes ('...')     No Quotes (...)     |
|  - Expands variables ($VAR)- Literal string only     - Subject to word   |
|  - Expands commands ($(..))- Zero expansion          |   splitting and   |
|  - Prevents word splitting - Treats $ literally      |   glob expansion  |
+--------------------------------------------------------------------------+
```

### Practical Demonstration:
```bash
USER="alice"

# Double Quotes: Expands variable
echo "Hello, ${USER}!"
# Output: Hello, alice!

# Single Quotes: Suppresses all expansion
echo 'Hello, ${USER}!'
# Output: Hello, ${USER}!
```

> [!WARNING]
> **The Golden Rule: ALWAYS Double-Quote Your Variables!**
> 
> If a variable contains spaces (e.g., `DIR="/home/my documents"`), unquoted expansion `ls $DIR` causes Bash to split the string into two separate arguments: `ls /home/my` and `documents`, leading to errors or destructive file deletions! 
> 
> **Always write `"${DIR}"`, never `$DIR`**.

---

## 5. Readonly Constants

To prevent a variable from being accidentally reassigned, overwritten, or modified later in the script, declare it as **`readonly`** or use `declare -r`:

```bash
readonly DB_HOST="database.internal.company.com"
declare -r MAX_RETRIES=5

# Attempting to reassign throws an error:
DB_HOST="localhost"
```

```
-bash: DB_HOST: readonly variable
```

---

## 6. Unsetting Variables

To remove a variable from memory:

```bash
TEMP_TOKEN="xyz123secret"
unset TEMP_TOKEN
echo "Token is: '${TEMP_TOKEN}'"
```

```
Token is: ''
```

*(Note: You cannot `unset` a `readonly` variable; it remains immutable until the subshell terminates).*

---

## 7. Summary Takeaways

- Bash is untyped; all variables are treated as strings.
- Never place spaces around the `=` sign during assignment.
- Always wrap variables in curly braces and double quotes: `"${VAR}"`.
- Double quotes expand variables; single quotes treat characters literally.
- Use `readonly VAR="value"` for immutable configuration constants.
