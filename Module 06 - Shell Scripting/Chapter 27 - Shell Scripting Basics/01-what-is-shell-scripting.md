# Chapter 27: Shell Scripting Basics - What is Shell Scripting?

## 1. Introduction: The Automation Engine of Linux

A **shell script** is an executable text file containing a sequence of commands that the Linux command interpreter (the shell) executes in order. 

Whenever you type commands interactively into a terminal—such as `mkdir`, `curl`, `grep`, `systemctl`, or `docker`—the shell interprets and executes them immediately. Shell scripting allows you to bundle hundreds of these operational commands into a reproducible, version-controlled, and automated program capable of handling conditional logic, loops, error detection, and inter-process communication.

In DevOps, cloud engineering, and site reliability engineering (SRE), shell scripting is the primary glue that connects disparate operating system utilities, provisions infrastructure, orchestrates CI/CD deployment pipelines, runs container entrypoints, and automates server maintenance.

```
+--------------------------------------------------------------------------+
|                       Shell Scripting in the DevOps Stack                |
+--------------------------------------------------------------------------+
|  Continuous Integration & CD       Docker & Container Runtimes           |
|  - GitHub Actions / GitLab CI      - entrypoint.sh scripts               |
|  - Automated test runners          - Container initialization & secrets  |
|  - Artifact builds and packaging   - Health check probes                 |
|                                                                          |
|  Cloud & Infrastructure            System Administration                 |
|  - Cloud-Init bootstrapping        - Nightly backups & log rotation      |
|  - Terraform local-exec hooks      - Monitoring probes & health alerts   |
|  - Golden AMI provisioning         - Service failure recovery            |
+--------------------------------------------------------------------------+
```

---

## 2. Interactive Shell vs. Non-Interactive Scripting

There is a fundamental behavioral difference between typing commands manually into an interactive terminal and running an automated shell script:

| Characteristic | Interactive Shell | Non-Interactive Script |
| :--- | :--- | :--- |
| **Input Source** | Keyboard (`stdin` from user) | Text file read line-by-line |
| **Error Handling** | User visually observes error and types fix | Script continues blindly unless error handling (`set -e`) is configured |
| **Environment** | Loads `~/.bashrc`, aliases, prompts | Minimal environment; does **not** load user aliases |
| **Prompts & Prompts** | Prompts for confirmation (`rm -i`) | Runs silently; will stall/hang if interactive prompt is reached |
| **Lifecycle** | Remains open until user exits | Spawns a subshell, executes commands, and exits with a status code |

---

## 3. When to Use Shell Scripting (and When Not To)

While shell scripting is exceptionally powerful, knowing when **not** to use it is an essential architectural skill:

```
+--------------------------------------------------------------------------+
|                        Language Selection Guide                          |
+--------------------------------------------------------------------------+
|                                                                          |
|  Task: Orchestrating CLI tools, pipes, file redirections, systemctl?    |
|  ======> USE BASH / SHELL SCRIPTING                                      |
|                                                                          |
|  Task: Complex nested data structures, heavy REST API parsing, math?     |
|  ======> USE PYTHON / GO                                                 |
|                                                                          |
|  Task: High-performance concurrency, compiled systems software?          |
|  ======> USE GO / RUST                                                   |
|                                                                          |
+--------------------------------------------------------------------------+
```

### Ideal Shell Scripting Use Cases:
- Gluing Linux system binaries together with pipes (`|`) and redirections.
- Automating repetitive OS administration tasks (backups, user creation, log audits).
- Container startup initialization scripts (`entrypoint.sh`).
- Lightweight CI/CD build and deploy steps.

### When to Transition to Python or Go:
- The script exceeds 300–400 lines of complex logic.
- You need to parse deeply nested JSON or YAML payloads (where `jq` becomes unmaintainable).
- You require multi-threading, asynchronous network I/O, or high mathematical precision.
- Cross-platform Windows/macOS/Linux binary portability is required without shell dependencies.

---

## 4. Shell Script Execution Lifecycle

```
1. Script Written on Disk (/opt/scripts/deploy.sh)
                     |
                     v
2. Kernel reads Shebang: #!/bin/bash
                     |
                     v
3. Kernel spawns binary: /bin/bash /opt/scripts/deploy.sh
                     |
                     v
4. Subshell parses lines sequentially (Syntax validation -> Expansion -> Execution)
                     |
                     v
5. Last command finishes -> Script returns Exit Status Code ($?) to parent process
```

---

## 5. Summary Takeaways

- Shell scripts automate sequences of Linux commands in a repeatable, version-controlled manner.
- Scripts execute in a non-interactive subshell without aliases or user terminal prompts.
- Shell scripting is the universal glue for CI/CD, Docker entrypoints, and Cloud-Init.
- Keep shell scripts focused on system orchestration; transition to Python or Go when data structures or complexity become unwieldy.
