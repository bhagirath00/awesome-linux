# Chapter 26: Disk and Storage Management - The /etc/fstab Configuration File

## 1. Introduction: Why /etc/fstab?

Mounts created with the `mount` command are temporary; they exist only in the running kernel's memory and are lost when the system reboots.

To make filesystems, external data volumes, swap partitions, and network NFS shares **mount automatically and permanently on system boot**, they must be defined in **`/etc/fstab`** (**File Systems Table**).

During early boot, `systemd` translates every entry in `/etc/fstab` into dynamic `.mount` and `.swap` units (`systemd-fstab-generator`) and mounts them before launching background user-space daemons.

---

## 2. Anatomy of an /etc/fstab Entry

The `/etc/fstab` file is composed of lines containing **six whitespace-delimited fields**:

```
# <device/UUID>                          <mount_point> <type> <options>       <dump> <pass>
UUID=9e32a410-b981-4210-8910-c412581023 /var/data     ext4   defaults,noatime 0      2
---------------------------------------- ---------     ----   ---------------- -      -
                Field 1                   Field 2    Field 3       Field 4   Field 5 Field 6
```

---

## 3. Detailed Breakdown of the Six Fields

| Field | Header | Description | Typical Production Examples |
| :--- | :--- | :--- | :--- |
| **1** | **Device / UUID** | The block device to be mounted. **Always use UUID**, never raw device nodes (`/dev/sda1`), because device node letters can shuffle after reboot or disk hot-plugging! | `UUID=4f31...`<br>`LABEL=DATA`<br>`/dev/mapper/vg-lv` |
| **2** | **Mount Point** | The target directory where the filesystem will be attached. Must be an existing directory (except for swap). | `/`<br>`/boot`<br>`/mnt/data`<br>`none` (for swap) |
| **3** | **Filesystem Type** | Format driver of the filesystem. | `ext4`, `xfs`, `btrfs`, `swap`, `nfs` |
| **4** | **Mount Options** | Comma-separated list of filesystem behavior flags. Default is usually `defaults` (`rw, suid, dev, exec, auto, nouser, async`). | `defaults,noatime,nofail` |
| **5** | **Dump Flag** | Used by the legacy `dump` backup tool. Almost universally set to `0` (disabled) today. | `0` (do not dump) |
| **6** | **Fsck Pass** | Dictates the order in which `fsck` checks filesystems at boot:<br>• **`1`**: Root filesystem (`/`) (checked first).<br>• **`2`**: Other local partitions.<br>• **`0`**: Do not check (used for swap, XFS, and network NFS). | `1` (Root)<br>`2` (Data)<br>`0` (XFS/Swap) |

---

## 4. Critical Mount Options for Production & Cloud

- **`defaults`**: Applies standard settings (`rw, suid, dev, exec, auto, nouser, async`).
- **`noatime`**: **Massive Performance Optimization**. Prevents the kernel from writing access timestamps to disk every time a file is read. Highly recommended for web assets, databases, and container storage.
- **`nofail`**: **Life-Saver for Cloud Volumes**. By default, if a disk listed in `/etc/fstab` is missing or unplugged during boot, **systemd halts the entire server and drops into emergency rescue mode!** Adding `nofail` ensures that if an external AWS EBS volume or backup drive is unavailable, the server boots normally without hanging.
- **`_netdev`**: Designates that this filesystem requires network connectivity (MANDATORY for NFS, iSCSI, and Ceph mounts). Prevents systemd from attempting to mount before the network stack is up.

---

## 5. Finding UUIDs with `blkid` and `lsblk`

Never guess device UUIDs. Retrieve them directly:

```bash
sudo blkid /dev/sdb1
```

```
/dev/sdb1: UUID="9e32a410-b981-4210-8910-c412581023a4" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="b4125890-01"
```

---

## 6. Testing /etc/fstab Before Rebooting (MANDATORY RULE)

> [!CAUTION]
> **A syntax error, typo, or missing UUID in `/etc/fstab` can make your Linux server unbootable!**
> 
> Never reboot a server after editing `/etc/fstab` without validating it first using **`mount -a`**:
> ```bash
> sudo mount -a
> ```
> `mount -a` attempts to mount **all** filesystems defined in `/etc/fstab` that are not already mounted.
> - If `mount -a` returns **silently with no errors**, your configuration is syntactically sound and safe to reboot!
> - If it prints any error message, fix `/etc/fstab` immediately before restarting the system!

---

## 7. Complete Production /etc/fstab Example

```fstab
# /etc/fstab: static file system information.

# Root partition on NVMe SSD (ext4, checked first)
UUID=312048f1-a124-4f81-9981-e41258102914 /               ext4    noatime,errors=remount-ro 0 1

# EFI System Partition (FAT32, no fsck)
UUID=7A8B-C102                            /boot/efi       vfat    umask=0077               0 0

# Swap Space
UUID=d4128410-b981-4123-aa81-812481023912 none            swap    sw                       0 0

# Production PostgreSQL Database Volume (XFS, nofail for cloud safety)
UUID=c8149102-3141-4891-b124-a12481023812 /var/lib/pgsql  xfs     defaults,noatime,nofail  0 0

# Remote NFS Backup Storage Share
nas.company.internal:/backups             /mnt/nas        nfs     defaults,_netdev,nofail  0 0
```

---

## 8. Summary Takeaways

- `/etc/fstab` configures permanent filesystems mounted on boot.
- Always use `UUID=` instead of `/dev/sdX` device names.
- Six fields: Device, Mount Point, Type, Options, Dump (0), Pass (1 for root, 2 for local, 0 for XFS/swap).
- Use `noatime` for I/O performance and `nofail` for non-critical cloud disks.
- **Always verify changes with `sudo mount -a` before rebooting!**
