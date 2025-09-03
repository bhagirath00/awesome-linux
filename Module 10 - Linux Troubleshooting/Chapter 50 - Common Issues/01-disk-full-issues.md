# Troubleshooting Disk Full Issues (ENOSPC & Inode Exhaustion)

## 1. Architectural Concept & Deep Dive

The error `No space left on device` (`ENOSPC` errno 28) is one of the most common and disruptive production outages in Linux environments. It causes databases to crash into read-only recovery mode, web servers to fail writing session files, and containers to get evicted by kubelet.

However, a disk outage in Linux is governed by two completely distinct physical structures:
1. **Block Exhaustion**: All storage data blocks on the filesystem are filled by file contents. Measured via `df -h`.
2. **Inode Exhaustion**: The filesystem table has exhausted its allocation of index nodes (inodes), even if gigabytes of free disk space remain! Measured via `df -i`.

```
+─────────────────────────────────────────────────────────────────────────+
|                       FILESYSTEM DISK STRUCTURE                         |
|                                                                         |
|  Superblock: Filesystem metadata, block size, total counts             |
|                                                                         |
|  INODE TABLE (Index Nodes)            DATA BLOCKS (Payload)             |
|  +─────────────────────────────+      +─────────────────────────────+   |
|  | Inode 1: root (/)           |      | Block 001: "hello world..." |   |
|  | Inode 2: /var/log/app.log   |───┐  | Block 002: "2026-10-14..."  |   |
|  | Inode 3: ...                |   │  | Block 003: [Free]           |   |
|  | Inode N: [Exhausted?]       |   │  | Block M:   [Exhausted?]     |   |
|  +─────────────────────────────+   │  +─────────────────────────────+   |
|                                    │                 ▲                  |
|                                    └─────────────────┘                  |
|                               Directs to physical storage blocks        |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Inode Exhaustion vs Block Exhaustion

| Metric | Inspection Tool | Root Cause Pattern | Typical Culprits |
| :--- | :--- | :--- | :--- |
| **Block Exhaustion** | `df -h` | Massive files accumulating on disk. | Unrotated logs, core dumps, uncompressed database exports, Docker images. |
| **Inode Exhaustion** | `df -i` | Millions of tiny micro-files (0-byte to 1KB) exhausting the inode table. | PHP session directories (`/var/lib/php/sessions`), mail queues (`/var/spool/postfix`), cache trees. |

---

## 3. The Phantom Disk Consumer: Unlinked Open Files (`lsof +L1`)

A classic production puzzle:
> `df -h` reports `/var` is 100% full (0 bytes free). You run `du -sh /var` and it reports only 10 GB used on a 100 GB disk. Where are the missing 90 GB?

### The Linux Kernel Mechanism
In POSIX filesystems, calling `rm /var/log/app.log` executes `unlink(2)`. It removes the directory entry from the namespace, but **it does NOT free the disk blocks if any running process still holds an open file descriptor to that inode!**

```
Directory Tree:        rm /var/log/app.log (Unlinks file pointer)
                             │
                             ▼
Kernel VFS:            Task PID 1245 holds Open File Descriptor (fd 3)
                       Inode Reference Count = 1
                             │
                             ▼
