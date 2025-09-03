# Writing Production `iptables` Rules: Hardening, NAT, and Persistence

## Overview

Writing `iptables` rules requires precise syntax. A single misplaced rule or forgotten connection-tracking clause can instantly sever remote SSH connections, crash Docker container networking, or leave a production database exposed.

This lesson provides production-grade templates for building a hardened stateful firewall, implementing Network Address Translation (NAT) and port forwarding (as used by Docker and Kubernetes), and persisting rules across system reboots.

---

## 1. Core Rule Criteria Primitives

An `iptables` rule consists of matching criteria followed by a jump target (`-j`):

| Matching Parameter | Syntax | Example |
| :--- | :--- | :--- |
| **Protocol** | `-p [tcp\|udp\|icmp\|all]` | `-p tcp` |
| **Destination Port** | `--dport PORT` | `--dport 443` |
| **Multiple Ports** | `-m multiport --dports P1,P2` | `-m multiport --dports 80,443` |
| **Source IP / CIDR** | `-s IP[/MASK]` | `-s 192.168.1.50` *(or `-s 10.0.0.0/16`)* |
| **Incoming Interface**| `-i IFACE` | `-i eth0` *(valid in INPUT/PREROUTING/FORWARD)* |
| **Outgoing Interface**| `-o IFACE` | `-o eth0` *(valid in OUTPUT/POSTROUTING/FORWARD)* |
| **Connection State** | `-m conntrack --ctstate STATE`| `-m conntrack --ctstate ESTABLISHED,RELATED` |

---

## 2. The Complete Production Hardened Firewall Script

Here is an enterprise-grade, battle-tested baseline script that establishes a **Default DROP** stateful firewall:

```bash
#!/usr/bin/env bash
# ==============================================================================
# Script: /opt/scripts/firewall-init.sh
# Purpose: Production Stateful iptables Firewall Baseline
# ==============================================================================
set -euo pipefail

echo "Flushing existing rules..."
iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X

# Step 1: Set Default Policies to DROP (Safety First!)
# Allow local outgoing packets, but DROP incoming and forwarded by default
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Step 2: Permit all loopback (localhost) traffic (Mandatory for local IPC/systemd)
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

# Step 3: Stateful Inspection - Allow all established and related reply traffic
# (This ensures outbound HTTP/DNS requests can receive response packets)
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Step 4: Drop all invalid packets immediately
iptables -A INPUT -m conntrack --ctstate INVALID -j DROP

# Step 5: Whitelist Specific Inbound Public Services
# SSH (Port 22) - Restrict to office bastion if possible, or open:
iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW -j ACCEPT

# Web Services: HTTP (80) and HTTPS (443)
iptables -A INPUT -p tcp -m multiport --dports 80,443 -m conntrack --ctstate NEW -j ACCEPT

# Allow ICMP Ping (Echo Request) with rate-limiting to prevent Ping-of-Death
iptables -A INPUT -p icmp --icmp-type echo-request -m limit --limit 5/sec --limit-burst 10 -j ACCEPT

# Step 6: Log Dropped Packets with Rate-Limiting before final drop
iptables -A INPUT -m limit --limit 5/min -j LOG --log-prefix "IPTABLES-DROPPED: " --log-level 4

echo "Production firewall applied successfully."
```

---

## 3. Network Address Translation (NAT) & Port Forwarding

The `nat` table translates IP addresses and ports. This is the exact mechanism Docker uses to expose container ports on the host.

### Prerequisite: Enable Kernel IP Forwarding
Before Linux will route packets between interfaces, IPv4 forwarding must be enabled:
```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

### A. Destination NAT / Port Forwarding (`DNAT`)
Forward traffic arriving on host port `8080` directly to an internal container IP `172.17.0.2` on port `80`:

```bash
sudo iptables -t nat -A PREROUTING \
    -p tcp --dport 8080 \
    -j DNAT --to-destination 172.17.0.2:80
```

### B. Source NAT / Masquerading (`MASQUERADE`)
Allow containers or VMs on a private subnet (`172.17.0.0/16`) to share the host's public Internet connection on `eth0`:

```bash
sudo iptables -t nat -A POSTROUTING \
    -s 172.17.0.0/16 -o eth0 \
    -j MASQUERADE
```

---

## 4. Persisting `iptables` Rules Across Reboots

By default, **`iptables` rules are purely in-memory and are completely lost upon server reboot**.

To make rules permanent:

### On Ubuntu & Debian: `iptables-persistent`
```bash
# Install persistence helper:
sudo apt update && sudo apt install -y iptables-persistent netfilter-persistent

# Save current in-memory rules to disk (/etc/iptables/rules.v4):
sudo netfilter-persistent save

# Reload rules manually from disk:
sudo netfilter-persistent reload
```

### On RHEL & Rocky Linux: `iptables-services`
```bash
# Export active rules:
sudo iptables-save | sudo tee /etc/sysconfig/iptables

# Enable service to restore on boot:
sudo systemctl enable --now iptables
```

---

## Summary

- A production firewall uses stateful filtering via `-m conntrack --ctstate ESTABLISHED,RELATED`.
- Match multiple ports efficiently using `-m multiport --dports 80,443`.
- Docker implements port forwarding via `DNAT` in `PREROUTING` and internet sharing via `MASQUERADE` in `POSTROUTING`.
- Standard `iptables` rules disappear on reboot; save them permanently using `netfilter-persistent save` or `iptables-save`.
