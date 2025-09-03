# Chapter 36: APT Package Manager - APT Pinning and Version Locking

## 1. Introduction: What is APT Pinning?

By default, whenever you run `apt upgrade`, APT automatically selects the package with the **highest version number** across all enabled repositories.

However, in enterprise production environments, automatic version upgrades can cause catastrophic outages:
- Upgrading a major version of PostgreSQL (`15` to `16`) without planning halts database operations due to disk format incompatibility.
- A third-party repository might release an untested build that conflicts with system packages.
- A Kubernetes worker node requires Docker or `containerd` to stay locked at a specific certified patch level to prevent cluster evictions.

**APT Pinning** allows system administrators to assign **numerical priority scores** to specific packages, repositories, or distribution releases, controlling exactly which version APT installs or preventing upgrades altogether.

---

## 2. APT Pinning Priority Scale

Priorities are configured using integer scores:

| Priority Value | Meaning and Behavior |
| :--- | :--- |
| **`P >= 1000`** | **Forced Install / Downgrade**: Installs this version even if it causes a **downgrade** of an installed package. |
| **`990 <= P < 1000`** | **Target Release**: Installs this version unless the currently installed version is newer. |
| **`500 <= P < 990`** | **Standard Default**: The default priority assigned to packages in configured repositories. |
| **`100 <= P < 500`** | **Installed Version**: Priority of packages already installed on the local system. |
| **`0 < P < 100`** | **Available but Not Auto-Installed**: Only installs if explicitly requested by the user. |
| **`P < 0`** | **Blacklisted / Forbidden**: Prevents the package from **EVER** being installed. |

---

## 3. Configuring Pinning in `/etc/apt/preferences.d/`

Always configure pinning rules inside dedicated drop-in files in **`/etc/apt/preferences.d/`**.

A pinning stanza contains three fields:
1. `Package`: The target package name (supports wildcards `*`).
2. `Pin`: The matching criteria (version, release, or origin domain).
3. `Pin-Priority`: The integer score.

### Example 1: Lock Nginx to Version 1.24
```bash
sudo tee /etc/apt/preferences.d/nginx-pin << 'EOF'
Package: nginx*
Pin: version 1.24.*
Pin-Priority: 1001
EOF
```
Because the priority is `1001`, `apt upgrade` will refuse to upgrade Nginx to version `1.26` or higher!

### Example 2: Blacklist an Unwanted Package
Prevent a package from ever being installed on the server:
```bash
sudo tee /etc/apt/preferences.d/block-snapd << 'EOF'
Package: snapd
Pin: release *
Pin-Priority: -1
EOF
```

---

## 4. Temporary Version Holding with `apt-mark hold`

While APT Pinning offers granular, multi-repository priority scoring, simple single-package version locking is achieved much faster using **`apt-mark`**:

```bash
# Lock a package at its current installed version (Freeze updates)
sudo apt-mark hold containerd.io
```
```
containerd.io set on hold.
```

Now, even if you run `sudo apt upgrade -y`, `containerd.io` will be skipped:
```
The following packages have been kept back:
  containerd.io
0 upgraded, 0 newly installed, 0 to remove and 1 not upgraded.
```

### Inspecting and Releasing Holds:
```bash
# Show all packages currently locked on hold:
apt-mark showhold

# Unhold the package when ready to perform a controlled upgrade:
sudo apt-mark unhold containerd.io
```

---

## 5. Summary Takeaways

- APT Pinning controls package version selection via numeric priorities (`Pin-Priority`).
- A priority > `1000` forces APT to keep that version or downgrade if necessary.
- Store pinning rules cleanly in `/etc/apt/preferences.d/<name>`.
- For simple version locking on production nodes, use **`apt-mark hold <pkg>`**.
- Inspect locked packages anytime with `apt-mark showhold`.
