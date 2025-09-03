# Chapter 19: Firewall Management - firewalld Commands

## 1. Introduction to firewall-cmd

`firewall-cmd` is the primary command-line client used to query, modify, and monitor the dynamic `firewalld` daemon. All administrative changes—including assigning interfaces to zones, exposing ports, configuring NAT masquerading, and creating complex rich rules—are managed through this utility.

---

## 2. Managing the firewalld Service

Before issuing `firewall-cmd` commands, verify that the daemon is active and enabled via `systemctl`:

```bash
# Verify daemon status
sudo systemctl status firewalld

# Enable and start daemon on boot
sudo systemctl enable --now firewalld

# Query state via firewall-cmd
sudo firewall-cmd --state
```

```
running
```

---

## 3. Querying Zones and Active Rules

```bash
# Check the default zone
sudo firewall-cmd --get-default-zone
```

```
public
```

```bash
# List all active zones with their assigned interfaces
sudo firewall-cmd --get-active-zones
```

```
public
  interfaces: eth0
```

```bash
# Inspect all rules currently applied to the default zone
sudo firewall-cmd --list-all
```

```
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: eth0
  sources: 
  services: cockpit dhcpv6-client ssh
  ports: 
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
```

---

## 4. Opening and Closing Ports & Services

### Opening Services
Services automatically map to standard ports. For instance, `http` maps to `80/tcp` and `https` maps to `443/tcp`.

```bash
# Add HTTP service to runtime configuration
sudo firewall-cmd --add-service=http

# Add HTTPS permanently and reload to apply
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

### Opening Specific Custom Ports
Useful for modern web apps, databases, and microservices (e.g., Node.js on `3000/tcp`, Redis on `6379/tcp`).

```bash
# Allow TCP port 8080 permanently
sudo firewall-cmd --permanent --add-port=8080/tcp

# Allow UDP port range for media streaming / WebRTC
sudo firewall-cmd --permanent --add-port=10000-10050/udp

# Remove port access
sudo firewall-cmd --permanent --remove-port=8080/tcp

# Commit changes to runtime
sudo firewall-cmd --reload
```

---

## 5. Zone Management and Interface Assignment

You can bind specific network interfaces or subnets to dedicated zones:

```bash
# Move eth1 interface from public to internal zone permanently
sudo firewall-cmd --permanent --zone=internal --change-interface=eth1

# Assign an IP subnet (e.g., Office VPN) to the trusted zone
sudo firewall-cmd --permanent --zone=trusted --add-source=10.50.0.0/16

# Reload configuration
sudo firewall-cmd --reload
```

---

## 6. Advanced Port Forwarding and NAT Masquerading

`firewalld` makes port forwarding and network address translation simple without raw iptables chains:

### Enable Masquerading (NAT Gateway)
```bash
# Turn on IP masquerading for outward traffic
sudo firewall-cmd --permanent --zone=public --add-masquerade
```

### Local Port Redirection
Redirect incoming traffic on port 80 to an internal application running on port 8080:

```bash
sudo firewall-cmd --permanent --zone=public \
  --add-forward-port=port=80:proto=tcp:toport=8080
```

### Remote Port Forwarding
Forward incoming port 2222 on public IP to an internal host `192.168.1.100` on standard SSH port 22:

```bash
sudo firewall-cmd --permanent --zone=public \
  --add-forward-port=port=2222:proto=tcp:toport=22:toaddr=192.168.1.100
sudo firewall-cmd --reload
```

---

## 7. Rich Language Rules

Rich rules provide granular filtering capabilities combining source addresses, destination ports, protocols, rate limits, and custom audit logging:

```bash
# Allow SSH access ONLY from a trusted management workstation (192.168.1.50)
sudo firewall-cmd --permanent --zone=public --add-rich-rule='
  rule family="ipv4"
  source address="192.168.1.50"
  service name="ssh"
  accept'

# Drop all traffic from a malicious subnet with rate-limited kernel logging
sudo firewall-cmd --permanent --zone=public --add-rich-rule='
  rule family="ipv4"
  source address="198.51.100.0/24"
  log prefix="[ATTACKER_DROP] " level="warning" limit value="5/m"
  drop'

# Apply the rules
sudo firewall-cmd --reload
```

---

## 8. Command Cheat Sheet

| Command Syntax | Operation Description |
| :--- | :--- |
| `firewall-cmd --state` | Check if firewalld daemon is operational. |
| `firewall-cmd --get-active-zones` | Display currently bound interfaces and zones. |
| `firewall-cmd --list-all` | List all open services, ports, and rules in current zone. |
| `firewall-cmd --permanent --add-service=<svc>` | Add persistent service rule (e.g., `http`, `dns`). |
| `firewall-cmd --permanent --add-port=<port>/<proto>` | Open custom port (e.g., `9090/tcp`). |
| `firewall-cmd --reload` | Synchronize permanent configuration into runtime without terminating active connections. |
| `firewall-cmd --runtime-to-permanent` | Save active runtime state into permanent XML configuration files. |
| `firewall-cmd --panic-on` | Emergency blackout: drop ALL incoming and outgoing packets. |
| `firewall-cmd --panic-off` | Disable panic mode and restore standard filtering. |

---

## 9. Summary Takeaways

- `firewall-cmd` provides a unified interface for ports, services, zones, port forwarding, and rich rules.
- Always remember `--reload` when modifying with `--permanent`, or use `--runtime-to-permanent` after validating runtime rules.
- Rich rules give granular enterprise-grade control over source IPs, logging, and rate limiting.
