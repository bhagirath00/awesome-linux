# Chapter 34: Advanced Shell Scripting - Parsing CLI Flags with getopts

## 1. Introduction: Professional CLI Argument Parsing

When writing automation scripts for teams, hardcoding positional parameters (`$1`, `$2`, `$3`) is brittle and error-prone:
- Users must memorize the exact order of arguments.
- Optional flags (like `-v` for verbose or `-p 8080` for port) cannot be easily toggled.

The **`getopts`** shell built-in provides a standardized, robust parser for single-character command-line options and flags following POSIX utility guidelines (e.g., `./deploy.sh -e staging -p 8080 -v`).

---

## 2. Anatomy of the getopts Command

**Syntax:**
```bash
while getopts "<optstring>" <variable>; do
    case "$<variable>" in
        # handle options...
    esac
done
```

### The `optstring` Specification:
The `optstring` defines which flag letters are recognized:
- A letter by itself (e.g., `v`) indicates a **boolean flag** (takes no argument).
- A letter followed by a colon (e.g., `e:`) indicates that this option **requires an argument**.
- A leading colon (e.g., `:e:p:v`) turns on **silent error reporting**, allowing your script to handle missing arguments gracefully instead of printing built-in shell errors.

### The Two Automatic getopts Variables:
1. **`$OPTARG`**: Holds the argument value passed to the option (e.g., if `-e staging`, `$OPTARG` is `"staging"`).
2. **`$OPTIND`**: Holds the index of the next argument to be processed.

---

## 3. Production getopts Script Template

```bash
#!/usr/bin/env bash
set -euo pipefail

# Default configuration values
ENVIRONMENT="staging"
PORT="8080"
VERBOSE=false

usage() {
    cat << EOF
Usage: $(basename "$0") [OPTIONS] [TARGET_HOST]

Options:
  -e <env>     Deployment environment (default: staging)
  -p <port>    Target application port (default: 8080)
  -v           Enable verbose debug logging
  -h           Show this help message and exit

Example:
  $(basename "$0") -e production -p 443 -v web01.company.internal
EOF
    exit 1
}

# Parse options using getopts
while getopts ":e:p:vh" opt; do
    case "${opt}" in
        e)
            ENVIRONMENT="${OPTARG}"
            ;;
        p)
            PORT="${OPTARG}"
            if [[ ! "${PORT}" =~ ^[0-9]+$ ]]; then
                echo "[ERROR] Port must be a valid integer!" >&2
                exit 1
            fi
            ;;
        v)
            VERBOSE=true
            ;;
        h)
            usage
            ;;
        :)
            echo "[ERROR] Option -${OPTARG} requires an argument!" >&2
            usage
            ;;
        \?)
            echo "[ERROR] Invalid option: -${OPTARG}" >&2
            usage
            ;;
    esac
done

# Shift processed flags out of the argument list:
shift $(( OPTIND - 1 ))

# Remaining positional arguments (like TARGET_HOST) are now in $1, $2, etc.:
TARGET_HOST="${1:-localhost}"

echo "=========================================="
echo "Deployment Plan:"
echo "  Target Host : ${TARGET_HOST}"
echo "  Environment : ${ENVIRONMENT}"
echo "  Port        : ${PORT}"
echo "  Verbose     : ${VERBOSE}"
echo "=========================================="
```

---

## 4. Execution Examples

### 1. Running with Custom Flags:
```bash
./deploy.sh -e production -p 443 -v node-cluster-01
```
```
==========================================
Deployment Plan:
  Target Host : node-cluster-01
  Environment : production
  Port        : 443
  Verbose     : true
==========================================
```

### 2. Missing Required Option Argument:
```bash
./deploy.sh -e
```
```
[ERROR] Option -e requires an argument!
Usage: deploy.sh [OPTIONS] [TARGET_HOST]
...
```

---

## 5. Summary Takeaways

- `getopts` is the POSIX standard built-in parser for command-line flags.
- Use colons after letters (`e:`) to designate options that require arguments.
- Access the argument value via `$OPTARG`.
- Always call `shift $(( OPTIND - 1 ))` after the `while` loop to access remaining trailing positional parameters.
- Provide a clear, actionable `usage()` help menu triggered by `-h`.
