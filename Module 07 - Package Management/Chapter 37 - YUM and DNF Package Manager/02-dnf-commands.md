# Chapter 37: YUM and DNF Package Manager - Essential DNF Commands and Workflows

## 1. Refreshing Metadata and Upgrading the System

In DNF, updating metadata and upgrading packages is streamlined:

```bash
# Check for available package updates without installing
sudo dnf check-update

# Upgrade all installed packages to their latest versions
sudo dnf upgrade -y

# Synchronize distribution packages strictly to repository states (Distro Sync)
sudo dnf distro-sync -y
```

> [!NOTE]
> Unlike `apt` where you must manually run `apt update` before installing software, **DNF automatically checks for and downloads expired repository metadata in the background** whenever you run an install or search command!

---

## 2. Installing, Removing, and Reinstalling Packages

```bash
# Install a package
sudo dnf install -y nginx

# Install a specific version
sudo dnf install -y nginx-1.24.0-2.el9.x86_64

# Reinstall a corrupted package
sudo dnf reinstall -y nginx

# Remove a package and its orphaned dependencies
sudo dnf remove -y nginx
```

---

## 3. Searching and Querying Packages

```bash
# Search repository catalogs by keyword or name
dnf search postgresql

# Display detailed metadata about a package (Version, repo, size, license)
dnf info nginx
```

```
Name         : nginx
Epoch        : 1
Version      : 1.24.0
Release      : 2.el9
Architecture : x86_64
Size         : 1.4 M
Source       : nginx-1.24.0-2.el9.src.rpm
Repository   : @System
From repo    : appstream
Summary      : A high performance web server and reverse proxy
URL          : http://nginx.org/
License      : BSD-2-Clause
Description  : Nginx is a web server and a reverse proxy server for HTTP, SMTP,
             : POP3 and IMAP protocols, with a strong focus on high
             : concurrency, performance and low memory usage.
```

### Finding Which Package Provides a File (`dnf provides`)
When compiling software or troubleshooting a missing binary or library (e.g., `semanage` or `nslookup`):

```bash
# "Which package contains the 'semanage' binary?"
dnf provides /usr/sbin/semanage
# Or simply:
dnf provides semanage
```
```
policycoreutils-python-utils-3.5-1.el9.noarch : SELinux policy core python utilities
Repo        : appstream
Matched from:
Filename    : /usr/sbin/semanage
```
You immediately know to install `sudo dnf install -y policycoreutils-python-utils`!

---

## 4. Managing Package Groups

DNF groups related packages into cohesive bundles (e.g., Development Tools, Container Management):

```bash
# List available environment groups and package groups
dnf group list
```

```
Available Environment Groups:
   Server with GUI
   Server
   Minimal Install
Installed Groups:
   Development Tools
   Headless Management
```

```bash
# Install entire C/C++ compiler and build chain:
sudo dnf groupinstall -y "Development Tools"

# Remove group
sudo dnf groupremove -y "Development Tools"
```

---

## 5. Transaction History and Rollbacks (`dnf history`)

One of DNF's standout enterprise features is the **Transaction History Engine**. Every install, upgrade, and removal operation is logged as an atomic transaction that can be inspected, undone, or rolled back!

### 1. View Transaction History
```bash
dnf history
```

```
ID     | Command line             | Date and time    | Action(s)      | Altered
-------------------------------------------------------------------------------
    14 | install -y nginx         | 2026-10-14 06:12 | Install        |    4   
    13 | upgrade -y               | 2026-10-12 04:30 | Upgrade        |   28 EE
    12 | install -y postgresql-se | 2026-10-10 11:20 | Install        |    2   
```

### 2. Inspect an Individual Transaction
```bash
dnf history info 14
```

### 3. Undo a Problematic Transaction
If transaction `14` broke your server, undoing it completely removes all packages installed during that specific operation:
```bash
sudo dnf history undo 14 -y
```

---

## 6. DNF Command Cheat Sheet

| Command | Function |
| :--- | :--- |
| `dnf check-update` | Check for available package updates. |
| `dnf upgrade -y` | Perform system-wide package updates. |
| `dnf install <pkg>` | Install package and dependencies. |
| `dnf remove <pkg>` | Uninstall package and unused dependencies. |
| `dnf provides <file>` | Identify which package supplies a specific binary or library path. |
| `dnf groupinstall <group>`| Install bundled collection of packages (e.g., "Development Tools"). |
| `dnf clean all` | Purge local metadata cache and downloaded package archives. |
| `dnf history` | View audit trail of all package management operations. |
| `dnf history undo <id>`| Roll back an entire package transaction. |

---

## 7. Summary Takeaways

- DNF handles metadata updates automatically during installations.
- Use `dnf provides <binary>` to identify which package contains a missing utility.
- Install developer toolchains in one step with `dnf groupinstall "Development Tools"`.
- Use `dnf history` to audit installations and `dnf history undo <id>` for instant rollbacks.
