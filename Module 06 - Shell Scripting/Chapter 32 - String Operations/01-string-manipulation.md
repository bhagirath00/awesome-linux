# Chapter 32: String Operations - Bash Parameter Expansion & String Manipulation

## 1. Introduction: Why Native Parameter Expansion Matters

In shell scripting, developers often rely heavily on external utilities like `sed`, `awk`, `cut`, or `tr` to manipulate text:
```bash
# Slow external process execution:
FILE_EXT=$(echo "database.backup.tar.gz" | awk -F'.' '{print $NF}')
```
Every time you invoke an external utility inside a subshell (`$(...)`), the operating system kernel must:
1. Fork a child process (`fork()`).
2. Search the filesystem `PATH` for the binary.
3. Allocate memory and load binary libraries (`execve()`).
4. Read the pipe and teardown the process.

Inside a loop running 10,000 iterations, spawning external processes takes minutes. In contrast, **Bash Parameter Expansion** executes **entirely in memory inside the shell process**, executing in microseconds with zero process creation overhead.

---

## 2. String Length: `${#VAR}`

To calculate the number of characters in a string:

```bash
API_TOKEN="sec-981248012481023948"

# Get string length
echo "${#API_TOKEN}"
# Output: 22
```

---

## 3. Substring Extraction (Slicing): `${VAR:offset:length}`

Extract subsets of a string by character indices:

```bash
IMAGE_TAG="frontend-v2.4.1-alpine"

# Extract 8 characters starting at index 0 (Prefix):
echo "${IMAGE_TAG:0:8}"
# Output: frontend

# Extract starting at index 9 to the end of string:
echo "${IMAGE_TAG:9}"
# Output: v2.4.1-alpine

# Negative indexing: Extract the last 6 characters (space or parentheses required):
echo "${IMAGE_TAG: -6}"
# Output: alpine
```

---

## 4. Pattern Stripping (Prefix and Suffix Trimming)

Parameter expansion provides four operators to trim text based on glob patterns:

| Operator | Action | Mnemonic | Example String: `URL="https://api.company.com/v1/users"` | Result |
| :--- | :--- | :--- | :--- | :--- |
| **`${VAR#pattern}`** | Remove **shortest matching prefix** | `#` is near the front (start) | `${URL#*/}` | `/api.company.com/v1/users` |
| **`${VAR##pattern}`**| Remove **longest matching prefix** | `##` aggressively eats prefix | `${URL##*/}` | `users` |
| **`${VAR%pattern}`** | Remove **shortest matching suffix** | `%` is near the end | `${URL%/*}` | `https://api.company.com/v1` |
| **`${VAR%%pattern}`**| Remove **longest matching suffix** | `%%` aggressively eats suffix | `${URL%%/*}` | `https:` |

### The Classic DevOps Use Case: Parsing Paths & Filenames
Without invoking `basename` or `dirname`:

```bash
FILEPATH="/opt/deployments/releases/app-v2.1.tar.gz"

# 1. Extract Directory Path (equivalent to dirname):
DIR_NAME="${FILEPATH%/*}"
echo "Directory: ${DIR_NAME}"
# Output: /opt/deployments/releases

# 2. Extract Full Filename (equivalent to basename):
FILE_NAME="${FILEPATH##*/}"
echo "Filename:  ${FILE_NAME}"
# Output: app-v2.1.tar.gz

# 3. Extract File Extension:
EXT="${FILE_NAME#*.}"
echo "Extension: ${EXT}"
# Output: tar.gz

# 4. Extract Base Name without extension:
BASE_NAME="${FILE_NAME%%.*}"
echo "Base Name: ${BASE_NAME}"
# Output: app-v2
```

---

## 5. Search and Replace

Bash allows search-and-replace using glob patterns:

```bash
DATA="server-prod-01,server-prod-02,server-prod-03"

# Replace FIRST occurrence: ${VAR/pattern/replacement}
echo "${DATA/prod/staging}"
# Output: server-staging-01,server-prod-02,server-prod-03

# Replace ALL occurrences: ${VAR//pattern/replacement}
echo "${DATA//prod/staging}"
# Output: server-staging-01,server-staging-02,server-staging-03

# Delete ALL matching patterns (leave replacement empty):
echo "${DATA//-prod/}"
# Output: server-01,server-02,server-03
```

---

## 6. Case Conversion (Bash 4.0+)

Convert strings to uppercase or lowercase natively:

```bash
ENV_NAME="staging"

# Convert to UPPERCASE using ^^
echo "${ENV_NAME^^}"
# Output: STAGING

# Convert to LOWERCASE using ,,
USER_INPUT="ADMIN"
echo "${USER_INPUT,,}"
# Output: admin

# Capitalize first letter only using ^
BRANCH="feature"
echo "${BRANCH^}"
# Output: Feature
```

---

## 7. Default and Fallback Values

Handle unset or empty variables gracefully:

| Syntax | Behavior | Example |
| :--- | :--- | :--- |
| **`${VAR:-default}`** | Return `default` if `VAR` is unset or empty, but **do not assign**. | `PORT="${PORT:-8080}"` |
| **`${VAR:=default}`** | If `VAR` is unset or empty, **assign** `default` to `VAR`. | `${DEPLOY_ENV:=staging}` |
| **`${VAR:?error_msg}`**| Exit script with `error_msg` if `VAR` is unset or empty. | `${AWS_SECRET_KEY:?Error: AWS Secret required!}` |
| **`${VAR:+alternate}`**| If `VAR` is set, return `alternate`; otherwise return empty. | `${DEBUG:+--verbose}` |

---

## 8. Summary Takeaways

- Use parameter expansion instead of `sed`/`awk`/`basename` to eliminate process overhead.
- `${VAR#*pat}` and `${VAR##*pat}` strip prefixes from the start.
- `${VAR%pat*}` and `${VAR%%pat*}` strip suffixes from the end.
- Use `${VAR//find/replace}` for global search and replace.
- Use `${VAR^^}` for uppercase, `${VAR,,}` for lowercase.
- Use `${VAR:-default}` to provide robust configuration fallbacks.
