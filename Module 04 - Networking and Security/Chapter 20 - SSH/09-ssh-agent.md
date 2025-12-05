# Chapter 20: SSH - SSH Agent and Agent Forwarding

## 1. What is ssh-agent?

`ssh-agent` is a background key management daemon included with OpenSSH that holds decrypted private keys in memory. 

When your private key is protected by a strong passphrase, entering that passphrase every time you initiate an SSH connection, run an Ansible playbook, or push Git commits becomes cumbersome. `ssh-agent` prompts you for your passphrase **once**, caches the decrypted private key in RAM, and signs authentication challenges on behalf of client programs for the duration of your session without ever writing the unencrypted key to disk.

---

## 2. Using ssh-agent and ssh-add

### 1. Starting the Agent
On modern Linux desktop environments and macOS, `ssh-agent` starts automatically upon login. In headless terminal sessions or remote containers, start it manually:
```bash
eval "$(ssh-agent -s)"
```

```
Agent pid 34182
```

This command initializes the background process and sets the necessary environment variables (`SSH_AUTH_SOCK` and `SSH_AGENT_PID`).

### 2. Adding Keys with `ssh-add`
```bash
# Add default identity key (~/.ssh/id_ed25519)
ssh-add

# Add a specific key with a lifetime timeout (e.g., 8 hours / 28800 seconds)
ssh-add -t 8h ~/.ssh/id_ed25519_prod
```

```
Enter passphrase for /home/admin/.ssh/id_ed25519_prod: **********
Identity added: /home/admin/.ssh/id_ed25519_prod (devops@enterprise)
Lifetime set to 28800 seconds
```

### 3. Listing and Managing Cached Keys
```bash
# List fingerprints of all currently cached keys
ssh-add -l

# List full public key strings in OpenSSH format
ssh-add -L

# Remove a specific key from agent memory
ssh-add -d ~/.ssh/id_ed25519_prod

# Flush all keys from agent memory immediately
ssh-add -D
```

---

## 3. SSH Agent Forwarding (`ssh -A`)

### The Use Case
Consider an engineer who needs to log into a jump server / bastion (`Host A`), and from there access an internal Git server or private database node (`Host B`). 

Instead of copying private keys onto `Host A` (which is a severe security violation), **Agent Forwarding** allows `Host A` to reach back through the encrypted tunnel to the engineer's local `ssh-agent` to sign authentication requests for `Host B`:

```
+-------------------+            +-------------------+            +-------------------+
| Engineer Laptop   |            | Bastion (Host A)  |            | Private Node (B)  |
| ssh-agent (Key in | =====>     | (NO private keys  | =====>     | authorized_keys   |
| memory)           |            | stored on disk)   |            | accepts Key       |
+---------^---------+            +---------+---------+            +-------------------+
          |                                |
          +--- Signed challenge forwarded -+
```

### Running Agent Forwarding:
```bash
ssh -A devops@bastion.company.com
```

---

## 4. Security Risks of Agent Forwarding

> [!CAUTION]
> **Never enable agent forwarding to untrusted servers!**
> 
> When you forward your agent to a remote server, SSH creates a Unix domain socket on the remote machine (e.g., `/tmp/ssh-XXXXXX/agent.XXXX`). Anyone with `root` privileges on that remote server can hijack that socket and authenticate to any other server or repository your key has access to for as long as your connection remains open!

### The Modern Alternative: `ProxyJump`
For multi-hop connections, **do not forward your agent**. Instead, use `ProxyJump` (or the `-J` command flag), which keeps all SSH authentication strictly local to your machine:

```bash
# Direct command line syntax
ssh -J devops@bastion.company.com devops@private-internal-node

# Or via ~/.ssh/config:
# Host private-node
#     ProxyJump bastion.company.com
```

---

## 5. Summary Takeaways

- `ssh-agent` caches decrypted private keys in RAM, saving repeated passphrase entries.
- Use `ssh-add -t <duration>` to automatically expire cached keys after work hours.
- Agent Forwarding (`-A`) allows remote hosts to sign authentication challenges using your local key.
- Avoid Agent Forwarding whenever possible due to root-socket hijacking risks; use `ProxyJump` (`-J`) instead.
