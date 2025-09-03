# Chapter 43: Network Performance - TCP Congestion Control and Keepalive Tuning

## 1. Introduction: TCP Congestion Control Algorithms

TCP is a reliable, stream-oriented transport protocol. Because network conditions fluctuate wildly, TCP employs **Congestion Control Algorithms** to determine how fast it can transmit data packets into the network without overwhelming intermediate routers and causing packet drops.

Historically, the default congestion algorithm across all Linux distributions was **CUBIC**. CUBIC uses packet loss as an indicator of congestion: when a packet drops, CUBIC drastically cuts its transmission rate (window size) in half and slowly ramps back up.

---

## 2. The Modern Standard: Google BBR (Bottleneck Bandwidth and RTT)

Developed by Google in 2016 and merged into the mainline Linux kernel, **BBR** (**Bottleneck Bandwidth and RTT**) revolutionized network transport.

Instead of waiting for packets to drop before reacting, BBR builds an explicit real-time mathematical model of the network path by continuously measuring:
1. **Max Bandwidth**: The maximum transmission speed of the network bottleneck.
2. **Min RTT**: The minimum round-trip time (latency) of the connection.

```
+--------------------------------------------------------------------------+
|                         CUBIC vs. BBR Throughput                         |
+--------------------------------------------------------------------------+
|  CUBIC (Loss-Based):                                                     |
|  Throughput:  /\  /\  /\ (Sawtooth pattern: Cuts speed 50% on drop!)     |
|              /  \/  \/  \                                                |
|                                                                          |
|  BBR (Model-Based - Google):                                             |
|  Throughput:  ======================== (Smooth, maximum line saturation!)|
|  - Up to 2x-10x faster throughput on lossy Wi-Fi, 4G/5G, and Trans-Ocean |
+--------------------------------------------------------------------------+
```

---

## 3. Enabling BBR Congestion Control

BBR requires the **Fair Queueing (`fq`)** queueing discipline.

### Step 1: Check Available and Active Congestion Algorithms
```bash
# Available algorithms in current kernel
sysctl net.ipv4.tcp_available_congestion_control
```
```
net.ipv4.tcp_available_congestion_control = reno bbr cubic
```

```bash
# Check currently active algorithm
sysctl net.ipv4.tcp_congestion_control
```
```
net.ipv4.tcp_congestion_control = cubic
```

### Step 2: Enable BBR Permanently
Add to `/etc/sysctl.d/99-bbr.conf`:

```ini
# Enforce Fair Queueing scheduler (prerequisite for BBR pacing)
net.core.default_qdisc = fq

# Switch congestion algorithm to BBR
net.ipv4.tcp_congestion_control = bbr
```

Apply immediately:
```bash
sudo sysctl --system
```

Verify activation:
```bash
sysctl net.ipv4.tcp_congestion_control
```
```
net.ipv4.tcp_congestion_control = bbr
```
Your server will immediately achieve significantly higher throughput and lower latency over public internet and cloud connections!

---

## 4. Tuning TCP Keepalive Parameters

When an application establishes a long-lived TCP connection (e.g., a database connection pool, a WebSocket stream, or an SSH tunnel), network firewalls and NAT gateways drop idle connections from their state tables after a period of silence (often 5 to 15 minutes). 

When the server finally tries to send data on the dead socket, it hangs waiting for a timeout.

**TCP Keepalive** sends periodic empty probe packets to keep connections alive through firewalls and detect dead peers quickly:

```ini
# Time (in seconds) the connection must remain completely idle before sending keepalive probes
# Default is 7200s (2 HOURS! Much too long for cloud firewalls)
net.ipv4.tcp_keepalive_time = 300

# Interval (in seconds) between individual keepalive probes if no reply is received
net.ipv4.tcp_keepalive_intvl = 15

# Number of unacknowledged probes sent before considering the connection dead
net.ipv4.tcp_keepalive_probes = 5
```

With these settings:
- Keepalive probes begin after **5 minutes** (300s) of inactivity.
- If a client or router crashes, the server detects the dead socket within **75 seconds** (5 probes x 15s) and cleanly frees the file descriptor and memory.

---

## 5. TCP Fast Open (TFO)

Standard TCP handshakes require 3 packets (`SYN` -> `SYN-ACK` -> `ACK`) before the client can send its HTTP payload, adding a full Round-Trip Time (RTT) of latency.

**TCP Fast Open (TFO)** enables data exchange inside the initial `SYN` packet itself!

```bash
# 3 = Enable TFO for both incoming (server) and outgoing (client) connections
sudo sysctl -w net.ipv4.tcp_fastopen=3
```

---

## 6. Summary Takeaways

- Google BBR delivers dramatically higher throughput than CUBIC over real-world networks.
- BBR requires `net.core.default_qdisc = fq` and `net.ipv4.tcp_congestion_control = bbr`.
- Default TCP keepalive of 2 hours is obsolete; tune `tcp_keepalive_time = 300` to prevent cloud firewall disconnects.
- Enable `tcp_fastopen = 3` to eliminate a full RTT latency handshake on repeated connections.
