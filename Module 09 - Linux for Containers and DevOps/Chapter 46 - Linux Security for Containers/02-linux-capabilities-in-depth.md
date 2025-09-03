# Linux Capabilities in Depth for Containers

## 1. Architectural Concept & Deep Dive

Historically in Unix, permission checks followed a binary model: either a process was running as a normal user (UID != 0, subject to standard DAC permissions), or it was running as the superuser **root** (UID == 0, completely bypassing all kernel checks).

This all-or-nothing model was a catastrophic security flaw. If a daemon like `ntpd` needed only to adjust system time, or a web server needed only to bind to port 80, it had to run with full UID 0 privileges, exposing the entire host to takeover in the event of an exploit.

Starting in Linux 2.2, the kernel split the monolithic powers of root into **Linux Capabilities** (`capabilities(7)`). Capabilities are distinct, independent privilege flags that can be enabled or disabled individually per process or binary.

```
                   Traditional Unix Root (UID 0)
         [ All-or-Nothing Monolithic God Mode Privileges ]
                                │
                                ▼
                   Linux Kernel Capabilities
    ┌───────────────────────────┼───────────────────────────┐
    ▼                           ▼                           ▼
CAP_NET_BIND_SERVICE     CAP_NET_ADMIN             CAP_SYS_ADMIN
(Bind ports < 1024)      (Configure interfaces,    (Mount filesystems,
                          iptables, routing)        swap, bpf, cgroups)
    ▼                           ▼                           ▼
CAP_CHOWN                CAP_SETUID                CAP_KILL
(Change file ownership)  (Change process UID)      (Send signals to any PID)
```

In container runtimes, capabilities form the primary line of privilege enforcement: even if a process runs as `root` (UID 0) inside a container, **it is NOT a full root user** because the runtime strips over 60% of kernel capabilities before running the container entrypoint.

---

## 2. The Four Capability Sets of a Process

Every Linux task maintains four distinct 64-bit bitmasks in its `task_struct`:

| Capability Set | Kernel Bitmask | Meaning & Purpose |
| :--- | :--- | :--- |
| **Effective (`CapEff`)** | Active Capabilities | The exact capabilities currently being checked by the kernel for permission to execute actions. |
| **Permitted (`CapPrm`)** | Allowed Capabilities | The limiting super-set of capabilities the process may move into its Effective set. |
| **Inheritable (`CapInh`)**| Preserved Capabilities | Capabilities that can be preserved across an `execve(2)` call. |
| **Bounding (`CapBnd`)** | Maximum Ceiling | The absolute ceiling of capabilities a process and its children can ever acquire, even if executing a setuid-root binary. |

```
Bounding Set (CapBnd) ──► Hard ceiling
       │
       ▼
Permitted Set (CapPrm) ──► What process is allowed to use
       │
       ▼
Effective Set (CapEff) ──► What kernel evaluates during syscall execution
```

Container runtimes restrict the container by pruning the **Bounding Set** (`CapBnd`) before dropping privileges or executing the container process.

---

## 3. Critical Capabilities Breakdown

Linux defines approximately 41 capabilities. The table below outlines the most critical capabilities in container and DevOps security:

| Capability | Description | Exploitation Risk / Container Breakout Threat |
| :--- | :--- | :--- |
| **`CAP_SYS_ADMIN`** | The "new root". Permits mounting, namespaces, cgroups, BPF, device ioctls. | **CRITICAL**. Easily leads to full host container breakout. Never grant to untrusted containers. |
| **`CAP_NET_ADMIN`** | Modify network interfaces, routing tables, firewall (iptables/nftables), IPsec. | High. Can spoof host traffic, alter ARP tables, or sniff host interfaces if network namespace is shared. |
| **`CAP_NET_RAW`** | Craft raw packets (ICMP ping, ARP spoofing). | Medium. Allows packet sniffing and ping floods. Default in Docker; should be dropped if not needed. |
| **`CAP_NET_BIND_SERVICE`**| Bind to privileged ports (< 1024) such as 80 and 443. | Low. Safe for web servers that must bind port 80 without running as UID 0. |
| **`CAP_DAC_OVERRIDE`** | Bypass file read, write, and execute permission checks (ignore POSIX mode bits). | High. Allows writing to any file owned by other users within the container mount. |
| **`CAP_SYS_PTRACE`** | Trace arbitrary processes via `ptrace(2)`. | High. Enables inspecting and injecting code into memory of other processes running in the same PID namespace. |
| **`CAP_SYS_MODULE`** | Load and unload kernel modules (`init_module`). | **EXTREME**. Instant host kernel takeover. |

