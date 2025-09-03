# Special Permissions in Linux: SUID, SGID, and the Sticky Bit

## Overview

Standard POSIX permissions (`rwx` for User, Group, Others) solve 90% of access control needs. However, complex multi-user systems and collaborative environments present challenges that standard permissions cannot address:
- How can an unprivileged regular user change their password if `/etc/shadow` is strictly writable by `root` only?
- How can multiple developers work in a shared directory without constantly re-assigning group ownership on newly created files?
- How can a public temporary directory like `/tmp` be world-writable without allowing users to maliciously delete each other's files?

To solve these problems, Linux implements **Three Special Permission Bits**:
1. **SUID (Set User ID)**
2. **SGID (Set Group ID)**
3. **The Sticky Bit**

---

## 1. The Three Special Bits at a Glance

| Special Bit | Octal Value | Indicator in `ls -l` | Applied to Files | Applied to Directories |
| :--- | :---: | :---: | :--- | :--- |
| **SUID** *(SetUID)* | **`4`** | Replaces Owner `x` with **`s`** (`rwsr-xr-x`) | Executable runs with permissions of the **file Owner**, not the calling user. | *(Ignored by Linux kernel)*. |
| **SGID** *(SetGID)* | **`2`** | Replaces Group `x` with **`s`** (`rwxr-sr-x`) | Executable runs with permissions of the **file Group**. | **New files created inside inherit the parent directory's group automatically!** |
| **Sticky Bit** | **`1`** | Replaces Others `x` with **`t`** (`rwxrwxrwt`) | *(Obsolete on modern Linux files)*. | **Only the file owner or root can delete files inside**, even if directory is world-writable. |

---

## 2. The 4-Digit Octal Notation

When applying special permissions, prepend an extra **4th leading digit** to the standard 3-digit octal mode:

$$\text{Special Digit} = \text{SUID}(4) + \text{SGID}(2) + \text{Sticky}(1)$$

```text
Special Mode:  4          7          5          5
               │          │          │          │
               │          └──────────┴──────────┴─ Standard Mode (rwxr-xr-x)
               └─ SUID Active (4)
```

### High-Visibility DevOps Octal Modes
- **`4755`** : Standard SUID binary (e.g., `/usr/bin/passwd`).
- **`2775`** : Collaborative directory with SGID enabled.
- **`1777`** : World-writable scratch directory with Sticky Bit (e.g., `/tmp`).

---

## 3. The Lowercase vs Uppercase Mystery: `s` vs `S` and `t` vs `T`

When inspecting special permissions with `ls -l`, you will sometimes see a lowercase letter (`s`, `t`) and other times an uppercase letter (`S`, `T`):

```text
-rwsr-xr-x   (Lowercase 's')  vs  -rwSr-xr-x   (Uppercase 'S')
drwxrwxrwt   (Lowercase 't')  vs  drwxrwxrwT   (Uppercase 'T')
```

### The Rule:
- **Lowercase (`s`, `t`)** : The special bit is active **AND** the underlying standard execute (`x`) bit is **ACTIVE**. *(Healthy, normal state).*
- **Uppercase (`S`, `T`)** : The special bit is active, **BUT** the underlying standard execute (`x`) bit is **MISSING / INACTIVE**!

```bash
# Example: Adding SUID to a non-executable file:
touch test.txt && chmod 4644 test.txt
$ ls -l test.txt
-rwSr--r-- 1 ubuntu ubuntu 0 Oct 14 10:40 test.txt
# -> Notice the uppercase 'S'! It warns you that SUID is set, but nobody can execute the file!
```

---

## 4. Security Auditing: Hunting SUID/SGID Binaries

SUID binaries are primary targets for attackers seeking **Privilege Escalation**. If an attacker finds a vulnerability in a binary owned by root with SUID set, they can execute arbitrary code with full root authority.

### Automated Security Audit Recipe:
```bash
# Find all SUID and SGID files across the root filesystem:
find / -xdev \( -perm -4000 -o -perm -2000 \) -type f -exec ls -la {} +
```

### Common Legitimate SUID Binaries on Linux:
- `/usr/bin/passwd` (Needs root to write to `/etc/shadow`).
- `/usr/bin/sudo` (Needs root to evaluate credentials and escalate privileges).
- `/usr/bin/newgrp` (Needs root to switch GIDs).
- `/usr/bin/su` (Needs root to switch user contexts).

---

## Summary

- Special permissions extend standard DAC: SUID=4, SGID=2, Sticky Bit=1.
- SUID runs executables with the file owner's privileges.
- SGID on directories enforces automatic group inheritance for all new files.
- The Sticky bit prevents users from deleting each other's files in shared directories.
- Lowercase (`s`, `t`) indicates underlying execute is active; uppercase (`S`, `T`) indicates missing execute.
- Regularly audit SUID binaries using `find / -perm /6000 -type f` to prevent security escalation.
