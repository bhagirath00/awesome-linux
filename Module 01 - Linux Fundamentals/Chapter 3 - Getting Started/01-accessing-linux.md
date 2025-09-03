# Accessing Linux: SSH, Cloud Consoles, WSL2 & Containers

## Introduction

As a DevOps engineer, you will rarely sit in front of a physical Linux keyboard and monitor. More than 98% of your work involves accessing remote Linux instances hosted in AWS, GCP, Azure, bare-metal data centers, or local virtualized sandboxes.

---

## 1. The Five Methods of Accessing Linux

```text
+--------------------------------------------------------------------------+
|                        METHODS TO ACCESS LINUX                           |
+--------------------------------------------------------------------------+
| 1. SSH (Secure Shell)       | Industry standard for remote servers/cloud |
| 2. Cloud Web Consoles       | Browser emergency consoles (AWS, GCP)      |
| 3. WSL2 (Windows)           | Native Linux environment on Windows laptop |
| 4. Docker Container         | Instant zero-overhead sandbox for testing  |
| 5. Virtual Machines         | Isolated hypervisor guests (Multipass/VBox)|
+--------------------------------------------------------------------------+
```

---

## 2. SSH: The Industry Standard (Port 22)

**SSH (Secure Shell)** encrypts all communication between your local machine and the remote server.

### Password vs SSH Key-Pair Authentication
In enterprise DevOps, **password authentication over SSH is disabled** because it is vulnerable to brute-force attacks. Instead, authentication uses **Asymmetric Cryptographic Key Pairs**:
- **Private Key (`id_ed25519`)**: Kept strictly secret on your local laptop.
- **Public Key (`id_ed25519.pub`)**: Placed on the remote server in `~/.ssh/authorized_keys`.

```text
Local Machine (DevOps Laptop)                      Remote Linux Server
+---------------------------+                      +---------------------------+
| Private Key: id_ed25519   |                      | ~/.ssh/authorized_keys    |
| (NEVER SHARED)            |                      | (Contains your public key)|
+---------------------------+                      +---------------------------+
              \                                                 /
               \------------ Encrypted Challenge --------------/
                                  (Verified)
```

### Step-by-Step: Setting Up SSH Keys

```bash
# 1. Generate a modern, highly secure Ed25519 SSH key pair
ssh-keygen -t ed25519 -C "your_email@example.com"
# Prompts for file location (press Enter for default: ~/.ssh/id_ed25519)
# Prompts for optional passphrase (recommended for production keys)

# 2. Copy your public key to the remote server automatically
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@192.168.1.100

# 3. Connect to the server
ssh -i ~/.ssh/id_ed25519 ubuntu@192.168.1.100
```

---

## 3. Productivity Pro-Tip: The SSH Config File (`~/.ssh/config`)

Instead of typing long IP addresses, usernames, and key paths every time, configure shortcuts in `~/.ssh/config`:

```text
# Open ~/.ssh/config on your laptop
Host prod-web
    HostName 34.201.88.12
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519
    Port 22

Host k8s-master
    HostName 10.0.1.50
    User devops
    IdentityFile ~/.ssh/k8s_key
```

Now you can connect instantly by typing:
```bash
ssh prod-web
```

---

## 4. Instant Disposable Linux via Docker

If you need a completely clean, disposable Linux environment to test commands, packages, or scripts without polluting your local system, run an ephemeral container:

```bash
# Spin up an interactive Ubuntu container that deletes itself on exit (--rm)
docker run --rm -it ubuntu:24.04 bash

# Spin up an ultra-minimal Alpine Linux container
docker run --rm -it alpine:latest sh
```

---

## 5. Lightweight Linux VMs via Canonical Multipass

If you want a dedicated Ubuntu VM with its own kernel, systemd init, and IP address for local testing:

```bash
# Launch a new Ubuntu VM named 'dev-box'
multipass launch --name dev-box --cpus 2 --memory 2G --disk 10G

# Open a shell directly inside the VM
multipass shell dev-box

# Stop and cleanup
multipass delete dev-box && multipass purge
```