Storage Blocks:        NOT FREED! df continues to report 100% full!
```

### Detection and Remediation
```bash
# Locate all deleted files still held open in memory by active processes
sudo lsof +L1
```
```
COMMAND   PID USER   FD   TYPE DEVICE   SIZE/OFF NLINK   NODE NAME
java     1420 root    3w   REG    8,1 8945201408     0 412052 /var/log/app.log (deleted)
```
- Notice `NLINK 0` and size is ~8.9 GB!

**Remediation Options**:
1. **Signal the process to reopen**: `kill -HUP 1420` or `systemctl reload app`.
2. **Truncate the file via procfs**: If the process cannot be reloaded, find its file descriptor and truncate it without restarting:
   ```bash
   # Truncate directly via /proc/<PID>/fd/<FD>
   echo "" > /proc/1420/fd/3
   ```
   *The disk space is freed immediately without interrupting the process!*

---

## 4. Large File Hunting: Production CLI Runbook

### 1. Identify Which Filesystem / Mount Is Full
```bash
df -h
```
```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        98G   98G     0 100% /
/dev/sdb1       492G  120G  348G  26% /data
```

### 2. Check for Inode Exhaustion
```bash
df -i
```
```
Filesystem       Inodes   IUsed   IFree IUse% Mounted on
/dev/sda1       6553600 6553600       0  100% /
```

### 3. Trace Disk Consumption Top-Down with `du`
```bash
# Find top 10 largest directories under root, avoiding crossing filesystems (-x)
sudo du -xh / --max-depth=1 2>/dev/null | sort -rh | head -n 10
```
```
84G     /var
8.2G    /usr
2.1G    /opt
1.2G    /home
```

Descend into `/var`:
```bash
sudo du -xh /var --max-depth=1 2>/dev/null | sort -rh | head -n 10
```
```
72G     /var/lib/docker
10G     /var/log
```

### 4. Locate Files Larger Than 1GB
```bash
sudo find / -xdev -type f -size +1G -exec ls -lh {} + 2>/dev/null | awk '{print $5, $9}'
```

### 5. Locate Directories with Excessive Inode Counts
If `df -i` shows 100% inode usage:
```bash
# Count files per directory in /var
find /var -xdev -type d -exec sh -c 'echo -n "{}: "; find "{}" -maxdepth 1 -type f | wc -l' \; | sort -k2 -nr | head -n 10
```
```
/var/lib/php/sessions: 1250491
/var/spool/postfix/maildrop: 450120
```

---

## 5. Mount Point Overlay Masking

Another elusive issue occurs when a storage volume is mounted over a directory that already contained data:

```bash
# 1. Host writes 20 GB to /mnt/backup while storage is unmounted.
# 2. Administrator mounts external volume to /mnt/backup:
sudo mount /dev/sdb1 /mnt/backup
```
- Once `/mnt/backup` is mounted, the original 20 GB of files are masked and invisible to `du /mnt/backup`.
- However, those 20 GB still consume space on the root (`/`) filesystem!
- **Detection**: Unmount temporarily or inspect via bind mount:
  ```bash
  mkdir /tmp/root_peek
  mount --bind / /tmp/root_peek
  du -sh /tmp/root_peek/mnt/backup
  umount /tmp/root_peek
  ```

---

## 6. Emergency Production Recovery Tactics

When root (`/`) is 100% full, standard commands like `vim`, `nano`, `apt`, and even bash tab-completion will fail with `No space left on device`.

### Emergency Actions:
1. **Reclaim Reserved Root Blocks**:
   Linux ext4 filesystems reserve 5% of blocks for the `root` user to prevent total system deadlock. You can reduce this reserve to 1% to instantly grant administrative headroom:
   ```bash
   sudo tune2fs -m 1 /dev/sda1
   ```
2. **Safely Truncate Known Large Logs (NEVER `rm`)**:
   ```bash
   > /var/log/syslog
   > /var/log/nginx/access.log
   ```
3. **Clean Package Cache**:
   ```bash
   sudo apt-get clean || sudo dnf clean all
   ```
4. **Prune Docker Storage**:
   ```bash
   docker system prune -af --volumes
   ```

---

## 7. Key Takeaways
- Always verify both block usage (`df -h`) and inode usage (`df -i`).
- Deleting an open log file (`rm`) does not free disk space until the holding process closes it; inspect using `lsof +L1` and truncate via `/proc/<PID>/fd/<FD>`.
- Mount point masking can hide files underneath newly mounted partitions; inspect via root bind mounts.
- Use `tune2fs -m 1` to immediately reclaim space during a live incident.
