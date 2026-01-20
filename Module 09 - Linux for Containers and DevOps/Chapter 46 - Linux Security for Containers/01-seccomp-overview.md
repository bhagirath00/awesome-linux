# Linux Seccomp (Secure Computing Mode) for Containers

## 1. Architectural Concept & Deep Dive

**Seccomp (Secure Computing Mode)** is a Linux kernel security facility (introduced in Linux 2.6.12, expanded via **seccomp-bpf** in Linux 3.5) that restricts the system calls (syscalls) a process can make into the kernel.

The standard Linux kernel exposes more than 350-400 system calls. Typical containerized web microservices (Node.js, Go, Python, Java) require only 40 to 70 of these syscalls to execute correctly. The remaining hundreds of syscalls—including `reboot`, `kexec_load`, `swapon`, `sysfs`, `bpf`, and `init_module`—represent unnecessary attack surface. If a vulnerability exists in any kernel subsystem, an attacker who compromises a container can trigger that vulnerability only if the relevant syscall is accessible.

Seccomp acts as a syscall firewall operating right at the userspace-kernel boundary:

```
+─────────────────────────────────────────────────────────────+
|                     CONTAINER USERSPACE                     |
|                                                             |
|   Application Code (Go / Node.js / Python)                  |
|          │                    │                     │       |
|          ▼                    ▼                     ▼       |
|     read(2) [OK]         write(2) [OK]         reboot(2)    |
+──────────┼────────────────────┼─────────────────────┼───────+
           │                    │                     │
           ▼                    ▼                     ▼
┌─────────────────────────────────────────────────────────────┐
│                 SECCOMP-BPF FILTER ENGINE                   │
│   Evaluates BPF bytecode rules attached to task struct      │
│   • read(2)   -> ALLOW                              │
│   • write(2)  -> ALLOW                              │
│   • reboot(2) -> REJECT (EPERM or KILL)                     │
└─────────────────────────────────────────────────────┼───────┘
                                                      │ BLOCKED!
                                                      ▼
                                           Returns -EPERM or
                                           terminates process
```

---

## 2. Seccomp Actions & Architecture

When an application invokes a system call, the Seccomp BPF filter evaluates the syscall number and arguments, returning one of the following action codes:

| Action | Kernel Return Code | Behavior | Production Impact |
| :--- | :--- | :--- | :--- |
| **`SCMP_ACT_ALLOW`** | Immediate passthrough | Syscall executes normally inside kernel. | Allowed operations. |
| **`SCMP_ACT_ERRNO`** | `-EPERM` or `-ENOSYS` | Syscall blocked immediately without executing; returns error to application. | Preferred defensive mode; graceful failure without killing service. |
| **`SCMP_ACT_KILL_PROCESS`**| `SIGSYS` (core dump) | Immediately terminates entire process thread group. | Strict defense against exploitation attempts. |
| **`SCMP_ACT_KILL_THREAD`** | `SIGSYS` | Terminates only the calling thread. | Granular thread termination. |
| **`SCMP_ACT_LOG`** | Passthrough with audit | Syscall is allowed, but an audit log is emitted to syslog/auditd. | **Crucial for profiling new services** before enforcing. |
| **`SCMP_ACT_TRACE`** | `PTRACE_EVENT_SECCOMP` | Notifies an attached tracer (like ptrace or debugger). | Debugging and dynamic sandbox monitors. |

---

## 3. The Default Container Seccomp Profile

Container engines (Docker, containerd, CRI-O) apply a default Seccomp JSON profile automatically to all non-privileged containers.

### Blocked Syscalls in Default Profile
The default profile blocks approximately **44 to 50** dangerous system calls, including:
- **Kernel & Hardware Manipulation**: `reboot`, `kexec_load`, `acct`, `swapon`, `swapoff`, `sysfs`, `iopl`, `ioperm`.
- **Kernel Module Loading**: `init_module`, `finit_module`, `delete_module`.
- **Filesystem & Mount Modification**: `mount`, `umount2`, `pivot_root` (unless granted capabilities).
- **Process Tracing & Memory Injection**: `ptrace` (prevents inspecting host or container sibling memory).
- **Clock Manipulation**: `adjtimex`, `settimeofday`, `clock_settime` (prevents host time disruption).
- **Namespaces**: `unshare`, `setns` (blocks nested namespace creation unless allowed).

