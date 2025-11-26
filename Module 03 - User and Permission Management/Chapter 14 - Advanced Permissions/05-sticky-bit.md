# The Sticky Bit in Linux: Shared Directory Security and Protection

## Overview

In multi-user Linux operating systems, systems frequently require public, shared "scratchpad" directories where any user, container, or background process can create temporary files, sockets, and lock files.

However, if a directory is configured with world-write permissions (`rwxrwxrwx` / `777`), standard POSIX rules dictate that **any user can delete or rename ANY file inside that directory**, regardless of who owns it!

The **Sticky Bit** solves this critical security dilemma. When applied to a directory, the Sticky Bit ensures that:
> **Only the File Owner, the Directory Owner, or the `root` Superuser can delete, unlink, or rename a file.**

---

## 1. The Archetype Example: `/tmp` and `/var/tmp`

If you inspect the root temporary directory on any Linux system:

```bash
$ ls -ld /tmp /var/tmp
drwxrwxrwt 22 root root 4096 Oct 14 10:50 /tmp
drwxrwxrwt  3 root root 4096 Oct 14 00:00 /var/tmp
```

Notice the **`t`** at the very end of the permission string: `drwxrwxrwt`.
- **`rwx`** (Owner: root)
- **`rwx`** (Group: root)
- **`rwt`** (Others: World can read, write, and traverse, BUT the **Sticky Bit `t`** prevents users from deleting each other's files!).

---

## 2. Command Syntax: Setting and Removing the Sticky Bit

The Sticky Bit has a numerical octal weight of **`1`**:

```bash
# Numeric Mode: 1777 (Sticky + Full Access)
sudo chmod 1777 /var/shared-uploads

# Symbolic Mode (+t):
sudo chmod +t /var/shared-uploads
# Or:
sudo chmod o+t /var/shared-uploads

# Remove Sticky Bit:
sudo chmod -t /var/shared-uploads
```

### Understanding `t` vs `T`
- **Lowercase `t`** : Sticky bit active **AND** Others execute (`x`) is active (`rwxrwxrwt`). *(Standard)*.
- **Uppercase `T`** : Sticky bit active, **BUT** Others execute (`x`) is missing (`rwxrwxrwT`).

---

## 3. Hands-on Security Lab: Proving the Sticky Bit

To observe why the Sticky Bit is vital, perform this test:

### Experiment A: Directory WITHOUT Sticky Bit (`0777`)
```bash
# Step 1: Create world-writable folder without sticky bit
sudo mkdir /tmp/vulnerable
sudo chmod 777 /tmp/vulnerable

# Step 2: User 'alex' creates an important file
sudo -u alex touch /tmp/vulnerable/alex-secret.txt

# Step 3: Malicious user 'bob' tries to delete Alex's file
sudo -u bob rm /tmp/vulnerable/alex-secret.txt
# Output: DELETED! (Bob destroyed Alex's file!)
```

### Experiment B: Directory WITH Sticky Bit (`1777`)
```bash
# Step 1: Enable Sticky Bit
sudo chmod +t /tmp/vulnerable

# Step 2: User 'alex' creates file again
sudo -u alex touch /tmp/vulnerable/alex-secret.txt

# Step 3: Malicious user 'bob' tries to delete it:
sudo -u bob rm /tmp/vulnerable/alex-secret.txt
# Output:
# rm: cannot remove '/tmp/vulnerable/alex-secret.txt': Operation not permitted
```
**The kernel blocked Bob!** Even though the directory is world-writable, only Alex (the owner) or root can remove the file.

---

## 4. Auditing Sticky Bit Directories

To verify that all world-writable directories on your server have the Sticky Bit properly enabled:

```bash
# Find any world-writable directory that is MISSING the sticky bit (Security Risk!):
find / -xdev -type d -perm -0002 ! -perm -1000 -exec ls -ld {} +
```
*(If this command produces any output, those directories are vulnerable and should be remediated immediately with `chmod +t`).*

---

## Summary

- The Sticky Bit (octal `1`, indicator `t`) prevents users from deleting or renaming files they do not own in shared writable directories.
- Essential for public scratch spaces like `/tmp`, `/var/tmp`, and public upload staging directories.
- Applied numerically via `chmod 1777 <dir>` or symbolically via `chmod +t <dir>`.
- Audit systems to ensure all world-writable directories enforce the sticky bit.
