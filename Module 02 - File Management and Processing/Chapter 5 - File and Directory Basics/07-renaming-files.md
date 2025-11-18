# Renaming Files and Directories in Linux

## Overview

Renaming files is a frequent operation in systems administration and DevOps automation. While renaming a single file is handled simply with `mv`, managing bulk renames—such as standardizing hundreds of log files, stripping unwanted prefixes, changing file extensions across nested directories, or sanitizing filenames for cloud storage buckets (e.g., AWS S3)—requires specialized utilities and bash scripting techniques.

---

## 1. Single File Renaming: `mv`

In Linux, there is no separate command named `rename-file`. Instead, the `mv` command serves as the standard tool to rename single files or folders within the same directory:

```bash
# Rename file
mv server.conf.dist server.conf

# Rename directory
mv /app/v1 /app/v2
```

---

## 2. Bulk Renaming with Native Bash Parameter Expansion

When working in environments where third-party utilities cannot be installed (e.g., minimal Docker containers or restricted bastion hosts), native Bash parameter expansion is the fastest and most portable solution.

### Anatomy of Bash Parameter Expansions

| Syntax | Meaning | Example (`f="app-2026.log"`) | Result |
| :--- | :--- | :--- | :--- |
| `${f%.*}` | Remove shortest match from suffix (strips extension) | `${f%.*}` | `app-2026` |
| `${f##*.}`| Keep only the extension | `${f##*.}` | `log` |
| `${f#*-}` | Remove shortest match from prefix | `${f#*-}` | `2026.log` |
| `${f//search/replace}` | Replace all occurrences of pattern | `${f//2026/2027}` | `app-2027.log` |
| `${f,,}` | Convert to lowercase (Bash 4+) | `f="DATA.TXT"; echo "${f,,}"` | `data.txt` |
| `${f^^}` | Convert to uppercase (Bash 4+) | `f="data.txt"; echo "${f^^}"` | `DATA.TXT` |

### Practical Bash Renaming Recipes

#### Example A: Changing File Extensions in Bulk (`.htm` -> `.html`)
```bash
for file in *.htm; do
    [ -e "$file" ] || continue
    mv -- "$file" "${file%.htm}.html"
done
```

#### Example B: Prepending a Timestamp or Environment Prefix
```bash
# Prepend "backup-" to all SQL dumps
for file in *.sql; do
    [ -e "$file" ] || continue
    mv -- "$file" "backup-$(date +%F)-$file"
done
```

#### Example C: Replacing Spaces with Underscores (S3 Sanitization)
AWS S3 object keys with spaces can break URL parsers and CI/CD webhook URLs:
```bash
for file in *\ *; do
    [ -e "$file" ] || continue
    mv -- "$file" "${file// /_}"
done
```

---

## 3. The `rename` Utility (Perl-based)

On Debian, Ubuntu, and derivatives, the powerful Perl-based `rename` command supports full Regular Expression (PCRE) substitutions.

### Syntax
```bash
rename [OPTIONS] 's/PATTERN/REPLACEMENT/' FILES...
```

### Options
- `-n` : **Dry-run mode**. Prints what would be renamed without making changes (ALWAYS run this first!).
- `-v` : Verbose mode; shows successful renames.
- `-f` : Force overwriting existing destination files.

### Practical `rename` Examples

#### Dry Run Test First
```bash
rename -n 's/\.jpeg$/\.jpg/' *.jpeg
# Output:
# rename(image1.jpeg, image1.jpg)
# rename(image2.jpeg, image2.jpg)
```

#### Execute the Real Rename
```bash
rename -v 's/\.jpeg$/\.jpg/' *.jpeg
```

#### Strip Prefix from Files
```bash
# Rename 'draft-report.pdf' -> 'report.pdf'
rename 's/^draft-//' draft-*.pdf
```

#### Lowercase All File Names
```bash
rename 'y/A-Z/a-z/' *
```

---

## 4. Recursive Bulk Renaming with `find`

When target files reside in nested subdirectories across an entire repository:

```bash
# Recursively rename all .yaml files to .yml
find /app/kubernetes -type f -name "*.yaml" -exec bash -c '
    for file; do
        mv "$file" "${file%.yaml}.yml"
    done
' bash {} +
```

---

## Summary

- Use `mv oldname newname` for individual file or directory renames.
- Use native Bash parameter expansion (`${f%.*}`, `${f// /_}`) for portable bulk renaming without third-party dependencies.
- On Ubuntu/Debian, use `rename 's/pattern/replacement/' *.ext` for regex-based bulk renaming.
- Always perform a dry run (`-n` flag or `echo mv ...`) before executing destructive mass renames in production.
