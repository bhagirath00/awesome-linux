# Chapter 44: Linux Namespaces - What are Linux Namespaces?

## 1. The Core Secret of Containers: "Containers Do Not Exist"

One of the most foundational realizations in modern cloud and DevOps engineering is that **there is no such thing as a "container" inside the Linux kernel**. 

Unlike a Virtual Machine (which virtualizes physical CPU, RAM, and BIOS via a hypervisor like KVM or VMware), a container is simply **a standard Linux process running directly on the host kernel**, but constrained and isolated by two foundational kernel primitives:
1. **Linux Namespaces**: Control **WHAT a process can SEE** (Isolation / Virtual Views).
2. **Control Groups (cgroups)**: Control **HOW MUCH a process can USE** (Resource Limiting).

```
+--------------------------------------------------------------------------+
|                  Virtual Machines vs. Linux Containers                   |
+--------------------------------------------------------------------------+
|  Virtual Machine (Hardware Virtualization):                              |
|  [ App A ] [ App B ]                                                     |
|  [ Guest OS Kernel (vmlinuz) ]  <-- Duplicate kernel! High RAM / slow boot|
|  [ Hypervisor (KVM / QEMU) ]                                             |
|  [ Host Hardware ]                                                       |
+--------------------------------------------------------------------------+
|  Linux Container (Kernel Primitive Virtualization):                      |
|  [ Container 1 (PID 1)]  [ Container 2 (PID 1)]  [ Host Process (PID 94)]|
|  (Namespace Sandbox 1)   (Namespace Sandbox 2)   (Root Namespace)        |
|  +--------------------------------------------------------------------+  |
|  |           ONE SHARED LINUX HOST KERNEL (Namespaces + Cgroups)      |  |
|  +--------------------------------------------------------------------+  |
|  [ Host Hardware (Bare Metal / Cloud VM) ]                               |
+--------------------------------------------------------------------------+
```

---

## 2. The 8 Linux Kernel Namespaces

A **Namespace** wraps a global system resource in an abstraction that makes it appear to the processes within the namespace that they possess their own isolated instance of that resource.

| Namespace | CLI Flag (`unshare`) | Kernel System Call Flag | What it Isolates / Provides |
| :--- | :--- | :--- | :--- |
| **PID** | `--pid` | `CLONE_NEWPID` | Process IDs. Allows the container process to become **PID 1**. |
| **Network (NET)**| `--net` | `CLONE_NEWNET` | Network devices, IP addresses, routing tables, port bindings. |
| **Mount (MNT)** | `--mount` | `CLONE_NEWNS` | Filesystem mount points and directory trees. |
| **User (USER)** | `--user` | `CLONE_NEWUSER`| UID/GID mappings (e.g., container `root` is unprivileged on host).|
| **UTS** | `--uts` | `CLONE_NEWUTS` | System hostname and NIS domain name. |
| **IPC** | `--ipc` | `CLONE_NEWIPC` | Inter-Process Communication (POSIX message queues, shared memory). |
| **Cgroup** | `--cgroup` | `CLONE_NEWCGROUP`| Cgroup root directory view inside `/proc/self/cgroup`. |
| **Time** | `--time` | `CLONE_NEWTIME`| System clock offsets (monotonic and boot clocks). |

---

## 3. How the Kernel Represents Namespaces: `/proc/<PID>/ns/`

Every process running on a Linux machine has a dedicated subdirectory at **`/proc/<PID>/ns/`** containing symbolic links representing its membership in each namespace:

```bash
ls -l /proc/$$/ns/
```

```
total 0
lrwxrwxrwx 1 devops devops 0 Oct 14 06:30 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 devops devops 0 Oct 14 06:30 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 devops devops 0 Oct 14 06:30 mnt -> 'mnt:[4026531840]'
lrwxrwxrwx 1 devops devops 0 Oct 14 06:30 net -> 'net:[4026531992]'
lrwxrwxrwx 1 devops devops 0 Oct 14 06:30 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 devops devops 0 Oct 14 06:30 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx 1 devops devops 0 Oct 14 06:30 time -> 'time:[4026531834]'
lrwxrwxrwx 1 devops devops 0 Oct 14 06:30 time_for_children -> 'time:[4026531834]'
lrwxrwxrwx 1 devops devops 0 Oct 14 06:30 user -> 'user:[4026531837]'
lrwxrwxrwx 1 devops devops 0 Oct 14 06:30 uts -> 'uts:[4026531838]'
```

The numbers inside the brackets (e.g., `net:[4026531992]`) are **inode identifiers**. 
- If Process A and Process B share the same inode number for `net`, they share the **same network namespace** (can communicate over `localhost:8080`).
- If their inode numbers differ, they are in completely separate, isolated network namespaces!

---

## 4. The Two Fundamental Commands: `unshare` and `nsenter`

The Linux `util-linux` package provides two CLI tools to manipulate namespaces directly without Docker:

### 1. `unshare`: Create a New Namespace
Spawns a new program running inside a brand-new, isolated namespace:
```bash
# Spawn a shell with its own private UTS (hostname) namespace
sudo unshare --uts /bin/bash

# Inside the new namespace:
hostname container-isolated
hostname
# Output: container-isolated

exit # Exit namespace

# On the host:
hostname
# Output: original-server-hostname (Host was untouched!)
```

### 2. `nsenter`: Enter an Existing Container's Namespace
Allows an engineer or debugger to inject a terminal shell directly into the namespace of a running Docker container or Kubernetes pod:

```bash
# Find container PID
CONTAINER_PID=$(docker inspect --format '{{ .State.Pid }}' my-web-app)

# Enter the container's Network namespace using host tools:
sudo nsenter --target "${CONTAINER_PID}" --net /bin/bash
```
Now, running `ip a` or `tcpdump` reveals the container's virtual network interface directly from the host!

---

## 5. Summary Takeaways

- Containers are standard Linux processes isolated by Namespaces and constrained by Cgroups.
- Namespaces determine **what a process can see** (PIDs, mounts, networks, users).
- Every process's namespace memberships are visible as inode links in `/proc/<PID>/ns/`.
- Use `unshare` to create new isolated namespaces manually.
- Use `nsenter` to inject debugging tools directly into active container namespaces.
