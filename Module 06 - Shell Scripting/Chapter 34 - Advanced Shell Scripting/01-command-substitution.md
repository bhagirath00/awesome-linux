# Chapter 34: Advanced Shell Scripting - Command Substitution

## 1. Introduction: What is Command Substitution?

**Command Substitution** allows the output of a command to be captured, stored in a variable, or passed directly as an argument to another command. 

The shell executes the command inside a subshell, captures whatever the command prints to its standard output (`stdout`), strips any trailing newlines, and substitutes that text into the command line.

```
+--------------------------------------------------------------------------+
|                     Command Substitution Lifecycle                       |
+--------------------------------------------------------------------------+
|  User Command: CURRENT_DATE=$(date +%F)                                  |
|                                                                          |
|  1. Subshell Spawns: /bin/bash subshell created in memory                |
|  2. Executes: date +%F                                                   |
|  3. Captures stdout: "2026-10-14\n"                                      |
|  4. Strips trailing newline: "2026-10-14"                                |
|  5. Variable Assigned: CURRENT_DATE="2026-10-14"                         |
+--------------------------------------------------------------------------+
```

---

## 2. Syntax: Backticks vs. Modern `$()`

Bash supports two syntaxes for command substitution:

| Syntax | Example | Nesting Support | Readability | Recommendation |
| :--- | :--- | :--- | :--- | :--- |
| **Legacy Backticks** | `` `command` `` | Painful (Requires backslash escaping: `` `cmd1 \`cmd2\` ` ``) | Poor (Easily confused with single quotes `'`) | **Deprecated** |
| **Modern Dollar Parentheses** | `$(command)` | **Effortless nesting**: `$(cmd1 $(cmd2))` | Crystal clear | **Universal Gold Standard** |

### The Nesting Advantage of `$()`:
```bash
# Clean, readable nested command substitution:
CONFIG_ARCHIVE="/backup/config_$(basename $(dirname /opt/app/conf/app.yaml))_$(date +%s).tar.gz"
echo "${CONFIG_ARCHIVE}"
```
```
/backup/config_conf_1791984210.tar.gz
```

---

## 3. High-Value Production Use Cases

### 1. Dynamic Version and Commit Tagging in Docker Builds
```bash
GIT_COMMIT=$(git rev-parse --short HEAD 2>/dev/null || echo "unknown")
BUILD_TIMESTAMP=$(date +%Y%m%d%H%M%S)

docker build -t "registry.company.com/app:${GIT_COMMIT}-${BUILD_TIMESTAMP}" .
```

### 2. Dynamically Finding and Processing Files
```bash
TOTAL_CORES=$(grep -c ^processor /proc/cpuinfo)
echo "Parallelizing build across ${TOTAL_CORES} CPU threads..."
make -j"${TOTAL_CORES}"
```

### 3. Sizing Infrastructure Memory Pools
```bash
TOTAL_RAM_MB=$(free -m | awk '/^Mem:/{print $2}')
# Allocate 75% of RAM to Java heap
HEAP_SIZE=$(( TOTAL_RAM_MB * 75 / 100 ))
echo "Configuring JVM Max Heap (-Xmx): ${HEAP_SIZE}m"
```

---

## 4. Performance Caveat: Subshell Creation

Because `$(command)` forks a child subshell process, avoid executing command substitutions inside tight loops running millions of iterations.

When possible, use Bash internal built-in parameter expansions or `read` constructs:
```bash
# Slower (Forks external basename binary):
FILE=$(basename "$PATH")

# Faster (Built-in parameter expansion, zero forks):
FILE="${PATH##*/}"
```

---

## 5. Summary Takeaways

- Command substitution captures a command's standard output.
- Always use `$(...)` instead of legacy backticks `` `...` ``.
- `$()` supports clean, unlimited nesting without backslash escaping.
- Always double-quote command substitutions: `"${(...)}"`, to prevent word splitting.
