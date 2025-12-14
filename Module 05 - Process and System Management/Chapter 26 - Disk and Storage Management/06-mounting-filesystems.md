# Chapter 26: Disk and Storage Management - Mounting and Unmounting Filesystems

## 1. Introduction: The Linux Mount Hierarchy

In Windows operating systems, separate storage drives and partitions are represented by independent drive letters (`C:\`, `D:\`, `E:\`). 

In Linux, there are no drive letters. The entire operating system is organized as a **single, unified inverted directory tree** starting from the root directory (**`/`**).

**Mounting** is the operating system process of attaching a formatted storage filesystem on a block device (or network export) to a specific directory within this unified tree. That target directory is called the **Mount Point**. Once mounted, navigating into that directory accesses the files stored on the underlying disk partition.

```
+--------------------------------------------------------------------------+
|                        Unified Linux Directory Tree                      |
+--------------------------------------------------------------------------+
|                                    / (Root)                              |
|                                    |                                     |
|          +-------------------------+-------------------------+           |
|          |                         |                         |           |
|        /bin                      /mnt                      /var          |
|                                    |                         |           |
|                               +----+----+                    |           |
|                               |         |                    |           |
|                           /mnt/backup  /mnt/data             |           |
|                                         ^                    |           |
|                                         | Mount              v           |
|                                    +----+----+       +-------+-------+   |
|                                    |/dev/sdb1|       | /dev/nvme0n1  |   |
|                                    |(Ext4 50G|       |  (Fast NVMe)  |   |
|                                    +---------+       +---------------+   |
+--------------------------------------------------------------------------+
```

---

## 2. The mount Command Syntax and Usage

**Basic Syntax:**
```bash
sudo mount [options] <device_or_uuid> <mount_point_directory>
```

### Step-by-Step Mounting Walkthrough:
```bash
# 1. Create an empty directory to serve as the mount point
sudo mkdir -p /mnt/data

# 2. Attach the block device to the mount point
sudo mount /dev/sdb1 /mnt/data

# 3. Verify the mount is active and check available space
df -hT /mnt/data
```

```
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/sdb1      ext4   49G   24K   47G   1% /mnt/data
```

---

## 3. Useful Mount Options (`-o`)

The `-o` flag passes specific operational flags to the filesystem driver:

| Option | Meaning & Security Impact |
| :--- | :--- |
| **`ro`** | **Read-Only**: Mounts filesystem in read-only mode (prevents any data writes). |
| **`rw`** | **Read-Write**: Default operational mode. |
| **`noexec`** | **No Execution**: Disallows running any binary executables or scripts located on this partition (great security hardening for `/tmp` and upload dirs). |
| **`nosuid`** | **No SUID**: Ignores SUID and SGID permission bits (prevents privilege escalation attacks). |
| **`nodev`** | **No Device Nodes**: Prevents character or block special devices from functioning. |
| **`remount`** | Modifies mount options on an **already mounted** filesystem without unmounting it. |

### Practical Examples:
```bash
# Mount a partition read-only for digital forensics or immutable archives
sudo mount -o ro /dev/sdb1 /mnt/data

# Remount a currently mounted partition to switch from read-only to read-write
sudo mount -o remount,rw /mnt/data

# Remount the root filesystem read-write during rescue mode
sudo mount -o remount,rw /
```

---

## 4. The umount Command

To detach a filesystem safely:

**Syntax:**
```bash
sudo umount <mount_point_OR_device>
```

```bash
# Unmount by specifying the mount directory
sudo umount /mnt/data

# Or unmount by specifying the device node
sudo umount /dev/sdb1
```

---

## 5. Resolving the "target is busy" Unmount Error

If you attempt to unmount a filesystem and receive:
```
umount: /mnt/data: target is busy.
```

This error means a process, daemon, or user shell currently has an open file, an active socket, or has `/mnt/data` set as its **current working directory (`cwd`)**.

### Step 1: Identify Who is Using the Mount
```bash
# Option A: Using lsof
sudo lsof +D /mnt/data

# Option B: Using fuser
sudo fuser -vm /mnt/data
```

```
                     USER        PID ACCESS COMMAND
/mnt/data:           devops     3412 ..c..  bash
```
PID `3412` has its shell current working directory set to `/mnt/data`.

### Step 2: Clear and Terminate
Instruct the user to `cd` out of the directory, or kill the locking process:
```bash
sudo fuser -km /mnt/data
```

### Lazy Unmount (`-l`)
If an NFS share or hung external drive cannot close immediately, a **lazy unmount** detaches the filesystem from the directory tree instantly and cleans up all references as soon as the active processes close their handles:
```bash
sudo umount -l /mnt/data
```

---

## 6. Summary Takeaways

- Filesystems are attached to empty directories called mount points using `mount`.
- Use `-o ro` for immutable mounts and `-o noexec,nosuid` for security hardening.
- Use `-o remount,rw` to change options on the fly without unmounting.
- Troubleshoot "target is busy" errors using `lsof +D` or `fuser -vm`.
- Use `umount -l` (lazy unmount) for hung network or external storage.
