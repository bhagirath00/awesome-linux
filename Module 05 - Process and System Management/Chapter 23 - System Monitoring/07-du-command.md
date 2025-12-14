# Chapter 23: System Monitoring - The du Command (Disk Usage Analysis)

## 1. Introduction: What is the du Command?

While `df` displays high-level space availability across entire mounted filesystems, `du` (**Disk Usage**) estimates file space usage recursively down directory trees.

When `df` indicates that `/var` or `/data` is 95% full, `du` is the tactical diagnostic tool you employ to traverse the directory structure, identify bloated folders, and pinpoint rogue log files, core dumps, or abandoned Docker layers.

---

## 2. Command Output and Essential Flags

**Basic Syntax:**
```bash
du [options] [path...]
```

```bash
# Check the total size of current directory:
du -sh .
```
- `-s`: **Summary mode** (displays only the total for the specified argument, suppressing subdirectories).
- `-h`: **Human-readable** units (KB, MB, GB, TB).

```
14G	.
```

---

## 3. The Power of `--max-depth` (or `-d`)

Running plain `du` on a large directory recursively prints thousands of lines for every single nested folder. To inspect disk usage cleanly one tier at a time, use `--max-depth`:

```bash
sudo du -h --max-depth=1 /var | sort -hr
```

```
14G	/var
9.2G	/var/log
3.1G	/var/lib
1.4G	/var/cache
120M	/var/spool
4.0K	/var/tmp
```

By setting `--max-depth=1`, you immediately see that `/var/log` is consuming `9.2G`. You can then step inside `/var/log` to isolate the culprit:

```bash
sudo du -h --max-depth=1 /var/log | sort -hr | head -n 5
```

```
9.2G	/var/log
6.4G	/var/log/journal
1.8G	/var/log/nginx
820M	/var/log/syslog
```

---

## 4. Finding the Largest Files and Directories

Combine `du` with `sort` and `find` for high-precision cleanup operations:

### Top 10 Largest Directories on the Filesystem
```bash
sudo du -xh / | sort -rh | head -n 10
```
*(Note: `-x` or `--one-file-system` prevents traversing into separate mounted partitions like `/data` or network NFS shares).*

### Finding Individual Files Larger than 500MB
```bash
sudo find /var/log -type f -size +500M -exec du -h {} + | sort -rh
```

```
1.4G	/var/log/nginx/access.log
850M	/var/log/app-debug.log
```

---

## 5. The Mysterious Discrepancy: `df` vs. `du`

A frequent production mystery in Linux operations:
```
"df -h says /var is 100% full (0 bytes free), but du -sh /var only finds 10GB of files on a 50GB disk! Where did 40GB go?!"
```

### The Root Cause: Deleted Open Files
When you delete a file (`rm /var/log/app.log`) while a running process (e.g., Java, Python, or Nginx) still holds an **open file descriptor** to it:
- The directory link is removed, so `du` can no longer see the filename.
- However, the kernel **cannot free the underlying disk blocks** until the process closes the file handle or terminates!
- `df` reads the raw filesystem block bitmap and correctly reports the space as still consumed!

### Investigating Deleted Open Files:
```bash
sudo lsof +L1
```
*(Lists all open files with an inode link count of 0).*

```
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF NLINK NODE NAME
java    14205 root    4w   REG   8,1 42949672960     0 1052 /var/log/app.log (deleted)
```

### The Solution:
Truncate the file descriptor directly or reload/restart the holding service:
```bash
# Truncate the file descriptor in /proc without restarting process:
> /proc/14205/fd/4

# Or restart the service:
sudo systemctl restart my-app.service
```
Immediately, `df -h` will reclaim the 40GB of storage!

---

## 6. Summary Takeaways

- Use `du -sh <path>` for quick directory totals.
- Use `du -h --max-depth=1 <path> | sort -hr` to drill down into bloated folders layer by layer.
- Use `-x` (`--one-file-system`) to prevent traversing across mount boundaries.
- When `df` and `du` disagree, search for deleted open files using `lsof +L1`.
