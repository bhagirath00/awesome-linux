# The `ip` Command in Linux: Modern Network Management with iproute2

## Overview

The `ip` command (part of the **`iproute2`** suite) is the definitive, modern standard for inspecting and configuring network interfaces, IP addresses, routing tables, and ARP neighbor caches in the Linux kernel.

Historically, Linux relied on the `net-tools` package (`ifconfig`, `route`, `arp`, `netstat`). However, `net-tools` was officially declared **obsolete and deprecated in 2001**—it cannot handle policy routing, multi-table routing, network namespaces, WireGuard tunnels, or multiple IP addresses per interface without legacy aliases.

In modern DevOps, cloud engineering, and Kubernetes operations, the `ip` command is the universal tool used across all production systems.

---

## 1. Command Syntax and Object Hierarchy

The `ip` utility follows an intuitive object-oriented structure:

```bash
ip [OPTIONS] OBJECT [COMMAND [ARGUMENTS]]
```

### The Four Primary Objects

| Object | Shorthand | Network Layer | Primary Operational Responsibility |
| :--- | :---: | :---: | :--- |
| **`ip link`** | `l` | **Layer 2 (Data Link)** | Interface states (UP/DOWN), MAC addresses, MTU, veth devices. |
| **`ip addr`** | `a` | **Layer 3 (Network)** | IPv4 and IPv6 address assignment, CIDR subnets, secondary IPs. |
| **`ip route`**| `r` | **Layer 3 (Routing)** | Default gateway configuration, static routes, metrics. |
| **`ip neigh`**| `n` | **Layer 2/3 (ARP)** | Neighbor discovery, ARP table cache, MAC-to-IP mappings. |

---

## 2. Managing IP Addresses: `ip addr`

### A. Viewing IP Addresses (`ip a`)
```bash
# Display all assigned IPv4 and IPv6 addresses:
$ ip addr show
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP
    inet 192.168.1.50/24 brd 192.168.1.255 scope global dynamic eth0
       valid_lft 85420sec preferred_lft 85420sec
    inet6 fe80::a00:27ff:fe8a:4219/64 scope link
       valid_lft forever preferred_lft forever
```

### B. The DevOps Pro-Tip: Compact Brief View (`ip -br a`)
Default `ip a` outputs dozens of lines cluttered with IPv6 link-local addresses. Adding **`-br` (`--brief`)** formats the output as a clean, concise table:

```bash
$ ip -br a
lo               UNKNOWN        127.0.0.1/8 ::1/128
eth0             UP             192.168.1.50/24 fe80::a00:27ff:fe8a:4219/64
docker0          DOWN           172.17.0.1/16
```

### C. Adding and Deleting IP Addresses
```bash
# Assign an IP address with CIDR netmask to an interface:
sudo ip addr add 10.0.1.25/24 dev eth0

# Add a secondary (alias) IP to the same interface:
sudo ip addr add 10.0.1.26/24 dev eth0

# Remove an assigned IP address:
sudo ip addr del 10.0.1.26/24 dev eth0

# Flush all IP addresses from an interface:
sudo ip addr flush dev eth0
```

---

## 3. Managing the Routing Table: `ip route`

The routing table instructs the kernel where to forward packets based on destination subnet.

### A. Viewing the Active Routing Table
```bash
$ ip route show
default via 192.168.1.1 dev eth0 proto dhcp metric 100
10.0.1.0/24 dev eth0 proto kernel scope link src 10.0.1.25
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1
```

### B. Adding and Deleting Routes
```bash
# Add a Default Gateway:
sudo ip route add default via 192.168.1.1 dev eth0

# Add a Static Route to a remote VPC subnet via a specific gateway:
sudo ip route add 10.50.0.0/16 via 192.168.1.254 dev eth0

# Remove a static route:
sudo ip route del 10.50.0.0/16

# Verify which route the kernel will select for a specific destination IP:
$ ip route get 8.8.8.8
8.8.8.8 via 192.168.1.1 dev eth0 src 192.168.1.50 uid 1000
    cache
```
*(The `ip route get <IP>` diagnostic command tells you instantly which interface and source IP will be used to reach a target destination!)*.

---

## 4. Managing the ARP Cache: `ip neigh`

The Address Resolution Protocol (ARP) translates Layer 3 IP addresses into Layer 2 physical MAC addresses on local broadcast segments:

```bash
# View active ARP table:
$ ip neigh show
192.168.1.1 dev eth0 lladdr 00:11:22:33:44:55 REACHABLE
192.168.1.100 dev eth0 lladdr 66:77:88:99:aa:bb STALE

# Flush entire ARP cache (resolves stale MAC address bindings after router replacement):
sudo ip neigh flush all
```

---

## 5. Machine-Readable Automation: JSON Output (`-j` and `-p`)

In automated shell scripts and Python automation, parsing text tables using regex or `awk` is brittle. Modern `iproute2` natively supports **structured JSON output**:

```bash
# Get brief colorized view for humans:
ip -c -br a

# Get JSON output for scripting:
$ ip -j -p a show dev eth0
[ {
    "ifindex": 2,
    "ifname": "eth0",
    "flags": [ "BROADCAST","MULTICAST","UP","LOWER_UP" ],
    "mtu": 1500,
    "qdisc": "fq_codel",
    "operstate": "UP",
    "addr_info": [ {
        "family": "inet",
        "local": "192.168.1.50",
        "prefixlen": 24,
        "broadcast": "192.168.1.255",
        "scope": "global"
    } ]
} ]
```

### Parsing IP Addresses in CI/CD with `jq`:
```bash
# Extract primary IPv4 address cleanly in a single line without awk/sed:
PRIMARY_IP=$(ip -j a show dev eth0 | jq -r '.[0].addr_info[] | select(.family=="inet") | .local')
echo "Server IP is: $PRIMARY_IP"
```

---

## Summary

- The `ip` utility is the modern replacement for obsolete tools like `ifconfig`, `route`, and `arp`.
- Use `ip link` for Layer 2 interface management, `ip addr` for Layer 3 IPs, and `ip route` for routing.
- The most convenient daily inspection flag is `ip -br a` (brief summary).
- Test kernel routing decisions using `ip route get <destination-ip>`.
- Use `ip -j a | jq ...` to extract network parameters cleanly in automated scripts.
