# Chapter 28: Variables and Data Types - Environment Variables

## 1. Shell Variables vs. Environment Variables

In Linux, there is a crucial architectural distinction between a standard **Shell Variable** and an **Environment Variable**:

- **Shell Variable (Local)**: Exists strictly within the memory of the current running shell process. Child processes spawned by the shell (scripts, binaries, subshells) do **not** inherit it.
- **Environment Variable (Exported)**: Placed into the process's **environment block** (`envp[]`). When the process spawns child commands, the kernel copies all exported environment variables into the child's environment space.

```
+--------------------------------------------------------------------------+
|                       Environment Variable Inheritance                   |
+--------------------------------------------------------------------------+
|  Parent Shell (PID: 1000)                                                |
|  LOCAL_VAR="secret"        (Local Shell Variable)                        |
|  export API_KEY="xyz123"   (Exported Environment Variable)               |
|                                |                                         |
|                                v fork() & execve("./child-script.sh")    |
|  Child Subshell (PID: 1001)                                              |
|  +--------------------------------------------------------------------+  |
|  | $LOCAL_VAR  ====> EMPTY / UNSET (Not inherited!)                   |  |
|  | $API_KEY    ====> "xyz123"      (Inherited successfully!)          |  |
|  +--------------------------------------------------------------------+  |
+--------------------------------------------------------------------------+
```

---

## 2. Exporting Variables with `export`

To promote a local shell variable to an environment variable:

```bash
# Method 1: Declare and export in two steps
DATABASE_URL="postgres://user:pass@db.internal:5432/app"
export DATABASE_URL

# Method 2: Declare and export in a single step (Preferred)
export DATABASE_URL="postgres://user:pass@db.internal:5432/app"
```

### Passing One-Off Environment Variables to a Command:
You can pass an environment variable temporarily to a single command execution without polluting the parent shell's environment:

```bash
PORT=9000 NODE_ENV=production node server.js
```
The `PORT` and `NODE_ENV` variables exist *only* for the lifetime of `node server.js`; your terminal shell remains clean.

---

## 3. Standard Linux Environment Variables

Every Linux environment provides pre-configured system environment variables:

| Variable | Description | Example Value |
| :--- | :--- | :--- |
| **`PATH`** | Colon-delimited list of directories searched for executable binaries. | `/usr/local/bin:/usr/bin:/bin` |
| **`HOME`** | Absolute path to current user's home directory. | `/home/devops` |
| **`USER`** | Username of currently logged-in user. | `devops` |
| **`SHELL`** | Path to user's default login shell. | `/bin/bash` |
| **`HOSTNAME`** | Hostname of the current server machine. | `web-prod-01` |
| **`PWD`** | Current working directory (matches `pwd`). | `/opt/deployments` |
| **`LANG`** | System locale and character encoding. | `en_US.UTF-8` |

---

## 4. Inspecting Environment Variables

```bash
# Print all currently exported environment variables
printenv
# Or:
env

# Query a specific environment variable
printenv PATH
# Or:
echo "${PATH}"
```

---

## 5. Configuration Persistence Files

Where environment variables should be defined depends on who needs them and when:

```
+--------------------------------------------------------------------------+
|                  Environment Variable File Hierarchy                     |
+--------------------------------------------------------------------------+
|  Global (All Users)                                                      |
|  - /etc/environment         : Static KEY=VAL pairs (No bash logic)       |
|  - /etc/profile             : Global login shell initialization          |
|  - /etc/profile.d/*.sh      : Modular global startup scripts (Best practice)|
|                                                                          |
|  Per-User                                                                |
|  - ~/.bash_profile / ~/.profile : User login shells (SSH logins)         |
|  - ~/.bashrc                : Interactive non-login subshells (Terminals)|
+--------------------------------------------------------------------------+
```

### Adding Global Variables via `/etc/profile.d/`:
```bash
sudo tee /etc/profile.d/custom-devops.sh << 'EOF'
export JAVA_HOME="/usr/lib/jvm/java-17-openjdk-amd64"
export PATH="${JAVA_HOME}/bin:${PATH}"
EOF

# Grant execution
sudo chmod +x /etc/profile.d/custom-devops.sh
```

---

## 6. Summary Takeaways

- Local shell variables are private; environment variables are exported to child processes using `export`.
- Pass ephemeral variables directly before a command: `KEY=VAL command`.
- Use `printenv` or `env` to audit active environment variables.
- Place shared system-wide exports cleanly in `/etc/profile.d/*.sh`.
