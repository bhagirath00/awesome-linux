# Linux Networking Overview: Architecture, TCP/IP Stack, and OSI Model

## Overview

Linux powers the vast majority of the world's Internet routers, cloud load balancers, container networks, and hyper-scale compute clusters. Networking in Linux is not an add-on service; it is baked directly into the **Linux Kernel Networking Subsystem**.

In DevOps, Site Reliability Engineering, and cloud infrastructure, a deep understanding of Linux networking is critical for:
- Debugging Kubernetes CNI (Container Network Interface) routing and packet drops.
- Optimizing low-latency cloud ingress and egress across AWS VPCs or bare-metal clusters.
- Understanding how container network namespaces (`netns`) isolate network stacks.
- Configuring software-defined firewalls (`iptables`, `nftables`, `eBPF`).

---

## 1. The Linux Kernel Networking Stack vs The OSI Model

The Linux kernel implements the **TCP/IP Protocol Suite** (RFC 1122), which maps directly across the classic 7-layer OSI conceptual framework:

```text
OSI 7-Layer Model                     Linux Kernel Architecture
+-------------------------------+     +-----------------------------------------+
| Layer 7: Application          |     | User Space Applications                 |
| (HTTP, DNS, SSH, gRPC)        |     | (Nginx, Envoy, curl, sshd, Docker)      |
+-------------------------------+     +-----------------------------------------+
                                                           │
                                      POSIX Socket API (socket, bind, connect, send)
                                                           ▼
+-------------------------------+     +-----------------------------------------+
| Layer 4: Transport            |     | Kernel Transport Layer Protocols        |
| (TCP, UDP, SCTP, ICMP)        |     | (TCP Congestion Control, UDP Buffers)   |
+-------------------------------+     +-----------------------------------------+
| Layer 3: Network / Internet   |     | IP Routing Engine & Netfilter Firewall  |
| (IPv4, IPv6, ARP, Routing)    |     | (Routing Tables, iptables, nftables)    |
+-------------------------------+     +-----------------------------------------+
| Layer 2: Data Link            |     | Network Device Drivers & Bridge Layer   |
| (Ethernet, MAC, VLAN 802.1Q)  |     | (Virtual Interfaces, veth, bridge, bond)|
+-------------------------------+     +-----------------------------------------+
| Layer 1: Physical             |     | Hardware NIC / Transceiver              |
| (Cables, Fiber, Radio)        |     | (PCIe Network Card, SFP+ 100GbE)        |
+-------------------------------+     +-----------------------------------------+
```

---

## 2. Core Linux Networking Concepts

### A. Network Interfaces
A network interface is the software entity through which the Linux kernel transmits and receives packets.
- **Physical Interfaces**: Direct hardware controllers (`eth0`, `enp3s0`).
- **Loopback (`lo`)**: An internal virtual interface (`127.0.0.1`) that routes traffic entirely within memory without hitting network hardware.
- **Virtual Interfaces**: Software-defined abstractions crucial to containers and virtual machines (`veth` pairs, `docker0`, `cni0`, `tun/tap`, `br0`).

### B. Network Namespaces (`netns`)
The foundational technology behind Docker and Kubernetes pod networking. A network namespace provides an isolated instance of the network stack: its own routing table, firewall rules, socket list, and interfaces:

```text
Host Operating System (Root Netns)
  ├── eth0 (Physical IP: 192.168.1.100)
  ├── docker0 Bridge (172.17.0.1)
  │      │
  │      └── veth-host ◄──[ Virtual Ethernet Wire ]──► veth-container
  │                                                           │
  └── Container Netns (Isolated Pod Network) ─────────────────┘
        └── eth0 (Container IP: 172.17.0.2)
```

### C. The Linux Routing Table
When a packet needs to be sent, the kernel consults its internal routing table to determine which interface and gateway should handle the packet:

```bash
$ ip route show
default via 192.168.1.1 dev eth0 proto dhcp metric 100
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.50 metric 100
```

---

## 3. High-Frequency Network Ports in DevOps

Every network service binds to an IP address and an unsigned 16-bit integer called a **Port Number** (0–65535):

| Port | Protocol | Common Service | DevOps Relevance |
| :---: | :---: | :--- | :--- |
| **`22`** | TCP | **SSH** (Secure Shell) | Remote server administration, Ansible automation. |
| **`53`** | UDP/TCP | **DNS** (Domain Name System) | Name resolution, CoreDNS in Kubernetes. |
| **`80`** | TCP | **HTTP** | Unencrypted web traffic, Let's Encrypt ACME challenges. |
| **`443`** | TCP | **HTTPS** / TLS | Secure web traffic, API gateways, Kubernetes API server (`6443`). |
| **`2375 / 2376`**| TCP | **Docker Daemon** | Remote Docker engine API (2376 TLS encrypted). |
| **`6443`** | TCP | **Kubernetes API Server** | Master control plane endpoint for `kubectl`. |
| **`9090`** | TCP | **Prometheus Server** | Metrics collection dashboard and HTTP scrape target. |
| **`3000`** | TCP | **Grafana Dashboard** | Observability visualization UI. |

---

## 4. Key Configuration Files in Linux Networking

The Linux network configuration is defined across several standard locations:

- **`/etc/hosts`**: Static local hostname-to-IP lookup table. Always evaluated before DNS by default.
- **`/etc/resolv.conf`**: Upstream DNS nameserver resolvers and domain search paths.
- **`/etc/nsswitch.conf`**: Name Service Switch configuration defining lookup precedence (e.g., `hosts: files dns`).
- **`/etc/services`**: Maps canonical human-readable service names to standard port numbers.
- **`/etc/netplan/`** *(Ubuntu)* / **`/etc/NetworkManager/`** *(RHEL/Fedora)*: Declarative network configuration manifests.

---

## Summary

- The Linux networking subsystem is integrated directly into the kernel, implementing the full TCP/IP stack.
- Network namespaces (`netns`) isolate network interfaces, routing tables, and firewall rules for containers.
- Ports under 1024 are "privileged ports" requiring root or `CAP_NET_BIND_SERVICE` capability.
- The kernel consults the routing table (`ip route`) to determine packet egress paths.
- Key configuration files include `/etc/hosts`, `/etc/resolv.conf`, and `/etc/nsswitch.conf`.
