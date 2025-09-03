# Chapter 30: Functions - Passing and Parsing Function Arguments

## 1. Introduction: Function Argument Mechanics

In Bash, functions do not declare formal parameter lists in their signature (e.g., you cannot write `deploy(env, port, tag)`). 

Instead, functions receive parameters **positionally**, exactly like a shell script receives command-line arguments. Inside a function body:
- **`$1`** is the first argument passed to the function.
- **`$2`** is the second argument passed to the function.
- **`$#`** is the count of arguments passed to this specific function.
- **`$@`** represents all arguments passed to this function.

```
+--------------------------------------------------------------------------+
|                       Function Argument Scoping                          |
+--------------------------------------------------------------------------+
|  Script Scope                                                            |
|  $1 = "script_arg1" (Passed from terminal CLI)                           |
|       |                                                                  |
|       v Calls: provision_server "us-east-1" "t3.medium"                  |
|  +--------------------------------------------------------------------+  |
|  | Function Scope (provision_server)                                  |  |
|  | $1 = "us-east-1"  (Function's first argument - Overrides script $1)|  |
|  | $2 = "t3.medium"  (Function's second argument)                     |  |
|  | $# = 2            (Function's argument count)                      |  |
|  +--------------------------------------------------------------------+  |
|       |                                                                  |
|       v Function returns                                                 |
|  $1 = "script_arg1" (Script scope $1 is restored!)                       |
+--------------------------------------------------------------------------+
```

> [!NOTE]
> Inside a function, `$1`, `$2`, and `$#` reflect the arguments passed to the **function itself**, temporarily masking the script's outer positional parameters. The `$0` variable, however, remains unchanged and still references the **overall script name**.

---

## 2. Best Practice: Immediately Assign Arguments to Local Named Variables

Accessing raw `$1` and `$2` deep inside a 50-line function makes code obscure, unmaintainable, and prone to indexing errors. 

**Always assign positional parameters to descriptive `local` variables at the very top of the function:**

```bash
provision_database() {
    # Validate required argument count
    if [ "$#" -lt 2 ]; then
        echo "[ERROR] provision_database requires at least 2 arguments: <name> <engine>" >&2
        return 1
    fi

    # Assign to named local variables with optional defaults
    local db_name="$1"
    local db_engine="$2"
    local db_port="${3:-5432}"   # Default port to 5432 if $3 is omitted

    echo "Provisioning ${db_engine} database '${db_name}' on port ${db_port}..."
}

# Invocation:
provision_database "production_users" "postgresql"
provision_database "cache_store" "redis" 6379
```

---

## 3. Shifting Arguments with `shift`

The `shift` built-in command slides all positional parameters down by one index:
- `$1` is discarded.
- `$2` becomes the new `$1`.
- `$3` becomes the new `$2`, and `$#` decreases by 1.

This is extremely powerful when a function accepts one or two mandatory parameters followed by an arbitrary number of items:

```bash
backup_directories() {
    local backup_destination="$1"
    shift # Discard destination; now "$@" contains strictly the directories to back up

    echo "Destination archive directory: ${backup_destination}"

    for target_dir in "$@"; do
        echo "--> Archiving: ${target_dir} into ${backup_destination}"
    done
}

backup_directories "/opt/backups" "/etc/nginx" "/var/www" "/home/admin"
```

### Execution Output:
```
Destination archive directory: /opt/backups
--> Archiving: /etc/nginx into /opt/backups
--> Archiving: /var/www into /opt/backups
--> Archiving: /home/admin into /opt/backups
```

---

## 4. Passing Arrays to Functions

Because Bash flattens arguments into a single stream, passing an array to a function requires passing the expanded elements:

```bash
send_notifications() {
    local recipients=("$@")

    for user in "${recipients[@]}"; do
        echo "Sending alert email to: ${user}"
    done
}

USER_LIST=("alice@company.com" "bob@company.com" "carol@company.com")
send_notifications "${USER_LIST[@]}"
```

---

## 5. Summary Takeaways

- Functions receive arguments via `$1`, `$2`, `$@`, and `$#`.
- Inside functions, positional parameters refer to function arguments, not script arguments.
- Always map positional arguments to descriptive `local` variables at the top of the function.
- Use `shift` to cleanly separate mandatory flags from variable lists of items.
- Pass arrays by expanding them with `"${arr[@]}"`.
