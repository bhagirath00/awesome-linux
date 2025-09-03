# Chapter 21: Linux Security - Production Linux Hardening Checklist

## 1. Overview: The Enterprise Hardening Baseline

This production checklist consolidates industry best practices from CIS Benchmarks, NIST SP 800-123, and DISA STIG guidelines into an actionable, step-by-step hardening reference for Linux system administrators and DevOps engineers.

```
+--------------------------------------------------------------------------+
|                 Production Linux Hardening Pillars                       |
+--------------------------------------------------------------------------+
| [1] Identity & Access  --> Keys only, sudo rules, disable root login     |
| [2] Network & Firewall --> Default deny inbound, rate limiting, UFW/fw-d |
| [3] Kernel & sysctl    --> ASLR, SYN cookies, disable IP redirects/source|
| [4] Filesystem Security--> noexec/nosuid mounts, umask 027, permissions  |
| [5] MAC Protection     --> SELinux Enforcing or AppArmor Enforce         |
| [6] Auditing & Alerts  --> auditd syscall tracking, Fail2ban, Lynis scan |
+--------------------------------------------------------------------------+
```

---

## 2. Pillar 1: Identity, Authentication, and Access Control

- [ ] **Disable Direct Root Login**:
  Set `PermitRootLogin no` in `/etc/ssh/sshd_config`.
- [ ] **Enforce SSH Key Authentication Only**:
  Set `PasswordAuthentication no` and `KbdInteractiveAuthentication no`.
- [ ] **Implement Least Privilege `sudo` Configuration**:
  Never grant unconditional `ALL=(ALL) NOPASSWD: ALL`. Require passwords for privilege escalation.
- [ ] **Configure Password Complexity and Aging**:
  Edit `/etc/security/pwquality.conf` and `/etc/login.defs`:
  ```ini
  PASS_MAX_DAYS   90
  PASS_MIN_DAYS   7
  PASS_WARN_AGE   14
  ```
- [ ] **Enforce Session Idle Timeouts**:
  Add to `/etc/profile.d/timeout.sh`:
  ```bash
  export TMOUT=900
  readonly TMOUT
  ```

---

## 3. Pillar 2: Network Perimeter and Firewall

- [ ] **Set Default Firewall Stance to Deny Incoming**:
  ```bash
  # UFW
  sudo ufw default deny incoming
  sudo ufw default allow outgoing
  
  # firewalld
  sudo firewall-cmd --set-default-zone=drop
  ```
- [ ] **Expose Only Required Ports**:
  Keep only application listeners open (e.g., 80, 443, and hardened SSH port).
- [ ] **Install and Configure Fail2ban**:
  Activate jails for SSH and web proxies to automatically drop brute-force scanning bots.
- [ ] **Disable Unused Network Protocols (IPv6 if unused, wireless, bluetooth)**:
  Disable Bluetooth and uncommon legacy networking protocols (DCCP, SCTP, RDS).

---

## 4. Pillar 3: Kernel and sysctl Hardening

Create `/etc/sysctl.d/99-security-hardening.conf` to reinforce kernel networking and memory security against IP spoofing, SYN floods, and MITM attacks:

```ini
# Enable SYN flood protection
net.ipv4.tcp_syncookies = 1

# Ignore ICMP broadcast echo requests (Smurf attack prevention)
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Do not send ICMP redirects
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0

# Do not accept ICMP redirects (prevents routing table tampering)
net.ipv4.conf.all.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0

# Do not accept IP source routed packets
net.ipv4.conf.all.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0

# Enable reverse path filtering (prevents IP spoofing)
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# Log martian packets (packets with impossible source addresses)
net.ipv4.conf.all.log_martians = 1

# Restrict dmesg buffer to root only
kernel.dmesg_restrict = 1

# Enable full Address Space Layout Randomization (ASLR)
kernel.randomize_va_space = 2

# Restrict ptrace execution to processes with predefined child relationship
kernel.yama.ptrace_scope = 1
```

Apply immediately without reboot:
```bash
sudo sysctl --system
```

---

## 5. Pillar 4: Filesystem and Mount Security

- [ ] **Harden Shared Memory and Temporary Partitions**:
  In `/etc/fstab`, mount `/tmp`, `/var/tmp`, and `/dev/shm` with `nosuid`, `nodev`, and `noexec` options:
  ```fstab
  tmpfs     /dev/shm     tmpfs     defaults,nodev,nosuid,noexec     0 0
  /dev/sdX  /tmp         ext4      defaults,nodev,nosuid,noexec     0 2
  ```
- [ ] **Enforce Restrictive System umask**:
  Set default umask to `027` in `/etc/profile` and `/etc/bashrc` (new files created as `640`, directories as `750`).
- [ ] **Audit SUID and SGID Binaries**:
  Find all executables running with root SUID bits and strip unnecessary ones:
  ```bash
  sudo find / -perm /6000 -type f -exec ls -ld {} +
  # Strip SUID bit if not needed
  sudo chmod u-s /usr/bin/traceroute
  ```

---

## 6. Pillar 5: Mandatory Access Control (MAC)

- [ ] **Verify SELinux or AppArmor is Active and Enforcing**:
  - RHEL/Rocky: Verify `sestatus` reports `Current mode: enforcing`.
  - Ubuntu/Debian: Verify `sudo aa-status` reports profiles in enforce mode.
  - **Never** permanently set SELinux to `disabled` on production servers.

---

## 7. Pillar 6: Automated Auditing and Patching

- [ ] **Enable Unattended Security Upgrades**:
  - Ubuntu/Debian: Install and configure `unattended-upgrades`.
  - RHEL/Rocky: Enable `dnf-automatic`.
- [ ] **Enable auditd Kernel Tracking**:
  Persist audit watches on `/etc/passwd`, `/etc/shadow`, `/etc/sudoers`, and system calls.
- [ ] **Schedule Recurring Lynis Compliance Scans**:
  Run weekly cron scans:
  ```bash
  lynis audit system --cronjob --quiet
  ```

---

## 8. Summary Takeaways

- Hardening is an ongoing operational lifecycle, not a one-time setup.
- Layer authentication controls, firewall rules, kernel sysctl parameters, and MAC enforcement.
- Automate security patch management and regularly verify compliance with Lynis.
