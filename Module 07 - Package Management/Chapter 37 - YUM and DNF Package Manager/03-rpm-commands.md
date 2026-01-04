# Chapter 37: YUM and DNF Package Manager - The rpm Low-Level Package Manager

## 1. Introduction: What is the rpm Command?

**RPM** (**RPM Package Manager**, originally **Red Hat Package Manager**) is the fundamental, low-level package tool for RHEL, CentOS, Rocky Linux, AlmaLinux, Fedora, and openSUSE.

Just as `dpkg` operates under `apt` on Ubuntu, `rpm` is the engine that sits directly underneath `dnf`. It unpacks `.rpm` archive payloads, writes binaries and configuration files to disk, executes embedded pre/post-installation scripts, and maintains the authoritative system package database stored in **`/var/lib/rpm/`**.

---

## 2. Command Flags Syntax Philosophy

The `rpm` utility relies on major operational modes designated by primary flags:

| Primary Flag | Mode | Function |
| :--- | :--- | :--- |
| **`-i`** | **Install** | Install a local `.rpm` package file onto the system. |
| **`-U`** | **Upgrade** | Upgrade an existing package (or install if not present). |
| **`-e`** | **Erase** | Uninstall / remove an installed package. |
| **`-q`** | **Query** | Inspect the local RPM database or an uninstalled `.rpm` file. |
| **`-V`** | **Verify** | Check installed files against database checksums for tampering. |

---

## 3. Installing and Upgrading Local Packages

When installing an RPM package directly:

```bash
# -i: Install, -v: Verbose, -h: Print hash mark progress bar (#)
sudo rpm -ivh package-name.rpm

# -U: Upgrade (Installs new version and cleans up old version cleanly)
sudo rpm -Uvh package-name-v2.rpm
```

> [!NOTE]
> If a local RPM file has missing dependencies, `rpm -ivh` will fail. In modern RHEL/Rocky, pass the local file directly to **`dnf`**, which installs the local package and downloads all missing dependencies automatically:
> ```bash
> sudo dnf install -y ./package-name.rpm
> ```

---

## 4. Querying the RPM Database (`-q`)

`rpm -q` is extremely fast and versatile for auditing servers:

### 1. Check if a Package is Installed
```bash
rpm -q nginx
```
```
nginx-1.24.0-2.el9.x86_64
```

### 2. List All Files Installed by a Package (`-ql`)
```bash
rpm -ql nginx
```
```
/etc/nginx/nginx.conf
/usr/sbin/nginx
/usr/share/doc/nginx
/var/log/nginx
```

### 3. Find Which Package Installed a File on Disk (`-qf`)
```bash
rpm -qf /etc/fstab
```
```
setup-2.13.7-9.el9.noarch
```

### 4. Inspect an Uninstalled Package File (`-qp`)
To audit an RPM file downloaded from the internet before installing it:
```bash
# Inspect metadata of uninstalled file (-qip)
rpm -qip custom-tool.rpm

# Inspect files inside uninstalled package (-qlp)
rpm -qlp custom-tool.rpm

# Inspect pre/post install shell scripts inside package (-q --scripts -p)
rpm -qp --scripts custom-tool.rpm
```

---

## 5. Security Auditing: Package Verification (`rpm -V`)

`rpm -V` (**Verify**) is a critical digital forensics and security hardening capability. It compares the MD5/SHA256 checksums, file sizes, permissions, and ownership of all files installed on your system against the baseline values recorded in the original RPM database:

```bash
# Verify integrity of all files belonging to OpenSSH server
rpm -V openssh-server
```

- If `rpm -V` returns **no output**, all files are authentic and unmodified.
- If a binary or file has been modified (e.g., by an attacker or rootkit), it prints verification status flags:

```
S.5....T.  c /etc/ssh/sshd_config
..5......    /usr/sbin/sshd
```

### Verification Flags Legend:
| Character | Modified Attribute |
| :--- | :--- |
| **`S`** | File **Size** differs |
| **`M`** | File **Mode** (permissions) differs |
| **`5`** | **MD5/SHA Checksum** differs (File contents were altered!) |
| **`D`** | Device major/minor number mismatch |
| **`L`** | SymLink path mismatch |
| **`U`** | **User ownership** differs |
| **`G`** | Group ownership differs |
| **`T`** | Modification **Time** differs |
| **`c`** | File is marked as a Configuration file (normal for edits to occur) |

> [!WARNING]
> If a core system binary (like `/usr/sbin/sshd` or `/bin/ls`) displays `..5......` without a `c` configuration flag, **the executable has been altered or trojaned!**

---

## 6. Summary Takeaways

- `rpm` is the low-level database manager and file unpacker under `dnf`.
- Use `rpm -ql <pkg>` to inspect all files installed by a package.
- Use `rpm -qf <file>` to find which package owns a specific path on disk.
- Audit downloaded packages with `rpm -qlp <file.rpm>` before installation.
- Use `rpm -V <pkg>` to verify file integrity and detect compromised system binaries.
