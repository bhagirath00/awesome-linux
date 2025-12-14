# Chapter 24: Service Management - Service Best Practices and Troubleshooting

## 1. Introduction: Production-Grade Service Design

Designing and running services under `systemd` requires adhering to established operational patterns to prevent security vulnerabilities, runaway resource consumption, cascading boot stalls, and silent service failures.

```
+--------------------------------------------------------------------------+
|                     Production Service Architecture                      |
+--------------------------------------------------------------------------+
|  Security Sandboxing      Resource Limits (Cgroups)   Supervision Hooks  |
|  - User=non-root          - MemoryMax=1.5G            - Restart=on-fail  |
|  - ProtectSystem=strict   - CPUQuota=200%             - RestartSec=5s    |
|  - ProtectHome=true       - TasksMax=512              - WatchdogSec=30s  |
|  - NoNewPrivileges=true                               - OnFailure=alert  |
+--------------------------------------------------------------------------+
```

---

## 2. Security Sandboxing Directives

`systemd` provides powerful kernel namespace and filesystem sandboxing directives built directly into the unit configuration. By enabling these flags, you can severely confine a compromised service without configuring complex SELinux or AppArmor profiles:

```ini
[Service]
# Never run application code as root
User=nodeapp
Group=nodeapp

# Prevent child processes from acquiring new privileges via SUID/SGID
NoNewPrivileges=true

# Mount /usr, /boot, and /etc as read-only for this service
ProtectSystem=strict

# Make /home, /root, and /run/user inaccessible to the service
ProtectHome=true

# Provide an isolated /tmp and /var/tmp namespace unique to this service
PrivateTmp=true

# Restrict network socket families (only allow IPv4 and IPv6)
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX

# Explicitly whitelist the ONLY directories the service is permitted to write to
ReadWritePaths=/opt/nodeapp/data /var/log/nodeapp
```

---

## 3. Resource Management via Control Groups (cgroups)

Prevent a memory leak or runaway infinite loop in an application from bringing down the entire host server:

```ini
[Service]
# Hard memory ceiling (Kernel OOM killer terminates service if exceeded)
MemoryMax=2G

# Soft memory throttle limit
MemoryHigh=1.8G

# Throttle CPU to at most 2 cores (200% = 2 dedicated cores)
CPUQuota=200%

# Prevent fork bombs: maximum number of processes/threads this service can spawn
TasksMax=250
```

---

## 4. Crash Loops and Restart Limiting

If an application encounters a fatal configuration error (e.g., database credentials wrong), restarting instantly in an infinite loop burns 100% CPU and floods disk logs. 

Always implement **Restart Rate Limiting**:

```ini
[Unit]
Description=Backend API Service
# Stop attempting restarts if service fails 5 times within a 60-second window:
StartLimitIntervalSec=60s
StartLimitBurst=5

[Service]
Type=simple
ExecStart=/usr/local/bin/api-server
Restart=on-failure
RestartSec=5s
```

---

## 5. Automated Failure Alerting (`OnFailure=`)

When a critical production service fails, you can instruct `systemd` to automatically trigger an alerting service unit (which sends a Slack notification, PagerDuty alert, or webhook):

### Step 1: Add `OnFailure` to Primary Service
```ini
# /etc/systemd/system/payment-api.service
[Unit]
Description=Payment Gateway API
OnFailure=notify-slack@%n.service

[Service]
ExecStart=/opt/bin/payment-api
Restart=on-failure
```

### Step 2: Create Reusable Notification Template Unit
```ini
# /etc/systemd/system/notify-slack@.service
[Unit]
Description=Send Slack Alert on Failure of %i

[Service]
Type=oneshot
ExecStart=/usr/local/bin/send-slack-alert.sh "CRITICAL: Service %i has failed on $(hostname)!"
```

---

## 6. Systematic Troubleshooting Flowchart

```
                 Service Fails to Start / Crashes
                               |
                               v
                     systemctl status <unit>
                               |
            +------------------+------------------+
            |                                     |
    Syntax Error / Load Fail?              Active (Failed)?
            |                                     |
            v                                     v
  sudo systemd-analyze verify <unit>       journalctl -u <unit> -e -n 50
  sudo systemctl daemon-reload                    |
                                           Look for:
                                           - Missing config/env vars
                                           - Permission Denied (DAC/SELinux)
                                           - Port conflict (bind: address in use)
                                           - Killed by OOM (exit code 137)
```

---

## 7. Summary Takeaways

- Apply `systemd` security sandboxing (`ProtectSystem=strict`, `ProtectHome=true`, `PrivateTmp=true`).
- Always impose cgroup memory and CPU limits (`MemoryMax`, `CPUQuota`, `TasksMax`).
- Prevent infinite crash loops using `StartLimitIntervalSec` and `StartLimitBurst`.
- Automate failure notifications using `OnFailure=notify-slack@%n.service`.
- Verify unit file syntax prior to deployment using `systemd-analyze verify`.
