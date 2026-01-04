# Chapter 37: YUM and DNF Package Manager - Configuring DNF and YUM Repositories

## 1. Introduction: The .repo File Architecture

In Red Hat, CentOS, Rocky Linux, and AlmaLinux systems, repository configurations are defined in **`.repo`** INI-formatted text files stored in **`/etc/yum.repos.d/`**.

Whenever you invoke `dnf` or `yum`, the package manager scans `/etc/yum.repos.d/` for all files ending in `.repo`, reads their URLs and security parameters, and connects to the active endpoints.

---

## 2. Anatomy of a .repo File

A standard repository stanza consists of a section header enclosed in square brackets `[repo-id]` followed by key-value configuration directives:

```ini
[docker-ce-stable]
name=Docker CE Stable - $basearch
baseurl=https://download.docker.com/linux/centos/$releasever/$basearch/stable
enabled=1
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg
```

### Directive Breakdown:
| Directive | Description | Value Options |
| :--- | :--- | :--- |
| **`[repo-id]`** | Unique internal alphanumeric repository identifier. | No spaces (e.g., `[hashicorp]`). |
| **`name`** | Human-readable description displayed in DNF CLI output. | Any descriptive string. |
| **`baseurl`** | Direct HTTP/HTTPS/FTP URL pointing to the repository root directory. | Must contain `repodata/` directory. |
| **`mirrorlist`** | URL returning a dynamically generated list of regional geographic mirrors. | Alternative to static `baseurl`. |
| **`enabled`** | Controls whether DNF queries this repository during package operations. | `1` = Enabled, `0` = Disabled. |
| **`gpgcheck`** | Enforces cryptographic GPG signature validation on all downloaded packages. | **`1` = Highly Recommended**, `0` = Insecure. |
| **`gpgkey`** | URL or local file path (`file:///...`) to the vendor's public GPG signing key. | HTTPS URL or `/etc/pki/rpm-gpg/`. |

---

## 3. Dynamic DNF Variable Substitution

DNF automatically substitutes built-in variables within `baseurl` paths, ensuring scripts remain portable across CPU architectures and OS releases:

- **`$releasever`**: Replaced by the major OS release version (e.g., `9` on RHEL 9 or Rocky 9).
- **`$basearch`**: Replaced by the system CPU architecture (e.g., `x86_64`, `aarch64`).
- **`$infra`**: Replaced by the infrastructure type (e.g., `container`, `stock`).

By using `https://repo.company.com/rhel/$releasever/$basearch/`, the exact same configuration file works automatically on both Intel x86_64 and ARM64 servers!

---

## 4. Practical Workflows: Adding and Managing Repositories

### Method 1: Using `dnf config-manager` (Recommended CLI Tool)
The `dnf-plugins-core` package provides the `config-manager` utility to add and manage `.repo` files programmatically:

```bash
# 1. Install DNF core plugins
sudo dnf install -y dnf-plugins-core

# 2. Add an official repository in one command (e.g., HashiCorp Terraform / Vault)
sudo dnf config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo

# 3. Enable or disable a repository without deleting the file
sudo dnf config-manager --set-disabled hashicorp
sudo dnf config-manager --set-enabled hashicorp
```

### Method 2: Manual .repo File Creation (Automated DevOps / Ansible)
Create `/etc/yum.repos.d/postgresql.repo`:

```bash
sudo tee /etc/yum.repos.d/pgdg.repo << 'EOF'
[pgdg16]
name=PostgreSQL 16 for RHEL / Rocky $releasever - $basearch
baseurl=https://download.postgresql.org/pub/repos/yum/16/redhat/rhel-$releasever-$basearch
enabled=1
gpgcheck=1
gpgkey=https://download.postgresql.org/pub/repos/yum/RPM-GPG-KEY-PGDG
EOF
```

---

## 5. Auditing Repositories: `dnf repolist`

```bash
# List all currently enabled repositories:
dnf repolist
```

```
repo id                       repo name
appstream                     Rocky Linux 9 - AppStream
baseos                        Rocky Linux 9 - BaseOS
docker-ce-stable              Docker CE Stable - x86_64
epel                          Extra Packages for Enterprise Linux 9 - x86_64
hashicorp                     Hashicorp Stable - x86_64
```

```bash
# List ALL repositories (both enabled and disabled) with full URLs:
dnf repolist all -v
```

### Installing a Package from a Temporarily Enabled Repository:
If a repository is disabled by default (`enabled=0`) to prevent accidental package collisions, you can enable it for a single command invocation:
```bash
sudo dnf --enablerepo=pgdg16 install -y postgresql16-server
```

---

## 6. Summary Takeaways

- Repository files reside in `/etc/yum.repos.d/*.repo`.
- Use `$releasever` and `$basearch` variables to write portable repository configurations.
- Always enforce `gpgcheck=1` and specify trusted public GPG keys.
- Manage repos programmatically with `dnf config-manager --add-repo <url>`.
- Audit enabled repositories with `dnf repolist`.
