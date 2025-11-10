# Runlevels & Systemd Targets

## Introduction

In Linux, a system can operate in different operating modes — from an emergency maintenance shell to a fully networked multi-user server, to a desktop graphical workstation.

Historically in Unix and older Linux distributions, these states were called **Runlevels**. In modern Linux systems managed by **`systemd`**, runlevels have been replaced by **Target Units (`.target`)**.

---

## 1. Traditional Runlevels vs Modern Systemd Targets

```text
+----------+-----------------------+-----------------------------------------------+
| RUNLEVEL |    SYSTEMD TARGET     |                  DESCRIPTION                  |
+----------+-----------------------+-----------------------------------------------+
|    0     | poweroff.target       | Halt and power down the physical machine.     |
+----------+-----------------------+-----------------------------------------------+
|    1 / S | rescue.target         | Single-user maintenance mode (no networking,  |
|          |                       | minimal services, root shell for repair).     |
+----------+-----------------------+-----------------------------------------------+
|    2     | multi-user.target     | Multi-user mode without networking (rare).    |
+----------+-----------------------+-----------------------------------------------+
|    3     | multi-user.target     | Standard Headless Server State:               |
|          |                       | Full networking, multi-user, CLI only.        |
|          |                       | *The default target for 99% of cloud servers* |
+----------+-----------------------+-----------------------------------------------+
|    4     | multi-user.target     | User-definable / unused.                      |
+----------+-----------------------+-----------------------------------------------+
|    5     | graphical.target      | Full Multi-user + GUI Desktop (X11 / Wayland) |
+----------+-----------------------+-----------------------------------------------+
|    6     | reboot.target         | Reboot the system.                            |
+----------+-----------------------+-----------------------------------------------+
|   N/A    | emergency.target      | Minimal shell with root filesystem read-only  |
|          |                       | (used when fsck fails on corrupted disks).    |
+----------+-----------------------+-----------------------------------------------+
```

---

## 2. Managing Targets with `systemctl`

### A. Checking the Current Default Boot Target
On any production cloud server (AWS EC2, GCP Compute Engine, DigitalOcean), the default target should always be `multi-user.target` to prevent wasting CPU and RAM on display managers:

```bash
# Check what target the server boots into by default
systemctl get-default
# Output on a server:
# multi-user.target
```

### B. Changing the Default Boot Target
If a server was accidentally installed with a desktop environment, you can switch it back to headless CLI mode to reclaim ~1GB of RAM:

```bash
# Set headless multi-user mode as permanent default
sudo systemctl set-default multi-user.target

# Set graphical desktop as permanent default
sudo systemctl set-default graphical.target
```

### C. Switching Targets on the Fly (`isolate`)
You can change the system operating state immediately without rebooting using `isolate`:

```bash
# Switch to rescue mode for filesystem maintenance
sudo systemctl isolate rescue.target

# Return to standard multi-user server mode
sudo systemctl isolate multi-user.target
```

---

## 3. Practical DevOps Scenarios

### Scenario 1: Resetting a Lost Root Password via Rescue Target
If an engineer loses root credentials on an on-premise or bare-metal Linux server:
1. Reboot the server and press `e` at the GRUB menu.
2. Locate the line starting with `linux /boot/vmlinuz-...`.
3. Append `systemd.unit=rescue.target` (or `init=/bin/bash`) to the end of the line.
4. Press `Ctrl + X` to boot.
5. The kernel boots directly into a root maintenance shell without prompting for a password.
6. Mount the root filesystem read-write and reset the password:
   ```bash
   mount -o remount,rw /
   passwd root
   exec /sbin/init
   ```

### Scenario 2: Emergency Target on Corrupted Disk Boot
If an automated server reboot fails because a secondary disk partition listed in `/etc/fstab` failed to mount, `systemd` halts normal boot and drops the console into `emergency.target`:
- Only `/` is mounted, and it is mounted read-only.
- To recover:
  ```bash
  # Remount root as writable
  mount -o remount,rw /

  # Comment out the failing disk entry in /etc/fstab
  nano /etc/fstab

  # Resume normal boot
  systemctl default
  ```
