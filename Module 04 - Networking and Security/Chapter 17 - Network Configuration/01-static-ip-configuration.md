# Static IP Address Configuration in Linux: Netplan, NetworkManager, and Legacy Files

## Overview

While consumer workstations and cloud instances often obtain network parameters dynamically via DHCP, production servers—such as Kubernetes master nodes, database clusters, reverse proxies, and storage appliances—require **deterministic Static IP addresses**.

If a database server changes its IP address dynamically upon reboot, backend services, firewall rules, and DNS records will fail immediately.

Because Linux network management has evolved across generations, configuring static IPs depends on your distribution's network stack:
- **Ubuntu 18.04+ (Modern)**: Netplan YAML manifests.
- **RHEL 8+ / Rocky Linux / Fedora**: NetworkManager Keyfiles or `nmcli`.
- **Debian / Legacy**: `/etc/network/interfaces`.

---

## 1. Modern Ubuntu: Static IP via Netplan

Netplan is the declarative, YAML-based network configuration abstraction layer used by modern Ubuntu Server systems.

Configuration files reside in `/etc/netplan/` (typically `01-netcfg.yaml` or `50-cloud-init.yaml`).

### The Production Static IP Template (`/etc/netplan/01-static.yaml`)
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp3s0:
      dhcp4: false
      dhcp6: false
      addresses:
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1
          metric: 100
      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8
        search:
          - company.internal
```

### The Remote Administrator's Life-Saver: `netplan try`
If you apply a syntax error in your IP configuration over SSH using `netplan apply`, the network will drop and **you will be locked out of the server permanently**!

Instead, always use **`netplan try`**:
```bash
sudo netplan try
```
- `netplan try` applies the configuration temporarily and starts a **120-second countdown**.
- If your SSH connection survives and you press `ENTER`, the changes become permanent.
- If the network drops or you fail to confirm within 120 seconds, **Netplan automatically rolls back to the previous working configuration**!

```bash
# If confident (e.g., local console), apply directly:
sudo netplan apply
```

---

## 2. Modern RHEL / Rocky Linux: Static IP via `nmcli`

On Red Hat Enterprise Linux, Rocky Linux, and CentOS Stream, NetworkManager is the default network engine.

### Configuring Static IP with a Single Command:
```bash
# 1. Modify connection profile to static IP, gateway, and DNS:
sudo nmcli connection modify "enp3s0" \
    ipv4.method manual \
    ipv4.addresses "192.168.1.100/24" \
    ipv4.gateway "192.168.1.1" \
    ipv4.dns "1.1.1.1,8.8.8.8"

# 2. Reactivate connection to apply changes immediately:
sudo nmcli connection up "enp3s0"
```

---

## 3. Legacy Configuration Systems (For Reference)

### A. Debian Legacy: `/etc/network/interfaces`
```ini
# /etc/network/interfaces
auto eth0
iface eth0 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 1.1.1.1 8.8.8.8
```
*Apply with*: `sudo systemctl restart networking`

### B. RHEL 7 / CentOS 7 Legacy: `/etc/sysconfig/network-scripts/ifcfg-eth0`
```ini
DEVICE=eth0
BOOTPROTO=static
ONBOOT=yes
IPADDR=192.168.1.100
PREFIX=24
GATEWAY=192.168.1.1
DNS1=1.1.1.1
```
*Apply with*: `sudo systemctl restart network`

---

## Summary

- Production databases, reverse proxies, and Kubernetes control planes require persistent Static IPs.
- On Ubuntu Server, configure static IPs via Netplan YAML in `/etc/netplan/`.
- Always test Netplan configurations remotely over SSH using `sudo netplan try` to enable automated 120-second rollback.
- On RHEL and Rocky Linux, configure static networking using `nmcli connection modify`.
