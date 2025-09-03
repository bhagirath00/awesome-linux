# The `sort` Command in Linux: Sorting Text, Numbers, and Multi-Column Data

## Overview

The `sort` command reorders lines in text files and streams according to defined collation rules: alphabetically, numerically, by human-readable storage units (K, M, G), by month, or by software version numbers.

In Unix pipeline philosophy, `sort` is a critical bridge utility. Commands like `uniq` require strictly sorted input to detect duplicates. In DevOps operations, `sort` is used daily for analyzing disk usage dumps, auditing user accounts by UID, ranking web server traffic, and sorting Git deployment tags.

---

## Command Syntax and Anatomy

```bash
sort [OPTIONS] [FILE...]
```

When run without flags, `sort` performs an **ASCII alphabetical sort**.

---

## Essential `sort` Flags Reference

| Flag | Long Option | Description | DevOps Practical Use Case |
| :--- | :--- | :--- | :--- |
| `-n` | `--numeric-sort` | Compares according to numerical string value. | Sorting port numbers, process PIDs, and response codes. |
| `-r` | `--reverse` | Reverses the sorting order (descending). | Finding top resource consumers (highest values first). |
| `-h` | `--human-numeric-sort` | Compares human-readable numbers (`2K`, `15M`, `4G`). | **Sorting `du -sh` or `ls -lh` disk capacity dumps.** |
| `-k KEY` | `--key=POS1[,POS2]` | Restricts sort comparison to specific columns. | Sorting tables by column 2, column 5, etc. |
| `-t CHAR` | `--field-separator=CHAR` | Specifies field delimiter character. | Sorting `/etc/passwd` by colon (`-t ':'`). |
| `-u` | `--unique` | Outputs only the first of an equal run; removes duplicates. | Deduplicating lists without invoking a separate `uniq` process. |
| `-V` | `--version-sort` | Natural sort of version numbers (`v1.9` before `v1.10`). | **Sorting semantic version tags in Git/Docker.** |
| `-o FILE` | `--output=FILE` | Writes result to FILE instead of standard output. | Safely overwriting the same input file in-place. |

---

## 1. Alphabetical vs Numerical Sorting (`-n`)

By default, `sort` compares characters alphabetically from left to right according to the ASCII table. This produces counterintuitive results with numbers:

```text
Default ASCII Sort:           Numerical Sort (sort -n):
1                             1
100                           2
2                             5
25                            25
5                             100
```
Because the character `'1'` comes before `'2'` in ASCII, `100` is sorted before `2`!

> [!WARNING]
> Always supply the **`-n`** flag whenever sorting numerical values (PIDs, CPU %, byte counts, response times).

---

## 2. Sorting by Key and Delimiter (`-k` and `-t`)

Tables often contain multiple columns. Use `-t` to define the delimiter and `-k` to specify which column to sort on:

### Syntax Anatomy: `-k POS1[,POS2][OPTS]`
- `-k 3` : Sort from column 3 to the end of the line.
- `-k 3,3` : Sort strictly by column 3 only.
- `-k 3,3n` : Sort strictly by column 3 numerically.
- `-k 3,3nr` : Sort strictly by column 3 numerically in descending (reverse) order.

### Practical Example: Sort Users by UID in `/etc/passwd`
The UID is the 3rd field in `/etc/passwd`:

```bash
$ sort -t':' -k3,3n /etc/passwd | head -n 5
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
```

### Multi-Column Compound Sorting
You can chain multiple keys. For example, sort by Group ID (column 4) ascending, and then by User ID (column 3) descending:

```bash
sort -t':' -k4,4n -k3,3nr /etc/passwd
```

---

## 3. High-Frequency DevOps Recipes

### A. Find the Top 10 Largest Folders (`sort -h -r`)
When investigating a disk full alert:
```bash
# du -h prints human readable sizes (K, M, G)
# sort -hr sorts human sizes descending
du -h --max-depth=1 /var | sort -hr | head -n 10
```
Output:
```text
45G     /var
32G     /var/lib
11G     /var/log
1.8G    /var/cache
140M    /var/spool
```

### B. Natural Version Sorting: `sort -V`
Standard alphabetical sort ranks `v1.10.0` *before* `v1.2.0`. Version sort (`-V`) respects semantic numbering:

```bash
$ cat versions.txt
v1.2.0
v1.10.0
v1.9.0
v2.0.0

$ sort -V versions.txt
v1.2.0
v1.9.0
v1.10.0
v2.0.0
```

### C. In-Place File Sorting: `-o`
In bash, writing `sort file.txt > file.txt` **truncates the file to 0 bytes before reading it**! Use `-o` to safely overwrite an input file:

```bash
# Safe in-place sort:
sort -u allowlist.txt -o allowlist.txt
```

---

## 4. Performance Tuning for Large Datasets (Gigabytes+)

When sorting multi-gigabyte files (e.g., massive database dumps or 50GB web logs):
1. **Buffer Size (`-S`)**: Prevents `sort` from spilling temporary chunks to slow disk storage:
   ```bash
   sort -S 4G -o sorted_big.log big.log
   ```
2. **Parallel Threads (`--parallel`)**: Utilizes multiple CPU cores:
   ```bash
   sort --parallel=8 -S 4G big.log -o sorted_big.log
   ```
3. **Custom Temp Directory (`-T`)**: If `/tmp` is a small in-memory tmpfs, point `sort` to a large NVMe disk mount:
   ```bash
   sort -T /data/tmp -S 4G big.log
   ```

---

## Summary

- Use `sort -n` for numerical comparisons to prevent ASCII ordering bugs.
- Use `sort -hr` to rank disk usage from `du -h` from largest to smallest.
- Isolate columns using `-t 'DELIM' -k POS,POS[FLAGS]`.
- Use `sort -V` for version strings (v1.9 before v1.10).
- Overwrite files in place safely using `sort file -o file` instead of redirection.
- On multi-gigabyte files, accelerate execution using `-S <RAM>`, `--parallel=<CORES>`, and `-T <DIR>`.
