# Chapter 30: Functions - Defining and Calling Functions in Bash

## 1. Introduction: Modular Shell Scripting

As shell scripts grow to automate complex infrastructure deployments, duplicating code blocks across a script creates maintenance liabilities. 

A **function** is a reusable block of code assigned a descriptive name. When invoked, the function executes its commands and returns control to the point of invocation.

In DevOps, functions enable:
- Code reusability (DRY: Don't Repeat Yourself).
- Modular architecture (separating logging, validation, deployment, and cleanup).
- Cleaner error handling and readable orchestration flows.

---

## 2. Syntax Conventions for Defining Functions

Bash supports two syntactical conventions for defining functions:

### Convention 1: POSIX Standard (Recommended)
```bash
my_function() {
    # commands...
}
```

### Convention 2: Bash `function` Keyword
```bash
function my_function {
    # commands...
}
# OR:
function my_function() {
    # commands...
}
```

> [!TIP]
> **Production Best Practice**: Use the POSIX format **`my_function() { ... }`** without the `function` keyword. It is shorter, cleaner, and 100% portable across all Unix shells (Bash, Dash, Zsh, BusyBox).

---

## 3. Order of Declaration: Define Before Calling

Unlike compiled languages with forward-declarations or languages with hoisting (like JavaScript), **Bash interprets scripts top-to-bottom line by line**.

A function **MUST be defined earlier in the script before it is called**. Invoking a function before its declaration results in a runtime error:
```
line 5: log_message: command not found
```

```bash
#!/usr/bin/env bash
set -euo pipefail

# 1. Define functions first at the top of the file
log_info() {
    echo "[INFO]  $(date '+%Y-%m-%d %H:%M:%S') - $1"
}

log_error() {
    echo "[ERROR] $(date '+%Y-%m-%d %H:%M:%S') - $1" >&2
}

# 2. Invoke functions later in the main execution logic
log_info "Initializing production deploy..."
log_error "Failed to connect to database replica!"
```

---

## 4. Invoking Functions (No Parentheses!)

In many programming languages (Python, Java, JavaScript), functions are invoked with parentheses: `my_function()`.

In Bash, **you NEVER use parentheses when calling a function!** In Bash, functions are invoked **exactly like external Linux commands**, with space-separated arguments:

```bash
# CORRECT:
log_info "Deployment succeeded"

# WRONG (Syntax Error / Subshell invocation):
log_info("Deployment succeeded")
```

---

## 5. The Enterprise `main()` Pattern

In professional DevOps automation, scripts should be structured with a clean entrypoint function (traditionally named `main`), passing script CLI arguments down into it:

```bash
#!/usr/bin/env bash
set -euo pipefail

# ==============================================================================
# Helper Functions
# ==============================================================================
setup_environment() {
    echo "Setting up temporary scratch directories..."
}

deploy_application() {
    local target="$1"
    echo "Deploying release to '${target}'..."
}

cleanup() {
    echo "Cleaning up build artifacts..."
}

# ==============================================================================
# Main Orchestration Entrypoint
# ==============================================================================
main() {
    local target_env="${1:-staging}"

    setup_environment
    deploy_application "${target_env}"
    cleanup

    echo "Pipeline execution finished successfully."
}

# Invoke main, forwarding all script arguments:
main "$@"
```

---

## 6. Summary Takeaways

- Use the clean POSIX syntax: `function_name() { ... }`.
- Functions must be declared above the line where they are invoked.
- Call functions like standard commands without parentheses: `my_func arg1 arg2`.
- Organize complex scripts using the `main "$@"` entrypoint design pattern.
