# Access Control Lists (ACLs) in Linux: Architecture and Beyond Standard DAC

## Overview

Standard Linux Discretionary Access Control (DAC)—the classic User, Group, Others (`ugo`) permission model—is simple and robust. However, in enterprise environments, standard DAC quickly reaches its limits.

Consider a real-world scenario:
- Developer `alex` owns `/var/www/app/config.json`.
- Group `developers` needs **Read and Write** access.
- Auditor `maria` from compliance needs **Read-Only** access.
- Automation user `ci-deployer` needs **Read and Write** access.
- Everyone else on the server must have **Zero** access.

**Under standard Linux DAC, this is mathematically impossible!** You can only define *one* user and *one* group. To give Maria access, you would either have to add her to `developers` (violating least privilege by granting her write access) or grant read access to Others (exposing the file to everyone on the machine).

**Access Control Lists (POSIX ACLs)** solve this problem by allowing administrators to assign fine-grained permissions to **multiple arbitrary users and multiple arbitrary groups** on a single file or directory.

---

## 1. How ACLs Work: The Extended Attribute Layer

POSIX.1e ACLs are stored directly within the filesystem's **Extended Attributes (xattr)** alongside the file's standard inode metadata.

```text
================ STANDARD PERMISSION MODEL (UGO) ================
Owner: alex (rw-)  │  Group: developers (r--)  │  Others: (---)


================ ACCESS CONTROL LIST (POSIX ACL) ================
Owner:           alex          -> rw-
Named User:      maria         -> r--   <-- Specific named user!
Named User:      ci-deployer   -> rwx   <-- Another named user!
Primary Group:   developers    -> rw-
Named Group:     auditors      -> r--   <-- Specific named group!
Mask:                          -> rwx   <-- Maximum allowable effective limit
Others:                        -> ---
```

---

## 2. The Plus Sign Indicator in `ls -l`

How do you know if a file has an active Access Control List? Look at the **11th character** of the `ls -l` permission string:

```bash
$ ls -l /var/www/app/config.json
-rw-rw----+ 1 alex developers 1024 Oct 14 10:55 /var/www/app/config.json
          ▲
          └─ The '+' symbol indicates an active Extended ACL!
```

- If a file ends with a space or dot `.`, it has standard DAC permissions or basic SELinux context.
- If a file ends with **`+`**, an **Extended ACL** is attached to the inode!

---

## 3. Base ACLs vs Extended ACLs

- **Base ACL Entries**: Exactly match standard Unix permissions:
  1. `user::rwx` (File owner)
  2. `group::rwx` (Owning group)
  3. `other::rwx` (World)
- **Extended ACL Entries**: Any additional specific rules:
  1. `user:username:rwx` (Named user)
  2. `group:groupname:rwx` (Named group)
  3. `mask::rwx` (Defines the upper bound of permissions for all named users and groups)

---

## 4. Filesystem Support and Prerequisites

On modern Linux distributions (Ubuntu 20.04+, RHEL 8+, Debian 11+), all major filesystems—**ext4, XFS, Btrfs, and ZFS**—have ACL support enabled natively in the kernel by default.

Verify your filesystem mount options:
```bash
$ tune2fs -l /dev/sda1 | grep "Default mount options"
Default mount options:    user_xattr acl
```

The userland management tools for ACLs are packaged in the `acl` package:
```bash
sudo apt install acl     # Debian/Ubuntu
sudo dnf install acl     # RHEL/Rocky/Fedora
```

---

## Summary

- Standard DAC limits files to a single owner, single group, and others.
- Access Control Lists (ACLs) allow binding permissions to multiple arbitrary named users and named groups on a single file.
- An active ACL is identified by a **plus sign (`+`)** at the end of the `ls -l` mode string (`-rw-r--r--+`).
- ACLs are natively supported on modern ext4 and XFS filesystems.
- ACL inspection and management are handled via `getfacl` and `setfacl`.
