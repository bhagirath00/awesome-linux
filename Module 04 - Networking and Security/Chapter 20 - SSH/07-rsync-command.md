# Chapter 20: SSH - Rsync Over SSH (Remote Synchronization)

## 1. Introduction: What is Rsync?

`rsync` (**Remote Sync**) is the standard utility for synchronizing files and directories between two locations (either locally or across a network) with minimal data transfer. 

Its standout feature is the **delta-transfer algorithm**: instead of copying whole files, `rsync` calculates checksums across fixed blocks of data and transmits **only the differences (deltas)** between the source and destination files. When transferring over a network, `rsync` operates through an encrypted SSH tunnel by default.

```
Source File:      [ Block A ] [ Block B (Modified) ] [ Block C ]
                                      |
                         (Delta Algorithm detects difference)
                                      v
Network Transfer:                 [ Block B' ]  (Only changed bytes sent!)
                                      |
Destination File: [ Block A ] [ Block B (Updated)  ] [ Block C ]
```

---

## 2. Command Syntax and Trailing Slash Semantics

**Basic Syntax:**
```bash
rsync [options] SOURCE DESTINATION
```

### The Critical Trailing Slash Rule
In `rsync`, a trailing slash on the source directory fundamentally changes what is copied:

- **Without trailing slash (`src/dir`)**: Copies the folder `dir` itself inside the destination folder.
- **With trailing slash (`src/dir/`)**: Copies the *contents* of `dir` directly into the destination folder without creating an extra nested `dir`.

```bash
# Copies the directory 'html' INTO /var/www/ -> Result: /var/www/html/
rsync -avz html /var/www/

# Copies the CONTENTS of 'html' directly INTO /var/www/ -> Result: /var/www/index.html
rsync -avz html/ /var/www/
```

---

## 3. The Gold Standard Flags: `-avzP`

In production DevOps workflows, the standard combination is almost universally:
```bash
rsync -avzP source/ user@remote:/destination/
```

| Flag | Meaning | Detailed Description |
| :--- | :--- | :--- |
| `-a` | **Archive mode** | Equivalent to `-rlptgoD` (Recurses directories, preserves symlinks, permissions, modification times, owner, group, and special/device files). |
| `-v` | **Verbose** | Outputs list of files being transferred. |
| `-z` | **Compress** | Compresses data blocks during transfer to maximize network bandwidth. |
| `-P` | **Progress & Partial** | Combines `--progress` (real-time transfer meter) and `--partial` (keeps partially transferred files so interrupted transfers can be resumed). |
| `-h` | **Human-readable** | Formats numbers in KB, MB, and GB. |

---

## 4. Advanced Production Use Cases

### 1. Resuming an Interrupted Multi-Gigabyte Transfer
If a connection drops during a large backup or image transfer, re-running the command with `-P` continues exactly where it left off:
```bash
rsync -avzP --append-verify large-db-backup.tar.gz backup-server:/storage/backups/
```

### 2. Mirroring / Deleting Extraneous Destination Files (`--delete`)
For production website deployment or continuous mirror syncing, use `--delete` to remove files from the destination that no longer exist in the source:

```bash
# WARNING: Test with --dry-run first!
rsync -avzP --delete ./build/ deploy@web01:/var/www/app/
```

> [!WARNING]
> Always run with `--dry-run` (`-n`) first when using `--delete` to preview which files will be deleted before executing destructive operations!
> ```bash
> rsync -avzP --delete --dry-run ./build/ deploy@web01:/var/www/app/
> ```

### 3. Excluding Files and Directories (`--exclude`)
Exclude temporary files, cache, or `.git` repositories during deployments:

```bash
rsync -avzP \
  --exclude '.git' \
  --exclude 'node_modules' \
  --exclude '*.log' \
  --exclude '.env' \
  ./src/ devops@prod-server:/opt/webapp/
```

### 4. Specifying Custom SSH Port or Key (`-e ssh`)
```bash
rsync -avzP -e "ssh -p 22022 -i ~/.ssh/prod_key" ./data/ admin@remote:/data/
```

---

## 5. Summary Takeaways

- `rsync` is vastly superior to `scp` for backups and deployments due to its delta-transfer algorithm.
- Always remember the trailing slash rule (`source/` transfers contents; `source` transfers the folder itself).
- `-avzP` is the standard flag combination for high-fidelity, resumable syncs.
- Always test destructive `--delete` operations with `--dry-run` first.
