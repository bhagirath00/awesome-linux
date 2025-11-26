# Modifying Users in Linux (`usermod`)

## Overview

The `usermod` (user modify) command updates an existing user account's system attributes: group memberships, default shell, home directory path, account expiration dates, and account locking status.

In DevOps operations, `usermod` is most frequently used for:
- Granting developers access to administrative groups (`sudo`, `wheel`) or container runtimes (`docker`).
- Locking compromised accounts during security incident responses.
- Transitioning legacy default shells to modern shells (`/bin/bash` -> `/bin/zsh`).
- Migrating application user home directories to dedicated persistent storage disks.

---

## Command Syntax and Essential Flags

```bash
usermod [OPTIONS] USERNAME
```

### Essential Flags Reference

| Flag | Long Option | Description | DevOps Importance |
| :--- | :--- | :--- | :--- |
| **`-aG`** | `--append --groups` | **Appends the user to supplementary groups**. | **The single most important flag pairing in user administration.** |
| **`-g`** | `--gid` | Modifies the user's **primary group**. | Changing default file ownership group. |
| **`-s`** | `--shell` | Changes the user's login shell. | Restricting access (`/usr/sbin/nologin`) or changing default shell. |
| **`-d`** | `--home` | Sets a new home directory path. | Pointing user directory to a new mount point. |
| **`-m`** | `--move-home` | Moves existing home directory contents to the new `-d` path. | Must be combined with `-d` to migrate existing files safely. |
| **`-l`** | `--login` | Changes the account's username. | Handling corporate name changes or service renaming. |
| **`-L`** | `--lock` | **Locks the user's password**. | Disables password login during offboarding or security alerts. |
| **`-U`** | `--unlock` | Unlocks the user's password. | Restoring access. |
| **`-e`** | `--expiredate` | Sets account expiration date (`YYYY-MM-DD`). | Auto-expiring contractor access. |

---

## 1. The Critical DevOps Pitfall: Forgetting `-a` with `-G`

This is one of the most infamous mistakes in Linux administration:

```text
CORRECT:
usermod -aG docker alex
  ──> The '-a' (append) flag PRESERVES all existing group memberships (sudo, adm, staff)
      and safely ADDS alex to the 'docker' group.

CATASTROPHIC:
usermod -G docker alex
  ──> WITHOUT '-a', usermod REPLACES all existing supplementary groups with ONLY 'docker'!
      Alex is immediately STRIPPED of 'sudo' permissions and LOCKED OUT of root access!
```

> [!WARNING]
> **Always verify before executing**: When adding a user to a group, ensure you type **`usermod -aG <group> <user>`**.
> Check memberships immediately after running using: `id <user>`

---

## 2. High-Frequency DevOps Recipes

### A. Granting Docker Management Permissions
To allow a developer to run `docker` commands without prepending `sudo`:

```bash
# Add user 'ubuntu' to the docker group:
sudo usermod -aG docker ubuntu

# Note: The user must log out and log back in (or run 'newgrp docker') for group changes to take effect!
```

### B. Changing a User's Default Login Shell
Switch a user's interactive shell to Zsh or Bash:

```bash
sudo usermod -s /bin/zsh alex

# Verify change in /etc/passwd:
grep alex /etc/passwd
# alex:x:1001:1001::/home/alex:/bin/zsh
```

*(Alternatively, users can change their own shell using the `chsh -s /bin/bash` command).*

### C. Migrating a User's Home Directory to New Storage
When moving an application user's data from a full root disk to a mounted NVMe disk at `/data/home/deploy`:

```bash
# -d specifies the new path; -m moves all existing files:
sudo usermod -d /data/home/deploy -m deploy
```

### D. Locking and Unlocking Accounts (Security Triage)
During an offboarding event or suspected compromised credential incident:

```bash
# Instantly lock user password:
sudo usermod -L bad_actor

# Inspect /etc/shadow:
sudo grep bad_actor /etc/shadow
# bad_actor:!$6$qZ7...  <-- The '!' prepended to the password hash disables password login!

# Unlock account:
sudo usermod -U bad_actor
```

> [!IMPORTANT]
> **SSH Key Alert**: `usermod -L` locks the **password** entry in `/etc/shadow`. If the user has SSH public keys configured in `~/.ssh/authorized_keys`, they can **still log in via SSH keys**! To completely lock out a compromised user:
> 1. Lock password: `usermod -L <user>`
> 2. Change shell to nologin: `usermod -s /usr/sbin/nologin <user>`
> 3. Expire the account: `usermod -e 1 <user>`
> 4. Kill active processes: `pkill -u <user>`

---

## Summary

- Always use `usermod -aG <group> <user>` with the `-a` append flag; omitting `-a` strips all other group memberships.
- Migrate home directories safely using `usermod -m -d /new/path <user>`.
- Change login shells with `usermod -s /path/to/shell <user>`.
- Lock passwords with `usermod -L <user>` and unlock with `usermod -U <user>`.
- To completely revoke access, pair `usermod -L` with setting shell to `/usr/sbin/nologin` and killing active user processes with `pkill -u <user>`.
