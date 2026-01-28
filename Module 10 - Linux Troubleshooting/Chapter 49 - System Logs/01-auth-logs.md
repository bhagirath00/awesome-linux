# Authentication & Security Logs: /var/log/auth.log and /var/log/secure

## 1. Architectural Concept & Deep Dive

Authentication and authorization logging is the foundational layer of Linux host intrusion detection, forensic auditing, and compliance (SOC2, PCI-DSS, ISO 27001). Every time an entity attempts to access the system—whether via an interactive SSH terminal, a `sudo` command invocation, a cron job, a PKI certificate, or a PAM module—the Linux PAM (Pluggable Authentication Modules) subsystem logs an immutable audit event.

Depending on the Linux distribution family, authentication logs reside at:
- **Debian / Ubuntu**: `/var/log/auth.log`
- **RHEL / CentOS / Rocky / Fedora / Amazon Linux**: `/var/log/secure`
- **Systemd Journal**: `journalctl -u ssh` or `journalctl _COMM=sshd _COMM=sudo`

```
+─────────────────────────────────────────────────────────────────────────+
|                        AUTHENTICATION LOG FLOW                          |
|                                                                         |
|   Inbound Access Vector (SSH / Console / Sudo / SU / Cron)              |
|                               │                                         |
|                               ▼                                         |
|                 PAM (Pluggable Authentication Modules)                  |
|                 • pam_unix.so (Check /etc/shadow)                       |
|                 • pam_sssd.so / pam_ldap.so (Active Directory)          |
|                 • pam_faillock.so (Account Lockout)                     |
|                               │                                         |
|                               ▼                                         |
|                       syslog(LOG_AUTHPRIV)                              |
|                               │                                         |
|            ┌──────────────────┴──────────────────┐                      |
|            ▼                                     ▼                      |
|    Debian/Ubuntu                          RHEL/Rocky/Fedora             |
|   /var/log/auth.log                       /var/log/secure               |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Anatomy of Authentication Log Entries

### 1. Successful SSH Key Login
```
Oct 14 07:10:01 node1 sshd[24102]: Accepted publickey for deployer from 198.51.100.25 port 54210 ssh2: RSA SHA256:7uK...
Oct 14 07:10:01 node1 sshd[24102]: pam_unix(sshd:session): session opened for user deployer(uid=1000) by (uid=0)
```
- **Fields**: Timestamp, Hostname, Daemon PID, Auth mechanism (`publickey`), User, Remote IP, Source Port, TLS/Key Fingerprint, Session State.

### 2. Failed SSH Password Brute-Force Attempt
```
Oct 14 07:12:15 node1 sshd[24155]: Failed password for invalid user admin from 203.0.113.88 port 41202 ssh2
Oct 14 07:12:18 node1 sshd[24155]: Connection closed by authenticating user admin 203.0.113.88 port 41202 [preauth]
```
- **Security Indicator**: `invalid user` indicates an attacker is scanning user dictionaries.

### 3. Sudo Command Execution & Privilege Escalation
```
Oct 14 07:15:30 node1 sudo: pam_unix(sudo:session): session opened for user root(uid=0) by deployer(uid=1000)
Oct 14 07:15:30 node1 sudo: deployer : TTY=pts/0 ; PWD=/home/deployer ; USER=root ; COMMAND=/bin/systemctl restart nginx
```
- **Audit Value**: Captures the exact calling user (`deployer`), their UID, the target user (`root`), TTY, current working directory (`PWD`), and the exact executed command.

---

## 3. Hands-on Security Triaging & Forensic CLI Commands

### 1. Identify Failed Logins & Attacker IPs
```bash
# Detect top 10 IP addresses failing SSH authentication
grep "Failed password" /var/log/auth.log* | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn | head -n 10
```
```
 14502 203.0.113.88
  8210 198.51.100.4
   412 192.0.2.14
