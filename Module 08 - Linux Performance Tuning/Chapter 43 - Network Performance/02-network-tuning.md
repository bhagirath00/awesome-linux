# Chapter 43: Network Performance - Core Kernel Network Tuning (sysctl)

## 1. Introduction: Why Default Network Settings Fall Short

Linux distributions ship with conservative, general-purpose network defaults designed to run safely on small virtual machines with 512MB of RAM without exhausting memory.

When deploying high-throughput microservices, API gateways, load balancers (HAProxy, Nginx), or database clusters processing tens of thousands of concurrent connections, these low default limits trigger connection timeouts, dropped SYN packets, and ephemeral port exhaustion.

Tuning the kernel network subsystem via `/etc/sysctl.d/` unlocks the true hardware potential of modern cloud networks.

---

## 2. Essential Kernel Network Parameters

Below are the most critical `sysctl` network tunables for production Linux servers:

```
+--------------------------------------------------------------------------+
|                  Core Linux Network Performance Suite                    |
+--------------------------------------------------------------------------+
|  [ somaxconn ]           ===> Maximum listen socket backlog queue depth  |
|  [ tcp_max_syn_backlog ] ===> Maximum pending half-open SYN handshakes   |
|  [ netdev_max_backlog ]  ===> Maximum packets queued before TCP stack    |
|  [ ip_local_port_range ] ===> Ephemeral port range for outbound connects |
|  [ tcp_tw_reuse ]        ===> Reuses TIME_WAIT sockets safely for clients|
+--------------------------------------------------------------------------+
```

---

## 3. Detailed Parameter Breakdown

### 1. Connection Backlog: `net.core.somaxconn`
When an application calls `listen(fd, backlog)` to accept connections, `somaxconn` dictates the **maximum length of the socket listen backlog queue** for completely established connections awaiting `accept()`:
- **Default**: `128` or `4096` (Much too small for sudden web bursts).
- **Production Standard**: **`65535`**.

### 2. SYN Handshake Backlog: `net.ipv4.tcp_max_syn_backlog`
The maximum number of queued half-open connection requests (client sent SYN, server replied SYN-ACK, awaiting client ACK). If this queue overflows, legitimate incoming connections are dropped:
- **Default**: `512` or `1024`.
- **Production Standard**: **`16384`** or **`65535`**.

### 3. NIC Input Queue: `net.core.netdev_max_backlog`
The maximum number of packets queued on the input interface before being processed by the CPU protocol stack:
- **Default**: `1000`.
- **Production Standard**: **`16384`** (Essential for 10GbE / 40GbE network interfaces).

### 4. Ephemeral Outbound Ports: `net.ipv4.ip_local_port_range`
When a reverse proxy (like Nginx) or microservice initiates thousands of outbound TCP connections to backend databases or internal APIs, each connection consumes an ephemeral local port:
- **Default**: `32768 60999` (~28,000 available ports).
- **Production Standard**: **`10240 65535`** (Provides over 55,000 available outbound source ports).

### 5. Reusing TIME_WAIT Sockets: `net.ipv4.tcp_tw_reuse`
When a TCP connection closes, the socket enters the `TIME_WAIT` state for 60 seconds to ensure delayed packets are not confused with a new connection. Under heavy request volume, all ephemeral ports can become exhausted by `TIME_WAIT` sockets:
- Enabling **`net.ipv4.tcp_tw_reuse = 1`** allows the kernel to safely reuse `TIME_WAIT` sockets for outgoing connections when protocol timestamps are active.

> [!WARNING]
> Never enable the obsolete `net.ipv4.tcp_tw_recycle`! It was removed from modern Linux kernels because it breaks connections from clients behind corporate NAT firewalls. Always use `tcp_tw_reuse=1`.

---

## 4. Production Configuration: `/etc/sysctl.d/99-network.conf`

Create `/etc/sysctl.d/99-network-performance.conf`:

```ini
# Maximum socket listen queue depth
net.core.somaxconn = 65535

# Maximum pending SYN connections
net.ipv4.tcp_max_syn_backlog = 65535

# Maximum packets queued at network interface input
net.core.netdev_max_backlog = 16384

# Expand ephemeral port range for outgoing connections
net.ipv4.ip_local_port_range = 10240 65535

# Safely reuse TIME_WAIT sockets for outgoing connections
net.ipv4.tcp_tw_reuse = 1

# Reduce FIN timeout from 60s to 15s to release closed sockets faster
net.ipv4.tcp_fin_timeout = 15

# Enable TCP Window Scaling (RFC 1323)
net.ipv4.tcp_window_scaling = 1

# Enable TCP Selective Acknowledgments (SACK)
net.ipv4.tcp_sack = 1
```

Apply immediately without reboot:
```bash
sudo sysctl --system
```

---

## 5. Summary Takeaways

- Default Linux network parameters are tuned for low-memory desktop machines, not enterprise servers.
- Increase `net.core.somaxconn = 65535` to prevent listen queue overflows.
- Expand `ip_local_port_range` to `10240 65535` to avoid outbound port exhaustion on reverse proxies.
- Enable `tcp_tw_reuse = 1` to recycle closed sockets safely.
- Persist configurations cleanly in `/etc/sysctl.d/99-network-performance.conf`.
