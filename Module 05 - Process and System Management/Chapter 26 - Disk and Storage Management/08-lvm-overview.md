# Chapter 26: Disk and Storage Management - Logical Volume Manager (LVM) Overview

## 1. Introduction: What is LVM?

**LVM** (**Logical Volume Manager**) is a device-mapper storage management technology that provides an abstraction layer between physical storage disks and the filesystems running on a Linux system.

In traditional static partitioning (`fdisk`), partitions are locked to fixed, contiguous physical boundaries on a single drive. If `/var` runs out of disk space on a traditional partition:
- You cannot easily expand it across a second physical disk.
- You must power down the server, boot a live CD, resize adjacent partitions, or perform complex data migrations.

With LVM, storage disks are pooled together into a dynamic storage pool. Partitions (**Logical Volumes**) can be created, resized on the fly, expanded across multiple physical disks, and snapshotted with **zero system downtime**.

---

## 2. The Three-Tier LVM Architecture

LVM organizes storage into three foundational tiers:

```
+--------------------------------------------------------------------------+
|                         LVM Three-Tier Architecture                      |
+--------------------------------------------------------------------------+
|  Tier 3: Logical Volumes (LV) (Formatted with ext4 / xfs filesystems)    |
|  +---------------------------+   +------------------------------------+  |
|  | lv_root (Mounted on /)    |   | lv_data (Mounted on /var/lib/data) |  |
|  | Size: 40 GB               |   | Size: 260 GB                       |  |
|  +-------------+-------------+   +-----------------+------------------+  |
|                |                                   |                     |
|                +-----------------+                 |                     |
|                                  |                 |                     |
|  Tier 2: Volume Group (VG)       v                 v                     |
|  +--------------------------------------------------------------------+  |
|  | Volume Group: "vg_production" (Total Capacity: 300 GB)             |  |
|  | Pooled Physical Extents (PE): [4MB] [4MB] [4MB] [4MB] [4MB]...     |  |
|  +-------------------------------+-----------------+------------------+  |
|                                  |                 |                     |
|  Tier 1: Physical Volumes (PV)   v                 v                     |
|  +-------------------------------+   +-------------+------------------+  |
|  | Physical Volume: /dev/sdb1    |   | Physical Volume: /dev/sdc1     |  |
|  | (100 GB SSD)                  |   | (200 GB SSD)                   |  |
|  +-------------------------------+   +--------------------------------+  |
|                                                                          |
|  Physical Hardware Storage Disks (SATA, NVMe, SAN LUNs, Cloud EBS)       |
+--------------------------------------------------------------------------+
```

### 1. Physical Volumes (PV)
Raw block devices (e.g., `/dev/sdb1`, `/dev/nvme0n1`, `/dev/sdc`) initialized for LVM use. LVM writes a metadata header and divides the device into fixed-size chunks called **Physical Extents (PE)** (typically 4MB).

### 2. Volume Groups (VG)
The storage pool created by aggregating one or more Physical Volumes together. The Volume Group treats all PEs from all underlying PVs as a single contiguous allocation pool.

### 3. Logical Volumes (LV)
Virtual partitions carved out of a Volume Group. To the Linux kernel and user applications, an LV looks and acts identically to a standard physical partition (`/dev/mapper/vg_production-lv_data`).

---

## 3. Key Advantages of LVM in DevOps

| Capability | Traditional Partitioning | Logical Volume Management (LVM) |
| :--- | :--- | :--- |
| **Resizing Partitions** | Risky, complex, requires unmounting and reboot | **Instant online expansion** (`lvextend -r`) without unmounting or downtime |
| **Spanning Disks** | Impossible on a single filesystem | Seamlessly spans across multiple physical disks |
| **Snapshots** | Not supported | Instantaneous Copy-on-Write snapshots for consistent backups |
| **Thin Provisioning** | Allocates 100% of physical space upfront | Allocates blocks only when data is actually written |
| **Live Data Migration** | Requires offline backups | Move data between physical disks live (`pvmove`) with no downtime |

---

## 4. Logical Volume Device Paths

An active Logical Volume is accessible through two equivalent symbolic paths in `/dev/`:

1. **Standard Device Mapper Path**:
   ```
   /dev/mapper/<VolumeGroupName>-<LogicalVolumeName>
   # Example: /dev/mapper/vg_production-lv_data
   ```
2. **Simplified Symlink Path**:
   ```
   /dev/<VolumeGroupName>/<LogicalVolumeName>
   # Example: /dev/vg_production/lv_data
   ```

Both paths point directly to the same internal device node.

---

## 5. Summary Takeaways

- LVM abstracts storage into **Physical Volumes (PV) -> Volume Groups (VG) -> Logical Volumes (LV)**.
- Volume Groups pool disparate physical drives into a unified storage pool.
- Logical Volumes can be resized online without interrupting active services.
- Point-in-time snapshots enable consistent database backups.
