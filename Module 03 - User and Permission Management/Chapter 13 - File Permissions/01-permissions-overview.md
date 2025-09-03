# Linux File Permissions Overview: The Discretionary Access Control (DAC) Model

## Overview

Security in Linux is built upon the **Discretionary Access Control (DAC)** model. Under DAC, every filesystem object—whether a regular text file, directory, named pipe, or device driver—is owned by an identified **User** and an identified **Group**, with three distinct tiers of access rights governing:
1. What the **Owner (User)** can do.
2. What members of the **Group** can do.
3. What everyone else (**Others / World**) can do.

In DevOps, cloud engineering, and Site Reliability Engineering, mastering Linux permissions is critical: an overly permissive private key (`~/.ssh/id_rsa`) causes SSH to refuse connection, an incorrectly permissioned Nginx docroot causes HTTP 403 Forbidden errors, and loose configuration permissions can expose database passwords to unprivileged container processes.

---

## 1. Anatomy of the 10-Character Permission String

When you inspect a file using `ls -l`, the very first column displays the 10-character mode string:

```bash
$ ls -l /var/www/html/index.php
-rwxr-xr-- 1 www-data devteam 4096 Oct 14 10:15 /var/www/html/index.php
```

```text
-  r w x  r - x  r - -
│  ─────  ─────  ─────
│    │      │      │
│    │      │      └─ Tier 3: OTHERS (World) Permissions  (r--)
│    │      └──────── Tier 2: GROUP Permissions            (r-x)
│    └─────────────── Tier 1: OWNER (User) Permissions     (rwx)
└──────────────────── File Type Indicator                  (-)
```

### The 10-Character Breakdown

| Character Position | Scope | Description | Allowed Values |
| :---: | :--- | :--- | :--- |
| **`1`** | **File Type** | Identifies the filesystem object type. | `-` (regular file), `d` (directory), `l` (symlink), `s` (socket), `p` (pipe), `c` (char dev), `b` (block dev). |
| **`2, 3, 4`** | **Owner (User - `u`)** | Permissions granted strictly to the single user account that owns the file. | `r` (read), `w` (write), `x` (execute), `-` (none). |
| **`5, 6, 7`** | **Group (`g`)** | Permissions granted to any user belonging to the file's assigned group. | `r` (read), `w` (write), `x` (execute), `-` (none). |
| **`8, 9, 10`**| **Others / World (`o`)**| Permissions granted to every other user on the entire operating system. | `r` (read), `w` (write), `x` (execute), `-` (none). |

---

## 2. How the Linux Kernel Evaluates Permissions: The Strict Precedence Rule

When a process (e.g., Python script, Nginx worker, or Bash shell) attempts to read, write, or execute a file, the Linux kernel evaluates access in strict, sequential order:

```text
Process attempts to access File
  │
  ▼
1. Is the process UID identical to the File's OWNER UID?
   ├── YES ──> Kernel evaluates OWNER bits ONLY.
   │           (If Owner bits say "no write", write is DENIED immediately,
   │            even if Group or Others explicitly allow write!)
   │
   └── NO  ──> 2. Does the process belong to the File's GID?
               ├── YES ──> Kernel evaluates GROUP bits ONLY.
               │
               └── NO  ──> 3. Kernel evaluates OTHERS bits.
```

> [!IMPORTANT]
> **Kernel First-Match Rule**: Linux does **not** combine permissions across tiers. If you are the Owner of a file and the Owner bits are `r--`, but Group is `rwx`, you **CANNOT** write to the file! The kernel stops evaluating as soon as it determines you are the Owner.

---

## 3. High-Visibility DevOps Permission Baselines

Production infrastructure enforces strict standard permissions based on asset sensitivity:

| Asset | Target Mode | Numeric | Operational Rationale |
| :--- | :--- | :---: | :--- |
| **SSH Private Keys** (`~/.ssh/id_rsa`) | `-rw-------` | `600` | SSH client strictly refuses to connect if accessible by Group or Others. |
| **SSH Directory** (`~/.ssh`) | `drwx------` | `700` | Only owner can list or enter SSH directory. |
| **SSL / TLS Private Keys** (`server.key`)| `-rw-------` | `600` | Protects cryptographic private keys from local users. |
| **Production Configs** (`/etc/hosts`) | `-rw-r--r--` | `644` | World can read; only root can modify. |
| **Shared Web Directories** | `drwxr-xr-x` | `755` | Owner can edit; web server can traverse and read. |
| **Temporary Scratch Space** (`/tmp`) | `drwxrwxrwt` | `1777` | World can write, but Sticky Bit prevents deleting others' files. |

---

## Summary

- Linux enforces Discretionary Access Control (DAC) via three permission tiers: Owner (`u`), Group (`g`), and Others (`o`).
- The 10-character string in `ls -l` defines File Type (pos 1), Owner (pos 2–4), Group (pos 5–7), and Others (pos 8–10).
- The kernel uses a strict first-match evaluation: Owner -> Group -> Others (it never merges permissions).
- Critical secrets (SSH keys, TLS certificates, `.env` files) must be locked down to `600` or `700`.
