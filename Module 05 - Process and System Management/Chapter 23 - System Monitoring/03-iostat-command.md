# Chapter 23: System Monitoring - The iostat Command

## 1. Introduction: What is iostat?

`iostat` (**Input/Output Statistics**) is part of the `sysstat` monitoring suite. It monitors Linux system storage input/output device loading by observing the time physical disks and network filesystems spend operating relative to their average transfer rates.

When an application or database experiences sudden latency spikes and `top` shows high `%wa` (I/O wait), `iostat` is the definitive tool to identify exactly which disk device, volume group, or partition is saturated.

---

## 2. Installation

`iostat` is included in the `sysstat` package:

```bash
# Ubuntu / Debian
sudo apt install -y sysstat

# RHEL / Rocky / AlmaLinux
sudo dnf install -y sysstat
```

---

## 3. The Gold Standard Command: `iostat -xz 1`

Running `iostat` without flags prints basic global counters. In production performance investigations, the universal invocation is:

```bash
iostat -xz 1
```

Flags explained:
- `-x`: Displays **extended statistics** (essential for latency and queue depth).
- `-z`: Suppresses output for inactive/idle devices (reduces visual clutter).
- `1`: Polls every 1 second continuously.

```
Linux 6.5.0-28-generic (db-server-01)    10/14/2026      _x86_64_        (8 CPU)

Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm  r_await  w_await   aqu-sz  rareq-sz  wareq-sz  svctm  %util
sda             12.00   85.00    340.00   4210.00     0.00    14.00   0.00  14.14     1.20     3.40     0.15     28.33     49.53   0.85   8.24
nvme0n1        145.00  480.00   9240.00  32400.00     2.00    85.00   1.36  15.04     0.45     0.82     0.48     63.72     67.50   0.21  13.12
```

---

## 4. Key Metrics and Diagnostic Interpretations

| Metric Column | Meaning | Healthy Baseline | Problem Indicator / Bottleneck |
| :--- | :--- | :--- | :--- |
| **`r/s` & `w/s`** | Read & Write IOPS | Workload-dependent | Sudden drop during heavy load indicates backend throttling. |
| **`rkB/s` & `wkB/s`**| Read/Write Throughput (KB/s)| Workload-dependent | Reaching hardware bus limits (SATA ~550MB/s, NVMe ~3-7GB/s). |
| **`r_await`** | Average Read Latency (ms) | **< 1.0 ms** (NVMe)<br>**< 5.0 ms** (SSD) | Values > 10–20ms mean storage cannot keep up with read requests. |
| **`w_await`** | Average Write Latency (ms)| **< 1.0 ms** (NVMe)<br>**< 5.0 ms** (SSD) | Large write spikes indicate filesystem flush stalls or write exhaustion. |
| **`aqu-sz`** | Average Queue Size | Close to 0 (or ~1-2) | High queue size indicates requests are backing up waiting for disk. |
| **`%util`** | **Device Utilization %** | **< 60%** | **Consistently > 85–100% means the device is completely saturated!** |

> [!CAUTION]
> On multi-disk RAID arrays, SSDs with internal parallelism, or modern NVMe drives, `%util` can approach 100% while the drive is still capable of servicing further parallel requests. Therefore, **always cross-reference `%util` with `r_await` and `w_await` (latency)**. If `%util` is 99% AND latency increases from 0.5ms to 45ms, the storage subsystem is undeniably saturated!

---

## 5. Helpful Options and Formatting

### Human-Readable Units (`-h`)
Converts raw kilobyte numbers into MB/s and GB/s:
```bash
iostat -xzh 1
```

### Filtering Specific Devices
Focus only on your database data drive:
```bash
iostat -xz 1 nvme0n1
```

---

## 6. Summary Takeaways

- `iostat -xz 1` is the definitive command for storage performance analysis.
- Focus on **`r_await`** and **`w_await`** for I/O response latency in milliseconds.
- Device saturation occurs when high **`%util`** coincides with elevated **`aqu-sz`** (queue length) and latency.
- Disregard the initial output line (system uptime average) and evaluate subsequent interval samples.
