# The Linux Filesystem Hierarchy Standard (FHS)

## What is the FHS?

The **Filesystem Hierarchy Standard (FHS)** is the formal specification defining the directory structure and directory contents in Linux and Unix-like operating systems.

Because of the FHS, whether you are running Ubuntu on AWS, Rocky Linux on an on-premise server, or Alpine inside a Docker container, **configuration files are always in `/etc`, logs are always in `/var/log`, and binaries are always in `/usr/bin`**.

---

## The Complete Root (`/`) Directory Tree

In Linux, there are no drive letters (`C:\` or `D:\`). Everything branches from a single root node denoted by a forward slash **`/`**:

```text
/ (The Root Directory)
├── bin  -> usr/bin     (Essential user command binaries)
├── boot                (Kernel vmlinuz, GRUB bootloader, initramfs)
├── dev                 (Special device files: disks, ttys, /dev/null, /dev/urandom)
├── etc                 (System-wide host configuration files)
├── home                (Personal directories for standard users: /home/ubuntu)
├── lib  -> usr/lib     (Shared system libraries required by binaries)
├── lib64 -> usr/lib64  (64-bit shared C libraries)
├── media               (Automatic mount points for removable media: USBs, CD-ROMs)
├── mnt                 (Temporary manual mount point for filesystems)
├── opt                 (Optional third-party standalone vendor software)
├── proc                (Pseudo-filesystem exposing kernel state & processes)
├── root                (The home directory of the root superuser - NOT /home/root)
├── run                 (Volatile runtime data since boot: PIDs, sockets in tmpfs)
├── sbin -> usr/sbin    (Essential system administration binaries for root)
├── srv                 (Site-specific data served by this system)
├── sys                 (Pseudo-filesystem exposing kernel hardware device drivers)
├── tmp                 (Temporary scratchpad files, wiped on reboot)
├── usr                 (Secondary hierarchy: user utilities, docs, libraries)
└── var                 (Variable data: application logs, caches, spool, databases)
```

---

## Modern Linux Architecture: The Usr-Merge

In modern distributions (Ubuntu 20.04+, RHEL 9+, Debian 12+), you will notice that `/bin`, `/sbin`, and `/lib` are actually **symbolic links** pointing into `/usr`:

```bash
ls -ld /bin /sbin /lib /lib64
# Output:
# lrwxrwxrwx 1 root root 7 Apr 20 2024 /bin -> usr/bin
# lrwxrwxrwx 1 root root 8 Apr 20 2024 /sbin -> usr/sbin
# lrwxrwxrwx 1 root root 7 Apr 20 2024 /lib -> usr/lib
# lrwxrwxrwx 1 root root 9 Apr 20 2024 /lib64 -> usr/lib64
```

### Why was this done?
In the 1970s, disks were too small (e.g., 2MB), so Dennis Ritchie and Ken Thompson split binaries across `/bin` (needed for single-user recovery boot) and `/usr/bin` (mounted on a second drive). Today, all storage drives are gigabytes/terabytes, so modern Linux merged them into `/usr` for cleaner atomic system updates, snapshots, and container packaging.

---

## Exploring the Hierarchy via CLI

```bash
# 1. View the official Linux manual overview of the filesystem hierarchy
man 7 hier

# 2. View disk usage and mount points of top-level directories
df -h /

# 3. View immediate contents of root with file types and permissions
ls -lah /
```
