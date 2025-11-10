# Linux Filesystem Types and the VFS Layer

## What is a Filesystem?

A **filesystem** is the underlying data structure and algorithm that an operating system uses to organize, store, index, retrieve, and manage data on storage devices (NVMe SSDs, SATA hard drives, SAN volumes, or virtual disks).

Without a filesystem, storage is merely an unformatted, continuous sequence of raw binary blocks (sectors). The filesystem translates raw blocks into the intuitive hierarchy of files, folders, permissions, timestamps, and metadata.

---

## The Linux Virtual Filesystem (VFS) Architecture

One of the greatest achievements of the Linux kernel is the **Virtual Filesystem (VFS)** abstraction layer.

Applications never communicate directly with disk storage or specific filesystem drivers. Instead, standard user programs invoke POSIX system calls (`open`, `read`, `write`, `stat`) against the VFS. The VFS acts as a common interface, translating generic requests into filesystem-specific drivers:

```text
+--------------------------------------------------------------------------+
|                       User Space Applications                           |
|               (Nginx, PostgreSQL, Python, Docker, bash)                  |
+--------------------------------------------------------------------------+
                                     │
                      Standard System Calls (open, write)
                                     ▼
+--------------------------------------------------------------------------+
|                     Linux Kernel VFS (Virtual Filesystem)                |
|                    (Common Inode, Dentry, File Objects)                  |
+--------------------------------------------------------------------------+
          │                │                │               │
          ▼                ▼                ▼               ▼
     [ ext4 driver ]  [ XFS driver ]  [ OverlayFS ]   [ NFS / CIFS ]
          │                │                │               │
          ▼                ▼                ▼               ▼
     Local NVMe SSD   Cloud EBS Block   Docker Image     AWS EFS Network
```

Thanks to the VFS, your code writes to an ext4 partition, a network NFS share, a RAM-backed tmpfs, or a Docker OverlayFS layer using identical code.

---

## Comparison of Major Linux Filesystems

Different workloads require different filesystems. Here is how modern production Linux filesystems compare:

| Filesystem | Default On | Key Features | Best Used For |
| :--- | :--- | :--- | :--- |
| **ext4** | Ubuntu, Debian | Journaling, Extents, battle-tested stability, low CPU overhead, fast repair via `fsck`. Max file size: 16 TB. | General-purpose Linux servers, web servers, microservices, container hosts. |
| **XFS** | RHEL, Rocky, Alma, CentOS | 64-bit journaling, allocation groups for parallel multi-threaded I/O, online resizing, delayed allocation. | High-throughput enterprise workloads, PostgreSQL/MySQL databases, big data. |
| **Btrfs** | Fedora Workstation, openSUSE | Copy-on-Write (CoW), writable snapshots, subvolumes, built-in software RAID, self-healing scrub. | Desktop Linux, workstations, backup targets, snapshot-heavy environments. |
| **ZFS** | TrueNAS, Ubuntu Server (opt-in) | Combined filesystem and volume manager (zpool), end-to-end data integrity checksums, copy-on-write, transparent compression. | Enterprise NAS/SAN storage arrays, mission-critical database storage, long-term archiving. |
| **tmpfs** | All Linux Distros (`/dev/shm`, `/run`) | Stored directly in volatile RAM and swap. Zero disk I/O, nanosecond access speeds, wiped cleanly on reboot. | High-speed cache, ephemeral lock files, secure secret decryptions, Docker `/dev/shm`. |
| **OverlayFS** | Container Runtimes (Docker, containerd) | Union mount combining read-only image layers (`lowerdir`) with a thin writable container layer (`upperdir`). | Container image layering, lightweight immutable live systems. |

---

## In-Depth Analysis: The Workhorse Filesystems

### 1. ext4 (Fourth Extended Filesystem)
The default on Debian and Ubuntu distributions:
- **Journaling**: Protects against filesystem corruption during sudden power loss or kernel crashes by recording transactions in a dedicated journal before committing them to the storage blocks.
- **Extents**: Instead of recording every individual disk block, ext4 maps contiguous ranges of blocks (up to 128MB per extent), dramatically reducing metadata overhead and fragmentation.
- **Backwards Compatibility**: Can be mounted directly by older kernels if needed.

