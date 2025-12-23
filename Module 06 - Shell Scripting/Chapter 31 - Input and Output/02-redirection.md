# Chapter 31: Input and Output - I/O Redirection and File Descriptors

## 1. Introduction: The Three Standard Streams

Every Linux process is born with three default communication channels called **Standard Streams**, identified by numeric integer **File Descriptors (FD)**:

```
+--------------------------------------------------------------------------+
|                       Standard File Descriptors                          |
+--------------------------------------------------------------------------+
|  FD 0: Standard Input (stdin)   ===> Keyboard / File Read                |
|                                                                          |
|                     +-----------------------------+                      |
|                     |       Running Process       |                      |
|                     +-----------------------------+                      |
|                                |             |                           |
|  FD 1: Standard Output (stdout)v             v FD 2: Standard Error (stderr)|
|  Terminal Screen (Success / Data)           Terminal Screen (Errors/Logs)|
+--------------------------------------------------------------------------+
```

| Stream Name | File Descriptor | Default Target | Purpose |
| :--- | :--- | :--- | :--- |
| **`stdin`** | **`0`** | Keyboard input | Feeds input data into the process. |
| **`stdout`**| **`1`** | Terminal screen | Normal, successful program output. |
| **`stderr`**| **`2`** | Terminal screen | Diagnostic error messages and debug logs. |

---

## 2. Redirection Operators Summary

| Redirection Operator | Function | Example |
| :--- | :--- | :--- |
| **`>`** | Overwrite `stdout` to file. | `echo "v1.0" > version.txt` |
| **`>>`** | Append `stdout` to file. | `date >> activity.log` |
| **`<`** | Read `stdin` from file. | `mysql < database.sql` |
| **`2>`** | Overwrite `stderr` to file. | `command 2> error.log` |
| **`2>>`** | Append `stderr` to file. | `command 2>> error.log` |
| **`&>`** or **`>&`** | Redirect **both** `stdout` and `stderr` (Bash). | `command &> combined.log` |
| **`2>&1`** | Merge `stderr` (2) into `stdout` (1) (POSIX). | `command > combined.log 2>&1` |
| **`> /dev/null 2>&1`**| Discard **all** output completely (Silent). | `ping -c 1 8.8.8.8 > /dev/null 2>&1` |

---

## 3. Dissecting the Classic: `> /dev/null 2>&1`

A frequent question in Linux interviews and production troubleshooting:
```bash
command > /dev/null 2>&1
```

Why must it be written in this exact order?

1. **`> /dev/null`**: Redirects file descriptor 1 (`stdout`) to the bit-bucket `/dev/null` (data written here is instantly discarded by the kernel).
2. **`2>&1`**: Redirects file descriptor 2 (`stderr`) to point to the current destination of file descriptor 1 (which is now `/dev/null`).

### The Order Trap:
If you write `command 2>&1 > /dev/null`:
- First, `2>&1` points `stderr` to the screen (where `stdout` was pointing originally).
- Then, `> /dev/null` redirects `stdout` to `/dev/null`.
- **Result: Errors still print to your terminal screen!**

*(In modern Bash, the shortcut **`&> /dev/null`** achieves the exact same result in cleaner syntax).*

---

## 4. Redirecting Output for an Entire Script Block

Instead of appending `>> /var/log/deploy.log 2>&1` to 50 separate command lines in a script, you can redirect standard output and error for an **entire code block** or the **entire script**:

### Method 1: Block Redirection
```bash
{
    echo "Starting build step 1..."
    make compile
    echo "Starting test step 2..."
    make test
} >> /var/log/build.log 2>&1
```

### Method 2: Global Redirection using `exec`
The `exec` built-in can replace the file descriptors for the entire running script:

```bash
#!/usr/bin/env bash

# Redirect all future stdout and stderr of this script to deploy.log
exec >> /var/log/deploy.log 2>&1

echo "This message goes to /var/log/deploy.log"
uptime
df -h
echo "Everything in this script is logged automatically!"
```

---

## 5. Summary Takeaways

- File descriptors: `0` (stdin), `1` (stdout), `2` (stderr).
- Use `>` to overwrite and `>>` to append.
- Redirect errors using `2>` or discard with `2> /dev/null`.
- Merge stderr into stdout using `2>&1` or `&>`.
- Use `exec >> log.txt 2>&1` at the top of scripts to automate centralized file logging.
