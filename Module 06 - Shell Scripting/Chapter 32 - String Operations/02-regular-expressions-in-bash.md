# Chapter 32: String Operations - Regular Expressions in Bash

## 1. Introduction: The Regex Matching Operator (`=~`)

In modern Bash (within the `[[ ... ]]` conditional block), you can evaluate strings against **POSIX Extended Regular Expressions (ERE)** natively using the **`=~`** operator.

This eliminates the need to spawn external processes like `grep`, `sed`, or `awk` just to validate user input, verify IP address formats, or parse semver strings.

```bash
if [[ "$STRING" =~ REGEX_PATTERN ]]; then
    echo "String matched pattern!"
fi
```

---

## 2. The Golden Rule of Bash Regex: DO NOT QUOTE THE REGEX!

The single most common bug when writing regex in Bash:

```bash
# WRONG (Treated as a literal string, NOT a regex!):
if [[ "$EMAIL" =~ "^[a-z]+@[a-z]+\.[a-z]+$" ]]; then ...

# CORRECT (Unquoted regex):
if [[ "$EMAIL" =~ ^[a-z]+@[a-z]+\.[a-z]+$ ]]; then ...
```

> [!WARNING]
> In Bash 3.2 and higher, wrapping a regular expression in quotes forces Bash to treat all special characters (`^`, `*`, `+`, `?`, `[ ]`) as **literal characters**!
> 
> **Best Practice Recommendation**: To avoid escaping and quoting headaches, **always store complex regular expressions in a separate variable**:
> ```bash
> REGEX='^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
> if [[ "$EMAIL" =~ $REGEX ]]; then
>     echo "Valid email address."
> fi
> ```
> When referencing the variable `$REGEX`, **leave it unquoted** inside `[[ ]]`.

---

## 3. Capturing Match Groups: The `BASH_REMATCH` Array

When a regular expression contains capturing parentheses `( ... )`, Bash automatically populates a built-in array variable named **`BASH_REMATCH`**:

- **`${BASH_REMATCH[0]}`**: Contains the **entire string match**.
- **`${BASH_REMATCH[1]}`**: Contains the text matched by the **first capture group `()`**.
- **`${BASH_REMATCH[2]}`**: Contains the text matched by the **second capture group `()`**.

### Example: Parsing Semantic Version Strings (SemVer)
```bash
#!/usr/bin/env bash
VERSION="v2.14.7-beta.1"

# Regex pattern with 4 capture groups: Major, Minor, Patch, Pre-release
SEMVER_REGEX='^v?([0-9]+)\.([0-9]+)\.([0-9]+)(-([a-zA-Z0-9.]+))?$'

if [[ "${VERSION}" =~ ${SEMVER_REGEX} ]]; then
    echo "Full Match    : ${BASH_REMATCH[0]}"
    echo "Major Version : ${BASH_REMATCH[1]}"
    echo "Minor Version : ${BASH_REMATCH[2]}"
    echo "Patch Version : ${BASH_REMATCH[3]}"
    echo "Pre-release   : ${BASH_REMATCH[5]:-None}"
else
    echo "[ERROR] Invalid version format: ${VERSION}" >&2
    exit 1
fi
```

### Execution Output:
```
Full Match    : v2.14.7-beta.1
Major Version : 2
Minor Version : 14
Patch Version : 7
Pre-release   : beta.1
```

---

## 4. Practical DevOps Input Validation Examples

### 1. Validating an IPv4 Address
```bash
validate_ipv4() {
    local ip="$1"
    local octet="(25[0-5]|2[0-4][0-9]|1[0-9][0-9]|[1-9]?[0-9])"
    local ip_regex="^${octet}\.${octet}\.${octet}\.${octet}$"

    if [[ "${ip}" =~ ${ip_regex} ]]; then
        return 0
    else
        echo "[ERROR] Invalid IPv4 address: '${ip}'" >&2
        return 1
    fi
}
```

### 2. Validating AWS S3 Bucket Name Constraints
```bash
validate_s3_bucket() {
    local bucket="$1"
    # S3 rules: 3-63 chars, lowercase, numbers, hyphens, start/end alphanumeric
    local s3_regex='^[a-z0-9][a-z0-9-]{1,61}[a-z0-9]$'

    if [[ "${bucket}" =~ ${s3_regex} ]]; then
        return 0
    fi
    return 1
}
```

---

## 5. Summary Takeaways

- Use the `=~` operator inside `[[ ... ]]` for native regex matching without spawning `grep`.
- Never put quotes around the regular expression inside `[[ ]]`.
- Store complex regex patterns inside a separate variable (`REGEX='...'`).
- Extract matched capture groups using the `${BASH_REMATCH[@]}` array.
- Use regex validation at the entrypoint of scripts to prevent malformed arguments from causing deployment failures.
