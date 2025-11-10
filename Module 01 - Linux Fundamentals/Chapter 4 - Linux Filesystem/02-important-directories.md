# Important Linux Directories: Deep Dive for DevOps

## Introduction

In daily cloud and DevOps operations, 90% of your configuration, debugging, and log auditing takes place inside four core directories: **`/etc`**, **`/var`**, **`/proc`**, and **`/dev`**.

---

## 1. `/etc`: The Configuration Engine

The name stands for **"Editable Text Configuration"** (or *et cetera*). It contains all host-specific, plain-text system configuration files. Binaries do not belong here.

### Crucial DevOps Files in `/etc`:
| File / Directory | Purpose in DevOps |
|---|---|
| **`/etc/hosts`** | Local DNS hostname-to-IP resolution overrides (useful for testing microservices). |
| **`/etc/resolv.conf`** | Configured upstream DNS nameservers (e.g., `nameserver 8.8.8.8`). |
| **`/etc/fstab`** | Filesystem Table: defines which disk partitions, NFS mounts, or EBS volumes mount on boot. |
| **`/etc/passwd`** | List of all system user accounts, UIDs, GIDs, and default shells. |
| **`/etc/shadow`** | Encrypted password hashes (readable strictly by `root`). |
| **`/etc/sudoers`** | Permissions defining which users can execute commands via `sudo`. |
| **`/etc/systemd/system/`** | Custom `systemd` unit files for running production background services. |
| **`/etc/docker/daemon.json`** | Configuration for the Docker daemon (insecure registries, logging drivers, storage). |
| **`/etc/security/limits.conf`** | User resource limits (maximum open file descriptors `nofile`, max processes `nproc`). |

---

## 2. `/var`: Variable Data, Logs & Docker Storage

The `/var` directory contains files whose contents dynamically expand as the system runs: logs, databases, package caches, and container storage.

```text
/var
├── log/
│   ├── syslog (or /var/log/messages on RHEL) - General system logs
│   ├── auth.log (or secure)                  - SSH logins, sudo attempts, auth failures
│   ├── nginx/ or apache2/                   - Web server access and error logs
│   └── journal/                              - Binary systemd journal storage
├── lib/
│   ├── docker/                               - Docker images, overlay2 layers, container volumes
│   └── containerd/                           - Containerd runtime state
└── run/ -> /run/                             - Active PID files and Unix domain sockets (/var/run/docker.sock)
```

> **Production Storage Tip**: Because `/var/log` and `/var/lib/docker` can grow rapidly and fill up disk space, production servers often mount `/var` or `/var/log` on an independent disk partition so a log spike does not crash the root (`/`) filesystem.

---

## 3. `/proc`: The Kernel Process Virtual Filesystem

The `/proc` directory **does not exist on your physical hard drive or SSD**. It is a **pseudo-filesystem created dynamically in RAM by the Linux kernel**. It acts as a window directly into the kernel's data structures and active processes.

```bash
# 1. Inspect real-time CPU architecture and core details
cat /proc/cpuinfo | grep "model name" | head -n 1

# 2. Inspect real-time memory stats (RAM, Buffers, Cached, Swap)
cat /proc/meminfo | head -n 10

# 3. Every running process has its own numerical folder named after its PID:
ls -d /proc/[0-9]* | head -n 5

# Example: Inspecting process PID 1245
cat /proc/1245/cmdline   # Exact command line string used to start the process
cat /proc/1245/environ   # All environment variables active inside that process
ls -l /proc/1245/fd      # All open file descriptors (sockets, logs, pipes) held by the process
cat /proc/1245/status    # Memory footprint and thread count
```

---

## 4. `/dev`: Device Files & Special Pseudo-Devices

Under Linux's *"Everything is a file"* philosophy, hardware devices and kernel utilities are represented as special node files inside `/dev`:

### Physical Device Nodes:
- `/dev/sda`, `/dev/sdb`: SATA or SCSI hard drives.
- `/dev/nvme0n1`: NVMe high-speed SSD drives (`p1`, `p2` denote partitions).

### The Three Essential DevOps Pseudo-Devices:
1. **`/dev/null` (The Bit Bucket / Black Hole)**:
   Discards all data written to it immediately and returns EOF on reads. Essential for silencing command output:
   ```bash
   # Suppress both stdout (1) and stderr (2)
   ./run_backup.sh > /dev/null 2>&1
   ```
2. **`/dev/zero`**:
   Produces an infinite stream of null bytes (`0x00`). Used to initialize empty files or test disk write speeds:
   ```bash
   # Create a clean 1GB dummy file:
   dd if=/dev/zero of=testfile.img bs=1M count=1024
   ```
3. **`/dev/urandom`**:
   Produces a continuous stream of cryptographically secure pseudo-random bytes from kernel entropy. Used to generate secure tokens and passwords:
   ```bash
   # Generate a random 32-character alphanumeric secret token:
   tr -dc 'A-Za-z0-9' < /dev/urandom | head -c 32 ; echo
   ```

---

## 5. `/tmp` vs `/var/tmp`: Which Scratchpad to Use?

- **`/tmp`**: Stored in a RAM-backed filesystem (`tmpfs`) on modern distributions. Cleared automatically upon reboot, or pruned every 10 days by `systemd-tmpfiles`.
- **`/var/tmp`**: Stored on physical persistent disk. Survives system reboots and is only cleaned after 30 days.
