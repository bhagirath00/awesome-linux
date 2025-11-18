# The `tr` Command in Linux: Translating, Deleting, and Squeezing Characters

## Overview

The `tr` utility (short for **Translate**) reads characters from standard input, transforms or deletes them, and writes the result to standard output.

`tr` is a pure stream filter. Unlike commands like `cat`, `grep`, or `sed`, **`tr` does NOT accept file paths as arguments**—it exclusively processes standard input (`stdin`) via pipes (`|`) or shell redirection (`<`).

In DevOps workflows, `tr` is the fastest and most lightweight tool for sanitizing Windows line endings, changing letter casing, compressing repeated whitespace, reformatting delimited data, and generating cryptographically secure random passwords from `/dev/urandom`.

---

## Command Syntax and Anatomy

```bash
tr [OPTIONS] SET1 [SET2] < input_file
# Or via pipeline:
cat input_file | tr [OPTIONS] SET1 [SET2]
```

- **`SET1`**: Characters to search for or match.
- **`SET2`**: Replacement characters mapped 1-to-1 with `SET1`.

---

## Essential `tr` Flags Reference

| Flag | Long Option | Description | DevOps Practical Use Case |
| :--- | :--- | :--- | :--- |
| **`-d`** | `--delete` | Deletes all characters specified in `SET1`. | **Stripping Windows carriage returns (`\r`).** |
| **`-s`** | `--squeeze-repeats` | Squeezes consecutive duplicate instances of a character down to a single instance. | Collapsing multiple spaces into single spaces. |
| **`-c`, `-C`** | `--complement` | Inverts `SET1` to match all characters **NOT** in the set. | Sanitizing input to strictly allow alphanumeric characters. |
| **`-cd`** | *(combined)* | **Delete everything EXCEPT `SET1`**. | **Generating passwords and tokens from `/dev/urandom`.** |

---

## 1. Character Translation (Case Conversion)

### Converting Lowercase to Uppercase
```bash
$ echo "kubernetes cluster production" | tr 'a-z' 'A-Z'
KUBERNETES CLUSTER PRODUCTION
```

### Using POSIX Character Classes
For universal international language support:
```bash
$ echo "database_backup_2026" | tr '[:lower:]' '[:upper:]'
DATABASE_BACKUP_2026
```

---

## 2. Deleting Characters (`-d`)

### The #1 DevOps Use Case: Removing Windows CRLF (`\r`)
When shell scripts edited on Windows fail on Linux with `/bin/bash^M: bad interpreter`:

```bash
tr -d '\r' < entrypoint.sh > entrypoint-clean.sh && mv entrypoint-clean.sh entrypoint.sh
chmod +x entrypoint.sh
```

### Stripping Quotes and Delimiters
```bash
# Strip quotation marks from JSON/CSV exports:
echo '"web-server-01","10.0.1.5"' | tr -d '"'
# Output: web-server-01,10.0.1.5
```

---

## 3. Squeezing Repeated Characters (`-s`)

When processing unstructured command outputs or messy log files where columns are separated by irregular, varying numbers of spaces:

```bash
# Collapse multiple spaces down to a single space:
$ echo "nginx    running    port    80" | tr -s ' '
nginx running port 80

# Squeeze repeated blank lines down to a single newline:
tr -s '\n' < messy-config.txt > clean-config.txt
```

---

## 4. Complementing Sets (`-c`): The Password Generator Superpower

Combining **`-c` (complement)** with **`-d` (delete)** instructs `tr` to **delete every character in the universe EXCEPT what you explicitly listed**.

### Generating a Secure 16-Character Alphanumeric Password
Read raw cryptographic bytes from `/dev/urandom`, delete everything that is not a letter or digit, and slice the first 16 characters:

```bash
tr -dc 'A-Za-z0-9' < /dev/urandom | head -c 16; echo
# Output: K9vL3xR8zP1qM5sT
```

### Generating a 32-Character Complex API Secret
```bash
tr -dc 'A-Za-z0-9!@#$%^&*' < /dev/urandom | head -c 32; echo
# Output: 9f#xL2@mQ!8$zR^vP4&wK1*sD7#bT6%y
```

---

## 5. Converting Delimiters and Lists

### Convert Comma-Separated Values into Vertical Rows
```bash
# Great for passing CSV lists into a bash 'while read' loop:
$ echo "app,db,cache,worker" | tr ',' '\n'
app
db
cache
worker
```

### Convert Newlines into Space-Separated Lists
```bash
# Convert a list of Docker container IDs into a single command line:
docker ps -q | tr '\n' ' '
```

### Replace Spaces with Underscores (Filename Sanitization)
```bash
$ echo "Production Database Backup" | tr ' ' '_'
Production_Database_Backup
```

---

## Summary

- `tr` is a stream filter; it reads from `stdin` and writes to `stdout` (never pass filenames directly).
- Strip Windows carriage return (`^M`) line endings with `tr -d '\r' < file`.
- Convert casing using `tr 'a-z' 'A-Z'` or `tr '[:lower:]' '[:upper:]'`.
- Squeeze multiple consecutive spaces or blank lines using `tr -s ' '` or `tr -s '\n'`.
- Generate secure random passwords and API tokens using `tr -dc 'A-Za-z0-9' < /dev/urandom | head -c <LENGTH>`.
