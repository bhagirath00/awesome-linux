# Linux Command Structure & Chaining Operators

## The Universal Anatomy of a Linux Command

Almost all Linux commands follow a standardized syntax:

```text
command  [options / flags]  [arguments / operands]
   |             |                     |
   |             |                     +-- The target (file, directory, IP, string)
   |             +------------------------ Modifies command behavior
   +-------------------------------------- The binary or builtin name
```

### Example Breakdown:
```bash
ls -l -h /var/log
# command: ls
# options: -l (long format), -h (human readable file sizes)
# argument: /var/log (the target directory to list)
```

---

## 1. Short Options vs Long Options

Linux commands support two styles of options:

### Short Options (Single Dash `-`)
- Uses a single dash followed by a single letter: `-a`, `-l`, `-h`.
- **Can be combined (bundled) together**:
  ```bash
  # These two commands are 100% identical:
  ls -l -a -h
  ls -lah
  ```

### Long Options (Double Dash `--`)
- Uses two dashes followed by descriptive words: `--all`, `--human-readable`, `--recursive`.
- Cannot be bundled together:
  ```bash
  # Long option equivalents:
  ls --all --human-readable -l
  ```

### Options Taking Their Own Arguments
Some options require values:
```bash
# Short option with argument:
tar -f backup.tar.gz -c /var/www

# Long option with argument (using '=' or space):
git commit --message="feat: updated configs"
docker run --name=my-app nginx
```

---

## 2. Command Exit Statuses (`$?`)

Every single command executed in Linux returns an **Exit Code (Return Code)** between 0 and 255.
The shell stores this code in the special variable **`$?`**:

- **`0`**: **Success** (command finished with zero errors).
- **Non-zero (`1` to `255`)**: **Failure / Error**.

```bash
# Check existing directory:
ls /home
echo $?
# Output: 0 (Success)

# Check non-existent directory:
ls /fake_directory_that_does_not_exist
echo $?
# Output: 2 (Error)
```

### Standard Exit Code Meanings in DevOps:
| Code | Meaning | Example Scenario |
|---|---|---|
| **0** | Success | Container built successfully, tests passed |
| **1** | General catchall error | Script failed on syntax or logic error |
| **2** | Misuse of command | Missing required argument |
| **126** | Command invoked cannot execute | Missing execute permission (`chmod +x`) |
| **127** | Command not found | Binary not installed or missing from `$PATH` |
| **137** | Process killed via `SIGKILL` (128 + 9) | **Out-Of-Memory (OOM) Killer terminated the container** |

---

## 3. Command Chaining Operators

In CI/CD pipelines and Bash automation, you chain commands together using logical operators:

### 1. Logical AND (`&&`)
Executes the second command **only if the first command succeeds** (exit code 0):
```bash
# If tests pass, then deploy:
npm run test && docker build -t app:v1 . && docker push app:v1
```

### 2. Logical OR (`||`)
Executes the second command **only if the first command fails** (exit code non-zero):
```bash
# If ping fails, send an alert:
ping -c 1 8.8.8.8 || echo "CRITICAL: Network Gateway is Down!" | mail -s "Alert" admin@example.com
```

### 3. Semicolon (`;`)
Executes sequentially **regardless of whether prior commands succeed or fail**:
```bash
date ; hostname ; uname -r
```

### 4. Background Execution (`&`)
Launches the command as an asynchronous background job, immediately freeing the prompt:
```bash
python3 -m http.server 8000 &
# Returns job ID and PID: [1] 49214
```

### 5. The Pipe (`|`)
Connects the standard output (`stdout`) of the command on the left directly into the standard input (`stdin`) of the command on the right:
```bash
# Find all listening ports and extract unique port numbers
ss -tulpn | awk '{print $5}' | cut -d: -f2 | sort -u
```
