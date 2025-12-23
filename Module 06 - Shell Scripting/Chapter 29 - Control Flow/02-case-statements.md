# Chapter 29: Control Flow - Case Conditional Statements

## 1. Introduction: What is a Case Statement?

When a script needs to branch between three or more possible values of a variable (such as CLI subcommands, operating system names, or deployment environments), chaining multiple `if...elif...elif...else` blocks becomes verbose and difficult to maintain.

The Bash **`case`** statement provides clean, elegant multi-way branching based on **glob pattern matching**. It is the standard idiom in Linux for writing CLI wrappers, initialization scripts, and system dispatchers.

```
+--------------------------------------------------------------------------+
|                        Case Statement Architecture                       |
+--------------------------------------------------------------------------+
|  Input Variable: "$ACTION"                                               |
|                                                                          |
|  case "$ACTION" in                                                       |
|      start)    =====> Starts service daemon                              |
|      stop)     =====> Sends SIGTERM signal                               |
|      restart)  =====> Calls stop then start                              |
|      status)   =====> Queries systemctl is-active                        |
|      *)        =====> Default catch-all (Prints usage & exits)           |
|  esac                                                                    |
+--------------------------------------------------------------------------+
```

---

## 2. Syntax Rules and Terminators

```bash
case "$EXPRESSION" in
    pattern1)
        # Commands executed if pattern1 matches
        ;;
    pattern2|pattern3)
        # Vertical pipe '|' acts as OR (matches pattern2 OR pattern3)
        ;;
    *)
        # Default fallback match (wildcard)
        ;;
esac
```

### Essential Syntax Elements:
- **`in`**: Initiates the pattern matching block.
- **`pattern)`**: Defines the match criteria, terminated by a closing parenthesis.
- **`;;`**: **Break terminator**. Stops further pattern evaluation and exits the `case` block immediately (equivalent to `break` in C/Java/Python).
- **`esac`**: Terminates the case block (`case` spelled backwards).

---

## 3. Pattern Matching Capabilities (Globs)

Unlike `switch` statements in other languages which only match static literal constants, Bash `case` statements support powerful **shell glob wildcards**:

| Pattern Syntax | Matches | Example |
| :--- | :--- | :--- |
| `[yY]|[yY][eE][sS]` | Case-insensitive variations of "yes" or "y". | Prompts for confirmation. |
| `*.tar.gz|*.tgz` | Any string ending in `.tar.gz` or `.tgz`. | Archive extraction dispatcher. |
| `[0-9]` | Any single digit. | Single-digit input validation. |
| `[a-z]` | Any single lowercase letter. | Character classification. |
| `*` | **Wildcard (Catch-all)**. Always placed at the end. | Default error handler / usage menu. |

---

## 4. Production Script Example: Service Manager Dispatcher

```bash
#!/usr/bin/env bash
set -euo pipefail

SERVICE_NAME="payment-gateway"
ACTION="${1:-help}"

case "${ACTION}" in
    start)
        echo "Starting ${SERVICE_NAME} daemon..."
        sudo systemctl start "${SERVICE_NAME}"
        ;;

    stop)
        echo "Stopping ${SERVICE_NAME} daemon..."
        sudo systemctl stop "${SERVICE_NAME}"
        ;;

    restart|reload)
        echo "Restarting ${SERVICE_NAME}..."
        sudo systemctl restart "${SERVICE_NAME}"
        ;;

    status)
        if systemctl is-active --quiet "${SERVICE_NAME}"; then
            echo "${SERVICE_NAME} is RUNNING."
            exit 0
        else
            echo "${SERVICE_NAME} is STOPPED."
            exit 1
        fi
        ;;

    -h|--help|help)
        echo "Usage: ${0} {start|stop|restart|reload|status}"
        exit 0
        ;;

    *)
        echo "[ERROR] Unknown action '${ACTION}'!" >&2
        echo "Usage: ${0} {start|stop|restart|reload|status}" >&2
        exit 1
        ;;
esac
```

---

## 5. Summary Takeaways

- Use `case` instead of deeply nested `if-elif` chains for multi-way branching.
- Patterns end with `)` and blocks must terminate with `;;`.
- Use the vertical bar `|` for OR condition matching (e.g., `restart|reload`).
- Always include a wildcard default handler `*)` at the end to catch invalid input and display usage instructions.
