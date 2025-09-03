# Group Information Files in Linux: `/etc/group`, `/etc/gshadow`, and Auditing

## Overview

In Linux, group definitions, group identifiers, and supplementary memberships are stored directly in two core configuration files located in `/etc`:
1. **`/etc/group`**: The world-readable file defining all system and user groups and their member lists.
2. **`/etc/gshadow`**: The privileged security file holding optional group passwords and group administrator definitions.

Understanding how to read, safely edit, and audit these files is essential for enforcing the Principle of Least Privilege and detecting unauthorized privilege escalation.

---

## 1. The `/etc/group` File

The `/etc/group` file is text-based and world-readable:
- **Permissions**: `-rw-r--r--` (`0644`).
- **Access**: Read access is granted to all users so system tools (`ls`, `ps`, `tar`) can translate numerical GIDs into human-readable group names.

### The 4-Field Anatomy of `/etc/group`

```text
docker : x : 998 : ubuntu,alex,jenkins
  [1]   [2]  [3]           [4]
```

| Field # | Field Name | Description | Example |
| :---: | :--- | :--- | :--- |
| **1** | **Group Name** | The name of the group. | `docker` |
| **2** | **Password Placeholder** | Usually **`x`**, indicating the group password (if any) is stored securely in `/etc/gshadow`. | `x` |
| **3** | **Group ID (GID)** | The unique numerical integer identifying the group to the kernel. | `998` |
| **4** | **Member List** | Comma-separated list of usernames belonging to this group as **supplementary members**. | `ubuntu,alex,jenkins` |

### The "Missing User" Mystery in Field 4
A common point of confusion for new administrators:
- You create user `alex` (primary group `alex`).
- You inspect `/etc/group`:
  ```text
  alex:x:1001:
  ```
- **Why is Field 4 empty? Where is alex?**
  Field 4 lists only **supplementary** members! Because `alex` has group `1001` configured as their **Primary Group** inside `/etc/passwd`, Linux does not duplicate the username in `/etc/group`.

---

## 2. The `/etc/gshadow` File

The `/etc/gshadow` companion file stores encrypted group passwords and delegates administrative authority:
- **Permissions**: `-rw-r-----` (`0640` owned by `root:shadow`).
- **Access**: Restricted to `root` and the `shadow` group.

### The 4-Field Anatomy of `/etc/gshadow`

```text
devteam : ! : sarah : alex,ubuntu
   [1]   [2]    [3]       [4]
```

| Field # | Field Name | Description |
| :---: | :--- | :--- |
| **1** | **Group Name** | Must match an existing group in `/etc/group`. |
| **2** | **Encrypted Password** | Password hash required if an external user attempts to temporarily join the group using `newgrp`. If `!` or `*`, no password is set. |
| **3** | **Group Administrators** | Comma-separated list of users empowered to add/remove group members via `gpasswd`. |
| **4** | **Member List** | Explicit members list. |

---

## 3. Safe Editing and Verification: `vigr` and `grpck`

Never edit `/etc/group` directly with Vim or Nano without locking mechanisms. Concurrent operations from package installers or automated user additions can corrupt the file.

### 1. Safe Editing with File Locking: `vigr`
```bash
# Safely edit /etc/group with automated lock:
sudo vigr

# Safely edit /etc/gshadow:
sudo vigr -s
```

### 2. Group Database Integrity Audit: `grpck`
The `grpck` (group check) command scans `/etc/group` and `/etc/gshadow` for syntax corruption, duplicate GIDs, and nonexistent member accounts:

```bash
$ sudo grpck
group 'developers': user 'former_employee' does not exist
grpck: no changes
```

---

## 4. Security Auditing Recipes: Sensitive Groups

Security compliance standards (CIS Benchmarks, ISO 27001) mandate regular auditing of accounts with administrative or privileged socket access:

### Recipe 1: Audit Members of Administrative Groups
```bash
# Audit members with root sudo/wheel access:
grep -E '^(sudo|wheel|adm):' /etc/group
```

### Recipe 2: Audit Members of the Docker Group (Root Equivalent)
Membership in the `docker` group is effectively equivalent to full root access because any user in the `docker` group can mount the host's root filesystem inside a container:

```bash
$ grep '^docker:' /etc/group
docker:x:998:ubuntu,alex
```
*(Ensure only authorized deployment accounts or developers belong to this group).*

### Recipe 3: Detect Duplicate GIDs
Duplicate GIDs cause permission collisions where two different group names share identical access rights:

```bash
cut -d: -f3 /etc/group | sort | uniq -d
```
*(If this prints any numbers, multiple groups share that GID!)*

---

## Summary

- `/etc/group` defines groups, GIDs, and supplementary member lists (`0644`, world-readable).
- `/etc/gshadow` stores encrypted group passwords and group administrators (`0640`, root-only).
- A user's name does not appear in Field 4 of their primary group because primary membership is stored in `/etc/passwd`.
- Always edit group files safely using `vigr` and `vigr -s`.
- Verify database integrity with `grpck`.
- Routinely audit sensitive groups (`sudo`, `wheel`, `docker`) to maintain least privilege.
