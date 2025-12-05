# Linux Network Bridging: Software Switching for Containers & Virtual Machines

## Overview

A **Linux Network Bridge** is a software implementation of a **Layer 2 Ethernet Switch** built directly into the Linux kernel. Just like a physical hardware switch forwards frames between Ethernet cables based on MAC addresses, a software bridge forwards network frames between physical NICs and virtual network interfaces.

Network bridging is the architectural foundation of modern cloud virtualization and containerization:
- **Docker**: Automatically creates the **`docker0`** bridge to connect container virtual ethernet (`veth`) interfaces together.
- **Kubernetes**: CNI plugins (Flannel, Calico, bridge) manage bridges (e.g. `cni0`) on worker nodes.
- **KVM / QEMU / Proxmox**: Hypervisors attach Virtual Machine interfaces (`vnet0`, `tap0`) to a host bridge (`br0`), allowing VMs to appear directly on the physical LAN with their own dedicated IP addresses.

---

## 1. Architectural Diagram: The Bridge in Action

```text
+--------------------------------------------------------------------------+
|                     LINUX HOST (Hypervisor / Node)                      |
|                                                                          |
|     Virtual Machine 1            Virtual Machine 2      Docker Container |
|     (IP: 192.168.1.101)          (IP: 192.168.1.102)    (IP: 192.168.1.103)|
|             │                            │                      │        |
|             ▼ [vnet0]                    ▼ [vnet1]              ▼ [veth0]|
|    +──────────────────────────────────────────────────────────────────+  |
|    |                      LINUX SOFTWARE BRIDGE (br0)                 |  |
|    |             (Internal MAC Learning Table / Forwarding DB)        |  |
|    +──────────────────────────────────────────────────────────────────+  |
|                                    │                                     |
|                                    ▼ [enp3s0]                            |
|                       Physical Host Network Card (NIC)                   |
+------------------------------------┬─────────────────────────────────────+
                                     │
                                     ▼
                   Physical LAN Network Switch & Gateway (192.168.1.1)
```

---

## 2. Managing Bridges via the Modern `ip` Command

Modern Linux manages bridges natively using `iproute2` without needing legacy third-party tools:

```bash
# 1. Create a software bridge named br0:
sudo ip link add name br0 type bridge

# 2. Attach a physical interface (enp3s0) as a slave port to the bridge:
sudo ip link set dev enp3s0 master br0

# 3. Bring the bridge interface UP:
sudo ip link set dev br0 up

# 4. Assign an IP address to the BRIDGE, not the physical slave NIC:
sudo ip addr add 192.168.1.50/24 dev br0
```

> [!IMPORTANT]
> **Crucial Bridging Rule**: Once a physical NIC (`enp3s0`) is attached to a bridge, **it loses its standalone IP address**. The IP address, gateway, and routing configuration must be assigned to the **Bridge interface (`br0`) itself**!

---

## 3. Inspecting Bridge State and MAC Learning Tables

Inspect bridge member ports and the internal Layer 2 Forwarding Database (FDB):

```bash
# View interfaces attached to bridges:
$ bridge link show
2: enp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master br0 state forwarding
7: vnet0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master br0 state forwarding

# Inspect the Bridge Forwarding Database (FDB / MAC Address table):
$ bridge fdb show br0
08:00:27:8a:42:19 dev enp3s0 master br0 permanent
52:54:00:12:34:56 dev vnet0 master br0
```

---

## 4. Production Netplan Bridge Configuration (Ubuntu KVM Host)

Here is a production-grade Netplan manifest used on KVM hypervisors and LXD container hosts:

```yaml
# /etc/netplan/01-kvm-bridge.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp3s0:
      dhcp4: false
      dhcp6: false

  bridges:
    br0:
      interfaces:
        - enp3s0
      addresses:
        - 192.168.1.50/24
      routes:
        - to: default
          via: 192.168.1.1
      parameters:
        stp: true            # Spanning Tree Protocol prevents network loops
        forward-delay: 2     # Reduces port initialization delay
      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8
```

### Why Enable STP (`stp: true`)?
The **Spanning Tree Protocol (STP)** detects and blocks network loops. If a bridge accidentally connects to multiple ports on the same physical switch, a broadcast storm can crash the physical network within seconds. Enabling `stp: true` prevents broadcast loops.

---

## Summary

- A Linux Network Bridge operates as a virtual Layer 2 Ethernet switch in the kernel.
- Bridges are the fundamental mechanism connecting Docker containers (`docker0`) and KVM virtual machines (`br0`) to the physical network.
- Once a physical NIC is enslaved to a bridge, IP addresses must be assigned to the bridge interface itself.
- Inspect bridge ports and MAC tables using `bridge link show` and `bridge fdb show`.
- Always enable Spanning Tree Protocol (`stp: true`) on production bridges to prevent fatal network broadcast loops.
