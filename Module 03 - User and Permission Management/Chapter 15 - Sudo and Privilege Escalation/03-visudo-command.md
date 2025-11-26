# Safe Sudoers Editing: `visudo` and Syntax Validation

## Overview

A single typo, missing colon, or syntax mistake in `/etc/sudoers` can completely disable the `sudo` command on a server:

```text
>>> /etc/sudoers: syntax error near line 28 <<<
sudo: parse error in /etc/sudoers near line 28
sudo: no valid sudoers sources found, quitting
sudo: unable to initialize policy plugin
```

If direct root SSH login is disabled (as is standard on AWS, Azure, and Google Cloud), **a syntax error in `/etc/sudoers` locks you out of your server permanently!**

To eliminate this catastrophe, Linux provides the dedicated **`visudo`** utility. `visudo` safely edits `/etc/sudoers` and `/etc/sudoers.d/` files with **mandatory syntax validation and file locking**.

---

## 1. How `visudo` Protects Your Servers

When you execute `sudo visudo`, the tool executes four protective steps:

```text
1. Acquires a Lock (/etc/sudoers.lock) ──> Prevents concurrent collisions from other admins.
2. Creates Temporary Working Copy     ──> Opens /etc/sudoers.tmp in your editor.
3. Parses Syntax Upon Save (:wq)      ──> Runs internal grammar and syntax parser.
4. Error Trap or Atomic Commit:
   ├── Syntax Valid   ──> Atomically overwrites /etc/sudoers. Changes live!
   └── Syntax INVALID ──> BLOCKS SAVE! Prompts: "What now? [e, x, q, ?]"
```

### The Error Prompt: "What now?"
If you make a syntax mistake and try to exit, `visudo` halts and prompts:

```text
>>> /etc/sudoers: syntax error near line 32 <<<
What now? [e, x, q, ?]
```

- **`e` (Edit)**: Returns directly to the editor at the exact line of the syntax error so you can fix it. *(Always choose this!)*
- **`x` (Exit without saving)**: Discards your changes and exits, leaving the original working file intact.
- **`q` (Quit)**: Saves the broken file anyway! *(NEVER press `q`!)*

---

## 2. Editing Drop-In Files with `visudo -f`

You should always use `visudo` when creating or modifying files in `/etc/sudoers.d/`:

```bash
# Safely create or edit a drop-in file:
sudo visudo -f /etc/sudoers.d/99-ci-runner
```

---

## 3. Automated Validation in CI/CD and Ansible: `visudo -c`

In automated DevOps deployments (e.g., Ansible playbooks or Docker container builds), you cannot use interactive editors. You need to validate sudoers files **non-interactively**.

The **`-c` (`--check`)** flag audits sudoers syntax and returns an exit code:

### Check All Active Sudoers Files:
```bash
$ sudo visudo -c
/etc/sudoers: parsed OK
/etc/sudoers.d/99-ci-runner: parsed OK
/etc/sudoers.d/README: parsed OK
```

### Validate a Newly Generated Drop-In Before Deployment:
```bash
#!/usr/bin/env bash
set -euo pipefail

TMP_SUDOERS=$(mktemp)
cat << 'EOF' > "$TMP_SUDOERS"
deployer ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart api
EOF

# Validate syntax using visudo -cf
if visudo -cf "$TMP_SUDOERS"; then
    echo "Syntax valid. Deploying to /etc/sudoers.d/..."
    install -m 0440 -o root -g root "$TMP_SUDOERS" /etc/sudoers.d/99-deployer
    rm -f "$TMP_SUDOERS"
else
    echo "ERROR: Generated sudoers syntax is invalid! Aborting." >&2
    rm -f "$TMP_SUDOERS"
    exit 1
fi
```

---

## 4. Configuring the Preferred Editor for `visudo`

By default, `visudo` opens in `vim` or `nano`. You can override the editor on the fly or permanently:

```bash
# Temporary override:
sudo EDITOR=nano visudo

# Or using environment variables in ~/.bashrc:
export SUDO_EDITOR=nano
```

---

## 5. Emergency Recovery: What to Do if Sudoers is Corrupted

If someone edited `/etc/sudoers` with standard `nano` and locked everyone out:

1. **Cloud Instances (AWS EC2 / GCP Compute)**:
   - Stop the instance.
   - Detach the root volume and attach it as a secondary disk to a working recovery VM.
   - Mount the volume, fix the syntax error in `/etc/sudoers`, unmount, and reattach to the original instance.
2. **Physical / Virtual Servers (Console Access)**:
   - Reboot into the GRUB bootloader menu.
   - Edit the kernel boot arguments, appending `init=/bin/bash`.
   - Boot into the root single-user shell, remount the disk read-write (`mount -o remount,rw /`), and fix `/etc/sudoers` using `visudo`.

---

## Summary

- Never edit `/etc/sudoers` with standard `vim` or `nano`; always use `visudo`.
- Edit drop-in files safely with `sudo visudo -f /etc/sudoers.d/<filename>`.
- `visudo` prevents server lockouts by validating syntax before committing changes.
- Validate generated files non-interactively in CI/CD pipelines using `visudo -cf <file>`.
