# Chapter 26: Disk and Storage Management - Partitioning Disks (Step-by-Step)

## 1. Introduction: Partitioning Workflow

Partitioning is the process of carving a raw storage block device (such as an attached AWS EBS volume, a SAN LUN, or a physical NVMe SSD) into logical boundary segments called **partitions**. 

This guide demonstrates two industry workflows:
1. **Interactive Partitioning** using `fdisk`.
2. **Automated Non-Interactive Partitioning** using `parted` (for CI/CD pipelines, Cloud-Init, and Ansible).

---

## 2. Interactive Step-by-Step Partitioning with fdisk

Let's partition a new unformatted 100GB disk: `/dev/sdb`.

### Step 1: Open Target Disk
```bash
sudo fdisk /dev/sdb
```

### Step 2: Initialize a Modern GPT Partition Table
Type **`g`** to create an empty GPT disklabel:
```
Command (m for help): g
Created a new GPT disklabel (GUID: B4125890-E142-4F12-8901-7A1248102394).
```

### Step 3: Create the First Partition
Type **`n`** for new partition:
```
Command (m for help): n
Partition number (1-128, default 1): <Press Enter>
First sector (2048-209715166, default 2048): <Press Enter>
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-209715166, default 209715166): +50G

Created a new partition 1 of type 'Linux filesystem' and of size 50 GiB.
```

### Step 4: Verify Partition Table in Memory
Type **`p`** to print the table:
```
Device       Start       End   Sectors Size Type
/dev/sdb1     2048 104859647 104857600  50G Linux filesystem
```

### Step 5: Write Changes to Disk
Type **`w`** to save:
```
Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

---

## 3. Automated Non-Interactive Partitioning with `parted`

In DevOps automation, scripts cannot pause for interactive keyboard prompts. The **`parted`** utility is designed for non-interactive scripting:

```bash
# 1. Label the disk with GPT in one command
sudo parted -s /dev/sdb mklabel gpt

# 2. Create a 50GB ext4 partition aligned to 1MiB boundaries
sudo parted -s /dev/sdb mkpart primary ext4 1MiB 50GiB

# 3. Create a second partition using remaining space
sudo parted -s /dev/sdb mkpart primary ext4 50GiB 100%

# 4. Print partition table
sudo parted /dev/sdb print
```

```
Model: VirtIO Block Device (virtblk)
Disk /dev/sdb: 107GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name     Flags
 1      1049kB  53.7GB  53.7GB               primary
 2      53.7GB  107GB   53.7GB               primary
```

---

## 4. Deleting a Partition

### With `fdisk`:
Open the disk (`fdisk /dev/sdb`), type **`d`**, enter the partition number to remove, and type **`w`** to commit.

### With `parted`:
```bash
# Delete partition number 2 immediately
sudo parted -s /dev/sdb rm 2
```

---

## 5. Summary Takeaways

- Initialize modern disks with GPT (`g` in fdisk or `mklabel gpt` in parted).
- Always use human-readable size notation (`+50G` or `50GiB`).
- Ensure partition boundaries are aligned to 1MiB (sector 2048) for optimal performance.
- Use `parted -s` for unattended CI/CD automation and configuration management scripts.
