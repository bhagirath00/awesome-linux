# Read, Write, and Execute: Files vs Directories

## Overview

The three basic permission bits in Linux—**Read (`r`)**, **Write (`w`)**, and **Execute (`x`)**—have fundamentally different operational meanings depending on whether they are applied to a **regular file** or a **directory**.

Confusing these differences is the primary reason engineers encounter puzzling permission errors:
- Why an engineer cannot `cd` into a directory even though it has read (`r`) permission.
- Why a read-only (`0444`) file can still be completely deleted by a standard user.
- Why a script with execute (`x`) permission fails to run if it lacks read (`r`) permission.

---

## 1. Permission Matrix: Files vs Directories

| Permission Bit | Meaning on a REGULAR FILE | Meaning on a DIRECTORY |
| :---: | :--- | :--- |
| **`r` (Read)** | Permission to **read and view file content** (`cat`, `less`, `grep`, `head`). | Permission to **list file names** inside the directory (`ls`). |
| **`w` (Write)** | Permission to **modify or truncate file content** (`vim`, `echo >`, `sed -i`). *(Does NOT govern deletion!)* | Permission to **create, delete, and rename files** inside the directory! |
| **`x` (Execute)** | Permission to **execute the file as a program or script** (`./deploy.sh`, binary). | Permission to **enter (`cd`) and traverse** the directory, and access inodes of files within. |

---

## 2. In-Depth Analysis: Permissions on Directories

A directory in Linux is simply a special file that contains an index of `(filename -> inode)` mappings.

### What Execute (`x`) on a Directory Really Means
On a directory, **`x` is the Traverse (Search) permission**:
- Without `x`, you **cannot `cd`** into the directory.
- Without `x`, you **cannot access any file** inside the directory, even if you have full `rwx` permissions on the file itself!

### The Directory Permission Combinations

```bash
# Case 1: Read ONLY (r--) on directory /data/restricted
$ ls /data/restricted
file1.txt   file2.txt    <-- Can see names, BUT:
ls: cannot access '/data/restricted/file1.txt': Permission denied
$ cd /data/restricted
bash: cd: /data/restricted: Permission denied
# (Cannot inspect file metadata or open files because 'x' is missing!)

# Case 2: Execute ONLY (--x) on directory (The Blind Gateway)
$ cd /data/restricted    # Succeeded! Can enter.
$ ls                     # Fails: cannot list names without 'r'
ls: cannot open directory '.': Permission denied
$ cat /data/restricted/file1.txt # SUCCEEDS! Can read file if exact name is known!

# Case 3: Read + Execute (r-x) (The Standard Read-Only Directory)
$ ls /data/restricted    # Succeeded: lists files and attributes
$ cd /data/restricted    # Succeeded: can enter
$ touch newfile.txt      # FAILS: cannot create or delete files without 'w'
```

---

## 3. The Classic DevOps Puzzle: "Who Can Delete a File?"

Consider this scenario on a production server:

```bash
$ ls -ld /var/shared-project/
drwxrwxrwx 2 root root 4096 Oct 14 10:20 /var/shared-project/

$ ls -l /var/shared-project/production.conf
-r--r--r-- 1 root root 1200 Oct 14 10:20 /var/shared-project/production.conf
```
Notice that `production.conf` is owned by `root`, and has **read-only permissions (`0444`)**.

**Can a regular, unprivileged user `ubuntu` delete this file?**

### The Answer: **YES!**
A regular user can run `rm /var/shared-project/production.conf`, and the file will be deleted immediately:

```bash
$ rm /var/shared-project/production.conf
rm: remove write-protected regular file '/var/shared-project/production.conf'? y
# -> DELETED!
```

### Why?
**Deleting a file does NOT modify the file itself—it modifies the parent directory's directory table!**
Because `/var/shared-project/` has write (`w`) permission for Others, any user on the system has permission to unlink directory entries inside it, regardless of who owns the individual files!

> [!WARNING]
> To prevent users from deleting each other's files in shared writable directories, you must enable the **Sticky Bit (`chmod +t`)** on the parent directory (detailed in Chapter 14).

---

## 4. Script Execution: Why Bash Scripts Need Both `r` and `x`

When executing a compiled binary (ELF binary like `/bin/ls`), only **`x`** is required because the Linux kernel executes machine code directly into memory.

However, when running a shell script (`./deploy.sh`):
```bash
$ chmod 100 deploy.sh # Execute only (--x------)
$ ./deploy.sh
bash: ./deploy.sh: Permission denied
```

### Why does a script require Read (`r`) AND Execute (`x`)?
When you run `./deploy.sh`, the kernel reads the shebang line (`#!/bin/bash`) and spawns the `/bin/bash` interpreter. Bash must be able to **read the text lines of the script** into memory to interpret them!
- Compiled binaries need: `x`
- Interpreted scripts (Bash, Python, Perl) need: **`r` AND `x`** (`chmod 755` or `chmod +rx`).

---

## Summary

- On files: `r` views content, `w` modifies content, and `x` executes binaries.
- On directories: `r` lists filenames (`ls`), `w` creates/deletes/renames files, and `x` traverses (`cd`) and accesses inodes.
- File deletion is governed by the **parent directory's write (`w`) permission**, not the file's own write permission!
- Shell scripts require both `r` and `x` permissions so the interpreter can read the code.
