# The GNU Nano Text Editor in Linux

## Overview

**GNU Nano** is an easy-to-use, modeless command-line text editor available on virtually all Linux distributions. Unlike Vim or Emacs, which rely on modal states (insert, command, visual), Nano behaves like a traditional desktop text editor: you type directly to insert text, navigate with standard arrow keys, and perform operations using `Ctrl` and `Alt` keyboard shortcuts displayed permanently at the bottom of the screen.

In DevOps, Nano is often the default interactive editor configured on Ubuntu, Debian, and Raspberry Pi OS instances. It is ideal for quick configuration updates (`/etc/hosts`, `~/.bashrc`), writing simple scripts, and onboarding engineers new to the Linux terminal.

---

## Launching Nano

```bash
# Open an existing file or create a new file
nano /etc/nginx/sites-available/default

# Open directly at a specific line and column number
nano +45,12 /etc/fstab

# Open in view-only (read-only) mode
nano -v /var/log/syslog
```

---

## Interface Anatomy

Nano's terminal screen is divided into four primary regions:

```text
+--------------------------------------------------------------------+
| GNU nano 7.2              File: /etc/hosts                         | <- [ Title Bar ]
+--------------------------------------------------------------------+
| 127.0.0.1   localhost                                              |
| 127.0.1.1   web-server-prod-01                                     |
|                                                                    | <- [ Edit Area ]
| # The following lines are desirable for IPv6 capable hosts         |
| ::1     ip6-localhost ip6-loopback                                 |
+--------------------------------------------------------------------+
| [ Read 5 lines ]                                                   | <- [ Status Message ]
+--------------------------------------------------------------------+
| ^G Help       ^O WriteOut   ^W Where Is   ^K Cut        ^J Justify |
| ^X Exit       ^R Read File  ^\ Replace    ^U Paste      ^C Location| <- [ Shortcut Bar ]
+--------------------------------------------------------------------+
```

### Understanding Shortcut Notation
- **`^` (Caret)**: Press the **`Ctrl`** key + the indicated letter (e.g., `^X` = `Ctrl + X`).
- **`M-` (Meta)**: Press the **`Alt`** key (or press `Esc` then the letter). For example, `M-U` = `Alt + U` (Undo).

---

## Essential Nano Shortcut Cheatsheet

### 1. File Management & Exiting

| Shortcut | Function | Description |
| :--- | :--- | :--- |
| `Ctrl + O` | **WriteOut** | Saves the file without exiting. Prompts for filename (press `Enter` to confirm). |
| `Ctrl + X` | **Exit** | Quits Nano. If changes are unsaved, prompts `Save modified buffer? (Y/N)`. |
| `Ctrl + R` | **Read File** | Inserts the contents of an external file into the current document at cursor position. |

### 2. Searching and Replacing

| Shortcut | Function | Description |
| :--- | :--- | :--- |
| `Ctrl + W` | **Where Is** | Searches forward for a string or regex pattern. |
| `Alt + W` | **Repeat Search** | Finds the next occurrence of the last search term. |
| `Ctrl + \` | **Replace** | Interactive search and replace throughout the document. |

### 3. Cursor Navigation

| Shortcut | Description |
| :--- | :--- |
| `Ctrl + A` | Jump to the **start of the current line**. |
| `Ctrl + E` | Jump to the **end of the current line**. |
| `Ctrl + Y` | Page Up (scroll back one full screen). |
| `Ctrl + V` | Page Down (scroll forward one full screen). |
| `Ctrl + _` *(or `Alt + G`)* | **Go to Line Number**: Prompts for target line and column. |
| `Ctrl + C` | Displays current line, column, and character position in status bar. |

### 4. Text Manipulation (Cut, Copy, Paste, Undo)

| Shortcut | Description |
| :--- | :--- |
| `Ctrl + K` | **Cut (Kill)** the entire current line into the clipboard buffer. |
| `Alt + 6` *(or `Alt + ^`)* | **Copy** the current line or active selection into the clipboard. |
| `Ctrl + U` | **Paste (Uncut)** the clipboard buffer at current cursor location. |
| `Alt + U` | **Undo** the last action. |
| `Alt + E` | **Redo** the last undone action. |

---

## Critical DevOps Flags When Running Nano

When editing system configuration files (e.g., `/etc/fstab`, SSH keys, crontabs), standard Nano behaviors can inadvertently corrupt files if you are not careful:

### 1. Disable Automatic Line Wrapping (`-w` / `--nowrap`)
By default, older Nano configurations wrap long lines automatically by inserting hard newline characters. In files like `/etc/fstab` or `authorized_keys`, this invalidates the configuration!
```bash
# ALWAYS disable line wrapping for critical configs:
nano -w /etc/fstab
```

### 2. Display Line Numbers (`-l` / `--linenumbers`)
When debugging compiler errors or Python syntax traces:
```bash
nano -l /app/server.py
```

### 3. Automatic Backup Before Overwrite (`-B` / `--backup`)
Creates a backup file with a tilde `~` suffix (e.g., `nginx.conf~`) before saving changes:
```bash
nano -B /etc/nginx/nginx.conf
```

---

## Customizing Nano: `~/.nanorc`

Configure permanent developer-friendly defaults in your user configuration file:

```bash
cat << 'EOF' > ~/.nanorc
# Enable line numbers in margin
set linenumbers

# Disable destructive line wrapping
set nowrap

# Use 2 spaces for tabs (standard for YAML/JSON)
set tabsize 2
set tabstospaces

# Enable mouse support (click to position cursor)
set mouse

# Enable smooth scrolling
set smooth

# Highlight syntax for common formats
include "/usr/share/nano/*.nanorc"
EOF
```

---

## Configuring the Default System Editor

Many Linux utilities (such as `crontab -e`, `visudo`, `git commit`) automatically invoke the default system editor:

```bash
# Set default editor for the current user in ~/.bashrc:
export EDITOR="nano"
export VISUAL="nano"

# On Debian/Ubuntu, configure the system-wide default interactive editor:
sudo update-alternatives --config editor
```

---

## Summary

- Nano is a beginner-friendly modeless editor with visible shortcut keys on screen.
- Save files with `Ctrl + O` -> `Enter`; exit with `Ctrl + X`.
- Search with `Ctrl + W`; find/replace with `Ctrl + \`.
- Always use `nano -w` to prevent destructive automatic line wrapping when editing configuration files like `/etc/fstab` or authorized keys.
- Customize Nano preferences permanently via `~/.nanorc`.
