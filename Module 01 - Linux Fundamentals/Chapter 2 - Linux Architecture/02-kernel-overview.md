# The Linux Kernel: Architecture, Modules & Subsystems

## What is the Linux Kernel?

The **Linux Kernel** is the master controller of the computer. It executes with full hardware privileges, managing every CPU cycle, byte of physical memory, disk access, and network packet flowing through the machine.

---

## 1. Monolithic vs Microkernel: Why Linux is Modular

In operating system design, there are two primary paradigms:

```text
+---------------------------------------+---------------------------------------+
|          MONOLITHIC KERNEL            |              MICROKERNEL              |
|               (Linux)                 |            (Mach, Minix, QNX)         |
+---------------------------------------+---------------------------------------+
| All core services run together in the | Only minimal code (scheduling & IPC)  |
| same kernel memory space (Ring 0):    | runs in Ring 0. File systems, drivers,|
| - File systems                        | and networking run as user-space      |
| - Device drivers                      | processes (Ring 3).                   |
| - Network stack                       |                                       |
| - Virtual memory                      | Benefit: Isolation & crash safety.    |
|                                       | Drawback: Performance penalty from    |
| Benefit: Ultra-fast performance; zero | frequent context switching between    |
| context-switching overhead for IPC.   | user and kernel space.                |
+---------------------------------------+---------------------------------------+
```

### The Best of Both Worlds: Loadable Kernel Modules (LKMs)
Although Linux is monolithic, it is **modular**. You do not need to recompile the entire kernel to add a new network driver or filesystem. The kernel can dynamically load and unload **Loadable Kernel Modules (`.ko` files)** at runtime on demand.

```bash
# 1. List all currently loaded kernel modules
lsmod | head -n 15

# 2. View metadata and parameters of a specific module
modinfo overlay

# 3. Dynamically load a kernel module (e.g., overlay filesystem used by Docker)
sudo modprobe overlay

# 4. Dynamically load iptables bridge module (required for Kubernetes CNI)
sudo modprobe br_netfilter

# 5. Remove an unused module safely
sudo rmmod dummy_driver 2>/dev/null || true
```

---

## 2. Core Subsystems of the Linux Kernel

```text
+--------------------------------------------------------------------------+
|                     SYSTEM CALL INTERFACE (SCI)                          |
+--------------------------------------------------------------------------+
      |                   |                   |                   |
      v                   v                   v                   v
+-------------+     +-------------+     +-------------+     +-------------+
|   PROCESS   |     |   MEMORY    |     |   VIRTUAL   |     |   NETWORK   |
|  SCHEDULER  |     |  MANAGEMENT |     | FILE SYSTEM |     |    STACK    |
|             |     |             |     |    (VFS)    |     | (Netfilter) |
+-------------+     +-------------+     +-------------+     +-------------+
      |                   |                   |                   |
Process queues       Page tables         ext4 / XFS          TCP/IP buffers
Context switch       Swap / OOM          Inodes / Super      iptables / eBPF
cgroups v2           HugePages           OverlayFS           Routing tables
```

### 1. Process Scheduler (CFS / EEVDF)
Allocates CPU time across thousands of threads fairly and efficiently. In modern kernels (Linux 6.6+), the Earliest Eligible Virtual Deadline First (EEVDF) scheduler guarantees low latency for interactive and high-priority workloads.

### 2. Memory Management Subsystem
- Maps physical RAM into virtual memory addresses for each process.
- Manages **Demand Paging** and **Swapping** (paging memory to disk when RAM is full).
- Houses the **OOM (Out-of-Memory) Killer**: when RAM is exhausted, it scans processes by their `oom_score` and terminates the highest consumer to prevent system collapse.

### 3. Virtual File System (VFS)
An abstraction layer that presents a unified filesystem API to user programs. Whether an application writes to an `ext4` SSD, an `XFS` database disk, an `NFS` network share, or a Docker `OverlayFS` layer, it calls the exact same `write()` function.

### 4. Network Stack
Handles TCP/IP sockets, UDP packets, routing, ARP resolution, and packet filtering via **Netfilter** (the engine behind iptables, UFW, and Kubernetes kube-proxy).

---

## 3. Tuning the Kernel for DevOps: `sysctl`

As a DevOps engineer, you will regularly tune kernel parameters to prepare servers for high-throughput Docker, Kubernetes, Nginx, or database workloads.

Kernel runtime settings live in `/proc/sys/` and are configured via `/etc/sysctl.conf`:

```bash
# View all active kernel parameters
sysctl -a | grep -E "ip_forward|swappiness"

# 1. Enable IP Forwarding (Mandatory for Kubernetes & Docker routing)
sudo sysctl -w net.ipv4.ip_forward=1

# 2. Reduce swapping aggressiveness (0 = swap only to avoid OOM, 60 = default, 100 = aggressive)
sudo sysctl -w vm.swappiness=10

# 3. Increase maximum open file descriptors (Essential for busy web servers)
sudo sysctl -w fs.file-max=2097152

# 4. Make kernel settings permanent across reboots
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
vm.swappiness = 10
EOF

# Reload and apply sysctl settings immediately
sudo sysctl --system
```

---

## 4. Diagnosing Kernel Events

When hardware errors occur, containers crash, or network interfaces drop, the kernel logs messages to its ring buffer:

```bash
# Print kernel boot and runtime messages with human-readable timestamps
sudo dmesg -T | tail -n 20

# Filter kernel log for disk or hardware issues
sudo dmesg -T | grep -i -E "error|nvme|sda|corrupt"

# Query systemd kernel journal logs
sudo journalctl -k -n 50 --no-pager
```
