# Chapter 34: Advanced Shell Scripting - Arithmetic Operations in Bash

## 1. Introduction: Integer Math in Shell Scripting

By default, Bash treats all variables as character strings. When performing calculations (e.g., computing percentage memory utilization, incrementing retry counters, or calculating elapsed time), the shell must explicitly evaluate expressions arithmetically.

> [!IMPORTANT]
> **Bash ONLY supports 64-bit integer arithmetic!** 
> 
> Bash does **not** support native floating-point math (decimals like `3.14`). For decimal calculations, you must pipe expressions into external calculators like **`bc`** or **`awk`**.

---

## 2. The Arithmetic Expansion Syntax: `$(( ... ))`

The modern, universally recommended syntax for integer math in Bash is **`$(( expression ))`**:

```bash
A=15
B=4

# Addition
SUM=$(( A + B ))       # 19

# Subtraction
DIFF=$(( A - B ))      # 11

# Multiplication
PROD=$(( A * B ))      # 60

# Integer Division (Truncates decimal; 15 / 4 = 3)
DIV=$(( A / B ))       # 3

# Modulo (Remainder)
MOD=$(( A % B ))       # 3

# Exponentiation (15^2)
EXP=$(( A ** 2 ))      # 225
```

### Notice:
Inside `$(( ... ))`, you **do not need the `$` prefix** on variable names (`A + B` works identically to `$A + $B`).

---

## 3. Increment and Decrement Operators

Bash supports standard C-style pre- and post-increment operators:

```bash
COUNT=0

# Post-increment (evaluates then increments)
COUNT=$(( COUNT + 1 ))

# Shorthand:
(( COUNT++ ))
(( COUNT-- ))
(( COUNT += 5 ))
```

### In While-Loops:
```bash
RETRY=0
MAX_RETRIES=3

while (( RETRY < MAX_RETRIES )); do
    echo "Attempt #$(( RETRY + 1 ))..."
    (( RETRY++ ))
done
```

---

## 4. Arithmetic Evaluation in Conditions: `(( ... ))`

Instead of clumsy string-based test comparisons like `if [ "$A" -gt 10 ]`, you can use arithmetic evaluation brackets **`(( ... ))`** directly:

```bash
AGE=25

if (( AGE >= 18 && AGE <= 65 )); then
    echo "Valid working age."
fi

# Bitwise operations
FLAGS=7
if (( (FLAGS & 4) != 0 )); then
    echo "Bit 4 is set."
fi
```

Inside `(( ... ))`, you can use standard mathematical symbols: `>`, `<`, `>=`, `<=`, `==`, `!=`, `&&`, `||`.

---

## 5. Floating-Point Arithmetic with `bc` and `awk`

When you need fractional or floating-point precision (e.g., CPU percentages or load ratios):

### Method 1: Using `bc` (Basic Calculator)
Set the `scale` variable to dictate the number of decimal digits:

```bash
# Calculate disk percentage with 2 decimal places:
USED=4812
TOTAL=9812

PERCENT=$(echo "scale=2; (${USED} / ${TOTAL}) * 100" | bc)
echo "Disk Usage: ${PERCENT}%"
# Output: Disk Usage: 49.04%
```

### Method 2: Using `awk` (Fast & Built-in)
`awk` is universally installed and handles floating-point math without `bc`:

```bash
PERCENT=$(awk "BEGIN { printf \"%.2f\", (${USED} / ${TOTAL}) * 100 }")
echo "Disk Usage: ${PERCENT}%"
```

---

## 6. Summary Takeaways

- Bash natively supports 64-bit signed **integer math** only.
- Always use `$(( ... ))` for calculations and `(( ... ))` for numeric conditional tests.
- Variables do not require the `$` prefix inside arithmetic evaluation blocks.
- Use `(( count++ ))` for clean loop increments.
- Use `bc` or `awk` whenever floating-point decimal precision is required.
