# Chapter 28: Variables and Data Types - Variable Scoping (Global vs. Local)

## 1. Introduction: Scope in Bash

Variable **scope** defines the context in which a variable is accessible and valid within a script. 

By default in Bash, **all variables are GLOBAL**. 

When a variable is assigned inside a function, loop, or conditional block without explicit scoping keywords, it pollutes the global namespace. If a function modifies a variable name that happens to be used elsewhere in the script, it can silently overwrite that variable's value, creating subtle, hard-to-debug runtime errors.

---

## 2. Global vs. Local Variables in Functions

To confine a variable's lifespan strictly to the function in which it is declared, use the **`local`** built-in keyword:

```bash
#!/usr/bin/env bash

# Global variable
APP_STATUS="HEALTHY"

inspect_service() {
    # Local variable: exists strictly inside this function
    local temp_log="/tmp/audit.log"
    local APP_STATUS="WARNING"

    echo "[Inside Function]  APP_STATUS: ${APP_STATUS}"
    echo "[Inside Function]  temp_log  : ${temp_log}"
}

echo "[Before Function] APP_STATUS: ${APP_STATUS}"

inspect_service

echo "[After Function]  APP_STATUS: ${APP_STATUS}"
echo "[After Function]  temp_log  : '${temp_log:-EMPTY}'"
```

### Execution Output:
```
[Before Function] APP_STATUS: HEALTHY
[Inside Function]  APP_STATUS: WARNING
[Inside Function]  temp_log  : /tmp/audit.log
[After Function]  APP_STATUS: HEALTHY
[After Function]  temp_log  : 'EMPTY'
```

Notice that the function's internal assignment did not overwrite the global `APP_STATUS`, and the `temp_log` variable was automatically destroyed the instant the function returned!

> [!TIP]
> **Best Practice Rule**: **Always** declare function variables with `local` (e.g., `local file="$1"`, `local count=0`) unless you explicitly intend to modify a global configuration state.

---

## 3. Subshell Scoping: The Pipe Trap

One of the most notorious gotchas in Bash scripting involves subshells created by **pipes (`|`)**.

When you pipe the output of a command into a `while` loop, **Bash executes the right-hand side of the pipe inside a separate subshell process!** 

Any variable modified inside that loop exists only inside that child subshell and is lost the moment the loop finishes:

### The Buggy Pattern:
```bash
#!/usr/bin/env bash
COUNT=0

# The pipe '|' creates a child subshell for the while loop:
cat /etc/hosts | while read -r line; do
    COUNT=$((COUNT + 1))
done

# The global COUNT remains 0!
echo "Total lines counted: ${COUNT}"
```
```
Total lines counted: 0
```

### The Fix 1: Process Substitution (`< <(...)`)
Avoid piping into the loop. Instead, feed the loop using **Process Substitution**, keeping the loop in the parent shell:

```bash
COUNT=0

while read -r line; do
    COUNT=$((COUNT + 1))
done < <(cat /etc/hosts)

echo "Total lines counted: ${COUNT}"
```
```
Total lines counted: 12
```

### The Fix 2: Bash `shopt -s lastpipe`
In modern Bash scripts where job control is disabled (standard in non-interactive scripts), turn on `lastpipe` to force Bash to execute the last command of a pipeline in the foreground parent shell:

```bash
shopt -s lastpipe
COUNT=0

cat /etc/hosts | while read -r line; do
    COUNT=$((COUNT + 1))
done

echo "Total lines counted: ${COUNT}"
# Output: 12 (Succeeds!)
```

---

## 4. Scope Hierarchy Summary

```
Process Tree & Variable Visibility:

[ Parent Shell / Script ]
  ├── Global Variables (Visible to whole script and child functions)
  │
  ├── [ Function Call ]
  │     ├── Local Variables (Visible ONLY within this function & functions it calls)
  │     └── Global Overrides (Persists globally if NOT marked 'local'!)
  │
  └── [ Subshell / Pipeline (cmd | while) ]
        └── Isolated Subshell Memory (All variable changes discarded on exit!)
```

---

## 5. Summary Takeaways

- Variables in Bash are global by default unless explicitly scoped.
- Always use `local` inside functions to prevent accidental variable collisions.
- Pipes (`|`) spawn subshells; variable mutations inside a piped `while` loop do not persist to the parent script.
- Use process substitution `< <(command)` or `shopt -s lastpipe` to preserve loop state.
