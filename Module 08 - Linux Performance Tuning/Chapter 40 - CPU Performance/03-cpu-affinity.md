# Chapter 40: CPU Performance - CPU Affinity and NUMA Optimization

## 1. Introduction: The Cost of CPU Migration

In modern multi-core, multi-socket enterprise servers, the Linux kernel scheduler constantly attempts to load-balance runnable processes across all available CPU cores.

However, moving a high-throughput, low-latency process (like an in-memory Redis cache, a high-frequency trading engine, or a network firewall) from CPU 0 to CPU 7 comes with severe performance penalties:
1. **CPU L1/L2 Cache Invalidation**: The process's active working data stored inside the ultra-fast L1 and L2 caches (nanosecond access) of CPU 0 must be purged and reloaded from slower main RAM into CPU 7's caches.
2. **Translation Lookaside Buffer (TLB) Flushes**: Virtual-to-physical memory page translation caches must be dumped and rebuilt.

**CPU Affinity** (also called **CPU Pinning**) allows system architects to bind a process or thread strictly to a dedicated physical CPU core, guaranteeing maximum CPU cache locality.

---

## 2. Managing CPU Affinity with `taskset`

The `taskset` command is the standard utility for querying and setting the CPU affinity of a process. It uses a **CPU affinity mask** represented either as a hexadecimal bitmask or a comma-separated list of CPU core IDs.

```bash
# Query the CPU affinity of a running process (PID 14205)
taskset -p 14205
```
```
pid 14205's current affinity mask: f  (Hex 'f' = binary 1111 = Cores 0, 1, 2, 3)
```

### Setting Affinity by Core ID (`-c`)
The `-c` flag makes core assignment simple without calculating hex bitmasks:

```bash
# 1. Pin an already-running process to CPU Core 2 strictly:
sudo taskset -cp 2 14205

# 2. Pin a process to a range of dedicated cores (Cores 4 through 7):
sudo taskset -cp 4-7 14205

# 3. Launch a brand-new process pinned to Core 1 from startup:
taskset -c 1 redis-server /etc/redis/redis.conf
```

---

## 3. Understanding Non-Uniform Memory Access (NUMA)

In multi-socket enterprise servers (e.g., dual Intel Xeon or AMD EPYC motherboards), CPU cores are grouped into physical **NUMA Nodes**. 

Each NUMA node possesses its own dedicated local memory controller and physical RAM DIMM slots:

```
+--------------------------------------------------------------------------+
|                        NUMA Architecture Diagram                         |
+--------------------------------------------------------------------------+
|  NUMA Node 0                                   NUMA Node 1               |
|  +--------------------+   +---------------+    +--------------------+   +|
|  | Socket 0 (CPUs 0-7)|<->| Local RAM Node0|    | Socket 1 (CPUs 8-15|<->||
|  +---------+----------+   +---------------+    +---------+----------+   +|
|            |                                             |               |
|            +============= Interconnect Bus =============+                |
|                      (QPI / UPI / Infinity Fabric)                       |
|                      High Latency Cross-Traffic!                         |
+--------------------------------------------------------------------------+
```

- **Local Memory Access**: When CPU 0 reads from its local RAM (Node 0), latency is **~60 nanoseconds**.
- **Remote NUMA Access**: When CPU 0 reads from RAM attached to Socket 1 (Node 1), traffic must cross the CPU interconnect bus, doubling latency to **~120–140 nanoseconds**!

---

## 4. Auditing and Optimizing NUMA with `numactl`

```bash
# Install NUMA control utilities
sudo apt install -y numactl   # Ubuntu/Debian
sudo dnf install -y numactl   # RHEL/Rocky

# Inspect hardware NUMA topology:
numactl --hardware
```

```
available: 2 nodes (0-1)
node 0 cpus: 0 1 2 3 4 5 6 7
node 0 size: 32140 MB
node 0 free: 14200 MB
node 1 cpus: 8 9 10 11 12 13 14 15
node 1 size: 32140 MB
node 1 free: 18200 MB
node distances:
node   0   1 
  0:  10  21 
  1:  21  10 
```
The distance between Node 0 and Node 1 is `21` (more than double the internal latency `10`).

### Pinning Applications to Local NUMA Nodes with `numactl`:
Guarantee that a high-performance database (PostgreSQL or Redis) runs strictly on the CPU cores of Node 0 **AND** allocates memory strictly from Node 0's local RAM:

```bash
numactl --cpunodebind=0 --membind=0 /usr/bin/postgres -D /data/pgsql
```
This guarantees zero remote interconnect memory hops, maximizing I/O and transaction throughput!

---

## 5. Summary Takeaways

- CPU migration causes expensive L1/L2 cache invalidations and latency jitter.
- Use `taskset -c <core> <PID>` to bind performance-critical processes to dedicated cores.
- In multi-socket servers, cross-node memory access doubles RAM latency.
- Use `numactl --cpunodebind=0 --membind=0` to pin workloads to local NUMA nodes.
