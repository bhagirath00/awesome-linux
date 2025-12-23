# Chapter 29: Control Flow - Loop Control with break and continue

## 1. Introduction: Loop Interruption Keywords

During loop execution, conditions frequently arise where the standard sequential flow must be altered:
- An unrecoverable error occurs, requiring the loop to **terminate immediately** before finishing all items.
- A specific record or file is invalid or ignored, requiring the script to **skip the current item** and jump straight to the next iteration.

Bash provides two built-in control keywords for this purpose:
- **`break`**: Instantly exits the enclosing loop entirely.
- **`continue`**: Skips the remainder of the current iteration and jumps to the top of the loop for the next cycle.

---

## 2. Using `break` to Terminate Loops

`break` is frequently used when searching for an item or when an operation succeeds/fails:

```bash
#!/usr/bin/env bash

SERVERS=("web-01" "web-02" "web-03" "web-04")
TARGET="web-03"

for server in "${SERVERS[@]}"; do
    echo "Pinging ${server}..."
    if [[ "${server}" == "${TARGET}" ]]; then
        echo "[FOUND] Located target server: ${server}! Halting scan."
        break
    fi
done

echo "Execution resumed after loop."
```

### Execution Output:
```
Pinging web-01...
Pinging web-02...
Pinging web-03...
[FOUND] Located target server: web-03! Halting scan.
Execution resumed after loop.
```
Notice that `web-04` was never evaluated; `break` halted the loop immediately.

---

## 3. Using `continue` to Skip Iterations

`continue` skips processing for the current cycle without terminating the entire loop:

```bash
#!/usr/bin/env bash

CONFIG_FILES=("app.conf" "db.conf" "test.tmp" "cache.conf" "old.bak")

for file in "${CONFIG_FILES[@]}"; do
    # Skip temporary and backup files
    if [[ "${file}" =~ \.(tmp|bak)$ ]]; then
        echo "--> Skipping temporary file: ${file}"
        continue
    fi

    echo "==> Deploying configuration: ${file}"
done
```

### Execution Output:
```
==> Deploying configuration: app.conf
==> Deploying configuration: db.conf
--> Skipping temporary file: test.tmp
==> Deploying configuration: cache.conf
--> Skipping temporary file: old.bak
```

---

## 4. Multi-Level Breaking in Nested Loops

When loops are nested inside other loops (e.g., iterating regions, and within each region iterating clusters), issuing plain `break` or `continue` affects only the **innermost loop**.

Both `break` and `continue` accept an optional numeric argument **`N`** specifying the number of nested loop levels to break out of:

```bash
for region in "us-east-1" "eu-central-1"; do
    echo "Checking region: ${region}"

    for cluster in 1 2 3; do
        echo "  Auditing cluster ${cluster}..."

        if [[ "${region}" == "us-east-1" && "${cluster}" -eq 2 ]]; then
            echo "  [ALERT] Critical fault in ${region} cluster ${cluster}! Aborting ALL regions."
            break 2   # Breaks out of BOTH the inner cluster loop AND the outer region loop!
        fi
    done
done

echo "Audit aborted."
```

```
Checking region: us-east-1
  Auditing cluster 1...
  Auditing cluster 2...
  [ALERT] Critical fault in us-east-1 cluster 2! Aborting ALL regions.
Audit aborted.
```

---

## 5. Summary Takeaways

- `break` exits the entire loop immediately.
- `continue` skips the rest of the current iteration and proceeds to the next item.
- Use `break N` and `continue N` to control multiple levels of nested loops.
- Use `continue` at the top of loops to cleanly filter out invalid files, empty lines, and comments.
