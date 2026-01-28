# Troubleshooting Permission Denied (EACCES) & Operation Not Permitted (EPERM)

## 1. Architectural Concept & Deep Dive

When a Linux command or container service crashes with `Permission denied` (`EACCES` errno 13) or `Operation not permitted` (`EPERM` errno 1), the failure is governed by one of **five distinct Linux kernel permission subsystems**:

1. **POSIX Discretionary Access Control (DAC)**: Traditional file mode bits (`rwx`), UIDs, and GIDs.
2. **Directory Traversal Bit**: The execute (`+x`) permission on *every parent directory* leading to the target file.
3. **POSIX Access Control Lists (ACLs)**: Extended per-user and per-group permissions overriding standard DAC (`getfacl`).
4. **Filesystem Attributes (ext4/xfs flags)**: Immutable or append-only flags (`chattr +i`).
5. **Mandatory Access Control (MAC)**: Kernel security modules (**SELinux** or **AppArmor**) enforcing context policies regardless of root privileges.

```
+─────────────────────────────────────────────────────────────────────────+
|                    KERNEL PERMISSION EVALUATION FLOW                    |
|                                                                         |
|                       Process invokes open(2)                           |
|                                  │                                      |
|                                  ▼                                      |
|   1. Parent Traversal Check: Do all parent dirs have +x?                |
|      ├── NO  ──► Return EACCES ("Permission denied")                    |
|      └── YES                                                            |
|          ▼                                                              |
|   2. Standard DAC Check: Does UID/GID match r/w/x bits?                 |
|      ├── NO  ──► Check Extended ACLs (getfacl)                          |
|      │           ├── NO  ──► Return EACCES                              |
|      └── YES                                                            |
|          ▼                                                              |
|   3. Extended Attributes: Is file marked immutable (+i)?                |
|      ├── YES ──► Return EPERM ("Operation not permitted")               |
|      └── NO                                                             |
|          ▼                                                              |
|   4. MAC Check: Does SELinux / AppArmor allow transition?               |
|      ├── NO  ──► Return EACCES / AVC Denial                             |
|      └── YES ──► Syscall succeeds!                                      |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. EACCES vs EPERM: The Critical Distinction

Understanding the difference between the two standard POSIX error codes speeds up root-cause analysis:

| Error String | Kernel Errno | Underlying Cause |
| :--- | :--- | :--- |
| **`Permission denied`** | `EACCES` (13) | Standard file permission mismatch, lack of `+x` on parent folder, or SELinux/AppArmor MAC block. |
| **`Operation not permitted`**| `EPERM` (1) | The process lacks the required **Linux Capability** (e.g. `CAP_NET_ADMIN`), or the file has the immutable flag set (`chattr +i`). |

---

## 3. The "Parent Directory Traversal" Trap

A pervasive production mistake:
A file `/data/app/config.json` has `chmod 777` (world readable/writable/executable). Yet the `deployer` user still receives `Permission denied` when trying to read it!

### Why?
To access any file or directory in Linux, a process **MUST have execute (`+x`) permissions on EVERY parent directory in the path**, all the way up to `/`!

```
/           (drwxr-xr-x) -> has +x (Pass)
└── data    (drwxr-x---) -> owned by root:admin. deployer not in admin -> NO +x!
    └── app (drwxrwxrwx) -> Traversal fails here! Target never reached!
        └── config.json
