# Chapter 21: Linux Security - AppArmor Basics

## 1. Introduction: What is AppArmor?

**AppArmor** (**Application Armor**) is a Linux Security Module (LSM) alternative to SELinux. Enabled by default on **Ubuntu**, **Debian**, and **openSUSE**, AppArmor provides Mandatory Access Control (MAC) by binding access restrictions directly to program executable paths rather than relying on filesystem-wide extended attribute labels.

### AppArmor vs. SELinux Comparison

| Characteristic | AppArmor | SELinux |
| :--- | :--- | :--- |
| **Default Distributions** | Ubuntu, Debian, SUSE | RHEL, Rocky Linux, Fedora, CentOS |
| **Policy Model** | **Path-based** (`/usr/bin/nginx`) | **Inode/Label-based** (`httpd_sys_content_t`) |
| **Filesystem Dependency**| None; works on any filesystem | Requires extended attributes (`xattr`) support |
| **Configuration Complexity** | Simple, human-readable text profiles | Sophisticated, steep learning curve |
| **Container Integration** | Default profile on Docker/Kubernetes on Ubuntu | Enforced via `container_t` / `spc_t` |

---

## 2. AppArmor Operational Modes

Each individual AppArmor security profile operates in one of two primary modes:

| Mode | Behavior |
| :--- | :--- |
| **Enforce** | The profile policy is strictly applied. Disallowed actions are **blocked** and logged to syslog / audit logs. |
| **Complain** | Disallowed actions are **permitted**, but logged with a warning. Used to profile applications during testing. |

---

## 3. Querying AppArmor Status

Use `aa-status` to inspect active profiles and confined processes:

```bash
sudo aa-status
```

```
apparmor module is loaded.
42 profiles are loaded.
38 profiles are in enforce mode.
   /usr/bin/man
   /usr/sbin/mysqld
   /usr/sbin/tcpdump
   docker-default
4 profiles are in complain mode.
   /usr/sbin/named
0 profiles are in kill mode.
0 profiles are in unconfined mode.
7 processes have profiles defined.
7 processes are in enforce mode.
   /usr/sbin/mysqld (1120)
0 processes are in complain mode.
0 processes are unconfined but have a profile defined.
```

---

## 4. Anatomy of an AppArmor Profile

AppArmor profiles are plain-text files stored in `/etc/apparmor.d/`. Their names typically match the path to the executable with slashes replaced by dots (e.g., `/etc/apparmor.d/usr.sbin.mysqld`).

Here is a simplified example profile for a microservice binary `/usr/local/bin/myapp`:

```apparmor
#include <tunables/global>

/usr/local/bin/myapp {
  #include <abstractions/base>
  #include <abstractions/nameservice>

  # Network capability
  network inet tcp,

  # Allow reading its configuration file
  /etc/myapp/config.yaml r,

  # Allow read/write access to its runtime log directory
  /var/log/myapp/* rw,
  /var/log/myapp/ a,

  # Deny access to sensitive files explicitly
  deny /etc/shadow rwx,
  deny /etc/ssh/* rwx,

  # Execute child worker binaries
  /usr/local/bin/worker ix,
}
```

### Access Permission Identifiers:
- `r`: Read
- `w`: Write
- `a`: Append-only
- `x`: Execute
- `ix`: Execute child process inheriting the parent's AppArmor confinement
- `px`: Execute child process with its own discrete profile
- `ux`: Execute unconfined (bypasses AppArmor)
- `k`: File locking
- `m`: Memory mapping executable files

---

## 5. Profile Management Tools

The `apparmor-utils` package provides essential administrative utilities:

```bash
# Install utilities on Ubuntu/Debian
sudo apt install -y apparmor-utils
```

### Switching Profile Modes
```bash
# Put MySQL profile into complain mode for debugging
sudo aa-complain /usr/sbin/mysqld

# Return MySQL profile back to strict enforcement
sudo aa-enforce /usr/sbin/mysqld

# Disable a profile completely
sudo ln -s /etc/apparmor.d/usr.sbin.mysqld /etc/apparmor.d/disable/
sudo apparmor_parser -R /etc/apparmor.d/usr.sbin.mysqld
```

### Generating Profiles Automatically with `aa-genprof`
`aa-genprof` creates a new profile by running the application, monitoring all files and network sockets it touches, and prompting you to approve or deny each capability:

```bash
sudo aa-genprof /usr/local/bin/custom-api
```

---

## 6. Docker and AppArmor

When running Docker containers on an Ubuntu host, Docker automatically confines every container using the default `docker-default` profile located in `/etc/apparmor.d/docker`:

```bash
# Verify container is protected by AppArmor
docker inspect --format '{{ .AppArmorProfile }}' web-container
```

```
docker-default
```

The `docker-default` profile blocks containers from accessing `/proc/sysrq-trigger`, writing to `/proc/sys/`, mounting new filesystems, or accessing raw hardware devices, preventing container breakouts.

---

## 7. Summary Takeaways

- AppArmor provides path-based Mandatory Access Control on Ubuntu and Debian systems.
- Profiles operate in either **Enforce** (blocking) or **Complain** (auditing) mode.
- Profile rules are human-readable text files inside `/etc/apparmor.d/`.
- Manage profiles with `aa-status`, `aa-enforce`, `aa-complain`, and `aa-genprof`.
- AppArmor serves as a core layer in Docker container isolation on Linux hosts.
