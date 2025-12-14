# Chapter 24: Service Management - The systemctl Command

## 1. Introduction: What is systemctl?

`systemctl` is the central command-line utility used to examine and control the state of the `systemd` system and service manager. 

Whether you need to start, stop, restart, enable, mask, or inspect services, timers, mount points, or system targets, all operations are executed through `systemctl`.

---

## 2. Basic Command Syntax and Conventions

**Syntax:**
```bash
systemctl [command] [unit_name]
```

> [!NOTE]
> If you omit the unit extension for a service, `systemctl` automatically assumes `.service`. For example, `systemctl status nginx` is completely identical to `systemctl status nginx.service`.

---

## 3. Core Service Lifecycle Commands

### Starting, Stopping, and Restarting Services
```bash
# Start a service immediately
sudo systemctl start nginx

# Stop an active service
sudo systemctl stop nginx

# Restart a service (terminates process and starts fresh instance)
sudo systemctl restart nginx

# Reload configuration without restarting process or dropping active sessions
sudo systemctl reload nginx

# Graceful reload if supported; otherwise fallback to restart
sudo systemctl reload-or-restart nginx
```

---

## 4. Boot-Time Persistence: Enable vs. Start

A frequent point of confusion for beginners is the difference between **`start`** and **`enable`**:

- **`start` / `stop`**: Controls the **immediate runtime state** in RAM. It does not affect whether the service starts when the machine reboots.
- **`enable` / `disable`**: Controls the **boot-time persistence** by creating or removing symbolic links in `/etc/systemd/system/*.wants/`. It does not start the service right now.

```
+--------------------------------------------------------------------------+
|                       start vs. enable in systemd                        |
+--------------------------------------------------------------------------+
|  Command                       Immediate State?      Starts on Next Boot?|
+--------------------------------------------------------------------------+
|  systemctl start nginx         Active (Running)      No                  |
|  systemctl enable nginx        No change (Inactive)  Yes                 |
|  systemctl enable --now nginx  Active (Running)      Yes                 |
|  systemctl disable nginx       No change (Running)   No                  |
+--------------------------------------------------------------------------+
```

> [!TIP]
> Use the **`--now`** flag to combine both actions into a single command:
> ```bash
> # Enable to start on boot AND start immediately right now:
> sudo systemctl enable --now nginx
> 
> # Disable from boot AND stop immediately right now:
> sudo systemctl disable --now nginx
> ```

---

## 5. Querying and Checking Status

```bash
# Comprehensive status report (PID, memory, cgroup tree, recent logs)
systemctl status nginx
```

```
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2026-10-14 05:12:00 UTC; 2h 15min ago
       Docs: man:nginx(8)
   Main PID: 1420 (nginx)
      Tasks: 3 (limit: 4642)
     Memory: 8.2M
        CPU: 1.240s
     CGroup: /system.slice/nginx.service
             ├─1420 "nginx: master process /usr/sbin/nginx -g daemon on; master_process on;"
             ├─1421 "nginx: worker process"
             └─1422 "nginx: worker process"

Oct 14 05:12:00 web01 systemd[1]: Starting A high performance web server...
Oct 14 05:12:00 web01 systemd[1]: Started A high performance web server.
```

### Checking Boolean States in Automation Scripts:
```bash
# Is the service currently executing? (Returns exit code 0 if active)
systemctl is-active nginx

# Is the service configured to start on boot?
systemctl is-enabled nginx

# Has the service failed?
systemctl is-failed nginx
```

---

## 6. Masking Units: The Ultimate Override

Sometimes you must guarantee that an unwanted service (e.g., `cups` printer daemon, `apache2` when migrating to `nginx`) is **never started under any circumstance**—neither manually, nor by another dependent service, nor on boot.

Disabling a service (`systemctl disable`) only removes boot links; another service can still trigger it. **Masking** permanently neutralizes it by linking the unit file to `/dev/null`:

```bash
# Mask the service completely
sudo systemctl mask apache2
```

```
Created symlink /etc/systemd/system/apache2.service -> /dev/null.
```

If anyone attempts to start it:
```bash
sudo systemctl start apache2
```
```
Failed to start apache2.service: Unit apache2.service is masked.
```

To restore the service later:
```bash
sudo systemctl unmask apache2
```

---

## 7. Reloading the systemd Daemon (`daemon-reload`)

Whenever you create a new custom service unit file or modify an existing file in `/etc/systemd/system/`, `systemd` continues using its in-memory cached copy until you instruct it to re-parse the configuration directories:

```bash
sudo systemctl daemon-reload
```

---

## 8. Summary Takeaways

- `systemctl start/stop` controls immediate RAM state; `enable/disable` controls boot links.
- Use `systemctl enable --now <unit>` for clean single-step deployment.
- Check state cleanly in bash scripts using `is-active` and `is-enabled`.
- Use `systemctl mask` to link units to `/dev/null` preventing rogue launches.
- Always execute `sudo systemctl daemon-reload` after altering unit files.
