# Linux Architecture Overview

## The Four-Tier Architecture of Linux

The Linux operating system is structured in four distinct, coordinated layers. Each layer has specific responsibilities and boundaries to guarantee security, stability, and high performance.

```text
+--------------------------------------------------------------------------+
| 1. USER APPLICATIONS & UTILITIES (User Space)                            |
|    - Shells: Bash, Zsh, Sh                                               |
|    - DevOps tools: Docker CLI, Kubectl, Git, Terraform                   |
|    - Production Services: Nginx, PostgreSQL, Redis, Python runtimes      |
|    - GNU Coreutils: ls, grep, awk, sed, curl                             |
+--------------------------------------------------------------------------+
                                     |
                                     | System Calls (Syscalls)
                                     v
+--------------------------------------------------------------------------+
| 2. GNU C LIBRARY (glibc / musl)                                          |
|    - Translates user API calls (e.g., printf, open) into raw kernel      |
|      software interrupts and syscall numbers.                            |
+--------------------------------------------------------------------------+
                                     |
                                     | Ring 3 (User) -> Ring 0 (Kernel)
                                     v
+--------------------------------------------------------------------------+
| 3. THE LINUX KERNEL (Kernel Space)                                       |
|    - System Call Interface (SCI)                                         |
|    - Process Scheduler & IPC (Inter-Process Communication)               |
|    - Memory Management Subsystem (Virtual Memory, MMU, Paging)           |
|    - Virtual File System (VFS) -> ext4, XFS, btrfs, overlayfs            |
|    - Network Stack (TCP/IP, sockets, netfilter, eBPF)                    |
|    - Device Drivers (Storage, Network, GPU, USB)                         |
+--------------------------------------------------------------------------+
                                     |
                                     v
+--------------------------------------------------------------------------+
| 4. HARDWARE LAYER                                                        |
|    - Physical CPU (Registers, Cache, MMU)                                |
|    - RAM (Physical Memory)                                               |
|    - Storage Drives (NVMe, SSD, HDD)                                     |
|    - Network Interface Cards (NICs), PCIe devices                        |
+--------------------------------------------------------------------------+
```

---

## User Space vs Kernel Space (CPU Rings)

Modern x86 and ARM processors enforce hardware-level privilege rings to protect the system from crashes and rogue applications:

```text
       [Ring 3: User Space (Least Privileged)]
            * Applications, Scripts, Web Servers
            * If an app crashes, only its process dies
                    |
                    | Syscall trap (int 0x80 / sysenter)
                    v
       [Ring 0: Kernel Space (Most Privileged)]
            * Direct execution access to physical memory & CPU
            * Complete control over hardware devices
            * Kernel panic stops the whole system
```

### Why this separation matters in DevOps:
1. **Fault Isolation**: If your Node.js or Python API leaks memory or crashes with an unhandled exception, it runs in **Ring 3 (User Space)**. The kernel safely terminates the process and reclaims its allocated memory. The server remains online.
2. **Container Boundaries**: Containers share the host server's **Ring 0 (Kernel Space)**. A container cannot access hardware or bypass kernel rules without making syscalls, which the host kernel polices using security capabilities and namespaces.

---

## How a System Call Works in Practice

When you execute a command as simple as `cat /etc/hostname`, here is the exact sequence of events across the architecture:

```text
1. Terminal runs 'cat' command in User Space (Ring 3).
2. 'cat' invokes glibc function: open("/etc/hostname", O_RDONLY).
3. glibc places syscall number (SYS_openat = 257 on x86_64) into the CPU RAX register.
4. CPU executes 'syscall' instruction, switching CPU privilege from Ring 3 to Ring 0.
5. Kernel's System Call Interface catches the request.
6. Kernel checks VFS permissions: does this user have read rights on this file?
7. Kernel reads blocks from ext4 driver -> NVMe disk controller -> reads into memory buffer.
8. Kernel places file contents into user space buffer.
9. CPU switches back from Ring 0 to Ring 3.
10. 'cat' receives the data and writes it to standard output (stdout / terminal).
```

---

## Architecture Inspection Commands

You can inspect each layer of the Linux architecture directly from your terminal:

```bash
# 1. View Kernel release and hardware architecture
uname -r    # Example: 6.8.0-31-generic
uname -m    # Example: x86_64 (or aarch64 on ARM/Graviton)

# 2. Trace system calls made by an application in real time
strace -c ls /etc

# 3. View CPU architecture details
lscpu

# 4. View block storage devices connected to the hardware
lsblk

# 5. View loaded kernel modules (device drivers)
lsmod | head -n 10
```