---

## 4. Default Container Capabilities

By default, Docker and containerd grant only **14 capabilities** to non-privileged containers, discarding all others:

```
Default 14 Capabilities in Docker:
  CAP_CHOWN, CAP_DAC_OVERRIDE, CAP_FOWNER, CAP_FSETID,
  CAP_KILL, CAP_SETGID, CAP_SETUID, CAP_SETPCAP,
  CAP_NET_BIND_SERVICE, CAP_NET_RAW, CAP_SYS_CHROOT,
  CAP_MKNOD, CAP_AUDIT_WRITE, CAP_SETFCAP
```

Notice what is **NOT** included:
- `CAP_SYS_ADMIN` (Disabled)
- `CAP_NET_ADMIN` (Disabled)
- `CAP_SYS_MODULE` (Disabled)
- `CAP_SYS_RAWIO` (Disabled)
- `CAP_SYS_PTRACE` (Disabled)

---

## 5. Hands-on Terminal Lab: Inspecting and Restricting Capabilities

### Step 1: Inspect Capabilities of a Process (`capsh` & `getpcaps`)
```bash
# Check capabilities of current process
capsh --print
```
```
Current: = cap_chown,cap_dac_override,cap_setuid,cap_setgid...
Bounding set =cap_chown,cap_dac_override...
Securebits: 00/0x0/1'b0
```

To decode raw bitmasks from `/proc/<PID>/status`:
```bash
grep Cap /proc/$$/status
```
```
CapInh: 0000000000000000
CapPrm: 000001ffffffffff
CapEff: 000001ffffffffff
CapBnd: 000001ffffffffff
CapAmb: 0000000000000000
```
Decode the hexadecimal bitmask using `capsh`:
```bash
capsh --decode=000001ffffffffff
```

### Step 2: The Gold Standard: Drop ALL and Add Only What You Need
In Docker:
```bash
# Drop all capabilities, add only NET_BIND_SERVICE to run an unprivileged web server
docker run --rm -it \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  nginx:alpine
```

Verify inside the container that `chown` or `kill` are blocked:
```sh
# Inside container:
chown guest /tmp
# Output: chown: /tmp: Operation not permitted
```

---

## 6. Hardening Capabilities in Kubernetes Pod Specs

Never run containers with `--privileged`. Instead, use explicit capability declarations in `securityContext`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hardened-microservice
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: api
        image: mycompany/api:v1.2
        securityContext:
          runAsNonRoot: true
          runAsUser: 10001
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL       # Drop all 14 default capabilities!
            add:
            - NET_BIND_SERVICE  # Only if binding to port 80/443
```

---

## 7. Verification & Troubleshooting Runbook

### Diagnostic Checklist
```bash
# Inspect capabilities of all processes running inside a container PID
getpcaps <CONTAINER_PID>

# Check if privilege escalation is disabled
cat /proc/<PID>/status | grep NoNewPrivs
```
- If `NoNewPrivs: 1`, the process cannot acquire new capabilities even by running SUID binaries.

### Common Pitfalls & Solutions
1. **Application fails with `Operation not permitted` on port binding**:
   - Cause: Process is running as non-root user and trying to bind port 80 without `CAP_NET_BIND_SERVICE`.
   - Solution: Add capability via `cap-add=NET_BIND_SERVICE` or configure application to listen on port 8080.
2. **Wireshark / tcpdump fails inside container**:
   - Cause: Capturing packets requires `CAP_NET_RAW` and `CAP_NET_ADMIN`.
   - Solution: Run container with `--cap-add=NET_ADMIN --cap-add=NET_RAW`.
3. **VPN / WireGuard / Cilium fails to start**:
   - Cause: Creating `tun` devices and routing tables requires `CAP_NET_ADMIN`.
   - Solution: Grant `NET_ADMIN` explicitly rather than granting full `--privileged`.

---

## 8. Key Takeaways
- Linux Capabilities decompose root into 41 distinct privileges (e.g. `CAP_NET_ADMIN`, `CAP_CHOWN`).
- Container runtimes strip over half of the kernel's capabilities by default, blocking dangerous actions even for UID 0 inside containers.
- The principle of least privilege dictates dropping all capabilities (`--cap-drop=ALL` / `capabilities.drop: ["ALL"]`) and selectively adding back only the specific capabilities required.
- `CAP_SYS_ADMIN` is functionally equivalent to root; never grant it to production containers.
