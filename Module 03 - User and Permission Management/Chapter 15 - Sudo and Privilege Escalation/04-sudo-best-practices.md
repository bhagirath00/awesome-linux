# Sudo Best Practices, GTFOBins, and Hardening in Production

## Overview

Granting `sudo` privileges is the most sensitive authorization action on a Linux operating system. In enterprise DevOps, a poorly written `sudoers` rule—such as allowing a junior developer to run `sudo vim /etc/nginx/nginx.conf`—unintentionally grants that developer **unrestricted root shell access to the entire operating system**.

Understanding privilege escalation vectors (documented by the **GTFOBins** project) and implementing defense-in-depth hardening ensures administrative delegation remains secure, audited, and strictly limited to intended tasks.

---

## 1. The GTFOBins Danger: The "Accidental Root Shell" Trap

A classic mistake made by systems administrators is attempting to restrict an engineer to editing or viewing specific files by granting `sudo` access to standard text editors or pagers:

```sudoers
# SEVERELY FLAWED RULE:
alex ALL=(root) NOPASSWD: /usr/bin/vim /etc/nginx/nginx.conf, /usr/bin/less /var/log/*
```
The administrator thinks: *"Alex can only edit nginx configs and view logs."*

### How the Attacker Gets a Full Root Shell in 3 Seconds:
1. **Via Vim**: Alex runs `sudo vim /etc/nginx/nginx.conf`.
   Inside Vim, Alex types:
   ```vim
   :!/bin/bash
   ```
   **Vim immediately spawns a root shell (`#`)!** Alex is now full root on the host!
2. **Via Less**: Alex runs `sudo less /var/log/nginx/access.log`.
   Inside Less, Alex presses:
   ```text
   !/bin/bash
   ```
   **Less immediately spawns a root shell!**
3. **Via Find**: If granted `sudo find`:
   ```bash
   sudo find . -exec /bin/sh \; -quit
   # -> Spawns root shell!
```

---

## 2. Preventing Shell Escapes: The `NOEXEC` Tag

The `NOEXEC` tag tells the `sudo` subsystem to inject a special wrapper library (`sudo_noexec.so`) that intercepts `execve()` system calls, preventing the invoked binary from spawning child processes (such as `/bin/bash` or `/bin/sh`):

```sudoers
# Hardened rule: Prevents Vim and Less from spawning subshells:
alex ALL=(root) NOEXEC: /usr/bin/less /var/log/nginx/*
```
If Alex attempts to run `!bash` inside `less`, the system displays:
`Cannot execute shell: Permission denied`

---

## 3. Command Hardening Best Practices

### A. Always Use Absolute Binary Paths
Never specify commands without absolute paths:
```sudoers
# BAD (Vulnerable to PATH hijacking):
alex ALL=(root) systemctl

# GOOD (Secure):
alex ALL=(root) /usr/bin/systemctl
```

### B. Avoid Wildcards (`*`) in Command Arguments
Wildcards allow argument injection:
```sudoers
# DANGEROUS:
alex ALL=(root) /bin/cat /var/log/*
# Alex runs: sudo cat /var/log/../../etc/shadow  <-- Reads root password hashes!

# SECURE:
alex ALL=(root) /bin/cat /var/log/app/app.log
```

---

## 4. Enterprise Sudo Hardening Defaults (`/etc/sudoers`)

Place these global hardening directives in `/etc/sudoers` or `/etc/sudoers.d/00-hardening`:

```sudoers
# 1. Reset environment variables to prevent LD_PRELOAD hijacking:
Defaults    env_reset

# 2. Enforce a clean, unmodifiable PATH for all sudo commands:
Defaults    secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin"

# 3. Reduce password cache timeout from 15 minutes to 5 minutes:
Defaults    timestamp_timeout=5

# 4. Enforce mandatory password re-prompt on every invocation (Zero Trust):
# Defaults    timestamp_timeout=0

# 5. Route all sudo logs to a dedicated, append-only log file:
Defaults    logfile="/var/log/sudo.log"

# 6. Include full input/output logs (replayable session recordings!):
# Defaults    log_input, log_output
# Defaults    iolog_dir="/var/log/sudo-io"
```

---

## 5. Production Sudo Security Checklist

| Check | Objective | Hardening Standard |
| :--- | :--- | :--- |
| **No Universal NOPASSWD** | Prevent lateral movement | Human developers must never have `NOPASSWD: ALL`. |
| **No Shell-Spawning Sudo** | Stop GTFOBins escapes | Never grant sudo to `vim`, `nano`, `less`, `awk`, `find`, or `python` without `NOEXEC`. |
| **Enforce `secure_path`** | Prevent binary hijacking | Ensure `Defaults secure_path` is active in sudoers. |
| **Centralized Logging** | Compliance & Auditing | Forward `/var/log/auth.log` or `/var/log/sudo.log` to SIEM / Datadog / CloudWatch. |
| **Dedicated Drop-Ins** | Maintainability | Use `/etc/sudoers.d/` with mode `0440` instead of modifying `/etc/sudoers`. |

---

## Summary

- Never grant `sudo` access to tools that can spawn subshells (`vim`, `less`, `find`, `awk`) without the `NOEXEC` tag.
- Always qualify command paths with absolute paths (`/usr/bin/systemctl`).
- Restrict wildcards (`*`) in arguments to prevent directory traversal attacks.
- Reduce password caching windows using `Defaults timestamp_timeout=5`.
- Ship dedicated sudo audit logs to centralized SIEM platforms for security compliance.
