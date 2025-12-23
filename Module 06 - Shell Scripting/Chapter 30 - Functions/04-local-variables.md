# Chapter 30: Functions - Local Variables and Function Isolation

## 1. The Danger of Implicit Global Variables

In shell scripting, variable pollution is one of the leading causes of production bugs. Consider this scenario:

```bash
#!/usr/bin/env bash

# Global iterator variable
i=100

process_cluster() {
    # BUG: 'i' is assigned without 'local', silently corrupting the global 'i'!
    for i in 1 2 3; do
        echo "Processing node ${i}..."
    done
}

echo "Initial value of i: ${i}"
process_cluster
echo "Value of i after function call: ${i}"  # Output is now 3, NOT 100!
```

Because variables in Bash default to global scope, the loop inside `process_cluster` silently mutated the global variable `i` from `100` to `3`. If outer code relied on `i=100`, subsequent logic breaks catastrophically.

---

## 2. Enforcing Isolation with `local`

The `local` keyword confines a variable strictly to the function in which it is declared.

```bash
process_cluster() {
    # Confines 'i' strictly to this function:
    local i
    for i in 1 2 3; do
        echo "Processing node ${i}..."
    done
}
```

Now, the outer variable `i` remains untouched at `100`.

---

## 3. Dynamic Scoping in Bash

Bash uses **Dynamic Scoping** (not Lexical Scoping like Python or C). 

Under dynamic scoping, a `local` variable declared in Function A is **visible to any child function invoked by Function A**, but is invisible to the parent script:

```
[ Main Script ]
  ├── $parent_var (Global)
  │
  └── Calls: func_A()
        ├── local $a_var = "alpha"
        │
        └── Calls: func_B()
              └── $a_var IS VISIBLE HERE! (Because func_B was called from func_A)
```

### Demonstration:
```bash
#!/usr/bin/env bash

func_b() {
    echo "Inside func_b, my_var is: '${my_var:-UNSET}'"
}

func_a() {
    local my_var="Declared in func_a"
    func_b
}

func_b  # Directly called from script
func_a  # Called through func_a
```

```
Inside func_b, my_var is: 'UNSET'
Inside func_b, my_var is: 'Declared in func_a'
```

---

## 4. Best Practice Guidelines for Function Variables

1. **Always declare variables as `local`**:
   ```bash
   deploy() {
       local target_env="$1"
       local port="${2:-8080}"
       local -a active_nodes=()
   }
   ```
2. **Declare and assign in separate lines when capturing command output**:
   Under `set -e`, combining `local result=$(failing_command)` will **mask the command's exit code** because the `local` declaration itself returns 0!
   
   ```bash
   # WRONG (Masks error under set -e!):
   local output=$(cat /nonexistent/file)
   
   # CORRECT:
   local output
   output=$(cat /nonexistent/file)
   ```

---

## 5. Summary Takeaways

- Variables in Bash are global unless explicitly declared with `local`.
- Always declare loop counters and temporary variables with `local`.
- Bash functions use dynamic scoping: child functions inherit local variables from parent callers.
- Separate `local var` declaration from `var=$(command)` assignments to prevent masking exit codes under `set -e`.
