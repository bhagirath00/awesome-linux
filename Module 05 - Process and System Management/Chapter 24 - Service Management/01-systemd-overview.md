# Chapter 24: Service Management - systemd Architecture and Overview

## 1. Introduction: What is systemd?

**`systemd`** is the standard system and service manager for modern Linux operating systems. Launched in 2010 by Lennart Poettering and Kay Sievers, it was designed to replace the legacy Unix **SysV init** (`/etc/init.d/`, `/etc/rc.local`) and Ubuntu **Upstart** initialization systems.

As **PID 1**, `systemd` is the very first process executed by the Linux kernel during the boot sequence. It is responsible for initializing the user-space environment, mounting filesystems, spawning background service daemons, managing process cgroups, and supervising system lifecycle events (reboot, shutdown).

```
+--------------------------------------------------------------------------+
|                        systemd Kernel & Boot Hierarchy                   |
+--------------------------------------------------------------------------+
|  [ Linux Kernel Initialization ] (vmlinuz & initramfs)                   |
|                              |                                           |
|                              v Spawns PID 1                              |
|                    [ /sbin/init -> systemd (PID: 1) ]                    |
|                              |                                           |
|       +----------------------+----------------------+                    |
|       |                      |                      |                    |
|       v                      v                      v                    |
|  [ Mounts & VFS ]     [ systemd-udevd ]     [ systemd-journald ]         |
|  (/etc/fstab)         (Hardware & Devices)  (Binary Log Collector)       |
|       |                      |                      |                    |
|       +----------------------+----------------------+                    |
|                              |                                           |
|                              v Target: multi-user.target                 |
|       +----------------------+----------------------+                    |
|       |                      |                      |                    |
|       v                      v                      v                    |
|  [ sshd.service ]     [ docker.service ]    [ nginx.service ]            |
+--------------------------------------------------------------------------+
```

---

## 2. Why systemd Replaced SysV Init

The legacy SysV init system suffered from major structural flaws:
1. **Serial Execution (Slow Boot Times)**: SysV init executed complex shell scripts in `/etc/rc*.d/` sequentially one after another. If script `S20network` stalled, the entire boot process froze.
2. **Fragile Process Tracking**: SysV init relied on daemons voluntarily writing their own PIDs into `/var/run/daemon.pid`. If a daemon crashed or forked multiple worker processes, orphaned children were left running undetected.
3. **Complex Boilerplate Shell Scripts**: Every service required a hundreds-of-lines bash script implementing `start`, `stop`, `restart`, `status` with custom sleep loops.

### The systemd Breakthrough:
- **Massive Parallelization**: Daemons are started concurrently using socket-based activation (e.g., listening sockets are created before the daemon starts, allowing dependent services to connect immediately).
- **Cgroup Process Supervision**: Every service and its child processes are placed inside a dedicated **Linux Control Group (cgroup)**. Even if a process forks a thousand worker threads or double-forks to detach, `systemd` tracks every single child and can terminate the entire group cleanly without leaving rogue processes behind.
- **Declarative Unit Files**: Simple, structured INI-style configuration files replace fragile shell scripts.

---

## 3. The Concept of systemd Units

Everything managed by `systemd` is organized as an **Unit**. A unit is defined in a declarative configuration file ending with a specific extension denoting its type:

| Unit Extension | Unit Type | Description & Purpose |
| :--- | :--- | :--- |
| **`.service`** | Service | System daemons and applications (e.g., `nginx.service`, `docker.service`). |
| **`.socket`** | Socket | An IPC socket or network port managed by systemd to enable on-demand service activation. |
| **`.target`** | Target | Groups of other units used for synchronization and runlevels (e.g., `multi-user.target`). |
| **`.timer`** | Timer | Monotonous or calendar-based schedulers (modern replacement for `cron`). |
| **`.mount`** | Mount | Filesystem mount point management (derived from `/etc/fstab`). |
| **`.path`** | Path | Monitors file or directory changes via `inotify` to trigger services. |
| **`.slice`** | Slice | Hierarchical resource management node for cgroup memory and CPU limits. |

---

## 4. Targets vs. Legacy Runlevels

In legacy SysV init, system operational states were represented by numeric **Runlevels** (0 to 6). In `systemd`, these are replaced by **Target Units** (`.target`), which can inherit and include other targets:

```
+-----------------------+-----------------------------+--------------------+
| Legacy SysV Runlevel  | systemd Target Unit         | Operational State  |
+-----------------------+-----------------------------+--------------------+
| Runlevel 0            | poweroff.target             | System shutdown    |
| Runlevel 1 (Single)   | rescue.target               | Single-user repair |
| Runlevel 3 (Multi)    | multi-user.target           | Headless CLI / Net |
| Runlevel 5 (GUI)      | graphical.target            | Full X11/Wayland   |
| Runlevel 6            | reboot.target               | System reboot      |
+-----------------------+-----------------------------+--------------------+
```

### Checking and Changing Default Target:
```bash
# Get current default boot target
systemctl get-default

# Set headless server multi-user target as permanent default
sudo systemctl set-default multi-user.target

# Switch to rescue mode immediately
sudo systemctl isolate rescue.target
```

---

## 5. Configuration File Locations and Precedence

Unit files are stored in three primary filesystem directories with strict loading precedence:

```
HIGHEST PRECEDENCE (Overrides all)
    |
    |   /etc/systemd/system/        <-- Custom system administrator units and overrides
    |
    |   /run/systemd/system/        <-- Transient runtime-generated units (lost on reboot)
    |
    v   /lib/systemd/system/        <-- Upstream vendor package defaults (apt, dnf, rpm)
LOWEST PRECEDENCE
```

> [!TIP]
> Never edit files directly inside `/lib/systemd/system/`! When the package is updated via `apt` or `dnf`, your changes will be overwritten. Always place custom modifications in `/etc/systemd/system/` or use `systemctl edit <unit>`.

---

## 6. Summary Takeaways

- `systemd` is PID 1, orchestrating system initialization, service supervision, and cgroups.
- It replaces fragile SysV init shell scripts with declarative INI-style unit files.
- Control Groups (cgroups) ensure no child process escapes supervision or orphaned kills.
- Legacy runlevels are replaced by Target units (e.g., `multi-user.target`).
- System admin configurations live in `/etc/systemd/system/` and override vendor defaults.
