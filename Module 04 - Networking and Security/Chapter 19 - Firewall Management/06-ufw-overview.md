# Chapter 19: Firewall Management - UFW Overview

## 1. Introduction: What is UFW?

`UFW` (**Uncomplicated Firewall**) is the default firewall configuration front-end for Debian and Ubuntu-based Linux systems. Developed by Canonical, UFW is designed to make firewall administration straightforward, human-readable, and error-resistant.

Under the hood, UFW is not a separate packet-filtering engine; rather, it acts as a user-friendly abstraction layer that generates and manages underlying kernel rules via `iptables` or `nftables`.

```
+-------------------------------------------------------------+
|                      UFW Architecture                       |
+-------------------------------------------------------------+
|  User Interface                                             |
|  +-------------------------------------------------------+  |
|  | ufw allow 22/tcp | ufw default deny incoming | status |  |
|  +---------------------------+---------------------------+  |
|                              |                              |
|                              v                              |
|  Configuration Backend                                      |
|  +-------------------------------------------------------+  |
|  | /etc/ufw/ufw.conf       (Enable/Disable toggle)       |  |
|  | /etc/default/ufw        (Default policies & ipv6)     |  |
|  | /etc/ufw/user.rules     (Generated custom rules)      |  |
|  | /etc/ufw/before.rules   (Low-level packet hooks/NAT)  |  |
|  +---------------------------+---------------------------+  |
|                              |                              |
|                              v                              |
|  Kernel Packet Filter                                       |
|  +-------------------------------------------------------+  |
|  | iptables / netfilter / nftables engine                |  |
|  +-------------------------------------------------------+  |
+-------------------------------------------------------------+
```

---

## 2. Core Philosophy and Default Security Stance

A secure system follows the principle of **Default Deny**:

- **Incoming Traffic**: By default, **all** unsolicited incoming connections from the outside world are dropped/denied.
- **Outgoing Traffic**: By default, **all** outgoing connections initiated by local processes (e.g., DNS queries, `curl`, package updates) are allowed.
- **Routing/Forwarding**: By default, forwarding between network interfaces is disabled (unless configured as a router or container host).

```
Incoming Traffic ----> [ DEFAULT POLICY: DENY ] ----X  (Dropped unless explicit rule exists)
Outgoing Traffic <---- [ DEFAULT POLICY: ALLOW ] <--- (Permitted for system updates, outbound API)
```

> [!WARNING]
> Before enabling UFW on a remote server (such as an AWS EC2 instance, DigitalOcean droplet, or bare metal server), **always allow SSH (`ufw allow ssh` or `ufw allow 22/tcp`) first**. Enabling UFW with default settings without an explicit SSH rule will immediately sever your remote connection and lock you out!

---

## 3. Configuration File Hierarchy

| File Path | Description |
| :--- | :--- |
| `/etc/default/ufw` | Core defaults: `DEFAULT_INPUT_POLICY="DROP"`, `DEFAULT_OUTPUT_POLICY="ACCEPT"`, `IPV6=yes`. |
| `/etc/ufw/ufw.conf` | Controls if UFW activates on system startup (`ENABLED=yes`). |
| `/etc/ufw/before.rules` | Custom rules applied *before* user-created rules (frequently used for NAT/masquerading and loopback). |
| `/etc/ufw/user.rules` | System-generated rules created whenever you run `ufw allow`, `ufw deny`, etc. |
| `/etc/ufw/after.rules` | Rules evaluated *after* user rules (e.g., final logging and packet rejection). |
| `/etc/ufw/applications.d/` | Directory containing application profiles for Nginx, OpenSSH, Apache, etc. |

---

## 4. Application Profiles

UFW supports application integration via profiles stored in `/etc/ufw/applications.d/`. When software packages like Nginx or OpenSSH are installed via `apt`, they register their ports with UFW automatically:

```bash
# List all available application profiles
sudo ufw app list
```

```
Available applications:
  Nginx Full
  Nginx HTTP
  Nginx HTTPS
  OpenSSH
```

Each profile specifies the required ports and protocols, making firewall configuration as simple as:
```bash
sudo ufw allow "Nginx Full"
```

---

## 5. Summary Takeaways

- UFW abstracts complex `iptables`/`nftables` syntax into concise, human-friendly commands.
- It operates on a strict **Default Deny Incoming / Allow Outgoing** security model.
- Application profiles streamline port management for standard web servers and services.
- Never activate UFW on a remote server without first allowing port 22/SSH!
