# Chapter 30: Functions - Creating Reusable Function Libraries

## 1. Introduction: Shared Code Architecture

In enterprise DevOps engineering, common operations—such as formatting timestamps, sending Slack/Teams notifications, validating IPv4 addresses, checking AWS CLI credentials, and parsing YAML files—should not be copy-pasted across dozens of deployment scripts.

A **function library** is a shell script containing reusable function definitions that is imported (**sourced**) into other scripts using the **`source`** command (or POSIX **`.`**).

```
+--------------------------------------------------------------------------+
|                    Function Library Architecture                         |
+--------------------------------------------------------------------------+
|  Shared Library: /usr/local/lib/devops-utils.sh                          |
|  ├── log_info()                                                          |
|  ├── log_error()                                                         |
|  ├── send_slack_alert()                                                  |
|  └── require_root()                                                      |
|                                |                                         |
|                 source /usr/local/lib/devops-utils.sh                    |
|                                |                                         |
|         +----------------------+----------------------+                  |
|         |                                             |                  |
|         v                                             v                  |
|  [ deploy-web.sh ]                          [ database-backup.sh ]       |
|  log_info "Deploying v2.1"                  require_root                 |
|  send_slack_alert "#releases"               log_info "Starting pg_dump"  |
+--------------------------------------------------------------------------+
```

---

## 2. Writing a Reusable Library: `lib/utils.sh`

A well-architected library file should:
1. Define pure functions and constants.
2. Prevent being executed directly as a standalone script.
3. Prevent duplicate loading (include guards).

```bash
# /usr/local/lib/devops-utils.sh
# Check if already sourced (Include Guard)
[[ -n "${_DEVOPS_UTILS_LOADED:-}" ]] && return 0
readonly _DEVOPS_UTILS_LOADED=1

# ANSI Color Codes
readonly COLOR_RESET="\033[0m"
readonly COLOR_GREEN="\033[0;32m"
readonly COLOR_RED="\033[0;31m"
readonly COLOR_YELLOW="\033[0;33m"

# Structured Logging Functions
log_info() {
    printf "${COLOR_GREEN}[INFO]${COLOR_RESET}  %s - %s\n" "$(date '+%Y-%m-%d %H:%M:%S')" "$*"
}

log_warn() {
    printf "${COLOR_YELLOW}[WARN]${COLOR_RESET}  %s - %s\n" "$(date '+%Y-%m-%d %H:%M:%S')" "$*" >&2
}

log_error() {
    printf "${COLOR_RED}[ERROR]${COLOR_RESET} %s - %s\n" "$(date '+%Y-%m-%d %H:%M:%S')" "$*" >&2
}

# Privilege Enforcer
require_root() {
    if [[ "${EUID}" -ne 0 ]]; then
        log_error "This operation requires root superuser privileges! Please run with sudo."
        exit 1
    fi
}

# Binary Dependency Checker
require_binary() {
    local bin="$1"
    if ! command -v "${bin}" &>/dev/null; then
        log_error "Mandatory binary '${bin}' is missing from system PATH!"
        exit 1
    fi
}
```

---

## 3. Sourcing the Library in Consumer Scripts

To import the library, use `source` (or `.`):

### Resolving Library Path Relative to the Script (CRITICAL PATTERN)
Never rely on hardcoded relative paths like `source ./lib/utils.sh` because if the user runs the script from a different directory (e.g., `cd /tmp && /opt/app/deploy.sh`), the relative path will break!

Always resolve paths **relative to the directory where the script itself lives**:

```bash
#!/usr/bin/env bash
set -euo pipefail

# 1. Dynamically locate the script's root directory:
SCRIPT_DIR="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" &>/dev/null && pwd)"

# 2. Source the shared library reliably:
# shellcheck source=lib/utils.sh
source "${SCRIPT_DIR}/lib/utils.sh"

# 3. Use library functions immediately:
require_binary "docker"
require_binary "curl"

log_info "All prerequisite binaries are verified."
log_warn "Disk space utilization is approaching 80%."
log_info "Deployment pipeline ready."
```

---

## 4. Summary Takeaways

- Consolidate common utility functions into shared library scripts.
- Use include guards (`[[ -n "${_LIB_LOADED}" ]] && return 0`) to prevent duplicate inclusion.
- Always resolve library paths using `$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)`.
- Reusable libraries ensure consistent logging formats and error handlers across your entire infrastructure codebase.
