# Command-Line HTTP and API Testing: `curl` vs `wget`

## Overview

In modern cloud computing, microservices, and continuous delivery pipelines, **`curl`** and **`wget`** are the primary tools used for interacting with HTTP/HTTPS web endpoints, REST APIs, and downloading artifacts.

While both utilities download files over networks, their design philosophies are fundamentally different:
- **`curl`** (Client URL): A bidirectional, library-backed (`libcurl`) data transfer tool. It supports 25+ protocols (HTTP, HTTPS, FTP, SFTP, LDAP, gRPC, WebSocket), full HTTP method control (`POST`, `PUT`, `DELETE`), custom headers, authentication tokens, and millisecond-level latency profiling. It is the **industry standard for REST API testing and CI/CD pipelines**.
- **`wget`**: A dedicated, recursive file downloader. It excels at downloading large files over unstable connections with automatic retry, resuming interrupted transfers, and recursively mirroring entire documentation websites.

---

## 1. Comparing `curl` vs `wget`

| Feature | `curl` | `wget` |
| :--- | :--- | :--- |
| **Primary Focus** | REST API testing, data transfer, protocol debugging. | Downloading files and website mirroring. |
| **HTTP Methods** | Full support (`GET`, `POST`, `PUT`, `DELETE`, `PATCH`, `HEAD`). | Primarily `GET` and `POST`. |
| **Recursive Mirroring** | No (cannot crawl links). | **Yes (`wget -r`)**. |
| **Resumable Downloads** | Supports (`-C -`). | **Exceptional (`wget -c`)** with automatic retry loops. |
| **Bidirectional (Uploads)**| **Yes** (supports multipart form uploads, SFTP PUT). | No (primarily download-only). |
| **DevOps Usage** | **Kubernetes healthchecks, CI/CD webhooks, API triage.** | Downloading large ISOs, tarballs, and dataset archives. |

---

## 2. The `curl` Command: Essential Flags Reference

```bash
curl [OPTIONS] [URL]
```

| Flag | Long Option | Description | DevOps Practical Use Case |
| :--- | :--- | :--- | :--- |
| **`-I`** | `--head` | Fetches **HTTP headers only** (sends `HEAD` request). | Quick verification of server response codes and SSL. |
| **`-s`** | `--silent` | Disables progress meter and error messages. | Clean output for shell variable assignment. |
| **`-S`** | `--show-error` | When used with `-s`, displays errors only if the transfer fails. | Best practice for CI/CD scripting (`curl -sS`). |
| **`-L`** | `--location` | **Follows HTTP 301/302 redirects** automatically. | Essential when requesting shortlinks or SSL upgrades. |
| **`-k`** | `--insecure` | **Bypasses SSL/TLS certificate verification**. | Testing internal staging environments with self-signed certs. |
| **`-X METHOD`** | `--request` | Specifies HTTP method (`POST`, `PUT`, `DELETE`). | Calling REST API endpoints. |
| **`-H HEADER`** | `--header` | Adds custom HTTP header. | Passing `Authorization: Bearer <TOKEN>` or `Content-Type`. |
| **`-d DATA`** | `--data` | Sends HTTP POST data. Automatically sets `Content-Type: application/x-www-form-urlencoded`. | Submitting API payloads. |
| **`-o FILE`** | `--output` | Writes output to specified file path instead of stdout. | Saving downloaded artifacts. |
| **`-O`** | `--remote-name`| Writes output to a local file named after the remote file. | Downloading files from URLs. |
| **`--connect-timeout`**| *(none)* | Maximum time in seconds allowed for connection phase. | **Preventing automated scripts from hanging indefinitely.** |

---

## 3. High-Frequency DevOps Recipes with `curl`

### A. Testing API Endpoints with JSON Payloads
Submit a JSON POST request with an Authorization Bearer token:

```bash
curl -sS -X POST "https://api.example.com/v1/deploy" \
    -H "Authorization: Bearer secret-token-xyz" \
    -H "Content-Type: application/json" \
    -d '{"environment": "production", "version": "v2.4.0"}' \
    | jq .
```

### B. Extracting Clean HTTP Status Codes in Scripts
In health-check probes, you only care whether the server returns HTTP `200`:

```bash
STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://api.example.com/health)

if [ "$STATUS" -eq 200 ]; then
    echo "Healthcheck passed (HTTP $STATUS)."
else
    echo "CRITICAL: Healthcheck failed with HTTP $STATUS!" >&2
    exit 1
fi
```

### C. Profiling HTTP Network & TLS Latency (The SRE Holy Grail)
`curl` can break down the exact microsecond latency of every stage of an HTTP request:

```bash
curl -w @- -o /dev/null -s "https://api.github.com" << 'EOF'
  DNS Lookup Time:        %{time_namelookup} s
  TCP Connection Time:    %{time_connect} s
  TLS / SSL Handshake:    %{time_appconnect} s
  Pre-transfer Time:      %{time_pretransfer} s
  Time to First Byte:     %{time_starttransfer} s (TTFB)
  -----------------------------------------------
  Total Request Time:     %{time_total} s
EOF
```
Output:
```text
  DNS Lookup Time:        0.012 s
  TCP Connection Time:    0.024 s
  TLS / SSL Handshake:    0.048 s
  Pre-transfer Time:      0.048 s
  Time to First Byte:     0.082 s (TTFB)
  -----------------------------------------------
  Total Request Time:     0.095 s
```
*(This output immediately isolates whether latency is caused by slow DNS, network routing, or backend database processing!)*.

---

## 4. The `wget` Command: Large Downloads & Mirroring

```bash
# 1. Download file and save locally:
wget https://releases.ubuntu.com/24.04/ubuntu-24.04-live-server-amd64.iso

# 2. Resume an interrupted/broken download (-c):
wget -c https://releases.ubuntu.com/24.04/ubuntu-24.04-live-server-amd64.iso

# 3. Download in the background (-b):
# Runs as background daemon, logging progress to 'wget-log':
wget -b https://example.com/huge-dataset.tar.gz

# 4. Recursively mirror an entire documentation website for offline use:
wget -r -np -k https://docs.example.com/
# -r: recursive
# -np: do not ascend to parent directories
# -k: convert links to relative paths for offline browsing
```

---

## Summary

- Use `curl` for API interactions, sending custom headers/data, and Kubernetes health checks.
- Use `wget` for downloading large files, resuming broken transfers (`-c`), and mirroring documentation.
- Always use `--connect-timeout 5` in automated scripts to prevent hanging.
- Profile network, TLS, and TTFB latency using `curl -w`.
