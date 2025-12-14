# Chapter 26: Disk and Storage Management - LVM Commands and Operations

## 1. The LVM Command Naming Matrix

LVM commands follow a clean, consistent naming convention based on the three storage tiers:

| Tier | Create | Display Summary | Display Detailed | Extend / Grow | Reduce / Shrink |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Physical Volume** | `pvcreate` | `pvs` | `pvdisplay` | `pvresize` | (None) |
| **Volume Group** | `vgcreate` | `vgs` | `vgdisplay` | `vgextend` | `vgreduce` |
| **Logical Volume** | `lvcreate` | `lvs` | `lvdisplay` | `lvextend` | `lvreduce` |

---

## 2. End-to-End Walkthrough: Creating an LVM Volume

Let's configure storage from two new disks (`/dev/sdb` and `/dev/sdc`) into an elastic data volume mounted at `/data`.

### Step 1: Initialize Physical Volumes (`pvcreate`)
```bash
sudo pvcreate /dev/sdb /dev/sdc
```
```
  Physical volume "/dev/sdb" successfully created.
  Physical volume "/dev/sdc" successfully created.
```

### Step 2: Create the Volume Group (`vgcreate`)
Combine both disks into a Volume Group named `vg_data`:
```bash
sudo vgcreate vg_data /dev/sdb /dev/sdc
```
```
  Volume group "vg_data" successfully created.
```

Verify with `vgs`:
```bash
sudo vgs
```
```
  VG      #PV #LV #SN Attr   VSize   VFree  
  vg_data   2   0   0 wz--n- 299.99g 299.99g
```

### Step 3: Create a Logical Volume (`lvcreate`)
Carve out a 50GB Logical Volume named `lv_app`:

```bash
# -L: Specify exact size (50G)
# -n: Specify logical volume name
sudo lvcreate -L 50G -n lv_app vg_data
```
```
  Logical volume "lv_app" created.
```

*(Tip: You can also allocate percentages, e.g., `lvcreate -l 100%FREE -n lv_app vg_data`).*

### Step 4: Format and Mount
```bash
# Format with ext4
sudo mkfs.ext4 /dev/vg_data/lv_app

# Mount to /data
sudo mkdir -p /data
sudo mount /dev/vg_data/lv_app /data
```

---

## 3. Extending a Logical Volume Online (Zero Downtime)

When disk space runs low on an active application, you can expand the Logical Volume **and its underlying filesystem simultaneously** while users are reading and writing data!

### The Magic Flag: `lvextend -r`
The **`-r`** (or `--resizefs`) flag instructs LVM to resize both the block device AND the filesystem (`resize2fs` for ext4 or `xfs_growfs` for XFS) in a single command:

```bash
# Add 20GB of additional space to lv_app online:
sudo lvextend -r -L +20G /dev/vg_data/lv_app

# Or expand to use 100% of all remaining free space in the Volume Group:
sudo lvextend -r -l +100%FREE /dev/vg_data/lv_app
```

```
  Size of logical volume vg_data/lv_app changed from 50.00 GiB (12800 extents) to 70.00 GiB (17920 extents).
  Logical volume vg_data/lv_app successfully resized.
resize2fs 1.46.5 (30-Dec-2021)
Filesystem at /dev/mapper/vg_data-lv_app is mounted on /data; on-line resizing required
old_desc_blocks = 7, new_desc_blocks = 9
The filesystem on /dev/mapper/vg_data-lv_app is now 18350080 (4k) blocks long.
```
Your filesystem is now 70GB, fully expanded online with zero downtime!

---

## 4. Expanding the Volume Group with New Hardware (`vgextend`)

When the Volume Group itself runs out of free extents (`VFree` = 0), attach a new physical disk (e.g., `/dev/sdd`) and extend the pool:

```bash
# 1. Initialize new disk
sudo pvcreate /dev/sdd

# 2. Add to existing Volume Group
sudo vgextend vg_data /dev/sdd

# 3. Now you can extend any logical volume!
sudo lvextend -r -L +100G /dev/vg_data/lv_app
```

---

## 5. Creating Point-in-Time LVM Snapshots

LVM snapshots use Copy-on-Write (CoW). A snapshot records only the blocks that change after the snapshot is taken, making creation instantaneous:

```bash
# Create a 5GB snapshot of lv_app named lv_app_snapshot
sudo lvcreate -L 5G -s -n lv_app_snapshot /dev/vg_data/lv_app
```

### Restoring / Merging a Snapshot:
If a software upgrade fails, roll back the entire volume back to the exact snapshot state:
```bash
# 1. Unmount volume
sudo umount /data

# 2. Revert to snapshot
sudo lvconvert --merge /dev/vg_data/lv_app_snapshot

# 3. Remount
sudo mount /dev/vg_data/lv_app /data
```

---

## 6. Summary Takeaways

- Command hierarchy: `pvcreate` -> `vgcreate` -> `lvcreate`.
- Always use `lvextend -r` to expand logical volumes and filesystems in one step without unmounting.
- Scale volume groups across new drives using `vgextend`.
- Use Copy-on-Write snapshots (`lvcreate -s`) for zero-risk software deployments and consistent backups.
