# The `find` Command in Linux: Fundamentals & Search Criteria

## Overview

The `find` utility is the most powerful search tool in the Linux operating system. Unlike index-based lookup tools, `find` traverses the live filesystem hierarchy in real time, inspecting raw inode metadata to locate files and directories matching precise operational criteria.

In DevOps and cloud operations, `find` is the core engine used for:
- Identifying orphaned files and zombie container layers consuming disk space.
- Locating logs older than compliance retention policies (e.g., 30 or 90 days).
- Security auditing: discovering world-writable files, insecure SUID binaries, and unowned files.
- Automated cleanup cron jobs and maintenance scripts.

---

## Command Syntax and Anatomy

```bash
find [STARTING_PATHS...] [TESTS & CRITERIA] [ACTIONS]
```

If no starting path is provided, `find` defaults to the current working directory (`.`). If no action is specified, `find` prints matches to standard output (`-print`).

---

## 1. Searching by Name and Case Sensitivity

| Test | Description | Example |
| :--- | :--- | :--- |
| `-name PATTERN` | Matches file name case-sensitively. Supports globbing (`*`, `?`, `[]`). | `find /etc -name "*.conf"` |
| `-iname PATTERN` | Matches file name **case-insensitively**. | `find /app -iname "readme*"` |
| `-path PATTERN` | Matches against the full relative or absolute path. | `find . -path "*/tests/*.py"` |
| `-regex PATTERN`| Matches full path against regular expressions. | `find /var/log -regex ".*-[0-9]\{8\}\.gz"` |

> [!WARNING]
> **Always quote search patterns!**
> If you write `find . -name *.log` without quotes, the bash shell expands `*.log` *before* passing arguments to `find`. If multiple `.log` files exist in the current directory, `find` receives syntax errors. Always use quotes:
> ```bash
> find /var/log -name "*.log"
> ```

---

## 2. Searching by File Type (`-type`)

Filter results to specific filesystem object types:

| Flag | Object Type | DevOps Example |
| :--- | :--- | :--- |
| `-type f` | Regular file | `find /var/log -type f -name "*.log"` |
| `-type d` | Directory | `find /opt -type d -name "cache"` |
| `-type l` | Symbolic link | `find /usr/bin -type l` (Audit symlinks) |
| `-type s` | Unix domain socket | `find /run -type s` (Find Docker/systemd sockets) |
| `-type p` | Named pipe (FIFO) | `find /tmp -type p` |
| `-type b`, `-c` | Block device, Character device | `find /dev -type b` (Disks & partitions) |

---

## 3. Searching by File Size (`-size`)

The `-size` parameter supports size multipliers:
- `c` : Bytes
- `k` : Kilobytes (1024 bytes)
- `M` : Megabytes (1024 KiB)
- `G` : Gigabytes (1024 MiB)

Prefixes control numerical comparison:
- `+N` : Strictly **greater than** `N`.
- `-N` : Strictly **less than** `N`.
- `N`  : Exactly `N`.

```bash
# Find files strictly larger than 500 Megabytes:
find /var/log -type f -size +500M

# Find empty (0-byte) files:
find /tmp -type f -empty

# Find empty directories:
find /var/data -type d -empty
```

---

## 4. Searching by Timestamps (`mtime`, `atime`, `ctime`)

Linux tracks three distinct timestamps on every inode:
- **`mtime` (Modification time)**: When file **content** was last written.
- **`atime` (Access time)**: When the file was last **read**.
- **`ctime` (Change time)**: When file **metadata/inode** (permissions, ownership) changed.

### Time Intervals: Days vs Minutes
- `-mtime`, `-atime`, `-ctime`: Measured in **24-hour days**.
- `-mmin`, `-amin`, `-cmin`: Measured in **minutes**.

```text
                  -mtime -7                     -mtime +7
      [ Last 7 Days (Recent) ]          [ Older than 7 Days ago ]
 ──────────────────────────────────▲──────────────────────────────────
                               7 Days Ago
```

### Practical Timestamp Recipes

```bash
# Find files modified within the last 60 minutes (debugging a recent outage):
find /var/log -type f -mmin -60

# Find log archives older than 30 days (log retention purge candidates):
find /var/log/nginx -type f -name "*.gz" -mtime +30

# Find files modified within the last 24 hours:
find /etc -type f -mtime 0
```

---

## 5. Searching by Permissions and Ownership

### By Owner and Group
```bash
# Find files owned by a specific user:
find /var/www -user www-data

# Find orphaned files where the UID has no matching user in /etc/passwd:
# (Common when Docker containers mount host volumes or deleted accounts leave remnants)
find /home -nouser -o -nogroup
```

### By POSIX Permissions (`-perm`)
- `-perm 644` : Exact match (`-rw-r--r--`).
- `-perm -400` : File must have **at least** read permission for owner.
- `-perm /222` : File is writable by **anyone** (owner, group, or other).

### Security Auditing Recipes
```bash
# Find dangerous world-writable files on the host:
find / -xdev -type f -perm -0002

# Find SUID binaries (executes with owner/root permissions):
find / -xdev -perm /4000 -type f
```

---

## 6. Controlling Search Depth: `-maxdepth` and `-mindepth`

By default, `find` recurses deeply into all subdirectories. In massive storage trees (e.g., millions of user uploads), this causes high disk I/O. Use depth controls to bound the search:

```bash
# Search only in the immediate directory without descending into subfolders:
find /var/log -maxdepth 1 -type f -name "*.log"

# Search subdirectories, ignoring top-level files:
find /app -mindepth 2 -type f
```

> [!IMPORTANT]
> Always place `-maxdepth` and `-mindepth` as the **first parameters** immediately after the starting directory path to avoid performance penalties.

---

## 7. Combining Criteria with Boolean Logic

- **AND** (default, or `-a`): Both conditions must be met.
- **OR** (`-o`): Either condition matches.
- **NOT** (`!` or `-not`): Inverts the match condition.

```bash
# Find files that are either .yaml OR .yml:
find /app/k8s -type f \( -name "*.yaml" -o -name "*.yml" \)

# Find all files in /var/log except .gz archives:
find /var/log -type f ! -name "*.gz"
```
*(Parentheses must be escaped `\(` and `\)` so the shell does not interpret them as subshells).*

---

## Summary

- `find` traverses live filesystem inodes to locate files matching specific criteria.
- Always quote glob patterns (`-name "*.log"`) to prevent shell expansion bugs.
- Filter by size (`-size +100M`), type (`-type f`, `-type d`), and age (`-mtime +30`, `-mmin -60`).
- Limit filesystem traversal overhead using `-maxdepth 1`.
- Combine expressions with boolean operators `-o` (OR) and `!` (NOT).
