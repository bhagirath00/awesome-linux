# Chapter 23: System Monitoring - The df Command (Disk Space Usage)

## 1. Introduction: What is the df Command?

The `df` (**Disk Free**) command displays the amount of available and used disk space on all currently mounted Linux filesystems. 

It reads filesystem metadata directly from the superblock and the `/proc/mounts` or `/etc/mtab` tables, providing instantaneous total capacity, used space, available space, and mount point locations.

---

## 2. Command Output and Essential Flags

The universal production flag combination is **`df -hT`**:

```bash
df -hT
```

Flags explained:
- `-h`: **Human-readable** units (KB, MB, GB, TB instead of 1K-blocks).
- `-T`: Displays the **Filesystem Type** (`ext4`, `xfs`, `btrfs`, `tmpfs`).

```
Filesystem     Type      Size  Used Avail Use% Mounted on
udev           devtmpfs  3.8G     0  3.8G   0% /dev
tmpfs          tmpfs     784M  1.4M  783M   1% /run
/dev/sda1      ext4       49G   18G   29G  39% /
/dev/sda15     vfat      105M  6.1M   99M   6% /boot/efi
/dev/sdb1      xfs       200G   82G  119G  41% /data
tmpfs          tmpfs     3.9G     0  3.9G   0% /dev/shm
```

---

## 3. Demystifying Output Columns

| Column | Meaning | Description |
| :--- | :--- | :--- |
| `Filesystem` | Source Device | The physical block device (`/dev/sda1`), virtual device, or network share. |
| `Type` | Filesystem Format | `ext4`, `xfs`, `zfs`, `nfs`, `overlay` (Docker). |
| `Size` | Total Capacity | Total size of the filesystem partition. |
| `Used` | Consumed Space | Space occupied by existing files and directory structures. |
| `Avail` | Available Space | Free space available for non-root users. |
| `Use%` | Percentage Used | Consumed percentage (`Used / (Used + Avail)`). |
| `Mounted on` | Mount Point | The directory tree path where the filesystem is attached. |

> [!NOTE]
> On `ext4` filesystems, the kernel by default reserves **5%** of disk blocks strictly for the `root` superuser (`tune2fs -m 5`). This ensures that even if a runaway log fills the disk to 100% for regular applications, system daemons (`sshd`, `syslog`) still have space to function, allowing administrators to log in and resolve the issue.

---

## 4. Inode Exhaustion: The Hidden Outage (`df -i`)

A Linux filesystem can run out of space even when `df -h` shows hundreds of gigabytes available! This occurs when the filesystem runs out of **Inodes**.

Every single file, directory, and symbolic link requires exactly one inode data structure to store its metadata. If an application (e.g., an unconfigured mail queue, session cache, or microservice log folder) creates millions of tiny 0-byte or 1-byte files, the inode table becomes 100% full. At that point, the system returns **"No space left on device"**, crashing databases and web servers!

Always verify inode availability using the **`-i`** flag:

```bash
df -ih
```

```
Filesystem     Inodes IUsed IFree IUse% Mounted on
udev             978K   420  978K    1% /dev
tmpfs            980K   890  979K    1% /run
/dev/sda1        3.2M  420K  2.8M   14% /
/dev/sdb1         10M  9.9M  100K   99% /data   <-- CRITICAL WARNING: INODES EXHAUSTED!
```

---

## 5. Filtering Filesystems

Cloud instances and Kubernetes nodes often mount dozens of temporary `tmpfs` and `overlay` filesystems that clutter output. You can exclude or filter by filesystem type:

```bash
# Exclude pseudo and temporary filesystems (-x)
df -hT -x tmpfs -x devtmpfs -x squashfs

# Only display ext4 and xfs filesystems (-t)
df -hT -t ext4 -t xfs
```

---

## 6. Summary Takeaways

- Always use `df -hT` for human-readable sizes and filesystem types.
- Check inode capacity with `df -ih` whenever receiving "No space left on device" errors on a drive that appears to have free gigabytes.
- Exclude noise like `tmpfs` and `squashfs` with `-x`.
- Monitor root filesystems closely when `Use%` crosses 85-90%.
