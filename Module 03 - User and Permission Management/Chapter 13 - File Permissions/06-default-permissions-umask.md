# Default Permissions and the `umask` in Linux

## Overview

When an application, script, or user creates a new file or directory, what determines its initial permissions? Why does `touch file.txt` result in `-rw-r--r--` (`644`) rather than `-rwxrwxrwx` (`777`)?

The answer is **`umask`** (User Creation Mask). The `umask` acts as a security filter that automatically masks out (removes) specific permissions when any filesystem object is created.

In cloud security and DevOps engineering, configuring `umask` properly ensures newly generated logs, database dumps, and application artifacts are not inadvertently exposed to untrusted local users.

---

## 1. Base Permissions Before Masking

The Linux kernel defines fixed maximum base permissions for all new creations:

- **Base Directory Permission**: **`777` (`rwxrwxrwx`)**
  - Directories require execute (`x`) permission so users and processes can enter and traverse them.
- **Base Regular File Permission**: **`666` (`rw-rw-rw-`)**
  - For safety against malware, **no regular file is ever created with execute (`x`) permission by default**, regardless of umask!

---

## 2. How `umask` Math Works

The `umask` specifies the permissions that should **NOT** be given to newly created objects:

$$\text{Final File Permission} = 666 - \text{umask}$$
$$\text{Final Directory Permission} = 777 - \text{umask}$$

```text
================ STANDARD UMASK 022 ================
Base Directory:  7 7 7  (rwxrwxrwx)     Base File:  6 6 6  (rw-rw-rw-)
umask Mask:    - 0 2 2  (----w--w-)     umask Mask: - 0 2 2  (----w--w-)
───────────────────────────────────     ───────────────────────────────
Resulting Dir:   7 5 5  (rwxr-xr-x)     Resulting File: 6 4 4 (rw-r--r--)
```

---

## 3. Standard Production `umask` Values

| `umask` Value | Resulting File Mode | Resulting Directory Mode | Security Posture & Usage |
| :---: | :---: | :---: | :--- |
| **`0022`** | `644` (`rw-r--r--`) | `755` (`rwxr-xr-x`) | **Standard Default**: World can read and traverse; only owner can modify. |
| **`0002`** | `664` (`rw-rw-r--`) | `775` (`rwxrwxr-x`) | **User Private Group (UPG)**: Owner and Group can write; Others read-only. |
| **`0027`** | `640` (`rw-r-----`) | `750` (`rwxr-x---`) | **Hardened Production Server**: Group can read; Others completely locked out! |
| **`0077`** | `600` (`rw-------`) | `700` (`rwx------`) | **Maximum Security (Zero Trust)**: Only Owner has any access; Group and Others blocked. |

---

## 4. Inspecting and Setting the `umask`

### View Current Mask
```bash
# View numerical value:
$ umask
0022

# View human-readable symbolic value (-S):
$ umask -S
u=rwx,g=rx,o=rx
```

### Setting Temporary Mask
```bash
# Set mask for current shell session:
umask 0077

touch test-secure.txt
mkdir secure-dir

$ ls -ld test-secure.txt secure-dir
drwx------ 2 ubuntu ubuntu 4096 Oct 14 10:30 secure-dir
-rw------- 1 ubuntu ubuntu    0 Oct 14 10:30 test-secure.txt
```

### The Subshell DevOps Pro-Trick
To generate a sensitive file (like an API token or private key) with `600` permissions without altering the running script's global umask:

```bash
# Executes umask 077 inside an isolated subshell:
(umask 077 && echo "SECRET_TOKEN=xyz123" > /app/secrets.env)

# Verify permissions:
ls -l /app/secrets.env
# -rw------- 1 ubuntu ubuntu 21 Oct 14 10:31 /app/secrets.env
```

---

## 5. Permanent System-Wide Configuration

To configure default umask values across all users and services on a Linux server:

1. **`/etc/login.defs`**:
   ```ini
   UMASK 027
   ```
2. **`/etc/profile` and `/etc/bash.bashrc`**:
   ```bash
   umask 027
   ```
3. **Systemd Services (`/etc/systemd/system/app.service`)**:
   Systemd daemons run with their own umask settings:
   ```ini
   [Service]
   UMASK=0027
   ```

---

## Summary

- Regular files start at base `666`; directories start at base `777`.
- The `umask` value subtracts permissions from these base values.
- Default `022` yields `644` for files and `755` for directories.
- Hardened environments use `027` (blocking Others) or `077` (blocking Group and Others).
- Use subshell execution `(umask 077 && touch file)` to generate locked-down files atomically.
