# Chapter 44: Linux Namespaces - Network Namespaces and Virtual Networking

## 1. Introduction: What is a Network Namespace?

A **Network Namespace** (`netns`) provides complete logical isolation of the system's network stack. 

Inside an isolated network namespace, a process sees its own private:
- **Network Interface Devices** (physical Ethernet, virtual ethernet `veth`, loopback `lo`).
- **IPv4 and IPv6 Routing Tables**.
- **Packet Filtering Firewalls** (`iptables`, `nftables`).
- **Listening TCP/UDP Socket Ports** (Port 80 can be bound simultaneously by 10 different containers on the same physical host without conflict!).
- **ARP Tables and Neighbor Discovery**.

In Kubernetes and Docker, every single container or Pod runs inside its own dedicated Network Namespace.

```
+--------------------------------------------------------------------------+
|                       Virtual Ethernet (veth) Bridge                     |
+--------------------------------------------------------------------------+
|  Host Network Namespace                                                  |
|  - Physical NIC: eth0 (IP: 198.51.100.10)                                |
|  - Linux Bridge: docker0 / cni0 (IP: 172.17.0.1)                         |
|                      |                                                   |
|                      v (One end of virtual cable plugged into bridge)    |
|                  [ veth0_host ]                                          |
|                         |                                                |
|             ========== Virtual Ethernet Cable ==========                 |
|                         |                                                |
|  Container Net Namespace v (Other end plugged into container!)           |
|                  [ eth0_cont ] (IP: 172.17.0.2)                          |
|  - Loopback: lo                                                          |
|  - Default Gateway: 172.17.0.1                                           |
+--------------------------------------------------------------------------+
```

---

## 2. Managing Network Namespaces with `ip netns`

The `iproute2` suite provides the **`ip netns`** command subsystem to create, inspect, and wire network namespaces manually.

```bash
# 1. Create two isolated network namespaces: "red" and "blue"
sudo ip netns add red
sudo ip netns add blue

# 2. List all network namespaces
ip netns list
```
```
blue
red
```

```bash
# 3. Execute commands inside a network namespace using 'ip netns exec'
sudo ip netns exec red ip link show
```
```
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```
Notice that only `lo` exists, and it is currently `DOWN`. The host's physical `eth0` is completely invisible!

---

## 3. Connecting Two Namespaces: The Virtual Ethernet Pair (`veth`)

A **veth pair** acts as a bidirectional virtual Ethernet patch cable. Any packet transmitted into one end of the veth pair instantly emerges from the other end:

```bash
# 1. Create a connected veth pair: 'veth-red' <===> 'veth-blue'
sudo ip link add veth-red type veth peer name veth-blue

# 2. Move 'veth-red' into the 'red' namespace:
sudo ip link set veth-red netns red

# 3. Move 'veth-blue' into the 'blue' namespace:
sudo ip link set veth-blue netns blue
```

### Configuring IP Addresses and Bringing Links Up:
```bash
# Configure 'red' namespace:
sudo ip netns exec red ip addr add 10.0.0.1/24 dev veth-red
sudo ip netns exec red ip link set veth-red up
sudo ip netns exec red ip link set lo up

# Configure 'blue' namespace:
sudo ip netns exec blue ip addr add 10.0.0.2/24 dev veth-blue
sudo ip netns exec blue ip link set veth-blue up
sudo ip netns exec blue ip link set lo up
```

### Testing End-to-End Connectivity:
Ping from `red` into `blue` across the virtual cable:
```bash
sudo ip netns exec red ping -c 3 10.0.0.2
```
```
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.045 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.038 ms
64 bytes from 10.0.0.2: icmp_seq=3 ttl=64 time=0.041 ms
```
You have built a functioning container network stack by hand!

---

## 4. Kubernetes Pod Networking: The Pause Container

A common architectural question in Kubernetes:
```
"How do multiple containers inside the same Pod (e.g., App + Istio Sidecar) communicate over localhost?"
```

### The Architecture:
1. Kubernetes launches an ultra-minimal C binary called the **`pause` container**.
2. The `pause` container creates a single Network Namespace.
3. When the application container and sidecar container start, Kubernetes uses the Docker/CRI flag `--net=container:pause` (joining the `pause` container's network namespace).
4. **Result**: Both containers share the exact same `lo` loopback device, IP address, and port space!

---

## 5. Summary Takeaways

- Network namespaces isolate network interfaces, IP addresses, routing, and ports.
- Multiple containers can bind to port `80` simultaneously on one host using separate network namespaces.
- Containers connect to host bridges using **virtual ethernet pairs (`veth`)**.
- Kubernetes Pods share a single network namespace across all containers via the `pause` container.
