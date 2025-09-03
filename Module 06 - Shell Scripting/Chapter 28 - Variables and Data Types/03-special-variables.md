# Chapter 28: Variables and Data Types - Bash Special Parameters and Variables

## 1. Introduction: Built-in Shell Parameters

Bash provides a set of pre-defined, read-only special parameters that are automatically populated and updated by the shell runtime. 

These variables provide instant access to:
- Script positional arguments passed via the command line.
- The exit status code of the most recently executed command.
- The Process ID (PID) of the current script or background tasks.
- The total count of passed arguments.

In DevOps automation, mastering these parameters is essential for building flexible CLI tools, CI/CD runners, and error-handling routines.

---

## 2. Master Table of Bash Special Variables

| Special Variable | Description | Typical Value / Use Case |
| :--- | :--- | :--- |
| **`$0`** | Name of the script currently executing (as invoked). | `./deploy.sh` or `/usr/local/bin/deploy` |
| **`$1`, `$2`..`$9`** | Positional arguments passed to the script or function. | `$1` is the first argument, `$2` is the second. |
| **`${10}`, `${11}`**| Positional arguments beyond 9 (**must use curly braces**). | `${10}` is the tenth argument. |
| **`$#`** | **Argument Count**: Total number of positional arguments passed. | Used for input validation (`if [ $# -lt 2 ]`). |
| **`$*`** | All positional arguments as a **single word** ("$1 $2 $3"). | Combined string output. |
| **`$@`** | All positional arguments as **separate quoted words** ("$1" "$2"). | **Gold standard for loop iteration**. |
| **`$?`** | **Exit Status Code**: Exit code of the most recently executed foreground command. | `0` = Success; `1-255` = Failure. |
| **`$$`** | **Process ID (PID)**: The PID of the currently executing script. | Creating unique temp files (`/tmp/out.$$`). |
| **`$!`** | **Background PID**: The PID of the most recently spawned background command (`&`). | Waiting for or killing background tasks. |
| **`$_`** | Last argument of the previous command. | Fast shell navigation shortcut. |

---

## 3. The Critical Difference: `$@` vs. `$*`

When passing arguments down into loops or child commands, confusing `$@` and `$*` is a classic source of production bugs:

```
Consider running: ./script.sh "Hello World" "Production DB"
```

### When Quoted:
- **`"$*"` (Single String)**: Collapses all arguments into **one single string**, delimited by the first character of `IFS` (space):
  `"Hello World Production DB"` (Total: 1 argument).
- **`"$@"` (Array Preservation - PREFERRED)**: Preserves every original argument as an individual, safely quoted element:
  `"Hello World"` and `"Production DB"` (Total: 2 separate arguments).

### The Golden Rule:
Always iterate over `"$@"` with double quotes:
```bash
for arg in "$@"; do
    echo "Processing item: ${arg}"
done
```

---

## 4. Practical Demonstration Script: `args-demo.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "Script Name (\$0)     : ${0}"
echo "Total Arguments (\$#) : ${#}"
echo "Current PID (\$$)     : ${$}"

# Validate required arguments
if [ "$#" -lt 2 ]; then
    echo "Error: Missing required arguments!"
    echo "Usage: ${0} <ENVIRONMENT> <SERVICE_NAME>"
    exit 1
fi

ENV="$1"
SERVICE="$2"

echo "Deploying service '${SERVICE}' to '${ENV}' environment..."

# Run an external command and capture exit code
systemctl is-active --quiet "${SERVICE}" || true
STATUS="$?"

if [ "${STATUS}" -eq 0 ]; then
    echo "Service is currently active."
else
    echo "Service is inactive (Status code: ${STATUS})."
fi
```

Execution:
```bash
./args-demo.sh staging payment-api
```
```
Script Name ($0)     : ./args-demo.sh
Total Arguments ($#) : 2
Current PID ($$)     : 24125
Deploying service 'payment-api' to 'staging' environment...
Service is currently active.
```

---

## 5. Capturing Background PIDs with `$!`

When spawning concurrent background tasks in a script, record their PIDs using `$!` to manage and wait for them:

```bash
# Start background backup
tar -czf /backup/db.tar.gz /var/lib/pgsql &
BACKUP_PID="$!"

echo "Backup running in background under PID: ${BACKUP_PID}..."

# Wait specifically for the backup job to finish
wait "${BACKUP_PID}"
echo "Backup finished with exit code: $?"
```

---

## 6. Summary Takeaways

- `$0` is the script name; `$1` through `${10}` are positional CLI arguments.
- Always validate argument count using `$#`.
- Always iterate over arguments using `"$@"` to preserve spaces within parameters.
- `$?` holds the exit status code of the preceding command (`0` = success).
- Use `$$` for script PID and `$!` to track background processes.
