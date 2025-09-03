# Chapter 36: APT Package Manager - Adding Third-Party Repositories Securely

## 1. Introduction: The Deprecation of `apt-key`

For over a decade, tutorials instructed developers to add third-party repository keys using:
```bash
# DEPRECATED AND INSECURE:
curl -s https://vendor.com/key.gpg | sudo apt-key add -
```

### Why `apt-key add` Was Deprecated
When you ran `apt-key add`, the vendor's public GPG key was injected into the system's **global trusted keyring** (`/etc/apt/trusted.gpg`). 

Once a key was inside `/etc/apt/trusted.gpg`, **that key was trusted to sign ANY package for the entire operating system!** If a compromised third-party repository decided to publish a poisoned package named `libc6`, `systemd`, or `openssh-server`, APT would happily install it, overwriting core system binaries with malicious code.

---

## 2. The Modern, Secure Standard: `/etc/apt/keyrings/`

In modern Linux security architecture, third-party keys are isolated inside dedicated, read-only files in **`/etc/apt/keyrings/`**, and each repository definition explicitly references its key using the **`signed-by`** parameter.

If a vendor's key is signed-by for Docker, that key can **ONLY** sign Docker packages. APT will outright reject it if it attempts to sign anything else!

```
+--------------------------------------------------------------------------+
|                     Modern Secure Keyring Architecture                   |
+--------------------------------------------------------------------------+
|  /etc/apt/keyrings/docker.gpg  <-- Isolated binary GPG keyring           |
|                                                                          |
|  /etc/apt/sources.list.d/docker.list:                                    |
|  deb [signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker... |
|                                                                          |
|  Result: Key is strictly scoped to docker.com; cannot poison system!     |
+--------------------------------------------------------------------------+
```

---

## 3. End-to-End Walkthrough: Adding the Official Docker CE Repository

Here is the exact, production-hardened recipe recommended by Docker and Canonical:

### Step 1: Install Prerequisite Network Tools
```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
```

### Step 2: Create the Dedicated Keyrings Directory
Ensure the directory exists with secure permissions (`755`):
```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

### Step 3: Download and Armor/De-Armor the GPG Key
Download the vendor's ASCII-armored key and convert it to OpenPGP binary format using `gpg --dearmor`:
```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```
*(Or de-armored: `curl -fsSL ... | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg`)*.

### Step 4: Add the Repository Definition with `signed-by`
Dynamically inject the current system architecture (`dpkg --print-architecture`) and distribution release codename:
```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Step 5: Update Indexes and Install
```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

---

## 4. Personal Package Archives (PPA) with `add-apt-repository`

On Ubuntu systems, community software is often hosted on Launchpad via **PPAs**:

```bash
# Install software-properties-common for add-apt-repository tool
sudo apt install -y software-properties-common

# Add a community PPA (automatically retrieves GPG key and adds source)
sudo add-apt-repository -y ppa:ansible/ansible

# Refresh and install
sudo apt update
sudo apt install -y ansible
```

### Removing a PPA:
```bash
sudo add-apt-repository --remove -y ppa:ansible/ansible
```

---

## 5. Summary Takeaways

- Never use `apt-key add` in modern Linux; it is deprecated and presents a major security risk.
- Always store third-party public keys isolated in `/etc/apt/keyrings/`.
- Restrict repositories using the `[signed-by=/path/to/key.gpg]` directive.
- Automate repository configuration using `$(dpkg --print-architecture)` and `$VERSION_CODENAME`.
