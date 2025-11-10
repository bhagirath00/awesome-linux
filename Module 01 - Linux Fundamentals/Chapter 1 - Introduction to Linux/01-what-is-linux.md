# What is Linux?

## Introduction

Linux is the foundational operating system of modern computing. If you use a smartphone, browse the web, run a container in Docker, deploy a Kubernetes cluster, or spin up an instance on AWS, Google Cloud, or Azure, you are interacting with Linux.

In the world of DevOps, **Linux is not optional — it is the default environment**.

```text
+--------------------------------------------------------------------------+
|                              USER / DEVOPS                               |
|                (CI/CD, Scripts, Docker CLI, Kubernetes)                  |
+--------------------------------------------------------------------------+
                                     |
                                     v
+--------------------------------------------------------------------------+
|                             USER SPACE                                   |
|   +-------------------+  +-------------------+  +-------------------+    |
|   |   Shell (Bash)    |  |  GNU Utilities    |  |  Applications     |    |
|   |  (sh, bash, zsh)  |  | (ls, grep, sed)   |  | (Nginx, Python)   |    |
|   +-------------------+  +-------------------+  +-------------------+    |
|                                    |                                     |
|                         System Calls (syscalls)                          |
+------------------------------------|-------------------------------------+
                                     v
+--------------------------------------------------------------------------+
|                            KERNEL SPACE                                  |
|   +------------------------------------------------------------------+   |
|   |                      The Linux Kernel                            |   |
|   |  * Process Management       * Memory Management (Virtual Memory) |   |
|   |  * File Systems (ext4, XFS) * Device Drivers                     |   |
|   |  * Networking Stack (TCP/IP)* Security (cgroups, namespaces)     |   |
|   +------------------------------------------------------------------+   |
+------------------------------------|-------------------------------------+
                                     v
+--------------------------------------------------------------------------+
|                              HARDWARE                                    |
|                   (CPU, RAM, Disks, NICs, GPUs)                          |
+--------------------------------------------------------------------------+
```

---

## The Definition: Kernel vs Operating System

Technically, **Linux is just a kernel** — not a complete operating system.

### 1. What is the Kernel?
The kernel is the core program that manages the hardware of the computer. It controls:
- **CPU Scheduling**: Deciding which process runs on which CPU core and for how long.
- **Memory Management**: Allocating physical RAM, handling virtual memory paging, and preventing processes from overwriting each other's memory.
- **Device Drivers**: Translating generic software read/write calls into signals that specific storage disks, network cards, and GPUs understand.
- **System Calls (Syscalls)**: Serving as the bridge between software programs and the hardware.
- **Security & Isolation**: Enforcing file permissions, user boundaries, network namespaces, and control groups (`cgroups`).

### 2. What is GNU/Linux?
A kernel alone cannot be used by humans or developers. You need tools to interact with it:
- A command interpreter (**Shell**, such as `bash` or `zsh`).
- Core utilities (**GNU Coreutils**, such as `ls`, `cp`, `mv`, `cat`, `grep`).
- A C library (**glibc** or `musl`) providing standard API implementations.
- Compilers, editors, and package managers.

When the Linux kernel is combined with GNU software utilities and other open-source tools, it forms a complete **GNU/Linux Operating System** (commonly referred to simply as "Linux").

---

## Core Characteristics of Linux

### 1. Multi-User and Multi-Tasking
Linux was built from the ground up to allow multiple users to log in simultaneously (via SSH or terminals), run independent tasks, and share resources without interfering with one another.

### 2. Everything is a File (The Unix Philosophy)
One of the most powerful paradigms in Linux:
- Text files and binaries are files (`/etc/hosts`, `/bin/bash`).
- Directories are files containing lists of filenames.
- Hard drives, partitions, and USBs are files (`/dev/sda1`, `/dev/nvme0n1`).
- Running processes expose their status as files (`/proc/cpuinfo`, `/proc/meminfo`).
- Hardware sensors and kernel settings are exposed as files (`/sys/class/net/eth0/operstate`).

Because everything is represented as a file, you can read, write, redirect, pipe, and manipulate system state using standard text processing commands (`cat`, `grep`, `awk`, `echo`).

```bash
# Example: Inspecting CPU details by reading a virtual file
cat /proc/cpuinfo | grep "model name" | head -n 1

# Example: Checking available memory
cat /proc/meminfo | grep MemTotal
```

### 3. Modularity and Composability
Small, single-purpose utilities that do one job exceptionally well, chained together using the pipe operator (`|`):

```bash
# Find all active Nginx processes, extract their PIDs, and count them
ps aux | grep nginx | grep -v grep | awk '{print $2}' | wc -l
```

### 4. Open Source and Free (GPLv2)
Linux is licensed under the **GNU General Public License version 2 (GPLv2)**. Anyone can:
- Inspect the source code.
- Modify it to their needs.
- Distribute their modifications freely.
This transparency eliminated vendor lock-in and created the global cloud and container ecosystem.

---

## How Linux Interacts with Hardware

When an application wants to interact with hardware, it does not access hardware directly. It makes a **System Call** to the kernel:

```text
User Application (e.g., Python script)
       |
       | 1. open("app.log", O_WRONLY)
       v
  C Library (glibc)
       |
       | 2. sys_open (syscall instruction)
       v
 Linux Kernel (Ring 0 - Privileged Mode)
       |
       | 3. Checks file permissions, maps inode, sends write to disk driver
       v
 Physical SSD / NVMe Storage
```

This clean boundary guarantees system stability: even if user applications crash or throw errors, the kernel remains secure and stable.

---

## Summary Checklist for DevOps

| Aspect | Description |
|---|---|
| **What it is** | A Unix-like, open-source operating system kernel created by Linus Torvalds. |
| **Why it dominates servers** | Stability, security, low resource footprint, scriptability, and zero licensing fees. |
| **Key philosophy** | "Everything is a file" + small, composable tools connected via pipes. |
| **DevOps relevance** | Underpins Docker containers, Kubernetes worker nodes, AWS EC2, and CI/CD runners. |
