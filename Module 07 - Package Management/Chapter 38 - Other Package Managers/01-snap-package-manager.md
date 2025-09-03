# Chapter 38: Other Package Managers - Snap Package Manager (snapd)

## 1. Introduction: What is Snap?

**Snap** is a software packaging, deployment, and sandboxing system developed by Canonical (the company behind Ubuntu). 

Traditional package managers (`apt`, `dnf`) install files directly into shared system directories (`/usr/bin`, `/usr/lib`). If App A requires `libssl.so.1.1` and App B requires `libssl.so.3.0`, a "dependency conflict" occurs.

A **Snap package** (or "snap") bundles the application binary together with **all of its required dependencies, libraries, assets, and runtimes** inside a single, read-only compressed **SquashFS** filesystem image. When executed, `snapd` mounts the SquashFS image as a loop device and executes the binary inside an isolated sandbox using AppArmor, seccomp, and Linux namespaces.

```
+--------------------------------------------------------------------------+
|                         Snap Execution Architecture                      |
+--------------------------------------------------------------------------+
|  User invokes: certbot (Snap Package)                                    |
|                                |                                         |
|                                v                                         |
|  snapd Background Daemon                                                 |
|  - Mounts /var/lib/snapd/snaps/certbot_3412.snap (SquashFS Loop Device)  |
|  - Mount Point: /snap/certbot/current                                    |
|                                |                                         |
|                                v Confinement & Security Sandbox          |
|  - AppArmor Profile (Restricts filesystem write boundaries)              |
|  - Seccomp Filter (Blocks forbidden kernel syscalls)                     |
|  - Linux Namespaces (Mount & PID isolation)                              |
|                                |                                         |
|                                v                                         |
|  Execution Container (Runs with self-contained Python & OpenSSL runtime) |
+--------------------------------------------------------------------------+
```

---

## 2. Managing the snapd Service

`snapd` is pre-installed on Ubuntu, but can also be installed on Debian, RHEL, Rocky Linux, and Fedora:

```bash
# On RHEL / Rocky / AlmaLinux:
sudo dnf install -y epel-release
sudo dnf install -y snapd
sudo systemctl enable --now snapd.socket

# Enable classic snap support symlink:
sudo ln -s /var/lib/snapd/snap /snap
```

---

## 3. Essential Snap Commands

### 1. Searching for Software
```bash
snap find certbot
```

### 2. Installing Packages
The most common enterprise server use case for Snap is **Certbot** (the official Let's Encrypt SSL/TLS certificate client, which the Electronic Frontier Foundation exclusively distributes via Snap):

```bash
# Install Certbot with classic confinement
sudo snap install --classic certbot

# Create symlink so certbot is in system PATH
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

### 3. Listing Installed Snaps
```bash
snap list
```

```
Name     Version    Rev    Tracking       Publisher     Notes
certbot  2.9.0      3412   latest/stable  certbot-eff*  classic
core22   20240111   1122   latest/stable  canonical*    base
snapd    2.61.1     20674  latest/stable  canonical*    snapd
```

### 4. Updating and Refreshing Snaps
By default, `snapd` checks for and automatically updates installed snaps up to 4 times a day in the background. You can force a manual update:
```bash
sudo snap refresh
```

### 5. Reverting to Previous Versions
If an automatic snap update breaks an application, Snap keeps prior revisions, allowing instantaneous rollbacks:
```bash
sudo snap revert certbot
```

### 6. Removing Snaps
```bash
sudo snap remove certbot
```

---

## 4. Snap Confinement Models: Strict vs. Classic

Every snap package operates under one of two primary security models:

| Confinement Model | Flag | Security Boundaries | Use Case |
| :--- | :--- | :--- | :--- |
| **Strict (Default)** | None | Heavily sandboxed. Process cannot read user home directories, network devices, or arbitrary system files unless explicit "plugs/interfaces" are granted. | Microservices, web apps, isolated CLI tools. |
| **Classic** | `--classic` | **Unconfined**. The application has full access to the host operating system filesystem and system calls, exactly like a standard `.deb` or `.rpm` package. | Development compilers, IDEs, system administration tools (Certbot). |

---

## 5. Pros and Cons of Snap in DevOps

### Advantages:
- **Zero Dependency Conflicts**: Applications bundle their exact runtime dependencies.
- **Always Up-to-Date**: Automatic background updates deliver rapid security patches.
- **Atomic Rollbacks**: One-command rollbacks (`snap revert`).

### Disadvantages:
- **Loop Device Pollution**: Every snap creates a mounted virtual loop device, cluttering `lsblk` and `df` output (`/dev/loop0`, `/dev/loop1`).
- **Slower Cold Startup Times**: Decompressing SquashFS images causes slower execution starts compared to native ELF binaries.
- **Proprietary Backend**: The central Snap Store backend is proprietary Canonical infrastructure, unlike open decentralized APT/RPM repositories.

---

## 6. Summary Takeaways

- Snap packages bundle all dependencies into self-contained, read-only SquashFS images.
- Managed via the `snapd` background daemon.
- Certbot is the most common production use case for Snap on Linux servers.
- Use `--classic` when installing tools that require full system filesystem access.
- Roll back problematic updates instantly using `snap revert <package>`.
