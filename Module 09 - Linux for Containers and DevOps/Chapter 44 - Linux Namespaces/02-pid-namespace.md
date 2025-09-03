# Chapter 44: Linux Namespaces - PID Namespaces and Process Isolation

## 1. Introduction: The Magic of Container PID 1

On a standard Linux host, Process IDs (PIDs) form a single, contiguous global namespace originating from systemd (PID 1) and expanding up to `/proc/sys/kernel/pid_max` (typically 4,194,304).

When you run an application inside a Docker container:
```bash
docker run -d --name web nginx
docker exec -it web ps aux
```
You see:
```
USER   PID %CPU %MEM COMMAND
root     1  0.0  0.2 nginx: master process /usr/sbin/nginx
```
Inside the container, Nginx is **PID 1**. Yet, if you run `ps aux | grep nginx` on the **host machine**, Nginx has an entirely different PID, such as `28412`!

This illusion is created by the **PID Namespace**.

```
+--------------------------------------------------------------------------+
|                     Nested PID Namespace Hierarchy                       |
+--------------------------------------------------------------------------+
|  Host Root PID Namespace (Level 0)                                       |
|  ├── systemd (PID: 1)                                                    |
|  ├── dockerd (PID: 980)                                                  |
|  └── nginx   (PID: 28412 on Host!)                                       |
|                    |                                                     |
|                    v Container PID Namespace (Level 1)                   |
|              +---------------------------------------------------------+ |
|              | Container Process Table:                                | |
|              | └── nginx (PID: 1 inside container!)                    | |
|              +---------------------------------------------------------+ |
+--------------------------------------------------------------------------+
```

---

## 2. PID Namespace Hierarchy and Nested Translation

PID namespaces form a parent-child hierarchy:
1. **Children are Invisible to Peers**: A process inside Container A cannot see, inspect, or send signals (`kill`) to processes in Container B or on the host.
2. **Parents See Everything**: The parent (host) namespace can see all child processes, but maps them to different PID integers.
3. **Nested Translation**: A process has multiple PIDs simultaneously: one for each namespace ancestor level up to the root host namespace.

---

## 3. Creating a Custom PID Namespace with `unshare`

To observe this firsthand without Docker, spawn a shell inside a new PID and Mount namespace:

```bash
# --pid: New PID namespace
# --mount-proc: Mounts a fresh /proc filesystem matching the new PID space
# --fork: Forks child process so it receives PID 1
sudo unshare --pid --mount-proc --fork /bin/bash
```

Inside the new shell:
```bash
# Check current PID
echo "$$"
# Output: 1

# List running processes
ps aux
```
```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.1  10240  4800 pts/0    S    06:35   0:00 /bin/bash
root         4  0.0  0.0  11400  3200 pts/0    R+   06:35   0:00 ps aux
```
Only two processes exist! The hundreds of host processes are completely invisible!

---

## 4. The Critical Role and Responsibilities of PID 1

Being PID 1 in a Linux namespace carries unique kernel responsibilities:

### 1. Signal Immunity
By default, the Linux kernel protects PID 1 from accidental termination. **PID 1 ignores all signals (including `SIGTERM` and `SIGINT`) unless the process explicitly registers a custom signal handler!**
- If your containerized Node.js, Python, or Go app runs as PID 1 without a custom signal handler for `SIGTERM`, running `docker stop` will hang for 10 seconds before being forcefully killed with `SIGKILL`!

### 2. Zombie Reaping
When a parent process dies before its child, the child becomes an orphan. The kernel reparents the orphan to **PID 1**. PID 1 is responsible for listening for `SIGCHLD` and harvesting the child's exit status with `waitpid()`.
- If PID 1 does not reap child processes, the container's PID table fills with unkillable **Zombie (`<defunct>`)** processes!

### The Solution: Tini / Dumb-init
In Dockerfiles, use an ultra-lightweight init wrapper like **`tini`** to serve as PID 1 and properly reap zombies and forward signals:
```dockerfile
# Add Tini
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["node", "server.js"]
```

---

## 5. Summary Takeaways

- PID namespaces isolate process tables, allowing containers to see their application as PID 1.
- Processes have multiple PIDs: one inside the container, another on the host.
- Parent namespaces can monitor and terminate child namespace processes.
- PID 1 in containers must handle signal forwarding and orphan zombie reaping (use `tini`).
