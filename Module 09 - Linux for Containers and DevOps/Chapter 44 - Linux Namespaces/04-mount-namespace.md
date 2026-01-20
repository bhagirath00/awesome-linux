# Mount Namespace (CLONE_NEWNS)

## 1. Architectural Concept & Deep Dive

The **Mount Namespace** (`CLONE_NEWNS`) was the very first namespace introduced into the Linux kernel (Linux 2.4.19, 2002). Because it was the first, developers named its flag `CLONE_NEWNS` (short for "New Namespace") rather than `CLONE_NEWMOUNT`.

A Mount namespace provides an isolated list of filesystem mount points for a process tree. File operations such as mounting (`mount(2)`), unmounting (`umount2(2)`), and altering mount flags inside one namespace do not affect the host mount table or other mount namespaces—provided mount propagation flags are configured properly.

### Why chroot(2) Is Insufficient for Container Runtimes
Historically, Unix systems used `chroot(2)` to jail a process inside a subfolder:
- **Root Escape Vector**: A root process inside a chroot jail can easily break out by creating another chroot or opening file descriptors outside the jail (`fchdir` breakout).
- **Global Mount Pollution**: Any mount or remount operation executed inside `chroot` alters global host state.
- **Shared `/proc` and `/sys`**: Exposing `/proc` leaks host process IDs, kernel knobs, and hardware state unless mounted independently in an isolated mount namespace.

Container runtimes (`runc`, `containerd`, `crio`) construct an isolated mount namespace, mount a clean root filesystem (`rootfs`), and execute `pivot_root(2)` to swap the root filesystem securely.

```
+-----------------------------------------------------------------------+
|                              HOST OS                                  |
|  Mount Table:                                                         |
|    /dev/sda1 -> / (rootfs)                                            |
|    tmpfs     -> /run                                                  |
|    proc      -> /proc                                                 |
|                                                                       |
|  +-----------------------------+     +-----------------------------+  |
|  |   Default Host Mount NS     |     | Container Mount Namespace   |  |
|  |                             |     |                             |  |
|  |  /                          |     |  / (merged rootfs)          |  |
|  |  +-- /var/lib/docker/...    |     |  +-- /bin                   |  |
|  |  +-- /home                  |     |  +-- /etc                   |  |
|  |  +-- /etc                   |     |  +-- /proc (isolated procfs)|  |
|  |  +-- /proc                  |     |  +-- /dev (container devpts)|  |
|  +-----------------------------+     +-----------------------------+  |
+-----------------------------------------------------------------------+
```

---

## 2. Mount Propagation Types

Mount propagation governs how mount and unmount events in one namespace replicate to peer or child mount namespaces. Defined in `Documentation/filesystems/sharedsubtree.txt`:

| Propagation Type | Flag / Setting | Replication Behavior | Common Use Case |
| :--- | :--- | :--- | :--- |
| **Shared** | `MS_SHARED` (`--make-shared`) | Mount/unmount events propagate bidirectional: host to namespace, and namespace to host. | Systemd root mounts, CSI plugins publishing storage back to host. |
| **Slave** | `MS_SLAVE` (`--make-slave`) | Unidirectional propagation: host mount events appear in container, but container mounts do NOT affect host. | Read-only shared volumes, host storage syncing into container. |
| **Private** | `MS_PRIVATE` (`--make-private`) | No propagation in either direction. Complete mount isolation. | Default container execution boundary (`runc`). |
| **Unbindable** | `MS_UNBINDABLE` (`--make-unbindable`) | Private mount that cannot be duplicated via `--bind`. | Preventing runaway recursive bind mount nesting. |

```
    Host Mount (Shared)
          │  ▲
          │  │ (Bidirectional replication)
          ▼  │
    Peer Mount (Shared)
          │
          │ (Unidirectional propagation only)
          ▼
    Slave Mount (Slave)
          │
          X (No propagation)
          ▼
    Private Mount (Isolated)
```

---

## 3. Hands-on Terminal Lab: Creating an Isolated Root

### Step 1: Create an Isolated Namespace with Private Propagation
By default, modern systemd sets `/` as `shared`. To prevent container experiments from leaking to the host, we unshare and make mounts private:

```bash
# Spawn bash inside a new Mount and PID namespace
sudo unshare --mount --pid --fork /bin/bash

# Inside the new namespace, mark all existing mounts private
mount --make-rprivate /
```

### Step 2: Inspect Mountinfo
Verify mount propagation settings via `/proc/self/mountinfo`:

```bash
cat /proc/self/mountinfo | grep -E " / " | head -n 1
```

```
42 27 8:1 / / rw,relatime shared:1 - ext4 /dev/sda1 rw,errors=remount-ro
```

When private:
```
42 27 8:1 / / rw,relatime - ext4 /dev/sda1 rw,errors=remount-ro
```
*(Notice the absence of `shared:1` tag).*

### Step 3: Temporary In-Memory File System Test
Create a mount inside the namespace and verify host invisibility:

