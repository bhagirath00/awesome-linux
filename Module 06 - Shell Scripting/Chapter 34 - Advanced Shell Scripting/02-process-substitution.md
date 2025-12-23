# Chapter 34: Advanced Shell Scripting - Process Substitution

## 1. Introduction: What is Process Substitution?

In Linux, many command-line tools expect a **file path argument** on disk (e.g., `diff file1 file2` or `source config.env`) and do not accept input via standard pipes (`|`).

Traditionally, engineers worked around this by writing intermediate outputs to temporary files on disk:
```bash
# The slow, messy temporary file approach:
curl -s http://server-a/config > /tmp/conf_a.txt
curl -s http://server-b/config > /tmp/conf_b.txt
diff -u /tmp/conf_a.txt /tmp/conf_b.txt
rm -f /tmp/conf_a.txt /tmp/conf_b.txt
```

**Process Substitution** eliminates the need for temporary files entirely. It executes a command inside a subshell and connects its standard stream to a dynamic **named pipe (FIFO)** or a virtual file descriptor node in **`/dev/fd/<N>`**. To the receiving program, the process output appears as a real, readable file on disk!

```
+--------------------------------------------------------------------------+
|                     Process Substitution Mechanics                       |
+--------------------------------------------------------------------------+
|  User types: diff <(curl http://srv1/conf) <(curl http://srv2/conf)      |
|                                                                          |
|  1. Subshell 1 executes: curl http://srv1/conf                           |
|  2. Subshell 2 executes: curl http://srv2/conf                           |
|  3. Kernel creates virtual file descriptors: /dev/fd/63 and /dev/fd/62   |
|  4. diff executes: diff /dev/fd/63 /dev/fd/62                            |
|  5. File descriptors automatically closed and torn down!                 |
+--------------------------------------------------------------------------+
```

---

## 2. Syntax: `<(command)` vs. `>(command)`

> [!WARNING]
> Notice that there is **NO SPACE** between `<` (or `>`) and the opening parenthesis `(`! Writing `< (command)` causes a syntax error.

### 1. Reading from a Process: `<(command)`
Acts as a readable input file containing the output of `command`:
```bash
diff -u <(sort list1.txt) <(sort list2.txt)
```

### 2. Writing to a Process: `>(command)`
Acts as a writable output destination file that feeds standard input into `command`:
```bash
# Stream output to an application while simultaneously feeding a compressed archive
tar -cf >(gzip -9 > backup.tar.gz) /data/
```

---

## 3. Top Production Use Cases

### 1. Comparing Two Live Server Configurations
Compare the live Nginx configurations on two remote production web nodes without copying files locally:

```bash
diff -u <(ssh web01 "cat /etc/nginx/nginx.conf") <(ssh web02 "cat /etc/nginx/nginx.conf")
```

### 2. Fixing the Piped While-Loop Subshell Trap
As covered in Chapter 28, piping into a `while` loop spawns a subshell, preventing variables inside the loop from persisting. 

Process substitution solves this permanently while keeping code clean:

```bash
TOTAL_USERS=0

# Feeds getent output into loop as an input file; loop remains in parent shell!
while IFS=: read -r username _ uid _; do
    if [[ "${uid}" -ge 1000 ]]; then
        TOTAL_USERS=$((TOTAL_USERS + 1))
    fi
done < <(getent passwd)

echo "Total human users on system: ${TOTAL_USERS}"
```
```
Total human users on system: 4
```

### 3. Comparing Local and Remote Checksums
```bash
cmp <(md5sum local-release.zip | cut -d' ' -f1) <(curl -s https://releases.company.com/release.zip.md5)
```

---

## 4. Summary Takeaways

- Process substitution (`<(cmd)`) turns command output into a virtual file path in `/dev/fd/`.
- Eliminates manual scratch and temporary file creation.
- Avoid spaces between the angle bracket and parenthesis: `<(cmd)`, not `< (cmd)`.
- Use `done < <(command)` to safely iterate loops without subshell variable loss.
- Compare remote servers cleanly using `diff <(ssh srv1) <(ssh srv2)`.
