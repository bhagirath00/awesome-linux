# Chapter 41: Memory Performance - Huge Pages and Transparent Huge Pages (THP)

## 1. The Problem with Standard 4KB Memory Pages

On modern `x86_64` CPU architectures, virtual memory is mapped to physical RAM in discrete blocks called **Pages**. By default, Linux uses a standard page size of **4 Kilobytes (4 KB)**.

Every virtual memory page requires an entry in the kernel's **Page Table**. To accelerate memory access, the CPU hardware caches virtual-to-physical address mappings in a high-speed hardware cache called the **TLB (Translation Lookaside Buffer)**.

### The Scale Problem in Databases:
Consider an enterprise PostgreSQL, MySQL, Redis, or Oracle server with **256 GB of RAM**:
- At 4KB per page, 256 GB requires **67,108,864 individual page table entries**!
- The page table data structures alone consume **500MB+ of kernel RAM**!
- The CPU's physical TLB hardware cache can only hold ~1,500 to 3,000 entries.
- **Result: A continuous 95%+ "TLB Miss Rate"**. Every memory read incurs a multi-cycle "page table walk" through RAM, introducing 10–30% CPU latency overhead!

```
+--------------------------------------------------------------------------+
|                     Standard Pages vs. Huge Pages                        |
+--------------------------------------------------------------------------+
|  Standard 4KB Pages:                                                     |
|  256 GB RAM = 67,108,864 Pages!                                          |
|  [4K][4K][4K][4K][4K][4K][4K][4K][4K]... (Massive TLB Misses / Thrashing)|
|                                                                          |
|  Huge Pages (2MB Pages):                                                 |
|  256 GB RAM = 131,072 Pages! (512x Fewer Entries!)                       |
|  [------------ 2MB ------------][------------ 2MB ------------]...       |
|  (99% TLB Hit Rate / Instantaneous Hardware Addressing!)                 |
+--------------------------------------------------------------------------+
```

---

## 2. Standard (Static) Huge Pages (2MB / 1GB)

**Huge Pages** allow the Linux kernel to manage memory in large chunks: **2 Megabytes (2MB)** or **1 Gigabyte (1GB)** instead of 4KB.

Static Huge Pages are pre-allocated in kernel memory during system boot. Applications like PostgreSQL and Oracle allocate shared memory buffers directly out of this pool.

### Configuring Static Huge Pages:
To pre-allocate 2,048 Huge Pages (each 2MB = 4 GB total):

```bash
# 1. Allocate in running kernel
sudo sysctl vm.nr_hugepages=2048

# 2. Make permanent in /etc/sysctl.d/99-hugepages.conf
echo "vm.nr_hugepages=2048" | sudo tee /etc/sysctl.d/99-hugepages.conf
sudo sysctl --system
```

Verify in `/proc/meminfo`:
```bash
grep -i huge /proc/meminfo
```
```
HugePages_Total:    2048
HugePages_Free:     2048
HugePages_Rsvd:        0
Hugepagesize:       2048 kB
```

Configure PostgreSQL to utilize them in `postgresql.conf`:
```ini
huge_pages = on
```

---

## 3. Transparent Huge Pages (THP) and the Database Hazard

To make huge pages accessible without modifying applications, the Linux kernel introduced **Transparent Huge Pages (THP)**. THP attempts to automatically coalesce 4KB pages into 2MB chunks in the background via a kernel thread named `khugepaged`.

### Why THP is Lethal for Databases (Redis, MongoDB, Postgres):
While THP benefits pure mathematical batch crunching, it is disastrous for low-latency databases and key-value stores:
1. **Memory Allocation Latency Spikes**: When a database modifies a 4KB chunk of memory, THP attempts to allocate and zero an entire 2MB block synchronously, causing latency spikes of tens to hundreds of milliseconds.
2. **Aggressive Copy-on-Write (CoW)**: When Redis forks a background process to save an RDB snapshot (`BGSAVE`), modifying a single byte forces Linux to copy an entire **2MB block** instead of a 4KB page, rapidly triggering Out-Of-Memory (OOM) crashes!

> [!WARNING]
> **Every major database vendor (Redis, MongoDB, Oracle, PostgreSQL, Cassandra) strictly demands that Transparent Huge Pages (THP) be DISABLED in production!**

---

## 4. Disabling Transparent Huge Pages in Production

Check current THP state:
```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
```
```
[always] madvise never
```
*(The brackets indicate `always` is active).*

### Disable THP Immediately in Runtime:
```bash
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/defrag
```

### Make Disabling THP Permanent Across Reboots:
Create a systemd one-shot service:

```bash
sudo tee /etc/systemd/system/disable-thp.service << 'EOF'
[Unit]
Description=Disable Transparent Huge Pages (THP)
DefaultDependencies=no
After=sysinit.target local-fs.target
Before=mongod.service redis.service postgresql.service

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/enabled && echo never > /sys/kernel/mm/transparent_hugepage/defrag'

[Install]
WantedBy=basic.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now disable-thp.service
```

---

## 5. Summary Takeaways

- Standard 4KB pages exhaust CPU TLB caches on systems with large RAM footprints.
- Static Huge Pages (2MB) reduce page table sizes and dramatically boost database performance.
- Pre-allocate static huge pages via `vm.nr_hugepages`.
- **Always disable Transparent Huge Pages (THP)** on database servers (Redis, MongoDB, PostgreSQL) to prevent unpredictable latency spikes and OOM crashes.
