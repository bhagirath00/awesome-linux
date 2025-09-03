# `iptables` Fundamentals: Tables, Chains, Targets, and Rule Evaluation

## Overview

The `iptables` utility is the traditional, low-level user-space command-line interface used to configure the Linux kernel's Netfilter packet filtering rules.

Even though modern distributions use high-level frontends (`ufw`, `firewalld`), **understanding `iptables` is mandatory for DevOps and Kubernetes engineers**. Docker and Kubernetes CNI plugins (Kube-Proxy in iptables mode, Calico, Flannel) directly inject thousands of raw `iptables` rules into the kernel to manage service cluster IPs, port mappings, and pod egress NAT.

---

## 1. The Core Architecture: Tables, Chains, and Targets

An `iptables` configuration is organized into a strict three-tier hierarchy:

```text
TABLES (The Purpose)
  └── CHAINS (The Timing / Hook Point)
        └── RULES & TARGETS (The Action to Take)
```

```text
+-------------------------------------------------------------------------------+
|                       IPTABLES TABLES AND THEIR CHAINS                        |
+-------------------------------------------------------------------------------+
|  TABLE: filter (Default)      CHAINS: INPUT, FORWARD, OUTPUT                  |
|  - Purpose: Standard packet filtering (Allow or Block).                      |
|                                                                               |
|  TABLE: nat                   CHAINS: PREROUTING, POSTROUTING, OUTPUT         |
|  - Purpose: Network Address Translation, Port Forwarding, Masquerading.       |
|                                                                               |
|  TABLE: mangle                CHAINS: PREROUTING, INPUT, FORWARD, ...         |
|  - Purpose: Packet header modification (TTL adjustment, QoS Type of Service). |
|                                                                               |
|  TABLE: raw                   CHAINS: PREROUTING, OUTPUT                      |
|  - Purpose: Bypassing connection tracking (NOTRACK) for high-speed traffic.   |
+-------------------------------------------------------------------------------+
```

---

## 2. The Built-in Chains of the `filter` Table

When you invoke `iptables` without specifying a table (`-t`), it defaults to the **`filter` table**:

1. **`INPUT` Chain**: Evaluates packets destined **for this local server** (e.g., an incoming SSH connection or HTTP request).
2. **`OUTPUT` Chain**: Evaluates packets created **by this local server** heading out to the network.
3. **`FORWARD` Chain**: Evaluates packets routed **through this server** to another destination (used by Docker container bridges and Kubernetes nodes).

---

## 3. The Rule Targets: What Happens on a Match?

When a packet satisfies all criteria of a rule, `iptables` executes a **Target** action:

| Target | Description | Operational Impact |
| :--- | :--- | :--- |
| **`ACCEPT`** | Permits the packet to proceed through the network stack. | Traffic allowed. |
| **`DROP`** | **Silently discards the packet**. No response is sent back to the sender. | Port appears `filtered` in port scans. Attackers hang waiting for timeout. |
| **`REJECT`** | Drops the packet, but actively transmits an ICMP Unreachable or TCP RST. | Port appears `closed`. Informs sender immediately that traffic was blocked. |
| **`LOG`** | Logs packet metadata to `dmesg` / syslog, then **continues evaluating subsequent rules**. | Auditing denied packets. |
| **`MASQUERADE`**| Replaces source IP with outgoing interface IP (Dynamic NAT). | Sharing host internet with Docker containers. |
| **`DNAT`** | Rewrites destination IP/port (Port forwarding). | Mapping host port 8080 to container port 80. |

---

## 4. Fundamental Command Operations

```bash
sudo iptables [OPTIONS] [CHAIN] [CRITERIA] -j TARGET
```

### Essential Management Flags

| Flag | Meaning | Example |
| :--- | :--- | :--- |
| **`-L -n -v`** | **List rules** numerically with packet and byte counters. | `sudo iptables -L -n -v` |
| **`-P`** | Sets the **Default Policy** for a chain (`ACCEPT` or `DROP`). | `sudo iptables -P FORWARD DROP` |
| **`-A`** | **Appends** a new rule to the **bottom** of the chain. | `sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT` |
| **`-I`** | **Inserts** a rule at a specific position (default: line 1 at top).| `sudo iptables -I INPUT 1 -p tcp --dport 443 -j ACCEPT` |
| **`-D`** | **Deletes** a rule by chain name and line number. | `sudo iptables -D INPUT 3` |
| **`-F`** | **Flushes (deletes) all rules** in a chain or table. | `sudo iptables -F` |
| **`-Z`** | Zeroes packet and byte hit counters. | `sudo iptables -Z` |

---

## 5. The Critical Rule: Top-to-Bottom Sequential Evaluation

Rules within an `iptables` chain are evaluated in strict **sequential order from top to bottom**:

```text
Incoming Packet
       │
       ▼
Rule 1: Is packet on Port 22? ──► YES ──► ACCEPT (Evaluation STOPS immediately!)
       │
       NO
       ▼
Rule 2: Is packet on Port 80? ──► NO
       │
       ▼
Rule 3: DROP EVERYTHING!     ──► MATCHES ──► Packet is DROPPED!
```

> [!WARNING]
> **The First-Match Wins Rule**:
> As soon as a packet matches a terminating rule (`ACCEPT`, `DROP`, `REJECT`), evaluation stops. If you place a broad `DROP` rule at Line 1, all subsequent `ACCEPT` rules below it will **NEVER be evaluated!**
> Always use `iptables -L --line-numbers` to verify rule ordering.

---

## Summary

- `iptables` manages the kernel Netfilter framework across tables (`filter`, `nat`, `mangle`).
- The `filter` table processes local packets via `INPUT`, `OUTPUT`, and `FORWARD` chains.
- Core targets include `ACCEPT`, `DROP` (silent discard), `REJECT` (active reset), and `LOG`.
- Rules are evaluated sequentially from top to bottom; the first matching rule terminates evaluation.
- Inspect active rules using the standard command: `sudo iptables -L -n -v --line-numbers`.
