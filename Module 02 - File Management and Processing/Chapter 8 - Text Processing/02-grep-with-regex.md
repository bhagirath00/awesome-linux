# Regular Expressions with `grep`: BRE, ERE, and PCRE

## Overview

While basic string matching handles simple queries, real-world DevOps triage demands identifying complex, dynamic data patterns: IPv4/IPv6 addresses, UUIDs, git commit SHAs, ISO 8601 timestamps, email addresses, and HTTP status codes.

To accomplish this, `grep` implements three distinct Regular Expression engines:
1. **BRE (Basic Regular Expressions)** — Standard POSIX default (`grep`).
2. **ERE (Extended Regular Expressions)** — Modern POSIX standard (`grep -E` or `egrep`).
3. **PCRE (Perl-Compatible Regular Expressions)** — Advanced modern standard (`grep -P`).

---

## 1. Comparing the Three Regex Flavors

| Feature | BRE (`grep`) | ERE (`grep -E`) | PCRE (`grep -P`) |
| :--- | :--- | :--- | :--- |
| **One or more (`+`)** | Must escape: `\+` | Native: `+` | Native: `+` |
| **Zero or one (`?`)** | Must escape: `\?` | Native: `?` | Native: `?` |
| **Alternation (OR `\|`)** | Must escape: `\|` | Native: `\|` | Native: `\|` |
| **Grouping `()`** | Must escape: `\(\)` | Native: `()` | Native: `()` |
| **Interval bounds `{N,M}`**| Must escape: `\{N,M\}` | Native: `{N,M}` | Native: `{N,M}` |
| **Shorthand classes (`\d`, `\w`, `\s`)** | Not supported | Not supported | **Supported (`\d`, `\w`, `\s`)** |
| **Lookahead & Lookbehind** | Unsupported | Unsupported | **Supported (`(?<=...)`, `(?=...)`)** |

> [!TIP]
> **DevOps Standard**: Default to **`grep -E`** (Extended Regular Expressions) for everyday regexes to avoid messy backslash escaping. Switch to **`grep -P`** when you need shorthand classes (`\d`, `\s`) or zero-width lookaround assertions.

---

## 2. Core Metacharacters & Anchors

### Line & Word Anchors
- **`^` (Caret)**: Matches the **beginning of a line**.
  - `grep -E '^server' /etc/nginx/nginx.conf`
- **`$` (Dollar)**: Matches the **end of a line**.
  - `grep -E 'false$' /etc/settings.env`
- **`^$`**: Matches a completely **empty line**.
- **`\<` and `\>`** (or **`\b`** in ERE/PCRE): Matches a **word boundary**.
  - `grep -E '\bapp\b' log.txt` (matches `app`, but not `application` or `webapp`).

### Character Sets & Ranges
- **`.` (Dot)**: Matches any single character (except newline).
- **`[abc]`**: Matches any character in the set (`a`, `b`, or `c`).
- **`[^abc]`**: Inverted set; matches any character **except** `a`, `b`, or `c`.
- **`[0-9]`** or **`[a-zA-Z]`**: Character ranges.

### POSIX Character Classes
For maximum locale portability across scripts:
- `[:digit:]` : Equivalent to `[0-9]`
- `[:alpha:]` : Equivalent to `[a-zA-Z]`
- `[:alnum:]` : Alphanumeric `[0-9a-zA-Z]`
- `[:space:]` : Whitespace characters (space, tab, newline)

---

## 3. The DevOps Superpower: `grep -o` (Extract Matches Only)

By default, `grep` prints the **entire line** containing a match. In automation scripts, you rarely want the full line—you want to extract the **exact token** that matched (such as an IP address, UUID, or secret key).

Adding the **`-o` (`--only-matching`)** flag instructs `grep` to print **only the matching text segment**, one per line!

```bash
# Full line output without -o:
$ grep -E '192\.168\.[0-9]+\.[0-9]+' access.log
2026-10-14T09:00:01Z host-1 Client connected from 192.168.1.55 on port 443

# Exact token output with -o:
$ grep -E -o '192\.168\.[0-9]+\.[0-9]+' access.log
192.168.1.55
```

---

## 4. Production DevOps Regex Recipes

### A. Extract All IPv4 Addresses from an Access Log
```bash
grep -E -o '([0-9]{1,3}\.){3}[0-9]{1,3}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head -n 10
```
*Pipeline Breakdown*:
1. Extracts all raw IPv4 strings (`grep -E -o`).
2. Sorts them alphabetically (`sort`).
3. Counts unique occurrences (`uniq -c`).
4. Sorts numerically descending (`sort -nr`).
5. Displays the top 10 most frequent visitor IPs (`head -n 10`).

### B. Extract Full 64-Character Docker Container IDs
```bash
grep -E -o '[0-9a-f]{64}' /var/log/docker.log | head -n 5
```

### C. Match HTTP 4xx Client Errors and 5xx Server Errors
```bash
# Matches HTTP response codes starting with 4 or 5 followed by 2 digits:
grep -E '" [45][0-9]{2} ' /var/log/nginx/access.log
```

### D. Extract Values with Zero-Width Lookarounds (`grep -P -o`)
When extracting a configuration parameter value without including the key name itself:

```bash
# Input file contents:
# DB_HOST=postgres.production.internal
# DB_USER=appuser

# Extract only the hostname using PCRE positive lookbehind (?<=...):
$ grep -P -o '(?<=DB_HOST=).*' /etc/environment
postgres.production.internal
```

---

## 5. Modern High-Performance Alternative: `ripgrep` (`rg`)

On modern Linux environments, the Rust-based tool **`ripgrep` (`rg`)** has become the industry standard for developer workstations and CI runners:
- **10x to 15x faster** than GNU `grep` on multi-core processors.
- Automatically respects `.gitignore` rules (never wastes time scanning `node_modules` or `.git`).
- Uses PCRE2 regex syntax natively with syntax-highlighted output.

```bash
# Ripgrep searches the whole directory recursively by default:
rg "DATABASE_URL" --type yaml
```

---

## Summary

- Use `grep -E` (Extended Regular Expressions) to write clean regexes with `+`, `?`, `|`, and `()`.
- Use `grep -P` (Perl-Compatible Regular Expressions) for `\d`, `\s`, and lookaround assertions.
- Use `^` to anchor patterns to line starts and `$` to line ends.
- Master the `grep -o` flag to extract exact matched tokens (IPs, hashes, tokens) for downstream pipeline processing.
