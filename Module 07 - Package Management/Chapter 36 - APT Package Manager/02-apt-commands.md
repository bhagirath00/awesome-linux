# Chapter 36: APT Package Manager - Essential APT Commands and Workflows

## 1. Updating Metadata vs. Upgrading Packages

One of the most critical fundamentals in Debian and Ubuntu administration:
- **`apt update`**: Downloads the **latest package lists and version metadata** from configured repositories. It **does NOT install or upgrade any software**.
- **`apt upgrade`**: Installs new versions of already-installed packages based on the updated metadata lists.

```bash
# Step 1: Refresh repository indexes
sudo apt update

# Step 2: Upgrade all installed packages to their latest versions safely
sudo apt upgrade -y
```

---

## 2. Installing, Reinstalling, and Removing Packages

### Installing Packages
```bash
# Install single package
sudo apt install -y nginx

# Install specific version
sudo apt install -y nginx=1.24.0-1ubuntu1

# Install multiple packages
sudo apt install -y curl git jq htop
```

### Reinstalling Broken Packages
If a configuration file was accidentally deleted or a binary was corrupted:
```bash
sudo apt install --reinstall nginx
```

### Removing vs. Purging Packages
There is a crucial difference between `remove` and `purge`:
- **`apt remove <pkg>`**: Deletes program binaries and libraries, but **leaves configuration files intact in `/etc/`**.
- **`apt purge <pkg>`**: Deletes binaries AND **permanently wipes all associated configuration files from `/etc/`**.

```bash
# Remove nginx but keep /etc/nginx configuration intact
sudo apt remove nginx

# Completely eradicate package AND all custom configuration files
sudo apt purge nginx
```

---

## 3. System Maintenance and Cache Cleaning

Over months of operation, old package archives accumulate in `/var/cache/apt/archives/`, consuming gigabytes of disk space.

```bash
# 1. Autoremove: Remove orphan packages installed as dependencies that are no longer needed
sudo apt autoremove -y

# 2. Clean: Delete ALL downloaded .deb archives from /var/cache/apt/archives/
sudo apt clean

# 3. Autoclean: Delete ONLY obsolete .deb archives that can no longer be downloaded
sudo apt autoclean
```

---

## 4. Searching and Inspecting Packages

```bash
# Search repository catalog for keywords
apt search postgresql

# Display detailed package metadata (Version, Size, Dependencies, Description)
apt show nginx
```

```
Package: nginx
Version: 1.24.0-1ubuntu1
Priority: optional
Section: httpd
Origin: Ubuntu
Maintainer: Ubuntu Developers <ubuntu-devel-discuss@lists.ubuntu.com>
Installed-Size: 1,420 kB
Depends: libc6 (>= 2.38), libpcre2-8-0, libssl3, nginx-common (= 1.24.0-1ubuntu1)
Download-Size: 540 kB
APT-Sources: http://archive.ubuntu.com/ubuntu noble/main amd64 Packages
Description: small, powerful, scalable web/proxy server
```

### Listing Installed Packages:
```bash
# List all packages currently installed on the system
apt list --installed

# Check if a specific package is installed
apt list --installed | grep redis
```

---

## 5. Safe Automation in CI/CD and Dockerfiles

When running `apt` inside unattended scripts or Docker container builds:
1. Set `DEBIAN_FRONTEND=noninteractive` to suppress interactive curses prompts (e.g., timezone selection).
2. Clean up cache in the **same Docker layer** to minimize image size:

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    git \
    ca-certificates \
 && rm -rf /var/lib/apt/lists/*
```
The `--no-install-recommends` flag prevents APT from pulling in hundreds of non-essential optional packages!

---

## 6. APT Command Quick Reference

| Command | Function |
| :--- | :--- |
| `apt update` | Refresh package lists from remote repositories. |
| `apt upgrade -y` | Upgrade all installed packages to latest compatible versions. |
| `apt full-upgrade` | Upgrade system, installing or removing packages if dependencies require. |
| `apt install <pkg>` | Download and install package and dependencies. |
| `apt remove <pkg>` | Remove package binaries, keeping `/etc` configs. |
| `apt purge <pkg>` | Remove package binaries AND wipe all `/etc` configs. |
| `apt autoremove` | Remove orphan libraries and unused dependencies. |
| `apt clean` | Clear local downloaded archive cache (`/var/cache/apt/archives`). |
| `apt show <pkg>` | Display detailed package description and dependencies. |

---

## 7. Summary Takeaways

- Always run `apt update` before `apt install` or `apt upgrade`.
- Use `apt purge` when you want a clean slate without leftover `/etc/` config files.
- Reclaim disk storage on servers using `apt autoremove` and `apt clean`.
- In Dockerfiles, always use `--no-install-recommends` and remove `/var/lib/apt/lists/*`.
