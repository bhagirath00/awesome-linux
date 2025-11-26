# Numeric (Octal) Permissions in Linux: Binary Math & Best Practices

## Overview

In Linux systems administration, permissions are represented mathematically using **Base-8 (Octal) notation**. Every three-character permission triad (`rwx`) corresponds to a 3-bit binary number whose sum yields a single octal digit between `0` and `7`.

Numeric permissions are the universal notation used in Ansible playbooks, Dockerfiles, Terraform configurations, Kubernetes Pod Security Standards, and cloud deployment scripts.

---

## 1. The Binary to Octal Mathematics

Every permission bit has an assigned numerical weight based on powers of 2:

```text
Permission Bit    Binary Representation    Octal Value
      r                   100                  4
      w                   010                  2
      x                   001                  1
      -                   000                  0
```

To determine the octal digit for any permission tier (Owner, Group, or Others), simply **add the active values**:

$$\text{Value} = r(4) + w(2) + x(1)$$

### The Complete 0–7 Permutation Table

| Octal Digit | Binary Bits | Symbolic String | Access Granted | Common Context |
| :---: | :---: | :---: | :--- | :--- |
| **`0`** | `000` | `---` | No access whatsoever. | Complete lockdown. |
| **`1`** | `001` | `--x` | Execute only. | Blind directories (rare). |
| **`2`** | `010` | `-w-` | Write only. | Drop-box logging (rare). |
| **`3`** | `011` | `-wx` | Write and Execute. | Drop-box directories. |
| **`4`** | `100` | `r--` | **Read only**. | Read-only configs, AWS `.pem` keys. |
| **`5`** | `101` | `r-x` | **Read and Execute**. | Read-only directories, shared binaries. |
| **`6`** | `110` | `rw-` | **Read and Write**. | Standard files, documents, databases. |
| **`7`** | `111` | `rwx` | **Read, Write, and Execute**. | Standard directories, owner scripts. |

---

## 2. Constructing the Three-Digit Octal Code

A standard Linux numeric permission string consists of three digits:

```text
chmod  6       4       4    file.txt
       │       │       │
       │       │       └─ Tier 3: OTHERS (r--)  = 4 + 0 + 0 = 4
       │       └───────── Tier 2: GROUP  (r--)  = 4 + 0 + 0 = 4
       └───────────────── Tier 1: OWNER  (rw-)  = 4 + 2 + 0 = 6
```

### Result:
```bash
$ ls -l file.txt
-rw-r--r-- 1 ubuntu ubuntu 0 Oct 14 10:25 file.txt
```

---

## 3. The 4th Leading Digit: Special Permissions

While permissions are usually written with 3 digits, Linux actually uses a **4-digit octal value**:

```text
chmod 0 7 5 5 script.sh
      │ │ │ │
      │ └─┴─┴─ Standard Owner, Group, Others (755)
      └─────── Special Permissions Digit (SUID, SGID, Sticky)
```

The special digit is calculated using the same binary addition:
- **`4`** : **SUID** (Set User ID)
- **`2`** : **SGID** (Set Group ID)
- **`1`** : **Sticky Bit**

For example:
- `1777` = Sticky bit (`1`) + full access (`777`) on `/tmp`.
- `2775` = SGID (`2`) + read/write/execute for owner/group (`775`) on shared collaboration directories.

---

## 4. The DevOps Standard Permission Cheatsheet

| Numeric | Symbolic | Allowed For | Standard Production Use Case |
| :---: | :---: | :--- | :--- |
| **`600`** | `-rw-------` | Owner Read/Write only | **SSH Private Keys (`id_rsa`), AWS API tokens, `.env` secrets.** |
| **`400`** | `-r--------` | Owner Read only | **AWS EC2 SSH keys (`keypair.pem`), strictly immutable secrets.** |
| **`644`** | `-rw-r--r--` | Owner RW, Group R, Others R | **Standard configuration files (`/etc/hosts`, Nginx `.conf`).** |
| **`700`** | `drwx------` | Owner Full access only | **SSH directories (`~/.ssh`), private user home folders.** |
| **`755`** | `-rwxr-xr-x` | Owner RWX, Group RX, Others RX | **Executable deployment scripts, compiled binaries, directories.** |
| **`775`** | `-rwxrwxr-x` | Owner RWX, Group RWX, Others RX | **Collaborative team project directories with group write.** |
| **`777`** | `-rwxrwxrwx` | **WORLD FULL ACCESS** | **NEVER USE IN PRODUCTION! (Severe security vulnerability).** |

---

## 5. Why `chmod 777` is a Fatal Antipattern in DevOps

When an inexperienced engineer runs into a `Permission denied` error, a frequent mistake is typing:
```bash
sudo chmod -R 777 /var/www/
```

### Why this is a catastrophic security failure:
1. **Arbitrary File Tampering**: Every unprivileged user, background daemon, and compromised container can overwrite your executable application code.
2. **Web Shell Uploads**: If an attacker uploads a PHP or shell script to an upload directory, `777` guarantees the script has **executable** permissions! The attacker can trigger remote code execution immediately.
3. **SSH Failure**: OpenSSH specifically checks file modes; setting `777` on `~/.ssh` or private keys causes SSH to **refuse connections entirely**.

---

## Summary

- Octal permissions represent Base-8 values: `r=4`, `w=2`, `x=1`.
- Add values together to calculate tier digits: `rw- = 4+2 = 6`, `r-x = 4+1 = 5`, `rwx = 4+2+1 = 7`.
- Standard files use `644`, scripts and directories use `755`, private secrets use `600`.
- The 4th leading digit controls special permissions (SUID=4, SGID=2, Sticky=1).
- Never use `777` in production.
