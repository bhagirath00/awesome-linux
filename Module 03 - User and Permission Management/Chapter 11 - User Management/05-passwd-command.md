# Password Management in Linux: `passwd` and `chage`

## Overview

The `passwd` utility manages user authentication credentials and password aging policies on Linux systems. It updates the secure `/etc/shadow` file where cryptographic one-way hashes of user passwords reside.

In enterprise DevOps and cloud security compliance (CIS Benchmarks, PCI-DSS, SOC 2), password management involves much more than entering credentials: it requires enforcing password complexity, setting mandatory rotation periods, forcing password resets on first login, and securely setting credentials non-interactively in container and VM build pipelines.

---

## 1. Changing Passwords: User vs Superuser

### Regular User Context
A standard user running `passwd` can only change their **own** password. Linux enforces security by requiring the user to verify their **current password** before entering the new one:

```bash
$ passwd
Changing password for ubuntu.
Current password:
New password:
Retype new password:
passwd: password updated successfully
```

### Superuser (Root) Context
The `root` superuser can change the password for **any user account** without needing to know the old password:

```bash
sudo passwd alex
# Directly prompts for New password without asking for old credentials!
```

---

## 2. Essential Administrative `passwd` Flags

```bash
passwd [OPTIONS] USERNAME
```

| Flag | Long Option | Description | DevOps Practical Use Case |
| :--- | :--- | :--- | :--- |
| **`-l`** | `--lock` | Locks the user password by prepending `!` to the hash in `/etc/shadow`. | Immediate credential revocation during security incidents. |
| **`-u`** | `--unlock` | Unlocks the password by removing the `!`. | Restoring access. |
| **`-d`** | `--delete` | Deletes the user password (account becomes passwordless). | Ephemeral test containers only (avoid in production!). |
| **`-e`** | `--expire` | **Immediately expires the password**. | Forces developer to choose a new password upon first login. |
| **`-S`** | `--status` | Displays password status summary (Locked, Password set, No password). | Quick credential status auditing. |

### Inspecting Account Password Status (`-S`)
```bash
$ sudo passwd -S alex
alex P 10/14/2026 0 90 7 -1
```
- Column 1: Username (`alex`).
- Column 2: Status (`P` = Password active, `L` = Locked, `NP` = No password).
- Column 3: Date of last password change.
- Column 4: Minimum password age (days).
- Column 5: Maximum password age (days).
- Column 6: Password warning period (days).
- Column 7: Inactivity lock period (-1 = disabled).

---

## 3. Password Aging Policies: The `chage` Command

While `passwd` can handle basic aging, the dedicated **`chage`** (change age) command provides granular control over expiration lifecycles:

```bash
# View comprehensive password aging details:
$ sudo chage -l alex
Last password change                                    : Oct 14, 2026
Password expires                                        : Jan 12, 2027
Password inactive                                       : never
Account expires                                         : never
Minimum number of days between password change          : 0
Maximum number of days between password change          : 90
Number of days of warning before password expires       : 7
```

### Enterprise Policy Recipes

#### 1. Enforce 90-Day Password Rotation with 7-Day Warning
```bash
sudo chage -M 90 -W 7 alex
```

#### 2. Force User to Change Password on Next Login
When onboarding a new engineer with a temporary password:
```bash
sudo chage -d 0 alex
# Or:
sudo passwd -e alex
```
*On next SSH or console login, Linux immediately forces the user to set a new private password before granting shell access.*

---

## 4. Non-Interactive Password Setting in DevOps Automation: `chpasswd`

Standard `passwd` requires an interactive TTY terminal, which breaks automated provisioning in CI/CD pipelines, Dockerfiles, and cloud-init scripts.

The standard Unix solution for non-interactive automation is **`chpasswd`**:

```bash
# Set password non-interactively via pipeline:
echo "username:SecretPassword123!" | sudo chpasswd
```

### Cloud-Init Provisioning Example:
```bash
#!/usr/bin/env bash
set -euo pipefail

NEW_USER="ci-agent"
RANDOM_PASS=$(tr -dc 'A-Za-z0-9!@#$' < /dev/urandom | head -c 20)

useradd -m -s /bin/bash "$NEW_USER"
echo "$NEW_USER:$RANDOM_PASS" | chpasswd
passwd -e "$NEW_USER"  # Force change on first use

echo "User $NEW_USER provisioned with temporary credentials."
```

---

## 5. Password Hashing Under the Hood (`/etc/shadow`)

Linux never stores plain-text passwords. When a password is set, the kernel computes a one-way cryptographic hash with a unique random **salt**:

```text
$6$qZ7yX9wK$vR4pL1mQ...
 ──┬─ ───┬──── ─────────
   │     │        │
   │     │        └─ 86-Character Encrypted Hash
   │     └─ 8-Character Random Salt
   └─ Hash Algorithm Identifier ($6$ = SHA-512)
```

### Algorithm Identifiers in `/etc/shadow`:
- **`$1$`** : MD5 (Deprecated / Insecure)
- **`$5$`** : SHA-256
- **`$6$`** : SHA-512 (Standard on modern Linux)
- **`$y$`** : Yescrypt (Default on modern Ubuntu 22.04+ & Debian 12, resistant to GPU brute-forcing)

---

## Summary

- Regular users can only change their own passwords; `root` can change any password.
- Force users to choose a new password on their next login with `passwd -e <user>`.
- Configure enterprise rotation policies (e.g. 90-day expiry) using `chage -M 90 <user>`.
- Use `echo "user:pass" | chpasswd` for non-interactive scripting in CI/CD pipelines and Dockerfiles.
- Password hashes and salts are securely maintained in `/etc/shadow` using modern algorithms like SHA-512 (`$6$`) or Yescrypt (`$y$`).