### Verifying Seccomp Status of a Process
```bash
# Check if Seccomp is active for a process (0 = disabled, 2 = seccomp-bpf filter)
grep Seccomp /proc/<PID>/status
```
```
Seccomp:        2
Seccomp_filters:        1
```

---

## 4. Hands-on Terminal Lab: Applying Custom Seccomp Profiles

### Step 1: Create a Custom Restrictive Seccomp Profile (`deny-mkdir.json`)
```json
{
  "defaultAction": "SCMP_ACT_ALLOW",
  "architectures": [
    "SCMP_ARCH_X86_64",
    "SCMP_ARCH_X86",
    "SCMP_ARCH_X32"
  ],
  "syscalls": [
    {
      "names": [
        "mkdir",
        "mkdirat"
      ],
      "action": "SCMP_ACT_ERRNO",
      "args": []
    }
  ]
}
```

### Step 2: Run a Docker Container with the Custom Profile
```bash
docker run --rm -it \
  --security-opt seccomp=deny-mkdir.json \
  alpine:latest sh
```

### Step 3: Test Syscall Interception
Inside the container:
```sh
# Normal command works
echo "Hello from container"
# Output: Hello from container

# Attempt to execute mkdir
mkdir /test_dir
# Output: mkdir: can't create directory '/test_dir': Operation not permitted
```
The kernel intercepted the `mkdirat` syscall at the seccomp layer and returned `EPERM` instantly.

---

## 5. Seccomp in Kubernetes (SecurityProfilesOperator & RuntimeDefault)

Starting in Kubernetes 1.27, `RuntimeDefault` is the recommended default seccomp profile across all production clusters:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secured-api-pod
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault  # Uses containerd/CRI-O's built-in hardened seccomp profile
  containers:
  - name: api
    image: myrepo/api:1.0
    securityContext:
      allowPrivilegeEscalation: false
```

### Applying a Custom Profile via ConfigMap / HostPath
```yaml
securityContext:
  seccompProfile:
    type: Localhost
    localhostProfile: profiles/restrictive-api.json
```
*(File must reside in the kubelet seccomp root: `/var/lib/kubelet/seccomp/profiles/restrictive-api.json`).*

---

## 6. Auditing & Debugging Syscall Violations

When creating a new custom profile, you must discover which syscalls your application actually requires without breaking production.

### Step 1: Use `SCMP_ACT_LOG` for Profiling
Set `"defaultAction": "SCMP_ACT_LOG"` in your profile. Run your application under full integration test load.

### Step 2: Inspect Audit Logs
```bash
# Query auditd logs for seccomp events
sudo ausearch -m SECCOMP -ts recent
```
```
type=SECCOMP msg=audit(1697264891.123:456): auid=4294967295 uid=1000 gid=1000 ses=4294967295 
subj=unconfined pid=14502 comm="python3" exe="/usr/bin/python3.10" 
sig=0 arch=c000003e syscall=257 compat=0 ip=0x7f23a4123b31 code=0x7ffc0000
```
- `syscall=257`: Translate to syscall name via `ausyscall x86_64 257` (`openat`).

---

## 7. Verification & Troubleshooting Runbook

### Common Pitfalls & Solutions
1. **Application crashes with `Bad system call (core dumped)`**:
   - Cause: Process executed a syscall blocked by an `SCMP_ACT_KILL` action.
   - Solution: Run `dmesg -T` or `coredumpctl` to find the violating syscall number. Convert via `ausyscall <ARCH> <NUM>` and whitelist it in the seccomp profile.
2. **`--privileged` container bypasses Seccomp entirely**:
   - Cause: Running `docker run --privileged` or `privileged: true` disables Seccomp completely.
   - Solution: Avoid `--privileged`. Instead, grant only the specific Linux Capabilities required via `--cap-add`.
3. **Seccomp performance overhead**:
   - Fact: Seccomp-BPF uses a linear instruction list or radix-tree evaluation in kernel memory; overhead is negligible (< 1-2% for syscall-heavy workloads).

---

## 8. Key Takeaways
- Seccomp restricts which system calls a containerized process can invoke on the host kernel.
- Docker and containerd enforce a default profile blocking ~44 high-risk syscalls (e.g. `reboot`, `mount`, `ptrace`).
- `SCMP_ACT_ERRNO` gracefully rejects calls with error codes, while `SCMP_ACT_LOG` enables non-disruptive profiling.
- In Kubernetes, configure `seccompProfile.type: RuntimeDefault` on all Pod specs to meet CIS Benchmarks.
