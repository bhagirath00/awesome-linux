# Vim Advanced: Power Editing for DevOps & SREs

## Overview

Once you have mastered Vim's core modal editing mechanics, unlocking Vim's advanced capabilities transforms your terminal productivity. In DevOps environments, advanced Vim enables you to:
- Comment out dozens of YAML or Dockerfile configuration lines simultaneously using **Block Visual Mode**.
- Execute global regular expression replacements across complex URLs without syntax errors.
- Compare live production configs side-by-side using **Vimdiff**.
- Paste external code cleanly without destroying indentation via `:set paste`.
- Automate repetitive file refactoring using recorded **Macros**.

---

## 1. Visual Modes & Block Selection

Vim offers three distinct visual selection modes:

| Mode | Key Binding | Scope | Primary Use Case |
| :--- | :--- | :--- | :--- |
| **Character Visual** | `v` | Selects character by character. | Precise inline text highlighting. |
| **Line Visual** | `V` (Shift+V) | Selects entire lines. | Moving or deleting entire blocks of code. |
| **Block Visual** | `Ctrl + v` | Selects rectangular columns/matrices. | **Mass multi-line commenting, indenting, and column edits.** |

---

## 2. The Power Technique: Mass Multi-Line Commenting

One of the most practical daily DevOps skills is commenting out a block of 10 or 20 lines in a configuration file (such as Nginx, YAML, or Dockerfile):

```text
Step 1: Position cursor at the first character of the first line you want to comment.
Step 2: Press Ctrl + v  ──> Enters VISUAL BLOCK mode.
Step 3: Press j (or down arrow) repeatedly to select the first column across all target lines.
Step 4: Press Shift + I (capital I) ──> Enters Insert mode at the front of the block.
Step 5: Type the comment character (e.g., # or //).
Step 6: Press Esc!
        ──> Watch the # comment character automatically propagate to ALL selected lines instantly!
```

### Removing Mass Comments:
1. Position cursor on the first `#` character.
2. Press `Ctrl + v`.
3. Press `j` to select the `#` column across all lines.
4. Press `d` (or `x`).
5. All comments are deleted simultaneously.

---

## 3. Global Search and Replace (`:s`)

Vim features a powerful substitution engine using Unix ed/sed syntax.

### Syntax Anatomy
```vim
:[range]s/{pattern}/{replacement}/[flags]
```

### Common Substitution Recipes

| Command | Action |
| :--- | :--- |
| `:s/dev/prod/g` | Replace all occurrences of `dev` with `prod` on the **current line only**. |
| `:%s/dev/prod/g` | Replace all occurrences across the **entire file** (`%` = all lines). |
| `:%s/dev/prod/gc` | Replace with **confirmation prompt** for every single occurrence (`c` = confirm: `y/n/a/q`). |
| `:15,40s/dev/prod/g` | Replace only between lines `15` and `40`. |
| `:%s/\s\+$//e` | Strip all trailing whitespace from the end of all lines across the file. |

### Avoiding "Leaning Toothpick Syndrome" (Changing Delimiters)
When replacing file paths or URLs, standard slashes (`/`) require messy escaping (`\/usr\/local\/bin`). Vim allows using any punctuation character as a custom delimiter:

```vim
" Ugly (error-prone escaping):
:%s/\/var\/log\/nginx/\/mnt\/backups\/nginx/g

" Clean (using # as delimiter):
:%s#/var/log/nginx#/mnt/backups/nginx#g
```

---

## 4. Multi-File Splits and Diffing

Inspect multiple configuration files side-by-side inside a single terminal session without switching windows:

```vim
" Split screen horizontally
:split /etc/hosts
:sp

" Split screen vertically
:vsplit /etc/resolv.conf
:vsp
```

### Navigating Between Splits
- Press `Ctrl + w` followed by navigation keys:
  - `Ctrl + w, h` : Move cursor to the **left** window.
  - `Ctrl + w, l` : Move cursor to the **right** window.
  - `Ctrl + w, k` : Move cursor to the **top** window.
  - `Ctrl + w, j` : Move cursor to the **bottom** window.
- Close the active split with `:q`.

### Side-by-Side Configuration Diffing: `vimdiff`
To compare a newly staged configuration against the active production configuration:

```bash
vimdiff /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak
```

In `vimdiff`:
- Differences are highlighted with vivid color coding.
- `]c` : Jump to the **next** difference.
- `[c` : Jump to the **previous** difference.
- `do` (*diff obtain*): Pull changes from the other window into the current buffer.
- `dp` (*diff put*): Push changes from the current buffer to the other window.

---

## 5. The Critical Clipboard Fix: `:set paste`

When pasting code from your host machine into Vim running over SSH, auto-indentation often triggers a cascading "staircase effect" that ruins Python and YAML formatting:

```yaml
# Result of dirty paste without paste mode:
metadata:
    name: app
        spec:
            replicas: 3
                selector:  <-- RUINED!
```

### The Solution:
Before pasting text from your system clipboard:
```vim
:set paste
```
Now paste your code (`Ctrl + Shift + V` or right-click). The text pastes with exact verbatim formatting!
After pasting, restore standard indentation behavior:
```vim
:set nopaste
```

---

## 6. Automating Tasks with Macros

Macros record a sequence of keystrokes and replay them across multiple lines.

### Example: Wrap 50 lines in quotes and append a comma (`"item",`)
1. Position cursor on the first item.
2. Press `qa` to start recording into register `a` (status bar displays `recording @a`).
3. Press `I` -> type `"` -> press `Esc`.
4. Press `A` -> type `",` -> press `Esc`.
5. Press `j` (moves to next line) -> press `0` (moves to line start).
6. Press `q` to stop recording.
7. Now replay the macro on the next line: `@a`.
8. Replay the macro across the remaining 48 lines instantly: `48@a`!

---

## 7. Essential Production `.vimrc` for DevOps

Place these settings in `~/.vimrc` to configure an optimal editing environment:

```vim
" Enable syntax highlighting
syntax on

" Display line numbers in margin
set number

" Highlight current line
set cursorline

" Indentation rules for modern YAML/Python/JSON
set tabstop=2
set shiftwidth=2
set expandtab
set autoindent

" Enhanced searching
set hlsearch
set incsearch
set ignorecase
set smartcase

" Keep cursor 5 lines away from screen edge when scrolling
set scrolloff=5

" Quick toggle for paste mode using F2 key
set pastetoggle=<F2>
```

---

## Summary

- Use `Ctrl + v` (Block Visual Mode) to comment out or uncomment dozens of lines at once.
- Use `:%s#old#new#g` to execute clean file-wide replacements without escaping slashes.
- Split windows with `:sp` (horizontal) and `:vsp` (vertical); navigate with `Ctrl + w + hjkl`.
- Use `vimdiff file1 file2` for high-visibility visual diffing and merging of configuration files.
- Always use `:set paste` before pasting code over SSH to prevent YAML staircase distortion.
- Record repetitive editing actions into macros with `qa ... q` and replay with `@a`.
