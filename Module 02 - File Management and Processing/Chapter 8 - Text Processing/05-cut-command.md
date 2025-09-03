# The `cut` Command in Linux: Slicing Columns and Character Streams

## Overview

The `cut` utility is a lightweight, high-performance Unix command designed to remove sections, extract columns, and slice byte/character ranges from each line of a file or data stream.

While `awk` is a full programming language capable of complex logic, `cut` is a dedicated, single-purpose binary. For simple field extraction tasks on delimited files (such as `/etc/passwd` or CSV exports), `cut` is **substantially faster, consumes fewer CPU cycles, and requires much simpler syntax**.

---

## Command Syntax and Anatomy

```bash
cut [OPTIONS] [FILE...]
```

Every `cut` invocation must specify exactly one slicing mode:
- **`-f` (Fields)**: Extract columns separated by a delimiter.
- **`-c` (Characters)**: Extract specific character offsets.
- **`-b` (Bytes)**: Extract specific byte positions.

---

## 1. Extracting Delimited Fields (`-d` and `-f`)

The most common operational mode for `cut` is splitting lines by a delimiter character.

### Specifying Field Ranges
- `-f 1` : First column only.
- `-f 1,3` : Columns 1 and 3.
- `-f 2-5` : Columns 2, 3, 4, and 5 (inclusive range).
- `-f 3-` : Column 3 to the end of the line.
- `-f -4` : Start of line through column 4.

### Delimiter (`-d`)
By default, `cut` uses a **Tab (`\t`)** character as its field delimiter. To split by colon, comma, or pipe, specify `-d`:

```bash
# Extract username (field 1) and user shell (field 7) from /etc/passwd:
$ cut -d':' -f1,7 /etc/passwd | head -n 5
root:/bin/bash
daemon:/usr/sbin/nologin
bin:/usr/sbin/nologin
sys:/usr/sbin/nologin
ubuntu:/bin/bash
```

### Changing the Output Delimiter (`--output-delimiter`)
You can transform delimiters on the fly. For instance, convert colon-delimited data to comma-separated CSV format:

```bash
$ cut -d':' -f1,3,7 --output-delimiter=',' /etc/passwd | head -n 3
root,0,/bin/bash
daemon,1,/usr/sbin/nologin
bin,2,/usr/sbin/nologin
```

### Inverting Selections: `--complement`
Print all fields **except** the ones specified:

```bash
# Print all columns EXCEPT the password hash placeholder ($2):
cut -d':' --complement -f2 /etc/passwd
```

---

## 2. Extracting by Character Position (`-c`)

When dealing with fixed-width output—such as timestamps, git commit hashes, or fixed-width legacy mainframe exports:

### Extract Short Git Commit SHAs (First 7 Characters)
```bash
$ git rev-parse HEAD | cut -c 1-7
7f2a18b
```

### Slice ISO 8601 Timestamps
```bash
# Extract only the YYYY-MM-DD date from an ISO timestamp string
$ echo "2026-10-14T09:45:00Z" | cut -c 1-10
2026-10-14
```

---

## 3. The Major Limitation: `cut` vs Variable Whitespace

A classic beginner mistake is attempting to use `cut` on commands like `ps aux`, `df -h`, or `ls -l`:

```bash
# ATTEMPTING TO CUT df -h:
$ df -h | cut -d' ' -f1
Filesystem
/dev/root



tmpfs
```

### Why does this fail?
`cut` supports **only a single delimiter character**. When commands output table rows formatted with *variable amounts of spacing* (e.g. 3 spaces between column 1 and column 2), `cut` treats every single space as a separate empty field!

### The Rule of Thumb: `cut` vs `awk`
- If the file uses a **strict, single-character delimiter** (colon `:` in `/etc/passwd`, comma `,` in CSV, tab in TSV) ──> **Use `cut`** (faster, cleaner).
- If the file uses **variable spaces or multiple whitespace characters** (like `ps aux`, `df -h`, `docker ps`, `netstat`) ──> **Use `awk`** (which automatically collapses consecutive whitespace).

```bash
# Clean extraction using awk:
df -h | awk '{print $1, $5}'
```

---

## Summary

- Use `cut -d'DELIM' -f FIELDS` to extract specific columns from delimited text.
- Use `cut -c START-END` to slice fixed-width strings and hashes.
- Reformat output separators instantly using `--output-delimiter="CHAR"`.
- Use `--complement` to output all columns except the excluded ones.
- Never use `cut -d' '` on variable-whitespace CLI tables (`ps`, `df`); use `awk` instead.
