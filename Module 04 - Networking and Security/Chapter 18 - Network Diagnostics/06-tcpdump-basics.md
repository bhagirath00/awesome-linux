# Packet Sniffing and Traffic Analysis: `tcpdump` Basics for SREs

## Overview

**`tcpdump`** is the premier command-line packet analyzer for Linux, powered by the **`libpcap`** packet capture library. It intercepts and displays raw network packets as they arrive at, pass through, or leave a network interface.

When higher-level application logs, `curl`, and metrics fail to reveal why an API is timing out or dropping connections, `tcpdump` provides the **ground truth of the network wire**:
- Capturing DNS queries to verify whether CoreDNS in Kubernetes is responding.
- Diagnosing why a microservice is receiving sudden TCP RST (Reset) packets.
- Verifying whether a cloud firewall or NAT gateway is dropping outgoing SYN packets.
- Exporting raw traffic captures to `.pcap` files for deep graphical analysis in Wireshark.

---

## 1. Command Syntax and Essential Flags

```bash
sudo tcpdump [OPTIONS] [BPF_FILTER_EXPRESSION]
```

*Note: Capturing raw network packets requires `root` privileges (or `CAP_NET_RAW` / `CAP_NET_ADMIN` capabilities).*

### Essential Flags Reference

| Flag | Long Option | Description | DevOps Practical Context |
| :--- | :--- | :--- | :--- |
| **`-i IFACE`** | `--interface` | Capture on specified interface (e.g. `eth0`). Use **`any`** for all interfaces. | Monitoring container veth or bridge interfaces. |
| **`-nn`** | *(none)* | **Do not resolve IPs to hostnames and do not resolve ports to names**. | **Mandatory performance flag**. Avoids generating reverse DNS lookups for every captured packet! |
| **`-c COUNT`** | *(none)* | Exits automatically after capturing `COUNT` packets. | Capturing small, targeted traces without filling disk. |
| **`-A`** | *(none)* | Prints packet payload data in **ASCII text**. | **Inspecting plaintext HTTP headers, JSON payloads, and Redis commands.** |
| **`-X`** | *(none)* | Prints packet payload in both **Hex and ASCII**. | Deep protocol reverse-engineering. |
| **`-s SIZE`** | `--snapshot-length`| Number of bytes to capture per packet. Use `-s 0` for entire packet. | Ensuring full payload capture. |
| **`-w FILE.pcap`**| *(none)* | **Writes raw packets to a binary PCAP file**. | Exporting captures for team review in Wireshark. |
| **`-r FILE.pcap`**| *(none)* | Reads and analyzes packets from a saved PCAP file. | Post-mortem incident analysis. |

---

## 2. Berkeley Packet Filter (BPF) Syntax

`tcpdump` allows you to filter traffic directly inside the Linux kernel using BPF expressions, ensuring the system only captures relevant packets without overloading CPU or memory:

### Primary Primitives:
- **Protocol**: `tcp`, `udp`, `icmp`, `arp`, `ip`, `ip6`.
- **Direction**: `src` (source), `dst` (destination).
- **Type**: `host`, `net` (CIDR subnet), `port`, `portrange`.

### Combining Expressions with Boolean Operators:
- `and` (or `&&`)
- `or` (or `||`)
- `not` (or `!`)

```bash
# Capture TCP traffic on port 443 originating from a specific IP:
sudo tcpdump -i eth0 -nn src host 192.168.1.100 and tcp port 443

# Capture all traffic EXCEPT SSH (Port 22) to keep terminal screen clean:
sudo tcpdump -i eth0 -nn not port 22
```

---

## 3. High-Frequency DevOps Diagnostic Recipes

### A. Debugging DNS Failures in Real Time (Port 53)
When applications intermittently fail to resolve domain names:

```bash
sudo tcpdump -i any -nn port 53
```
Output:
```text
10:15:02.120 IP 10.0.1.50.48291 > 1.1.1.1.53: 4210+ A? api.stripe.com. (32)
10:15:02.132 IP 1.1.1.1.53 > 10.0.1.50.48291: 4210 2/0/0 A 140.82.121.4 (64)
```
*(Confirms that the query left the server and the DNS server answered within 12ms!)*.

### B. Inspecting Plaintext HTTP / API Payloads (`-A`)
Inspect incoming HTTP request headers and bodies on port 80:

```bash
sudo tcpdump -i eth0 -nn -A 'tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12:2]&0xf0)>>2)) != 0)'
```
Output reveals raw HTTP headers:
```text
GET /api/v1/health HTTP/1.1
Host: api.example.internal
User-Agent: Prometheus/2.50.0
Accept: */*
```

### C. Hunting TCP Reset (RST) Packets
When a connection is abruptly severed, find who sent the TCP Reset:

```bash
sudo tcpdump -i eth0 -nn 'tcp[tcpflags] & (tcp-rst) != 0'
```

### D. Capturing a Wireshark Trace File (`.pcap`)
Capture 500 packets of database traffic for offline team analysis:

```bash
# Capture 500 packets on PostgreSQL port 5432 and save to trace.pcap:
sudo tcpdump -i eth0 -nn -s 0 -c 500 port 5432 -w /tmp/postgres_trace.pcap
```
You can now download `postgres_trace.pcap` to your local workstation and open it in Wireshark for deep graphical analysis.

---

## Summary

- `tcpdump` captures raw packets directly from the kernel network subsystem.
- Always use the `-nn` flag to disable reverse DNS and service name lookups for maximum capture performance.
- Use Berkeley Packet Filter (BPF) expressions (`port 53`, `host 10.0.1.5`) to isolate target traffic.
- Inspect ASCII text payloads using `-A`.
- Save captures to `.pcap` files using `-w <file.pcap>` for graphical inspection in Wireshark.
