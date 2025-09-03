# Troubleshooting Linux Network Failures: Layer 1 to Layer 7

## 1. Architectural Concept & Deep Dive

Network degradation in Linux rarely announces itself simply as "offline." Instead, it appears as intermittent latency spikes, TCP connection drops, DNS resolution timeouts, or silent packet loss.

A structured OSI/TCP model troubleshooting methodology prevents wasting time inspecting application logs when the failure is caused by an MTU mismatch or an iptables state table exhaustion:

```
+─────────────────────────────────────────────────────────────────────────+
|                 LAYERED NETWORK DIAGNOSTIC HIERARCHY                    |
|                                                                         |
|  Layer 7 (Application):   HTTP status, TLS certs      curl -Iv, openssl |
|         ▲                                                               |
|  Layer 5/6 (DNS/Session): Name resolution, resolve.conf dig, nslookup   |
|         ▲                                                               |
|  Layer 4 (Transport):     TCP states, syn-backlog     ss -tulpn, nc -zv |
|         ▲                                                               |
|  Layer 3 (Network):       Routing, Netfilter, MTU     ip route, iptables|
|         ▲                                                               |
|  Layer 2 (Data Link):     MAC addresses, ARP cache    ip neigh, arp     |
|         ▲                                                               |
|  Layer 1 (Physical/Host): NIC link, carrier, drops    ip link, ethtool  |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Step-by-Step Triage Runbook (L1 to L7)

### Step 1: Physical & Interface Layer (L1/L2)
Verify if the network interface is physically UP and negotiating speed properly:

```bash
# Check link status and carrier signal
ip link show dev eth0
```
- Verify flags contain `UP` and `LOWER_UP`. If `NO-CARRIER`, the cable or virtual NIC is disconnected.

```bash
# Inspect duplex, link speed, and auto-negotiation
sudo ethtool eth0 | grep -E "(Speed|Duplex|Link detected)"
```
```
Speed: 10000Mb/s
Duplex: Full
Link detected: yes
```

### Step 2: Layer 3 - IP Addressing & Routing
Verify valid IP assignment and default gateway routing:

```bash
# 1. Verify IP address assignment
ip -4 addr show dev eth0

# 2. Verify Default Gateway exists
ip route show
```
```
default via 192.168.1.1 dev eth0 proto dhcp src 192.168.1.50 metric 100
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.50
```
- If `default via` is missing, outbound internet routing will immediately fail with `Network is unreachable`.

### Step 3: Layer 3/4 - Ping & Path Latency (`mtr`)
Test connectivity to the gateway and remote hosts:

```bash
# Ping default gateway
ping -c 3 192.168.1.1

# Trace route with live packet loss analysis per hop
mtr --report -c 10 1.1.1.1
```

---

## 3. Investigating Packet Drops & Buffer Exhaustion

When networks experience packet drops under load, inspect kernel ring buffers and socket queues:

### 1. Interface-Level Drops (`ip -s link`)
```bash
ip -s link show dev eth0
```
```
RX: bytes  packets  errors  dropped overrun mcast
    4510291 1420194       0     4501       0     0
```
- **`dropped` / `overrun`**: The kernel network stack could not process incoming packets fast enough, filling the NIC DMA ring buffer.
- **Fix**: Increase ring buffer size:
  ```bash
  sudo ethtool -G eth0 rx 4096 tx 4096
  ```

### 2. Kernel TCP Stack Drops (`netstat -s`)
```bash
netstat -s | grep -E "(listen queue|overflowed|retransmitted)"
```
```
    4512 times the listen queue of a socket overflowed
    12045 packets retransmitted
```
- **Listen queue overflowed**: Application socket backlog is too small.
- **Fix**: Tune kernel listen backlog and application `somaxconn`:
  ```bash
  sudo sysctl -w net.core.somaxconn=4096
  sudo sysctl -w net.ipv4.tcp_max_syn_backlog=4096
  ```

---

## 4. Connection Exhaustion: `TIME_WAIT` & Ephemeral Ports

High-throughput microservices initiating thousands of short-lived HTTP connections can exhaust local ephemeral ports:

```bash
# Inspect TCP socket states
ss -s
```
```
Total: 45012
TCP:   42100 (estab 2100, closed 38000, orphaned 0, timewait 38000)
```
- **Analysis**: 38,000 sockets stuck in `TIME_WAIT`. The host cannot allocate new source ports (`EADDRNOTAVAIL`).
- **Remediation**:
  1. Expand local port range:
     ```bash
     sudo sysctl -w net.ipv4.ip_local_port_range="1024 65535"
     ```
  2. Enable socket reuse for outgoing connections:
     ```bash
     sudo sysctl -w net.ipv4.tcp_tw_reuse=1
     ```

---

## 5. DNS Resolution Outages

DNS is the most common cause of "intermittent network timeouts."

### 1. Test Resolution with Exact Latency (`dig`)
```bash
dig api.github.com +stats
```
Look at the footer:
```
;; Query time: 2 msec
;; SERVER: 127.0.0.53#53(127.0.0.53)
;; WHEN: Wed Oct 14 07:55:10 UTC 2026
```
- If `Query time` is > 2000 ms, the configured upstream nameserver is timing out.

### 2. Inspect Resolver Configuration
```bash
cat /etc/resolv.conf
```
- If using `systemd-resolved` (default on Ubuntu):
  ```bash
  resolvectl status
  ```
  Verify active DNS servers and search domains. Flush cache if poisoned:
  ```bash
  sudo resolvectl flush-caches
  ```

---

## 6. Packet Capture Deep Dive (`tcpdump`)

When debugging elusive connection drops, SSL handshakes, or HTTP 502s, capture raw packets at the interface:

```bash
# Capture TCP SYN packets to verify if remote host is responding
sudo tcpdump -ni eth0 "tcp[tcpflags] & (tcp-syn) != 0 and port 443" -c 10
```

```bash
# Capture full HTTP/TLS payload to file for Wireshark inspection
sudo tcpdump -i eth0 host 10.0.1.25 and port 80 -w /tmp/capture.pcap
```

---

## 7. MTU Mismatch & Black Hole Routers

If small packets (`ping`) succeed, but large transfers (SSH banner exchange, `git clone`, HTTP POST) hang indefinitely:
- Cause: **Path MTU Black Hole**. The network path requires an MTU smaller than standard 1500 bytes (common in VXLAN, IPSec, or AWS Jumbo frames), and an intermediary router silently drops packets without sending ICMP `Fragmentation Needed` packets.

### Testing MTU Path
```bash
# Ping with Don't Fragment (DF) flag set and sweep payload size
ping -M do -s 1472 -c 2 1.1.1.1
```
*(1472 payload + 28 bytes IP/ICMP header = 1500 bytes MTU).*
- If 1472 fails with `Frag needed and DF set`, lower the interface MTU:
  ```bash
  sudo ip link set dev eth0 mtu 1400
  ```

---

## 8. Key Takeaways
- Always diagnose network failures along the OSI stack: Interface -> Route -> Firewall -> TCP -> DNS -> Application.
- High `TIME_WAIT` counts exhaust client ports; tune `net.ipv4.tcp_tw_reuse=1`.
- Packet drops in `ip -s link` point to small ring buffers; increase via `ethtool -G`.
- When small pings pass but large data streams freeze, diagnose MTU mismatches using `ping -M do`.
