# Chapter 33: Error Handling in Scripts - Structured Logging and Syslog Integration

## 1. Introduction: Production Logging Principles

In automated DevOps and SRE workflows, shell scripts execute unattended inside CI/CD runners, Kubernetes init-containers, and cron jobs. 

Unstructured, raw `echo` statements scattered throughout a script make post-incident triage painful. Production-grade scripts adhere to three logging principles:
1. **Timestamped and Levelled**: Every log line includes an ISO-8601 timestamp and severity (`INFO`, `WARN`, `ERROR`, `DEBUG`).
2. **Standard Error Routing**: Normal messages go to `stdout` (FD 1); warnings and fatal errors go to `stderr` (FD 2).
3. **Dual-Destination**: Logs print to terminal/CI consoles while simultaneously appending to persistent disk files or system log daemons (`syslog` / `journald`).

---

## 2. A Production Structured Logger Framework

Here is a robust, modular logging framework ready for enterprise deployment:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Configuration
LOG_FILE="/var/log/app-deployment.log"
LOG_LEVEL="${LOG_LEVEL:-INFO}" # Default to INFO; can override via LOG_LEVEL=DEBUG

# ANSI Colors
readonly C_RESET='\033[0m'
readonly C_RED='\033[0;31m'
readonly C_GREEN='\033[0;32m'
readonly C_YELLOW='\033[0;33m'
readonly C_BLUE='\033[0;34m'

# Internal core log writer
_log() {
    local level="$1"
    local color="$2"
    local stream="$3"
    shift 3
    local message="$*"
    local timestamp
    timestamp=$(date '+%Y-%m-%d %H:%M:%S')

    # Format for terminal (with color)
    local console_entry="${color}[${level}]${C_RESET} ${timestamp} - ${message}"

    # Format for disk file (clean plaintext without ANSI escape codes)
    local file_entry="[${level}] ${timestamp} - ${message}"

    # Output to appropriate file descriptor (stdout or stderr)
    if [[ "${stream}" == "stderr" ]]; then
        echo -e "${console_entry}" >&2
    else
        echo -e "${console_entry}"
    fi

    # Append to log file if defined and writable
    if [[ -n "${LOG_FILE:-}" ]] && [[ -w "$(dirname "${LOG_FILE}")" ]]; then
        echo "${file_entry}" >> "${LOG_FILE}"
    fi
}

# Public Logging Interfaces
log_debug() {
    [[ "${LOG_LEVEL}" == "DEBUG" ]] && _log "DEBUG" "${C_BLUE}" "stdout" "$@"
}

log_info() {
    _log "INFO" "${C_GREEN}" "stdout" "$@"
}

log_warn() {
    _log "WARN" "${C_YELLOW}" "stderr" "$@"
}

log_error() {
    _log "ERROR" "${C_RED}" "stderr" "$@"
}

log_fatal() {
    _log "FATAL" "${C_RED}" "stderr" "$@"
    exit 1
}
```

### Usage in Application Logic:
```bash
log_info "Initializing production migration..."
log_debug "Connecting to database socket at /var/run/postgresql/.s.PGSQL.5432"
log_warn "Cache hit ratio is below 70%"
# log_fatal "Cannot bind to port 80! Exiting."
```

---

## 3. Integrating with the Linux System Logger: `logger`

Instead of managing local file rotations, scripts can stream logs directly into the Linux kernel logging daemon (`systemd-journald` / `rsyslog`) using the **`logger`** utility:

```bash
# Log a message with custom tag and priority
logger -t "deploy-script" -p user.info "Starting Kubernetes pod rollout"
logger -t "deploy-script" -p user.err "Rollout failed: ContainerCrashLoopBackOff"
```

### Inspecting Script Logs via journalctl:
```bash
journalctl -t deploy-script -e
```
```
Oct 14 06:40:12 web-prod-01 deploy-script[1420]: Starting Kubernetes pod rollout
Oct 14 06:40:15 web-prod-01 deploy-script[1420]: Rollout failed: ContainerCrashLoopBackOff
```

---

## 4. Summary Takeaways

- Standardize logging formats with timestamps, severity levels, and context.
- Route errors and warnings to `stderr` (`>&2`) to allow clean stdout piping.
- Strip ANSI color escape codes when appending to disk log files.
- Use the `logger` CLI utility to integrate script logs natively into `journald` and `syslog`.
