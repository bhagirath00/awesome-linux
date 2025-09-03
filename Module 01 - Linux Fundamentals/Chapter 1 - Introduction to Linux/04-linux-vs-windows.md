# Linux vs Windows: A DevOps Comparison

## Introduction

Both Linux and Windows are powerful operating systems, but they were built for fundamentally different philosophies. Windows was designed originally for personal desktop computers with a graphical user interface (GUI). Linux was created in the Unix tradition: **headless, text-driven, multi-user, and built for networked servers**.

Understanding these differences prevents common deployment bugs, script failures, and architecture mistakes.

---

## 1. Architectural & Core Differences

```text
+------------------------------------------+------------------------------------------+
|                  LINUX                   |                 WINDOWS                  |
+------------------------------------------+------------------------------------------+
| Single unified root tree:                | Multiple independent drive letters:      |
| /                                        | C:\, D:\, E:\                            |
| ├── bin                                  |                                          |
| ├── etc                                  | Paths use backslash:                     |
| └── var                                  | C:\Users\patel\Github                    |
| Paths use forward slash:                 |                                          |
| /var/log/nginx/access.log                |                                          |
+------------------------------------------+------------------------------------------+
| STRICTLY Case-Sensitive:                 | Case-Insensitive (by default):           |
| file.txt, File.txt, FILE.TXT             | file.txt, File.txt, and FILE.TXT         |
| are 3 completely different files!        | all refer to the EXACT same file.        |
+------------------------------------------+------------------------------------------+
| UNIX Line Endings:                       | DOS/Windows Line Endings:                |
| LF (\n - Line Feed)                      | CRLF (\r\n - Carriage Return + Line Feed)|
| Byte: 0x0A                               | Bytes: 0x0D 0x0A                         |
+------------------------------------------+------------------------------------------+
| Headless by default:                     | GUI by default:                          |
| Operates 100% via SSH & CLI text stream. | Windows Server defaults to Desktop GUI   |
| RAM idle footprint: ~150MB - 300MB.      | (Server Core exists, but heavier).       |
|                                          | RAM idle footprint: ~2GB - 4GB.          |
+------------------------------------------+------------------------------------------+
```

---

## 2. The Line Ending Gotcha: LF vs CRLF

One of the most frequent errors encountered by developers moving between Windows and Linux servers is line endings.

- When you write a Bash script on Windows: `test.sh`
- Windows editors default to saving lines with **CRLF** (`\r\n`).
- When executed inside a Linux container or server, the Linux shell reads the trailing `\r` as part of the command:

```bash
# What you see in the file:
#!/bin/bash
echo "Hello World"

# What Linux executes:
#!/bin/bash\r: No such file or directory
# error: command not found: echo\r
```

### How to Fix Line Endings in Linux:

```bash
# 1. Using dos2unix utility
sudo apt install -y dos2unix
dos2unix my-script.sh

# 2. Using sed to strip carriage returns
sed -i 's/\r$//' my-script.sh

# 3. Configure Git to automatically handle line endings across your team
git config --global core.autocrlf input  # (On Linux / macOS)
git config --global core.autocrlf true   # (On Windows)
```

---

## 3. Filesystem Hierarchy vs Drive Letters

In Windows, every storage device or partition is assigned a drive letter:
- `C:\` = Primary OS drive
- `D:\` = Secondary drive
- `E:\` = USB drive or external volume

In Linux, there are **no drive letters**. Everything begins at the root directory (`/`). External drives, network shares, or AWS EBS storage volumes are **mounted** into specific directories within this single tree:

```bash
# In Linux, mounting a new 500GB SSD to /data
sudo mount /dev/nvme1n1 /data

# Now all files saved in /data reside physically on that 500GB SSD,
# seamlessly integrated into the root filesystem.
```

---

## 4. Resource Utilization & Licensing

| Factor | Linux Server | Windows Server |
|---|---|---|
| **Base Memory Footprint** | ~150 MB – 500 MB RAM | ~2 GB – 4 GB RAM |
| **Storage Footprint** | ~1.5 GB – 3 GB disk space | ~15 GB – 32 GB disk space |
| **Licensing Cost** | **$0** (Free & Open Source) | Core-based commercial license ($1,000+ per server) |
| **Reboot Requirements** | Kernel updates can use kexec; package installs rarely require reboots | Frequent mandatory system reboots for updates |
| **Containers** | Native kernel cgroups & namespaces | Containerized Windows images are 5GB–15GB+ |

In cloud infrastructure where compute cost scales with RAM and vCPU usage, running thousands of microservices on lightweight Linux instances saves companies millions of dollars compared to Windows Server licenses.

---

## 5. Bridging the Gap: WSL2 (Windows Subsystem for Linux)

If your developer laptop runs Windows, you do not need to install dual-boot or heavy virtual machines to work with Linux. Modern Windows ships with **WSL2**:

```text
+--------------------------------------------------------------+
|                         WINDOWS 11 / 10                      |
|                                                              |
|   +--------------------------+    +----------------------+   |
|   |    Windows Host OS       |    |  Lightweight Hyper-V |   |
|   |  (VS Code, Chrome, UI)   |    |  Micro-VM (<1 sec)   |   |
|   +--------------------------+    +----------------------+   |
|                ^                             |               |
|                |   9P Protocol & Plan9 FS    v               |
|                +-----------------------> [Real Linux Kernel] |
|                                          [Ubuntu / Debian]   |
|                                          [Native Docker]     |
+--------------------------------------------------------------+
```

### Useful WSL2 Commands:
```powershell
# Install Ubuntu on WSL2
wsl --install -d Ubuntu-24.04

# List installed Linux distributions
wsl --list --verbose

# Access Windows files from inside Linux:
# /mnt/c/Users/patel/Github/...

# Access Linux files from Windows Explorer:
# \\wsl$\Ubuntu-24.04\home\patel\...
```

---

## Summary Comparison Matrix

| Feature | Linux | Windows |
|---|---|---|
| **Default Shell** | Bash / Zsh | PowerShell / CMD |
| **Remote Access** | SSH (native port 22) | RDP (GUI) / WinRM |
| **Configuration** | Plain text files in `/etc` | Windows Registry / Group Policy |
| **File Permissions** | Read, Write, Execute (`rwxrwxrwx`) | Access Control Lists (ACLs) |
| **Package Management** | Native (`apt`, `dnf`, `apk`, `pacman`) | `winget`, `choco` (third-party/add-on) |
| **Cloud Market Share** | > 90% of public cloud VMs | < 10% (mostly legacy .NET/Active Directory) |
