# Chapter 33: Error Handling in Scripts - Debugging and Tracing Shell Scripts

## 1. Introduction: Debugging Shell Scripts

Unlike high-level languages with integrated graphical debuggers, stepping through shell scripts relies on built-in shell diagnostic tracing flags, syntax verifiers, and static analysis linters.

Debugging a faulty shell script typically follows a three-stage methodology:
1. **Static Syntax Checking**: Validating syntax without running any commands (`bash -n`).
2. **Automated Static Analysis**: Scanning for non-portable syntax and logic bugs (`shellcheck`).
3. **Execution Tracing**: Printing every evaluated command and expanded variable in real time (`bash -x`).

---

## 2. Syntax Validation: `bash -n`

Before running a script on a production server, verify that there are no unmatched quotes, missing `fi`/`done`/`esac` terminators, or broken brackets:

```bash
bash -n deploy.sh
```

- If `bash -n` produces **no output**, syntax is valid.
- If there is a syntax error, it displays the offending line immediately **without executing any dangerous commands**!

```
deploy.sh: line 42: syntax error: unexpected end of file (expected "fi")
```

---

## 3. Real-Time Execution Tracing: `bash -x` (xtrace)

The **`xtrace`** option (`-x`) is the most powerful debugging tool in Bash. It causes the shell to print each command to `stderr` **after performing variable expansion and command substitution**, but **before executing it**.

Every traced line is prefixed by the **`PS4`** prompt variable (default is `+`):

### Method 1: Running the Entire Script with Tracing
```bash
bash -x deploy.sh
```

### Method 2: Enabling Tracing Inside the Script File
```bash
#!/usr/bin/env bash
set -x # Enable tracing for the whole script
```

### Method 3: Targeted / Selective Tracing (Best Practice)
Tracing an entire 500-line script generates overwhelming log noise. Enable tracing **only around the problematic code block**:

```bash
echo "Initializing..."

# --- BEGIN DEBUG TRACE ---
set -x
TARGET_ENV="production"
DEPLOY_CMD="kubectl rollout status deployment/web -n ${TARGET_ENV}"
${DEPLOY_CMD}
set +x
# --- END DEBUG TRACE ---

echo "Continuing normal execution..."
```

---

## 4. Customizing the Debug Prompt (`PS4`)

By customizing the `PS4` shell prompt, you can configure Bash to print the **source filename**, **line number**, and **function name** on every traced line:

```bash
export PS4='+ [${BASH_SOURCE}:${LINENO} in ${FUNCNAME[0]:-main}()] '
```

### Example Output:
```
+ [deploy.sh:12 in setup_env()] mkdir -p /opt/app/build
+ [deploy.sh:13 in setup_env()] cd /opt/app/build
+ [deploy.sh:25 in main()] deploy_binary web-prod
```
Now you can immediately see which line number and function executed every command!

---

## 5. Automated Static Linting with ShellCheck

**ShellCheck** is the industry-standard static analysis tool for shell scripts. It finds subtle bugs, unquoted variables, portability issues, and security vulnerabilities that human code review often misses:

```bash
# Install ShellCheck
sudo apt install -y shellcheck   # Ubuntu/Debian
sudo dnf install -y ShellCheck   # RHEL/Fedora

# Run analysis
shellcheck deploy.sh
```

```
In deploy.sh line 14:
tar -czf $BACKUP_DIR/data.tar.gz $SRC_DIR
         ^---------^ SC2086: Double quote to prevent globbing and word splitting.
                                 ^------^ SC2086: Double quote to prevent globbing and word splitting.

In deploy.sh line 22:
if [ $COUNT == 10 ]; then
            ^-- SC2039: In POSIX sh, == is undefined. Use = instead.
```

---

## 6. Summary Takeaways

- Test syntax safely without executing code using `bash -n script.sh`.
- Trace variable expansion in real time using `set -x` and turn off with `set +x`.
- Enhance trace readability with custom `PS4` line numbers: `PS4='+ [${LINENO}] '`.
- Run `shellcheck` in your CI/CD pipeline to catch shell bugs automatically.