```bash
# Inside namespace:
mkdir -p /mnt/ns-secret
mount -t tmpfs tmpfs /mnt/ns-secret
touch /mnt/ns-secret/isolated_payload.txt
ls -l /mnt/ns-secret/
```

```
total 0
-rw-r--r-- 1 root root 0 Oct 14 06:10 isolated_payload.txt
```

Now inspect from the host in a separate terminal:
```bash
# From host:
ls -l /mnt/ns-secret/
```
```
total 0
# Directory is empty; host cannot see the tmpfs mounted inside the namespace!
```

---

## 4. pivot_root(2) vs chroot(2)

Container runtimes use `pivot_root(2)` instead of `chroot(2)` to establish the container root filesystem.

### Comparison Table

| Feature | `chroot(2)` | `pivot_root(2)` |
| :--- | :--- | :--- |
| **Mechanism** | Changes current process root directory pointer (`fs->root`). | Atomically moves current root mount to `old_root` and makes `new_root` the new root mount. |
| **Old Root Access** | Old root remains accessible via open file descriptors or relative path traversal (`..`). | Old root is explicitly unmounted and detached (`umount -l /oldroot`). |
| **Filesystem Boundaries** | Does not alter mount tables. | Re-anchors mount points at kernel VFS mount structure. |
| **Security Jail Strength** | Easily escaped by UID 0. | Impenetrable via standard VFS path resolution; no parent traversal. |

### The pivot_root Workflow Implemented by Container Runtimes

```bash
# 1. Prepare target root directory containing rootfs
TARGET_ROOT="/var/lib/containers/alpine_root"
OLD_ROOT="$TARGET_ROOT/.oldroot"
mkdir -p "$OLD_ROOT"

# 2. TARGET_ROOT must be a mount point for pivot_root to succeed
mount --bind "$TARGET_ROOT" "$TARGET_ROOT"

# 3. Enter isolated mount namespace
cd "$TARGET_ROOT"

# 4. Swap roots: new_root becomes /, old host root placed into .oldroot
pivot_root . .oldroot

# 5. Mount necessary virtual filesystems
mount -t proc proc /proc
mount -t sysfs sys /sys
mount -t devtmpfs dev /dev

# 6. Detach and remove old host root completely
umount -l /.oldroot
rmdir /.oldroot

# 7. Execute process inside hardened container root
exec /bin/sh
```

---

## 5. Kubernetes & Production Scenarios

### Kubernetes MountPropagation Settings
In Kubernetes pod specs, `volumeMounts` support `mountPropagation`:
- `None` (Default): Equivalent to `MS_PRIVATE`. Container cannot receive mounts created by host, host cannot see container mounts.
- `HostToContainer`: Equivalent to `MS_SLAVE`. Container receives newly created mounts under that volume directory from the host.
- `Bidirectional`: Equivalent to `MS_SHARED`. Container can mount volumes (e.g., CSI storage plugins) that become visible on the host node.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: csi-driver-node
spec:
  containers:
  - name: csi-node-driver
    image: registry.k8s.io/sig-storage/csi-node-driver:v1.28.0
    securityContext:
      privileged: true
    volumeMounts:
    - name: mountpoint-dir
      mountPath: /var/lib/kubelet/pods
      mountPropagation: Bidirectional
  volumes:
  - name: mountpoint-dir
    hostPath:
      path: /var/lib/kubelet/pods
      type: Directory
```

---

## 6. Verification & Troubleshooting Runbook

### Diagnosis Commands
```bash
# View all mountpoints of a target container PID
cat /proc/<PID>/mounts

# View detailed propagation flags and peer groups
cat /proc/<PID>/mountinfo

# Detect leaked host mounts inside container namespace
grep -E "(ext4|xfs|btrfs)" /proc/<PID>/mounts
```

### Common Pitfalls & Solutions
1. **`pivot_root: Invalid argument (EINVAL)`**:
   - Cause: `new_root` or `old_root` is not a mountpoint, or `new_root` is on the same filesystem as current root.
   - Solution: Perform `mount --bind "$NEW_ROOT" "$NEW_ROOT"` before calling `pivot_root`.
2. **Mount leaking to host**:
   - Cause: Systemd default is `shared:1`. Mounts created inside an unshared mount namespace propagate back to the host.
   - Solution: Run `mount --make-rprivate /` immediately upon creating the mount namespace.
3. **CSI Driver fails with "propagation must be bidirectional"**:
   - Cause: Pod lacks `privileged: true` or Docker daemon's host root is not shared.
   - Solution: Set `mountPropagation: Bidirectional` and grant container `privileged: true`.

---

## 7. Key Takeaways
- The Mount namespace (`CLONE_NEWNS`) isolates filesystem mount tables per process hierarchy.
- Propagation flags (`shared`, `slave`, `private`, `unbindable`) control how mount events travel between namespaces.
- `pivot_root` replaces the root filesystem entirely and allows unmounting the host root, preventing breakout attacks that plague `chroot`.
- In Kubernetes, `mountPropagation: Bidirectional` is mandatory for CSI storage plugins to export node mounts to the host kubelet.
