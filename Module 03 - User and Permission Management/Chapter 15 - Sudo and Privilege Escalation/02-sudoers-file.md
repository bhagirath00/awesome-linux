# The `/etc/sudoers` File: Syntax, Aliases, and Drop-In Configuration

## Overview

The `/etc/sudoers` file is the master policy specification for the `sudo` subsystem. It defines **who** can run **which commands**, on **which host machines**, and as **which target users**.

In automated DevOps environments and cloud provisioning, mastering `/etc/sudoers` syntax allows you to configure passwordless execution for CI/CD runners, grant junior developers restricted access to service restarts without full root authority, and implement modular drop-in configurations via `/etc/sudoers.d/`.

---

## 1. The Core Rule Syntax: `WHO WHERE = (AS_WHOM) WHAT`

Every rule in `/etc/sudoers` adheres to a strict 4-part structure:

```text
alex      ALL = (ALL : ALL)      ALL
────      ───   ───────────      ───
 [1]      [2]       [3]          [4]
 WHO     WHERE    AS_WHOM        WHAT
```

### Breaking Down the 4 Components:
1. **WHO**: Specifies the user or group:
   - `username` : Applies to an individual user (`alex`).
   - `%groupname` : Leading **`%`** specifies an entire **group** (e.g. `%sudo`, `%wheel`).
2. **WHERE**: Specifies which network hosts the rule applies to (almost always `ALL`).
3. **(AS_WHOM)**: Specifies which user and group the command can run as:
   - `(ALL : ALL)` : Can run as any user and any group.
   - `(root)` : Can run as root only.
   - `(postgres)` : Restricted strictly to running as user postgres.
4. **WHAT**: Comma-separated list of absolute paths to permitted binary executables:
   - `ALL` : Unrestricted access to any binary on the system.
   - `/usr/bin/systemctl restart nginx, /usr/bin/nginx -t` : Restricted strictly to these two commands.

---

## 2. Standard Default Group Rules

On Ubuntu and Debian systems:
```sudoers
# Members of group 'sudo' inherit full root privileges:
%sudo   ALL=(ALL:ALL) ALL
```

On Red Hat, CentOS, Rocky Linux, and Fedora systems:
```sudoers
# Members of group 'wheel' inherit full root privileges:
%wheel  ALL=(ALL) ALL
```

---

## 3. Passwordless Sudo: The `NOPASSWD` Tag

In automated environments (such as GitHub Actions runners, Jenkins agents, or AWS cloud-init scripts), automation tools cannot enter interactive passwords:

### A. Full Passwordless Sudo (Dedicated Service Accounts Only)
```sudoers
# Allow CI deployer to execute any command as root without prompting for password:
deployer ALL=(ALL) NOPASSWD: ALL
```

### B. Least-Privilege Passwordless Sudo (Recommended Best Practice)
Never grant `NOPASSWD: ALL` to human developers or unvetted scripts. Restrict passwordless execution strictly to safe, designated maintenance commands:

```sudoers
# Allow developer 'alex' to restart and inspect nginx without a password:
alex ALL=(root) NOPASSWD: /usr/bin/systemctl restart nginx, /usr/bin/systemctl status nginx
```

---

## 4. Aliases: Structuring Enterprise Sudo Policies

For enterprise environments with hundreds of servers and multiple teams, `/etc/sudoers` supports modular **Aliases**:

```sudoers
# 1. Define User Aliases
User_Alias DEVOPS_LEADS = alex, sarah
User_Alias JUNIOR_DEVS  = bob, charlie

# 2. Define Command Aliases
Cmnd_Alias WEB_OPS = /usr/bin/systemctl restart nginx, /usr/bin/systemctl reload nginx, /usr/bin/nginx -t
Cmnd_Alias LOG_OPS = /usr/bin/journalctl, /usr/bin/tail /var/log/nginx/*

# 3. Assign Rules Cleanly
DEVOPS_LEADS ALL=(ALL:ALL) ALL
JUNIOR_DEVS  ALL=(root) NOPASSWD: WEB_OPS, LOG_OPS
```

---

## 5. The Production Standard: `/etc/sudoers.d/` Drop-In Directory

In automated configuration management (Ansible, Puppet, Terraform), **never edit the main `/etc/sudoers` file directly**. Package updates can overwrite or conflict with changes.

Instead, place isolated policy files in the **`/etc/sudoers.d/`** directory:

```bash
# Example drop-in file: /etc/sudoers.d/99-ci-deployer
deployer ALL=(ALL) NOPASSWD: ALL
```

### Strict Rules for Files in `/etc/sudoers.d/`:
1. **Permissions**: The file mode must be strictly **`0440`** (`-r--r-----`) owned by `root:root`. If permissions are wrong, `sudo` will ignore the file!
2. **File Naming**: Filenames **MUST NOT contain dots (`.`) or end in tildes (`~`)**!
   - `99-ci-deployer` (Valid)
   - `deployer.conf` (**INVALID**: Sudo will ignore it because of the dot!).

---

## Summary

- The sudoers rule syntax is: `WHO WHERE = (AS_WHOM) [TAGS:] WHAT`.
- Prefix group names with `%` (e.g. `%sudo`, `%wheel`).
- Use `NOPASSWD: /path/to/binary` for automated scripts, enforcing the Principle of Least Privilege.
- Always use absolute paths for permitted commands to prevent binary path hijacking.
- Place custom policies in `/etc/sudoers.d/<filename>` with strict `0440` permissions and no dots in the filename.
