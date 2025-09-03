# Chapter 19: Firewall Management - UFW Commands

## 1. Initial Setup and Enabling UFW

Follow this exact sequence when initializing a newly provisioned Ubuntu or Debian server:

```bash
# 1. Check current status
sudo ufw status

# 2. Set secure default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# 3. CRITICAL: Allow SSH before enabling
sudo ufw allow 22/tcp comment 'Standard SSH'

# 4. Enable the firewall
sudo ufw enable
```

```
Command may disrupt existing ssh connections. Proceed with operation (y|n)? y
Firewall is active and enabled on system startup
```

---

## 2. Managing Ports and Protocol Rules

### Opening Ports
```bash
# Allow HTTP (port 80) and HTTPS (port 443)
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Allow a port range for passive FTP or WebRTC
sudo ufw allow 60000:61000/udp
```

### Allowing by Service / Application Name
```bash
# Allow OpenSSH profile
sudo ufw allow OpenSSH

# Allow Nginx Full (both HTTP and HTTPS)
sudo ufw allow 'Nginx Full'
```

### Denying and Rejecting Traffic
- `deny`: Drops the packet silently (attacker experiences a timeout).
- `reject`: Drops the packet and sends an ICMP port-unreachable response (reveals host exists).

```bash
# Drop all inbound traffic on database port
sudo ufw deny 3306/tcp comment 'Block MySQL from public'
```

---

## 3. IP and Subnet Filtering

You can restrict sensitive administrative ports (databases, internal APIs, management UIs) to specific trusted IP addresses or subnets:

```bash
# Allow PostgreSQL (5432) ONLY from backend app server (192.168.1.150)
sudo ufw allow from 192.168.1.150 to any port 5432 proto tcp

# Allow all traffic from internal VPN subnet (10.8.0.0/24)
sudo ufw allow from 10.8.0.0/24

# Allow a specific management workstation to access SSH
sudo ufw allow from 203.0.113.45 to any port 22 proto tcp
```

---

## 4. Rate Limiting with UFW

UFW includes a built-in rate-limiting mechanism specifically designed to thwart brute-force password guessing attacks (e.g., against SSH):

```bash
sudo ufw limit ssh
```

> [!NOTE]
> `ufw limit` permits incoming connections on port 22, but if an IP address initiates **6 or more connections within a 30-second window**, UFW automatically blocks that IP address.

---

## 5. Inspecting and Deleting Rules

Always view rules using numbered mode when preparing to delete or rearrange:

```bash
sudo ufw status numbered
```

```
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 22/tcp                     ALLOW IN    Anywhere                  # Standard SSH
[ 2] 80/tcp                     ALLOW IN    Anywhere                 
[ 3] 443/tcp                    ALLOW IN    Anywhere                 
[ 4] 5432/tcp                   ALLOW IN    192.168.1.150            
[ 5] 22/tcp (v6)                ALLOW IN    Anywhere (v6)             # Standard SSH
[ 6] 80/tcp (v6)                ALLOW IN    Anywhere (v6)            
[ 7] 443/tcp (v6)               ALLOW IN    Anywhere (v6)            
```

### Deleting Rules
```bash
# Delete rule by its exact specification
sudo ufw delete allow 80/tcp

# Delete rule by its index number (safest method)
sudo ufw delete 4
```

---

## 6. Logging and Resetting UFW

```bash
# Configure logging levels (off, low, medium, high, full)
sudo ufw logging medium

# View UFW kernel logs in real-time
sudo tail -f /var/log/ufw.log
# OR via systemd journal
sudo journalctl -u ufw.service -f

# Reset UFW to factory defaults (wipes all custom rules)
sudo ufw reset
```

---

## 7. Command Cheat Sheet

| Command | Function |
| :--- | :--- |
| `ufw status verbose` | Check detailed firewall status and active policy. |
| `ufw status numbered` | Show all active rules with numbered indices. |
| `ufw enable / disable` | Turn firewall on or off. |
| `ufw default deny incoming` | Enforce drop policy on all unsolicited inbound traffic. |
| `ufw allow <port>/<proto>` | Permit inbound port (e.g., `ufw allow 8080/tcp`). |
| `ufw allow from <IP> to any port <port>` | Whitelist specific IP to a dedicated port. |
| `ufw limit <port>/<proto>` | Protect service with connection rate limiting. |
| `ufw delete <number>` | Remove rule by index number. |
| `ufw reload` | Reload firewall rules without resetting connection state. |

---

## 8. Summary Takeaways

- Always allow SSH before executing `ufw enable`.
- Use `ufw limit ssh` to protect remote access from brute-force botnets.
- Use numbered listings (`ufw status numbered`) to safely delete obsolete rules by index.
- Restrict sensitive ports like databases and caches (`5432`, `3306`, `6379`) to explicit internal IPs using `ufw allow from <IP>`.
