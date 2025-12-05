# Chapter 20: SSH - SSH Server Hardening and Best Practices

## 1. Why SSH Hardening is Critical

Because TCP port 22 is open on almost every Linux server worldwide, automated scanning botnets constantly bombard internet-facing servers with hundreds of thousands of credential-stuffing and dictionary attacks every day.

Hardening OpenSSH according to CIS (Center for Internet Security) Benchmarks and NIST guidelines dramatically reduces your attack surface and eliminates unauthorized access risks.

---

## 2. Essential Hardening Directives (/etc/ssh/sshd_config)

All changes are configured in `/etc/ssh/sshd_config` or inside drop-in configuration snippets in `/etc/ssh/sshd_config.d/99-hardening.conf`.

### 1. Disable Root Login Completely
Never allow anyone to log in directly as `root`. Force users to log in with their own named user account and escalate privileges using `sudo`:
```sshconfig
PermitRootLogin no
```

### 2. Disable Password Authentication (Enforce Keys Only)
Passwords can be guessed; 256-bit cryptographic keys cannot.
```sshconfig
PasswordAuthentication no
ChallengeResponseAuthentication no
KbdInteractiveAuthentication no
```

> [!WARNING]
> Ensure your public key is added to `~/.ssh/authorized_keys` and tested before disabling `PasswordAuthentication`! Keep an active SSH session open while testing in a new terminal window.

### 3. Change the Default Port (Optional / Defense-in-Depth)
Moving SSH to a non-standard port (e.g., `2222` or `22022`) eliminates 99% of noisy automated bots from spamming your system auth logs:
```sshconfig
Port 22022
```

### 4. Limit Authentication Attempts and Idle Timeouts
Disconnect brute-force bots quickly and log out stale sessions automatically:
```sshconfig
# Limit max password/key attempts per connection to 3
MaxAuthTries 3

# Close unauthenticated connection attempts after 30 seconds
LoginGraceTime 30

# Disconnect idle sessions after 15 minutes of inactivity (300s x 3)
ClientAliveInterval 300
ClientAliveCountMax 3
```

### 5. Whitelist Allowed Users or Groups
Restrict SSH access strictly to authorized team members:
```sshconfig
AllowUsers deployer devops-lead
# OR restrict to a system group
AllowGroups ssh-users
```

### 6. Disable Insecure Features
```sshconfig
# Disable legacy X11 graphical forwarding
X11Forwarding no

# Ignore legacy .rhosts authentication
IgnoreRhosts yes

# Do not permit empty passwords under any circumstance
PermitEmptyPasswords no
```

---

## 3. Strong Modern Cryptographic Ciphers

Deprecate legacy 3DES, Blowfish, and SHA-1 algorithms by enforcing modern AEAD ciphers, robust Key Exchange (KEX) algorithms, and HMACs:

```sshconfig
# Enforce strong Key Exchange Algorithms
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512

# Enforce AEAD Symmetric Ciphers
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com

# Enforce secure MAC algorithms
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
```

---

## 4. Applying and Validating Hardening

Always test configuration syntax before reloading the daemon:

```bash
# 1. Test configuration syntax
sudo sshd -t

# 2. If no errors are returned, reload sshd
sudo systemctl reload sshd

# 3. Verify socket is listening on new port
sudo ss -tulpn | grep sshd
```

```
tcp   LISTEN 0      128          0.0.0.0:22022      0.0.0.0:*    users:(("sshd",pid=1452,fd=3))
tcp   LISTEN 0      128             [::]:22022         [::]:*    users:(("sshd",pid=1452,fd=4))
```

---

## 5. Security Checklist Summary

| Directive | Recommended Setting | Security Benefit |
| :--- | :--- | :--- |
| `PermitRootLogin` | `no` | Prevents direct root credential attacks. |
| `PasswordAuthentication` | `no` | Eliminates brute-force password guessing. |
| `Port` | High port (e.g., `22022`) | Cuts automated bot log noise. |
| `MaxAuthTries` | `3` | Thwarts automated credential guessing. |
| `ClientAliveInterval` | `300` | Terminate orphaned/idle connections. |
| `AllowGroups` | `ssh-users` | Implements principle of least privilege. |
| `X11Forwarding` | `no` | Reduces attack surface. |

---

## 6. Summary Takeaways

- Disabling root logins and enforcing SSH key authentication are the two most critical steps in securing Linux systems.
- Always validate changes with `sshd -t` prior to reloading the service.
- Restrict SSH access to explicit users or groups using `AllowUsers` or `AllowGroups`.
- Keep an existing terminal session open while testing newly hardened configurations in a parallel window.
