# Managing ACLs in Linux: `getfacl`, `setfacl`, and Default Inheritance

## Overview

Access Control Lists are managed using two core Linux utilities:
- **`getfacl`**: Reads and displays the active Access Control Lists on files and directories.
- **`setfacl`**: Modifies, adds, removes, and configures inheritance defaults for ACL entries.

In production environments, `setfacl` is the standard tool used to grant automated CI/CD runners access to shared web docroots, grant monitoring daemons read rights to secure application logs, and configure persistent directory permissions that propagate automatically to all future files.

---

## 1. Inspecting Permissions: `getfacl`

```bash
getfacl [OPTIONS] FILE...
```

### Demonstration:
```bash
$ getfacl /var/www/app/config.json
# file: var/www/app/config.json
# owner: alex
# group: developers
user::rw-
user:maria:r--            <-- Named User: Maria has read-only
user:ci-deployer:rwx       <-- Named User: CI Runner has full access
group::rw-
group:auditors:r--        <-- Named Group: Auditors have read-only
mask::rwx                 <-- Effective permission ceiling
other::---                <-- World access is completely blocked!
```

---

## 2. Modifying Permissions: `setfacl`

```bash
setfacl [OPTIONS] RULES FILE...
```

### Essential Flags Reference

| Flag | Long Option | Description | DevOps Practical Context |
| :--- | :--- | :--- | :--- |
| **`-m`** | `--modify` | **Adds or modifies** existing ACL rules. | Granting access to a specific user or group. |
| **`-x`** | `--remove` | **Removes** specific ACL entries. | Revoking access for a specific user without touching others. |
| **`-b`** | `--remove-all` | **Strips all extended ACL entries**, reverting to standard DAC. | Resetting permissions back to basic POSIX `ugo`. |
| **`-R`** | `--recursive` | Applies ACL rules **recursively** down a directory tree. | Provisioning an entire docroot or storage mount. |
| **`-d`** | `--default` | **Sets Default ACLs on a directory** for automatic inheritance. | **Enforces persistent permissions on all future files created inside.** |

---

## 3. The `setfacl` Rule Syntax

ACL rules are passed to `-m` or `-x` as colon-separated strings:

| Target | Rule Format | Example | Description |
| :--- | :--- | :--- | :--- |
| **Named User** | `u:USERNAME:PERMS` | `u:maria:r--` | Grant user `maria` read permission. |
| **Named Group** | `g:GROUPNAME:PERMS` | `g:auditors:r-x` | Grant group `auditors` read and execute. |
| **Owner** | `u::PERMS` | `u::rw-` | Update base owner permission. |
| **Owning Group** | `g::PERMS` | `g::r--` | Update base group permission. |
| **Others** | `o::PERMS` | `o::---` | Block all others. |
| **Mask** | `m::PERMS` | `m::r-x` | Set maximum allowable effective ceiling. |

### Practical Examples
```bash
# Grant user 'maria' read-only access to a file:
setfacl -m u:maria:r config.json

# Grant user 'jenkins' read and write access:
setfacl -m u:jenkins:rw config.json

# Revoke maria's ACL entry:
setfacl -x u:maria config.json

# Strip all ACLs from the file (returns to standard DAC, removes '+' from ls -l):
setfacl -b config.json
```

---

## 4. The Critical Concept: The ACL Mask (`mask::`)

The **Mask** defines the **maximum allowable permission ceiling** for all named users, named groups, and the owning group:

```text
Named User Rule:   user:ci-deployer:rwx
Mask Setting:      mask::r--
───────────────────────────────────────
Effective Access:  r-- (Write and Execute are FILTERED OUT by the mask!)
```

If a named user has `rwx`, but the mask is `r--`, `getfacl` will display:
```text
user:ci-deployer:rwx   #effective:r--
```
`ci-deployer` will **NOT** be able to write or execute the file!

> [!TIP]
> By default, `setfacl` recalculates the mask automatically to accommodate the most permissive rule added. If you want to enforce a hard ceiling, set the mask explicitly:
> ```bash
> setfacl -m m::r-x file.txt
> ```

---

## 5. The DevOps Superpower: Default ACLs (`-d`)

When developers or background processes create new files inside `/var/www/uploads`, standard umask rules apply, often locking out other services.

With **Default ACLs (`setfacl -d`)**, any file or directory created inside the directory in the future **automatically inherits the defined ACLs**!

```bash
# Step 1: Set standard access permissions on the directory:
sudo setfacl -m u:www-data:rwx /var/www/uploads

# Step 2: Set DEFAULT permissions on the directory for all future creations:
sudo setfacl -d -m u:www-data:rwx /var/www/uploads
sudo setfacl -d -m u:ci-deployer:rwx /var/www/uploads
```

Now, whether `alex` creates a file, `root` creates a file, or a Python script creates a file inside `/var/www/uploads`, **both `www-data` and `ci-deployer` will immediately have `rwx` permissions on the new file automatically!**

---

## 6. Backing Up and Restoring ACLs

When migrating filesystems or creating disaster recovery manifests, standard `tar` archives may drop extended attributes unless explicitly instructed (`tar --xattrs`).

Use `getfacl` and `setfacl` to dump and restore ACL manifests:

```bash
# Backup all ACL definitions across the web tree:
getfacl -R /var/www/html > /backups/permissions_acl.bak

# Restore all ACL definitions perfectly:
setfacl --restore=/backups/permissions_acl.bak
```

---

## Summary

- Use `getfacl <file>` to inspect active Access Control Lists.
- Use `setfacl -m u:<user>:<perms> <file>` to grant permissions to specific named users.
- Use `setfacl -b <file>` to wipe all ACLs and revert to standard POSIX DAC.
- The `mask` defines the maximum allowable effective permissions across all named entries.
- Use `setfacl -d -m ... <dir>` to enforce automatic ACL inheritance on all future files.
- Backup and restore complex permission architectures using `getfacl -R > backup.bak` and `setfacl --restore=backup.bak`.
