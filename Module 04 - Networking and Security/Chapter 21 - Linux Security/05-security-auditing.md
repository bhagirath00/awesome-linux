# Chapter 21: Linux Security - Security Auditing and auditd

## 1. Introduction: The Linux Audit Framework (auditd)

The Linux Audit Framework is an enterprise-grade kernel subsystem designed to track security-relevant events on a system. Unlike standard application loggers (such as `syslog` or `journald`), which can be disabled or bypassed by root processes, the Linux Audit system operates directly inside the Linux kernel via system call hooks.

The framework is managed by the user-space daemon **`auditd`**. It provides tamper-evident logging required by regulatory compliance standards (PCI DSS Section 10, SOC 2, HIPAA, and ISO 27001).

```
+-------------------------------------------------------------------------+
|                        auditd Kernel Architecture                       |
+-------------------------------------------------------------------------+
|  User Space                                                             |
|  +--------------------+   +--------------------+   +------------------+ |
|  | auditctl (Rules)   |   | ausearch (Query)   |   | aureport (Stats) | |
|  +---------+----------+   +---------^----------+   +--------^---------+ |
|            |                        |                       |           |
|            | Configure              | Parse Audit Logs      | Summary   |
|            v                        |                       |           |
|  +----------------------------------+-----------------------+---------+ |
|  |                           auditd Daemon                            | |
|  |                   Writes to /var/log/audit/audit.log               | |
|  +----------------------------------^---------------------------------+ |
|                                     | Netlink Socket                    |
|  Kernel Space                       |                                   |
|  +----------------------------------+---------------------------------+ |
|  |                       Audit Filter / Classifier                    | |
|  |                    (user, task, fstype, exit hooks)                | |
|  +----------------------------------^---------------------------------+ |
|                                     | Syscall Intercept                 |
|  [ Process Execution / open() / unlink() / execve() / ptrace() ]        |
+-------------------------------------------------------------------------+
```

---

## 2. Managing the auditd Service

### Installation
- **RHEL / Rocky / Alma / Fedora**: Pre-installed as standard.
- **Ubuntu / Debian**:
  ```bash
  sudo apt install -y auditd audispd-plugins
  ```

### Service Control
```bash
sudo systemctl enable --now auditd
sudo systemctl status auditd
```

> [!NOTE]
> Systemd is generally prevented from restarting `auditd` directly via `systemctl restart auditd` to prevent denial-of-service against audit trails. Use `service auditd restart` or `auditctl` to reload rules.

---

## 3. Configuring Audit Rules with `auditctl`

Audit rules fall into three categories:
1. **Control rules**: Adjust audit buffer sizes, failure flags, and rate limits.
2. **File System Watches**: Monitor access, writes, or permission alterations on specific sensitive files and directories.
3. **System Call Rules**: Audit every time a specific kernel syscall (e.g., `execve`, `ptrace`, `settimeofday`) is triggered.

### Syntax for File Watches:
```bash
auditctl -w <path_to_file> -p <permissions> -k <custom_key>
```
Permissions filter:
- `r`: Read
- `w`: Write
- `x`: Execute
- `a`: Attribute change (ownership, chmod, timestamp)

### Production Audit Rules Examples:
```bash
# 1. Watch /etc/passwd and /etc/shadow for unauthorized modifications
sudo auditctl -w /etc/shadow -p wa -k identity_tampering
sudo auditctl -w /etc/passwd -p wa -k identity_tampering

# 2. Watch SSH daemon configuration
sudo auditctl -w /etc/ssh/sshd_config -p wa -k sshd_config_change

# 3. Audit all commands executed as root (sys_execve)
sudo auditctl -a always,exit -F arch=b64 -S execve -F euid=0 -k root_command_execution
```

---

## 4. Making Audit Rules Persistent

Dynamic rules added with `auditctl` are volatile and lost upon reboot. To make them permanent, define them in `/etc/audit/rules.d/audit.rules`:

```bash
# Append rules to persistent file
cat << 'EOF' | sudo tee -a /etc/audit/rules.d/audit.rules

# Monitor privilege escalation files
-w /etc/sudoers -p wa -k sudoers_tamper
-w /etc/sudoers.d/ -p wa -k sudoers_tamper

# Monitor system reboot and shutdown commands
-a always,exit -F arch=b64 -S reboot -S shutdown -k system_power_events

# Lock audit configuration (prevents any further modifications until next reboot)
-e 2
EOF

# Load rules from rules.d
sudo augenrules --load
```

> [!WARNING]
> The `-e 2` directive permanently locks the audit configuration. Once loaded, no one (not even the `root` superuser) can disable audit logging, delete rules, or alter buffers without rebooting the server.

---

## 5. Searching and Reporting: ausearch and aureport

The raw audit log file (`/var/log/audit/audit.log`) contains low-level hexadecimal data. Specialized utilities make querying easy:

### Querying Logs with `ausearch`
```bash
# Search for events matching our custom key "identity_tampering"
sudo ausearch -k identity_tampering --interpret
```

```
type=PROCTITLE msg=audit(10/14/2026 04:45:12.890:421) : proctitle=vi /etc/shadow
type=PATH msg=audit(10/14/2026 04:45:12.890:421) : item=0 name=/etc/shadow inode=1052678 dev=fd:00 mode=file,000 ouid=root ogid=root rdev=00:00 nametype=NORMAL
type=CWD msg=audit(10/14/2026 04:45:12.890:421) : cwd=/home/admin
type=SYSCALL msg=audit(10/14/2026 04:45:12.890:421) : arch=x86_64 syscall=openat success=yes exit=3 a0=0xffffff9c a1=0x7ffe42 a2=0x20000 a3=0x0 items=1 ppid=1420 pid=2985 auid=1000 uid=0 gid=0 euid=0 comm=vi exe=/usr/bin/vi key=identity_tampering
```

The `--interpret` flag converts numeric IDs and hex codes into readable usernames and system commands.

### Generating Summary Metrics with `aureport`
```bash
# Generate high-level security executive summary
sudo aureport --summary
```

```
Summary Report
======================
Range of time in logs: 10/10/2026 00:00:01 - 10/14/2026 04:50:00
Selected time for report: 10/10/2026 00:00:01 - 10/14/2026 04:50:00
Number of changes in configuration: 12
Number of changes to accounts, groups, or roles: 4
Number of logins: 84
Number of failed logins: 312
Number of authentications: 142
Number of failed authentications: 28
Number of users: 6
Number of terminals: 14
Number of host names: 4
Number of executables: 89
Number of commands: 52
Number of files: 34
Number of AVC's: 0
Number of MAC events: 0
Number of failed syscalls: 140
Number of anomaly events: 0
Number of responses to anomaly events: 0
Number of crypto events: 512
```

```bash
# Review all failed user login attempts
sudo aureport -l --failed
```

---

## 6. Summary Takeaways

- `auditd` operates within the Linux kernel to provide tamper-proof, legally admissible audit records.
- Track critical files using file watches (`auditctl -w <path> -p wa -k <key>`).
- Persist configuration in `/etc/audit/rules.d/audit.rules` and load via `augenrules --load`.
- Use `ausearch -k <key> --interpret` to analyze events.
- Generate security compliance summaries using `aureport`.
