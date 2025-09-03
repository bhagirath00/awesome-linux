# Chapter 19: Firewall Management - firewalld Overview

## 1. Introduction: What is firewalld?

`firewalld` is a dynamic firewall daemon management tool used by default on modern Red Hat-based enterprise Linux distributions (RHEL, CentOS, Fedora, Rocky Linux, and AlmaLinux). 

Unlike legacy `iptables` scripts that require flushing and reloading the entire packet filtering ruleset whenever a rule changes (which terminates established TCP connections and disrupts production state), `firewalld` operates dynamically. It applies rule changes on-the-fly via D-Bus interfaces to the Linux kernel `nftables` or `netfilter` subsystem without breaking established client sessions.

```
+-------------------------------------------------------------------------+
|                         firewalld Architecture                          |
+-------------------------------------------------------------------------+
|  User Space                                                             |
|  +--------------------+   +-----------------------+   +---------------+ |
|  | firewall-cmd (CLI) |   | firewall-config (GUI) |   | Cockpit / API | |
|  +---------+----------+   +-----------+-----------+   +-------+-------+ |
|            |                          |                       |         |
|            +-------------------+      |     +-----------------+         |
|                                |      |     |                           |
|                                v      v     v                           |
|                        +-----------------------+                        |
|                        |     D-Bus Interface   |                        |
|                        +-----------+-----------+                        |
|                                    |                                    |
|                                    v                                    |
|                        +-----------------------+                        |
|                        |    firewalld Daemon   |                        |
|                        | (Dynamic State Mgmt)  |                        |
|                        +-----------+-----------+                        |
|                                    |                                    |
|  Kernel Space                      v                                    |
|                        +-----------------------+                        |
|                        | nftables / Netfilter  |                        |
|                        |  (Packet Filtering)   |                        |
|                        +-----------------------+                        |
+-------------------------------------------------------------------------+
```

---

## 2. Core Concepts: Zones and Services

### The Concept of Zones
`firewalld` categorizes incoming traffic based on the level of trust assigned to network interfaces and source IP subnets. Every packet entering a network interface is evaluated against the zone assigned to that interface.

| Predefined Zone | Trust Level | Default Behavior & Use Case |
| :--- | :--- | :--- |
| `drop` | Lowest | Drops all incoming packets without sending ICMP error responses. Only outgoing connections allowed. |
| `block` | Very Low | Rejects all incoming packets with an ICMP `host-prohibited` message. |
| `public` | Low (**Default**) | For untrusted public networks. Only explicitly allowed incoming services/ports (e.g., SSH, cockpit) permitted. |
| `external` | Low | Used on gateway routers with NAT masquerading enabled; incoming traffic blocked except selected services. |
| `dmz` | Moderate | Demilitarized Zone for public-facing servers with limited access to internal infrastructure. |
| `work` | Moderate | Used for work environments where peer workstations are generally trusted. |
| `home` | High | Used for home networks where other devices are mostly trusted. |
| `internal` | High | Used for private networks; permits common services like DNS, DHCP, and Samba. |
| `trusted` | Maximum | Accepts all network traffic on assigned interfaces without any filtering. |

### The Concept of Services
Instead of requiring administrators to memorize arbitrary port numbers and protocol variants (e.g., `80/tcp`, `443/tcp`), `firewalld` provides preconfigured XML definitions called **Services** located in `/usr/lib/firewalld/services/` (system defaults) and `/etc/firewalld/services/` (custom definitions).

Each service file defines:
- Port numbers and protocols (`tcp`/`udp`).
- Kernel helper modules (e.g., connection tracking for FTP or SIP).
- Destination addresses (IPv4 / IPv6).

---

## 3. Runtime vs. Permanent Configuration

A cornerstone feature of `firewalld` is the strict separation between temporary and persistent states:

```
                      firewall-cmd --add-port=8080/tcp
                                     |
                                     v
                       +---------------------------+
                       |   Runtime Configuration   |  (Active immediately;
                       |       (In-Memory)         |   lost upon reboot/reload)
                       +-------------+-------------+
                                     |
    firewall-cmd --runtime-to-permanent  OR  --permanent flag used originally
                                     |
                                     v
                       +---------------------------+
                       |  Permanent Configuration  |  (Written to XML in
                       |   (/etc/firewalld/zones)  |   persists across reboots)
                       +---------------------------+
```

1. **Runtime Configuration (`default`)**:
   - Active immediately in the running kernel.
   - Discarded when the system reboots or when running `firewall-cmd --reload`.
   - Essential for testing risky rules without locking out remote SSH sessions permanently.

2. **Permanent Configuration (`--permanent`)**:
   - Saved directly to XML files in `/etc/firewalld/`.
   - Does **not** take effect immediately in the runtime environment until `firewall-cmd --reload` is invoked.

> [!TIP]
> The recommended DevOps workflow: Apply changes to runtime first to verify connectivity. Once confirmed working, execute `firewall-cmd --runtime-to-permanent` to commit the active state to disk.

---

## 4. Configuration Storage Architecture

| Location | Purpose | Writable By User? |
| :--- | :--- | :--- |
| `/usr/lib/firewalld/` | Upstream package defaults (zones, services, icmptypes). | **No** (managed by RPM package manager). |
| `/etc/firewalld/` | System-specific custom configurations and overrides. | **Yes** (modified by `firewall-cmd` or admin). |
| `/etc/firewalld/firewalld.conf` | Core daemon settings (DefaultZone, FirewallBackend). | **Yes**. |

Whenever a zone or service is modified in `/etc/firewalld/`, the file in `/etc/firewalld/` overrides the identically named template in `/usr/lib/firewalld/`.

---

## 5. Summary Takeaways

- `firewalld` provides dynamic, non-disruptive firewall rule updates via D-Bus without tearing down live network connections.
- Incoming network traffic is routed into **Zones** based on incoming interface or source IP address.
- Rules can be applied using named **Services** or raw port/protocol pairs.
- Changes are split between ephemeral **Runtime** state and persistent **Permanent** disk storage.
