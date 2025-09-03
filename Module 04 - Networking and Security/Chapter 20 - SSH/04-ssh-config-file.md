# Chapter 20: SSH - SSH Client Configuration (~/.ssh/config)

## 1. Why Use ~/.ssh/config?

Typing long, tedious SSH commands in production is inefficient and error-prone:
```bash
ssh -i ~/.ssh/enterprise_prod_rsa -p 22022 deploy-user@ec2-54-210-45-12.compute-1.amazonaws.com
```

With the user-level SSH configuration file (`~/.ssh/config`), you can define short, human-friendly aliases for complex hosts, automate identity key selection, handle proxy jumps through bastion hosts, and configure keep-alive timeouts.

Instead of the above command, you simply run:
```bash
ssh prod-app
```

---

## 2. Anatomy and Syntax of ~/.ssh/config

The configuration file consists of sections starting with a `Host` header followed by indented parameter declarations:

```sshconfig
# Global settings applied to all connections
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    AddKeysToAgent yes
    IdentitiesOnly yes

# Standard Web Server Alias
Host web-prod
    HostName 198.51.100.15
    User ubuntu
    Port 22
    IdentityFile ~/.ssh/id_ed25519_prod

# Database Server behind custom port
Host db-primary
    HostName db.internal.company.com
    User postgres_admin
    Port 54322
    IdentityFile ~/.ssh/id_rsa_db
```

---

## 3. Key Configuration Directives Explained

| Directive | Description | Example |
| :--- | :--- | :--- |
| `Host` | Nickname or pattern matching the command line argument. | `Host dev-*` |
| `HostName` | The actual FQDN or IP address of the target server. | `HostName 10.0.1.25` |
| `User` | The remote login account username. | `User ec2-user` |
| `Port` | The SSH daemon TCP listening port on the target. | `Port 2222` |
| `IdentityFile` | Explicit path to the private key for this host. | `IdentityFile ~/.ssh/aws_key` |
| `IdentitiesOnly` | Ensures SSH *only* uses keys listed in config (prevents "Too many authentication failures"). | `IdentitiesOnly yes` |
| `ServerAliveInterval` | Seconds between keepalive packets sent to prevent router timeout drops. | `ServerAliveInterval 30` |
| `ProxyJump` | Bastion / Jump host used to reach an isolated private subnet. | `ProxyJump bastion` |

---

## 4. Bastion / Jump Host Architecture (ProxyJump)

In cloud environments (AWS VPC, GCP, Azure), internal database and worker nodes are kept inside private subnets without public IP addresses. To access them, engineers route traffic through a hardened **Bastion / Jump Host**:

```
+------------------+          +-------------------+          +-------------------+
|  Engineer Laptop | =======> |   Bastion Host    | =======> | Private DB Node   |
| (Public Network) |          | (Public IP in DMZ)|          | (10.0.2.45 - Priv)|
+------------------+          +-------------------+          +-------------------+
```

With `~/.ssh/config`, this complex proxy chaining is completely automated using `ProxyJump`:

```sshconfig
# The Public-Facing Bastion
Host bastion
    HostName bastion.company.com
    User jumpuser
    IdentityFile ~/.ssh/id_ed25519_bastion

# Private Database Node accessed THROUGH Bastion
Host internal-db
    HostName 10.0.2.45
    User dbadmin
    IdentityFile ~/.ssh/id_ed25519_db
    ProxyJump bastion
```

Now, typing `ssh internal-db` automatically establishes the tunnel through `bastion` transparently in one single step without running multi-hop commands!

---

## 5. File Permissions

The `~/.ssh/config` file must have strict permissions:
```bash
chmod 600 ~/.ssh/config
```

---

## 6. Summary Takeaways

- `~/.ssh/config` simplifies complex hostnames, custom ports, and multi-identity SSH setups into simple aliases.
- Use `ProxyJump` for automated, seamless access through cloud bastions into private VPC subnets.
- Use `IdentitiesOnly yes` to prevent SSH from cycling through all keys in `ssh-agent`.
- Always set `chmod 600 ~/.ssh/config`.
