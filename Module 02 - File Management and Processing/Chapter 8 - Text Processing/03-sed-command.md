# The `sed` Command in Linux: Stream Editor & Automated Text Transformation

## Overview

**`sed`** (Stream Editor) is a non-interactive text editor that processes input streams or files line by line, applies transformation rules defined in a script, and outputs the result to standard output or directly modifies files in place.

Unlike interactive editors like Vim or Nano where a human opens and navigates a file, `sed` is **fully programmable and automated**. In DevOps, CI/CD pipelines, Docker container builds, and cloud-init provisioning scripts, `sed` is the primary workhorse used to dynamically modify configuration files, substitute environment parameters, strip headers, and sanitize data.

---

## 1. How `sed` Works: The Pattern Space

`sed` operates in a continuous read-evaluate-print loop using two memory buffers:
1. **Pattern Space**: The primary working buffer. `sed` reads a line of input, strips the trailing newline, places it into the pattern space, executes your commands against it, and (unless `-n` is set) prints the buffer before reading the next line.
2. **Hold Space**: A secondary clipboard buffer used in advanced multi-line scripts.

```text
Input File / Stream
         │
         ▼
[ Read Line into Pattern Space ] ──> [ Execute sed Commands ] ──> [ Output to stdout / File ]
         ▲                                                                      │
         └────────────────── Loop to next line ─────────────────────────────────┘
```

---

## 2. Command Syntax and Essential Flags

```bash
sed [OPTIONS] 'COMMANDS' [FILE...]
```

### Essential Flags

| Flag | Long Option | Description | DevOps Practical Use Case |
| :--- | :--- | :--- | :--- |
| `-i[SUFFIX]` | `--in-place` | **Modifies the file in-place** instead of printing to stdout. | Directly updating `/etc/hosts` or Kubernetes manifests in CI/CD. |
| `-n` | `--quiet` | Suppresses automatic printing of pattern space. | Printing only lines that match specific criteria (`sed -n '/pattern/p'`). |
| `-E` | `--regexp-extended` | Uses Extended Regular Expressions (ERE) avoiding backslash escapes. | Writing clean regexes with `+`, `?`, `|`, and `()`. |
| `-e SCRIPT` | `--expression` | Allows chaining multiple distinct sed expressions in a single command. | Executing search-replace and line deletions in one pass. |

---

## 3. The Substitution Command: `s/pattern/replacement/flags`

The substitution command `s` is the most widely used feature of `sed`.

```text
s / search_pattern / replacement_text / flags
 │        │                  │           │
[s]    [Regex]          [New String]   [g, i, p]
```

### Flags Reference
- `g` : **Global**. Replaces *all* occurrences on each line (default without `g` is to replace only the *first* occurrence).
- `i` or `I` : **Case-insensitive** matching.
- `p` : Prints the pattern space if a substitution took place (used with `-n`).
- `2` *(or any number N)* : Replaces only the `N`-th occurrence on the line.

### Changing Delimiters (Preventing Leaning Toothpicks)
When modifying URLs, paths, or file locations, using slashes `/` requires escaping every directory separator (`\/usr\/local\/bin`). `sed` allows you to use **any character** (such as `#`, `|`, or `@`) as the delimiter:

```bash
# Hard to read:
sed -i 's/\/var\/www\/html/\/var\/www\/v2/g' nginx.conf

# Clean and readable (using # as delimiter):
sed -i 's#/var/www/html#/var/www/v2#g' nginx.conf
```

---

## 4. Addressing Lines: Targeting Specific Locations

You can restrict where `sed` applies its commands:

### By Line Number
```bash
# Replace only on line 10:
sed '10s/debug = true/debug = false/' app.ini

# Replace between line 5 and line 20:
sed '5,20s/localhost/127.0.0.1/g' config.env

# Replace from line 50 to the end of the file ($):
sed '50,$s/old/new/g' file.txt
```

### By Pattern Matching
```bash
# Apply substitution only to lines that start with "DATABASE_URL":
sed '/^DATABASE_URL/s/localhost/db.production.internal/' .env
```

---

## 5. Deleting, Inserting, and Extracting Lines

### A. Deleting Lines (`d`)
```bash
# Delete line 5:
sed '5d' file.txt

# Delete lines 10 through 25:
sed '10,25d' file.txt

# Delete all lines starting with # (comments):
sed '/^#/d' config.conf

# Delete all empty lines:
sed '/^$/d' config.conf

# Combine both into a single clean configuration output:
sed -e '/^#/d' -e '/^$/d' /etc/ssh/sshd_config
```

### B. Extracting Line Ranges (`-n 'p'`)
Extract lines 40 to 60 without loading the entire document into an editor:
```bash
sed -n '40,60p' /var/log/syslog
```

### C. Inserting and Appending Text (`i` and `a`)
```bash
# Insert text BEFORE line 1 (prepends a banner):
sed -i '1i # Managed automatically by Ansible - DO NOT EDIT MANUALLY' /etc/motd

# Append text AFTER matching line:
sed -i '/\[Service\]/a Restart=always\nRestartSec=5' /etc/systemd/system/app.service
```

---

## 6. Real-World DevOps Automation Recipes

### Recipe 1: Automated Version Bump in CI/CD
Update the image tag in a Kubernetes deployment manifest during a GitHub Actions or GitLab CI workflow:

```bash
NEW_VERSION="v2.4.1"
sed -i -E "s#(image: myregistry.io/api:).*#\1$NEW_VERSION#" k8s/deployment.yaml
```

### Recipe 2: Capturing Groups and Backreferences (`\1`, `\2`)
Swap key-value pairs or reformat date strings:

```bash
# Input: 2026-10-14
# Reformat YYYY-MM-DD to DD/MM/YYYY:
echo "2026-10-14" | sed -E 's/([0-9]{4})-([0-9]{2})-([0-9]{2})/\3\/\2\/\1/'
# Output: 14/10/2026
```

### Recipe 3: The Cross-Platform CI/CD Pitfall: GNU `sed` vs macOS (BSD) `sed`
A classic failure in continuous integration pipelines occurs when a developer writes a script on macOS that runs on an Ubuntu runner:

- **GNU Linux `sed`**: `sed -i 's/old/new/' file.txt` (Modifies in-place without backup).
- **BSD macOS `sed`**: `sed -i '' 's/old/new/' file.txt` (Requires empty string for no backup).

```bash
# Cross-platform safe bash wrapper for in-place editing:
if [[ "$OSTYPE" == "darwin"* ]]; then
    sed -i '' 's/localhost/127.0.0.1/g' config.env
else
    sed -i 's/localhost/127.0.0.1/g' config.env
fi
```

---

## Summary

- `sed` is a stream editor designed for non-interactive, automated text transformation.
- The standard substitution command is `sed -i 's#old#new#g' file`.
- Always change delimiters (`s#pattern#replacement#g`) when editing URLs or filesystem paths.
- Delete comments and blank lines using `sed -e '/^#/d' -e '/^$/d' file`.
- Extract precise line slices using `sed -n 'N,Mp' file`.
- In production pipelines, always test your sed regex against stdout before enabling `-i` (in-place).
