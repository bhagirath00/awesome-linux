# Production Best Practices & Optimization for `find`

## Overview

In enterprise environments, running an unconstrained `find` command as root against the root directory (`/`) can trigger production incidents:
- Accidental traversal into slow, high-latency NFS or CIFS network storage mounts.
- Endless loops caused by circular symbolic links.
- High disk I/O saturation that starves production databases (PostgreSQL, MySQL) of storage bandwidth.
- Unintended execution of `-delete` across unverified file paths.

Adhering to production best practices ensures your automated maintenance jobs, security scans, and operational scripts execute safely, predictably, and with minimal system overhead.

---

## 1. Preventing Mount Point Traversal: `-xdev`

When searching from root (`find / ...`), by default `find` will cross mount boundaries into:
- Remote network shares (`/mnt/nfs`, `/mnt/s3fs`)
- External USB backup drives
- Ephemeral virtual pseudo-filesystems (`/proc`, `/sys`)
- Container overlay filesystems (`/var/lib/docker/overlay2`)

Searching across a multi-terabyte network mount can freeze your terminal and exhaust network bandwidth.

### The Solution: `-xdev` (or `-mount`)
The `-xdev` flag restricts the search strictly to the **same filesystem** as the starting directory:

```bash
# Safely audit the root filesystem without descending into network mounts or /proc:
find / -xdev -type f -perm /4000
```

> [!IMPORTANT]
> **Production Standard**: Always append `-xdev` whenever executing system-wide searches starting from `/`.

---

## 2. Pruning Directories: Skipping Heavy Trees with `-prune`

Often you need to search an entire repository or application docroot while explicitly skipping large directories like `.git`, `node_modules`, or `vendor`.

The `-prune` action instructs `find` **not to descend** into matching directories.

### Syntax Pattern
```bash
find START_DIR \( -name "DIR_TO_SKIP" -prune \) -o \( CRITERIA -print \)
```

### Practical Example: Search Code while Skipping `node_modules` and `.git`
```bash
find . \( -name "node_modules" -o -name ".git" \) -prune -o -type f -name "*.js" -print
```

### Visualizing `-prune` Execution Flow
```text
Is directory named "node_modules" or ".git"?
  ├── YES ──> -prune (Stops descending into this entire tree immediately!)
  └── NO  ──> Evaluates right side of -o: Is it a *.js file? Print it!
```

---

## 3. Controlling I/O and CPU Priority: `nice` and `ionice`

Automated maintenance cron jobs (such as nightly log cleanups or integrity checks) should never compete with production database queries for storage I/O bandwidth.

Use `nice` (CPU scheduling priority) and `ionice` (I/O scheduling class):

```bash
# Run find with lowest CPU priority (19) and idle I/O priority (class 3):
nice -n 19 ionice -c 3 find /var/log -type f -mtime +30 -delete
```

- **`nice -n 19`**: Guarantees `find` only receives CPU cycles when the system is otherwise idle.
- **`ionice -c 3`**: Ensures `find` will only perform disk reads and writes when no other process is requesting disk access.

---

## 4. Production Cron Maintenance Script Template

Here is a hardened, production-ready Bash script template for scheduled log cleanup incorporating locking, logging, error handling, and safe deletion:

```bash
#!/usr/bin/env bash
# ==============================================================================
# Script: /opt/scripts/cleanup-logs.sh
# Purpose: Idempotent, safe log retention cleanup with locking and logging
# ==============================================================================
set -euo pipefail

LOG_DIR="/var/log/custom-apps"
RETENTION_DAYS=14
LOCK_FILE="/var/run/cleanup-logs.lock"

# Step 1: Prevent overlapping concurrent execution using file locking (flock)
exec 200>"$LOCK_FILE"
flock -n 200 || { echo "Cleanup job already running. Exiting." >&2; exit 0; }

# Step 2: Validate target directory exists before running find
if [ ! -d "$LOG_DIR" ]; then
    echo "Directory $LOG_DIR does not exist. Aborting." >&2
    exit 1
fi

echo "[$(date '+%Y-%m-%d %H:%M:%S')] Starting automated log cleanup in $LOG_DIR..."

# Step 3: Execute safe deletion with depth limits and I/O throttling
nice -n 19 ionice -c 3 find "$LOG_DIR" \
    -mindepth 1 \
    -type f \
    -name "*.log.gz" \
    -mtime +"$RETENTION_DAYS" \
    -exec rm -f {} +

echo "[$(date '+%Y-%m-%d %H:%M:%S')] Cleanup completed successfully."
```

---

## 5. Troubleshooting Checklist: Common `find` Gotchas

| Mistake | Consequence | Correct Technique |
| :--- | :--- | :--- |
| Forgetting quotes: `find . -name *.log` | Shell expands `*.log` before `find` runs; errors if >1 file exists. | Always quote patterns: `find . -name "*.log"` |
| Placing `-delete` at the start of the command | Deletes everything in directory before filtering. | Always place `-delete` as the **last** parameter. |
| Using `-exec rm {} \;` on 100,000 files | Forks 100,000 separate processes; degrades server performance. | Use `-exec rm {} +` to batch arguments. |
| Omitting `-maxdepth` on massive trees | Excessive recursion into deep directories; unnecessary disk I/O. | Restrict search depth using `-maxdepth N`. |
| Running unrestricted on `/` without `-xdev` | Scans mounted network shares (NFS/EFS) and `/proc`. | Append `-xdev` to isolate searches to the local root filesystem. |

---

## Summary

- Restrict root-level searches to the local partition using `-xdev`.
- Prune massive dependency trees like `node_modules` and `.git` using `( -name "dir" -prune ) -o ...`.
- Throttle background maintenance jobs using `nice -n 19 ionice -c 3` to protect production database latency.
- Protect cron jobs against overlapping runs using `flock`.
- Always verify matches with `-print` before committing to destructive `-delete` or `-exec rm` actions.
