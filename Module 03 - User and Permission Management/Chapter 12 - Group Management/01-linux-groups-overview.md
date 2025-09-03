# Linux Groups Overview: Architecture, GIDs, and Primary vs Supplementary Groups

## Overview

In the Linux operating system, managing permissions for individual users on a file-by-file basis becomes impossible to scale as organizations and server fleets grow.

**Linux Groups** solve this by organizing multiple user accounts into unified administrative collections. By assigning permissions to a group rather than to individual accounts, any user assigned to that group automatically inherits the required access.

In modern DevOps, cloud engineering, and infrastructure administration, groups provide:
- **Role-Based Access Control (RBAC)**: Granting administrative capabilities (`sudo`, `wheel`), log reading rights (`adm`), or container orchestration privileges (`docker`).
- **Team Collaboration Spaces**: Allowing members of an engineering team (`devs`, `ops`) to read and write to shared application code directories (`/var/www/html`) without exposing files to the world.
- **Service Isolation**: Ensuring system daemons (`www-data`, `nginx`, `postgres`) can read specific configuration files while isolating them from untrusted users.

---

## 1. Group Identifiers (GID) and Allocation Ranges

Just as users are represented numerically by UIDs, the Linux kernel identifies groups internally using numerical integers called **Group Identifiers (GIDs)**.

```text
+-------------------------------------------------------------------------+
|                         LINUX GID RANGES                                |
+-------------------------------------------------------------------------+
|  GID 0             root group                                           |
|  GID 1 – 999       System Groups (daemon, adm, sudo, docker, www-data)  |
|  GID 1000+         Regular User & Collaboration Groups (devs, qa, ops)  |
+-------------------------------------------------------------------------+
```

---

## 2. Primary Group vs Supplementary Groups

Every Linux user has exactly **one Primary Group**, but can belong to **multiple Supplementary (Secondary) Groups**:

```text
User: "alex"
  ├── Primary Group:        "alex" (GID 1001)   <-- Automatically owns any newly created file!
  └── Supplementary Groups: ├── "sudo"   (GID 27)  <-- Root administrative rights
                            ├── "docker" (GID 998) <-- Socket access to Docker daemon
                            └── "devs"   (GID 1005)<-- Shared project folder read/write
```

### A. The Primary Group
- Stored directly in the **4th field of `/etc/passwd`**.
- **Crucial Behavior**: Whenever a user creates a new file or directory, that file's group ownership is automatically set to the user's **Primary Group** by default!

#### The User Private Group (UPG) Scheme
On older Unix systems, all users were often placed into a single shared primary group called `users`. If `umask` was loosely configured, any user could read another user's files!
Modern Linux distributions (Ubuntu, Debian, RHEL, Fedora) implement the **User Private Group (UPG)** standard:
- Whenever a new user `alex` is created, a dedicated group named `alex` with an identical GID is automatically created as their primary group.
- This isolates the user's files by default, making collaborative group sharing safe and explicit.

### B. Supplementary (Secondary) Groups
- Stored in the **`/etc/group`** file.
- Used to confer additional system capabilities, access to hardware devices (e.g., `plugdev`, `dialout`), or shared folder permissions.

---

## 3. Standard Linux System Groups in DevOps

Modern Linux distributions come pre-configured with standard administrative groups:

| Group Name | Typical GID | Security Purpose & Privileges |
| :--- | :--- | :--- |
| **`sudo`** *(Debian/Ubuntu)* | 27 | Members can run any command as root via `sudo`. |
| **`wheel`** *(RHEL/CentOS/Arch)*| 10 | The Red Hat / BSD equivalent of the `sudo` group. |
| **`adm`** | 4 | Allows members to read system logs in `/var/log` without root privileges. |
| **`docker`** | 998 / 999 | Allows read/write access to `/var/run/docker.sock` (runs Docker without `sudo`). |
| **`www-data`** | 33 | Shared web server runtime group (Nginx, Apache, PHP-FPM). |
| **`systemd-journal`** | 101 | Allows reading systemd journal logs via `journalctl` without root. |

---

## 4. Inspecting Groups: `id` and `groups`

```bash
# View groups for current user:
$ groups
ubuntu adm sudo docker

# View full numeric UID, primary GID, and supplementary GIDs:
$ id
uid=1000(ubuntu) gid=1000(ubuntu) groups=1000(ubuntu),4(adm),27(sudo),998(docker)

# View groups for another specific user:
$ id jenkins
uid=1002(jenkins) gid=1002(jenkins) groups=1002(jenkins),998(docker)
```

---

## 5. The Session Update Gotcha: Activating Groups with `newgrp`

A frequent source of confusion occurs when an administrator adds a user to a new group:
```bash
sudo usermod -aG docker ubuntu
```
The user types `docker ps`, and gets hit with:
`Got permission denied while trying to connect to the Docker daemon socket.`

### Why?
Group memberships are read into the user's login session token **only during the initial login process**. Adding a user to a group does **not** update their currently active shell session!

### Solutions:
1. **Option A (Cleanest)**: Log out of the SSH session or terminal and log back in.
2. **Option B (Instant in current shell)**: Run the **`newgrp`** command to start a new subshell with the updated group active:
   ```bash
   newgrp docker
   docker ps  # Works immediately without logging out!
   ```

---

## Summary

- Groups organize users into collective permission boundaries.
- The Primary Group determines the default group ownership of all newly created files.
- Modern Linux enforces the User Private Group (UPG) scheme for security isolation.
- Supplementary groups confer administrative rights (`sudo`), logging access (`adm`), or container permissions (`docker`).
- Newly assigned group memberships require logging out and back in, or invoking `newgrp <group>` to take effect in the active shell.
