# The `ifconfig` Command vs `ip`: Historical Context & Modern Migration

## Overview

The `ifconfig` (interface configuration) command is one of the most famous Unix networking utilities in computing history, originating in 1983 with 4.2BSD Unix. For nearly two decades, typing `ifconfig` was the universal reflex of every Unix and Linux systems administrator.

However, in modern Linux engineering, **`ifconfig` is officially obsolete, deprecated, and no longer maintained**. On modern server distributions (Ubuntu 20.04+, Debian 11+, RHEL 8+, Rocky Linux, and Alpine), `ifconfig` is not even installed by default.

This lesson documents `ifconfig` for legacy system maintenance while providing a comprehensive migration guide to the modern `iproute2` standard.

---

## 1. Why Was `ifconfig` Deprecated?

The Linux kernel networking subsystem underwent massive architectural advancements that the legacy `net-tools` code base (`ifconfig`, `route`, `arp`) could no longer support:

```text
Feature / Capability                       Legacy ifconfig       Modern iproute2 (ip)
──────────────────────────────────────────────────────────────────────────────────────
Multiple IPv4 addresses per interface     Requires fake aliases Native support
                                           (eth0:0, eth0:1)     (ip addr add)
Network Namespaces (Docker / K8s)         UNSUPPORTED           Fully integrated (ip netns)
Policy-Based Routing (ip rule)             UNSUPPORTED           Fully integrated
Traffic Control & QoS (tc)                UNSUPPORTED           Fully integrated
JSON output for scripting (-j)            UNSUPPORTED           Natively supported
Bonding, Teaming, WireGuard, VXLAN        UNSUPPORTED           Fully integrated
```

---

## 2. The Rosetta Stone: `ifconfig` to `ip` Translation Guide

When reading older documentation, legacy Bash scripts, or working on older production servers, use this quick reference mapping:

| Task / Objective | Legacy Command (`ifconfig`) | Modern Standard (`ip`) |
| :--- | :--- | :--- |
| **View all active interfaces** | `ifconfig` | `ip a` *(or `ip -br a`)* |
| **View all interfaces (including down)**| `ifconfig -a` | `ip a` |
| **Inspect a specific interface** | `ifconfig eth0` | `ip a show eth0` |
| **Enable an interface** | `ifconfig eth0 up` | `ip link set eth0 up` |
| **Disable an interface** | `ifconfig eth0 down` | `ip link set eth0 down` |
| **Assign an IP and Netmask** | `ifconfig eth0 192.168.1.50 netmask 255.255.255.0` | `ip addr add 192.168.1.50/24 dev eth0` |
| **Remove an IP address** | `ifconfig eth0 0.0.0.0` | `ip addr del 192.168.1.50/24 dev eth0` |
| **Add a secondary IP (Alias)** | `ifconfig eth0:1 10.0.0.5 netmask 255.255.255.0` | `ip addr add 10.0.0.5/24 dev eth0` |
| **Change MAC Address** | `ifconfig eth0 hw ether 00:11:22:33:44:55` | `ip link set eth0 address 00:11:22:33:44:55` |
| **Change MTU Size** | `ifconfig eth0 mtu 9000` | `ip link set eth0 mtu 9000` |
| **View Network Statistics** | `ifconfig -s` | `ip -s link` |

---

## 3. Installing `net-tools` (For Legacy Compatibility Only)

If an older vendor appliance or legacy deployment script requires `ifconfig`, you can install it from the `net-tools` package:

```bash
# Ubuntu / Debian:
sudo apt update && sudo apt install -y net-tools

# RHEL / Rocky / CentOS / Fedora:
sudo dnf install -y net-tools
```

---

## Summary

- `ifconfig` is a deprecated legacy tool that is absent from modern Linux distributions by default.
- Never write new DevOps automation scripts, Dockerfiles, or cloud-init configs using `ifconfig`.
- Use the modern `ip` command (`ip link`, `ip addr`, `ip route`) for all production infrastructure.