```

### 2. List All Users Targeted in Dictionary Attacks
```bash
grep "Invalid user" /var/log/auth.log* | awk '{print $(NF-5)}' | sort | uniq -c | sort -rn | head -n 5
```
```
 18451 admin
 12104 root
  4520 test
  1200 oracle
   850 ubuntu
```

### 3. Track All Sudo Invocations in the Last 24 Hours
```bash
grep -E "sudo:.*COMMAND=" /var/log/auth.log | awk -F: '{print $1":"$2":"$3, $5, $6}'
```

---

## 4. Built-in Forensic Utilities: `last`, `lastb`, `faillog`

Linux maintains binary log files for user session tracking that should not be edited manually:
- `/var/log/wtmp`: Tracks all successful logins, logouts, and system reboots.
- `/var/log/btmp`: Tracks all bad/failed login attempts.
- `/var/log/lastlog`: Tracks the most recent login for every system user.

```bash
# 1. View recent successful logins and reboot history
last -n 10

# 2. View failed login attempts recorded in /var/log/btmp
sudo lastb -n 10

# 3. View user account failure counts and lockouts
sudo faillog -a
```

---

## 5. Automated Intrusion Prevention: Fail2ban Integration

In production environments, manual log checking is supplemented by **Fail2ban**. Fail2ban continuously tails `/var/log/auth.log` or the systemd journal via regex filters, automatically injecting dynamic `nftables` or `iptables` drop rules when IP threshold violations occur.

```
Attacker ──► Multiple Failed Logins ──► /var/log/auth.log
                                                │
                                                ▼
                                         [ fail2ban ]
                                                │ (Regex Match > 5 attempts)
                                                ▼
                                 nftables / iptables REJECT Rule
```

### Checking Active Fail2ban Jail Status
```bash
sudo fail2ban-client status sshd
```
```
Status for the jail: sshd
|- Filter
|  |- Currently failed: 12
|  |- Total failed:     4501
|  `- File list:        /var/log/auth.log
`- Actions
   |- Currently banned: 3
   |- Total banned:     142
   `- Banned IP list:   203.0.113.88 198.51.100.4 192.0.2.14
```

---

## 6. Verification & Troubleshooting Runbook

### Security Hardening Checklist
```bash
# Ensure auth logs are strictly readable only by root/adm
ls -l /var/log/auth.log /var/log/secure
# Should be: -rw-r----- 1 root adm (or root root)

# Verify PAM account lockout module is active
grep pam_faillock /etc/pam.d/password-auth /etc/pam.d/system-auth
```

### Common Pitfalls & Solutions
1. **`/var/log/auth.log` is missing on Debian/Ubuntu**:
   - Cause: `rsyslog` service is stopped or uninstalled.
   - Solution: Run `sudo apt-get install -y rsyslog && sudo systemctl restart rsyslog`. Alternatively, query systemd journal directly via `journalctl -u ssh`.
2. **Attacker flooded auth.log causing disk exhaustion**:
   - Cause: Unrestricted SSH port (22) exposed to the public internet undergoing automated dictionary attacks.
   - Solution: Rotate and compress logs immediately (`sudo logrotate -f /etc/logrotate.d/rsyslog`). Disable password authentication (`PasswordAuthentication no` in `/etc/ssh/sshd_config`), switch to SSH keys, and enforce IP whitelisting or VPN access.
3. **`last` command shows "wtmp begins..." with very little history**:
   - Cause: `/var/log/wtmp` was rotated by `logrotate` or truncated.
   - Solution: Query rotated wtmp files using `last -f /var/log/wtmp.1`.

---

## 7. Key Takeaways
- Authentication logs (`/var/log/auth.log` on Debian/Ubuntu, `/var/log/secure` on RHEL) track all PAM events: SSH logins, sudo commands, and su switches.
- Binary logs (`/var/log/wtmp` and `/var/log/btmp`) are inspected using `last` and `lastb`.
- Regular triage of failed login patterns exposes unauthorized scanning and brute-force attempts.
- Automated tools like Fail2ban pair with auth logs to dynamically ban hostile actors at the network firewall layer.
