# Container Networking on Linux: Veth Pairs, Bridges, and Iptables

## 1. Architectural Concept & Deep Dive

When you run a container with `-p 8080:80`, what actually happens inside the Linux kernel?

Docker networking is not magic: it is a deterministic combination of:
1. **Network Namespaces (`CLONE_NEWNET`)**: Isolated TCP/IP stacks, loopbacks, routes, and iptables rules.
2. **Virtual Ethernet Pairs (`veth`)**: Software patch cables linking namespaces.
3. **Linux Network Bridges (`bridge` / `docker0`)**: Software Layer-2 Ethernet switches.
4. **IP Forwarding (`net.ipv4.ip_forward=1`)**: Kernel packet routing between network interfaces.
5. **Netfilter / Iptables (NAT table)**: Source NAT (MASQUERADE) for egress traffic to the internet, and Destination NAT (DNAT) for ingress port forwarding.

```
+─────────────────────────────────────────────────────────────────────────────+
|                                   HOST OS                                   |
|                                                                             |
|  Physical Interface: eth0 (192.168.1.50) ──► Default Gateway (Internet)    |
|                                     ▲                                       |
|                                     │ (iptables MASQUERADE / SNAT)          |
|                                     ▼                                       |
|  Linux Bridge: docker0 (172.17.0.1/16)                                      |
|       │                                     │                               |
|       ▼ veth0a                              ▼ veth1a                        |
|   (Host End)                            (Host End)                          |
+───────┼─────────────────────────────────────┼───────────────────────────────+
        │ [Virtual Cable]                     │ [Virtual Cable]
+───────┼─────────────────────────────+ +─────┼─────────────────────────────+
|       ▼ veth0b (renamed to eth0)    | |     ▼ veth1b (renamed to eth0)    |
| Container A Namespace               | | Container B Namespace             |
| IP: 172.17.0.2/16                   | | IP: 172.17.0.3/16                 |
| Default GW: 172.17.0.1              | | Default GW: 172.17.0.1            |
+─────────────────────────────────────+ +───────────────────────────────────+
```

---

## 2. Packet Flow Deep Dive: Ingress & Egress

### Ingress Flow (Port Forwarding: Host Port 8080 -> Container Port 80)
1. External packet arrives at host interface `eth0:8080` (e.g. from `203.0.113.5`).
2. Kernel **Netfilter `PREROUTING`** hook inspects packet.
3. Docker iptables rule performs **DNAT (Destination NAT)**:
   - Modifies destination IP: `192.168.1.50:8080` -> `172.17.0.2:80`.
4. Kernel routing table recognizes destination `172.17.0.2` is reachable via bridge `docker0`.
5. Packet traverses `docker0` -> `veth0a` -> `veth0b` (container `eth0`).
6. Web server inside container receives packet and processes request.

### Egress Flow (Container requesting `curl https://google.com`)
1. Container `172.17.0.2` sends packet destined for `142.250.190.46:443`.
2. Packet exits container `eth0` -> `veth0a` -> arrives at `docker0`.
3. Host kernel sees destination is outside local subnet and routes packet toward `eth0`.
4. Netfilter **`POSTROUTING`** chain evaluates rule:
   - `-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE`
5. **SNAT (Source NAT)** rewrites source IP from `172.17.0.2` to host external IP `192.168.1.50`.
6. Internet router sees packet from host, responds back, and host NAT table translates back to container.

---

## 3. The Five Standard Container Network Drivers

| Driver | Mechanics & Isolation | Use Case | Limitations |
| :--- | :--- | :--- | :--- |
| **`bridge`** (Default) | Private netns + `veth` pair + `docker0` bridge + iptables NAT. | Standard microservices, multi-container applications on single node. | Small NAT overhead for heavy traffic. |
| **`host`** | Container shares host network namespace directly (`CLONE_NEWNET` omitted). | High-throughput web servers, low-latency financial apps. | No port isolation; port conflicts with host services. |
| **`none`** | Container receives isolated loopback (`lo`) only; zero external interfaces. | Isolated batch calculators, security sandboxes, offline jobs. | Completely disconnected from outside world. |
| **`macvlan`** | Sub-interfaces created on physical NIC with unique MAC addresses. Container looks like a physical host on LAN. | Legacy apps requiring direct subnet IP, Telco workloads. | Host cannot directly ping container without bridge link. |
| **`ipvlan`** | Similar to macvlan but shares MAC address with host; operates at L2 or L3. | High container density exceeding switch MAC table limits. | Requires Linux 3.19+ and modern network hardware. |

---

## 4. Hands-on Lab: Connecting Two Isolated Network Namespaces Manually

Let's reproduce Docker's bridge networking without using Docker at all.

### Step 1: Create Two Network Namespaces
```bash
sudo ip netns add ns-web
sudo ip netns add ns-db
```

### Step 2: Create a Software Bridge
```bash
sudo ip link add name custom-br0 type bridge
sudo ip addr add 10.200.0.1/24 dev custom-br0
sudo ip link set dev custom-br0 up
```

