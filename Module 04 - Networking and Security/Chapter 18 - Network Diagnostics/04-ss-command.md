# The `ss` Command in Linux: High-Performance Socket Auditing and Diagnostics

## Overview

The **`ss`** (Socket Statistics) utility—part of the modern `iproute2` package—is the direct, modern replacement for legacy `netstat`.

While `netstat` parses text files line-by-line out of `/proc/net/`, `ss` interfaces directly with the Linux kernel's **`sock_diag` Netlink subsystem**. Because it queries the kernel binary interface directly, `ss` is **dramatically faster** (retrieving 100,000 active connections in under 50ms where `netstat` freezes), uses less CPU, and exposes deep TCP internal metrics like **RTT**, **congestion window sizes**, and **TCP BBR algorithm metrics**.

In cloud operations, Kubernetes node troubleshooting, and SRE incident triage, `ss` is the definitive tool for socket inspection.

---

## 1. The Daily Standard: `ss -tulnp`

The most common command to check which services are listening on which ports:

```bash
sudo ss -tulnp
```

```text
Netid  State   Recv-Q  Send-Q   Local Address:Port   Peer Address:Port  Process
tcp    LISTEN  0       128            0.0.0.0:22          0.0.0.0:*      users:(("sshd",pid=842,fd=3))
tcp    LISTEN  0       511            0.0.0.0:80          0.0.0.0:*      users:(("nginx",pid=1420,fd=6))
tcp    LISTEN  0       511            0.0.0.0:443         0.0.0.0:*      users:(("nginx",pid=1420,fd=7))
tcp    LISTEN  0       128          127.0.0.1:3306        0.0.0.0:*      users:(("mysqld",pid=1024,fd=21))
udp    UNCONN  0       0        127.0.0.53%lo:53          0.0.0.0:*      users:(("systemd-resolve",pid=620,fd=13))
```

### Understanding `Recv-Q` and `Send-Q` in Sockets
- In **`LISTEN` state**:
  - **`Send-Q`**: The maximum size of the listen backlog queue (connections waiting to be accepted).
  - **`Recv-Q`**: The number of client connections currently waiting in the backlog queue. *(If `Recv-Q` approaches `Send-Q`, your application is thread-blocked and dropping new incoming traffic!)*.
- In **`ESTABLISHED` state**:
  - **`Recv-Q`**: Bytes received by the kernel not yet read by the application.
  - **`Send-Q`**: Bytes queued in the kernel waiting to be sent across the network.

---

## 2. Essential Flags Reference

| Flag | Long Option | Description | DevOps Practical Context |
| :--- | :--- | :--- | :--- |
| **`-t`** | `--tcp` | Displays TCP sockets. | Filtering web and database traffic. |
| **`-u`** | `--udp` | Displays UDP sockets. | Filtering DNS, WireGuard, and NTP sockets. |
| **`-x`** | `--unix` | Displays **Unix Domain Sockets**. | **Inspecting Docker (`/var/run/docker.sock`) and systemd sockets.** |
| **`-l`** | `--listening` | Shows listening sockets only. | Discovering open ports. |
| **`-a`** | `--all` | Shows both listening and established sockets. | Full socket audit. |
| **`-n`** | `--numeric` | Displays raw port numbers instead of service names. | Fast output; avoids slow DNS lookups. |
| **`-p`** | `--processes` | Shows process names and PIDs. | Finding which service is hogging a port. |
| **`-s`** | `--summary` | **Outputs global socket statistics**. | Instant cluster connection health overview. |
| **`-i`** | `--info` | Shows internal TCP performance metrics (RTT, cwnd). | Deep network performance debugging. |

---

## 3. Instant Cluster Health: `ss -s`

When troubleshooting an overloaded server, before dumping thousands of socket lines, run `ss -s`:

```bash
$ ss -s
Total: 482
TCP:   1284 (estab 85, closed 1140, orphaned 0, timewait 1020)

Transport Total     IP        IPv6
RAW	  0         0         0        
UDP	  4         3         1        
TCP	  144       140       4        
INET	  148       143       5        
FRAG	  0         0         0        
```
*(Notice: 1020 sockets in `timewait`! This immediately informs the SRE team that HTTP connection pooling is needed).*

---

## 4. Advanced State Filtering

`ss` features built-in state filtering expressions that eliminate messy `grep` pipelines:

```bash
# View only ESTABLISHED TCP connections:
ss -t state established

# View only connections stuck in TIME_WAIT:
ss -t state time-wait

# View all actively connected sockets (syn-sent, established, close-wait):
ss -t state connected
```

---

## 5. Native Port and Address Filtering

You can filter traffic natively using BPF-like boolean expressions:

```bash
# Find all connections to or from HTTPS (Port 443):
ss -t '( sport = :443 or dport = :443 )'

# Find which process is listening on Port 8080:
sudo ss -tlp 'sport = :8080'

# Find all established connections to a specific remote database IP:
ss -t dst 10.0.1.25:5432
```

---

## 6. Inspecting Unix Domain Sockets (`-x`)

In containerized architectures, IPC sockets like Docker, containerd, and systemd are Unix Domain sockets:

```bash
$ sudo ss -xlp
Netid  State   Recv-Q  Send-Q   Local Address:Port      Process
u_str  LISTEN  0       128      /var/run/docker.sock    users:(("dockerd",pid=1240,fd=3))
u_str  LISTEN  0       4096     /run/containerd/containerd.sock users:(("containerd",pid=980,fd=7))
```

---

## Summary

- `ss` is the modern, high-speed replacement for `netstat`.
- Check open ports with `sudo ss -tulnp`.
- Get a high-level socket summary with `ss -s`.
- Filter by state natively: `ss -t state established`.
- Filter by port natively: `ss -tl 'sport = :8080'`.
- Inspect container sockets with `ss -xlp`.
