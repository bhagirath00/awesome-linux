# Chapter 26: Disk and Storage Management - The lsblk Command

## 1. Introduction: What is lsblk?

`lsblk` (**List Block Devices**) reads the `/sys` virtual filesystem and the `udev` database to gather information about all available or specified block devices. It displays its output in an intuitive, hierarchical tree format that visually depicts the relationship between physical disks, hardware partitions, LVM logical volume layers, and filesystem mount points.

Unlike legacy commands like `fdisk -l` (which requires `root` privileges and produces verbose unstructured text), `lsblk` can be executed by **any unprivileged user** and provides clear, immediate insight into system storage layout.

---

## 2. Standard Output Layout

Running `lsblk` without arguments outputs a visual block tree:

```bash
lsblk
```

```
NAME                    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda                       8:0    0    50G  0 disk 
├─sda1                    8:1    0     1G  0 part /boot
└─sda2                    8:2    0    49G  0 part 
  ├─ubuntu--vg-ubuntu--lv 253:0  0    30G  0 lvm  /
  └─ubuntu--vg-swap--lv   253:1  0     4G  0 lvm  [SWAP]
sdb                       8:16   0   200G  0 disk 
└─sdb1                    8:17   0   200G  0 part /data
nvme0n1                 259:0    0     1T  0 disk 
└─nvme0n1p1             259:1    0     1T  0 part /var/lib/docker
sr0                      11:0    1  1024M  0 rom  
```

### Column Meanings:
- **`NAME`**: The device name (indented tree shows partition and LVM hierarchy).
- **`MAJ:MIN`**: Major and Minor device numbers (identifies device driver and instance in `/dev/`).
- **`RM`**: Removable media indicator (`1` = removable like USB/CDROM, `0` = fixed internal drive).
- **`SIZE`**: Total capacity of the block device.
- **`RO`**: Read-only indicator (`1` = read-only, `0` = read-write).
- **`TYPE`**: Device type (`disk`, `part`, `lvm`, `rom`, `crypt`, `loop`).
- **`MOUNTPOINTS`**: Active directory path where the filesystem is attached.

---

## 3. The Power Flags for DevOps Engineers

### 1. The Filesystem Identification Flag (`-f`)
The **`-f`** flag prints the **Filesystem Type**, **Label**, and universally unique identifier (**UUID**):

```bash
lsblk -f
```

```
NAME                    FSTYPE      FSVER LABEL  UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
sda                                                                                                  
├─sda1                  ext4        1.0   BOOT   58f3c4e2-9b21-4f1a-b312-d812481023a1  750M    21% /boot
└─sda2                  LVM2_member LVM2         7a82b9-e123-f321-4451-9981-d1284a123                
  ├─ubuntu--vg-ubuntu--lv ext4      1.0          312048f1-a124-4f81-9981-e41258102914   18G    38% /
  └─ubuntu--vg-swap--lv swap        1            d4128410-b981-4123-aa81-812481023912                [SWAP]
sdb                                                                                                  
└─sdb1                  xfs               DATA   c8149102-3141-4891-b124-a12481023812  119G    41% /data
```

> [!TIP]
> Always use `lsblk -f` to look up the filesystem **UUID** when configuring persistent mounts in `/etc/fstab`!

### 2. Custom Output Columns (`-o`)
In automated Ansible scripts or monitoring pipelines, select exact columns:
```bash
lsblk -o NAME,SIZE,FSTYPE,TYPE,MOUNTPOINTS
```

### 3. Machine-Readable JSON Output (`-J`)
Cloud management scripts and Python microservices can parse disk layout effortlessly via JSON:
```bash
lsblk -J -f
```

### 4. Raw Output for Shell Script Parsing (`-r` or `-P`)
Pairs keys and values for easy shell parsing via `eval`:
```bash
lsblk -P -o NAME,SIZE,MOUNTPOINT
```

```
NAME="sda" SIZE="50G" MOUNTPOINT=""
NAME="sda1" SIZE="1G" MOUNTPOINT="/boot"
NAME="sda2" SIZE="49G" MOUNTPOINT=""
```

---

## 4. Summary Takeaways

- `lsblk` displays block devices, partitions, LVM layers, and mounts in a visual tree.
- Does not require `sudo` or `root` permissions for read operations.
- Always run `lsblk -f` to obtain partition filesystem types and UUIDs.
- Use `-J` for clean JSON integration in DevOps automation tools.
