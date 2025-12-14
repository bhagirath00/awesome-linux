# Chapter 26: Disk and Storage Management - Linux Storage Architecture Overview

## 1. Introduction: The Linux Storage Stack

Storage management is one of the foundational responsibilities of a Linux systems and DevOps engineer. Whether provisioning persistent storage volumes for Kubernetes pods in AWS EBS, configuring local NVMe solid-state arrays for PostgreSQL databases, or managing logical volume snapshots, understanding how data flows from physical flash chips down into file data blocks is vital.

In Linux, storage is organized in a layered abstraction stack:

```
+--------------------------------------------------------------------------+
|                        The Linux Storage Hierarchy                       |
+--------------------------------------------------------------------------+
|  User Space                                                              |
|  [ Applications: Nginx / Postgres / Docker / bash ]                      |
|                                |                                         |
|  Kernel Virtual Filesystem     v POSIX System Calls (read, write, open)  |
|  +--------------------------------------------------------------------+  |
|  |             VFS (Virtual Filesystem Abstraction Switch)            |  |
|  +-----------------------------+--------------------------------------+  |
|                                |                                         |
|  Filesystem Formats            v ext4 / xfs / btrfs / zfs / tmpfs        |
|  +--------------------------------------------------------------------+  |
|  | Filesystem Drivers (Superblocks, Inodes, Extents, Directories)     |  |
|  +-----------------------------+--------------------------------------+  |
|                                |                                         |
|  Block Layer                   v Block I/O Layer (bio, I/O Schedulers)   |
|  +--------------------------------------------------------------------+  |
|  | Device Mapper (LVM / dm-crypt LUKS / MD-RAID)                     |  |
|  +-----------------------------+--------------------------------------+  |
|                                |                                         |
|  Partition Table               v GPT / MBR                               |
|  +--------------------------------------------------------------------+  |
|  | Partitions (/dev/sda1, /dev/nvme0n1p2)                             |  |
|  +-----------------------------+--------------------------------------+  |
|                                |                                         |
|  Device Drivers & Hardware     v Hardware Bus Controllers                |
|  +--------------------------------------------------------------------+  |
|  | NVMe Driver (/dev/nvmeX) | SCSI/SATA Driver (/dev/sdX) | virtio-blk|  |
|  +--------------------------------------------------------------------+  |
+--------------------------------------------------------------------------+
```

---

## 2. Linux Device Naming Conventions

The Linux kernel discovers storage controllers during hardware enumeration and assigns special **Block Device Nodes** under `/dev/`:

| Device Path Pattern | Underlying Hardware Type | Example Device Names |
| :--- | :--- | :--- |
| **`/dev/sd[a-z]`** | SCSI, SATA, SAS, or USB drives | `/dev/sda` (First disk), `/dev/sdb` (Second disk) |
| **`/dev/sd[a-z][1-9]`**| Partitions on SCSI/SATA drives | `/dev/sda1` (First partition of disk A) |
| **`/dev/nvme[0-9]n[1-9]`**| Non-Volatile Memory Express (PCIe SSDs)| `/dev/nvme0n1` (Controller 0, Namespace 1) |
| **`/dev/nvme[0-9]n[1-9]p[1-9]`**| Partitions on NVMe drives | `/dev/nvme0n1p1` (Partition 1 of NVMe SSD) |
| **`/dev/vd[a-z]`** | VirtIO virtual disks (KVM, QEMU, Cloud VMs)| `/dev/vda` (Primary virtual disk on AWS/KVM) |
| **`/dev/xvd[a-z]`**| Xen virtualized block devices (Legacy AWS)| `/dev/xvda` |
| **`/dev/mapper/*`** | Device Mapper logical volumes (LVM / LUKS)| `/dev/mapper/vg00-lv_root` |

---

## 3. MBR vs. GPT Partition Schemes

Before a disk can host filesystems, it must be partitioned using a partition table standard:

| Characteristic | MBR (Master Boot Record) | GPT (GUID Partition Table) |
| :--- | :--- | :--- |
| **Standard Age** | Legacy (1983) | Modern Standard (UEFI Era) |
| **Max Disk Size Capacity** | **2.2 Terabytes (2 TiB)** | **9.4 Zettabytes (9.4 ZiB)** |
| **Max Primary Partitions** | **4 Primary Partitions** (Requires extended/logical partitions for more) | **128 Partitions** (Default on Linux) |
| **Redundancy & Integrity**| Single sector at LBA 0 (No backup) | Dual copies (Primary at start, Backup at end of disk) + CRC32 checksums |
| **Boot Mode** | Legacy BIOS | Modern UEFI (Requires EFI System Partition / ESP) |
| **Recommendation** | Deprecated | **Mandatory modern standard** |

---

## 4. Modern Linux Filesystem Types

| Filesystem | Developer / Maintainer | Characteristics & Best Use Cases |
| :--- | :--- | :--- |
| **`ext4`** | Linux Kernel Community | Fourth Extended Filesystem. Rock-solid stability, universal tool compatibility, default on Ubuntu/Debian. Best for general Linux workloads. |
| **`xfs`** | Silicon Graphics / Red Hat | High-performance 64-bit journaling filesystem. Exceptional parallel I/O throughput and scalability for massive multi-terabyte datasets. Default on RHEL/Rocky/CentOS. |
| **`btrfs`** | Open Source Consortium | Copy-on-Write (CoW) filesystem featuring built-in software RAID, subvolumes, and instant snapshotting. Default on Fedora/SUSE. |
| **`zfs`** | Sun Microsystems / OpenZFS| Advanced enterprise CoW filesystem with integrated volume management, data integrity checksumming, self-healing, and RAID-Z. |

---

## 5. Summary Takeaways

- Data passes from applications through the VFS layer, filesystem driver, block layer, and device driver.
- Disks are mapped as block special files in `/dev/` (`/dev/sda`, `/dev/nvme0n1`, `/dev/vda`).
- GPT has replaced legacy MBR, supporting disks larger than 2TB and 128 partitions.
- `ext4` and `xfs` are the two dominant production enterprise Linux filesystems.
