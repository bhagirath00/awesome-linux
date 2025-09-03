# The `su` Command in Linux: Switching Users and Login Environments

## Overview

The `su` (Substitute User / Switch User) utility allows you to switch your current terminal shell session to another user account. If no target username is specified, `su` defaults to switching to the `root` superuser.

While `sudo` is the preferred tool for executing individual privileged commands, `su` remains an important utility in systems administration for switching into dedicated service accounts, running batch scripts under alternative identities, and container entrypoint orchestration.

---

## 1. The Crucial Distinction: `su` vs `su -`

The most critical operational concept to understand about `su` is the difference between running with and without the hyphen **`-`** (`-l` or `--login`):

```text
================ DANGEROUS: su root (WITHOUT HYPHEN) ================
* Preserves the CALLING USER'S environment variables ($PATH, $HOME, aliases).
* Leaves you in the caller's current working directory (/home/alex).
* RISKS: Contaminates root's environment with user-defined aliases, custom binaries,
         or malicious $PATH injection!


================ RECOMMENDED: su - root (WITH HYPHEN) ================
* Initializes a complete, clean, isolated LOGIN SHELL.
* Clears the caller's environment.
* Sets $HOME to /root, $USER to root, and resets $PATH to clean system defaults.
* Changes current working directory to the target user's home directory (/root).
* Executes target user's login scripts (/etc/profile, ~/.bash_profile, ~/.bashrc).
```

> [!IMPORTANT]
> **Production Standard**: **Always use `su - <user>` (with the hyphen)** to guarantee a clean, unpolluted environment.

---

## 2. Comparing `su` vs `sudo`

| Feature | `su` | `sudo` |
| :--- | :--- | :--- |
| **Password Required** | Requires the **TARGET user's password** (e.g., must know root's password). | Requires the **CALLING user's own password**. |
| **Security Risk** | High credential sharing risk (multiple admins must know root password). | Zero credential sharing. |
| **Audit Logging** | Logs only the initial session switch; commands typed inside are **unlogged**. | **Logs every single individual command** and argument to syslog. |
| **Granularity** | All-or-nothing (full shell access). | Fine-grained (can permit single commands). |
| **Enterprise Standard** | Deprecated for human administrative access. | **The industry standard.** |

---

## 3. High-Frequency DevOps Recipes

### A. Executing a Single Command as Another User (`-c`)
In automated deployment scripts or systemd service units, execute a command as a service user without opening an interactive shell:

```bash
# Run a database backup script strictly as the 'postgres' user:
su - postgres -c "pg_dumpall > /var/backups/all_databases.sql"
```

### B. Switching to a Service Account Without a Login Shell (`-s`)
System service accounts (like `nginx`, `www-data`, `redis`) have their shell configured to `/usr/sbin/nologin` in `/etc/passwd`. If you need to debug their environment as root:

```bash
# Override the disabled shell and spawn a temporary bash session:
sudo su -s /bin/bash www-data
```

---

## 4. Modern Container Alternative: `gosu` / `su-exec`

In Docker entrypoint scripts, using standard `su` or `sudo` to step down from root to an unprivileged user introduces a critical Unix process problem:
- `su` and `sudo` spawn a separate child process and do not forward OS signals (`SIGTERM`, `SIGINT`) properly.
- As a result, running `docker stop` cannot gracefully shut down your application, causing a 10-second timeout before Docker forcefully kills it (`SIGKILL`)!

In modern Docker entrypoint architecture, use **`gosu`** (Go-based) or **`su-exec`** (C-based):
```bash
#!/usr/bin/env sh
# Docker entrypoint script:
# Exec replaces PID 1 directly, forwarding all signals seamlessly!
exec gosu appuser "$@"
```

---

## 5. Security Hardening: Restricting `su` with PAM

To prevent unauthorized local users from attempting to brute-force the root password via `su`, restrict `su` execution strictly to members of the `wheel` or `sudo` group.

In `/etc/pam.d/su`, uncomment the `pam_wheel.so` directive:

```pam
# /etc/pam.d/su
auth required pam_wheel.so use_uid
```
Now, any user who is not a member of the `wheel` group is blocked from executing `su` entirely, even if they know the root password!

---

## Summary

- Always use `su - <user>` (with the hyphen) to initialize a clean login shell and prevent environment contamination.
- `su` requires the target user's password; `sudo` requires your own password.
- Execute single commands under another user identity using `su - <user> -c "command"`.
- Override `/usr/sbin/nologin` for debugging daemon accounts with `su -s /bin/bash <user>`.
- In Docker entrypoint scripts, prefer `gosu` or `su-exec` to ensure proper signal forwarding.
- Restrict `su` to administrative group members via PAM (`pam_wheel.so`).
