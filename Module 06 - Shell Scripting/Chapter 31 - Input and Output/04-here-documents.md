# Chapter 31: Input and Output - Here Documents (Heredocs) and Herestrings

## 1. Introduction: What is a Here Document (Heredoc)?

A **Here Document** (commonly known as a **Heredoc**) is a multi-line input redirection syntax supported by Unix shells. 

Instead of chaining multiple `echo` or `printf` statements with escaped newlines and complicated quotes to generate configuration files, a Heredoc allows you to embed multi-line text blocks directly within your script, preserving whitespace, indentation, and newlines cleanly.

Heredocs are ubiquitous in DevOps automation:
- Generating Nginx, Dockerfile, or systemd configuration files on the fly.
- Executing multi-statement SQL queries against PostgreSQL or MySQL databases.
- Writing Cloud-Init bootstrapping scripts.

---

## 2. Basic Syntax and Delimiters

**Syntax:**
```bash
command << DELIMITER
Multi-line content
line 2
line 3
DELIMITER
```

The `DELIMITER` can be any alphanumeric token (conventionally uppercase: `EOF`, `CONFIG`, `YAML`, `SQL`). When the shell encounters `<< DELIMITER`, it streams all subsequent lines into the command's standard input until it encounters the delimiter token **on its own line with no surrounding whitespace**.

---

## 3. Quoted vs. Unquoted Delimiters (Variable Expansion)

The presence or absence of quotes around the delimiter token controls whether variables and command substitutions are evaluated:

### 1. Unquoted Delimiter (`<< EOF`): Expands Variables
Variables like `$PORT` and commands like `$(date)` are **dynamically evaluated**:

```bash
PORT=8080
SERVER_NAME="api.company.com"

cat << EOF > /etc/nginx/conf.d/api.conf
server {
    listen ${PORT};
    server_name ${SERVER_NAME};

    # Generated automatically on: $(date)
    location / {
        proxy_pass http://127.0.0.1:3000;
    }
}
EOF
```

### 2. Quoted Delimiter (`<< 'EOF'`): Pure Literal Text (No Expansion)
Wrapping the delimiter in single or double quotes (`'EOF'` or `"EOF"`) **suppresses all variable expansion**. The text is written exactly as typed, preserving literal `$` signs, backticks, and escape characters:

```bash
# Variables remain unexpanded (Literal text)
cat << 'EOF' > install-docker.sh
#!/bin/bash
CURRENT_USER=$(whoami)
echo "Installing Docker for user: ${CURRENT_USER}..."
EOF
```
The generated file will contain literal `${CURRENT_USER}`, not the username of the engineer running the parent script!

---

## 4. Stripping Leading Tabs (`<<- EOF`)

In clean shell scripts, code inside functions or conditional blocks should be indented. However, standard Heredocs treat indentation spaces as literal file content.

If you prefix the delimiter with a hyphen (**`<<- EOF`**), the shell automatically **strips all leading TAB characters** from the Heredoc content and the terminating delimiter:

```bash
generate_config() {
	# Notice: indented with real TABS (\t), not spaces
	cat <<- EOF > /tmp/app.conf
		[server]
		host = 127.0.0.1
		port = 8080
	EOF
}
```
The resulting `/tmp/app.conf` will have zero leading indentation!

---

## 5. What is a Herestring (`<<<`)?

While a Heredoc redirects multi-line text, a **Herestring** (`<<<`) provides an ultra-fast, clean way to pass a **single string or variable** directly to a command's standard input without spawning a slow `echo ... |` subshell pipeline:

```bash
# Slow / Wasteful Subshell Pipeline:
echo "192.168.1.100" | cut -d'.' -f1-3

# Fast, Idiomatic Bash Herestring (Zero subshells!):
cut -d'.' -f1-3 <<< "192.168.1.100"
```

```bash
# Feeding a variable into base64 decoding:
SECRET_HASH="ZXhhbXBsZQo="
base64 -d <<< "${SECRET_HASH}"
```

---

## 6. Summary Takeaways

- Heredocs (`<< EOF`) pass multi-line text blocks cleanly to commands without chained `echo` statements.
- Use unquoted `<< EOF` when you want variables (`${VAR}`) to expand.
- Use quoted `<< 'EOF'` when writing literal configuration files, Nginx blocks, or embedded bash scripts.
- Use `<<- EOF` to strip leading tab indentations.
- Use Herestrings (`command <<< "${STR}"`) instead of `echo ... | command` to eliminate subshell pipeline overhead.
