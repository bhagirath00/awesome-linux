# Chapter 42: Disk Performance - I/O Schedulers (Elevator Algorithms)

## 1. Introduction: What is an I/O Scheduler?

The Linux kernel **I/O Scheduler** (historically called the **Elevator Algorithm**) sits between the generic block I/O layer and physical disk hardware drivers.

Its responsibility is to:
- **Merge Adjacent Requests**: If an application requests block 10, and another requests block 11, the scheduler merges them into a single larger read/write operation.
- **Reorder Requests**: In mechanical hard drives (HDDs), reordering requests minimizes physical read/write head movement across spinning platters (like an elevator picking up passengers on floors in one direction).
- **Prioritize Latency vs. Throughput**: Ensuring real-time read operations are not starved by massive background write flushes.

```
+--------------------------------------------------------------------------+
|                     The Multi-Queue Block I/O Layer                      |
+--------------------------------------------------------------------------+
|  Applications & Filesystem Operations                                    |
|                                |                                         |
|  blk-mq (Multi-Queue Block Layer - Modern Linux Kernel)                  |
|  - Software Staging Queues (Mapped per CPU core to prevent locks)        |
|                                |                                         |
|  I/O Scheduler Engine          v (none, mq-deadline, bfq, kyber)         |
|  +--------------------------------------------------------------------+  |
|  | Dispatches and merges I/O requests to hardware submission queues   |  |
|  +-----------------------------+--------------------------------------+  |
|                                |                                         |
|  Hardware Dispatch Queues      v                                         |
|  [ NVMe SSD / SATA Controller / Cloud Hypervisor (virtio-blk) ]          |
+--------------------------------------------------------------------------+
```

---

## 2. Modern Multi-Queue (blk-mq) Schedulers

In modern Linux kernels (5.x and 6.x), legacy single-queue schedulers (CFQ, Deadline, Anticipatory) have been completely removed in favor of the **`blk-mq`** (multi-queue) subsystem:

| Scheduler | Architecture & Strategy | Recommended Hardware / Workload |
| :--- | :--- | :--- |
| **`none`** | **No-op / Pass-through**. Bypasses scheduling queues completely; requests are passed directly to hardware. | **NVMe SSDs, AWS EBS, Cloud VMs**. Modern NVMe drives have internal multi-queue parallel hardware processors. Software scheduling adds CPU overhead! |
| **`mq-deadline`**| Enforces strict read/write latency deadlines. Prevents starvation. | **SATA SSDs, Cloud block storage** where predictable read latency is needed. Default for many distributions. |
| **`bfq`** | Budget Fair Queueing. Complex fairness scheduler aimed at desktop interactivity. | **Mechanical HDDs, Desktops, USB drives**. Avoid in high-throughput cloud servers. |
| **`kyber`** | Lightweight scheduler developed by Meta (Facebook) enforcing strict P99 latency target SLAs. | **Fast NVMe arrays under heavy concurrency**. |

---

## 3. Checking and Changing Schedulers Dynamically

The active scheduler for any block device is exposed through sysfs at `/sys/block/<device>/queue/scheduler`:

```bash
# Check active scheduler for NVMe drive nvme0n1:
cat /sys/block/nvme0n1/queue/scheduler
```
```
[none] mq-deadline
```
*(The scheduler enclosed in brackets `[none]` is currently active).*

```bash
# Check scheduler for SATA drive sda:
cat /sys/block/sda/queue/scheduler
```
```
[mq-deadline] bfq none
```

### Switching Schedulers at Runtime:
To switch a drive to `none` (best for NVMe/Cloud VMs):
```bash
echo "none" | sudo tee /sys/block/nvme0n1/queue/scheduler
```

---

## 4. Persisting Schedulers with Udev Rules

Changes in `/sys` are ephemeral and lost upon reboot. To make scheduler selections permanent across hardware reboots, configure a **Udev rule** in `/etc/udev/rules.d/`:

Create `/etc/udev/rules.d/60-io-schedulers.rules`:

```udev
# Set 'none' scheduler for all NVMe SSDs
ACTION=="add|change", KERNEL=="nvme[0-9]*", ATTR{queue/scheduler}="none"

# Set 'none' scheduler for virtual cloud disks (KVM/AWS virtio)
ACTION=="add|change", KERNEL=="vd[a-z]*", ATTR{queue/scheduler}="none"

# Set 'mq-deadline' for SATA SSDs and rotational drives
ACTION=="add|change", KERNEL=="sd[a-z]*", ATTR{queue/scheduler}="mq-deadline"
```

Reload and trigger udev rules:
```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

---

## 5. Summary Takeaways

- Modern Linux kernels use the multi-queue `blk-mq` architecture.
- For high-speed NVMe drives and Cloud VMs (AWS EBS), **`none`** is the recommended scheduler to avoid software lock overhead.
- For SATA SSDs and mixed storage, **`mq-deadline`** provides fair latency guarantees.
- Inspect active schedulers via `/sys/block/<device>/queue/scheduler`.
- Make scheduler selections permanent using Udev rules.
