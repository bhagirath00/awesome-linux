# User Information Files in Linux: `/etc/passwd`, `/etc/shadow`, and Security Auditing

## Overview

In Linux, user configuration is not hidden inside a monolithic binary registry or proprietary database. The core identity of every account, group membership, authentication credential, and environment preference is recorded in clean, plaintext configuration files located in `/etc`.

Understanding the exact anatomy of these files—specifically **`/etc/passwd`**, **`/etc/shadow`**, **`/etc/login.defs`**, and **`/etc/default/useradd`**—is essential for security auditing, compliance verification, and understanding how the Linux kernel resolves authentication.

---

## 1. The `/etc/passwd` File

The `/etc/passwd` file contains basic account metadata for every user on the system.
- **Permissions**: `-rw-r--r--` (`0644`).
- **Access**: **World-readable**. Any user, process, or container can read `/etc/passwd`. (Because tools like `ls` and `ps` need to translate numerical UIDs into usernames like `"ubuntu"` or `"root"`).

### The 7-Field Anatomy of `/etc/passwd`

```text
ubuntu : x : 1000 : 1000 : Ubuntu User,,, : /home/ubuntu : /bin/bash
  [1]   [2]   [3]    [4]        [5]               [6]          [7]
```

| Field # | Field Name | Description | Example |
| :---: | :--- | :--- | :--- |
| **1** | **Username** | The login account name string. | `ubuntu` |
| **2** | **Password Placeholder** | Always an **`x`** on modern systems, indicating the actual password hash is secured in `/etc/shadow`. *(In the 1970s, encrypted passwords were stored directly here; modern Linux moved them to `/etc/shadow` to prevent offline dictionary attacks).* | `x` |
| **3** | **User ID (UID)** | The unique numerical integer identifying the user to the kernel. | `1000` |
| **4** | **Primary Group ID (GID)** | The numerical integer of the user's primary group. | `1000` |
| **5** | **GECOS / Comment** | User's full name, phone number, room number, or role description. | `Ubuntu User` |
| **6** | **Home Directory** | Absolute path to the user's initial working directory upon login. | `/home/ubuntu` |
| **7** | **Login Shell** | Absolute path to the user's default command-line interpreter. If set to `/usr/sbin/nologin` or `/bin/false`, interactive shell login is blocked. | `/bin/bash` |

---

## 2. The `/etc/shadow` File

The `/etc/shadow` file stores encrypted password hashes and password expiration parameters.
- **Permissions**: `-rw-r-----` (`0640` owned by `root:shadow`) or `-rw-------` (`0600` owned by `root:root`).
- **Access**: Strictly restricted to the `root` superuser and processes with shadow group privilege.

### The 9-Field Anatomy of `/etc/shadow`

```text
alex : $6$qZ7yX9wK$vR4pL1... : 19644 :  0  :  90  :  7  :  14  : 20454 :
 [1]           [2]              [3]    [4]   [5]    [6]    [7]     [8]   [9]
```

| Field # | Field Name | Description |
| :---: | :--- | :--- |
| **1** | **Username** | Must match an existing username in `/etc/passwd`. |
| **2** | **Encrypted Password Hash** | Cryptographic hash with salt (`$6$` = SHA-512, `$y$` = Yescrypt). If prepended with `!`, account is locked. If `*`, login is disabled. |
| **3** | **Last Password Change** | Number of days between January 1, 1970 (Unix Epoch) and the date the password was last changed. |
| **4** | **Minimum Password Age** | Minimum number of days required before the user is allowed to change their password again. |
| **5** | **Maximum Password Age** | Maximum number of days the password remains valid (e.g., `90`). |
| **6** | **Warning Period** | Number of days prior to password expiration that the user receives login warnings (e.g., `7`). |
| **7** | **Inactivity Period** | Number of days after password expiration before the account is permanently disabled. |
| **8** | **Account Expiration Date** | Date the entire account ceases to function, measured in days since Unix Epoch. |
| **9** | **Reserved Flag** | Reserved for future kernel use (currently blank). |

---

## 3. System Defaults: `/etc/login.defs` and `/etc/default/useradd`

- **`/etc/login.defs`**: Defines global parameters for account creation:
  - `UID_MIN` and `UID_MAX` (e.g., 1000 to 60000 for regular users).
  - `SYS_UID_MIN` and `SYS_UID_MAX` (e.g., 101 to 999 for system accounts).
  - `ENCRYPT_METHOD` (e.g., `SHA512` or `YESCRYPT`).
  - `UMASK` default setting (typically `022`).
- **`/etc/default/useradd`**: Defines fallback flags used when `useradd` runs without arguments:
  ```bash
  $ useradd -D
  GROUP=100
  HOME=/home
  INACTIVE=-1
  EXPIRE=
  SHELL=/bin/sh
  SKEL=/etc/skel
  CREATE_MAIL_SPOOL=no
  ```

---

## 4. Safe Editing and Integrity Verification: `pwck` and `vipw`

Directly modifying `/etc/passwd` with a standard text editor like Vim or Nano is dangerous: a syntax error, extra colon, or concurrent write can corrupt authentication across the entire server!

### 1. Safe Editing with File Locking: `vipw`
The `vipw` command opens `/etc/passwd` in your editor while placing a filesystem lock (`/etc/ptmp`) to prevent concurrent updates:
```bash
# Safely edit /etc/passwd:
sudo vipw

# Safely edit /etc/shadow:
sudo vipw -s
```

### 2. Integrity Verification: `pwck`
The `pwck` (password check) utility audits `/etc/passwd` and `/etc/shadow` for formatting errors, duplicate UIDs, invalid shells, or missing home directories:

```bash
$ sudo pwck
user 'guest': directory '/home/guest' does not exist
pwck: no changes
```

---

## 5. Security Auditing Recipes

### Recipe 1: Detect Unauthorized UID 0 Accounts (Backdoor Detection)
Only the `root` account should have UID `0`. Any other user with UID 0 possesses full superuser power and represents a critical security compromise:

```bash
# Search for any account with UID 0 other than root:
awk -F':' '($3 == 0 && $1 != "root") { print "ALERT: Unauthorized UID 0 account found: " $1 }' /etc/passwd
```

### Recipe 2: Find Accounts with Empty Password Hashes
Accounts with an empty password field in `/etc/shadow` allow unauthenticated logins:

```bash
sudo awk -F':' '($2 == "") { print "ALERT: Account with NO password: " $1 }' /etc/shadow
```

### Recipe 3: Querying User Identity via `getent`
In modern environments integrating centralized LDAP, Active Directory (SSSD), or FreeIPA, accounts might not exist locally in `/etc/passwd`.

Always use **`getent`** to query identity across all configured authentication sources:

```bash
# Queries local /etc/passwd AND remote LDAP/Active Directory simultaneously:
getent passwd alex
```

---

## Summary

- `/etc/passwd` holds public account metadata (UID, primary GID, shell, home directory) and is world-readable (`0644`).
- `/etc/shadow` stores salted cryptographic password hashes and expiration parameters, restricted strictly to `root` (`0640` or `0600`).
- Always use `vipw` and `vipw -s` to edit user files with automated file locking.
- Verify user database integrity using `pwck`.
- Audit systems regularly for unauthorized UID 0 accounts and empty password fields.
- Query users across local files and remote directory services using `getent passwd <user>`.
