# Troubleshooting Systemd Service Failures & Crash Loops

## 1. Architectural Concept & Deep Dive

In modern Linux, **Systemd** (PID 1) acts as the init system, process supervisor, and cgroup manager. When a daemon crashes, fails to bind a socket, or enters a crash-restart loop, systemd records the exact lifecycle state transition, exit status, and signal code.

Understanding systemd's state machine is essential for quickly categorizing service failure modes:

```
                  +───────────────────────────+
                  |         Inactive          |
                  +─────────────┬─────────────+
                                │ Start Trigger
                                ▼
                  +───────────────────────────+
                  |    Activating (ExecStart) |
                  +──────┬─────────────┬──────+
           Success       │             │ Failure (Exit Code != 0)
     ┌───────────────────┘             └───────────────────┐
     ▼                                                     ▼
+──────────+                                         +──────────+
|  Active  | ─── Process Exit / Crash ──────────────►|  Failed  |
+──────────+                                         +────┬─────+
                                                          │ Restart=always
                                                          ▼
                                                     +──────────+
                                                     |Auto-Rest.|
                                                     +──────────+
                                                          │ RestartSec=
                                                          ▼
                                                     Crash Loop!
```

---

## 2. Deciphering Systemd Exit Codes

When inspecting `systemctl status <unit>`, the `Process: ... (code=exited, status=...)` line reveals the root cause:

| Exit Status / Code | Meaning | Common Root Cause |
| :--- | :--- | :--- |
| **`status=1/FAILURE`** | Application generic error | Configuration syntax error, unhandled exception, missing DB connection. |
| **`status=127`** | Command not found | `ExecStart` points to a non-existent binary or missing shared library (`.so`). |
| **`status=137` / `SIGKILL`** | Process killed via Signal 9 | **OOM Killer** terminated the service due to cgroup or host RAM exhaustion. |
| **`status=143` / `SIGTERM`** | Process cleanly terminated | Another administrator or automation ran `systemctl stop`. |
| **`status=203/EXEC`** | Execve failure | Binary lacks execute permission (`chmod +x`), or bad shebang line (`#!/bin/bash\r` DOS line endings). |
| **`status=217/USER`** | User lookup failure | `User=` specified in unit file does not exist in `/etc/passwd`. |

---

## 3. The "Works in My Shell, Fails in Systemd" Trap

A classic DevOps bug:
A developer logs into SSH, runs `/usr/local/bin/app`, and it executes flawlessly. They start it via `systemctl start app.service`, and it immediately crashes.

### Why?
Interactive user shells load profile scripts (`/etc/profile`, `~/.bashrc`) that export custom environment variables, `PATH` directories, and ulimits. **Systemd executes in a sterile, minimal environment**:
- Default `PATH`: `/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin`.
- Missing environment variables (`JAVA_HOME`, `DATABASE_URL`, `NODE_ENV`).
- Default working directory is root (`/`), not the application source directory.

### Remediation via Systemd Overrides:
Always pass explicit environment variables in the unit definition:

```ini
[Service]
WorkingDirectory=/opt/myapp
Environment="NODE_ENV=production"
EnvironmentFile=/etc/default/myapp
ExecStart=/usr/bin/node /opt/myapp/server.js
```

---

## 4. Hands-on Triage Runbook: Debugging a Failing Service

### Step 1: Query Unit Status
```bash
systemctl status api.service
```
```
● api.service - Core API Microservice
     Loaded: loaded (/etc/systemd/system/api.service; enabled)
     Active: failed (Result: exit-code) since Wed 2026-10-14 08:15:20 UTC; 12s ago
    Process: 28410 ExecStart=/usr/bin/python3 /opt/api/main.py (code=exited, status=1/FAILURE)
   Main PID: 28410 (code=exited, status=1/FAILURE)
```

### Step 2: Query Extended Journal Output (`-xeu`)
The `-xeu` flags display detailed catalog explanations, errors, and system context:
```bash
journalctl -xeu api.service --no-pager
```
```
Oct 14 08:15:19 node1 python3[28410]: Traceback (most recent call last):
Oct 14 08:15:19 node1 python3[28410]:   File "/opt/api/main.py", line 12, in <module>
Oct 14 08:15:19 node1 python3[28410]:     server.bind(("0.0.0.0", 80))
Oct 14 08:15:19 node1 python3[28410]: PermissionError: [Errno 13] Permission denied
Oct 14 08:15:20 node1 systemd[1]: api.service: Main process exited, code=exited, status=1/FAILURE
Oct 14 08:15:20 node1 systemd[1]: api.service: Failed with result 'exit-code'.
```
- **Root Cause Identified**: The service is running as an unprivileged user (`User=appuser`) and attempting to bind to privileged port 80 without `CAP_NET_BIND_SERVICE`!

### Step 3: Fix Safely via Systemd Drop-in Overrides (`systemctl edit`)
Never edit unit files directly under `/lib/systemd/system/` (package updates will overwrite them). Instead, create a drop-in override:

```bash
sudo systemctl edit api.service
```
Add the missing capability:
```ini
[Service]
AmbientCapabilities=CAP_NET_BIND_SERVICE
```

Reload and restart:
```bash
sudo systemctl daemon-reload
sudo systemctl restart api.service
systemctl is-active api.service
```
```
active
```

---

## 5. Breaking Out of Crash-Restart Loops (`start-limit-hit`)

When a service crashes repeatedly within a short window, systemd triggers rate-limiting to prevent CPU burn:

```
systemd[1]: api.service: Start request repeated too quickly.
systemd[1]: api.service: Failed with result 'start-limit-hit'.
```

### 1. Resetting the Failure Counter
To re-enable testing immediately:
```bash
sudo systemctl reset-failed api.service
```

### 2. Tuning Restart Backoffs
In production unit files, prevent rapid-fire restart loops by configuring exponential backoffs:
```ini
[Unit]
StartLimitIntervalSec=300
StartLimitBurst=5

[Service]
Restart=on-failure
RestartSec=10s
```

---

## 6. Verification & Troubleshooting Runbook

### Health & Diagnostic Commands
```bash
# List all failed units across the entire operating system
systemctl --failed

# Verify syntax of unit files without reloading
systemd-analyze verify /etc/systemd/system/api.service

# Inspect all effective runtime properties of a service
systemctl show api.service --property=ExecStart,User,Environment,Restart
```

### Common Pitfalls & Solutions
1. **`Failed to start: Unit is masked`**:
   - Cause: Service was symlinked to `/dev/null` via `systemctl mask`.
   - Solution: Unmask the unit: `sudo systemctl unmask api.service`.
2. **`ExecStart: 203/EXEC` on shell script**:
   - Cause: Script was edited on Windows and contains CRLF (`\r\n`) line endings.
   - Solution: Run `dos2unix /path/to/script.sh` and ensure `chmod +x` is set.
3. **Changes to `.service` file do not take effect**:
   - Cause: Systemd caches unit configurations in memory.
   - Solution: Always run `sudo systemctl daemon-reload` before restarting.

---

## 7. Key Takeaways
- Decipher exit codes (`1/FAILURE`, `137/OOM`, `203/EXEC`) to immediately narrow the failure scope.
- Systemd runs in a minimal, sterile environment; explicitly define `WorkingDirectory` and `EnvironmentFile`.
- Use `journalctl -xeu <unit>` for comprehensive error traces.
- Always use drop-in overrides (`systemctl edit <unit>`) rather than modifying distribution defaults directly.
