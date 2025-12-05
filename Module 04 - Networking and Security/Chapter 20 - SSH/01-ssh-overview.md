# Chapter 20: SSH - SSH Overview and Architecture

## 1. Introduction: What is Secure Shell (SSH)?

**SSH** (Secure Shell) is an encrypted network communication protocol operating at the Application Layer (default port **22/TCP**) that enables secure remote terminal logins, remote command execution, automated file transfers, and encrypted network tunneling over untrusted networks.

Before SSH was designed by Tatu Ylönen in 1995, systems administrators relied on legacy cleartext protocols such as **Telnet**, **Rlogin**, and **FTP**. In cleartext protocols, usernames, passwords, and sensitive session keystrokes are transmitted in plaintext across network switches and routers, exposing systems to packet sniffing (eavesdropping) and Man-in-the-Middle (MitM) replay attacks.

```
+--------------------------------------------------------------------------+
|                       Telnet vs SSH Communication                        |
+--------------------------------------------------------------------------+
|  Telnet / Rlogin (Insecure - Plaintext):                                 |
|  [ Client ] ----"user: root | pass: Secret123"----> [ Network Switch ]   |
|                                                            |             |
|                                              (Sniffed by Hacker)         |
|                                                            v             |
|                                                     [ Server Compromised]|
+--------------------------------------------------------------------------+
|  SSH - Secure Shell (Encrypted Tunnel):                                  |
|  [ Client ] ====[ AES-256-GCM Encrypted / HMAC-SHA256 ]====> [ Server ]  |
|                                                            |             |
|                                              (Sniffer sees cipher noise) |
|                                                            v             |
|                                                   [ Connection Protected]|
+--------------------------------------------------------------------------+
```

---

## 2. Cryptographic Foundations of SSH

SSH combines three foundational cryptographic mechanisms to establish a tamper-proof session:

```
                +-------------------------------------------------+
                |             SSH Cryptography Stack              |
                +-------------------------------------------------+
                | 1. Asymmetric Cryptography (Public/Private Key) |
                |    - Server Host Authentication                 |
                |    - User Authentication (RSA, Ed25519)         |
                +------------------------+------------------------+
                                         |
                                         v
                +-------------------------------------------------+
                | 2. Diffie-Hellman Key Exchange (ECDH / Curve25519)|
                |    - Generates a shared temporary session key   |
                |    - Key is NEVER transmitted across wire       |
                +------------------------+------------------------+
                                         |
                                         v
                +-------------------------------------------------+
                | 3. Symmetric Cryptography (ChaCha20, AES-GCM)   |
                |    - High-speed payload encryption              |
                |    - One shared session key for data transfer   |
                +------------------------+------------------------+
                                         |
                                         v
                +-------------------------------------------------+
                | 4. Cryptographic Hashing / MAC (HMAC-SHA2-512)  |
                |    - Data integrity & anti-tampering validation |
                +-------------------------------------------------+
```

### 1. Asymmetric Cryptography (Public/Private Keys)
Used for **identity verification**:
- **Host Keys**: The server proves its identity to the client to prevent DNS spoofing and MitM attacks.
- **User Authentication**: The client proves their identity using public/private key pairs without ever sending a password.

### 2. Diffie-Hellman Key Exchange
A mathematical algorithm that allows the client and server to negotiate a shared temporary secret key over an open, untrusted network without either party actually transmitting the key itself.

### 3. Symmetric Encryption
Once the shared secret is established, bulk session data (terminal output, keystrokes, files) is encrypted using fast, hardware-accelerated symmetric ciphers (e.g., `aes256-gcm` or `chacha20-poly1305`).

### 4. Message Integrity (MAC)
Every packet contains a Hash-based Message Authentication Code (HMAC) to ensure that transmitted packets have not been intercepted, modified, or replayed by an attacker in transit.

---

## 3. SSH Protocol Layers

The SSH-2 protocol (defined in RFC 4251) is structured into three distinct layers:

| Layer | RFC | Responsibility |
| :--- | :--- | :--- |
| **Transport Layer** | RFC 4253 | Handles server authentication, confidentiality (ciphers), integrity checks, and key exchange. |
| **User Authentication Layer** | RFC 4252 | Authenticates the human user or service account (via `publickey`, `password`, or `gssapi-with-mic`). |
| **Connection Layer** | RFC 4254 | Multiplexes the single encrypted transport pipe into multiple logical channels (interactive shells, SCP/SFTP transfers, TCP port forwarding). |

---

## 4. SSH Connection Lifecycle Walkthrough

```
CLIENT                                                           SERVER
  |                                                                |
  | 1. TCP Handshake (SYN -> SYN-ACK -> ACK on Port 22)           |
  |--------------------------------------------------------------->|
  |                                                                |
  | 2. Protocol & Version Exchange ("SSH-2.0-OpenSSH_9.3")        |
  |<==============================================================>|
  |                                                                |
  | 3. Key Exchange (KEX) Negotiation                             |
  |    - Choose KEX algorithm (curve25519-sha256)                  |
  |    - Choose symmetric cipher (chacha20-poly1305)               |
  |    - Diffie-Hellman exchange produces Shared Secret Key (K)    |
  |<==============================================================>|
  |                                                                |
  | 4. Server Host Identity Check                                  |
  |    - Server signs hash with host private key                   |
  |    - Client checks ~/.ssh/known_hosts                          |
  |<---------------------------------------------------------------|
  |                                                                |
  | [ ENCRYPTED CHANNEL ESTABLISHED ]                             |
  |                                                                |
  | 5. User Authentication                                         |
  |    - Client sends signed challenge using user private key      |
  |    - Server validates signature against authorized_keys        |
  |--------------------------------------------------------------->|
  |    - Authentication ACCEPTED                                   |
  |<---------------------------------------------------------------|
  |                                                                |
  | 6. Interactive Terminal / Session Request (PTY allocated)      |
  |<==============================================================>|
```

---

## 5. Summary Takeaways

- SSH provides encrypted, authenticated remote administration over TCP port 22, replacing legacy cleartext tools like Telnet.
- It leverages asymmetric cryptography for identity, Diffie-Hellman for key exchange, and symmetric ciphers for high-speed session encryption.
- Built-in data integrity checks prevent packet alteration and replay attacks.
- OpenSSH is the universal de-facto standard implementation on all modern Linux distributions.
