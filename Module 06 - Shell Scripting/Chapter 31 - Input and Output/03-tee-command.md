# Chapter 31: Input and Output - The tee Command (Splitting Streams)

## 1. Introduction: What is the tee Command?

The `tee` utility is named after the plumbing **T-splitter** fitting. It reads data from standard input (`stdin`) and writes it simultaneously to two destinations:
1. **Standard Output (`stdout`)** (the terminal screen or next piped command).
2. **One or more Files on Disk**.

In DevOps pipelines and CI/CD jobs, `tee` allows engineers to observe real-time output in their terminal or build console while simultaneously archiving an identical, uncorrupted copy to disk for post-mortem auditing.

```
                                  +-------------------+
                             +--> |   Disk File       | (output.log)
                             |    +-------------------+
+--------------------+   +---+--+
| Command (stdout)   | =>| tee  |
+--------------------+   +---+--+
                             |    +-------------------+
                             +--> | Terminal Display  | (stdout screen)
                                  +-------------------+
```

---

## 2. Basic Command Syntax and Execution

**Syntax:**
```bash
command | tee [options] <file...>
```

```bash
# Display output on screen while saving to /tmp/docker-build.log (Overwriting)
docker build -t my-app:latest . | tee /tmp/docker-build.log
```

### Appending to Files (`-a`)
By default, `tee` overwrites existing files. To append new entries to an ongoing log file, use the **`-a`** (or `--append`) flag:

```bash
date | tee -a /var/log/cron-executions.log
```

### Splitting to Multiple Files Simultaneously
```bash
echo "Release v2.4 deployed" | tee prod.log backup.log audit.log
```

---

## 3. The Classic DevOps Problem: `sudo echo` vs. `sudo tee`

One of the most frequent permission errors encountered by Linux engineers:

```bash
# This FAILS with "Permission denied":
sudo echo "10.0.1.25 internal-db" >> /etc/hosts
```

### Why Did It Fail?
Shell redirection (`>>`) is evaluated by the **current unprivileged shell**, *before* `sudo` is even executed! The `echo` command ran with root privileges, but the shell attempting to open `/etc/hosts` for writing has only standard user permissions.

### The Solution: `sudo tee -a`
Pipe the unprivileged `echo` command into `tee` running under `sudo`:

```bash
# SUCCEEDS:
echo "10.0.1.25 internal-db" | sudo tee -a /etc/hosts
```

Because `tee` is executed directly by `sudo`, it possesses root write privileges to `/etc/hosts`!

*(Tip: To prevent `tee` from printing the line back to your terminal screen, redirect its output: `echo "..." | sudo tee -a /etc/hosts > /dev/null`)*.

---

## 4. Capturing Both stdout and stderr with tee

By default, pipes (`|`) only redirect standard output (FD 1). Errors printed to `stderr` (FD 2) bypass `tee` and are not written to the log file.

To capture **both normal output and error messages** into the log file:

```bash
# Bash shortcut: pipe both streams via |&
./build-and-deploy.sh |& tee deployment.log

# POSIX standard equivalent:
./build-and-deploy.sh 2>&1 | tee deployment.log
```

---

## 5. Summary Takeaways

- `tee` duplicates a data stream to both standard output and files.
- Use `tee -a` to append rather than overwrite.
- Use `echo "..." | sudo tee -a <file>` to write to root-protected system files.
- Use `|& tee <log>` to capture both standard output and standard error into the log.
