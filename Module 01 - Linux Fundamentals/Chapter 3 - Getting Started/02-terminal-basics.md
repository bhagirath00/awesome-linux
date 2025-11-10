# Terminal Basics: The Prompt, Shortcuts & History Mastery

## 1. Anatomy of the Linux Prompt (`PS1`)

When you open a terminal or SSH into a server, the shell presents a prompt string:

```text
ubuntu@prod-api-01:/var/log$
  |          |        |    |
  |          |        |    +-- Privilege Indicator: '$' = Regular user, '#' = Root
  |          |        +------- Current Working Directory (~ = /home/ubuntu)
  |          +---------------- Hostname of the server
  +--------------------------- Logged-in Username
```

- **`$`**: Indicates an unprivileged standard user.
- **`#`**: Indicates the **`root` superuser** (exercise extreme caution).

---

## 2. Who Am I & Where Am I?

Whenever you log into a Linux server, always orient yourself with these four fundamental commands:

```bash
# 1. Print Working Directory (shows absolute path of where you are standing)
pwd
# Output: /home/ubuntu

# 2. View your current active username
whoami
# Output: ubuntu

# 3. View your User ID (UID), Primary Group ID (GID), and groups
id
# Output: uid=1000(ubuntu) gid=1000(ubuntu) groups=1000(ubuntu),4(adm),27(sudo),999(docker)

# 4. View server hostname and IP configurations
hostname
hostname -I
```

---

## 3. High-Velocity Terminal Shortcuts

Mastering terminal shortcuts will make you 5x faster in daily DevOps operations:

### Cursor Navigation
| Shortcut | Action |
|---|---|
| **`Ctrl + A`** | Move cursor to the **beginning** of the line. |
| **`Ctrl + E`** | Move cursor to the **end** of the line. |
| **`Alt + F`** | Move cursor forward by one word. |
| **`Alt + B`** | Move cursor backward by one word. |

### Text Editing & Clearing
| Shortcut | Action |
|---|---|
| **`Ctrl + U`** | Cut/erase everything from the cursor back to the start of the line. |
| **`Ctrl + K`** | Cut/erase everything from the cursor to the end of the line. |
| **`Ctrl + W`** | Erase the single word immediately before the cursor. |
| **`Ctrl + L`** | Clear the screen (leaves command at the top, identical to `clear`). |

### Process Control
| Shortcut | Action |
|---|---|
| **`Ctrl + C`** | Send `SIGINT` to interrupt, cancel, or terminate the currently running command. |
| **`Ctrl + Z`** | Send `SIGTSTP` to pause/suspend the running process and send it to the background. |
| **`Ctrl + D`** | Send `EOF` (End of File) — closes the current shell session or logs out. |

---

## 4. History & Recall Power-Tricks

### 1. Reverse Interactive History Search (`Ctrl + R`)
Never re-type a long `docker run` or `kubectl` command from yesterday. Press **`Ctrl + R`** and start typing any substring of the command:

```text
(reverse-i-search)`kub`: kubectl get pods -n production -o wide
# Press Enter to execute, or Right Arrow to edit the command!
```

### 2. The "Permission Denied" Fix (`sudo !!`)
If you ran a command and forgot `sudo`:

```bash
apt update
# Output: Permission denied: are you root?

# Execute the exact last command prepended with sudo:
sudo !!
# Executes: sudo apt update
```

### 3. Using the Last Argument (`!$`)
If you just created a directory and want to navigate into it immediately:

```bash
mkdir -p /var/log/my-service/traces
cd !$
# Executes: cd /var/log/my-service/traces
```

### 4. Search and Execute from History
```bash
# View recent commands with line numbers
history | tail -n 15

# Re-run command by its history line number (e.g., line 482)
!482
```
