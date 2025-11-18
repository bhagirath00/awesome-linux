# The `uniq` Command in Linux: Deduplication & Frequency Analysis

## Overview

The `uniq` utility detects, filters, and reports adjacent repeated lines from standard input or text files.

In Unix pipeline philosophy, `uniq` is one of the most famous and widely combined tools. Because `uniq` provides **frequency counting (`-c`)**, it forms the core of log analysis pipelines: counting unique visitors, measuring HTTP error rates, detecting brute-force authentication attempts, and sanitizing configuration files.

---

## 1. The Cardinal Rule of `uniq`: Adjacent Lines Only!

The single most common mistake made with `uniq` is assuming it searches the entire file globally for duplicates:

```text
Input file (Unsorted):
apple
banana
apple      <-- NOT adjacent!
apple
```

If you run `uniq` directly on this unsorted file:
```bash
$ uniq file.txt
apple
banana
apple      <-- apple still appears twice!
```

### Why?
`uniq` only compares line `N` against line `N-1`. It does not maintain a hash map of all previously seen lines.

> [!IMPORTANT]
> **The Golden Rule**: For global deduplication and frequency analysis, **always sort before piping into `uniq`**:
> ```bash
> sort file.txt | uniq
> ```

---

## 2. Command Syntax and Essential Flags

```bash
uniq [OPTIONS] [INPUT [OUTPUT]]
```

### Essential Flags Reference

| Flag | Long Option | Description | DevOps Practical Use Case |
| :--- | :--- | :--- | :--- |
| **`-c`** | `--count` | **Prefix lines by the number of occurrences**. | **The foundational flag for frequency counting and ranking.** |
| **`-d`** | `--repeated` | Prints **only duplicate lines** (one representative line per duplicate group). | Finding duplicate IP addresses, duplicate keys, or duplicate hostnames. |
| **`-D`** | `--all-repeated` | Prints **all duplicate lines** (including each duplicate instance). | Inspecting all lines involved in a collision for comparison. |
| **`-u`** | `--unique` | Prints **only unique lines** (completely drops any line that had duplicates). | Isolating single-occurrence anomalies. |
| **`-i`** | `--ignore-case` | Ignores differences in case when comparing. | Case-insensitive deduplication of email addresses or domains. |
| **`-f N`** | `--skip-fields=N` | Avoids comparing the first `N` whitespace-separated fields. | Comparing log messages while ignoring the leading timestamp columns. |
| **`-s N`** | `--skip-chars=N` | Avoids comparing the first `N` characters. | Skipping fixed-width date prefixes. |
| **`-w N`** | `--check-chars=N` | Compares at most `N` characters in each line. | Grouping entries by common prefix. |

---

## 3. The Classic DevOps Frequency Pipeline

The most famous idiom in command-line Site Reliability Engineering is:

```bash
sort | uniq -c | sort -nr
```

```text
Step 1: sort          ──> Groups identical lines together into adjacent runs.
Step 2: uniq -c       ──> Counts occurrences of each line, outputting: "   COUNT LINE".
Step 3: sort -nr      ──> Numerically sorts descending, placing the most frequent item at the very top.
Step 4: head -n 10    ──> Displays the top 10 most frequent items.
```

---

## 4. Real-World DevOps Recipes

### A. Detect SSH Brute-Force Attackers
Analyze `/var/log/auth.log` to identify remote IP addresses repeatedly failing SSH authentication:

```bash
grep "Failed password for" /var/log/auth.log \
  | grep -E -o 'from [0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' \
  | awk '{print $2}' \
  | sort \
  | uniq -c \
  | sort -nr \
  | head -n 10
```
Output:
```text
   1482 198.51.100.42   <-- Attacker attempting brute-force dictionary attack!
    892 203.0.113.19
    312 192.0.2.88
```
*(You can immediately feed the top offending IP into `iptables` or `ufw` to block the subnet).*

### B. Find Duplicate Keys in Configuration Files
Ensure environment variable files (`.env`) or hosts files don't contain conflicting duplicate entries:

```bash
# Print only lines that appear more than once in .env:
cut -d'=' -f1 .env | sort | uniq -d
```

### C. Find Top 404 Not Found URLs in Web Logs
Identify broken links or malicious URL scanners probing your Nginx server:

```bash
awk '$9 == 404 { print $7 }' /var/log/nginx/access.log \
  | sort \
  | uniq -c \
  | sort -nr \
  | head -n 5
```
Output:
```text
    420 /wp-login.php       <-- Automated bot scanning for WordPress vulnerabilities
    310 /.env
    184 /phpmyadmin
     42 /assets/style.css   <-- Actual broken asset link!
```

### D. Counting Total Unique Items
Calculate the exact number of distinct client IP addresses visiting your cluster:

```bash
awk '{print $1}' access.log | sort -u | wc -l
```
*(Notice: `sort -u` performs sorting and deduplication in a single step!).*

---

## Summary

- `uniq` only detects **adjacent** duplicates; always pipe `sort` into `uniq`.
- Use `uniq -c` to count frequencies of each unique line.
- The standard diagnostic pipeline is `sort | uniq -c | sort -nr | head -n 10`.
- Use `uniq -d` to isolate duplicate collisions and `uniq -u` to extract strictly unique lines.
- Use `sort -u` as a shorthand when you only need unique lines without counts.
