# Linux File Types and Attributes

## The Unix Philosophy: "Everything is a File"

One of the defining architectural pillars of Linux and Unix systems is that **everything is exposed as a file**:
- A document or executable is a file (`-`).
- A folder is a file containing names and inode references (`d`).
- A hard disk partition or SSD is a file (`b`).
- A terminal keyboard input or serial port is a file (`c`).
- An inter-process communication channel is a file (`s` or `p`).

This unified abstraction allows standard Linux utilities (`cat`, `read`, `write`, `grep`, `redirects`) to interact with disks, networks, devices, and memory using identical system calls.

---

## The 7 Linux File Types

When you run `ls -l`, the **very first character** of each line indicates the file type:

```text
-rw-r--r--   -> Regular file (-)
drwxr-xr-x   -> Directory (d)
lrwxrwxrwx   -> Symbolic link (l)
crw-rw-rw-   -> Character special device (c)
brw-rw----   -> Block special device (b)
srw-rw----   -> Socket (s)
prw-r--r--   -> Named pipe / FIFO (p)
```

### Detailed Breakdown of Each Type

| Type Symbol | File Type | Description & Real-World DevOps Example |
| :---: | :--- | :--- |
| **`-`** | **Regular File** | Text files, binary executables, shell scripts, images, logs, config files (`/etc/hosts`, `/usr/bin/python3`). |
| **`d`** | **Directory** | A special file containing an index list of filenames and associated inode pointer numbers (`/etc`, `/var/log`). |
| **`l`** | **Symbolic Link** | A soft pointer to another file or path (`/bin -> usr/bin`, version alias `/usr/bin/python -> python3.11`). |
| **`c`** | **Character Device** | Provides direct, unbuffered stream access to hardware, transmitting byte-by-byte (`/dev/null`, `/dev/urandom`, `/dev/tty`). |
| **`b`** | **Block Device** | Communicates with storage hardware in fixed-size buffered data blocks (`/dev/sda`, `/dev/nvme0n1p1`, `/dev/loop0`). |
| **`s`** | **UNIX Domain Socket** | High-performance bidirectional local Inter-Process Communication (IPC) endpoint (`/var/run/docker.sock`). |
| **`p`** | **Named Pipe (FIFO)** | Unidirectional First-In, First-Out queue allowing two distinct processes to exchange streaming data (`mkfifo mypipe`). |

---

## Inspecting File Types via CLI

```bash
# 1. Regular File
$ ls -l /etc/resolv.conf
-rw-r--r-- 1 root root 82 Oct 14 00:00 /etc/resolv.conf

# 2. Directory
$ ls -ld /var/log
drwxr-xr-x 12 root root 4096 Oct 14 00:00 /var/log

# 3. Symbolic Link
$ ls -l /bin
lrwxrwxrwx 1 root root 7 Apr 20 2024 /bin -> usr/bin

# 4. Character Device (/dev/null, /dev/random)
$ ls -l /dev/null /dev/urandom
crw-rw-rw- 1 root root 1, 3 Oct 14 00:00 /dev/null
crw-rw-rw- 1 root root 1, 9 Oct 14 00:00 /dev/urandom

# 5. Block Device (Disks & NVMe drives)
$ ls -l /dev/sda /dev/nvme0n1
brw-rw---- 1 root disk 8, 0 Oct 14 00:00 /dev/sda
brw-rw---- 1 root disk 259, 0 Oct 14 00:00 /dev/nvme0n1

# 6. Unix Domain Socket (Docker daemon socket)
$ ls -l /var/run/docker.sock
srw-rw---- 1 root docker 0 Oct 14 00:00 /var/run/docker.sock
```

---

## The Myth of File Extensions in Linux

In Windows, the operating system relies strictly on file extensions (`.exe`, `.txt`, `.docx`) to decide how to execute or open a file.

In **Linux, file extensions are purely optional visual conventions for human users**. A file named `backup` without any `.tar.gz` extension, or even a file named `script.jpg`, can be an executable ELF binary or a Bash script.

### How Linux Determines What a File Is

Linux determines a file's actual format using:
1. **Magic Numbers / Magic Bytes**: The initial bytes stored in the file header (e.g., `\x7fELF` for Linux binaries, `\x89PNG` for PNG images, `PK\x03\x04` for ZIP archives).
2. **Shebang (`#!`) Line**: The first line of text scripts declaring the interpreter (e.g., `#!/bin/bash` or `#!/usr/bin/env python3`).
3. **Execution Permissions (`chmod +x`)**: A file will only execute if its permission mode has the execute bit set.

### Identifying Real File Formats with `file`

The `file` utility inspects the magic bytes and contents of any file regardless of its extension:

```bash
# Even if renamed to a misleading extension, file reveals the truth:
$ cp /bin/ls script.txt
$ file script.txt
script.txt: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked

$ file /var/log/syslog
/var/log/syslog: UTF-8 Unicode text

$ file archive.tar.gz
archive.tar.gz: gzip compressed data, was "backup.tar", last modified: Sun Oct 12 10:14:00 2026
```

---

## Linux Extended File Attributes (`chattr` & `lsattr`)

Beyond traditional read/write/execute permissions (`chmod`), Linux filesystems (ext4, XFS) support **extended file attributes** managed by the kernel.

These attributes provide powerful security and system integrity mechanisms for DevOps engineers and system administrators:

| Attribute Flag | Name | Functionality |
| :---: | :--- | :--- |
| **`i`** | **Immutable** | The file **cannot** be modified, deleted, renamed, or linked to, even by the **root superuser**. |
| **`a`** | **Append-Only** | The file can **only** have data appended to the end; existing content cannot be deleted or overwritten. Essential for secure audit logs. |
| **`s`** | **Secure Deletion** | When deleted, blocks are zeroed out on disk immediately. |
| **`c`** | **Compressed** | The kernel automatically compresses file blocks before writing to disk. |

### Practical DevOps Use Cases

#### 1. Protecting Critical Config Files with the Immutable Flag (`+i`)
Prevent automated scripts, accidental deletions, or unauthorized attackers from modifying `/etc/resolv.conf` or `/etc/passwd`:

```bash
# Make file immutable (even root cannot modify or rm -rf)
sudo chattr +i /etc/resolv.conf

# Verify attributes
lsattr /etc/resolv.conf
# Output: ----i---------e------- /etc/resolv.conf

# Attempting to delete returns:
# rm: cannot remove '/etc/resolv.conf': Operation not permitted

# To unlock and edit when authorized:
sudo chattr -i /etc/resolv.conf
```

#### 2. Tamper-Proof Audit Logging with Append-Only (`+a`)
Ensure that monitoring and security audit logs cannot be purged or truncated by attackers:

```bash
# Set audit log to append-only mode
sudo chattr +a /var/log/audit.log

# Appending new log entries works:
echo "user login event" >> /var/log/audit.log

# Overwriting or deleting fails:
rm /var/log/audit.log
# Output: Operation not permitted
```
