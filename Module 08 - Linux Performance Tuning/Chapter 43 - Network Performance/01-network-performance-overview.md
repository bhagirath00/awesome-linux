# Chapter 43: Network Performance - Linux Network Stack Architecture

## 1. Introduction: The Packet Journey

Understanding Linux network performance requires tracing how packets travel between the physical Network Interface Card (NIC) hardware and user-space applications (like Nginx, HAProxy, or Envoy).

The journey of an incoming packet through the Linux kernel involves hardware interrupts, direct memory access (DMA), softirqs, socket queues, and user-space context switches.

```
+--------------------------------------------------------------------------+
|                     Linux Inbound Packet Flow (Rx)                       |
+--------------------------------------------------------------------------+
|  1. Physical Network Wire (10/40/100 Gbps Ethernet)                      |
|                                |                                         |
|                                v                                         |
|  2. Network Interface Card (NIC) & DMA Controller                        |
|     - Writes raw packet into host RAM via Direct Memory Access (DMA)     |
|     - Appends descriptor to NIC Ring Buffer (rx-ring)                    |
|     - Generates Hardware Interrupt (HardIRQ) to CPU                      |
|                                |                                         |
|                                v                                         |
|  3. HardIRQ Handler                                                      |
|     - Disables NIC interrupts temporarily (NAPI mode)                    |
|     - Schedules Software Interrupt (ksoftirqd / NET_RX_SOFTIRQ)         |
|                                |                                         |
|                                v                                         |
|  4. NAPI Polling & Linux Network Subsystem                               |
|     - Allocates Socket Buffer (sk_buff) structure                        |
|     - Passes packet through iptables / nftables / eBPF filtering         |
|     - Evaluates routing table (Is packet local or forwarded?)            |
|                                |                                         |
|                                v                                         |
|  5. Protocol Handler (TCP / UDP)                                         |
|     - Validates TCP checksum and sequence numbers                        |
|     - Appends data to Socket Receive Buffer (rmem)                       |
|                                |                                         |
|                                v                                         |
|  6. User-Space Application (Nginx / Go / Node.js)                        |
|     - Awakened via epoll() event notification                            |
|     - Reads payload via read() or recv() system call                     |
+--------------------------------------------------------------------------+
```

---

## 2. Key Packet Drop Locations

When packets are dropped under high network traffic, diagnosing **where** they are dropped isolates the exact bottleneck:

| Drop Location | Cause of Packet Drop | Diagnostic Indicator | Solution / Tuning |
| :--- | :--- | :--- | :--- |
| **NIC Ring Buffer** | Packets arrive faster than CPU can drain the hardware ring. | `rx_dropped` / `rx_missed` in `ethtool -S` | Increase ring buffer size with `ethtool -G`. |
| **Kernel Backlog Queue**| `netif_receive_skb` queue between NIC and TCP stack is full. | `dropped` counter in `/proc/net/softnet_stat` | Increase `net.core.netdev_max_backlog`. |
| **Socket Receive Buffer**| Application is too slow reading from its TCP socket. | `TCPBacklogDrop` in `nstat` | Increase `net.ipv4.tcp_rmem` & socket buffers. |
| **TCP SYN Queue** | Influx of new connection requests exceeds listen backlog. | `listen queue overflow` in `netstat -s` | Increase `net.core.somaxconn` and app backlog. |

---

## 3. Auditing Packet Drops with `ip -s link` and `ethtool`

```bash
# Check high-level interface packet drop counters:
ip -s link show eth0
```

```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 00:16:3e:4a:2b:11 brd ff:ff:ff:ff:ff:ff
    RX:  bytes packets errors dropped missed mcast   
    1420912480 9412000      0     420      0     0 
    TX:  bytes packets errors dropped carrier collsns 
     842100240 6812000      0       0       0       0 
```
If `dropped` is greater than 0 and growing, investigate hardware ring buffers!

```bash
# Query detailed hardware driver statistics via ethtool
sudo ethtool -S eth0 | grep -i drop
```

---

## 4. Hardware Queues and Receive Side Scaling (RSS)

Modern enterprise NICs (Intel 82599, Mellanox ConnectX, AWS ENA) feature multiple hardware receive and transmit queues (Multi-Queue NICs). 

**Receive Side Scaling (RSS)** hashes packet 5-tuples (Source IP, Dest IP, Source Port, Dest Port, Protocol) to distribute network processing evenly across all CPU cores, preventing single-core CPU saturation from softirqs:

```bash
# Inspect number of hardware queues on NIC:
sudo ethtool -l eth0
```
```
Channel parameters for eth0:
Pre-set maximums:
RX:		0
TX:		0
Other:		0
Combined:	8    <-- Hardware supports up to 8 parallel queues!
Current hardware settings:
Combined:	8    <-- All 8 queues active across 8 CPU cores!
```

---

## 5. Summary Takeaways

- Packets traverse from NIC DMA -> Ring Buffers -> SoftIRQ -> Protocol Stack -> Socket.
- Drops at the NIC indicate ring buffer exhaustion; drops at the socket indicate slow applications.
- Check packet drops using `ip -s link` and `ethtool -S <interface>`.
- Modern NICs use Receive Side Scaling (RSS) to distribute packet processing across CPU cores.
