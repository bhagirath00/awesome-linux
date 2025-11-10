# The GNU `info` Documentation System

## What is GNU Info?

While **`man`** pages originated with Unix, the GNU Project introduced **`info`** (part of the GNU Texinfo project). 

GNU Info is a **hierarchical hypertext documentation reader** developed in the 1980s — years before Tim Berners-Lee invented the World Wide Web!

```text
+--------------------------------------------------------------------------+
|                        GNU INFO HYPERLINKED TREE                         |
+--------------------------------------------------------------------------+
|                     Top Node (info coreutils)                            |
|                                |                                         |
|         +----------------------+----------------------+                  |
|         |                      |                      |                  |
|         v                      v                      v                  |
|  [File Utilities]     [Text Processing]     [System Context]             |
|    - cp, ls, mv         - cat, sort, uniq      - date, uname             |
+--------------------------------------------------------------------------+
```

---

## 1. When to Use `info` vs `man`

| Feature | `man` Pages | GNU `info` |
|---|---|---|
| **Structure** | Flat, linear document | Hierarchical, multi-node hyperlinked tree |
| **Best For** | Quick command syntax, option definitions | Complete tutorials, architecture, deep technical specs |
| **GNU Tools Coverage** | Often truncated with "See info for full manual" | **Complete, authoritative GNU manuals** |

Many core GNU tools (such as `bash`, `tar`, `sed`, `gawk`, `coreutils`) contain only basic summaries in their man pages, directing you to `info` for complete documentation:

```bash
# View the ultimate 300+ page manual for the Bash shell
info bash

# View complete documentation for all coreutils utilities
info coreutils
```

---

## 2. Navigating the `info` Pager

Because `info` documents are interactive trees with cross-links, navigation differs from `less`:

| Key | Action |
|---|---|
| **`Enter`** | **Follow a hyperlink / node** (place cursor on a link starting with `*` and press Enter). |
| **`u`** | Move **Up** one level to the parent node. |
| **`n`** | Move to the **Next** sibling node at this level. |
| **`p`** | Move to the **Previous** sibling node at this level. |
| **`t`** | Jump directly to the **Top** node of the manual. |
| **`l`** | Go **Back** in history (like the Back button in a web browser). |
| **`Space`** | Scroll forward one screen (moves to next node when reaching bottom). |
| **`Backspace`** | Scroll backward one screen. |
| **`s`** / **`/`** | Search text across nodes. |
| **`q`** | **Quit** info. |

---

## Summary of Linux Documentation Tools

| Tool | When to Use It |
|---|---|
| **`command --help`** | When you just need to remember a specific flag or basic argument syntax. |
| **`tldr command`** | When you want 3 or 4 practical copy-paste examples. |
| **`man command`** | When you need the official, exhaustive offline option reference. |
| **`info command`** | When you need a deep, multi-chapter textbook on complex GNU tools (`bash`, `coreutils`). |
| **`whatis command`** | When you see an unknown command and want a 1-line description. |
| **`apropos keyword`** | When you don't know the command name and want to search by task. |
