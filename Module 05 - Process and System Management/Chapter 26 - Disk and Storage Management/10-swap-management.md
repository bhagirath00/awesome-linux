# Chapter 26: Disk and Storage Management - Swap Space Management

## 1. Introduction: What is Swap Space?

**Swap space** is an allocated portion of secondary storage (a dedicated raw disk partition or a continuous pre-allocated file) that the Linux kernel uses as an overflow extension for physical random-access memory (RAM).

When system memory approaches saturation, the kernel's memory management subsystem transfers inactive, dormant memory pages (anonymous memory allocated by background daemons or idle processes) out of fast physical RAM and writes them into swap space. This process—called **paging out**—frees physical RAM pages to be utilized by active applications and kernel disk page caches.

```
+--------------------------------------------------------------------------+
|                        Linux Virtual Memory & Swap                       |
+--------------------------------------------------------------------------+
|  Active Processes (High CPU)       Dormant Processes (Idle)              |
|  +---------------------------+     +---------------------------+         |
|  | Web Server / Worker Nodes |     | Idle Background Daemons   |         |
|  +-------------+-------------+     +-------------+-------------+         |
|                |                                 |                       |
|                v                                 v                       |
|  +---------------------------+     +---------------------------+         |
|  | Physical RAM (High Speed) |     | Paged Out to Disk / Swap  |         |
|  | [ Active Working Set ]    |     | [ Dormant Pages on NVMe ] |         |
|  +---------------------------+     +---------------------------+         |
+--------------------------------------------------------------------------+
```

---

## 2. Dedicated Swap Partition vs. Swap File

| Feature | Swap Partition | Swap File |
| :--- | :--- | :--- |
| **Storage Structure** | Dedicated raw disk partition (`/dev/sda3`) | Normal pre-allocated file inside root (`/swapfile`) |
| **Resizing Flexibility** | Hard; requires re-partitioning or LVM resize | **Extremely Easy**; delete and recreate or add a 2nd file |
| **Performance** | Historically faster on old mechanical HDDs | **Identical performance** on modern SSD/NVMe drives |
| **Cloud / VM Standard** | Less common | **Industry standard on AWS, GCP, Azure, DigitalOcean** |

---

## 3. Creating and Activating a Swap File (Step-by-Step)

Here is the production standard workflow for adding a 4GB swap file to a cloud instance:

### Step 1: Pre-allocate the File with `fallocate`
`fallocate` creates a 4GB file instantly without writing zeros:
```bash
sudo fallocate -l 4G /swapfile
```
*(If `fallocate` is unsupported on your filesystem, fallback to `dd`: `sudo dd if=/dev/zero of=/swapfile bs=1M count=4096 status=progress`)*.

### Step 2: Set Strict File Permissions (CRITICAL SECURITY STEP)
A swap file contains raw dumps of process memory, which can include cleartext passwords, SSL private keys, and user sessions. **Only `root` must be permitted to read or write to it**:
```bash
sudo chmod 600 /swapfile
```

### Step 3: Format the File as Linux Swap
```bash
sudo mkswap /swapfile
```
```
Setting up swapspace version 1, size = 4 GiB (4294963200 bytes)
no label, UUID=7a82b412-9812-4210-b124-c12481023941
```

### Step 4: Enable the Swap Space
```bash
sudo swapon /swapfile
```

### Step 5: Verify Active Swap
```bash
swapon --show
# Or via free:
free -h
```

```
NAME      TYPE SIZE USED PRIO
/swapfile file   4G   0B   -2
```

### Step 6: Make Swap Permanent Across Reboots
Add the entry to `/etc/fstab`:
```bash
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

---

## 4. Disabling and Deleting Swap

If you need to resize or remove a swap file (e.g., preparing a Kubernetes node):

```bash
# 1. Deactivate the swap space (pages are moved back into RAM)
sudo swapoff /swapfile

# 2. Remove the line from /etc/fstab
sudo sed -i '\#/swapfile#d' /etc/fstab

# 3. Delete the file from disk
sudo rm -f /swapfile
```

---

## 5. Kernel Swappiness Tuning (`vm.swappiness`)

The `vm.swappiness` kernel sysctl parameter controls the balance between reclaiming page cache memory vs. swapping out application anonymous memory:

- Scale: **`0` to `100`** (Default is typically `60`).
- **`0`**: Kernel avoids swapping aggressively, paging out only to avoid an Out-of-Memory (OOM) crash.
- **`10` - `20`**: **Recommended for production databases and high-performance servers** (favors keeping application memory in RAM).
- **`60`**: Balanced desktop/general-purpose default.
- **`100`**: Aggressive swapping (swaps out application pages to prioritize disk cache).

### Checking and Setting Swappiness:
```bash
# Check current value
cat /proc/sys/vm/swappiness
```
```
60
```

```bash
# Set swappiness to 10 temporarily
sudo sysctl vm.swappiness=10

# Set swappiness to 10 permanently across reboots
echo "vm.swappiness=10" | sudo tee /etc/sysctl.d/99-swappiness.conf
sudo sysctl --system
```

---

## 6. Swap and Kubernetes Nodes

> [!IMPORTANT]
> By default, the Kubernetes **`kubelet` agent refuses to start if swap is enabled** on a node (`[ERROR Swap]: running with swap on is not supported`). In Kubernetes environments, disable swap completely with:
> ```bash
> sudo swapoff -a
> sudo sed -i '/swap/s/^/#/' /etc/fstab
> ```

---

## 7. Summary Takeaways

- Swap acts as an overflow buffer for physical RAM, holding dormant memory pages.
- Swap files (`/swapfile`) offer superior flexibility compared to fixed swap partitions.
- Always enforce `chmod 600` on swap files to prevent memory snooping.
- Persist swap mounts in `/etc/fstab` with `none swap sw 0 0`.
- Tune `vm.swappiness=10` on production servers to prioritize keeping active applications in RAM.
- Disable swap completely when configuring Kubernetes worker nodes.
