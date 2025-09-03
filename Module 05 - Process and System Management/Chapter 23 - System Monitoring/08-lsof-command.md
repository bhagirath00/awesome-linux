# Chapter 23: System Monitoring - The lsof Command (List Open Files)

## 1. Introduction: "Everything in Linux is a File"

In UNIX and Linux design philosophy, **"everything is a file"**. 

Regular text files, directories, block devices (`/dev/sda`), raw hardware, named pipes, dynamic UNIX domain sockets, and even active TCP/UDP network connections are all represented internally as **File Descriptors (FDs)** assigned to processes.

`lsof` (**List Open Files**) is the utility designed to inspect, query, and trace these open file descriptors across every process running on the system.

---

## 2. Command Output Layout

Running `lsof` displays a comprehensive table of open handles:

```bash
sudo lsof -n -P | head -n 5
```

Flags explained:
- `-n`: Inhibits conversion of IP addresses to hostnames (much faster, no DNS lookups).
- `-P`: Inhibits conversion of port numbers to service names (displays `80` instead of `http`).

```
COMMAND     PID   USER   FD   TYPE             DEVICE SIZE/OFF    NODE NAME
systemd       1   root  cwd    DIR                8,1     4096       2 /
systemd       1   root  rtd    DIR                8,1     4096       2 /
systemd       1   root  txt    REG                8,1  1842000 1048590 /lib/systemd/systemd
sshd        890   root    3u  IPv4              24120      0t0     TCP *:22 (LISTEN)
nginx      1420 www-data  6u  IPv4              35100      0t0     TCP *:80 (LISTEN)
```

### Column Meanings:
- **`COMMAND`**: Executable binary name.
- **`PID`**: Process ID owning the open file descriptor.
- **`USER`**: Operating system user executing the process.
- **`FD`**: File Descriptor identifier:
  - `cwd`: Current working directory.
  - `rtd`: Root directory.
  - `txt`: Program executable code in memory.
  - `mem`: Memory-mapped library (`.so`).
  - `0u`, `1u`, `2u`: Standard Input (0), Output (1), Error (2) (`u` = read/write, `r` = read-only, `w` = write-only).
- **`TYPE`**: `REG` (Regular file), `DIR` (Directory), `CHR` (Character device), `FIFO` (Pipe), `IPv4`/`IPv6` (Network socket), `unix` (Domain socket).
- **`NAME`**: The target file path, device node, or network socket (`IP:PORT`).

---

## 3. High-Value Production Use Cases

### 1. Identifying Which Process is Listening on a Port
When a service fails to start because a port is already bound (`bind: address already in use`):
```bash
# Find what is listening on port 80 or 8080
sudo lsof -i :80
sudo lsof -i :8080
```

```
COMMAND   PID     USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
nginx    1420 www-data    6u  IPv4  35100      0t0  TCP *:80 (LISTEN)
```

### 2. Inspecting All Network Connections for a Process or User
```bash
# List all active network sockets for process "postgres"
sudo lsof -i -c postgres

# List all open files for a specific user
sudo lsof -u devops
```

### 3. Finding Why a Mount Point Cannot Be Unmounted
When attempting to unmount an external drive, USB, or NFS share:
```bash
sudo umount /mnt/backup
```
```
umount: /mnt/backup: target is busy.
```

Find out which user or process has an open file or active directory inside `/mnt/backup`:
```bash
sudo lsof +D /mnt/backup
```

```
COMMAND   PID   USER  FD   TYPE DEVICE SIZE/OFF NODE NAME
bash     3410 devops cwd    DIR    8,2     4096    2 /mnt/backup/project
```
Process `3410` has `/mnt/backup/project` as its active working directory (`cwd`). Once that shell leaves or is terminated, the drive unmounts cleanly!

### 4. Recovering Deleted Files That Are Still Open
If an important configuration or database file was accidentally deleted with `rm`, but the process that created it is still running, `lsof` can locate the file descriptor in `/proc`, allowing you to copy it back!

```bash
# 1. Locate the deleted open file
sudo lsof | grep -i "deleted" | grep config.json
```
```
my-app 18450 devops 5r REG 8,1 14205 92140 /opt/app/config.json (deleted)
```

```bash
# 2. Recover the data directly from the /proc filesystem:
cp /proc/18450/fd/5 /opt/app/config.json.recovered
```
Your deleted file is completely recovered!

---

## 4. Useful Flags Reference Table

| Flag | Function | Example |
| :--- | :--- | :--- |
| `-i [proto][@host][:port]` | Filter network sockets | `lsof -i :443` or `lsof -i tcp` |
| `-p <PID>` | Filter by single Process ID | `lsof -p 1420` |
| `-u <user>` | Filter by username | `lsof -u nginx` |
| `-c <command>` | Filter by executable name | `lsof -c python` |
| `+D <dir>` | Search directory hierarchy recursively | `lsof +D /var/log` |
| `+L1` | List open files with 0 links (deleted) | `lsof +L1` |
| `-n -P` | Skip DNS and port name lookups | `lsof -n -P` (Much faster) |

---

## 5. Summary Takeaways

- In Linux, all sockets, pipes, and devices are file descriptors inspected via `lsof`.
- Use `lsof -i :<port>` to resolve "address already in use" port conflicts.
- Use `lsof +D <mount>` to fix "target is busy" unmount errors.
- Use `lsof +L1` to detect ghost open files holding onto disk space.
- Recover accidentally deleted files by copying them out of `/proc/<PID>/fd/<FD>`.
