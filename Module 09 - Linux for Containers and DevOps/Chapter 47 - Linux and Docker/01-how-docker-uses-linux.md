# How Docker Uses the Linux Kernel: Deconstructing the Container Myth

## 1. Architectural Concept & Deep Dive

One of the most foundational realizations for any senior DevOps engineer is this:
> **Containers are not virtual machines. In fact, there is no such thing as a "container" in the Linux kernel.**

There is no `container_struct` in kernel memory, no `sys_create_container` system call, and no container hypervisor. A container is simply an **ordinary Linux process** running on the host kernel, wrapped in five distinct kernel isolation and security primitives:

1. **Linux Namespaces**: For isolation of process trees (PID), network interfaces (NET), mount tables (MNT), IPC, UTS, and User IDs.
2. **Control Groups (Cgroups v2)**: For metering and limiting physical hardware usage (CPU CFS quotas, RAM hard ceilings, Block I/O, PIDs).
3. **Union Filesystem (OverlayFS)**: For layered, copy-on-write image filesystems.
4. **Linux Security Primitives (Capabilities, Seccomp, AppArmor/SELinux)**: For restricting syscalls, dropping root powers, and confining mandatory access controls.
5. **Virtual Ethernet (veth) & Bridges**: For network routing and packet translation (iptables NAT).

```
+─────────────────────────────────────────────────────────────────────────+
|                           THE CONTAINER STACK                           |
|                                                                         |
|   1. Isolation: Namespaces (pid, net, mnt, ipc, uts, user)             |
|   2. Resource Caps: Cgroups (memory.max, cpu.max, pids.max)             |
|   3. Filesystem: OverlayFS (lowerdir [image] + upperdir [container])    |
|   4. Hardening: Capabilities (CapDrop) + Seccomp (Syscall Filter)       |
|   5. Networking: veth pair + bridge (docker0) + iptables MASQUERADE     |
|                                                                         |
|                                    │                                    |
|                                    ▼                                    |
|   +─────────────────────────────────────────────────────────────────+   |
|   |                  OCI Runtime (runc / crun)                      |   |
|   +────────────────────────────────┬────────────────────────────────+   |
|                                    │ clone(CLONE_NEWPID|NEWNS|NEWNET...)
|                                    ▼                                    |
|   +─────────────────────────────────────────────────────────────────+   |
|   |                       Standard Linux Kernel                     |   |
|   +─────────────────────────────────────────────────────────────────+   |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. The OCI Runtime Architecture: From Docker to runc

Docker does not execute containers directly. Modern container architecture adheres to the Open Container Initiative (OCI) specification:

```
[ Docker CLI / API ]
         │ (Docker Engine REST API)
         ▼
     [ dockerd ]
         │ (gRPC)
         ▼
    [ containerd ] ── (CRI plugin for Kubernetes)
         │ (gRPC)
         ▼
[ containerd-shim ]
         │ (forks / executes)
         ▼
       [ runc ] ── (OCI reference runtime: configures kernel primitives)
         │
         ▼
[ Container Process (PID 12450) ]
```

1. **`dockerd`**: High-level daemon handling image builds, volumes, CLI parsing, and network endpoints.
2. **`containerd`**: Complete container lifecycle supervisor (pulling images, unpacking rootfs, managing shims).
3. **`containerd-shim`**: Lightweight daemon remaining alive after `runc` exits to hold the container's standard I/O (stdin/stdout) file descriptors open and collect exit codes without keeping the main engine blocked.
4. **`runc`**: Low-level CLI tool that reads an OCI `config.json` bundle, executes the `clone()` syscall with specified namespace flags, attaches cgroups, applies seccomp filters, and pivots root.

---

## 3. Hands-on Lab: Building a Container from Scratch Using Pure Linux Primitives

To prove that a container is just Linux primitives, let's assemble a complete container manually using native shell commands.

### Step 1: Create a Root Filesystem (rootfs)
Download and extract a minimal Alpine rootfs:
```bash
mkdir -p /tmp/mycontainer/rootfs
cd /tmp/mycontainer
curl -sSL https://dl-cdn.alpinelinux.org/alpine/v3.18/releases/x86_64/alpine-minirootfs-3.18.4-x86_64.tar.gz | tar -xz -C rootfs/
```

### Step 2: Create Cgroups Limits for the Container
```bash
sudo mkdir -p /sys/fs/cgroup/mycontainer
echo "67108864" | sudo tee /sys/fs/cgroup/mycontainer/memory.max  # 64 MB Limit
echo "20" | sudo tee /sys/fs/cgroup/mycontainer/pids.max          # Max 20 PIDs
```

### Step 3: Launch Isolated Namespaces with `unshare`
We isolate PID, Mount, UTS (hostname), and IPC namespaces:
```bash
sudo unshare --mount --pid --uts --ipc --fork /bin/bash
```

*(You are now in a new namespace shell on the host!)*

### Step 4: Attach Shell to Cgroup
In another host terminal:
```bash
# Find PID of the unshare shell and attach to cgroup
SHELL_PID=$(pgrep -f "unshare.*mycontainer" | tail -n 1)
echo "$SHELL_PID" | sudo tee /sys/fs/cgroup/mycontainer/cgroup.procs
```

### Step 5: Pivot Root and Mount Virtual Filesystems
Back inside the namespace shell:
```bash
# Change hostname
hostname my-custom-container

