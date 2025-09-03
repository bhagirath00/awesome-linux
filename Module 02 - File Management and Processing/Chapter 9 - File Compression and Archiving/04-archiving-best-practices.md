# Archiving and Backup Best Practices in Production Linux

## Overview

In enterprise operations and Site Reliability Engineering, archiving and compression are not merely administrative conveniences—they are the core mechanisms for **Disaster Recovery (DR)**, compliance audits, and data retention.

A corrupted tarball or an untested backup script can result in permanent data loss. Adhering to production-grade archiving standards guarantees data integrity, prevents security vulnerabilities during extraction, and ensures rapid recovery during production outages.

---

## 1. The Industry Golden Standard: The 3-2-1 Backup Strategy

Every production backup architecture should follow the **3-2-1 Rule**:

```text
+--------------------------------------------------------------------------+
|                         THE 3-2-1 BACKUP RULE                            |
+--------------------------------------------------------------------------+
|  [ 3 Copies of Data ]     Active Production + 2 Independent Backups      |
|                                                                          |
|  [ 2 Different Media ]    Local NVMe Storage + Cloud Object Storage (S3) |
|                                                                          |
|  [ 1 Offsite Copy ]       Different AWS Region or Off-Premises Datacenter |
+--------------------------------------------------------------------------+
```

---

## 2. Preventing the Security Traps

### A. The "Absolute Path" Danger: Why Tar Strips Leading Slashes
When you archive a folder using an absolute path:
```bash
$ tar -czvf backup.tar.gz /etc/nginx
tar: Removing leading `/' from member names
```

### Why does `tar` remove the leading `/`?
This is a critical security protection. If a tarball contained absolute paths (`/etc/shadow` or `/bin/bash`), unpacking that archive on any server as root would **immediately overwrite the host's actual `/etc/shadow` or system binaries**!

By removing the leading slash, the archive stores relative paths (`etc/nginx/`). When extracted, files unpack relative to the current working directory, preventing accidental system overwrites.

> [!WARNING]
> **Never use `-P` (`--absolute-names`)** unless you have an extraordinary, isolated requirement. Always archive using relative paths:
> ```bash
> # Best practice: Change into parent directory before archiving:
> tar -czvf nginx-backup.tar.gz -C /etc nginx
> ```

---

### B. Defeating the "Tar Bomb"
A **tar bomb** occurs when an archive is packaged without an enclosing top-level directory. Extracting it pollutes the destination directory with hundreds of loose files:

```bash
# BAD: Tarball contains 500 loose files at root
tar -xzvf messy-archive.tar.gz
# -> 500 files scattered all over your home directory!
```

### Protection Pattern: Always Extract into an Explicit Target Directory (`-C` or `tar --one-top-level`)
```bash
# Method 1: Target directory with -C
mkdir -p /tmp/extracted && tar -xzvf messy-archive.tar.gz -C /tmp/extracted

# Method 2: GNU Tar built-in flag (creates an enclosing directory automatically)
tar --one-top-level=archive-contents -xzvf messy-archive.tar.gz
```

---

## 3. Cryptographic Checksumming & Verification

A backup file whose integrity has not been mathematically verified is a potential disaster. Transmission errors, bit rot, or network timeouts can corrupt archives silently.

### Step 1: Generate SHA-256 Checksum at Creation Time
```bash
# Create archive
tar -czvf /backups/app-2026-10-14.tar.gz /var/www/app

# Generate checksum manifest
sha256sum /backups/app-2026-10-14.tar.gz > /backups/app-2026-10-14.tar.gz.sha256
```

### Step 2: Verify Checksum Before Restoring
Before running an extraction or database restore:
```bash
$ sha256sum -c /backups/app-2026-10-14.tar.gz.sha256
/backups/app-2026-10-14.tar.gz: OK
```
If the output reports `OK`, the archive is 100% bit-for-bit identical to the day it was created.

---

## 4. Encrypting Archives for Regulatory Compliance (HIPAA, SOC 2, GDPR)

Storing unencrypted database archives or customer files on cloud storage buckets violates compliance standards.

### Encrypting with OpenSSL (AES-256-CBC)
Stream `tar` directly through `openssl` to generate an encrypted payload:

```bash
# Create and encrypt archive in a single pass:
tar -czf - /var/data | openssl enc -aes-256-cbc -salt -pbkdf2 -out /backups/data.tar.gz.enc

# Decrypt and unpack:
openssl enc -d -aes-256-cbc -pbkdf2 -in /backups/data.tar.gz.enc | tar -xzf - -C /restored/
```

### Encrypting with GnuPG (GPG)
```bash
# Asymmetric encryption using team's public GPG key:
tar -czf - /var/data | gpg --encrypt --recipient ops@company.com -o backup.tar.gz.gpg
```

---

## 5. Automated Backup Rotation Lifecycle

Production backup directories must not grow unbounded until the disk fills up. Implement an automated retention lifecycle:

```bash
#!/usr/bin/env bash
set -euo pipefail

BACKUP_DIR="/mnt/backups/database"
RETENTION_DAYS=14

# Step 1: Create fresh daily compressed backup
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
tar -czf "$BACKUP_DIR/db_$TIMESTAMP.tar.gz" /var/lib/postgresql/data

# Step 2: Purge backup archives older than 14 days
find "$BACKUP_DIR" -type f -name "db_*.tar.gz" -mtime +"$RETENTION_DAYS" -delete
```

---

## 6. The Cardinal Rule of SRE: Test Your Restores!

> **"Nobody wants backups. Everybody wants restores."**

A backup script is only a hypothesis until you have proven you can successfully restore it:
1. Schedule an automated monthly or quarterly restore test in a staging/sandbox environment.
2. Automate a script that pulls a random recent backup from S3, verifies the SHA-256 checksum, extracts it, boots the application, and runs integration health checks.

---

## Summary

- Enforce the 3-2-1 backup strategy across all mission-critical infrastructure.
- Never use `-P` (absolute names); always package and extract archives using relative paths.
- Prevent "tar bombs" using `tar --one-top-level` or `mkdir -p dir && tar -xf archive -C dir`.
- Always generate and verify SHA-256 checksums (`sha256sum -c archive.sha256`) to catch bit rot.
- Encrypt offsite backups using OpenSSL (`aes-256-cbc`) or GPG.
- Automatically prune old archives with `find -mtime +N -delete`.
- Regularly test recovery procedures in staging environments.
