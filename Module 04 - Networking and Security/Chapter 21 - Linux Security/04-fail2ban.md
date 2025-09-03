# Chapter 21: Linux Security - Fail2ban Intrusion Prevention

## 1. Introduction: What is Fail2ban?

`fail2ban` is an automated log-parsing intrusion prevention framework written in Python. It scans system log files (e.g., `/var/log/auth.log`, `/var/log/secure`, `/var/log/nginx/error.log`) for malicious patterns, such as multiple failed SSH login attempts or HTTP 401/403 brute-force sweeps.

When an offending IP address exceeds a configured threshold of failed attempts within a defined window of time, `fail2ban` automatically dynamically injects a packet-filtering rule into the system firewall (`iptables`, `nftables`, or `firewalld`) to block that IP address for a specified duration.

```
+--------------------------------------------------------------------------+
|                          Fail2ban Architecture                           |
+--------------------------------------------------------------------------+
|  1. Attacker                                                             |
|     Repeated SSH password guesses / scans                                |
|     198.51.100.22 -----> [ SSH Daemon: Port 22 ]                         |
|                                     |                                    |
|  2. Log Entry                       v                                    |
|     /var/log/auth.log <=== "Failed password for invalid user admin..."   |
|                                     |                                    |
|  3. Fail2ban Filter Engine          v                                    |
|     Regex filter matches failure event (failregex)                       |
|     Count = 5 failures in 10 minutes                                     |
|                                     |                                    |
|  4. Firewall Action                 v                                    |
|     Fail2ban issues firewall command:                                    |
|     iptables -I INPUT -s 198.51.100.22 -j DROP                           |
|                                                                          |
|  5. Result: Attacker packets dropped at network kernel layer!            |
+--------------------------------------------------------------------------+
```

---

## 2. Installation and Service Setup

### Ubuntu / Debian:
```bash
sudo apt update
sudo apt install -y fail2ban
```

### RHEL / Rocky Linux / AlmaLinux:
```bash
sudo dnf install -y epel-release
sudo dnf install -y fail2ban fail2ban-firewalld
```

### Starting the Daemon:
```bash
sudo systemctl enable --now fail2ban
sudo systemctl status fail2ban
```

---

## 3. Configuration Architecture: jail.conf vs. jail.local

Fail2ban organizes its rules into configurations called **Jails**.

| File | Purpose | Rule |
| :--- | :--- | :--- |
| `/etc/fail2ban/jail.conf` | Upstream package default settings. | **Never edit this file directly** (updates overwrite it). |
| `/etc/fail2ban/jail.local` | Custom user overrides and local jail activations. | **Always place custom settings here**. |
| `/etc/fail2ban/jail.d/*.conf` | Modular jail snippets for specific services. | Recommended for clean automation. |

### Creating `/etc/fail2ban/jail.local`:
```ini
[DEFAULT]
# Whitelist local loopback and corporate office IP
ignoreip = 127.0.0.1/8 ::1 192.168.1.50 203.0.113.10

# Ban duration in seconds (1 hour = 3600; 1d = 86400)
bantime = 1h

# Window of time in which failures are tracked
findtime = 10m

# Max failures permitted before banning IP
maxretry = 5

# Firewall backend (auto-detects nftables/iptables/firewalld)
banaction = ufw

# SSH Protection Jail
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = %(sshd_log)s
maxretry = 3
bantime = 24h
```

---

## 4. Administrative Commands with `fail2ban-client`

The `fail2ban-client` CLI tool is used to monitor, ban, and unban IP addresses:

```bash
# Check overall status and active jails
sudo fail2ban-client status
```

```
Status
|- Number of jail:      1
`- Jail list:           sshd
```

```bash
# Check detailed statistics for a specific jail
sudo fail2ban-client status sshd
```

```
Status for the jail: sshd
|- Filter
|  |- Currently failed: 2
|  |- Total failed:     87
|  `- File list:        /var/log/auth.log
`- Actions
   |- Currently banned: 3
   |- Total banned:     14
   `- Banned IP list:   198.51.100.22 203.0.113.88 192.0.2.14
```

### Manually Banning and Unbanning IPs
```bash
# Unban an IP accidentally blocked (e.g., an engineer who mistyped their password)
sudo fail2ban-client set sshd unbanip 203.0.113.88

# Manually ban a malicious scanner IP immediately for 48 hours
sudo fail2ban-client set sshd banip 198.51.100.99
```

---

## 5. Summary Takeaways

- Fail2ban automatically parses system logs and triggers dynamic firewall bans.
- Always configure settings in `/etc/fail2ban/jail.local` or `/etc/fail2ban/jail.d/`, never in `jail.conf`.
- Configure `ignoreip` to ensure administrators never accidentally lock themselves out.
- Inspect active bans and unban legitimate users via `fail2ban-client status <jail>` and `set <jail> unbanip <ip>`.
