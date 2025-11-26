# SUID and SGID in Depth: Architecture, Mechanics, and Team Collaboration

## Overview

**SUID (Set User ID)** and **SGID (Set Group ID)** are advanced permission mechanisms in Linux that alter how the kernel assigns process credentials and directory group ownership.

Understanding SUID and SGID is critical for:
- Understanding how regular users execute privileged system tasks (like `/usr/bin/passwd`).
- Architecting shared team collaboration spaces where multiple developers read and write to the same directory without permission drift.
- Security hardening: preventing privilege escalation attacks across container mounts and storage partitions.

---

## 1. Deep Dive: SUID (Set User ID)

When a regular user runs an executable binary, the resulting process normally runs with that user's own identity (**Real UID = Effective UID**).

When **SUID** is set on an executable file owned by `root`, the kernel temporarily sets the process's **Effective UID (EUID) to `root` (UID 0)** during execution!

```text
User 'alex' (UID 1001) executes /usr/bin/passwd (Owned by root, SUID bit active)
  │
  ▼
Linux Kernel assigns credentials to running process:
  ├── Real UID (RUID):      1001 (alex)
  └── Effective UID (EUID): 0 (root)  <-- Grants permission to write to /etc/shadow!
```

### Setting and Removing SUID
```bash
# Numeric mode (4xxx):
sudo chmod 4755 /usr/local/bin/custom-tool

# Symbolic mode (u+s):
sudo chmod u+s /usr/local/bin/custom-tool

# Remove SUID:
sudo chmod u-s /usr/local/bin/custom-tool
```

> [!WARNING]
> **Linux Ignores SUID on Shell Scripts!**
> On Linux, the kernel deliberately ignores the SUID bit on interpreted scripts (`#!/bin/bash`, `#!/usr/bin/python`). Because scripts are subject to environment variable manipulation and symlink race conditions, allowing SUID on scripts would create massive security vulnerabilities. SUID only takes effect on **compiled binary executables** (C, Go, Rust).

---

## 2. Deep Dive: SGID (Set Group ID)

SGID behaves differently depending on whether it is applied to an **executable file** or a **directory**.

### A. SGID on Executable Files
When executed, the process runs with the **Effective Group ID (EGID)** of the file's assigned group rather than the calling user's group (e.g., `/usr/bin/wall` running with `tty` group privileges).

### B. SGID on Directories: The DevOps Collaboration Standard
When **SGID is set on a directory**:
1. Any new file created inside that directory **automatically inherits the group ownership of the directory**, rather than the primary group of the user creating the file!
2. Any new subdirectory created inside **automatically inherits the SGID bit itself**, propagating the behavior down the entire tree!

```text
WITHOUT SGID on /data/shared (Group: devteam):
  User 'alex' (primary group 'alex') creates file ──> file owned by alex:alex (Group is NOT devteam!)
  User 'sarah' (primary group 'sarah') CANNOT EDIT IT! (Collision!)

WITH SGID (chmod 2775) on /data/shared (Group: devteam):
  User 'alex' creates file ──> file owned by alex:devteam!
  User 'sarah' creates file ──> file owned by sarah:devteam!
  (Both can read and edit each other's files seamlessly!)
```

---

## 3. Hands-on Tutorial: Provisioning an Enterprise Collaborative Directory

Here is the production-tested recipe for configuring a shared workspace for team `engineering`:

```bash
#!/usr/bin/env bash
set -euo pipefail

SHARED_DIR="/opt/engineering-workspace"
GROUP_NAME="engineering"

# Step 1: Create the team group
groupadd -f "$GROUP_NAME"

# Step 2: Create directory and set group ownership
mkdir -p "$SHARED_DIR"
chown root:"$GROUP_NAME" "$SHARED_DIR"

# Step 3: Grant Read/Write/Execute to Owner and Group (775)
chmod 775 "$SHARED_DIR"

# Step 4: Enable the SGID bit (2775 or g+s)
chmod g+s "$SHARED_DIR"

# Step 5: Verify permissions
ls -ld "$SHARED_DIR"
# Output: drwxrwsr-x 2 root engineering 4096 Oct 14 10:45 /opt/engineering-workspace
```
*(Notice the `s` in the group position: `drwxrwsr-x`)*.

---

## 4. Security Hardening: The `nosuid` Mount Option

Because SUID binaries are high-value targets for attackers, security baselines (CIS Benchmarks) dictate that **untrusted partitions must disable SUID execution at the filesystem mount level**.

In `/etc/fstab`, always append the `nosuid` option to temporary or user-writable mount points:

```ini
# /etc/fstab hardening:
/dev/sdb1    /tmp         ext4    defaults,nosuid,nodev,noexec    0 2
/dev/sdc1    /home        ext4    defaults,nosuid,nodev           0 2
/dev/sdd1    /var/tmp     ext4    defaults,nosuid,nodev           0 2
```

When `nosuid` is mounted, the Linux kernel treats all SUID/SGID bits as normal standard permissions, neutralizing privilege escalation attacks completely.

---

## Summary

- SUID (`chmod 4755` or `u+s`) executes binary programs with the privileges of the file owner.
- Linux deliberately ignores SUID on interpreted shell and Python scripts.
- SGID on directories (`chmod 2775` or `g+s`) forces all newly created files to inherit the parent directory's group.
- SGID on directories automatically propagates to all newly created subdirectories.
- Protect untrusted filesystems (`/tmp`, `/home`) against malicious SUID escalation using the `nosuid` mount option in `/etc/fstab`.
