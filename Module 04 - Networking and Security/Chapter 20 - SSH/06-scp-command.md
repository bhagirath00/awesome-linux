# Chapter 20: SSH - Secure Copy Protocol (SCP)

## 1. Introduction: What is SCP?

`scp` (Secure Copy Protocol) is a command-line tool used to copy files and directories securely between hosts on a network. It utilizes SSH for data transfer and authentication, ensuring that file payloads are encrypted in transit.

> [!NOTE]
> In modern OpenSSH releases (version 8.0 and above), the `scp` binary uses the **SFTP protocol** under the hood by default rather than the legacy BSD `rcp`-based protocol, eliminating historic security vulnerabilities related to wildcards and unsanitized filenames.

---

## 2. Command Syntax and Operation

**Basic Syntax:**
```bash
scp [options] [source_user@source_host:]source_path [dest_user@dest_host:]dest_path
```

`scp` operates in three main directions:
1. **Local to Remote** (Uploading)
2. **Remote to Local** (Downloading)
3. **Remote to Remote** (Transferring between two servers directly from your workstation)

---

## 3. Essential Command Examples

### 1. Uploading a File to a Remote Server
```bash
scp application.tar.gz devops@192.168.1.100:/opt/deployments/
```

### 2. Downloading a File from a Remote Server
```bash
scp devops@192.168.1.100:/var/log/nginx/error.log ./local_logs/
```

### 3. Copying an Entire Directory Recursively (`-r`)
```bash
scp -r ./dist/ devops@192.168.1.100:/var/www/html/
```

### 4. Specifying Custom Port, Identity Key, and Preserving Timestamps
```bash
# -P: Specify custom SSH port (must be uppercase P)
# -i: Provide path to private key
# -p: Preserve file modification times, access times, and modes
# -C: Enable compression during transfer
scp -P 2222 -i ~/.ssh/id_ed25519 -p -C release.zip ubuntu@prod.company.com:/srv/
```

---

## 4. Key Flags Reference Table

| Flag | Meaning | Use Case |
| :--- | :--- | :--- |
| `-r` | Recursive | Copy entire directory hierarchies. |
| `-P <port>` | Port | Connect to non-standard SSH port (**Note:** uppercase `-P`). |
| `-i <identity_file>` | Identity Key | Specify private key file for authentication. |
| `-p` | Preserve Attributes | Preserves modification times, access times, and file modes. |
| `-q` | Quiet Mode | Suppress progress meter and non-critical messages (ideal for scripts). |
| `-C` | Compression | Compresses data stream for slower network links. |
| `-v` | Verbose | Prints detailed debugging messages for troubleshooting connection issues. |

---

## 5. SCP vs. Rsync: When to Use Which?

While `scp` is universal and available on virtually all systems by default, it does not support delta-transfer (syncing only changed blocks) or resuming interrupted transfers:

```
Simple, one-off file copy  -----> Use scp (universal, built-in)
Large trees / Resuming / Delta -> Use rsync (faster, incremental)
```

---

## 6. Summary Takeaways

- `scp` transfers files encrypted over SSH.
- Use uppercase `-P` to designate custom SSH ports (lowercase `-p` preserves file timestamps).
- Modern `scp` uses the SFTP protocol internally for increased security.
- For recurring backups or large directories, `rsync` is preferred over `scp` due to delta synchronization.
