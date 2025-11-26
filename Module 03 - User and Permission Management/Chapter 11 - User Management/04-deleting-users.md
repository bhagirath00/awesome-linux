# Deleting Users in Linux: `userdel` and Offboarding Workflows

## Overview

Deleting a user account sounds straightforward, but in enterprise Linux systems and DevOps environments, improper account deletion can cause:
1. **Security Vulnerabilities (Orphaned Files)**: Files left behind with numerical UIDs that can be inherited by future users.
2. **Process Collisions**: Attempting to delete an account running background daemons.
3. **Ghost Cron Jobs**: Scheduled tasks lingering in `/var/spool/cron/crontabs`.

Understanding `userdel` and adhering to a structured offboarding checklist ensures user accounts and their associated resources are decommissioned cleanly and safely.

---

## 1. The `userdel` Command: Syntax and Flags

```bash
userdel [OPTIONS] USERNAME
```

### Essential Flags

| Flag | Long Option | Description | DevOps Practical Context |
| :--- | :--- | :--- | :--- |
| **`-r`** | `--remove` | **Removes the user's home directory and mail spool**. | **The standard flag for complete account removal.** |
| **`-f`** | `--force` | Forces removal of the user even if they are currently logged in or have active processes. | Emergency incident containment. |

---

## 2. The Danger of Deleting Without `-r`: Orphaned Files

If you run `userdel` without `-r`:
```bash
sudo userdel alex
```
1. `alex` is removed from `/etc/passwd`, `/etc/shadow`, and `/etc/group`.
2. **However, `/home/alex` and all files created by Alex remain on disk!**
3. The filesystem retains the numeric UID (e.g., `1002`). Because UID 1002 is no longer in `/etc/passwd`, `ls -l` displays the raw number:

```bash
$ ls -ld /home/alex
drwxr-xr-x 2 1002 1002 4096 Oct 14 09:00 /home/alex
```

### The Security Vulnerability:
Next month, you hire a contractor named `sarah`. You run `useradd sarah`. Linux assigns the next available UID: **`1002`**!

**Sarah now automatically inherits complete read, write, and execute ownership of all Alex's leftover private files, SSH keys, and tokens!**

### Auditing Orphaned Files:
Always scan for files belonging to deleted users:
```bash
# Find files whose UID does not match any current user in /etc/passwd:
find / -xdev -nouser
```

---

## 3. Resolving `user is currently used by process` Errors

If you attempt to delete a user who has active shell sessions, background jobs, or systemd user services running:

```bash
$ sudo userdel -r deployer
userdel: user deployer is currently used by process 18240
```

### The Clean Resolution:
Before running `userdel`, terminate all active processes owned by that user:

```bash
# Step 1: Inspect active processes:
ps -u deployer

# Step 2: Terminate all processes gracefully (SIGTERM):
sudo pkill -u deployer

# Step 3: If processes persist, force kill (SIGKILL):
sudo pkill -9 -u deployer

# Step 4: Now remove user and home directory cleanly:
sudo userdel -r deployer
```

---

## 4. The Production Offboarding Checklist Script

In automated environments, execute this standardized offboarding procedure:

```bash
#!/usr/bin/env bash
set -euo pipefail

TARGET_USER="$1"

echo "Beginning decommissioning of user: $TARGET_USER"

# Step 1: Lock account immediately to prevent new logins
usermod -L "$TARGET_USER"
usermod -s /usr/sbin/nologin "$TARGET_USER"

# Step 2: Terminate all active processes
pkill -u "$TARGET_USER" || true

# Step 3: Remove user crontab jobs
crontab -r -u "$TARGET_USER" 2>/dev/null || true

# Step 4: Archive user home directory for 30-day compliance retention
BACKUP_TAR="/mnt/backups/offboarded/${TARGET_USER}_$(date +%F).tar.gz"
mkdir -p /mnt/backups/offboarded
if [ -d "/home/$TARGET_USER" ]; then
    tar -czf "$BACKUP_TAR" "/home/$TARGET_USER"
    echo "User home directory archived to: $BACKUP_TAR"
fi

# Step 5: Remove sudoers privileges
rm -f "/etc/sudoers.d/99-$TARGET_USER"

# Step 6: Delete user and home directory
userdel -r "$TARGET_USER"

echo "User $TARGET_USER has been safely and completely decommissioned."
```

---

## Summary

- Use `userdel -r <user>` to delete an account along with its home directory.
- Running `userdel` without `-r` creates orphaned files with unmapped UIDs that future users can inherit.
- Terminate lingering user processes with `pkill -u <user>` before running `userdel`.
- Always archive the user's home directory and remove crontabs/sudoer permissions as part of a formal offboarding workflow.
- Audit for unowned files across the filesystem using `find / -xdev -nouser`.
