# Troubleshooting Linux Boot Failures, Grub, and Emergency Mode

## 1. Architectural Concept & Deep Dive

The Linux boot process is a multi-stage handoff sequence starting from physical hardware firmware and terminating at userspace systemd services. When a node fails to boot after a kernel upgrade, configuration change, or storage migration, diagnosing the issue requires identifying precisely which stage failed:

```
+─────────────────────────────────────────────────────────────────────────+
|                           LINUX BOOT SEQUENCE                           |
|                                                                         |
|  1. Firmware: UEFI / BIOS executes POST, locates EFI System Partition   |
|         │                                                               |
|         ▼                                                               |
|  2. Bootloader: GRUB2 presents menu, loads vmlinuz & initramfs into RAM |
|         │                                                               |
|         ▼                                                               |
|  3. Kernel Initialization: vmlinuz decompresses, initializes hardware   |
|         │                                                               |
|         ▼                                                               |
|  4. Initramfs (Early Userspace): Loads storage drivers (NVMe/RAID),     |
|     mounts real rootfs (/sysroot), executes switch_root                 |
|         │                                                               |
|         ▼                                                               |
|  5. Real Root: Systemd (PID 1) parses /etc/fstab, starts default target |
|         │                                                               |
|         ▼                                                               |
|  6. Multi-User Target: Login prompt & network services active           |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Boot Failure Stages & Symptoms

| Stage | Symptoms & Error Messages | Typical Root Cause | Recovery Method |
| :--- | :--- | :--- | :--- |
| **GRUB / Bootloader** | `grub rescue>` prompt<br>`error: no such partition` | Corrupted MBR/EFI, disk UUID changed after hypervisor clone. | Set `root` and `prefix` manually, reinstall GRUB. |
| **Kernel Decompression** | Screen freezes immediately after selecting kernel in GRUB. | Incompatible ACPI tables, unsupported CPU instruction set. | Add `nomodeset` or boot previous kernel version. |
| **Initramfs / Early Root** | `Kernel panic - not syncing: VFS: Unable to mount root fs`<br>`(initramfs)` shell prompt | Missing storage controller drivers in initramfs, wrong `root=` UUID. | Rebuild initramfs (`update-initramfs` or `dracut`). |
| **Systemd / Filesystem** | `You are in emergency mode. Cannot open access to console...` | Syntax error or missing disk UUID in `/etc/fstab`. | Boot with `systemd.unit=rescue.target`, edit `/etc/fstab`. |

---

## 3. The Number One Boot Failure: Broken `/etc/fstab`

In 80% of post-maintenance boot failures, the culprit is a non-existent or misconfigured mount point in `/etc/fstab`.

By default, Systemd converts every line in `/etc/fstab` into a mandatory `.mount` unit. If an external data volume or secondary drive fails to mount (e.g., EBS volume detached or NFS server unreachable), **Systemd refuses to boot and dumps the server into Emergency Mode!**

### The Fix from Emergency Shell:
1. When prompted with `Enter root password for maintenance (or press Control-D to continue)`: Enter password.
2. The root filesystem will likely be mounted **read-only**:
   ```bash
   # Remount root as read-write
   mount -o remount,rw /
   ```
3. Inspect systemd mount errors:
   ```bash
   journalctl -xb | grep -i "failed to mount"
   ```
4. Comment out the failing mount line in `/etc/fstab` or add the **`nofail`** option:
   ```
   # Adding nofail prevents systemd from blocking boot if disk is missing!
   UUID=3a4b5c6d-1234 /data ext4 defaults,nofail 0 2
   ```
5. Reboot: `systemctl reboot`.

---

## 4. Bypassing Systemd via GRUB (`init=/bin/bash`)

If the root password is lost, or systemd fails so severely that emergency mode cannot spawn a shell, you can bypass the entire init system directly from GRUB.

### Step-by-Step Procedure:
1. Reboot the server and access the console (IPMI / iLO / AWS EC2 Serial Console).
2. Highlight the desired Linux kernel entry and press **`e`** to edit boot parameters.
3. Locate the line starting with `linux` or `linux16` (or `linuxefi`).
4. Move cursor to the end of the line, remove `rhgb quiet` (to enable verbose boot logs), and append:
   ```
   init=/bin/bash rw
   ```
5. Press **`Ctrl+X`** or **`F10`** to boot.
6. The kernel boots and hands execution directly to bash (PID 1), granting an immediate root shell without prompting for a password!
7. Reset root password or repair corrupt configuration:
   ```bash
   passwd root
   # On SELinux systems (RHEL/Rocky/Fedora), force relabel on next boot:
   touch /.autorelabel
   exec /sbin/init
   ```

---

## 5. Rebuilding Corrupted Initramfs

If the kernel panics with `Unable to mount root fs`:

### On Debian / Ubuntu:
```bash
# Rebuild initramfs for currently running kernel
sudo update-initramfs -u -k all

