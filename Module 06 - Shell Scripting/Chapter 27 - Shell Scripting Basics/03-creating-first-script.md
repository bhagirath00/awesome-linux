# Chapter 27: Shell Scripting Basics - Creating and Executing Your First Script

## 1. Introduction: The Anatomy of a Production Script

Creating a shell script involves more than typing a few commands into a file. A production-grade script incorporates:
- A proper interpreter **Shebang** line (`#!/bin/bash`).
- Script header metadata and documentation comments.
- Proper POSIX execution permissions (`chmod +x`).
- Error-handling directives (`set -euo pipefail`).
- Meaningful exit status codes (`exit 0`).

---

## 2. Step-by-Step Walkthrough

### Step 1: Create the Script File
Open a new file named `server-health.sh` using your preferred text editor:

```bash
vim server-health.sh
```

### Step 2: Write the Script Content
```bash
#!/usr/bin/env bash
# ==============================================================================
# Script Name   : server-health.sh
# Description   : Captures high-level system resource metrics for DevOps audit.
# Author        : DevOps Engineering Team
# Version       : 1.0.0
# ==============================================================================

# Fail immediately if a command errors, variable is unset, or pipe fails
set -euo pipefail

echo "=========================================="
echo "          SYSTEM HEALTH REPORT            "
echo "=========================================="
echo "Date / Time   : $(date '+%Y-%m-%d %H:%M:%S %Z')"
echo "Hostname      : $(hostname -f 2>/dev/null || hostname)"
echo "Kernel Release: $(uname -r)"
echo "Uptime        : $(uptime -p)"
echo "------------------------------------------"

echo "CPU Load Averages (1m, 5m, 15m):"
cut -d' ' -f1-3 /proc/loadavg

echo "------------------------------------------"
echo "Memory Utilization:"
free -h | awk 'NR==1 || NR==2'

echo "------------------------------------------"
echo "Root Disk Usage:"
df -h / | awk 'NR==1 || NR==2'

echo "=========================================="
echo "Health check completed successfully."
exit 0
```

---

## 3. Granting Execution Permissions (`chmod +x`)

When a new file is created in Linux, the default system `umask` (typically `0022` or `0002`) creates the file with mode `644` (`-rw-r--r--`). 

If you attempt to execute it directly:
```bash
./server-health.sh
```
```
-bash: ./server-health.sh: Permission denied
```

To execute a script as a standalone program, the **Execution bit (`x`)** must be granted:

```bash
# Grant execution permissions to owner, group, and others
chmod +x server-health.sh

# Or grant execution strictly to the file owner (more secure)
chmod 755 server-health.sh
```

Verify with `ls -l`:
```bash
ls -l server-health.sh
```
```
-rwxr-xr-x 1 devops devops 1240 Oct 14 06:15 server-health.sh
```

---

## 4. The Three Methods of Executing a Script

| Method | Syntax | Requires `chmod +x`? | Execution Context |
| :--- | :--- | :--- | :--- |
| **Direct Path Execution** | `./server-health.sh`<br>`/opt/scripts/server-health.sh` | **YES** | Spawns a **child subshell** defined by the Shebang line. |
| **Interpreter Invocation** | `bash server-health.sh` | **NO** (Only read permissions needed) | Spawns a child `bash` subshell, ignoring the Shebang. |
| **Sourcing into Current Shell** | `source ./server-health.sh`<br>`. ./server-health.sh` | **NO** | Executes directly inside **current parent shell** (modifies your active environment). |

### Output of Running the Script:
```bash
./server-health.sh
```

```
==========================================
          SYSTEM HEALTH REPORT            
==========================================
Date / Time   : 2026-10-14 06:15:30 UTC
Hostname      : web-prod-01.company.internal
Kernel Release: 6.5.0-28-generic
Uptime        : up 14 days, 3 hours, 22 minutes
------------------------------------------
CPU Load Averages (1m, 5m, 15m):
0.24 0.18 0.09
------------------------------------------
Memory Utilization:
               total        used        free      shared  buff/cache   available
Mem:           7.7Gi       2.4Gi       1.8Gi       180Mi       3.5Gi       5.0Gi
------------------------------------------
Root Disk Usage:
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        49G   18G   29G  39% /
==========================================
Health check completed successfully.
```

---

## 5. Adding Scripts to System PATH

To execute your custom script from anywhere without typing `./` or the full path:

```bash
# Option 1: Move to standard user local binary directory
sudo cp server-health.sh /usr/local/bin/server-health

# Option 2: Add personal bin folder to PATH in ~/.bashrc
mkdir -p ~/bin
cp server-health.sh ~/bin/server-health
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

Now, typing `server-health` from any directory runs the script instantly!

---

## 6. Summary Takeaways

- Start every script with a Shebang (`#!/usr/bin/env bash`) and `set -euo pipefail`.
- Set execute permissions using `chmod +x <script.sh>`.
- Use `./script.sh` to run in a subshell; use `source` only when you intend to modify the active shell's environment variables.
- Place shared utilities in `/usr/local/bin` to make them globally accessible across the system.
