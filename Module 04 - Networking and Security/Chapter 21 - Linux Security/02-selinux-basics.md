# Chapter 21: Linux Security - SELinux Basics

## 1. Introduction: What is SELinux?

**SELinux** (**Security-Enhanced Linux**) is a Mandatory Access Control (MAC) security architecture developed originally by the United States National Security Agency (NSA) and integrated into the mainline Linux kernel. It is enabled by default on enterprise Red Hat family distributions (RHEL, CentOS, Fedora, Rocky Linux, and AlmaLinux).

In standard Discretionary Access Control (DAC), permissions are governed by file owners and permissions bits (`rwxrwxrwx`). If an attacker compromises a service running as `root` (or exploits a web vulnerability where the service has write permissions), DAC allows the attacker complete freedom across the filesystem.

Under SELinux, the kernel checks **every single operation** (file read, network socket bind, process fork) against a strict security policy database. Even if an attacker executes code as `root` within a compromised web server daemon, SELinux prevents that process from reading `/home`, accessing `/etc/shadow`, or communicating with unauthorized internal network ports!

```
+--------------------------------------------------------------------------+
|                             SELinux In Action                            |
+--------------------------------------------------------------------------+
|  Subject (Process)         Kernel LSM Hook            Target (Object)    |
|  +-------------------+     +--------------------+     +----------------+ |
|  | httpd (Apache)    | ==> | SELinux Policy Eng | ==> | /var/www/html  | |
|  | Context:          |     | "Does httpd_t have |     | Context:       | |
|  | httpd_t           |     |  read permission   |     | httpd_sys_     | |
|  +-------------------+     |  to httpd_sys_     |     | content_t      | |
|                            |  content_t?"       |     +----------------+ |
|                            +---------+----------+                        |
|                                      |                                   |
|                          ALLOWED (Access Granted)                        |
|                                                                          |
|  Subject (Attacker)                  |                Target (Object)    |
|  +-------------------+               |                +----------------+ |
|  | httpd (Compromised|               v                | /etc/shadow    | |
|  |  by Web Shell)    | ==> [ DENIED & LOGGED ]   ==X  | Context:       | |
|  | Context: httpd_t  |                                | shadow_t       | |
|  +-------------------+                                +----------------+ |
+--------------------------------------------------------------------------+
```

---

## 2. The Three SELinux Operating Modes

| Mode | System Behavior | Use Case |
| :--- | :--- | :--- |
| **Enforcing** | Security policy is strictly enforced. Violations are **blocked** and logged to `/var/log/audit/audit.log`. | **Production Standard** (Required for secure systems). |
| **Permissive** | Security policy is not enforced. Violations are **allowed**, but warnings are still logged. | **Troubleshooting & Development** (Used to discover what rules need updating). |
| **Disabled** | SELinux is completely turned off; no labels are maintained in the filesystem. | **Never recommended**; re-enabling requires full filesystem relabeling on reboot. |

### Checking and Changing Modes Dynamically
```bash
# Check current SELinux status and policy mode
sestatus
```

```
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Memory protection checking:     actual (secure)
Max kernel policy version:      33
```

```bash
# Switch to Permissive mode temporarily without rebooting (for debugging)
sudo setenforce 0

# Switch back to Enforcing mode
sudo setenforce 1
```

### Making Mode Changes Persistent
Persistent configuration is stored in `/etc/selinux/config`:
```bash
sudo sed -i 's/^SELINUX=.*/SELINUX=enforcing/' /etc/selinux/config
```

---

## 3. SELinux Security Contexts (Labels)

Every file, directory, process, port, and network socket on an SELinux system is tagged with a security context. You can inspect contexts by appending the `-Z` flag to standard commands (`ls -Z`, `ps -eZ`, `netstat -Z`, `id -Z`).

```bash
ls -lZ /var/www/html/index.html
```

```
-rw-r--r--. 1 root root system_u:object_r:httpd_sys_content_t:s0 142 Oct 14 04:30 /var/www/html/index.html
```

A context consists of four colon-delimited components:
```
system_u : object_r : httpd_sys_content_t : s0
--------   --------   -------------------   --
  User       Role            Type           Level (MLS/MCS)
```

1. **User (`system_u`)**: SELinux user identity.
2. **Role (`object_r`)**: SELinux role (processes have roles like `system_r`; files use generic `object_r`).
3. **Type (`httpd_sys_content_t`)**: **The most important field (Type Enforcement)**. Defines what kind of object this is. Processes running in domain `httpd_t` can only interact with files labeled with compatible types like `httpd_sys_content_t`.
4. **Level (`s0`)**: Multi-Level Security / Multi-Category Security (MLS/MCS) sensitivity level.

---

## 4. Managing File Contexts: chcon vs. semanage & restorecon

### The Common Mistake: `chcon`
`chcon` (change context) modifies the label in memory, but **does not write to the SELinux policy database**. The moment the server relabels the filesystem or runs `restorecon`, the change is wiped out!

### The Correct Production Method: `semanage` + `restorecon`
When serving web files from a custom location (e.g., `/srv/mywebsite/` instead of `/var/www/html/`):

```bash
# 1. Install SELinux management tools
sudo dnf install -y policycoreutils-python-utils

# 2. Add rule to policy database matching the new path regex
sudo semanage fcontext -a -t httpd_sys_content_t "/srv/mywebsite(/.*)?"

# 3. Apply the label to disk recursively
sudo restorecon -Rv /srv/mywebsite
```

```
Relabeled /srv/mywebsite from unconfined_u:object_r:default_t:s0 to unconfined_u:object_r:httpd_sys_content_t:s0
Relabeled /srv/mywebsite/index.html from unconfined_u:object_r:default_t:s0 to unconfined_u:object_r:httpd_sys_content_t:s0
```

---

## 5. SELinux Booleans

Booleans are toggles that allow administrators to adjust SELinux policy behavior at runtime without compiling custom policy modules:

```bash
# Search for booleans related to httpd/web servers
getsebool -a | grep httpd
```

```
httpd_can_network_connect --> off
httpd_can_network_connect_db --> off
httpd_enable_homedirs --> off
httpd_use_nfs --> off
```

For example, if your PHP web application needs to connect to an external PostgreSQL database or reverse-proxy to a Node.js backend:

```bash
# Enable network connections for web services permanently (-P)
sudo setsebool -P httpd_can_network_connect on
sudo setsebool -P httpd_can_network_connect_db on
```

---

## 6. Troubleshooting SELinux AVC Denials

When SELinux blocks an operation, it records an **Access Vector Cache (AVC)** denial in `/var/log/audit/audit.log`.

Use `audit2why` and `sealert` to analyze denials and receive human-readable remediation advice:

```bash
# Search recent AVC denials
sudo ausearch -m avc -ts recent

# Explain why the denial occurred and how to resolve it
sudo ausearch -m avc -ts recent | audit2why
```

---

## 7. Summary Takeaways

- SELinux provides Mandatory Access Control, confining processes even if run as root.
- Never set SELinux to `disabled`; use `permissive` temporarily for troubleshooting.
- The `Type` component (`_t`) in the security context determines access rights.
- Always use `semanage fcontext` followed by `restorecon` to persist custom directory labels.
- Use `setsebool -P` to toggle pre-packaged policy capabilities.
