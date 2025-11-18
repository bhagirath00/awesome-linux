# Executing Actions with `find`: `-exec`, `-delete`, and `xargs`

## Overview

Locating files with `find` is only the first half of systems automation. The true power of `find` lies in its ability to **take action** on matching files directly—deleting expired logs, normalizing permissions across nested web directories, compressing archives, or piping results into text-processing pipelines.

Understanding how `find` invokes external commands—specifically the difference between `-exec \;` and `-exec +`—is crucial for maintaining high server performance and preventing fork bombs in large filesystems.

---

## 1. Syntax of `-exec`

The `-exec` action allows executing any arbitrary Linux command on every file or directory that satisfies the search criteria:

```bash
find [PATH] [CRITERIA] -exec COMMAND {} \;
```

### Deconstructing the Syntax:
- **`{}` (Curly Braces)**: A placeholder automatically replaced by `find` with the path of the currently matched file.
- **`\;` (Escaped Semicolon)**: The termination marker signaling the end of the `-exec` command arguments. The backslash `\` is required so the shell passes the semicolon literally to `find` instead of treating it as a shell command separator.

```bash
# Example: Print file details for all .pem certificate files:
find /etc/ssl -type f -name "*.pem" -exec ls -lh {} \;
```

---

## 2. The Critical Performance Distinction: `-exec \;` vs `-exec +`

This is one of the most critical performance optimizations in Linux systems administration:

```text
find . -type f -name "*.log" -exec rm {} \;
  ├── File 1: spawns new 'rm' process (PID 1001) -> exits
  ├── File 2: spawns new 'rm' process (PID 1002) -> exits
  ├── File 3: spawns new 'rm' process (PID 1003) -> exits
  └── ... (If 50,000 files exist: 50,000 process forks! Extremely slow!)

find . -type f -name "*.log" -exec rm {} +
  └── Spawns 1 single 'rm' process: rm file1 file2 file3 ... file50000 (Instant!)
```

### Technical Breakdown

| Action | Behavior | Process Fork Overhead | Best Used For |
| :--- | :--- | :--- | :--- |
| **`-exec ... {} \;`** | Runs command **once per individual file**. | High (spawns 1 process per file). | Commands that strictly require a single file argument (e.g., specific rename tools). |
| **`-exec ... {} +`** | **Batches** multiple files into a single command invocation, respecting system `ARG_MAX`. | **Ultra-low (1 or 2 processes total)**. | Commands that accept multiple file arguments (`rm`, `chmod`, `chown`, `grep`, `tar`). |

> [!TIP]
> **Performance Rule**: Whenever possible, default to `-exec COMMAND {} +`. On large file trees with 100,000+ files, `-exec +` can be **50x to 100x faster** than `-exec \;`.

---

## 3. High-Frequency DevOps Recipes with `-exec`

### A. The Standard Web Permissions Normalization Pattern
In production web environments (Nginx, Apache, PHP-FPM, WordPress), security policies mandate that:
- **Directories** must be `755` (`drwxr-xr-x`) so users can enter them.
- **Files** must be `644` (`-rw-r--r--`) so they cannot be executed as binaries.

Executing `chmod -R 755` destroys security by making all files executable! Instead, use `find`:

```bash
# Normalize all directories to 755:
find /var/www/html -type d -exec chmod 755 {} +

# Normalize all files to 644:
find /var/www/html -type f -exec chmod 644 {} +
```

### B. Searching for Text Patterns Inside Specific File Types
Search for occurrences of the string `ssl_certificate` exclusively inside `.conf` files:

```bash
# -H prints the filename for each match:
find /etc/nginx -type f -name "*.conf" -exec grep -H "ssl_certificate" {} +
```

### C. Compressing Logs Older Than 7 Days
Locate uncompressed `.log` files older than 7 days and compress them using `gzip`:

```bash
find /var/log/app -type f -name "*.log" -mtime +7 -exec gzip {} +
```

---

## 4. The Built-in `-delete` Action

`find` provides a native `-delete` action. When deleting files found by criteria, `-delete` is significantly faster than invoking external `rm`:

```bash
# Delete all .tmp files older than 30 days:
find /var/cache -type f -name "*.tmp" -mtime +30 -delete
```

> [!WARNING]
> **Critical Rule for `-delete`**:
> `-delete` performs deletions **immediately upon evaluation**. Always place `-delete` as the **VERY LAST** argument in your command!
>
> If you write: `find /tmp -delete -name "*.log"`, `find` evaluates `-delete` first, deleting EVERYTHING in `/tmp` before it ever checks the name!
>
> Always test with `-print` first before switching to `-delete`:
> ```bash
> # Step 1: Verify matches safely:
> find /tmp -type f -name "*.log" -mtime +14
>
> # Step 2: Once verified, append -delete:
> find /tmp -type f -name "*.log" -mtime +14 -delete
> ```

---

## 5. Interactive Confirmation: `-ok`

When performing high-risk operations in production, replace `-exec` with `-ok`. `find` will prompt for confirmation before executing each command:

```bash
$ find /etc/ssl -type f -name "*.key" -ok chmod 600 {} \;
< chmod ... /etc/ssl/server.key > ? y
< chmod ... /etc/ssl/backup.key > ? n
```

---

## 6. Pairing `find` with `xargs` and Null Delimiters (`-print0`)

When processing millions of files through standard Unix pipelines, piping `find` into `xargs` is a common pattern. However, if any filename contains spaces, newlines, or quotes, standard pipelines will break or misinterpret arguments.

To make pipelines 100% resilient, use the **ASCII NUL (`\0`) delimiter**:

```bash
# -print0 prints filenames separated by a null byte
# xargs -0 reads arguments separated by a null byte
find /var/log -type f -name "*.log" -print0 | xargs -0 rm -f
```

---

## Summary

- Use `-exec COMMAND {} \;` to run a command once per matched file.
- Use `-exec COMMAND {} +` to batch arguments, reducing process fork overhead by up to 99%.
- Fix web directory and file permissions independently using `-type d -exec chmod 755 {} +` and `-type f -exec chmod 644 {} +`.
- Use the built-in `-delete` flag for high-speed file cleanup, always placing it at the very end of the command.
- Pair `find -print0` with `xargs -0` when streaming filenames containing whitespace or special characters.