```

### Diagnosis:
Use `namei -l` to audit permissions along the entire resolved path:
```bash
namei -l /data/app/config.json
```
```
f: /data/app/config.json
drwxr-xr-x root   root   /
drwxr-x--- root   admin  data          <-- FAILS HERE for non-admin users!
drwxrwxrwx root   root   app
-rwxrwxrwx root   root   config.json
```

---

## 4. Extended Access Control Lists (ACLs)

When standard `chmod` mode bits do not match the observed behavior, check if an ACL is set:

```bash
ls -l /var/www/html/index.html
```
```
-rw-r--r--+ 1 www-data www-data 4510 Oct 14 08:00 index.html
```
- Notice the **plus sign (`+`)** at the end of the permission string! This indicates extended ACLs are present.

### Inspecting and Modifying ACLs
```bash
# View all effective ACLs
getfacl /var/www/html/index.html
```
```
# file: var/www/html/index.html
# owner: www-data
# group: www-data
user::rw-
user:deployer:---     <-- User deployer is explicitly blocked from reading!
group::r--
mask::r--
other::r--
```

Remove the restricting ACL:
```bash
sudo setfacl -x u:deployer /var/www/html/index.html
```

---

## 5. Immutable Files (`chattr` and `lsattr`)

If even `root` (UID 0) receives `Operation not permitted` when trying to modify or delete a file:

```bash
sudo rm -f /etc/resolv.conf
# Output: rm: cannot remove '/etc/resolv.conf': Operation not permitted
```

### Check Linux Filesystem Extended Attributes
```bash
lsattr /etc/resolv.conf
```
```
----i---------e---- /etc/resolv.conf
```
- The **`i` (immutable)** attribute prevents any user—including root—from modifying, renaming, linking, or deleting the file!

### Clearing the Immutable Bit
```bash
sudo chattr -i /etc/resolv.conf
sudo rm /etc/resolv.conf  # Succeeds!
```

---

## 6. Mandatory Access Control: SELinux & AppArmor

On RHEL/CentOS/Fedora systems, **SELinux** enforces Mandatory Access Control. A process running as root or a web server can have correct DAC permissions (`0777`) and still be blocked by SELinux.

### Step 1: Check Current Enforcement Mode
```bash
getenforce
```
- `Enforcing`: Policy is actively blocking violations.
- `Permissive`: Policy allows access but logs violations.

### Step 2: Inspect Security Contexts
```bash
ls -Z /var/www/html/
```
```
-rw-r--r--. www-data www-data unconfined_u:object_r:user_home_t:s0 index.html
```
- Nginx/Apache requires `httpd_sys_content_t`. Because the context is `user_home_t`, SELinux blocks the web server!

### Step 3: Check Audit Logs with `audit2why`
```bash
sudo ausearch -m AVC -ts recent | audit2why
```
```
type=AVC msg=audit(1697268000.123:456): avc:  denied  { read } for  pid=1420 comm="nginx" name="index.html" dev="sda1" ino=41205 scontext=system_u:system_r:httpd_t:s0 tcontext=unconfined_u:object_r:user_home_t:s0 tclass=file permissive=0

        Was caused by:
                Missing type enforcement allow rule.
```

### Step 4: Restore Default Security Context
```bash
sudo restorecon -Rv /var/www/html/
```

---

## 7. Verification & Troubleshooting Runbook

### Diagnostic Checklist
```bash
# 1. Audit full directory traversal path
namei -om /path/to/target/file

# 2. Inspect ACLs
getfacl /path/to/target/file

# 3. Inspect immutable attributes
lsattr /path/to/target/file

# 4. Search for SELinux AVC denials
sudo ausearch -m AVC -ts recent
```

### Common Pitfalls & Solutions
1. **User added to group, but still receives `Permission denied`**:
   - Cause: Linux group memberships are populated into process tokens at login time (`login` / `pam`). Active shells do not inherit new group additions dynamically.
   - Solution: Log out and log back in, or run `newgrp <groupname>` in the active shell.
2. **Bash script fails with `Permission denied` even with `chmod +x`**:
   - Cause: The partition is mounted with the `noexec` mount option in `/etc/fstab` (common on `/tmp` and `/home`).
   - Solution: Check `mount | grep $(df -P script.sh | tail -1 | awk '{print $6}')`. Remount without `noexec` or execute via interpreter (`bash script.sh`).

---

## 8. Key Takeaways
- `EACCES` indicates permission/traversal/SELinux issues; `EPERM` indicates capability or immutable bit (`chattr +i`) restrictions.
- Lack of execute (`+x`) permission on any parent directory in the path blocks access regardless of file-level permissions.
- Use `namei -l` to audit directory traversal trees in seconds.
- In SELinux environments, verify contexts with `ls -Z` and remediate using `restorecon -Rv`.
