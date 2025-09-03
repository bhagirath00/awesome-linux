# The `netstat` Command in Linux: Network Statistics & Legacy Socket Auditing

## Overview

The `netstat` (network statistics) command is a classic Unix networking tool from the `net-tools` package. It monitors active TCP/UDP socket connections, listening daemon ports, routing tables, and interface counters.

While virtually every Linux engineer has memorized classic flags like `netstat -tulnp`, **`netstat` is formally deprecated on modern Linux systems**. On busy production servers with tens of thousands of active connections, `netstat` reads sequentially from `/proc/net/` files, causing noticeable kernel lockups.

This lesson covers `netstat` for maintaining legacy systems and scripts while demonstrating the TCP connection lifecycle.

---

## 1. The Classic Flag Combination: `netstat -tulnp`

The most famous flag combination in Linux systems administration:

```bash
sudo netstat -tulnp
```

```text
-t  : TCP sockets only
-u  : UDP sockets only
-l  : LISTENING sockets only (services waiting for connections)
-n  : NUMERIC output (displays raw numbers like 80 instead of translating to "http")
-p  : PROCESS ID and program name responsible for the socket
```

### Sample Output:
```text
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      842/sshd: /usr/sbin 
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      1024/mysqld         
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      1420/nginx: master  
tcp6       0      0 :::443                  :::*                    LISTEN      1420/nginx: master  
udp        0      0 127.0.0.53:53           0.0.0.0:*                           620/systemd-resolve 
```

### Critical Observations:
- **`0.0.0.0:80` (or `:::80`)**: Nginx is listening on **all network interfaces** (publicly accessible).
- **`127.0.0.1:3306`**: MySQL is bound strictly to **localhost** (secure; inaccessible from the external network).

---

## 2. Understanding the TCP Connection Lifecycle

`netstat` reports the exact state of active TCP connections according to RFC 793:

```text
CLIENT                                                          SERVER
  │                                                               │ (State: LISTEN)
  ├────── SYN (Initiates connection) ────────────────────────────►│ (State: SYN_RECV)
  │◄───── SYN + ACK (Acknowledges connection) ────────────────────┤
  ├────── ACK (Connection established) ──────────────────────────►│
  ▼                                                               ▼
(State: ESTABLISHED) ◄────── Active Data Exchange ──────► (State: ESTABLISHED)
  │                                                               │
  ├────── FIN (Close request) ───────────────────────────────────►│ (State: CLOSE_WAIT)
  │◄───── ACK ────────────────────────────────────────────────────┤
  │                                                               ├────── FIN (Server closes) ──►
  │◄───── ACK ────────────────────────────────────────────────────┤
  ▼                                                               ▼
(State: TIME_WAIT)                                            (State: CLOSED)
(Waits 2 * MSL: 60s)
  │
  ▼
(State: CLOSED)
```

### Key TCP States in DevOps Diagnostics:
- **`LISTEN`**: A server daemon is waiting for incoming client connections.
- **`ESTABLISHED`**: Data is actively flowing between client and server.
- **`TIME_WAIT`**: The client closed the connection and is waiting 60 seconds to ensure delayed packets clear the network. (A massive pileup of thousands of `TIME_WAIT` sockets indicates an application lacking HTTP Keep-Alive connection pooling!).
- **`CLOSE_WAIT`**: The remote end closed the connection, but the local application has not yet closed its socket. (A pileup of `CLOSE_WAIT` indicates an **application resource leak** in your code!).

---

## 3. The Rosetta Stone: `netstat` to `ss` Migration

On all modern production systems, replace `netstat` with the blazing-fast **`ss` (Socket Statistics)** tool:

| Diagnostic Task | Legacy `netstat` | Modern `ss` Equivalent |
| :--- | :--- | :--- |
| **All listening TCP/UDP ports with PIDs** | `netstat -tulnp` | `ss -tulnp` |
| **All established TCP connections** | `netstat -at` | `ss -t state established` |
| **Routing Table** | `netstat -r` | `ip route` |
| **Interface Statistics** | `netstat -i` | `ip -s link` |
| **Protocol Summary Statistics** | `netstat -s` | `ss -s` |

---

## Summary

- `netstat -tulnp` displays listening TCP/UDP ports, bound addresses, and owning PIDs.
- Sockets bound to `127.0.0.1` are private; sockets bound to `0.0.0.0` or `::` are public.
- High counts of `TIME_WAIT` indicate missing connection pooling; high `CLOSE_WAIT` indicates code socket leaks.
- `netstat` is deprecated due to high CPU overhead; migrate to `ss -tulnp`.
