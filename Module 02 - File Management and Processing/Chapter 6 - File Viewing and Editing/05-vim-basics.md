# Vim Basics: The DevOps Survival Guide

## Overview

**Vim** (Vi IMproved) is a ubiquitous, highly efficient, modal terminal text editor based on Bill Joy's original 1976 Unix `vi`.

In DevOps, cloud infrastructure, and Site Reliability Engineering, **Vim is the ultimate non-negotiable tool**. When you SSH into a remote cloud virtual machine, debug a crash inside an Alpine Linux container, drop into a single-user recovery emergency shell, or edit files via `kubectl exec`, graphical editors and Nano are often unavailable. Vim (or a minimal `vi` binary via BusyBox) is guaranteed to exist on virtually every Linux system on earth.

---

## 1. The Core Concept: Modal Editing

The single most important concept to grasp about Vim is that it is a **modal editor**. In standard editors, every key you type inserts a character onto the screen. In Vim, keys change meaning depending on the current **mode**:

```text
                     +---------------------------------------+
                     |              NORMAL MODE              |
                     | (Navigate, Delete, Yank, Paste, Undo) |
                     +---------------------------------------+
                       │   ▲               │   ▲       │   ▲
             i, a, o   │   │ Esc           │   │ Esc   │   │ Esc
                       ▼   │           :   ▼   │       │   │
                     +---------+       +-----------+   │   │
                     | INSERT  |       |  COMMAND  |   │   │
                     |  MODE   |       |   MODE    |   │   │
                     | (Type)  |       | (:w, :q)  |   │   │
                     +---------+       +-----------+   │   │
                                                       ▼   │ v, V, Ctrl+v
                                                   +-----------+
                                                   |  VISUAL   |
                                                   |   MODE    |
                                                   | (Select)  |
                                                   +-----------+
```

### The Golden Rule of Vim:
> **When in doubt or lost, press the `Esc` key repeatedly (twice) to return to NORMAL Mode.**

---

## 2. Entering Insert Mode (Typing Text)

From **Normal Mode**, press any of the following keys to start typing:

| Key | Action | Where Typing Begins |
| :--- | :--- | :--- |
| `i` | **Insert** | Immediately *before* the current cursor position. |
| `a` | **Append** | Immediately *after* the current cursor position. |
| `I` | **Insert at Line Start** | At the first non-blank character of the current line. |
| `A` | **Append at Line End** | At the very end of the current line. |
| `o` | **Open Line Below** | Inserts a new blank line *below* current line and enters Insert mode. |
| `O` | **Open Line Above** | Inserts a new blank line *above* current line and enters Insert mode. |
| `s` | **Substitute** | Deletes the character under cursor and enters Insert mode. |

*To stop typing and return to Normal mode, press **`Esc`**.*

---

## 3. Basic Navigation in Normal Mode

While the standard arrow keys work in modern Vim, keeping your hands on the home row using the classic `hjkl` keys is faster and works across every legacy terminal:

```text
          k (Up)
          ▲
          │
h (Left) ◄──┼──► l (Right)
          │
          ▼
          j (Down)
```

### Word and Line Navigation
- `w` : Jump forward to the start of the **next word**.
- `b` : Jump backward to the start of the **previous word**.
- `e` : Jump forward to the end of the word.
- `0` *(zero)* : Jump to the **very start** of the current line.
- `^` : Jump to the first non-whitespace character of the current line.
- `$` : Jump to the **very end** of the current line.

### Document-Wide Navigation
- `gg` : Jump to the **first line** of the file (top).
- `G` : Jump to the **last line** of the file (bottom).
- `42G` *(or `:42` + Enter)* : Jump directly to line number `42`.
- `Ctrl + d` : Scroll down half a screen.
- `Ctrl + u` : Scroll up half a screen.

---

## 4. Editing Operations in Normal Mode

Vim commands follow an intuitive grammar: **`[Verb / Action] + [Count] + [Noun / Motion]`**

### Deleting (Cutting)
- `x` : Delete single character under cursor.
- `dw` : Delete from cursor to the end of the word.
- `d$` : Delete from cursor to the end of the line.
- `dd` : **Delete (cut) entire line**.
- `5dd` : Delete (cut) 5 consecutive lines.
- `D` : Delete from cursor to end of line (equivalent to `d$`).

### Copying (Yanking) & Pasting
- `yy` : **Yank (copy) entire line**.
- `3yy` : Yank (copy) 3 lines.
- `yw` : Yank word.
- `p` : **Paste** clipboard content *below* current line.
- `P` : Paste clipboard content *above* current line.

### Changing Text (Delete + Enter Insert Mode)
- `cw` : Change word (deletes word and puts you in Insert mode).
- `cc` : Change entire line.
- `C` : Change from cursor to end of line.
- `r<char>` : Replace single character under cursor with `<char>` without entering Insert mode.

### Undo and Redo
- `u` : **Undo** last change.
- `Ctrl + r` : **Redo** previously undone change.

---

## 5. Command-Line Mode: Saving and Exiting

From **Normal Mode**, press `:` (colon) to enter Command-Line mode at the bottom of the screen.

| Command | Action |
| :--- | :--- |
| `:w` | **Write** (Save changes to disk). |
| `:w filename.txt` | Save as a new file name. |
| `:q` | **Quit** Vim (fails if unsaved changes exist). |
| `:wq` *(or `:x` or `ZZ`)* | **Save and Quit** in one step. |
| `:q!` *(or `ZQ`)* | **Force Quit** discarding all unsaved changes. |

---

## 6. The DevOps Life-Saver: "Forgot Sudo" Save Trick

A universal rite of passage for every Linux engineer: you spend 15 minutes editing `/etc/nginx/nginx.conf`, type `:wq`, and get hit with:

```text
E212: Can't open file for writing
Press ENTER or type command to continue
```

You forgot to open the file with `sudo`! Instead of losing your edits and retyping everything:

```vim
:w !sudo tee %
```

### How this works:
1. `:w` pipes the Vim editor buffer to standard input.
2. `!sudo tee` executes the external `tee` command as the root superuser.
3. `%` is a Vim shorthand variable representing the current file path (`/etc/nginx/nginx.conf`).
4. `tee` writes the buffer to the file with root permissions!
5. When prompted, reload the file in Vim (`L`) and quit (`:q!`).

---

## Summary

- Vim has 4 primary modes: Normal, Insert, Command, and Visual.
- Always press `Esc` to return to Normal Mode.
- Enter Insert mode with `i` (insert), `a` (append), or `o` (open line below).
- Navigate efficiently with `h j k l`, `w b`, and `gg` / `G`.
- Cut with `dd`, copy with `yy`, paste with `p`, undo with `u`.
- Save and exit with `:wq`; discard changes and force quit with `:q!`.
- Save root files opened without sudo using `:w !sudo tee %`.
