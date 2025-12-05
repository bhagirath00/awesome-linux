# Dynamic Host Configuration Protocol (DHCP) in Linux: Clients, Leases, and Netplan

## Overview

The Dynamic Host Configuration Protocol (DHCP) automates the assignment of network parameters to client machines. When a Linux server or container connects to a network, a DHCP client communicates with an upstream DHCP server (such as a router, AWS VPC DHCP option set, or dnsmasq daemon) to dynamically receive:
- An IPv4/IPv6 Address with CIDR Subnet Mask
- Default Gateway Router IP
- Upstream DNS Nameservers and Search Domains
- Network Time Protocol (NTP) Servers
- Static Host Route Options (Option 121)

---

## 1. The DORA Process: How DHCP Works

DHCP operates over **UDP ports 67 (Server)** and **68 (Client)** using a 4-stage handshake:

```text
Client (Linux Server)                                      DHCP Server / Router
       │                                                             │
       ├────── [1] DHCPDISCOVER (Broadcast 255.255.255.255) ────────►│ "Any DHCP server out there?"
       │                                                             │
       │◄───── [2] DHCPOFFER (Unicast / Broadcast) ──────────────────┤ "You can have IP 10.0.1.50!"
       │                                                             │
       ├────── [3] DHCPREQUEST (Broadcast) ─────────────────────────►│ "I accept IP 10.0.1.50!"
       │                                                             │
       │◄───── [4] DHCPACK (Acknowledgment) ─────────────────────────┤ "Lease confirmed for 24 hours!"
       │                                                             │
```

---

## 2. Linux DHCP Client Implementations

Depending on your distribution and initialization system, Linux uses different DHCP client daemons:

| Client Engine | Environment | Characteristics |
| :--- | :--- | :--- |
| **`systemd-networkd`** | Modern Ubuntu Server, CoreOS, Arch | Built directly into systemd; zero external dependencies. |
| **`NetworkManager`** | RHEL, Rocky Linux, Fedora, Desktop | Integrated internal DHCP client (`dhcpcd` or `dhclient`). |
| **`dhclient`** | Legacy Debian, Ubuntu, RHEL 7 | Classic ISC DHCP client daemon; standalone binary. |
| **`udhcpc`** | **Alpine Linux, BusyBox, Minimal Docker** | Ultra-lightweight (under 50KB); standard in container images. |

---

## 3. Configuring DHCP via Netplan (Ubuntu Server)

To configure an interface to obtain network parameters automatically via DHCP in Netplan:

```yaml
# /etc/netplan/01-dhcp.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: true
      dhcp6: false
```

### Overriding Specific DHCP Options
In cloud environments, you frequently want the DHCP server to assign the IP address and Gateway, but you want to enforce **custom DNS resolvers** instead of using the local ISP or cloud provider's DNS:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: true
      dhcp4-overrides:
        use-dns: false            # Ignores DNS servers offered by DHCP!
        use-domains: false        # Ignores search domains offered by DHCP
      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8
```

---

## 4. Releasing and Renewing Leases via CLI

When troubleshooting dynamic IP assignments, you can force the client to release its current lease and request a fresh one:

### Using Traditional `dhclient`
```bash
# Release current IP lease:
sudo dhclient -r eth0

# Request a fresh lease with verbose output:
$ sudo dhclient -v eth0
Listening on LPF/eth0/08:00:27:8a:42:19
Sending on   LPF/eth0/08:00:27:8a:42:19
DHCPDISCOVER on eth0 to 255.255.255.255 port 67 interval 3
DHCPOFFER of 192.168.1.50 from 192.168.1.1
DHCPREQUEST for 192.168.1.50 on eth0 to 255.255.255.255 port 67
DHCPACK of 192.168.1.50 from 192.168.1.1
bound to 192.168.1.50 -- renewal in 41200 seconds.
```

### Inspecting Active Lease Information
The active lease file records the lease expiration timestamp and server identifiers:
```bash
# On traditional systems:
cat /var/lib/dhcp/dhclient.leases

# On systemd-networkd systems:
cat /run/systemd/netif/leases/*
```

---

## Summary

- DHCP dynamically provisions IP addresses, netmasks, gateways, and DNS via the 4-step DORA handshake (UDP 67/68).
- Configure DHCP in Netplan using `dhcp4: true`.
- Override DHCP-provided DNS nameservers while retaining dynamic IP assignment using `dhcp4-overrides: { use-dns: false }`.
- Release and renew leases manually with `dhclient -r` and `dhclient -v`.
- Alpine Linux and minimal container images use `udhcpc`.
