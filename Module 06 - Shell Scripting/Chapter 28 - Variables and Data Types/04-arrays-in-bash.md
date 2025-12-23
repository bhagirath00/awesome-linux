# Chapter 28: Variables and Data Types - Arrays in Bash

## 1. Introduction: Array Types in Bash

While standard POSIX `/bin/sh` does not support arrays, modern **GNU Bash** supports two powerful array types:

1. **Indexed Arrays**: Elements are indexed by zero-based integers (`0, 1, 2...`).
2. **Associative Arrays (HashMaps / Dictionaries)**: Elements are indexed by arbitrary string keys (supported in Bash version 4.0+).

Arrays allow DevOps scripts to cleanly manage lists of microservices, target IP addresses, deployment regions, and key-value configuration dictionaries without fragile string splitting hacks.

---

## 2. Indexed Arrays

### 1. Declaration and Initialization
```bash
# Explicit declaration
declare -a SERVERS

# Initialize with space-separated elements inside parentheses
SERVERS=("web-01" "web-02" "web-03" "db-primary")

# Append an element to the array
SERVERS+=("redis-cache")
```

### 2. Accessing Array Elements
```bash
# Access first element (index 0)
echo "${SERVERS[0]}"
# Output: web-01

# Access last element (negative indexing in modern Bash)
echo "${SERVERS[-1]}"
# Output: redis-cache

# Print ALL elements as separate words (${SERVERS[@]})
echo "${SERVERS[@]}"
# Output: web-01 web-02 web-03 db-primary redis-cache

# Get the total length (number of elements) using ${#ARRAY[@]}
echo "${#SERVERS[@]}"
# Output: 5

# Get list of all populated indices using ${!ARRAY[@]}
echo "${!SERVERS[@]}"
# Output: 0 1 2 3 4
```

### 3. Iterating Over an Indexed Array
The industry standard pattern for looping through array items:
```bash
for server in "${SERVERS[@]}"; do
    echo "Running health check on: ${server}"
done
```

### 4. Slicing Arrays
Extract a subset of elements: `${ARRAY[@]:offset:length}`:
```bash
# Extract 2 elements starting at index 1:
echo "${SERVERS[@]:1:2}"
# Output: web-02 web-03
```

---

## 3. Associative Arrays (HashMaps / Key-Value Pairs)

Associative arrays **must be explicitly declared** using `declare -A` before use:

```bash
# MANDATORY: Declare associative array
declare -A APP_PORTS

# Populate key-value pairs
APP_PORTS=(
    ["nginx"]="80"
    ["https"]="443"
    ["postgres"]="5432"
    ["redis"]="6379"
)

# Add or update an individual key
APP_PORTS["node_api"]="3000"
```

### Accessing Values and Keys:
```bash
# Lookup value by key
echo "Postgres port: ${APP_PORTS["postgres"]}"
# Output: Postgres port: 5432

# Print all values
echo "${APP_PORTS[@]}"

# Print all keys (${!ARRAY[@]})
echo "${!APP_PORTS[@]}"
# Output: nginx https postgres redis node_api
```

### Iterating Over Keys and Values:
```bash
for service in "${!APP_PORTS[@]}"; do
    port="${APP_PORTS[${service}]}"
    echo "Service: ${service} -> Listening Port: ${port}"
done
```

---

## 4. Deleting Elements from Arrays

Use the `unset` command to remove specific elements or the entire array:

```bash
# Remove specific element at index 2
unset "SERVERS[2]"

# Remove key from associative array
unset "APP_PORTS[redis]"

# Wipe entire array from memory
unset SERVERS
```

---

## 5. Converting Strings to Arrays with `readarray` (or `mapfile`)

In shell scripting, you frequently need to read lines from a file or command output directly into an array. 

The **`readarray`** (synonym `mapfile`) built-in reads lines into an array efficiently without spawning slow subshells:

```bash
# Read output of 'docker ps -q' directly into an array
readarray -t RUNNING_CONTAINERS < <(docker ps -q)

echo "Total running containers: ${#RUNNING_CONTAINERS[@]}"

for container_id in "${RUNNING_CONTAINERS[@]}"; do
    echo "Inspecting container: ${container_id}"
done
```
The `-t` flag strips trailing newlines from each line.

---

## 6. Summary Takeaways

- Declare indexed arrays with `arr=()` and associative arrays with `declare -A dict`.
- Use `"${arr[@]}"` to expand all elements and `"${#arr[@]}"` to get total length.
- Use `"${!arr[@]}"` to retrieve all keys or indices.
- Always iterate over `"${arr[@]}"` with double quotes.
- Use `readarray -t array < <(command)` to load lines directly into an array.
