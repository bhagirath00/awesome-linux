# Chapter 20: SSH - Key-Based Authentication

## 1. Password Authentication vs. Public Key Authentication

While standard passwords can be guessed, brute-forced, or intercepted via keyloggers, **Key-Based Authentication** uses asymmetric cryptography to eliminate password transmission across the network entirely.

| Metric | Password Authentication | Public Key Authentication |
| :--- | :--- | :--- |
| **Security** | Susceptible to dictionary & brute-force attacks | Virtually immune to brute force (256-bit ECC / 4096-bit RSA) |
| **Automation** | Requires cleartext passwords or interactive prompts | Easily integrated into CI/CD pipelines, Ansible, Terraform |
| **Storage** | Password stored as salted hash in `/etc/shadow` | Public key stored on server; private key kept on local client |
| **Revocation** | Password reset changes credentials for all access | Individual keys can be revoked by removing a single line |

---

## 2. Asymmetric Cryptography Workflow in SSH

```
[ CLIENT WORKSTATION ]                              [ TARGET LINUX SERVER ]
+-------------------------+                         +-------------------------+
| Private Key: id_ed25519 |                         | Public Key Stored In:   |
| (NEVER SHARES THIS FILE)|                         | ~/.ssh/authorized_keys  |
+------------+------------+                         +------------+------------+
             |                                                   |
             | 1. Client requests login: "I want to login as devops"
             |-------------------------------------------------->|
             |                                                   |
             | 2. Server generates random nonce challenge (R)    |
             |    Server encrypts R using devops's Public Key    |
             |<--------------------------------------------------|
             |                                                   |
             | 3. Client decrypts R using local Private Key,     |
             |    hashes R with session ID, and sends signature  |
             |-------------------------------------------------->|
             |                                                   |
             | 4. Server verifies signature with Public Key      |
             |    Verification SUCCESS -> Shell granted!         |
             |<--------------------------------------------------|
```

---

## 3. Key Algorithms: Ed25519 vs. RSA

When generating an SSH key pair today, choosing the right cryptographic algorithm is paramount:

| Algorithm | Key Length | Performance | Recommendation Status | Notes |
| :--- | :--- | :--- | :--- | :--- |
| **Ed25519** | 256 bits | Extremely Fast | **Top Recommendation** | Modern Edwards-curve Digital Signature Algorithm; compact, collision-resistant, constant-time execution prevents side-channel attacks. |
| **RSA** | 4096 bits | Moderate | **Legacy Fallback** | Use only when legacy systems lack Ed25519 support. Keys under 2048 bits are considered cryptographically broken. |
| **ECDSA** | 256/384/521 bits | Fast | **Avoid** | Uses NIST standard curves with theoretical backdoor and poor implementation concerns (Sony PS3 key leak was caused by poor random number generator in ECDSA). |
| **DSA** | 1024 bits | Slow | **Deprecated / Forbidden** | Dropped by OpenSSH upstream; insecure. |

---

## 4. Generating Keys with ssh-keygen

### Recommended Modern Generation (Ed25519):
```bash
ssh-keygen -t ed25519 -C "devops-engineer@enterprise.internal"
```

```
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/admin/.ssh/id_ed25519): 
Enter passphrase (empty for no passphrase): **********
Enter same passphrase again: **********
Your identification has been saved in /home/admin/.ssh/id_ed25519
Your public key has been saved in /home/admin/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:d8N7jKx5O+U3G7zJ3nF5p8s0M2wK6L9y1A4b7C0d9E devops-engineer@enterprise.internal
The key's randomart image is:
+--[ED25519 256]--+
|      .o.        |
|     .o.o .      |
|    .. o =       |
|    . + O .      |
|   . = S *       |
|  . o * O .      |
|   + o B o       |
|  . = =.+ .      |
|   . oE+.        |
+----[SHA256]-----+
```

> [!TIP]
> Always protect your private key with a strong passphrase. If a laptop or backup drive is lost or compromised, the passphrase prevents an attacker from immediately using the key.

---

## 5. Deploying Public Keys to Remote Servers

### Method 1: Using `ssh-copy-id` (Recommended)
`ssh-copy-id` automatically copies your public key, creates `~/.ssh/authorized_keys` if missing, and configures the strictly required POSIX permissions:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub devops@192.168.1.100
```

### Method 2: Manual Installation via SSH Pipe
If `ssh-copy-id` is unavailable:
```bash
cat ~/.ssh/id_ed25519.pub | ssh devops@192.168.1.100 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

---

## 6. Strict File and Directory Permission Requirements

OpenSSH enforces strict permission checking (`StrictModes yes`). If permissions on the remote server's home directory, `.ssh` folder, or `authorized_keys` file are writable by group or others, OpenSSH will silently reject the key and force a password prompt!

| Path | Required Octal Mode | Symbolic Representation | Rationale |
| :--- | :--- | :--- | :--- |
| `/home/username/` | `755` or `700` | `drwxr-xr-x` / `drwx------` | Group or world must **never** have write permissions. |
| `~/.ssh/` | `700` | `drwx------` | Only owner can read, write, and execute. |
| `~/.ssh/authorized_keys` | `600` | `-rw-------` | Only owner can read and write. |
| `~/.ssh/id_ed25519` (Private) | `600` | `-rw-------` | Private key must never be readable by anyone else. |
| `~/.ssh/id_ed25519.pub` | `644` | `-rw-r--r--` | Public key can be safely readable. |

Fixing broken permissions in one command:
```bash
chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys
```

---

## 7. Summary Takeaways

- Ed25519 (`-t ed25519`) is the modern gold standard for SSH keys.
- Never share, copy, or expose the private key file (`id_ed25519`).
- Only copy the public key (`.pub`) to `~/.ssh/authorized_keys`.
- Permission mistakes (`chmod 777`) will break key-based authentication due to OpenSSH `StrictModes`.