# Prepare rootfs mount
cd /tmp/mycontainer/rootfs
mkdir -p oldroot
mount --bind /tmp/mycontainer/rootfs /tmp/mycontainer/rootfs
pivot_root . oldroot

# Mount private proc and sysfs
mount -t proc proc /proc
mount -t sysfs sys /sys
mount -t devtmpfs dev /dev

# Detach host oldroot
umount -l /oldroot
rmdir /oldroot

# Execute shell inside isolated root
exec /bin/sh
```

### Step 6: Verify the Handcrafted Container!
```sh
# Inside your handcrafted container:
hostname
# Output: my-custom-container

ps aux
# Output:
# PID   USER     TIME  COMMAND
#   1   root     0:00  /bin/sh
#   5   root     0:00  ps aux

cat /proc/1/cgroup
# Output: 0::/mycontainer
```
**You have built a working Linux container without Docker, containerd, or runc!**

---

## 4. How Docker Assembles the Bundle

When you run `docker run -d -p 80:80 --memory=256m nginx`, Docker:
1. Pulls the OCI image layers and mounts them using **OverlayFS** to create an ephemeral root directory.
2. Generates an OCI `config.json` specification detailing:
   - Namespace flags (`CLONE_NEWPID`, `CLONE_NEWNS`, `CLONE_NEWNET`, `CLONE_NEWIPC`, `CLONE_NEWUTS`).
   - Cgroup rules (`MemoryMax = 268435456`).
   - Dropped capabilities (`CAP_SYS_ADMIN`, `CAP_NET_ADMIN`, etc.).
   - Seccomp JSON filter.
3. Invokes `containerd`, which invokes `runc`.
4. Creates a `veth` pair, moves one end into the container network namespace as `eth0`, and attaches the other end to `docker0` bridge.
5. Injects iptables `DNAT` rule to map host port 80 to container IP port 80.

---

## 5. Verification & Troubleshooting Runbook

### Inspecting Docker's Kernel Attachments
```bash
# Find the host PID of a running container
docker inspect --format '{{.State.Pid}}' <CONTAINER_ID>

# View the container's exact namespaces
ls -l /proc/<HOST_PID>/ns/

# View the container's exact cgroup location
cat /proc/<HOST_PID>/cgroup

# View the container's filesystem mount layers
cat /proc/<HOST_PID>/mountinfo | grep overlay
```

### Common Pitfalls & Solutions
1. **Container PID 1 Zombie Reaping**:
   - Cause: PID 1 in Linux must reap orphaned child processes (`waitpid`). If an application (e.g. Node.js or Python) does not implement subreaping, zombie processes accumulate until the cgroup PID limit is reached.
   - Solution: Use `--init` flag in Docker (tini) or configure `shareProcessNamespace: true` in Kubernetes.
2. **"Docker is eating all my disk space"**:
   - Cause: Unused OverlayFS writable layers and build caches accumulated in `/var/lib/docker/overlay2`.
   - Solution: Run `docker system prune -a --volumes` to purge dangling layers.

---

## 6. Key Takeaways
- "Containers" are an illusion; they are standard host processes governed by Linux namespaces, cgroups, OverlayFS, capabilities, and seccomp.
- Docker is a high-level manager that delegates execution to `containerd` and `runc`.
- Any Linux engineer can create a functional container using standard CLI utilities (`unshare`, `pivot_root`, `cgroups`).
- Troubleshooting container failures requires mapping container symptoms directly back to host kernel subsystems.