### 2. XFS (High-Performance Scalable Filesystem)
The enterprise default for Red Hat Enterprise Linux (RHEL 7, 8, 9, 10):
- **Allocation Groups (AG)**: XFS divides the filesystem into equal-sized chunks (typically 8 or 16 AGs). Each AG manages its own inodes and free space independently. This allows multiple CPU cores to write and allocate disk space simultaneously without lock contention.
- **Dynamic Inodes**: Unlike ext4 (which pre-allocates a fixed number of inodes during formatting), XFS creates inodes on-demand as files are created, avoiding the dreaded `"No space left on device (Inodes exhausted)"` error.
- **Online Resizing**: Can be expanded live while mounted with `xfs_growfs`. (Note: XFS cannot be shrunk).

### 3. OverlayFS: The Engine Behind Docker Containers
Docker and Kubernetes do not duplicate operating system files for every container. They use OverlayFS to stack directories:

```text
[ Writable Container Layer ] (upperdir)  -> Modified / created files
──────────────────────────────────────────────────────────────────
[ Read-Only App Code Layer ] (lowerdir 2) -> /app/server.js
──────────────────────────────────────────────────────────────────
[ Read-Only Node.js Base   ] (lowerdir 1) -> /usr/local/bin/node
──────────────────────────────────────────────────────────────────
[ Read-Only Ubuntu Layer   ] (lowerdir 0) -> Core Linux binaries & libs
```

When a container modifies a file, OverlayFS copies the file up to the `upperdir` (Copy-on-Write) without mutating the underlying base image.

---

## Practical DevOps Inspection and Management Commands

### 1. View Filesystem Types and Disk Utilization (`df -hT`)

The `-T` flag prints the filesystem type column:

```bash
$ df -hT
Filesystem     Type      Size  Used Avail Use% Mounted on
/dev/root      ext4       49G   12G   37G  25% /
tmpfs          tmpfs     7.8G     0  7.8G   0% /dev/shm
/dev/nvme1n1   xfs       100G   24G   76G  24% /data
overlay        overlay    49G   12G   37G  25% /var/lib/docker/overlay2/.../merged
```

### 2. List Block Devices and UUIDs (`lsblk -f`)

Displays disk tree topology, filesystem formats, and unique UUID identifiers:

```bash
$ lsblk -f
NAME        FSTYPE FSVER LABEL UUID                                 MOUNTPOINTS
sda                                                                 
├─sda1      vfat   FAT32       74F8-11A2                            /boot/efi
└─sda2      ext4   1.0         9b31d4d8-4f10-48e0-a75d-3d4411124e93 /
nvme0n1                                                             
└─nvme0n1p1 xfs                8e52a921-12ec-47bf-8bb2-9017631310ad /data
```

### 3. Formatting Storage Devices (`mkfs`)

```bash
# Format partition as ext4
sudo mkfs.ext4 -L "app_data" /dev/sdb1

# Format partition as high-performance XFS
sudo mkfs.xfs -L "db_storage" /dev/nvme1n1
```

### 4. Mounting Storage and Configuring `/etc/fstab`

Manual mounting:
```bash
sudo mkdir -p /mnt/database
sudo mount /dev/nvme1n1 /mnt/database
```

To persist the mount across system reboots, add an entry to `/etc/fstab` using the block device's **UUID** (never rely on `/dev/sdb`, as device letters can shift after reboot):

```bash
# Obtain UUID
sudo blkid /dev/nvme1n1
# Output: /dev/nvme1n1: UUID="8e52a921-12ec-47bf-8bb2-9017631310ad" TYPE="xfs"

# Add entry to /etc/fstab:
# <filesystem-UUID>                         <mount-point>   <type>  <options>       <dump>  <pass>
UUID=8e52a921-12ec-47bf-8bb2-9017631310ad   /mnt/database   xfs     defaults,noatime 0       2
```

### 5. Expanding Filesystems in Cloud Environments (AWS / Azure / GCP)

When you expand an EBS volume from 50GB to 100GB in the AWS console, the disk increases, but the filesystem still occupies the old size:

```bash
# 1. Grow the partition
sudo growpart /dev/nvme0n1 1

# 2A. If using ext4:
sudo resize2fs /dev/nvme0n1p1

# 2B. If using XFS:
sudo xfs_growfs /mnt/database
```
