# Chapter 26: Disk and Storage Management - The fdisk Command

## 1. Introduction: What is fdisk?

`fdisk` (**Format Disk** / **Fixed Disk**) is a dialogue-driven, command-line utility used for creating and manipulating disk partition tables. 

Historically limited to MBR (Master Boot Record) partition tables, modern Linux versions of `fdisk` (from `util-linux`) fully support both modern **GPT (GUID Partition Table)** and legacy MBR tables.

> [!NOTE]
> `fdisk` operates in **interactive menu mode**. Changes made during an `fdisk` session are held strictly in temporary RAM. The physical partition table on disk is **not touched or modified until you explicitly issue the `w` (write) command**. If you make a mistake, simply type `q` to quit without saving!

---

## 2. Listing Partitions with `fdisk -l`

To inspect partition tables across all attached storage disks:

```bash
sudo fdisk -l
```

```
Disk /dev/sda: 50 GiB, 53687091200 bytes, 104857600 sectors
Disk model: QEMU HARDDISK   
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 7A8B9C10-1234-4567-89AB-CDEF01234567

Device       Start       End   Sectors Size Type
/dev/sda1     2048   2099199   2097152   1G EFI System
/dev/sda2  2099200 104855551 102756352  49G Linux filesystem
```

### Key Technical Attributes:
- **`Disklabel type`**: Identifies whether the disk is formatted as `gpt` or `dos` (MBR).
- **`Sector size`**: Typically 512 bytes (logical) or 4096 bytes (4K Advanced Format).
- **`Start / End Sectors`**: Partition boundary alignments. Modern partitions start at sector `2048` (1 MiB alignment) to optimize SSD read/write block performance.

---

## 3. Interactive Menu Navigation

To partition a target raw disk (e.g., `/dev/sdb`), open it with `fdisk`:

```bash
sudo fdisk /dev/sdb
```

```
Welcome to fdisk (util-linux 2.37.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Command (m for help): 
```

### Essential Interactive Single-Key Commands:
| Key | Action | Description |
| :--- | :--- | :--- |
| **`m`** | Menu / Help | Prints the full list of available commands and keybindings. |
| **`p`** | Print Table | Prints the current partition table currently held in memory. |
| **`n`** | New Partition | Prompts to create a new partition (specifying number, start, and size). |
| **`d`** | Delete Partition | Prompts for partition number to delete. |
| **`t`** | Change Type | Changes partition system type (e.g., Linux filesystem, Linux LVM, Swap). |
| **`g`** | Create GPT Label | Creates a brand-new, empty GPT partition table. |
| **`o`** | Create MBR Label | Creates a brand-new, empty MBR (DOS) partition table. |
| **`w`** | **Write and Exit** | **Commits changes permanently to disk** and exits. |
| **`q`** | **Quit without Saving**| Discards all pending changes in memory and exits safely. |

---

## 4. Kernel Partition Table Refresh: `partprobe`

When you write a new partition table with `w`, the kernel attempts to reread the partition table immediately. However, if any partition on that disk is currently mounted or in use, the kernel displays:

```
Re-reading the partition table failed.: Device or resource busy
The kernel still uses the old table. The new table will be used at the next reboot.
```

Instead of rebooting the production server, force the Linux kernel to rescan the storage bus and reload the partition table into memory using **`partprobe`**:

```bash
sudo partprobe /dev/sdb
```

Verify that the new device node (e.g., `/dev/sdb1`) now appears in `/dev/` and `lsblk`.

---

## 5. Summary Takeaways

- `fdisk` is the standard interactive partition table editor in Linux.
- Use `fdisk -l` to audit disk labels (`gpt` vs `dos`), sector boundaries, and sizes.
- Changes are held in RAM until committed with `w`; exit safely with `q`.
- Always verify 1MiB sector alignment (starting at sector 2048) on SSDs.
- Use `partprobe` to reload partition tables into the live kernel without rebooting.
