# Chapter 27: Shell Scripting Basics - Bash vs. Sh (POSIX Compliance)

## 1. Introduction: The Evolution of Unix Shells

A frequent source of critical bugs and deployment failures in Linux automation stems from confusing **`sh`** with **`bash`**. 

Many developers write scripts using Bash-specific syntax features (known as **Bashisms**), but begin their script with `#!/bin/sh`. While this might appear to work on systems where `/bin/sh` is symlinked to `bash`, it causes catastrophic syntax errors on Ubuntu, Debian, Alpine Linux, and FreeBSD where `/bin/sh` points to lightweight, POSIX-strict shells like **Dash** or **BusyBox ash**!

```
+--------------------------------------------------------------------------+
|                          Unix Shell Timeline & Lineage                   |
+--------------------------------------------------------------------------+
|  1979: Bourne Shell (`sh` / V7 Unix by Steve Bourne)                     |
|        - The grandfather of all modern Unix shells.                      |
|                                |                                         |
|                                v                                         |
|  1989: GNU Bash (Bourne-Again Shell by Brian Fox)                        |
|        - Powerful superset of `sh` featuring arrays, [[ ]], regex, etc.  |
|                                |                                         |
|                                v                                         |
|  1992: POSIX Standard (IEEE Std 1003.2)                                  |
|        - Codified minimum standard portable syntax for all Unix shells.  |
|                                |                                         |
|          +---------------------+---------------------+                   |
|          |                                           |                   |
|          v                                           v                   |
|  Debian Almquist Shell (`dash`)               GNU Bash (`bash`)          |
|  - Ultra-fast, lightweight (150KB)            - Feature-rich, heavy (1MB)|
|  - Strictly POSIX compliant                   - Rich extensions/Bashisms |
|  - Default /bin/sh on Ubuntu/Debian           - Default interactive CLI  |
+--------------------------------------------------------------------------+
```

---

## 2. Where Does /bin/sh Point on Modern Linux?

The target of the `/bin/sh` symlink varies significantly across distributions:

```bash
# Check where /bin/sh points on your system:
ls -l /bin/sh
```

- **On Ubuntu / Debian**:
  `/bin/sh -> dash` (Dash is used for system startup scripts because it boots up to 4x faster and consumes minimal RAM).
- **On RHEL / Rocky Linux / CentOS**:
  `/bin/sh -> bash` (Bash executes in backward-compatible POSIX compatibility mode when invoked as `sh`).
- **On Alpine Linux (Docker Containers)**:
  `/bin/sh -> /bin/busybox` (BusyBox ash; strictly POSIX-compliant; rejects all Bashisms).

---

## 3. Bashisms: Common Features Unsupported by POSIX /bin/sh

If your script starts with `#!/bin/sh`, you **cannot** use the following syntax constructs:

| Feature | GNU Bash (Allowed) | POSIX /bin/sh (Standard) | Why it Breaks in `sh` |
| :--- | :--- | :--- | :--- |
| **Test Condition** | `[[ $a == $b ]]` | `[ "$a" = "$b" ]` | `[[` is a Bash keyword. In pure `sh`, `[[` returns a syntax error! |
| **String Comparison**| `[ "$a" == "$b" ]` | `[ "$a" = "$b" ]` | `==` is a Bashism; POSIX standard requires a single `=`. |
| **Arrays** | `arr=("web1" "web2")` | **Not supported** | Pure `sh` has zero support for array data structures. |
| **String Slicing** | `${str:0:4}` | Use `cut` or `awk` | Parameter expansion slicing is unsupported in POSIX `sh`. |
| **Regex Matching** | `[[ $str =~ ^[0-9]+$ ]]`| Use `grep -E` | Regex operator `=~` is strictly Bash. |
| **Source File** | `source ./config.sh` | `. ./config.sh` | `source` is a Bashism; POSIX uses the dot (`.`) command. |
| **Process Substitution**| `<(curl -s URL)` | Temporary named pipe | `<(...)` requires `/dev/fd` and subshell multiplexing. |
| **Function Keyword**| `function my_func {` | `my_func() {` | The `function` keyword is invalid POSIX syntax. |

---

## 4. Demonstrating the Failure in Action

Consider this simple script named `test.sh`:

```bash
#!/bin/sh
# WRONG: using Bashisms under #!/bin/sh
arr=("alpha" "beta" "gamma")
if [[ "${arr[0]}" == "alpha" ]]; then
    echo "Matched!"
fi
```

- Running on **RHEL/CentOS** (`/bin/sh -> bash`): It might appear to succeed.
- Running on **Ubuntu** or inside an **Alpine Docker Container** (`/bin/sh -> dash`):
```
./test.sh: 3: Syntax error: "(" unexpected
```
The deployment crashes immediately!

### How to Fix:
Choose one of two clean approaches:
1. **If you need Bash features, declare Bash explicitly in the Shebang**:
   ```bash
   #!/bin/bash
   ```
2. **If you must use `#!/bin/sh` (for maximum Alpine/POSIX portability), write strict POSIX code**:
   ```bash
   #!/bin/sh
   # Pure POSIX: use arguments or space-delimited strings
   val="alpha"
   if [ "$val" = "alpha" ]; then
       echo "Matched!"
   fi
   ```

---

## 5. Detecting Bashisms with `checkbashisms` and `shellcheck`

Install automated static linters in your CI/CD pipeline to catch unintentional Bashisms before deployment:

```bash
# Ubuntu/Debian
sudo apt install -y devscripts shellcheck

# Lint script for non-portable syntax
checkbashisms deploy.sh
shellcheck deploy.sh
```

---

## 6. Summary Takeaways

- `bash` is a powerful superset with rich extensions; `sh` is the strict, portable POSIX standard.
- On Ubuntu/Debian, `/bin/sh` is **Dash**; on Alpine Linux, it is **BusyBox**.
- Never use `[[ ]]`, `==`, `arr=()`, or `source` inside a script marked with `#!/bin/sh`.
- If you use Bash features, **always use `#!/bin/bash`**.
- Validate script portability using `shellcheck`.
