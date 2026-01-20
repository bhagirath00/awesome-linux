# User Namespaces (CLONE_NEWUSER) & Rootless Containers

## 1. Architectural Concept & Deep Dive

The **User Namespace** (`CLONE_NEWUSER`), introduced in Linux 3.8 (2013), isolates security credentials: User IDs (UIDs), Group IDs (GIDs), keys, and Linux capabilities (`capabilities(7)`).

In traditional Unix systems, root (UID 0) is an absolute, global privilege level. If a process running as UID 0 is compromised, the entire host operating system is compromised. 

With User Namespaces:
- A process can have **UID 0 (root)** and full privileges *inside* the namespace.
- Simultaneously, that exact same process maps to an **unprivileged UID (e.g., UID 1000 or UID 100000)** *outside* on the host kernel.
- If a containerized process breaks out of its mount or network jail, it arrives on the host with ordinary user rights, unable to write to `/etc/shadow`, mount block devices, or sniff host interfaces.

```
+-------------------------------------------------------------------------+
|                                HOST OS                                  |
|  Actual Kernel Identity:                                                |
|    User: `deployer` (UID 1000, GID 1000)                                |
|    Subordinate UIDs allocated in /etc/subuid: 100000:65536               |
|                                                                         |
|         +-----------------------------------------------------+         |
|         |               USER NAMESPACE                        |         |
|         |                                                     |         |
|         |  Internal View:                                     |         |
|         |    UID 0 (root inside container)                    |         |
|         |    UID 1..65535                                     |         |
|         |                                                     |         |
|         |  Capabilities Inside NS:                            |         |
|         |    CAP_SYS_ADMIN, CAP_NET_ADMIN (scoped to NS)      |         |
|         |                                                     |         |
|         |  Capabilities on Host:                              |         |
|         |    NONE (standard unprivileged user)                |         |
|         +-----------------------------------------------------+         |
|                                    │                                    |
|               Mapping: Inside UID 0 ──► Host UID 100000                 |
+-------------------------------------------------------------------------+
```

---

## 2. UID / GID Mapping Mechanics

Every user namespace defines its identity translation via two virtual files in `/proc`:
- `/proc/<PID>/uid_map`
- `/proc/<PID>/gid_map`

### Mapping Format
```
ID-inside-ns   ID-outside-ns   range-length
```
For example:
```
0 100000 65536
```
This entry states:
- Inside the namespace, IDs `0` through `65535` map to host IDs `100000` through `165535`.
- A process running as container UID 0 will create files on disk owned by host UID 100000.

### Subordinate IDs (`/etc/subuid` and `/etc/subgid`)
To prevent regular users from mapping their containers to other real users' host UIDs, the system administrator configures subordinate ranges in `/etc/subuid`:

```bash
cat /etc/subuid
```
```
deployer:100000:65536
ubuntu:165536:65536
```
*(User `deployer` is permitted to allocate a pool of 65,536 UIDs starting at host UID 100000).*

---

## 3. Hands-on Terminal Lab: Creating an Unprivileged Root Namespace

An unprivileged user can create a user namespace without `sudo`!

### Step 1: Create Namespace Without Sudo
```bash
# As standard user (e.g. UID 1000)
unshare --user --map-root-user /bin/bash
```

### Step 2: Verify Identity Inside vs Outside
Inside the new shell:
```bash
whoami
id
```
```
root
uid=0(root) gid=0(root) groups=0(root)
```

Now check the mapping file:
```bash
cat /proc/self/uid_map
```
```
         0       1000          1
```
*(Internal UID 0 maps directly to host user UID 1000).*

### Step 3: Test Privilege Containment
Try to read a file that requires real host root permissions:
```bash
cat /etc/shadow
```
```
cat: /etc/shadow: Permission denied
```
Even though the process reports `uid=0(root)`, the kernel checks the underlying host credential (`UID 1000`), refusing access.

---

## 4. Rootless Containers: Podman & Docker Rootless

Rootless container engines (`podman` natively, or `dockerd-rootless.sh`) leverage user namespaces to completely eliminate the requirement for a root daemon.

### Comparison Table

| Feature | Rootful Docker (`dockerd` as root) | Rootless Docker / Podman |
| :--- | :--- | :--- |
| **Daemon Privilege** | Runs as host UID 0 with full kernel privileges. | Daemon/engine runs as ordinary host UID. |
| **Attack Surface** | Container escape grants instant host root takeover. | Container escape drops attacker into host unprivileged shell. |
| **Privileged Ports (<1024)**| Can bind directly (e.g. 80, 443). | Requires `sysctl net.ipv4.ip_unprivileged_port_start=80` or port mapping >1024. |
| **Storage Driver** | Kernel `overlay2` directly. | `fuse-overlayfs` (older kernels) or rootless `overlay2` (Linux 5.11+). |
| **Networking Stack** | Host `veth` pairs & iptables bridge. | `slirp4netns` or `passt` (user-mode network stack). |

### Configuring Rootless Podman
```bash
# Install podman & shadow-utils
sudo apt-get install -y podman uidmap

# Check subuid allocation for current user
podman system info | grep -A 5 -i idmappings
```

---

## 5. Kubernetes & Production Architecture

### User Namespaces in Kubernetes (KEP-127)
Kubernetes introduced native User Namespace support starting in v1.25 (alpha) and v1.30 (beta):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hardened-nginx
spec:
  hostUsers: false  # Request an isolated User Namespace!
  containers:
  - name: nginx
    image: nginx:alpine
    securityContext:
      allowPrivilegeEscalation: false
```
When `hostUsers: false` is configured:
1. Kubelet and CRI runtime (`containerd` 1.7+) request a dynamic range from host `/etc/subuid`.
2. Container UID 0 maps to host UID 655360.
3. Vulnerabilities targeting host filesystems or kernel interfaces are contained.

---

## 6. Verification & Troubleshooting Runbook

### Inspection Commands
```bash
# View user namespace inode for a process
ls -l /proc/<PID>/ns/user

# View process credentials and capabilities
cat /proc/<PID>/status | grep -E "(Uid|Gid|Cap)"

# Inspect active UID mapping
cat /proc/<PID>/uid_map
```

### Common Pitfalls & Solutions
1. **`newuidmap: write to uid_map failed: Operation not permitted`**:
   - Cause: User is missing subordinate UID configuration or `/usr/bin/newuidmap` lacks SUID flag.
   - Solution: Run `sudo chmod u+s /usr/bin/newuidmap` and verify `/etc/subuid` has an entry for the user.
2. **Permission denied when mounting filesystems inside namespace**:
   - Cause: Mounting block filesystems (ext4, xfs) requires capabilities in the initial user namespace.
   - Solution: Use filesystems designed for user namespaces (such as `tmpfs`, `procfs`, `sysfs`, or user-space `fuse-overlayfs`).
3. **Cannot bind to port 80 in rootless container**:
   - Cause: Linux kernel reserves ports below 1024 for real host root.
   - Solution: Lower threshold via `sudo sysctl -w net.ipv4.ip_unprivileged_port_start=80` or bind to port 8080 and reverse-proxy.

---

## 7. Key Takeaways
- User namespaces decouple internal container credentials from host credentials, mapping container UID 0 to an unprivileged host UID.
- Inode `/proc/<PID>/uid_map` and `/proc/<PID>/gid_map` define mapping boundaries.
- Rootless containers (Podman, Docker Rootless) leverage user namespaces to run secure containers without requiring sudo or root daemon privileges.
- Kubernetes native `hostUsers: false` prevents cluster takeover through container breakout exploits.
