# OverlayFS & Container Storage Drivers (overlay2)

## 1. Architectural Concept & Deep Dive

**OverlayFS** is a modern union mount filesystem built directly into the Linux kernel (merged in Linux 3.18). It allows multiple directories to be presented as a single unified directory tree while keeping the underlying layers isolated.

In container ecosystems, OverlayFS powers the **`overlay2`** storage driver used by Docker, containerd, CRI-O, and Podman. It replaces legacy, performance-inhibited storage drivers like `aufs`, `btrfs`, `devicemapper`, and `vfs`.

### The Core Problem OverlayFS Solves
When running 50 containers derived from the same 1 GB Ubuntu image:
- Without a union filesystem, each container would require 1 GB of copied disk space (50 GB total) and duplicate host RAM page cache pages.
- With OverlayFS, all 50 containers share the exact same read-only underlying image layers on disk and in the page cache. Only changes made by a container consume additional disk space.

```
+─────────────────────────────────────────────────────────────+
|               MERGED DIRECTORY (Unified View)               |
|               (What the container process sees)             |
|                                                             |
|       /app/config.json  [from upperdir - Modified]          |
|       /bin/bash         [from lowerdir - Shared Read-Only]  |
|       /etc/hosts        [from upperdir - Container-specific]|
+──────────────────────────────┬──────────────────────────────+
                               │
       ┌───────────────────────┴───────────────────────┐
       ▼                                               ▼
+─────────────────────────────+                 +─────────────────────────────+
|     UPPERDIR (Read-Write)   |                 |    LOWERDIR (Read-Only)     |
|   Container Ephemeral Layer |                 |      Docker Image Layer     |
|                             |                 |                             |
|  • /app/config.json         |                 |  • /bin/bash                |
|  • /etc/hosts               |                 |  • /app/config.json (orig)  |
|  • /deleted_file (whiteout) |                 |  • /deleted_file            |
+─────────────────────────────+                 +─────────────────────────────+
```

---

## 2. The Four Pillars of an OverlayFS Mount

When mounting an OverlayFS filesystem, four directories are required:

| Directory | Role & Permissions | Description |
| :--- | :--- | :--- |
| **`lowerdir`** | Read-Only (RO) | One or more underlying base layers (e.g. base image layers, Ubuntu rootfs, dependencies). Stackable up to 500 layers separated by colons (`:`). |
| **`upperdir`** | Read-Write (RW) | The top layer where all new files, file modifications, and file deletions made by the container are written. |
| **`workdir`** | Kernel Internal (Scratch) | An empty directory on the same filesystem as `upperdir`, used by the kernel to stage changes atomically before exposing them in `merged`. |
| **`merged`** | Mount Point | The final unified directory exposed to the container mount namespace. |

---

## 3. Copy-on-Write (CoW) Mechanics

OverlayFS implements high-performance Copy-on-Write semantics:

### 1. Reading Files
- If a file exists only in `lowerdir`, it is read directly from `lowerdir` (zero copy).
- If a file exists in both `upperdir` and `lowerdir`, the version in `upperdir` hides the lower version and is read.

### 2. Modifying Existing Files (The Copy-Up Penalty)
- When a process opens an existing file from `lowerdir` with write permissions (`O_WRONLY` or `O_RDWR`), OverlayFS triggers a **Copy-Up** operation:
  1. The entire file is copied from `lowerdir` into `upperdir`.
  2. The write operation executes on the newly created copy in `upperdir`.
  3. The original file in `lowerdir` remains completely untouched.
> **DevOps Caveat**: Modifying a 10 GB database file stored in an image layer triggers a full 10 GB copy-up on the first write! Always use **Named Volumes** or **Bind Mounts** for large or write-heavy datasets.

### 3. Deleting Files (Whiteouts & Opaque Directories)
How can a file be deleted if `lowerdir` is strictly read-only?
- OverlayFS creates a **Whiteout Device**: a character device with major and minor numbers set to `0/0` (`mknod c 0 0`) in `upperdir`.
- When OverlayFS sees a whiteout in `upperdir`, it masks and hides the file from `merged`, making it appear deleted.
- If an entire directory from `lowerdir` is deleted and recreated, OverlayFS sets an extended attribute (`trusted.overlay.opaque=y`) to block inheritance from lower layers.

---

## 4. Hands-on Terminal Lab: Building an OverlayFS Mount Manually

Let's build an OverlayFS mount from scratch to observe Copy-on-Write and Whiteouts firsthand.

