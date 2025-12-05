# NetworkManager in Linux: Architecture, `nmcli`, and `nmtui`

## Overview

**NetworkManager** is the default network configuration and management daemon across the enterprise Red Hat ecosystem (RHEL 8/9/10, Rocky Linux, AlmaLinux, CentOS Stream, Fedora) and Ubuntu Desktop workstations.

NetworkManager dynamically monitors and configures network devices, managing Ethernet NICs, Wi-Fi adapters, VLANs, software bridges, and VPN tunnels through a central D-Bus API.

For headless servers and automated DevOps environments, NetworkManager provides two primary interfaces:
- **`nmcli`**: A comprehensive command-line interface designed for terminal administration and Bash/Ansible scripting.
- **`nmtui`**: A curses-based Text User Interface for interactive terminal configuration without a desktop GUI.

---

## 1. NetworkManager Mental Model: Devices vs Connections

The fundamental concept to master in NetworkManager is the distinction between a **Device** and a **Connection**:

```text
+-------------------------------------------------------------------------------+
|                       NETWORKMANAGER ARCHITECTURE                             |
+-------------------------------------------------------------------------------+
|  DEVICE (Hardware Interface)             CONNECTION (Configuration Profile)   |
|  - The physical or virtual NIC           - A saved configuration profile      |
|  - Examples: enp3s0, eth0, wlp2s0        - Contains IP, Gateway, DNS, MTU     |
|  - Represents kernel object              - Stored in /etc/NetworkManager/     |
+-------------------------------------------------------------------------------+
```

A single physical **Device** (`enp3s0`) can have multiple saved **Connection profiles** (e.g., `"Office-Static"` vs `"DataCenter-DHCP"`), though only one connection profile can be actively applied to that device at any given moment.

---

## 2. Inspecting the Network Stack with `nmcli`

```bash
# Check overall NetworkManager health and connectivity:
$ nmcli general status
STATE      CONNECTIVITY  WIFI-HW  WIFI     WWAN-HW  WWAN
connected  full          enabled  enabled  enabled  enabled

# Inspect physical hardware devices and their bound profiles:
$ nmcli device status
DEVICE   TYPE      STATE      CONNECTION
enp3s0   ethernet  connected  Wired-Profile-1
docker0  bridge    connected  docker0
lo       loopback  unmanaged  --

# View all saved connection profiles:
$ nmcli connection show
NAME             UUID                                  TYPE      DEVICE
Wired-Profile-1  3a12b456-789c-4def-0123-456789abcdef  ethernet  enp3s0
```

---

## 3. High-Frequency `nmcli` Automation Recipes

### A. Create a New Static IP Connection Profile
Create a production connection profile named `prod-net` bound to interface `enp3s0`:

```bash
sudo nmcli connection add \
    type ethernet \
    con-name "prod-net" \
    ifname enp3s0 \
    ipv4.method manual \
    ipv4.addresses "192.168.1.100/24" \
    ipv4.gateway "192.168.1.1" \
    ipv4.dns "1.1.1.1,8.8.8.8" \
    connection.autoconnect yes
```

### B. Activate and Switch Connection Profiles
Activate the profile immediately onto the interface:

```bash
sudo nmcli connection up "prod-net"
# Output: Connection successfully activated (D-Bus active path: ...)
```

### C. Modify Existing Connection Parameters
Update DNS servers on an active connection:

```bash
# Add Cloudflare and Google DNS:
sudo nmcli connection modify "prod-net" ipv4.dns "1.1.1.1 8.8.8.8"

# Reactivate to apply changes:
sudo nmcli connection up "prod-net"
```

### D. Switch a Connection from Static to DHCP
```bash
sudo nmcli connection modify "prod-net" ipv4.method auto
sudo nmcli connection up "prod-net"
```

---

## 4. The Interactive Terminal Interface: `nmtui`

When configuring a remote server interactively over SSH and you prefer not to memorize long `nmcli` flags:

```bash
sudo nmtui
```

`nmtui` launches an interactive, keyboard-navigated text menu directly in your terminal:
- **Edit a connection**: Configure static IP, gateways, DNS, and MTU using cursor keys.
- **Activate a connection**: Toggle interfaces UP or DOWN.
- **Set system hostname**: Update the hostname atomically.

---

## 5. Keyfiles: How NetworkManager Stores Profiles on Disk

Modern NetworkManager versions store connection profiles as plaintext ini-style **keyfiles** under `/etc/NetworkManager/system-connections/`:

```ini
# /etc/NetworkManager/system-connections/prod-net.nmconnection
[connection]
id=prod-net
uuid=3a12b456-789c-4def-0123-456789abcdef
type=ethernet
interface-name=enp3s0
autoconnect=true

[ipv4]
address1=192.168.1.100/24,192.168.1.1
dns=1.1.1.1;8.8.8.8;
method=manual
```
*(Permissions on keyfiles must be strictly `-rw-------` `0600` owned by `root`).*

```bash
# Reload keyfiles modified by Ansible/scripts:
sudo nmcli connection reload
```

---

## Summary

- NetworkManager is the enterprise networking standard for RHEL, Rocky Linux, and Fedora.
- Distinguish between **Devices** (hardware NICs) and **Connections** (configuration profiles).
- Manage networks programmatically using `nmcli` and interactively using `nmtui`.
- Saved profiles are stored as keyfiles in `/etc/NetworkManager/system-connections/`.
- Reload manual file edits with `nmcli connection reload` and activate profiles with `nmcli con up <name>`.
