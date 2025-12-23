# Chapter 34: Advanced Shell Scripting - Production Script Best Practices and Design Patterns

## 1. Introduction: The Production-Ready Checklist

Writing a shell script that works once on a local development laptop is easy. Writing a shell script that executes reliably across thousands of heterogeneous servers, resilient to network drops, concurrent executions, and permission issues requires disciplined adherence to architectural design patterns.

```
+--------------------------------------------------------------------------+
|                  Enterprise Shell Scripting Architecture                 |
+--------------------------------------------------------------------------+
|  [1] Shebang & Safety    ===> #!/usr/bin/env bash | set -euo pipefail    |
|  [2] Dynamic Paths       ===> SCRIPT_DIR="$(cd "$(dirname ...)" && pwd)" |
|  [3] Concurrency Control ===> flock lockfile to prevent overlapping runs |
|  [4] Resource Cleanup    ===> trap cleanup EXIT INT TERM                 |
|  [5] Variable Scoping    ===> local vars in functions, quote everything  |
|  [6] Pre-Flight Asserts  ===> command -v check for required binaries     |
|  [7] Static Verification ===> shellcheck & bash -n in CI/CD pipeline     |
+--------------------------------------------------------------------------+
```

---

## 2. The Complete Production Script Template

Below is the definitive reference template incorporating all enterprise design patterns covered in Module 06:

```bash
#!/usr/bin/env bash
# ==============================================================================
# Script Name : deploy-service.sh
# Description : Production release deployment orchestrator with health checks.
# Author      : DevOps Platform Team <devops@company.internal>
# Version     : 2.1.0
# License     : MIT
# ==============================================================================

# Enable strict error handling:
# -e: Exit on command failure
# -u: Exit on uninitialized variables
# -o pipefail: Pipeline fails if any command fails
set -euo pipefail

# Trace execution if DEBUG environment variable is set (e.g., DEBUG=1 ./script.sh)
[[ "${DEBUG:-0}" == "1" ]] && set -x

# Resolve script root directory portably:
readonly SCRIPT_DIR="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" &>/dev/null && pwd)"
readonly SCRIPT_NAME="$(basename "$0")"

# Configuration Defaults
ENVIRONMENT="staging"
PORT=8080
LOCK_FILE="/var/run/${SCRIPT_NAME}.lock"
SCRATCH_DIR=""

# ANSI Colors
readonly COLOR_GREEN='\033[0;32m'
readonly COLOR_YELLOW='\033[0;33m'
readonly COLOR_RED='\033[0;31m'
readonly COLOR_RESET='\033[0m'

# ==============================================================================
# Logging Helpers
# ==============================================================================
log_info()  { printf "${COLOR_GREEN}[INFO]${COLOR_RESET}  %s - %s\n" "$(date '+%Y-%m-%d %H:%M:%S')" "$*"; }
log_warn()  { printf "${COLOR_YELLOW}[WARN]${COLOR_RESET}  %s - %s\n" "$(date '+%Y-%m-%d %H:%M:%S')" "$*" >&2; }
log_error() { printf "${COLOR_RED}[ERROR]${COLOR_RESET} %s - %s\n" "$(date '+%Y-%m-%d %H:%M:%S')" "$*" >&2; }

# ==============================================================================
# Cleanup and Exit Handling
# ==============================================================================
cleanup() {
    local exit_code="$?"
    log_info "Initiating resource cleanup..."

    # Remove temporary scratch directory
    if [[ -n "${SCRATCH_DIR}" && -d "${SCRATCH_DIR}" ]]; then
        rm -rf "${SCRATCH_DIR}"
        log_info "Removed scratch workspace: ${SCRATCH_DIR}"
    fi

    # Release file lock
    flock -u 200 2>/dev/null || true
    rm -f "${LOCK_FILE}" 2>/dev/null || true

    if [[ "${exit_code}" -eq 0 ]]; then
        log_info "Script finished successfully."
    else
        log_error "Script terminated with error (Exit code: ${exit_code})!"
    fi

    exit "${exit_code}"
}

# Register signal trap
trap cleanup EXIT INT TERM

# ==============================================================================
# Pre-Flight Verifications
# ==============================================================================
check_dependencies() {
    local deps=("curl" "tar" "systemctl")
    for dep in "${deps[@]}"; do
        if ! command -v "${dep}" &>/dev/null; then
            log_error "Missing required system binary: '${dep}'"
            exit 127
        fi
    done
}

# ==============================================================================
# Usage and CLI Parsing
# ==============================================================================
usage() {
    cat << EOF
Usage: ${SCRIPT_NAME} [OPTIONS] <TARGET_HOST>

Options:
  -e <env>     Target environment [dev|staging|production] (Default: staging)
  -p <port>    Application port (Default: 8080)
  -h           Show this help message and exit
EOF
    exit 1
}

# ==============================================================================
# Main Orchestration Logic
# ==============================================================================
main() {
    # 1. Parse command-line flags
    while getopts ":e:p:h" opt; do
        case "${opt}" in
            e) ENVIRONMENT="${OPTARG}" ;;
            p) PORT="${OPTARG}" ;;
            h) usage ;;
            \?|:) usage ;;
        esac
    done
    shift $(( OPTIND - 1 ))

    local target_host="${1:-}"
    if [[ -z "${target_host}" ]]; then
        log_error "Target host argument is required!"
        usage
    fi

    # 2. Acquire concurrency lock
    exec 200>"${LOCK_FILE}"
    flock -n 200 || { log_error "Another instance of ${SCRIPT_NAME} is active!"; exit 1; }

    # 3. Check dependencies
    check_dependencies

    # 4. Create secure scratch workspace
    SCRATCH_DIR=$(mktemp -d "/tmp/${SCRIPT_NAME}.XXXXXX")
    log_info "Scratch workspace initialized at: ${SCRATCH_DIR}"

    # 5. Core execution steps
    log_info "Deploying to host '${target_host}' in environment '${ENVIRONMENT}' on port ${PORT}..."
    sleep 1

    log_info "Deployment tasks complete."
}

# Entrypoint invocation:
main "$@"
```

---

## 3. Summary Best Practices Checklist

| Best Practice Rule | Implementation Requirement |
| :--- | :--- |
| **Strict Error Mode** | Always set `set -euo pipefail` on line 2. |
| **Double Quote Everything** | Always quote variables: `"${VAR}"` (prevents word splitting). |
| **Path Independence** | Resolve directory using `$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)`. |
| **Signal Trapping** | Register `trap cleanup EXIT INT TERM` for guaranteed resource teardown. |
| **File Locking** | Use `flock` to guarantee idempotent, non-overlapping runs. |
| **Linting** | Enforce `shellcheck` pass in CI/CD before any script merges into main branch. |
