# Moving Files and Directories in Linux (`mv`)

## Overview

The `mv` (move) command in Linux serves two primary purposes:
1. **Relocating** files or directories to a different folder path.
2. **Renaming** files or directories within the same directory tree.

In modern DevOps and systems architecture, `mv` is not just a daily convenience utility—it provides **POSIX atomic file operations**. This atomic guarantee is the technical foundation for zero-downtime deployment strategies, safe secret writing, and log file rotation.

---

## Command Syntax and Anatomy

```bash
# Rename file or move to new location
mv [OPTIONS] SOURCE DESTINATION

# Move multiple sources into a target directory
mv [OPTIONS] SOURCE1 SOURCE2 SOURCE3 TARGET_DIR/
```

---

## Essential `mv` Command Flags

| Flag | Long Option | Description | DevOps Practical Use Case |
| :--- | :--- | :--- | :--- |
| `-i` | `--interactive` | Prompts before overwriting an existing destination file. | Preventing catastrophic overwriting of production configs during manual maintenance. |
| `-f` | `--force` | Overwrites destination files without prompting (overrides `-i`). | Ensuring automated CI/CD and deployment scripts never hang waiting for stdin. |
| `-n` | `--no-clobber` | Does not overwrite an existing file; fails silently without error. | Safe migration batches where existing production assets must not be clobbered. |
| `-u` | `--update` | Moves only when SOURCE is newer than DESTINATION, or DESTINATION is missing. | Synchronizing modified artifacts into staging trees. |
| `-v` | `--verbose` | Explains what is being done line by line. | Verbose audit trails in continuous integration build runners. |
| `-b` | `--backup` | Makes a backup copy of any destination file that would be overwritten. | Safety net when rotating configuration templates. |

---

## 1. How `mv` Works Under the Hood

The internal behavior of `mv` depends completely on whether the source and destination reside on the **same filesystem** or **different filesystems**.

### Scenario A: Same Filesystem (Instant & Atomic)
When moving a file within the same disk partition (e.g., from `/var/log/app/incoming/data.csv` to `/var/log/app/processed/data.csv`):
- The physical storage blocks on the disk **do not move**.
- The file's inode number **remains identical**.
- Linux simply updates directory pointers: it unlinks the name from the old directory and links it to the new directory.
- **Execution Time**: Less than 1 millisecond, whether the file is 10 bytes or 500 Gigabytes!

```text
Before Move:
Directory /incoming  ──>  [ Inode #582910 ] ──> [ Physical Disk Blocks: 50 GB Data ]
Directory /processed ──>  (Empty)

After 'mv /incoming/data.csv /processed/':
Directory /incoming  ──>  (Unlinked)
Directory /processed ──>  [ Inode #582910 ] ──> [ Physical Disk Blocks: 50 GB Data ]
```

### Scenario B: Across Filesystems (Copy + Delete)
When moving across different partitions or mount points (e.g., from a local SSD `/tmp/backup.tar.gz` to an NFS mount `/mnt/nfs/backup.tar.gz`):
1. `mv` reads the entire contents of the source and writes them to the destination partition.
2. `mv` verifies the copy succeeded.
3. `mv` deletes the original file from the source partition.
4. **Execution Time**: Proportional to file size and network/disk write speed.

> [!WARNING]
> Moving across filesystems is **NOT atomic**. If the server is powered off or the process is killed midway through a cross-filesystem move, the destination file may be left partially written.

---

## 2. The DevOps Secret Weapon: Atomic File Replacement

In high-concurrency production systems (web servers like Nginx, databases like Redis, or microservices), updating a configuration file or application code in-place can cause race conditions:
- If a service reads `config.json` while your deployment script is halfway through writing it, the service will crash on invalid JSON syntax!

### The Atomic Swap Pattern
POSIX guarantees that `rename()` (invoked by `mv` on the same filesystem) is **atomic**. A reading process will either read the old complete file or the new complete file—it will never read a partial, corrupted file.

```bash
#!/usr/bin/env bash
set -euo pipefail

DEST_CONF="/etc/nginx/conf.d/api.conf"
TMP_CONF="/etc/nginx/conf.d/api.conf.tmp.$$"

# Step 1: Write and render the new configuration to a temporary file on the SAME filesystem
render_template > "$TMP_CONF"

# Step 2: Validate the configuration syntax before touching production
nginx -t -c "$TMP_CONF"

# Step 3: Atomic swap! Nginx never sees a partially written file
mv -f "$TMP_CONF" "$DEST_CONF"

# Step 4: Gracefully reload the service
systemctl reload nginx
```

---

## 3. Practical DevOps Moving Operations

### Moving Multiple Specific Files
```bash
mv api.log auth.log worker.log /var/log/archived/
```

### Moving with Pattern Matching (Globbing)
```bash
# Move all compressed log archives older than today to the backup directory
mv /var/log/nginx/*.gz /mnt/backups/nginx/
```

### Preventing Accidental Overwrites in Scripts
```bash
# If /backup/report.pdf already exists, do not overwrite it
mv -n report.pdf /backup/
```

### Creating Automatic Backups Before Moving
```bash
$ mv -b --suffix=.orig new_settings.env settings.env
$ ls -l settings.env*
-rw-r--r-- 1 root root 1200 Oct 14 09:45 settings.env
-rw-r--r-- 1 root root 1050 Oct 10 14:20 settings.env.orig
```

---

## Summary

- `mv` renames files or moves them across directory paths.
- On the same filesystem, `mv` is **instantaneous and atomic**, modifying only filesystem directory entries without touching disk blocks.
- Across different filesystems or mount points, `mv` executes a copy followed by a deletion.
- Always use the **Atomic Swap Pattern** (`write to .tmp` -> `mv -f .tmp production`) for mission-critical configuration and binary updates.