### Step 1: Create Directories
```bash
mkdir -p /tmp/overlay_lab/{lower,upper,work,merged}
cd /tmp/overlay_lab

# Create initial files in read-only lowerdir
echo "Original config" > lower/config.txt
echo "Static binary code" > lower/app.bin
```

### Step 2: Mount OverlayFS
```bash
sudo mount -t overlay overlay \
  -o lowerdir=lower,upperdir=upper,workdir=work \
  merged/
```

### Step 3: Inspect the Unified Merged Directory
```bash
ls -l merged/
```
```
total 8
-rw-r--r-- 1 user user 19 Oct 14 06:40 app.bin
-rw-r--r-- 1 user user 16 Oct 14 06:40 config.txt
```

### Step 4: Test Copy-on-Write (Modifying a File)
```bash
# Modify config.txt through the merged mount point
echo "Updated in container" >> merged/config.txt

# Inspect lowerdir vs upperdir
cat lower/config.txt
# Output: Original config (Untouched!)

cat upper/config.txt
# Output:
# Original config
# Updated in container
```
The file was automatically copied up to `upper/` upon write!

### Step 5: Test File Deletion (Whiteout Creation)
```bash
# Delete app.bin from merged
rm merged/app.bin

# Verify it disappeared from merged
ls -l merged/
# Output: config.txt (app.bin is gone!)

# Inspect upperdir to reveal the whiteout device
ls -l upper/
```
```
total 4
-rw-r--r-- 1 user user 37 Oct 14 06:42 config.txt
c--------- 1 root root 0, 0 Oct 14 06:43 app.bin
```
*(Notice the `c--------- 0, 0` character device representing the whiteout!)*

### Step 6: Clean Up
```bash
sudo umount merged/
rm -rf /tmp/overlay_lab
```

---

## 5. OverlayFS Inside Docker (`/var/lib/docker/overlay2`)

In production Docker environments:
```bash
# Inspect Docker storage info
docker info | grep -A 5 "Storage Driver"
```
```
 Storage Driver: overlay2
  Backing Filesystem: ext4
  Supports d_type: true
  Using metacopy: true
  Native Overlay Diff: true
```

Every container corresponds to a directory inside `/var/lib/docker/overlay2/<HASH>/`:
- `diff`: The `upperdir` holding container-specific modifications.
- `link`: Short identifier for layer referencing.
- `lower`: Colon-separated list of image layer hashes (`lowerdir`).
- `merged`: Active container root mount point.
- `work`: The atomic scratch directory.

---

## 6. Verification & Troubleshooting Runbook

### Health & Diagnostic Commands
```bash
# Verify backing filesystem supports d_type (mandatory for overlay2!)
xfs_info /var/lib/docker | grep ftype=1

# Check disk space consumed by container writable layers
docker ps -s

# View active mount parameters of a container
cat /proc/$(docker inspect -f '{{.State.Pid}}' <CONTAINER_ID>)/mountinfo | grep overlay
```

### Common Pitfalls & Solutions
1. **`overlay: filesystem does not support d_type`**:
   - Cause: Backing XFS filesystem formatted without `ftype=1` (common on older RHEL 7 installs).
   - Solution: Reformat filesystem with `mkfs.xfs -n ftype=1 /dev/sdX` or use ext4.
2. **Container build or execution becomes extremely slow**:
   - Cause: Heavy writes or package installations (`apt-get install`, large downloads) inside the container layer causing massive Copy-Up overhead.
   - Solution: Multi-stage builds; purge temporary caches (`rm -rf /var/lib/apt/lists/*`) in the same `RUN` command to prevent bloated lower layers.
3. **`No space left on device` but `df -h` shows disk space available**:
   - Cause: Inode exhaustion caused by millions of micro-files in OverlayFS layers.
   - Solution: Check inode usage with `df -i`. Prune old images and containers via `docker system prune -af`.

---

## 7. Key Takeaways
- OverlayFS (`overlay2`) unions read-only image layers (`lowerdir`) and a writable container layer (`upperdir`) into a unified mount (`merged`).
- Unmodified files are read directly from `lowerdir` with zero disk overhead and shared host page cache.
- Modifying an existing file triggers a Copy-Up of the entire file into `upperdir`.
- File deletions create whiteout character devices (`0/0`) in `upperdir` to mask lower layer files.
- Persistent or write-heavy storage must always use volumes to bypass the Copy-Up penalty.