# Verify kernel images exist in /boot
ls -lh /boot/vmlinuz* /boot/initrd*
```

### On RHEL / CentOS / Rocky / Fedora:
```bash
# Rebuild initramfs using dracut
sudo dracut --regenerate-all --force
```

---

## 6. Bare-Metal Recovery via Live CD & `chroot`

When a system refuses to boot from disk completely:
1. Boot the server using an Ubuntu or Rocky Linux Live ISO.
2. Assemble and mount the damaged root filesystem:
   ```bash
   # Mount root partition
   sudo mount /dev/nvme0n1p2 /mnt

   # Mount EFI partition
   sudo mount /dev/nvme0n1p1 /mnt/boot/efi

   # Bind mount virtual filesystems
   for dir in /dev /dev/pts /proc /sys /run; do
       sudo mount --bind $dir /mnt$dir
   done

   # Enter the broken OS installation
   sudo chroot /mnt /bin/bash
   ```
3. Inside the `chroot` environment:
   ```bash
   # Reinstall GRUB bootloader to disk
   grub-install /dev/nvme0n1
   update-grub

   # Reinstall kernel if deleted
   apt-get install --reinstall linux-image-generic
   ```
4. Exit and reboot:
   ```bash
   exit
   sudo reboot
   ```

---

## 7. Verification & Troubleshooting Runbook

### Health Verification Commands
```bash
# Measure boot time performance per stage
systemd-analyze

# Identify slowest starting services delaying boot
systemd-analyze blame

# Render critical boot chain dependency tree
systemd-analyze critical-chain
```

### Common Pitfalls & Solutions
1. **GRUB menu is hidden during boot**:
   - Cause: `GRUB_TIMEOUT=0` or `GRUB_TIMEOUT_STYLE=hidden` in `/etc/default/grub`.
   - Solution: Press and hold `Shift` (BIOS) or `Esc` (UEFI) repeatedly immediately after power-on.
2. **Cloud instance fails boot after resizing EBS/NVMe disk**:
   - Cause: Partition table not expanded or filesystem UUID altered.
   - Solution: Verify `/etc/fstab` uses filesystem UUIDs (`UUID=...`) rather than raw device paths (`/dev/sda1`), which can change order on reboot.
3. **`Timed out waiting for device /dev/disk/by-uuid/...`**:
   - Cause: Disk detached or storage array offline.
   - Solution: In `/etc/fstab`, ensure non-critical drives use `nofail,x-systemd.device-timeout=5s`.

---

## 8. Key Takeaways
- The boot sequence progresses through UEFI -> GRUB2 -> Kernel -> Initramfs -> Systemd.
- Broken `/etc/fstab` entries cause most emergency mode boot loops; prevent this by adding `nofail` to secondary volumes.
- Pass `init=/bin/bash rw` in GRUB to bypass systemd and recover locked systems without passwords.
- Always regenerate initramfs (`update-initramfs -u` / `dracut -f`) whenever upgrading disk drivers or cloning root partitions.
