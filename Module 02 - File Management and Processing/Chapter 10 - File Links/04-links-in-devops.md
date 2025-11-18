# Production Link Patterns in DevOps & Cloud Infrastructure

## Overview

In modern cloud computing, Site Reliability Engineering, and CI/CD pipelines, filesystem links are not merely convenience shortcuts. They are architectural building blocks that enable **zero-downtime releases**, **instant rollbacks**, **reproducible binary versioning**, and **zero-cost incremental storage snapshots**.

This lesson documents the battle-tested link patterns and security hardening configurations used across production Linux clusters.

---

## Pattern 1: Zero-Downtime Release Deployments

One of the most widely implemented deployment architectures in software engineering (used by tools like Capistrano, Deployer, and custom Ansible/Kubernetes pipelines) relies on an **atomic symlink cutover**.

### The Directory Architecture
```text
/var/www/production/
├── releases/
│   ├── 20261012_080000/    (Previous Release)
│   ├── 20261013_120000/    (Current Active Release)
│   └── 20261014_093000/    (Newly Deployed Staging Release)
├── shared/
│   ├── .env                (Persistent production database credentials)
│   ├── logs/               (Central application log directory)
│   └── uploads/            (User-uploaded media files)
└── current ──> releases/20261013_120000/  (The live symlink read by Nginx)
```

### The Atomic Cutover Script
Nginx points its `root` directive strictly to `/var/www/production/current/public`. When deploying a new version, the code is unpacked, tested, and atomically switched in a single millisecond:

```bash
#!/usr/bin/env bash
set -euo pipefail

DEPLOY_ROOT="/var/www/production"
RELEASE_TAG=$(date +%Y%m%d_%H%M%S)
NEW_RELEASE_DIR="$DEPLOY_ROOT/releases/$RELEASE_TAG"

echo "Deploying release: $RELEASE_TAG"

# Step 1: Clone and prepare new release in isolation
mkdir -p "$NEW_RELEASE_DIR"
git clone --depth 1 https://github.com/company/api.git "$NEW_RELEASE_DIR"
cd "$NEW_RELEASE_DIR"
npm ci --production

# Step 2: Symlink shared persistent assets (env and storage) into the release
ln -s "$DEPLOY_ROOT/shared/.env" "$NEW_RELEASE_DIR/.env"
ln -s "$DEPLOY_ROOT/shared/uploads" "$NEW_RELEASE_DIR/public/uploads"
ln -s "$DEPLOY_ROOT/shared/logs" "$NEW_RELEASE_DIR/storage/logs"

# Step 3: Run database migrations and smoke tests against NEW_RELEASE_DIR
npm run test:smoke

# Step 4: THE ATOMIC CUTOVER
# Switch the 'current' symlink to the new release without a single dropped packet!
ln -sfn "$NEW_RELEASE_DIR" "$DEPLOY_ROOT/current"

# Step 5: Reload service
systemctl reload api-service

echo "Release $RELEASE_TAG deployed with ZERO downtime!"
```

### Instant Rollback (Under 1 Second)
If an unhandled bug surfaces in production, rolling back to the previous stable release is an instantaneous one-line symlink switch:

```bash
# Rollback immediately:
ln -sfn /var/www/production/releases/20261013_120000 /var/www/production/current
systemctl reload api-service
```

---

## Pattern 2: Binary Version Management (`update-alternatives`)

Linux distributions use symlinks to manage multiple concurrent versions of runtimes (e.g., Python 3.10 vs 3.12, Java 17 vs Java 21) without breaking system utilities that rely on default paths:

```text
/usr/bin/java  ──>  /etc/alternatives/java  ──>  /usr/lib/jvm/java-21-openjdk/bin/java
```

### Switching Active Java Version:
```bash
sudo update-alternatives --config java
```

This updates the intermediate symlink in `/etc/alternatives/java` to point to the desired JDK, ensuring all scripts calling `/usr/bin/java` immediately use the new version.

---

## Pattern 3: Instant Storage Snapshots with Hard Links (`rsync --link-dest`)

When performing hourly or daily backups of multi-terabyte filesystems, storing full duplicates exhausts disk space rapidly.

The `--link-dest` flag in `rsync` leverages hard links to create **synthetic full backups**:

```bash
# Backup script:
PREVIOUS_BACKUP="/backups/daily.0"
NEW_BACKUP="/backups/daily.$(date +%F)"

rsync -a --delete \
    --link-dest="$PREVIOUS_BACKUP" \
    /var/www/ \
    "$NEW_BACKUP/"
```

### How it works:
1. `rsync` compares `/var/www/` against `PREVIOUS_BACKUP`.
2. For any file that **has not changed**, `rsync` creates a **hard link** in `NEW_BACKUP` pointing to the exact same inode as `PREVIOUS_BACKUP` (0 additional bytes of disk space!).
3. Only newly created or modified files consume new physical disk blocks.
4. Each `daily.X` folder looks and behaves like a complete standalone full backup!

---

## Pattern 4: Docker & Container Symlink Gotchas

When containerizing applications, symlinks introduce subtle production bugs if not handled carefully:

### The "Dangling Absolute Symlink" in Containers
Suppose a host directory has a symlink:
```bash
# On Host:
/app/certs -> /etc/ssl/certs
```

If you mount `/app` into a Docker container (`docker run -v /app:/app ...`), inside the container `/etc/ssl/certs` refers to the **container's internal certificates**, NOT the host's!

> [!TIP]
> **Container Best Practice**: Inside projects intended for containerization, **always use relative symbolic links** (`ln -s ../shared/certs ./certs`). Relative symlinks resolve identically inside and outside containers.

---

## Security Hardening: Protecting Against Symlink Exploits

Because symlinks redirect file operations, malicious local users have historically used them to escalate privileges or trick root daemons into overwriting `/etc/shadow` (Symlink Race Condition / TOCTOU).

### Kernel-Level Protection (`sysctl`)
Modern Linux kernels include native hardening flags enabled by default:

```bash
# Verify kernel symlink and hardlink protections:
$ sysctl fs.protected_symlinks fs.protected_hardlinks
fs.protected_symlinks = 1
fs.protected_hardlinks = 1
```

- **`fs.protected_symlinks = 1`**: Disallows following symlinks in world-writable, sticky-bit directories (like `/tmp`) unless the follower owns the symlink or the target.
- **`fs.protected_hardlinks = 1`**: Prevents users from creating hard links to files they do not own (preventing unauthorized pinning of sensitive inodes).

### Web Server Traversal Protection (Nginx & Apache)
In web servers, prevent attackers from symlinking `/etc/passwd` into your public docroot:

```nginx
# Nginx security setting:
# Disable following symlinks unless the symlink and target share the same owner:
disable_symlinks on from=$document_root;
```

---

## Summary

- Use the **Atomic Symlink Cutover Pattern** (`ln -sfn new_release current`) to achieve zero-downtime deployments and instant rollbacks.
- Use `rsync --link-dest` to create space-efficient incremental snapshots using hard links.
- In containerized architectures, always use **relative symlinks** rather than absolute symlinks to maintain volume portability.
- Ensure Linux kernel protections (`fs.protected_symlinks = 1` and `fs.protected_hardlinks = 1`) are active on all multi-tenant hosts.
- Configure web servers to enforce strict symlink ownership validation in public docroots.
