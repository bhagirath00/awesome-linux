# Creating Directories in Linux (`mkdir`)

## Overview

In Linux, directories are special files containing lists of filenames and their associated filesystem index numbers (inodes). The primary command for directory creation is `mkdir` (make directory).

Creating directories reliably is a fundamental building block in DevOps automation: initializing release directories for zero-downtime deployments, setting up persistent volume mount points for Docker containers, creating isolated sandbox environments for CI/CD runners, and scaffolding microservice source trees.

---

## Command Syntax and Anatomy

```bash
mkdir [OPTIONS] DIRECTORY...
```

By default, `mkdir` creates one or more directories with permissions determined by the current user's `umask` (usually `0777 - 0022 = 0755` -> `rwxr-xr-x`).

If any intermediate parent directory in the path does not exist, `mkdir` fails with an error:
```bash
$ mkdir /opt/myapp/releases/v1.0
mkdir: cannot create directory ‘/opt/myapp/releases/v1.0’: No such file or directory
```

---

## Essential `mkdir` Flags

| Flag | Long Option | Description | DevOps Importance |
| :--- | :--- | :--- | :--- |
| `-p` | `--parents` | Creates parent directories as needed without throwing an error if the directory already exists. | **Crucial for idempotent automation**. Scripts can run repeatedly without crashing. |
| `-m` | `--mode=MODE` | Sets file mode (octal permissions) directly upon directory creation. | Enforces strict security (e.g., `0700` for private keys or SSL certs) atomically. |
| `-v` | `--verbose` | Prints a message for each directory created. | Providing clear execution audit logs in CI/CD pipeline steps. |

---

## 1. Deep Dive: The Idempotent `-p` Flag

In infrastructure automation (Bash scripts, Ansible, Puppet, Docker entrypoints), tasks must be **idempotent**—running a script a second time should produce the exact same outcome without throwing errors.

Without `-p`:
```bash
$ mkdir app
$ mkdir app
mkdir: cannot create directory ‘app’: File exists
# Return code is 1 -> Fails automated CI/CD pipeline
```

With `-p`:
```bash
$ mkdir -p app
$ mkdir -p app
# Succeeded silently! Return code is 0 -> CI/CD pipeline proceeds smoothly
```

Furthermore, `-p` automatically creates all missing parent directories along the path:
```bash
mkdir -p /var/www/myproject/{frontend,backend}/{src,tests,dist}
```

This single command generates the complete directory tree:
```text
/var/www/myproject/
├── backend/
│   ├── dist/
│   ├── src/
│   └── tests/
└── frontend/
    ├── dist/
    ├── src/
    └── tests/
```

---

## 2. Setting Explicit Permissions on Creation (`-m`)

Instead of creating a folder with default permissions and then running `chmod`, use `-m` (`--mode`) to set permissions atomically at creation time:

```bash
# Create a secure directory accessible only by root or the current user (0700: rwx------)
mkdir -m 700 /root/.ssh

# Create an open upload scratch directory with sticky-bit (1777: rwxrwxrwt)
mkdir -m 1777 /var/shared-uploads

# Verify permissions
ls -ld /root/.ssh /var/shared-uploads
# drwx------ 2 root root 4096 Oct 14 09:35 /root/.ssh
# drwxrwxrwt 2 root root 4096 Oct 14 09:35 /var/shared-uploads
```

> [!WARNING]
> When combining `-p` and `-m`, the mode `-m` applies **only to the deepest (final) directory** created in the path. The intermediate parent directories are created with the default `umask` permissions.

---

## 3. Scaffolding Directory Trees with Brace Expansion

Bash brace expansion allows constructing complex, enterprise-grade deployment folder structures in a single concise line:

### Real-World DevOps Pattern: Zero-Downtime Deployment Hierarchy
Many deployment tools (Capistrano, Deployer, custom Ansible playbooks) maintain standard release directories:

```bash
mkdir -p /var/www/production/{releases,shared/{storage,config,logs},repo}
```

Visualizing the created structure:
```text
/var/www/production/
├── releases/             (Stores timestamped deployment folders)
├── repo/                 (Cached bare Git mirror)
└── shared/               (Assets persisted across all deployments)
    ├── config/           (Persistent .env, secrets)
    ├── logs/             (Application log output)
    └── storage/          (User uploads, media)
```

---

## 4. Secure Ephemeral Directories: `mktemp -d`

In automation scripts (such as GitHub Actions runners or container build hooks), hardcoding temporary directories like `/tmp/mybuild` introduces **race conditions** and **security vulnerabilities** (symlink attacks):

Instead of `mkdir /tmp/build`, always use `mktemp -d`:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Safely creates an unguessable temporary directory like /tmp/tmp.s3X8kP2q91
WORK_DIR=$(mktemp -d)
echo "Created temporary workspace at: $WORK_DIR"

# Ensure the directory is automatically cleaned up when the script exits
cleanup() {
    rm -rf "$WORK_DIR"
    echo "Cleaned up temporary workspace."
}
trap cleanup EXIT

# Perform automated tasks inside WORK_DIR safely
cd "$WORK_DIR"
git clone --depth 1 https://github.com/example/repo.git .
./build.sh
```

---

## Common Errors and Troubleshooting

### 1. `Permission denied`
```bash
$ mkdir /var/log/mycustomapp
mkdir: cannot create directory ‘/var/log/mycustomapp’: Permission denied
```
**Cause**: The current user lacks write permission on `/var/log`.
**Solution**: Use `sudo`:
```bash
sudo mkdir -p /var/log/mycustomapp
sudo chown -R appuser:appgroup /var/log/mycustomapp
```

### 2. `No space left on device` (with disk space remaining)
If `df -h` shows disk space available, but `mkdir` still fails with `No space left on device`, check the inode count:
```bash
df -i
```
If `IFree` is 0, the filesystem has run out of inodes. Clean up millions of small temporary files or session tokens to restore directory creation capability.

---

## Summary

- Use `mkdir <dir>` to create directories; always use `mkdir -p` in shell scripts for idempotency and automatic parent creation.
- Set strict directory permissions atomically during creation with `mkdir -m <mode>`.
- Leverage Bash brace expansion (`{}`) to scaffold multi-tier application architectures rapidly.
- Never use predictable hardcoded `/tmp` paths in automation; use `mktemp -d` paired with `trap cleanup EXIT`.
