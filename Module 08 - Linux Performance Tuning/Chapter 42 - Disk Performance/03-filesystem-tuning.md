# Chapter 42: Disk Performance - Filesystem Tuning and Mount Optimizations

## 1. Introduction: Tuning the Filesystem Layer

Once storage partitions and I/O schedulers are configured, the **filesystem layer** (`ext4`, `xfs`) dictates how efficiently the operating system commits write buffers, updates file access metadata, and flushes journal logs to disk.

By adjusting filesystem mount options in `/etc/fstab` and kernel dirty memory page ratios in `sysctl`, DevOps engineers can extract up to 30–50% higher write throughput for databases, log shippers, and container storage.

---

## 2. High-Performance Mount Options in /etc/fstab

```
+--------------------------------------------------------------------------+
|                  High-Impact Mount Optimization Flags                    |
+--------------------------------------------------------------------------+
|  [ noatime ]     ===> Completely eliminates access time write overhead.  |
|  [ nodiratime ]  ===> Eliminates directory read timestamp updates.      |
|  [ data=writeback]==> (ext4) Journal only records metadata (High speed). |
|  [ logbufs=8 ]   ===> (XFS) Allocates maximum memory for journal buffers.|
+--------------------------------------------------------------------------+
```

### 1. The `noatime` Option (The #1 Optimization)
By default in POSIX standards, **every time a file or directory is read**, the Linux kernel performs a write operation to disk to update the file's inode `access time` (`atime`). 

Reading a static image or querying a database table forces a physical disk write!
- Adding **`noatime`** completely eliminates these unnecessary metadata disk writes.
- Essential for web servers, static asset caches, and databases.

### 2. Tuning ext4 Journaling Modes (`data=...`)
`ext4` supports three journaling modes:
- `data=journal`: Both data and metadata are committed to the journal before writing to the main filesystem (Slowest, safest).
- `data=ordered` (**Default**): Only metadata is journaled; data is committed before metadata is updated (Balanced).
- `data=writeback`: Only metadata is journaled; data may be written before or after metadata. **Fastest throughput**, but recent unwritten data can be lost on sudden power failure.

### 3. XFS Mount Optimizations:
XFS scales exceptionally well out of the box, but high-write environments benefit from:
- `logbufs=8`: Increases in-memory log buffers to maximum (8) to smooth out write bursts.
- `logbsize=256k`: Increases log buffer size from default 32k to 256k.

### Optimized Production `/etc/fstab` Examples:
```fstab
# Optimized ext4 data volume:
UUID=312048f1-a124-4f81-9981-e41258102914 /var/lib/data ext4 defaults,noatime,nodiratime,data=ordered 0 2

# Optimized XFS database volume:
UUID=c8149102-3141-4891-b124-a12481023812 /var/lib/pgsql xfs defaults,noatime,logbufs=8,logbsize=256k 0 0
```

---

## 3. Kernel Dirty Page Writeback Tuning (sysctl)

When an application writes to a file, the data is initially written to RAM (as **Dirty Pages**). The kernel background thread `flusher` writes these pages out to disk periodically.

You can tune this behavior in `/etc/sysctl.d/99-disk-performance.conf`:

```ini
# Percentage of total system memory that can be dirty before background flush begins
vm.dirty_background_ratio = 5

# Hard ceiling: if dirty memory reaches this percentage, applications writing to disk
# are FROZEN and forced to flush dirty pages synchronously!
vm.dirty_ratio = 10

# Frequency (in 100ths of a second) at which the flusher thread wakes up (500 = 5 seconds)
vm.dirty_writeback_centisecs = 500

# Age (in 100ths of a second) at which a dirty page must be flushed to disk (3000 = 30 seconds)
vm.dirty_expire_centisecs = 3000
```

Apply immediately:
```bash
sudo sysctl --system
```

### Why Lowering `dirty_ratio` Prevents System Freezes:
On a server with 128 GB of RAM, the default `dirty_ratio = 20` allows **25.6 GB of dirty data** to accumulate in memory. When the background flusher finally attempts to write 25 GB to disk, the storage bus saturates completely, causing a system-wide freeze! 

Setting `vm.dirty_background_ratio = 5` and `vm.dirty_ratio = 10` flushes data continuously in small, manageable bursts.

---

## 4. Summary Takeaways

- Always mount high-performance filesystems with **`noatime`**.
- XFS benefits from `logbufs=8` and `logbsize=256k` for high-throughput write workloads.
- Lower `vm.dirty_background_ratio` (5%) and `vm.dirty_ratio` (10%) on servers with large RAM to avoid massive I/O flush stalls.
- Never disable journaling on storage partitions containing irreplaceable databases.
