# Chapter 36: APT Package Manager - The dpkg Low-Level Package Manager

## 1. Introduction: What is dpkg?

**`dpkg`** (**Debian Package**) is the core, low-level foundation of package management on Debian and Ubuntu systems. 

While `apt` handles network downloading and dependency calculation, `dpkg` is the engine that physically executes the installation:
- It unpacks the `.deb` archive onto the filesystem.
- It executes embedded maintainer scripts (`preinst`, `postinst`).
- It records file checksums and package metadata into the local database at `/var/lib/dpkg/status`.

---

## 2. Installing Local .deb Packages (`dpkg -i`)

When you download a proprietary `.deb` package directly from a vendor (e.g., Google Chrome, Slack, AWS SSM Agent):

```bash
# Download agent package
curl -fsSL -O https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/debian_amd64/amazon-ssm-agent.deb

# Install package directly
sudo dpkg -i amazon-ssm-agent.deb
```

### The "Unmet Dependencies" Trap and `apt -f install`
Because `dpkg` operates strictly offline and cannot download missing dependencies from the internet, installing a package via `dpkg -i` often terminates with an error:
```
dpkg: dependency problems prevent configuration of custom-app:
 custom-app depends on libssl1.1; however:
  Package libssl1.1 is not installed.
```

**The Universal Fix**: Instruct `apt` to evaluate the broken state, download the missing dependencies over the network, and complete the installation:
```bash
sudo apt-get install -f -y
# OR:
sudo apt --fix-broken install -y
```

*(Tip: In modern Ubuntu/Debian, you can bypass `dpkg -i` completely by passing the local file directly to `apt`: `sudo apt install ./amazon-ssm-agent.deb`. APT will install the local file AND download all dependencies in one command!).*

---

## 3. Querying the Installed Package Database

`dpkg` is exceptionally fast at querying installed files because it reads directly from the local status database:

```bash
# 1. Check if a package is installed and its current status (-s)
dpkg -s nginx
```
```
Package: nginx
Status: install ok installed
Priority: optional
Section: httpd
Installed-Size: 1420
Maintainer: Ubuntu Developers
Version: 1.24.0-1ubuntu1
```

```bash
# 2. List all files installed on disk by a package (-L)
dpkg -L nginx
```
```
/etc/nginx/nginx.conf
/usr/sbin/nginx
/usr/share/man/man8/nginx.8.gz
/var/log/nginx
```

```bash
# 3. Find WHICH package owns a specific file on disk (-S)
# "Where did /usr/bin/curl come from?"
dpkg -S /usr/bin/curl
```
```
curl: /usr/bin/curl
```

---

## 4. Reconfiguring Installed Packages: `dpkg-reconfigure`

Many packages include interactive configuration wizards that run during initial installation. You can relaunch these setup wizards at any time using **`dpkg-reconfigure`**:

```bash
# Change system timezone interactively
sudo dpkg-reconfigure tzdata

# Reconfigure system keyboard layout
sudo dpkg-reconfigure keyboard-configuration

# Reconfigure unattended security upgrades
sudo dpkg-reconfigure unattended-upgrades
```

---

## 5. dpkg Command Cheat Sheet

| Command Syntax | Operation Description |
| :--- | :--- |
| `dpkg -i <file.deb>` | Unpack and install local `.deb` archive. |
| `dpkg -r <pkg>` | Remove installed package binaries (leaves `/etc` configs). |
| `dpkg -P <pkg>` | Purge package and all configuration files permanently. |
| `dpkg -l [pattern]` | List all installed packages matching pattern. |
| `dpkg -s <pkg>` | Display detailed status information of an installed package. |
| `dpkg -L <pkg>` | List every single file installed on the system by `<pkg>`. |
| `dpkg -S <path>` | Identify which package installed `<path>` on disk. |
| `dpkg -c <file.deb>` | Inspect the contents of an **uninstalled** `.deb` file. |
| `dpkg-reconfigure <pkg>` | Relaunch interactive post-install configuration wizard. |

---

## 6. Summary Takeaways

- `dpkg` is the low-level engine that unpacks and registers `.deb` files locally.
- Fix broken dependencies after `dpkg -i` using `sudo apt --fix-broken install`.
- Modern `apt install ./file.deb` handles both local files and remote dependencies simultaneously.
- Use `dpkg -S <path>` to trace file ownership.
- Use `dpkg -L <pkg>` to list all files installed by a package.
