# Chapter 22: Process Management - Background and Foreground Processes (Job Control)

## 1. Introduction: Job Control Concepts

When you run a command in a Linux terminal, your shell spawns a new process that takes control of your terminal's standard input and standard output. Until that command finishes execution or exits, you cannot run another command in that shell session. This is known as a **foreground process**.

**Job Control** is a shell feature that allows you to manage multiple concurrent tasks from a single terminal by moving processes between the foreground, the background, or suspending them temporarily.

```
+-------------------------------------------------------------------------+
|                         Linux Job Control States                        |
+-------------------------------------------------------------------------+
|                                                                         |
|                          +-------------------+                          |
|                          | Foreground Process|                          |
|                          | (Controls TTY/kbd)|                          |
|                          +---------+---------+                          |
|                                    |                                    |
|                   Ctrl + Z         |     fg %1                          |
|             (Sends SIGTSTP)        |  (Resume in fg)                    |
|                                    v                                    |
|                          +-------------------+                          |
|                          |  Stopped Process  |                          |
|                          |    (Suspended)    |                          |
|                          +---------+---------+                          |
|                                    |                                    |
|                                    |  bg %1                             |
|                                    | (Sends SIGCONT)                    |
|                                    v                                    |
|                          +-------------------+                          |
|                          | Background Process|                          |
|                          | (Executes quietly)|                          |
|                          +-------------------+                          |
|                                                                         |
+-------------------------------------------------------------------------+
```

---

## 2. Launching in Background with Ampersand (`&`)

To run a time-consuming command in the background immediately, append an ampersand (`&`) to the end of the command line:

```bash
# Launch a database backup in the background
tar -czvf /backup/db_backup.tar.gz /var/lib/mysql &
```

```
[1] 24810
```

The shell responds with two key pieces of information:
- `[1]`: The **Job Number** (shell-specific identifier).
- `24810`: The system-wide **Process ID (PID)**.

Your shell prompt returns immediately, allowing you to continue typing commands while the backup runs concurrently in the background.

---

## 3. Controlling Jobs: `jobs`, `Ctrl+Z`, `bg`, and `fg`

### Step 1: Suspending a Foreground Process (`Ctrl + Z`)
If you started a long-running process in the foreground (e.g., `rsync` or a build script) and realize you need your terminal back:

```bash
rsync -avz large-dataset/ remote-backup:/data/
^Z
```

```
[1]+  Stopped                 rsync -avz large-dataset/ remote-backup:/data/
```
The kernel sends `SIGTSTP`, immediately pausing the program in memory and returning your shell prompt.

### Step 2: Listing Active Jobs (`jobs`)
```bash
jobs -l
```

```
[1]+ 24810 Stopped                 rsync -avz large-dataset/ remote-backup:/data/
[2]- 24905 Running                 sleep 600 &
```

### Step 3: Resuming in Background (`bg`)
To wake up the stopped process so it continues executing without blocking your terminal:
```bash
bg %1
```

```
[1]+ rsync -avz large-dataset/ remote-backup:/data/ &
```

### Step 4: Bringing Back to Foreground (`fg`)
To bring background job `%1` back into the foreground to view its real-time output or interact with it:
```bash
fg %1
```

---

## 4. Surviving Terminal Disconnection: `nohup` and `disown`

By default, when you close your SSH connection or terminal window, the shell sends a `SIGHUP` (Hangup) signal to **all child processes**, terminating them immediately.

### Method 1: `nohup` (No Hangup)
`nohup` intercepts and blocks the `SIGHUP` signal, redirecting standard output and error to `nohup.out`:

```bash
nohup ./long-migration-script.sh > /var/log/migration.log 2>&1 &
```

Even if your SSH session disconnects or your laptop runs out of battery, the script continues running on the remote server until completion!

### Method 2: `disown` (For already running jobs)
If you already started a process without `nohup`, you can detach it from the shell's job control table:

```bash
# Detach job %1 from current shell session
disown -h %1
```

---

## 5. Enterprise Terminal Multiplexers: `tmux` and `screen`

For production environments, terminal multiplexers like **`tmux`** are the industry standard:

```bash
# Start a new persistent tmux session
tmux new -s deployment

# Run any command inside tmux...
# Disconnect safely anytime: Press Ctrl+B, then D

# Reconnect to session from any computer over SSH:
tmux attach -t deployment
```

---

## 6. Job Control Cheat Sheet

| Command / Shortcut | Action |
| :--- | :--- |
| `<command> &` | Launch command directly in the background. |
| `Ctrl + Z` | Suspend current foreground process (`SIGTSTP`). |
| `jobs -l` | List all background/stopped jobs with their PIDs. |
| `bg %<job_id>` | Resume a stopped job in the background (`SIGCONT`). |
| `fg %<job_id>` | Bring background job into the foreground. |
| `kill %<job_id>` | Send termination signal to a job by job index. |
| `nohup <cmd> &` | Run command immune to shell hangups (`SIGHUP`). |
| `disown -h %<id>` | Protect an active job from terminal closure. |

---

## 7. Summary Takeaways

- Job numbers (`%1`) are local to the current shell; PIDs (`24810`) are system-wide.
- Use `Ctrl + Z` followed by `bg` to push accidentally long-running foreground tasks into the background.
- Standard background processes die on SSH logout unless protected by `nohup`, `disown`, or executed inside `tmux`.
