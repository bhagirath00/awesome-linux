# Network Discovery and Port Auditing: `nmap` Basics for DevOps

## Overview

**`nmap`** (Network Mapper) is the industry-standard network discovery and vulnerability scanner. It probes remote target IP addresses and networks by transmitting customized packets, analyzing the responses to determine open ports, running services, application versions, and underlying operating systems.

In DevOps, Site Reliability Engineering, and cloud security, `nmap` is not just a penetration testing tool. It is a critical defense-in-depth utility used to:
- Audit cloud firewall configurations (AWS Security Groups, Azure NSGs) to ensure database ports (3306, 5432, 6379) are not accidentally exposed to the public Internet.
- Discover live IP addresses across newly provisioned VPC subnets.
- Verify that updated application daemons are responding on expected ports.

---

## 1. Understanding Port States in Nmap

When `nmap` scans a port, it classifies it into one of three primary states:

| Port State | Technical Meaning | DevOps Significance |
| :--- | :--- | :--- |
| **`open`** | A service is actively listening on this port and accepting connections. | Service is reachable (e.g., Nginx on 80/443). |
| **`closed`** | The host is reachable, but **no service is listening** (host responds with TCP RST). | Server is up, but daemon is stopped or crashed. |
| **`filtered`** | A **firewall or security group is dropping packets**; no response is received. | Firewall rules are blocking traffic cleanly. |

---

## 2. Core Scanning Techniques

```bash
nmap [SCAN_TYPE] [OPTIONS] TARGET
```

### A. TCP SYN Stealth Scan (`-sS`) — The Gold Standard (Root Only)
The default scan when executed with `sudo`. It performs a "half-open" handshake:
1. Sends TCP SYN.
2. If server responds with SYN-ACK, port is **`open`**.
3. Nmap immediately sends TCP RST to tear down the connection before the 3-way handshake completes.
4. **Advantage**: Fast, stealthy, and does not flood application logs.

```bash
sudo nmap -sS 192.168.1.50
```

### B. TCP Connect Scan (`-sT`) — Non-Root Fallback
Completes the full 3-way TCP handshake using standard POSIX `connect()` system calls. Used when running without root privileges.

### C. Ping Sweep: Discover Live Hosts (`-sn`)
Scans a subnet to discover which IP addresses are online **without scanning individual ports**:

```bash
# Discover all live hosts in a /24 cloud subnet:
nmap -sn 10.0.1.0/24
```

---

## 3. Specifying Ports

By default, `nmap` scans the top 1,000 most common ports. You can target specific ports:

| Syntax | Description | Example |
| :--- | :--- | :--- |
| **`-p PORT`** | Single port or comma-separated list. | `nmap -p 80,443,8080 server.internal` |
| **`-p START-END`** | Numerical range of ports. | `nmap -p 1-1024 server.internal` |
| **`-p-`** | **Scans all 65,535 TCP ports**. | `nmap -p- server.internal` |
| **`-F`** | Fast scan (scans top 100 ports only). | `nmap -F server.internal` |
| **`--top-ports N`**| Scans the top `N` most common ports. | `nmap --top-ports 20 server.internal` |

---

## 4. Service Version & OS Detection (`-sV` and `-O`)

Beyond verifying that a port is open, you frequently need to verify which exact software version is running:

```bash
# -sV probes open ports to determine service and version info:
$ sudo nmap -sV -p 22,80,3306 192.168.1.50
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13
80/tcp   open  http    nginx 1.24.0 (Ubuntu)
3306/tcp open  mysql   MySQL 8.0.39-0ubuntu0.24.04.1
```

---

## 5. Production DevOps Security Audit Recipe

Perform an external security perimeter scan against your public cloud IP to ensure zero sensitive ports leaked through firewall rules:

```bash
sudo nmap -sS -sV -p 22,80,443,2375,3306,5432,6379,9090,6443 <PUBLIC_IP>
```

### Pass / Fail Criteria:
- Ports `80` and `443` should be **`open`**.
- Port `22` should be `open` (restricted to bastion IP) or `filtered`.
- Database ports (`3306`, `5432`, `6379`) and Docker (`2375`) must be **`filtered`**! If any report `open`, your firewall is broken!

> [!CAUTION]
> **Legal Notice**: Never run port scans against external servers, IPs, or domains that you do not own or do not have explicit written permission to test. Unauthorized port scanning can violate ISP Terms of Service and computer security laws.

---

## Summary

- `nmap` discovers hosts, open ports, and running service versions.
- Port states indicate access: `open` (listening), `closed` (no service), and `filtered` (firewall blocking).
- Default to `sudo nmap -sS` for fast TCP half-open SYN scans.
- Discover active hosts on subnets without port scanning using `nmap -sn <subnet>/24`.
- Scan all 65,535 ports with `nmap -p-` to detect services on non-standard ports.
- Detect exact service versions using `nmap -sV`.
