# Chapter 43: Network Performance - Network Buffer and Socket Tuning (BDP)

## 1. Introduction: Bandwidth-Delay Product (BDP)

To achieve maximum network throughput across high-speed or long-distance networks, the operating system's TCP socket buffers must be sized according to the **Bandwidth-Delay Product (BDP)**:

$$\text{BDP} = \text{Network Bandwidth (bits/sec)} \times \text{Round-Trip Latency (seconds)}$$

The BDP represents the exact volume of data that can be "in flight" across the network cable at any single millisecond. If the kernel's **TCP window size** (socket buffer) is smaller than the BDP, the sender will stall, forced to stop transmitting and wait for ACKs, artificially bottlenecking a 10Gbps connection down to 50Mbps!

```
+--------------------------------------------------------------------------+
|                     Bandwidth-Delay Product Problem                      |
+--------------------------------------------------------------------------+
|  Connection: 10 Gbps Cloud Connection across US to EU (Latency: 80ms)    |
|  BDP Calculation = 10,000,000,000 * 0.080 = 800,000,000 bits = 100 MB!   |
|                                                                          |
|  Default Socket Buffer: 4 MB (Too small!)                                |
|  Result: Max throughput is throttled to only 400 Mbps over 10 Gbps pipe! |
|                                                                          |
|  Tuned Socket Buffer: 128 MB                                             |
|  Result: Full 10 Gbps line-rate saturation achieved!                     |
+--------------------------------------------------------------------------+
```

---

## 2. Tuning TCP Socket Buffers (`tcp_rmem` and `tcp_wmem`)

In Linux, TCP automatically manages socket buffer scaling using dynamic auto-tuning. The boundaries for this auto-tuning are defined by vectors of **three integers**:
`min  default  max` (values in bytes).

- **`min`**: Minimum memory reserved per socket (even under memory pressure).
- **`default`**: Initial buffer allocated when socket is created.
- **`max`**: Maximum buffer ceiling the auto-tuner is permitted to scale up to.

```ini
# TCP Read Buffer (Receive / Inbound): min, default, max
net.ipv4.tcp_rmem = 4096 87380 67108864

# TCP Write Buffer (Send / Outbound): min, default, max
net.ipv4.tcp_wmem = 4096 65536 67108864
```
Here, `67108864` bytes sets the maximum buffer ceiling to **64 Megabytes**, permitting multi-gigabit throughput across international latency spans.

---

## 3. Global Network Core Memory Limits

The TCP buffer maximums (`tcp_rmem`, `tcp_wmem`) are constrained by the **global kernel core buffer ceilings** (`rmem_max` and `wmem_max`). 

If `rmem_max` is smaller than `tcp_rmem` max, the socket buffer cannot expand!

```ini
# Maximum receive buffer for all socket types (in bytes)
net.core.rmem_max = 67108864

# Maximum send buffer for all socket types (in bytes)
net.core.wmem_max = 67108864

# Default receive buffer
net.core.rmem_default = 1048576

# Default send buffer
net.core.wmem_default = 1048576
```

---

## 4. Hardware NIC Ring Buffers (`ethtool -G`)

Before packets reach the kernel TCP buffers, they land in the physical **NIC Ring Buffers** inside the network card. 

Under heavy traffic bursts, small ring buffers cause the NIC to drop packets before the kernel ever sees them.

```bash
# Check current and maximum supported ring buffer sizes
sudo ethtool -g eth0
```

```
Ring parameters for eth0:
Pre-set maximums:
RX:		4096
TX:		4096
Current hardware settings:
RX:		512    <-- DANGER: Running at only 1/8th of hardware capability!
TX:		512
```

### Expanding Ring Buffers to Hardware Maximums:
```bash
# Increase RX and TX ring buffers to 4096
sudo ethtool -G eth0 rx 4096 tx 4096
```

Make permanent via systemd or netplan:
On Ubuntu netplan, configure under `network.ethernets.eth0.receive-ring-size: 4096`.

---

## 5. Summary Takeaways

- Bandwidth-Delay Product (BDP) dictates minimum socket buffer size for line-rate throughput.
- Set `net.core.rmem_max` and `wmem_max` to at least 32MB–64MB for 10GbE+ networks.
- Configure `net.ipv4.tcp_rmem` and `tcp_wmem` auto-tuning maximums to match `rmem_max`.
- Expand hardware NIC ring buffers to their hardware maximums with `ethtool -G <iface> rx 4096`.
