# Chapter 30: Functions - Function Return Values and Exit Codes

## 1. Introduction: The Concept of Returning in Bash

A common misconception among engineers transitioning from Python, JavaScript, or Go into shell scripting is expecting `return` in Bash to return data (like a string, an array, or an object).

**In Bash, `return` does NOT return data!**

In Bash, `return` sets an **Integer Exit Status Code** (`0` to `255`) indicating the success or failure of the function. 
- **`0`**: Success / True.
- **`1` to `255`**: Failure / Error condition.

```
+--------------------------------------------------------------------------+
|                     How to Return Data in Bash Functions                 |
+--------------------------------------------------------------------------+
|  Method 1: Exit Status (return N)                                        |
|  - Strictly an integer: 0 to 255.                                        |
|  - Used for success/failure boolean branching. Checked via $?.           |
|                                                                          |
|  Method 2: Command Substitution (echo / stdout)                          |
|  - Print data to stdout (echo / printf).                                 |
|  - Caller captures data using: RESULT=$(my_function)                     |
|                                                                          |
|  Method 3: Output Variable by Reference (declare -n)                     |
|  - Modifies a caller-provided variable name directly without subshells.  |
+--------------------------------------------------------------------------+
```

---

## 2. Method 1: Using `return` for Status Codes

The `return` statement immediately halts function execution and returns an integer status code stored in the **`$?`** special variable:

```bash
check_disk_space() {
    local threshold="$1"
    local usage
    usage=$(df / --output=pcent | tail -n 1 | tr -dc '0-9')

    if [[ "${usage}" -ge "${threshold}" ]]; then
        echo "[ALERT] Disk usage (${usage}%) exceeds threshold (${threshold}%)!" >&2
        return 1   # Failure code
    fi

    return 0       # Success code
}

# The caller evaluates the function directly in an 'if' block:
if check_disk_space 90; then
    echo "Disk space is healthy."
else
    echo "Triggering automated disk cleanup..."
fi
```

---

## 3. Method 2: Returning Data via Standard Output (`echo` & `$()`)

To return strings, JSON payloads, or calculated values, print the data to **`stdout`** and capture it at the call site using **Command Substitution (`$(...)`)**:

```bash
calculate_checksum() {
    local target_file="$1"

    if [[ ! -f "${target_file}" ]]; then
        echo "ERROR: File missing" >&2  # Send error messages to stderr (fd 2)!
        return 1
    fi

    local hash
    hash=$(sha256sum "${target_file}" | cut -d' ' -f1)

    # "Return" the value by printing to stdout:
    echo "${hash}"
    return 0
}

# Capture the returned output:
FILE_HASH=$(calculate_checksum "/etc/hosts")
echo "Generated SHA256: ${FILE_HASH}"
```

> [!IMPORTANT]
> When returning data via `echo`, **send all log and diagnostic messages to standard error (`>&2`)**! If you print informational logs to standard output, they will become concatenated into the returned variable value!

---

## 4. Method 3: Returning by Reference (`declare -n` / Nameref)

Command substitution (`$(...)`) spawns a child subshell, which introduces minor performance overhead and prevents modifying the parent's environment.

In modern Bash (4.3+), you can pass a variable **by reference** using `declare -n`:

```bash
get_aws_instance_metadata() {
    local instance_id="$1"
    # Create nameref pointing to caller's variable name passed in $2
    declare -n out_result="$2"

    # Mutate the caller's variable directly:
    out_result="i-08a41249b124 (us-east-1a: Running)"
}

# Caller declares variable:
INSTANCE_INFO=""

# Pass variable name 'INSTANCE_INFO' by reference:
get_aws_instance_metadata "i-08a41249b124" INSTANCE_INFO

echo "Retrieved Metadata: ${INSTANCE_INFO}"
```
```
Retrieved Metadata: i-08a41249b124 (us-east-1a: Running)
```
No subshell was created, and the value was returned cleanly into `INSTANCE_INFO`!

---

## 5. Summary Takeaways

- `return` is strictly for integer exit status codes (`0` = success, non-zero = failure).
- Never attempt to return strings with `return "my_string"` (Bash will fail or evaluate to 0).
- To return strings or data, print to stdout (`echo`) and capture with `VAL=$(func)`.
- Divert diagnostic logs inside functions to `stderr` (`>&2`) to avoid polluting returned data.
- Use namerefs (`declare -n`) in high-performance scripts to return data by reference without subshells.
