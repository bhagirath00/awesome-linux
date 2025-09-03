# Chapter 24: Service Management - Managing Services and Units

## 1. Introduction: Querying and Listing systemd Units

In a production Linux environment hosting dozens of background daemons, microservices, and database engines, administrators must be able to inventory running services, locate broken dependencies, and inspect failed units.

`systemctl` provides comprehensive listing and filtering commands to inspect the state of all units managed by PID 1.

---

## 2. Listing Units: `list-units` vs. `list-unit-files`

There is an important distinction between active loaded units in memory and unit files on disk:

### 1. Listing Currently Loaded Units (`list-units`)
Shows units that `systemd` currently has loaded in memory or active in runtime:
```bash
# List all active units across all types
systemctl list-units

# Filter strictly for services that are currently running
systemctl list-units --type=service --state=running
```

```
UNIT                        LOAD   ACTIVE SUB     DESCRIPTION
containerd.service          loaded active running containerd container runtime
cron.service                loaded active running Regular background program processing daemon
dbus.service                loaded active running D-Bus System Message Bus
docker.service              loaded active running Docker Application Container Engine
nginx.service               loaded active running A high performance web server
sshd.service                loaded active running OpenBSD Secure Shell server
systemd-journald.service    loaded active running Journal Service
```

### 2. Listing All Unit Files on Disk (`list-unit-files`)
Shows every unit configuration file installed on disk and whether it is configured to boot on startup:
```bash
systemctl list-unit-files --type=service
```

```
UNIT FILE                              STATE           VENDOR PRESET
nginx.service                          enabled         enabled
apache2.service                        disabled        enabled
redis-server.service                   enabled         enabled
ufw.service                            enabled         enabled
whoopsie.service                       masked          enabled
```

---

## 3. Investigating and Resetting Failed Services

When an application crashes, fails its health check, or errors during boot, `systemd` flags the unit with an **`active (failed)`** state.

### 1. Listing All Failed Units
```bash
systemctl --failed
```

```
  UNIT                  LOAD   ACTIVE SUB    DESCRIPTION
● postgresql.service    loaded failed failed PostgreSQL RDBMS Server

LOAD   = Reflects whether the unit definition was properly loaded.
ACTIVE = The high-level unit activation state.
SUB    = The low-level unit activation sub-state.

1 loaded units listed.
```

### 2. Inspecting the Cause of Failure
```bash
# View recent log entries for the failed unit
journalctl -u postgresql.service -e --no-pager
```

### 3. Clearing Failed State (`reset-failed`)
After fixing the underlying issue (e.g., repairing corrupted file permissions, expanding disk space):
```bash
# Restart the service
sudo systemctl restart postgresql.service

# If the service remains in failed state, reset the error counter:
sudo systemctl reset-failed postgresql.service
```

---

## 4. Visualizing Dependencies with `list-dependencies`

Services rarely run in isolation; a web application service may depend on a local database service, which in turn depends on the network stack and mounted filesystem.

To view the hierarchical dependency tree of a unit:

```bash
systemctl list-dependencies nginx.service
```

```
nginx.service
● ├─system.slice
● ├─network.target
● └─sysinit.target
●   ├─dev-hugepages.mount
●   ├─dev-mqueue.mount
●   ├─kmod-static-nodes.service
●   ├─systemd-journald.service
●   ├─systemd-modules-load.service
●   ├─systemd-sysctl.service
●   ├─systemd-udevd.service
●   └─ufw.service
```

To see reverse dependencies (which services depend on this service):
```bash
systemctl list-dependencies --reverse nginx.service
```

---

## 5. Summary Takeaways

- Use `systemctl list-units --type=service --state=running` to see currently active daemons.
- Use `systemctl list-unit-files` to audit boot persistence across the entire system.
- Always check `systemctl --failed` during post-boot health verifications.
- Clear error states using `systemctl reset-failed <unit>`.
- Map service prerequisite hierarchies with `systemctl list-dependencies`.
