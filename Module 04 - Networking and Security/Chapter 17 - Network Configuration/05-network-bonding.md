# Network Bonding in Linux: Link Aggregation, High Availability, and LACP

## Overview

In enterprise data centers and bare-metal cloud infrastructure, relying on a single network interface creates a dangerous **Single Point of Failure (SPOF)**. If a physical network cable is accidentally unplugged, an SFP+ transceiver overheats, or a top-of-rack switch port fails, the entire server drops offline.

**Network Bonding** (also known as **Link Aggregation**, **NIC Teaming**, or **Port Channeling**) combines multiple physical network interfaces into a single logical virtual interface (e.g. `bond0`).

Network bonding delivers two foundational infrastructure capabilities:
1. **High Availability & Fault Tolerance**: Automatic sub-second failover if a physical link drops.
2. **Increased Bandwidth Capacity**: Aggregating two 10Gbps or 25Gbps NICs into a single 20Gbps or 50Gbps high-throughput pipe.

---

## 1. The 7 Linux Bonding Modes

The Linux kernel bonding driver supports seven distinct operational modes (Modes 0 through 6):

| Mode | Name | Primary Objective | Requires Switch Configuration? | Description |
| :---: | :--- | :--- | :---: | :--- |
| **`0`** | **`balance-rr`** | Bandwidth Aggregation | Yes (Static Trunk) | Round-robin packet distribution. Can cause out-of-order TCP packets. |
| **`1`** | **`active-backup`**| **Fault Tolerance (HA)** | **NO (Zero switch config)** | **Only one NIC is active**. If active NIC fails, secondary takes over instantly. |
| **`2`** | **`balance-xor`** | Bandwidth + Fault Tolerance | Yes | XOR hash of source and destination MACs. |
| **`3`** | **`broadcast`** | Extreme Fault Tolerance | Yes | Transmits identical packets on all interfaces simultaneously. |
| **`4`** | **`802.3ad` (LACP)**| **Bandwidth + Full HA** | **YES (LACP required)** | **The Enterprise Industry Standard**. IEEE 802.3ad Dynamic Link Aggregation. |
| **`5`** | **`balance-tlb`** | Transmit Load Balancing | No | Distributes outgoing traffic based on current load; receive via current NIC. |
| **`6`** | **`balance-alb`** | Adaptive Load Balancing | No | Includes TLB plus ARP negotiation to balance incoming IPv4 traffic. |

> [!TIP]
> **DevOps Rule of Thumb**:
> - If your network switches do **NOT** support LACP (or you don't control the switches): Use **Mode 1 (`active-backup`)**.
> - In enterprise data centers with managed switches configured for LACP: Use **Mode 4 (`802.3ad`)**.

---

## 2. Production Netplan Bonding Configuration

Here is an enterprise Netplan configuration bonding two physical NICs (`enp3s0` and `enp4s0`) into a high-availability `bond0` interface:

### Blueprint: Mode 1 (`active-backup`)
```yaml
# /etc/netplan/01-bonding.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp3s0:
      dhcp4: false
    enp4s0:
      dhcp4: false

  bonds:
    bond0:
      interfaces:
        - enp3s0
        - enp4s0
      addresses:
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1
      parameters:
        mode: active-backup
        mii-monitor-interval: 100   # Checks link health every 100ms
        primary: enp3s0             # Preferred active interface
      nameservers:
        addresses: [1.1.1.1, 8.8.8.8]
```

### Blueprint: Mode 4 (`802.3ad` LACP Aggregation)
```yaml
  bonds:
    bond0:
      interfaces: [enp3s0, enp4s0]
      addresses: [10.0.1.50/24]
      parameters:
        mode: 802.3ad
        mii-monitor-interval: 100
        lacp-rate: fast             # Transmit LACP packets every 1 second
        transmit-hash-policy: layer2+3
```

---

## 3. Monitoring Bond Health and Status: `/proc/net/bonding/`

The Linux kernel exposes the real-time operational status of all bonded interfaces via `/proc`:

```bash
$ cat /proc/net/bonding/bond0
Ethernet Channel Bonding Driver: v6.8.0

Bonding Mode: fault-tolerance (active-backup)
Primary Slave: enp3s0 (primary_reselect always)
Currently Active Slave: enp3s0
MII Status: up
MII Polling Interval (ms): 100
Up Delay (ms): 0
Down Delay (ms): 0

Slave Interface: enp3s0
MII Status: up
Speed: 10000 Mbps
Duplex: full
Link Failure Count: 0

Slave Interface: enp4s0
MII Status: up
Speed: 10000 Mbps
Duplex: full
Link Failure Count: 0
```

### Testing Failover in Production:
To verify that failover works without physically pulling cables, administratively down the active interface:
```bash
sudo ip link set dev enp3s0 down
```
Check `/proc/net/bonding/bond0`. You will see `Currently Active Slave` instantly switches to `enp4s0` with zero dropped TCP connections!

---

## Summary

- Network Bonding aggregates multiple physical NICs into a single logical `bond0` interface.
- Mode 1 (`active-backup`) provides immediate failover without requiring switch configuration.
- Mode 4 (`802.3ad` LACP) provides dynamic link aggregation and throughput doubling in enterprise data centers.
- The `mii-monitor-interval: 100` setting instructs the kernel to poll link carrier health every 100ms.
- Inspect real-time bond health and failovers via `cat /proc/net/bonding/bond0`.
