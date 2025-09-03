# Locating Commands in Linux: `which`, `whereis`, and `type`

## Overview

When you type a command in a Linux terminal—such as `python3`, `docker`, or `git`—the shell does not search the entire hard drive. Instead, it consults the **`$PATH`** environment variable, scanning a colon-separated list of directories from left to right until it finds a matching executable.

In DevOps automation, CI/CD pipelines, and configuration management, determining **which binary is executing**, where its documentation resides, and whether a command is a binary, an alias, or a shell builtin is essential for preventing version collisions and build failures.

---

## The Four Discovery Utilities Compared

| Tool | Type | What it Searches | Detects Aliases / Builtins? | Best Used For |
| :--- | :--- | :--- | :--- | :--- |
| **`which`** | External command / script | Directories in the current user's `$PATH`. | No (Usually ignores shell functions/builtins). | Quick path lookup of CLI utilities. |
| **`whereis`** | Binary utility | Standard system binary, manpage, and source directories. | No. | Finding both the executable and its man page documentation simultaneously. |
| **`type`** | **Shell Builtin** | Shell keywords, builtins, functions, aliases, and `$PATH`. | **Yes (Full inspection)**. | Determining how the current shell will actually interpret a command. |
| **`command -v`**| **POSIX Shell Builtin** | Current shell execution path. | No (Omits functions/aliases when specified). | **The gold standard for CI/CD and portability scripts.** |

---

## 1. The `which` Command

`which` searches the directories listed in your `$PATH` and prints the absolute path of the first matching executable:

```bash
$ which docker
/usr/bin/docker

$ which terraform
/usr/local/bin/terraform
```

### Finding All Matching Binaries in `$PATH` (`-a`)
If multiple versions of a tool are installed (e.g., Python 3 in `/usr/bin/python3` and an active virtual environment in `/opt/venv/bin/python3`):

```bash
$ which -a python3
/opt/venv/bin/python3
/usr/bin/python3
```
*(The first entry in the list is the one that executes by default because its directory appeared earlier in `$PATH`).*

---

## 2. The `whereis` Command

Unlike `which`, which strictly follows `$PATH`, `whereis` searches standard system locations for:
1. Binary executables (`-b`)
2. Manual pages / documentation (`-m`)
3. Source code trees (`-s`)

```bash
$ whereis nginx
nginx: /usr/sbin/nginx /etc/nginx /usr/share/nginx /usr/share/man/man8/nginx.8.gz
```

### Filtering Specific Components
```bash
# Locate binaries only:
whereis -b nginx
# nginx: /usr/sbin/nginx /etc/nginx

# Locate manual documentation pages only:
whereis -m nginx
# nginx: /usr/share/man/man8/nginx.8.gz
```

---

## 3. The Superior Utility: Shell Builtin `type`

`which` has a major flaw: it knows nothing about shell aliases, shell functions, or shell builtins (`cd`, `echo`, `read`, `export`).

The bash builtin `type` reveals the **exact nature** of any command:

```bash
# Shell Builtin:
$ type cd
cd is a shell builtin

# Shell Keyword:
$ type if
if is a shell keyword

# Aliased Command:
$ type ll
ll is aliased to `ls -lah --color=auto'

# External Binary:
$ type git
git is /usr/bin/git
```

### Inspecting All Occurrences with `type -a`
```bash
$ type -a echo
echo is a shell builtin
echo is /usr/bin/echo
echo is /bin/echo
```
This output reveals that when you type `echo`, Bash executes the fast internal **builtin** rather than spawning `/usr/bin/echo` as an external child process!

---

## 4. The DevOps Gold Standard: `command -v` in Scripts

In production bash scripts, Docker entrypoints, and CI/CD pipelines, **avoid using `which`**. `which` behaves inconsistently across distributions (on Alpine or minimal containers, `which` is a BusyBox symlink that may return differing exit codes).

Instead, POSIX standards specify **`command -v`**:

### Recipe: Validate Required Dependencies in Deployment Scripts
```bash
#!/usr/bin/env bash
set -euo pipefail

REQUIRED_TOOLS=(docker git jq curl kubectl)

for tool in "${REQUIRED_TOOLS[@]}"; do
    if ! command -v "$tool" >/dev/null 2>&1; then
        echo "Error: Required deployment dependency '$tool' is not installed or not in PATH." >&2
        exit 1
    fi
done

echo "All required dependencies verified successfully."
```

### Why `command -v` is superior:
1. Built directly into the shell (zero external process fork overhead).
2. Guaranteed to exist on every POSIX-compliant system (Debian, RHEL, Alpine, macOS).
3. Returns standard exit codes: `0` if found, `>0` if missing.

---

## Understanding `$PATH` Resolution Order

The `$PATH` variable is read sequentially from left to right:
```bash
$ echo $PATH
/home/ubuntu/.local/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin
```

If a custom binary exists in `/home/ubuntu/.local/bin/node` and the system binary is in `/usr/bin/node`, Linux executes the version in `/home/ubuntu/.local/bin` first. This is how package managers like `nvm` (Node Version Manager) and `pyenv` seamlessly switch software versions.

---

## Summary

- `which binary` finds the first matching executable in the user's `$PATH`.
- `whereis binary` finds the binary, configuration directories, and man pages.
- `type -a command` is the ultimate investigative tool, exposing aliases, shell builtins, and functions.
- Never use `which` in automated scripts; always use `command -v <tool> >/dev/null 2>&1` for POSIX-compliant dependency verification.
