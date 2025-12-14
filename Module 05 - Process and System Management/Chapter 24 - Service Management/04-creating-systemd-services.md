# Chapter 24: Service Management - Creating Custom systemd Services

## 1. Introduction: Writing Production Unit Files

In modern DevOps environments, custom applications (Python FastAPI apps, Node.js APIs, Go binaries, Java JARs) must be managed as robust background services that:
- Automatically start when the server boots.
- Automatically restart if the process crashes or encounters an unhandled exception.
- Run under an unprivileged system user (never `root`).
- Enforce memory and CPU utilization limits.
- Stream standard output and standard error directly into the centralized system journal.

All custom system services must be saved in **`/etc/systemd/system/<service-name>.service`**.

---

## 2. Anatomy of a .service Unit File

A systemd service unit file is divided into three primary sections: `[Unit]`, `[Service]`, and `[Install]`.

```ini
[Unit]
Description=Enterprise Payment Microservice API
Documentation=https://internal.wiki.company.com/payment-service
After=network-online.target postgresql.service
Wants=network-online.target

[Service]
Type=simple
User=appuser
Group=appgroup
WorkingDirectory=/opt/payment-service

ExecStart=/usr/bin/node /opt/payment-service/dist/server.js
ExecReload=/bin/kill -s HUP $MAINPID

Restart=always
RestartSec=5s
StartLimitIntervalSec=60s
StartLimitBurst=5

Environment=NODE_ENV=production PORT=8080
EnvironmentFile=/etc/payment-service/env.conf

# Security Hardening Sandbox Directives
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/opt/payment-service/logs

# Resource Constraints (Cgroups)
MemoryMax=1G
CPUQuota=150%

[Install]
WantedBy=multi-user.target
```

---

## 3. Section-by-Section Directive Breakdown

### 1. The `[Unit]` Section (Metadata & Ordering)
- **`Description`**: Human-readable name displayed in `systemctl status` and journal logs.
- **`After`**: Ordering dependency. Guarantees this service will only start **after** `network-online.target` and `postgresql.service` have finished initializing.
- **`Requires`**: Strict dependency. If the required unit fails or is stopped, this service will immediately stop as well.
- **`Wants`**: Weak dependency. Requests the dependent unit to start, but does not stop this service if the other unit fails.

### 2. The `[Service]` Section (Execution Logic)
| Directive | Description |
| :--- | :--- |
| **`Type`** | Service process model: `simple` (default; process executes directly in foreground), `forking` (legacy daemons that daemonize and fork), `oneshot` (runs once to completion and exits), `notify` (daemon signals systemd via sd_notify when ready). |
| **`User` / `Group`** | The unprivileged operating system account under which the process executes. |
| **`WorkingDirectory`** | The directory changed to before executing `ExecStart`. |
| **`ExecStart`** | **Must be an absolute path** to the executable binary and its arguments. |
| **`Restart`** | Restart policy: `no`, `on-failure` (exits with non-zero code or killed by signal), or `always`. |
| **`RestartSec`** | Delay in seconds between restart attempts (prevents CPU-burning crash loops). |
| **`StartLimitBurst`** | Max number of restart attempts within `StartLimitIntervalSec` before marking the unit as failed. |
| **`Environment`** | Hardcoded key-value environment variables. |
| **`EnvironmentFile`** | Path to an external file containing environment secrets (`KEY=VALUE`). |

### 3. The `[Install]` Section (Boot Links)
- **`WantedBy=multi-user.target`**: Instructs systemd to create a symbolic link inside `/etc/systemd/system/multi-user.target.wants/` when you execute `systemctl enable`. This ensures the service launches during standard multi-user system boot.

---

## 4. Step-by-Step Deployment Walkthrough

Let's deploy a production Python application step-by-step:

### Step 1: Create Dedicated Service User
```bash
sudo useradd -r -s /bin/false -d /opt/webapp appuser
```

### Step 2: Create the Service Unit File
```bash
sudo tee /etc/systemd/system/webapp.service << 'EOF'
[Unit]
Description=Python FastAPI Production Webapp
After=network.target

[Service]
Type=simple
User=appuser
Group=appuser
WorkingDirectory=/opt/webapp
ExecStart=/opt/webapp/venv/bin/uvicorn main:app --host 127.0.0.1 --port 8000
Restart=on-failure
RestartSec=3s
Environment="PYTHONUNBUFFERED=1"

[Install]
WantedBy=multi-user.target
EOF
```

### Step 3: Reload systemd and Start Service
```bash
# Inform systemd about the new unit file
sudo systemctl daemon-reload

# Enable to boot on startup and start immediately
sudo systemctl enable --now webapp.service

# Verify operational status
sudo systemctl status webapp.service
```

---

## 5. Overriding Existing Units Cleanly with `systemctl edit`

Never modify upstream unit files in `/lib/systemd/system/`. When you need to adjust an existing package (e.g., adding an environment variable to `docker.service`):

```bash
sudo systemctl edit docker.service
```

This opens a text editor and automatically creates a drop-in override file in `/etc/systemd/system/docker.service.d/override.conf`:

```ini
[Service]
# Clear existing ExecStart and define new parameters
ExecStart=
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --max-concurrent-downloads 10
```

---

## 6. Summary Takeaways

- Place custom service unit files in `/etc/systemd/system/`.
- Always specify absolute paths for executables in `ExecStart`.
- Always run microservices under dedicated non-root users (`User=appuser`).
- Use `Restart=on-failure` and `RestartSec=5s` to protect against unexpected crashes.
- Always execute `sudo systemctl daemon-reload` after editing unit files.