### Step 3: Create Veth Pairs and Connect to Namespaces
```bash
# Pair 1: ns-web
sudo ip link add veth-web type veth peer name veth-web-br
sudo ip link set veth-web netns ns-web
sudo ip link set veth-web-br master custom-br0
sudo ip link set veth-web-br up

# Pair 2: ns-db
sudo ip link add veth-db type veth peer name veth-db-br
sudo ip link set veth-db netns ns-db
sudo ip link set veth-db-br master custom-br0
sudo ip link set veth-db-br up
```

### Step 4: Configure IP Addresses & Default Routes Inside Namespaces
```bash
# Configure ns-web
sudo ip netns exec ns-web ip addr add 10.200.0.10/24 dev veth-web
sudo ip netns exec ns-web ip link set dev veth-web up
sudo ip netns exec ns-web ip link set dev lo up
sudo ip netns exec ns-web ip route add default via 10.200.0.1

# Configure ns-db
sudo ip netns exec ns-db ip addr add 10.200.0.20/24 dev veth-db
sudo ip netns exec ns-db ip link set dev veth-db up
sudo ip netns exec ns-db ip link set dev lo up
sudo ip netns exec ns-db ip route add default via 10.200.0.1
```

### Step 5: Test Inter-Namespace Connectivity!
```bash
sudo ip netns exec ns-web ping -c 3 10.200.0.20
```
```
PING 10.200.0.20 (10.200.0.20) 56(84) bytes of data.
64 bytes from 10.200.0.20: icmp_seq=1 ttl=64 time=0.082 ms
64 bytes from 10.200.0.20: icmp_seq=2 ttl=64 time=0.054 ms
64 bytes from 10.200.0.20: icmp_seq=3 ttl=64 time=0.051 ms

--- 10.200.0.20 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2048ms
```
**`ns-web` successfully talked to `ns-db` over the software bridge!**

### Step 6: Enable Internet Egress for Namespaces
```bash
# Enable IP forwarding on host
sudo sysctl -w net.ipv4.ip_forward=1

# Add NAT masquerade rule via physical interface (e.g. eth0)
sudo iptables -t nat -A POSTROUTING -s 10.200.0.0/24 -o eth0 -j MASQUERADE
```

---

## 5. Kubernetes Container Network Interface (CNI) Evolution

In Kubernetes, pod networking builds upon these exact primitives, implemented by **CNI plugins**:

```
+─────────────────────────────────────────────────────────────+
|                    KUBERNETES CNI SPECTRUM                  |
|                                                             |
|   Flannel (VXLAN Overlay)                                   |
|   • Encapsulates pod packets in UDP (Port 4789).            |
|   • Easy setup, but adds CPU overhead for packet packing.   |
|                                                             |
|   Calico (BGP Direct Routing)                               |
|   • Uses Linux kernel routing table with BGP peers.         |
|   • Native wire speed, no encapsulation penalty.            |
|                                                             |
|   Cilium (eBPF)                                             |
|   • Bypasses iptables and netfilter entirely!               |
|   • Uses eBPF programs attached to TC (Traffic Control)     |
|     hooks for ultra-high throughput and microsecond latency.|
+─────────────────────────────────────────────────────────────+
```

---

## 6. Verification & Troubleshooting Runbook

### Key Network Inspection Commands
```bash
# View all active network bridges
ip link show type bridge

# Find the peer interface index of a container veth
docker exec -it <CONTAINER> cat /sys/class/net/eth0/iflink
# On host:
ip link | grep "^<INDEX>:"

# Inspect Docker NAT rules
sudo iptables -t nat -S DOCKER
sudo iptables -t nat -S POSTROUTING

# Verify host IP forwarding is enabled
sysctl net.ipv4.ip_forward
```

### Common Pitfalls & Solutions
1. **Containers cannot access the internet**:
   - Cause: `net.ipv4.ip_forward` is set to `0`, or host firewall (UFW/firewalld) drops forward traffic.
   - Solution: Run `sudo sysctl -w net.ipv4.ip_forward=1` and ensure `DEFAULT_FORWARD_POLICY="ACCEPT"` in `/etc/default/ufw`.
2. **`docker-proxy` consuming high memory**:
   - Cause: Running thousands of exposed ports triggers Docker to spin up a separate userspace `docker-proxy` process per port.
   - Solution: Set `"userland-proxy": false` in `/etc/docker/daemon.json` so Docker relies exclusively on high-performance iptables DNAT rules.
3. **Container-to-container connection refused**:
   - Cause: Containers on different user-defined bridge networks cannot communicate by design.
   - Solution: Connect containers to a shared bridge network (`docker network connect <NET> <CONTAINER>`).

---

## 7. Key Takeaways
- Container networking is built from Linux network namespaces, `veth` pairs, bridge devices (`docker0`), and iptables.
- Ingress traffic uses DNAT to redirect host ports to private container IPs.
- Egress traffic uses SNAT (MASQUERADE) to allow containers to reach the internet using the host's physical IP.
- Kubernetes CNI plugins evolve these principles from simple bridges (Flannel) to BGP routing (Calico) and next-generation eBPF bypassing (Cilium).
