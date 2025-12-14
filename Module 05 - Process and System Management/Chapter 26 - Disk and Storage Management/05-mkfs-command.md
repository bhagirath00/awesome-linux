# Chapter 26: Disk and Storage Management - The mkfs Command (Formatting Filesystems)

## 1. Introduction: What is mkfs?

A freshly created raw partition (e.g., `/dev/sdb1`) is simply a contiguous array of empty binary sectors. Before the operating system can store files, directories, ownership metadata, and permissions, the partition must be formatted with a **Filesystem**.

The `mkfs` (**Make Filesystem**) command is a unified front-end wrapper that calls specific filesystem construction binaries (such as `mkfs.ext4`, `mkfs.xfs`, `mkfs.btrfs`, `mkfs.vfat`).

```
                              mkfs (Unified Wrapper)
                                        |
          +-----------------------------+-----------------------------+
          |                             |                             |
          v                             v                             v
     mkfs.ext4                       mkfs.xfs                     mkfs.vfat
 (Ubuntu / Debian)               (RHEL / Rocky)               (EFI / Boot USB)
```

---

## 2. Formatting as ext4 (`mkfs.ext4`)

`ext4` is the most widely deployed, stable, and backward-compatible journaling filesystem in the Linux ecosystem.

```bash
# Format /dev/sdb1 with an ext4 filesystem
sudo mkfs.ext4 -L "DATA_STORE" /dev/sdb1
```

```
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 13107200 4k blocks and 3276800 inodes
Filesystem UUID: 9e32a410-b981-4210-8910-c412581023a4
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (65536 blocks): done
Writing superblocks and filesystem accounting information: done
```

### Key Parameters:
- `-L <label>`: Assigns an alphanumeric volume label (e.g., `DATA_STORE`).
- **Reserved Blocks**: By default, `mkfs.ext4` reserves **5%** of disk blocks for root. On large multi-terabyte data disks, 5% can waste 100GB+ of storage! Reduce root reservation to 1% or 0%:
  ```bash
  # Set reserved blocks to 1%
  sudo tune2fs -m 1 /dev/sdb1
  ```

---

## 3. Formatting as XFS (`mkfs.xfs`)

`xfs` is the default 64-bit high-performance journaling filesystem on RHEL, Rocky Linux, and CentOS. It is engineered for large datasets and heavy parallel I/O.

```bash
# Format /dev/sdb1 with an XFS filesystem
sudo mkfs.xfs -L "APP_STORAGE" /dev/sdb1
```

```
meta-data=/dev/sdb1              isize=512    agcount=4, agsize=3276800 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1    bigtime=1, inobtcount=1
data     =                       bsize=4096   blocks=13107200, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=6400, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```

> [!NOTE]
> On modern Linux kernels, XFS enables **`reflink=1`** by default. Reflink enables Copy-on-Write (CoW) block cloning (e.g., `cp --reflink=always`), allowing instantaneous file duplication without consuming additional disk space.

---

## 4. Checking Filesystem Integrity: `fsck` and `xfs_repair`

Filesystems can become dirty or corrupted following unexpected power loss, kernel panics, or hardware faults.

> [!CAUTION]
> **NEVER run filesystem check or repair utilities on an actively mounted filesystem!**
> Running `fsck` or `xfs_repair` while a filesystem is mounted read-write will cause severe, irreversible data corruption! Always unmount (`umount`) the partition first!

### Checking ext4 Filesystems (`fsck` / `e2fsck`):
```bash
# 1. Unmount the filesystem
sudo umount /dev/sdb1

# 2. Check and automatically repair errors (-y)
sudo e2fsck -f -y /dev/sdb1
```

### Checking XFS Filesystems (`xfs_repair`):
XFS does not use `fsck`; it uses `xfs_repair`:
```bash
# 1. Unmount partition
sudo umount /dev/sdb1

# 2. Repair damaged XFS filesystem
sudo xfs_repair /dev/sdb1
```

---

## 5. Summary Takeaways

- `mkfs` initializes filesystem structures (superblocks, inodes, journals) onto raw partitions.
- Use `mkfs.ext4` for general Ubuntu/Debian deployments; use `tune2fs -m 1` to recover reserved root space.
- Use `mkfs.xfs` for high-throughput enterprise workloads and massive volumes.
- Never run `fsck` or `xfs_repair` on an actively mounted filesystem.
