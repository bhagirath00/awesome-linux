# Netplan Configuration in Depth: Declarative YAML Networking for Ubuntu

## Overview

**Netplan** is Canonical's default network configuration utility introduced in Ubuntu 18.04 LTS and standard in all modern Ubuntu Server releases (20.04, 22.04, 24.04).

Unlike legacy network stacks where administrators had to write shell scripts or disparate configuration files depending on the backend, Netplan acts as a **declarative abstraction layer**: you define your entire network architecture (IPs, routes, DNS, VLANs, bridges, bonds) in structured **YAML manifests**, and Netplan translates those manifests into native configurations for your chosen backend renderer (`systemd-networkd` or `NetworkManager`).

---

## 1. Netplan Architecture & The Two Renderers

```text
               [/etc/netplan/*.yaml Manifests]
                              │
                    [ Netplan Generator ]
                              │
               ┌──────────────┴──────────────┐
               ▼                             ▼
       [ systemd-networkd ]          [ NetworkManager ]
       (Default on Servers)          (Default on Desktops / Wi-Fi)
               │                             │
               └──────────────┬──────────────┘
                              ▼
                  Linux Kernel Networking
```

### Which Renderer Should You Use?
- **`renderer: networkd`**: The production standard for Ubuntu Server, cloud VMs (AWS, GCP, Azure), Kubernetes nodes, and container hosts. It is lightweight, fast, and has zero desktop dependencies.
- **`renderer: NetworkManager`**: Recommended for laptops, workstations with dynamic Wi-Fi connections, or environments integrating with desktop control panels.

---

## 2. YAML Syntax Rules & File Precedence

Netplan configuration files reside in `/etc/netplan/` with a `.yaml` extension (e.g. `01-netcfg.yaml`, `50-cloud-init.yaml`).

> [!WARNING]
> **Strict YAML Rules Apply**:
> 1. Use **spaces for indentation** (typically 2 spaces). **TAB CHARACTERS ARE FORBIDDEN**. A single tab will cause Netplan to fail immediately.
> 2. Files are evaluated in alphabetical order. Files starting with `99-` override directives from `01-`.

---

## 3. Production Netplan Configuration Blueprints

### Blueprint A: Dual-NIC Server (Static Public IP + Private Cluster Subnet)
```yaml
# /etc/netplan/01-server-network.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    # Public WAN Interface
    enp3s0:
      dhcp4: false
      addresses:
        - 198.51.100.50/24
      routes:
        - to: default
          via: 198.51.100.1
          metric: 100
      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8

    # Private High-Speed LAN for Database / Kubernetes Cluster
    enp4s0:
      dhcp4: false
      addresses:
        - 10.0.10.50/16
      mtu: 9000   # Jumbo frames for high-throughput database replication
```

---

### Blueprint B: Linux Software Bridge (For KVM Virtual Machines or LXD)
```yaml
# /etc/netplan/02-bridge.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp3s0:
      dhcp4: false

  bridges:
    br0:
      interfaces: [enp3s0]
      dhcp4: false
      addresses:
        - 192.168.1.150/24
      routes:
        - to: default
          via: 192.168.1.1
      parameters:
        stp: true
        forward-delay: 4
      nameservers:
        addresses: [1.1.1.1]
```

---

### Blueprint C: 802.1Q VLAN Tagging
```yaml
# /etc/netplan/03-vlan.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp3s0:
      dhcp4: false

  vlans:
    vlan100:
      id: 100
      link: enp3s0
      addresses:
        - 10.100.0.5/24
```

---

## 4. Operational Commands: Validate, Try, and Apply

### 1. Validate YAML Syntax Without Touching the Network
```bash
sudo netplan generate
```
*(If Netplan prints nothing, your YAML syntax is 100% valid. If there is an indentation error, it points to the exact line number).*

### 2. The Safe Remote Test: `netplan try`
Always use `netplan try` when working over SSH:
```bash
$ sudo netplan try
Do you want to keep these settings?

Press ENTER before the timeout to accept the new configuration
Changes will revert in 120 seconds
```
If you make a typo that severs SSH, do nothing. Wait 2 minutes, and the server automatically reverts to your previous working configuration!

### 3. Commit Changes Immediately
```bash
sudo netplan apply
```

### 4. Deep Diagnostic Debugging
If a configuration fails to apply:
```bash
sudo netplan --debug apply
```

---

## Summary

- Netplan is Ubuntu's declarative YAML networking abstraction layer.
- Uses `renderer: networkd` for headless production servers and cloud instances.
- Never use tabs in `/etc/netplan/*.yaml` files; use strict 2-space indentation.
- Validate YAML syntax with `netplan generate`.
- Always apply remote network changes with `sudo netplan try` to protect against SSH lockouts with automatic 120-second rollback.
