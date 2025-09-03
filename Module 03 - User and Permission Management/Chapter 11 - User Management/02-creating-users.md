# Creating Users in Linux: `useradd` and `adduser`

## Overview

User provisioning is a central responsibility in systems administration and DevOps automation. When onboarding team members, configuring CI/CD deployment agents, or spinning up cloud servers via Infrastructure as Code (Terraform, Ansible, cloud-init), user accounts must be created consistently, securely, and predictably.

Linux provides two distinct commands for creating users:
1. **`useradd`**: The standard, low-level POSIX binary utility available on **every** Linux distribution (RHEL, Debian, Alpine, Arch, SUSE).
2. **`adduser`**: A user-friendly, interactive Perl script wrapper primarily found on Debian and Ubuntu distributions.

---

## 1. Comparing `useradd` vs `adduser`

| Feature | `useradd` (System Standard) | `adduser` (Debian/Ubuntu Wrapper) |
| :--- | :--- | :--- |
| **Availability** | All Linux distributions (Universal). | Debian, Ubuntu, and derivatives. |
| **Interface** | Pure CLI flags; non-interactive by default. | Interactive prompts (asks for password, full name, room #). |
| **Creates Home Dir by Default?** | **No** (Must pass `-m` on Debian/Ubuntu). | **Yes** (Automatically creates `/home/<user>`). |
| **Copies `/etc/skel` by Default?** | Only if `-m` is specified. | Yes automatically. |
| **Prompts for Password?** | No (Account created in locked state). | Yes (Interactive password prompt). |
| **Automation & Scripting** | **The industry standard for Bash, Docker, CI/CD, and Ansible.** | Not suitable for automated non-interactive scripts. |

---

## 2. The `useradd` Command: Syntax and Essential Flags

```bash
useradd [OPTIONS] USERNAME
```

### Essential Flags Reference

| Flag | Long Option | Description | DevOps Practical Context |
| :--- | :--- | :--- | :--- |
| **`-m`** | `--create-home` | Creates the user's home directory (`/home/username`). | **Crucial flag**: ensures home folder exists and populates dotfiles from `/etc/skel`. |
| **`-s`** | `--shell` | Defines the user's login shell. | Setting `/bin/bash` for developers, or `/usr/sbin/nologin` for daemons. |
| **`-u`** | `--uid` | Sets a specific numerical User ID (UID). | Synchronizing UID across multi-node Kubernetes clusters or NFS storage. |
| **`-g`** | `--gid` | Specifies the user's primary group name or GID. | Setting default group ownership. |
| **`-G`** | `--groups` | Comma-separated list of **supplementary groups**. | Granting access to `sudo`, `docker`, or `adm`. |
| **`-c`** | `--comment` | GECOS field (Full name, service role, or department). | Documenting ownership of automated service accounts. |
| **`-d`** | `--home-dir` | Overrides the default home directory path. | Pointing web accounts directly to `/var/www/app`. |
| **`-r`** | `--system` | **Creates a system account** (allocates low UID < 1000, no home dir). | Creating service accounts for microservices and daemons. |
| **`-e`** | `--expiredate` | Sets an expiration date (`YYYY-MM-DD`). | Creating temporary contractor accounts that auto-disable. |

---

## 3. Practical User Creation Recipes

### A. Creating a Standard Human Developer Account
Create a developer user named `alex` with home directory, Bash shell, and access to `sudo` and `docker`:

```bash
sudo useradd -m -s /bin/bash -c "Alex Rivera (DevOps Team)" -G sudo,docker alex
```

### B. Creating a Non-Root System Service User
Create an unprivileged daemon account named `prometheus` to run metrics collectors securely:

```bash
sudo useradd -r -s /usr/sbin/nologin -c "Prometheus Daemon Service" -d /var/lib/prometheus prometheus
```
- `-r` : Allocates an unprivileged system UID (< 1000).
- `-s /usr/sbin/nologin` : Forbids interactive shell logins.

### C. Creating a Temporary Contractor Account
Create an account that automatically expires on December 31, 2026:

```bash
sudo useradd -m -s /bin/bash -e 2026-12-31 -c "Temporary Security Auditor" auditor
```

---

## 4. Under the Hood: `/etc/skel` (The Skeleton Directory)

When `useradd -m` creates a user's home directory, how does that new directory get populated with `.bashrc`, `.bash_logout`, and `.profile`?

Linux copies the contents of **`/etc/skel`** into the new user's home directory and re-assigns ownership to that user:

```bash
$ ls -la /etc/skel
drwxr-xr-x   2 root root 4096 Oct 14 00:00 .
drwxr-xr-x 135 root root 4096 Oct 14 00:00 ..
-rw-r--r--   1 root root  220 Jan 07  2024 .bash_logout
-rw-r--r--   1 root root 3771 Jan 07  2024 .bashrc
-rw-r--r--   1 root root  807 Jan 07  2024 .profile
```

> [!TIP]
> **DevOps Automation Pattern**: If your organization requires all new developer accounts to have standardized bash aliases, company proxy environment variables, or custom terminal prompts, add those configuration files directly into `/etc/skel` before creating user accounts.

---

## 5. Automated CI/CD & Cloud-Init Provisioning Script

Here is the standard non-interactive bash recipe for creating an automated deployment user (`deployer`) configured with passwordless sudo and authorized SSH key authentication:

```bash
#!/usr/bin/env bash
set -euo pipefail

USERNAME="deployer"
SSH_PUBKEY="ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGo4D8X1m9ZqR9Yx..."

# Step 1: Create user non-interactively
useradd -m -s /bin/bash -c "CI/CD Deployment Service Account" -G sudo "$USERNAME"

# Step 2: Configure SSH directory with strict permissions
USER_HOME="/home/$USERNAME"
mkdir -m 700 "$USER_HOME/.ssh"
echo "$SSH_PUBKEY" > "$USER_HOME/.ssh/authorized_keys"
chmod 600 "$USER_HOME/.ssh/authorized_keys"
chown -R "$USERNAME:$USERNAME" "$USER_HOME/.ssh"

# Step 3: Grant passwordless sudo for automation
echo "$USERNAME ALL=(ALL) NOPASSWD:ALL" > "/etc/sudoers.d/99-$USERNAME"
chmod 440 "/etc/sudoers.d/99-$USERNAME"

echo "User $USERNAME provisioned successfully."
```

---

## Summary

- Use `useradd` for all automated scripts, Dockerfiles, and cross-platform provisioning.
- Always include `-m` (create home directory) and `-s /bin/bash` when creating interactive user accounts.
- Create service accounts using `useradd -r -s /usr/sbin/nologin <daemon>`.
- Default dotfiles are cloned into new home directories from `/etc/skel`.
- Add users to supplementary groups like `sudo` and `docker` using `-G`.
