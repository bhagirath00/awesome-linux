# Hard Links in Linux: Inodes, Mechanics, and Architecture

## Overview

In the Linux filesystem, a file does not consist of just a "name" sitting on a disk. Every file is composed of two distinct components:
1. **The Inode (Index Node)**: A metadata structure stored in the filesystem table that holds the file's size, permissions, owner, timestamps, and pointers to the physical storage blocks on disk.
2. **The Directory Entry (Dentry)**: A human-readable text name mapped to that inode number.

A **Hard Link** is simply an **additional directory entry that points directly to an existing Inode**.

Understanding hard links is fundamental to understanding how Linux manages storage, how files are deleted, and how zero-cost instant snapshot tools (like Rsync/Rsnapshot and Git) achieve deduplication without duplicating disk space.

---

## 1. Architectural Anatomy: How Hard Links Work

When you create a hard link, you are **not copying data** and you are **not creating a shortcut**. You are creating a second, equal name for the exact same physical inode on disk:

```text
Directory Entry: "file1.txt" ──┐
                               ▼
                        [ INODE #482910 ]
                        ├── File Size: 10 MB
                        ├── Permissions: 0644
                        ├── Link Count: 2
                        └── Pointers to Data Blocks
                               │
                               ▼
                        [ Physical Disk Storage: 10 MB Data ]
                               ▲
Directory Entry: "file2.txt" ──┘
```

Both `file1.txt` and `file2.txt`:
- Share the **exact same Inode number**.
- Share the **exact same permissions and ownership**.
- Share the **exact same physical storage blocks**.
- Occupy **0 additional bytes of disk space** (except for the trivial few bytes required for the directory name entry).

---

## 2. Command Syntax and Creation

```bash
ln TARGET LINK_NAME
```

### Demonstration: Creating and Inspecting Hard Links
```bash
# Step 1: Create an initial file
echo "Production Configuration v1" > original.txt

# Step 2: Create a hard link
ln original.txt hardlink.txt

# Step 3: Inspect inode numbers (-i) and link counts (-l)
$ ls -li original.txt hardlink.txt
482910 -rw-r--r-- 2 ubuntu ubuntu 28 Oct 14 09:50 hardlink.txt
482910 -rw-r--r-- 2 ubuntu ubuntu 28 Oct 14 09:50 original.txt
```

### Key Observations:
1. Both files have the **exact same inode number**: `482910`.
2. The hard link counter (second column) increased from `1` to **`2`**.
3. If you edit `hardlink.txt`, changes appear in `original.txt` instantly because they are reading the identical storage blocks.

---

## 3. The Unlink Mechanism: What Happens When You `rm`?

If you delete `original.txt`:
```bash
rm original.txt
```

Does `hardlink.txt` break? **No!**

```bash
$ cat hardlink.txt
Production Configuration v1

$ ls -li hardlink.txt
482910 -rw-r--r-- 1 ubuntu ubuntu 28 Oct 14 09:50 hardlink.txt
```

### Explanation:
The command `rm` invokes the `unlink()` system call:
1. Linux removes the directory entry `original.txt`.
2. The kernel decrements the Inode's link count from `2` down to `1`.
3. Because the link count is still `> 0`, the inode and its data blocks **remain intact on disk**.
4. The physical data blocks are only released back to the free storage pool when the link count reaches **`0`** (and no running processes hold an open file descriptor).

---

## 4. The Two Strict Limitations of Hard Links

Because of how filesystems are designed, Linux enforces two absolute constraints on hard links:

### Limitation 1: Cannot Cross Filesystem / Partition Boundaries
Inodes are strictly local to an individual filesystem partition. Inode `#482910` on `/dev/sda1` (root partition `/`) has no relation to Inode `#482910` on `/dev/sdb1` (`/mnt/data`).

Attempting to hard-link across different disks or mount points results in an `EXDEV` error:
```bash
$ ln /tmp/app.log /mnt/nfs/app.log
ln: failed to create hard link ‘/mnt/nfs/app.log’ => ‘/tmp/app.log’: Invalid cross-device link
```

### Limitation 2: Cannot Hard Link Directories
To prevent circular reference loops that would crash filesystem walkers (`find`, `du`, `fsck`), standard Linux users and even root cannot create hard links to directories:

```bash
$ ln /var/log /tmp/logdir
ln: /var/log: hard link not allowed for directory
```
*(The only directory hard links that exist are the internal `.` (self) and `..` (parent) entries created automatically by the kernel).*

---

## 5. DevOps Applications: Instant Zero-Cost Snapshots

Tools like `rsnapshot`, Apple Time Machine, and Docker image layering rely heavily on hard links via the `cp -al` (*copy as link*) command:

```bash
# Create an instantaneous daily backup snapshot of a 100GB directory:
cp -al /backups/daily.0 /backups/daily.1
```
- **Time taken**: Less than 1 second!
- **Disk space consumed**: **0 MB**!
- Only files that are subsequently modified in `daily.0` will take up new disk space.

---

## Summary

- A hard link is an additional directory name pointing to an existing Inode.
- Hard links share the same inode number, permissions, and data blocks.
- Deleting one hard link decrements the link counter; data is only destroyed when the link count reaches 0.
- Hard links cannot cross filesystem mount points and cannot point to directories.
- Zero-cost snapshot systems use hard links (`cp -al`) to create instant incremental backups.
