# Chapter 29: Control Flow - For Loops in Bash

## 1. Introduction: Iteration in Shell Scripting

The **`for` loop** is the primary looping construct used when the set of items or the exact number of iterations is known before the loop starts.

In DevOps automation, `for` loops are constantly employed to:
- Iterate across lists of servers, IP addresses, or Kubernetes pods.
- Process collections of log files or backup archives matched by globs.
- Execute numbered retries or sequence steps.

Bash supports two distinct styles of `for` loops:
1. **List / Collection-based `for` loop** (Standard Shell style).
2. **C-Style three-expression `for` loop** (`for (( i=0; i<N; i++ ))`).

---

## 2. List-Based For Loops

**Syntax:**
```bash
for item in [list_of_items...]; do
    # Commands using "$item"
done
```

### 1. Iterating Over Static Lists
```bash
for env in dev staging production; do
    echo "Configuring infrastructure for environment: ${env}"
done
```

### 2. Iterating Over Array Elements (Best Practice)
```bash
REGIONS=("us-east-1" "us-west-2" "eu-west-1")

for region in "${REGIONS[@]}"; do
    echo "Deploying AMI image to AWS region: ${region}"
done
```

### 3. Iterating Over Positional Script Arguments (`"$@"`)
```bash
# If 'in "$@"' is omitted, 'for arg;' automatically iterates over "$@"!
for file in "$@"; do
    echo "Processing uploaded file: ${file}"
done
```

---

## 3. Iterating Over Files with Globs (Safe vs. Unsafe)

A very common requirement is processing files matching a pattern (e.g., all `.log` files in `/var/log`):

```bash
# SAFE: Direct Shell Globbing
for log_file in /var/log/nginx/*.log; do
    # Guard against case where no files match the glob
    [[ -e "${log_file}" ]] || continue
    echo "Compressing archive: ${log_file}"
    gzip -9 "${log_file}"
done
```

> [!WARNING]
> **Never iterate over `ls` output! (`for f in $(ls *.log)`)**
> 
> Parsing the output of `ls` is a dangerous shell scripting anti-pattern. If any file contains spaces or special characters (e.g., `audit report 2026.log`), word-splitting causes the loop to treat it as multiple separate files! 
> 
> **Always use direct shell globs (`for f in /path/*.log; do`)**.

---

## 4. Range and Sequence Generation

### Method 1: Brace Expansion `{START..END[..STEP]}`
Bash built-in brace expansion executes entirely in memory without spawning external subshell binaries:

```bash
# Count 1 to 5
for i in {1..5}; do
    echo "Step ${i}"
done

# Step by 10s: 0, 10, 20, 30, 40, 50
for i in {0..50..10}; do
    echo "Progress: ${i}%"
done
```

### Method 2: The `seq` Utility (When ranges are dynamic variables)
Brace expansion cannot evaluate variables (`{1..$MAX}` does not work). When range limits are dynamic variables, use `seq`:

```bash
MAX_REPLICAS=3

for i in $(seq 1 "${MAX_REPLICAS}"); do
    echo "Starting worker pod replica #${i}..."
done
```

---

## 5. C-Style For Loops (`(( ; ; ))`)

When counting or managing integer indices directly, the C-style syntax provides precise control over initialization, conditions, and increments:

```bash
for (( i = 0; i < 5; i++ )); do
    echo "Index: ${i}"
done
```

### Reversing / Decrementing:
```bash
for (( countdown = 10; countdown > 0; countdown-- )); do
    echo "Server rebooting in ${countdown} seconds..."
    sleep 1
done
```

---

## 6. Summary Takeaways

- Use `for item in "${arr[@]}"` to safely iterate arrays.
- Always use direct shell globs (`*.log`) instead of `$(ls)`.
- Use brace expansion `{1..10}` for static numeric ranges.
- Use C-style `for (( i=0; i<n; i++ ))` for numeric counting and indexed iteration.
