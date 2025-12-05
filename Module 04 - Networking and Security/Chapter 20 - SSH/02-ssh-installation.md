# Chapter 20: SSH - Installation and Service Management

## 1. Package Structure: Client vs. Server

The OpenSSH suite is split into two packages across modern Linux distributions:

| Package Component | Ubuntu / Debian Package | RHEL / Rocky / Alma Package | Primary Binaries |
| :--- | :--- | :--- | :--- |
| **SSH Client** | `openssh-client` | `openssh-clients` | `ssh`, `scp`, `sftp`, `ssh-keygen`, `ssh-copy-id`, `ssh-add` |
| **SSH Server Daemon** | `openssh-server` | `openssh-server` | `sshd`, `sftp-server`, `ssh-keyscan` |

---

## 2. Installing OpenSSH

### On Ubuntu / Debian:
```bash
# Update repository index
sudo apt update

# Install both client and server packages
sudo apt install -y openssh-server openssh-client
```

### On RHEL / Rocky Linux / CentOS / Fedora:
```bash
# Install OpenSSH server and client tools
sudo dnf install -y openssh-server openssh-clients
```

### On Arch Linux:
```bash
sudo pacman -S openssh
```

---

## 3. Managing the sshd Systemd Service

The OpenSSH daemon runs under systemd as `sshd.service` (or `ssh.service` on Debian/Ubuntu):

```bash
# Enable sshd to start on system boot and start it immediately
sudo systemctl enable --now sshd

# Check daemon operational status
sudo systemctl status sshd
```

```
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/lib/systemd/system/ssh.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2026-10-14 04:12:00 UTC; 5min ago
   Main PID: 1245 (sshd)
      Tasks: 1 (limit: 4642)
     Memory: 5.4M
     CGroup: /system.slice/ssh.service
             └─1245 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
```

```bash
# Reload sshd without disconnecting active user sessions
sudo systemctl reload sshd

# Restart sshd (if new ports or listeners are added)
sudo systemctl restart sshd
```

---

## 4. Key Configuration Files

```
/etc/ssh/
├── sshd_config         <-- Server configuration file (LISTENERS, AUTH, RULES)
├── sshd_config.d/      <-- Modular drop-in server configuration directory
├── ssh_config          <-- System-wide client defaults
├── ssh_config.d/       <-- Modular drop-in client configuration directory
├── ssh_host_rsa_key    <-- Server's private host key (RSA)
├── ssh_host_rsa_key.pub<-- Server's public host key (RSA)
├── ssh_host_ed25519_key<-- Server's private host key (Ed25519)
└── moduli              <-- Prime numbers for Diffie-Hellman key exchange
```

> [!IMPORTANT]
> Always validate the configuration file syntax before restarting the `sshd` daemon:
> ```bash
> sudo sshd -t
> ```
> If `sshd -t` outputs nothing, the configuration is valid. If there are syntax errors, it displays the offending line and error explanation. Restarting `sshd` with syntax errors will crash the service and prevent any future remote access!

---

## 5. Summary Takeaways

- OpenSSH is divided into client (`openssh-client` / `openssh-clients`) and daemon (`openssh-server`).
- The daemon service is controlled via `systemctl enable --now sshd`.
- Master server configuration is stored in `/etc/ssh/sshd_config`.
- Never restart `sshd` without running `sshd -t` syntax check first.
