# DNS Configuration in Linux: Resolution Order, `/etc/resolv.conf`, and `systemd-resolved`

## Overview

The Domain Name System (DNS) is the critical infrastructure layer that translates human-readable domain names (such as `api.github.com` or `database.production.internal`) into machine-routable IP addresses (`140.82.121.4`).

In DevOps, Site Reliability Engineering, and Kubernetes, DNS misconfigurations are the root cause behind a huge percentage of production incidents:
- Application microservices experiencing 5-second timeouts due to unreachable nameservers.
- Kubernetes pods failing cluster service discovery due to overly aggressive `ndots` search configurations.
- Direct DNS cache poisoning or stale resolution after DNS record migrations.

---

## 1. The Linux Resolution Precedence Stack

When an application invokes standard POSIX name resolution system calls (`getaddrinfo` or `gethostbyname`), the Linux C Library (`glibc`) consults the **Name Service Switch (NSS)** configuration file: **`/etc/nsswitch.conf`**.

```text
Application queries hostname "database.internal"
                     │
                     ▼
           [/etc/nsswitch.conf]
         (hosts: files dns myhostname)
                     │
       ┌─────────────┴─────────────┐
       ▼                           ▼
[ Stage 1: files ]        [ Stage 2: dns ]
  /etc/hosts                /etc/resolv.conf (or systemd-resolved)
  (Hardcoded Map)           (Upstream Public/VPC Resolvers)
```

By default, Linux evaluates **`files` (`/etc/hosts`) FIRST**, before ever querying upstream DNS servers.

---

## 2. Local DNS Mapping: `/etc/hosts`

The `/etc/hosts` file provides a local, static hostname-to-IP lookup table that takes precedence over external network DNS:

```bash
# /etc/hosts
127.0.0.1   localhost
127.0.1.1   web-server-01

# Static internal network overrides:
10.0.5.20   db.production.internal
10.0.5.21   cache.production.internal
```

### DevOps Kubernetes Pattern: `hostAliases`
In Kubernetes, you cannot edit `/etc/hosts` manually inside a container because container filesystems are ephemeral. Instead, Kubernetes provides the `hostAliases` manifest specification to inject `/etc/hosts` entries automatically:

```yaml
spec:
  hostAliases:
  - ip: "10.0.5.20"
    hostnames:
    - "legacy-database.internal"
```

---

## 3. Upstream Resolver Configuration: `/etc/resolv.conf`

The `/etc/resolv.conf` file defines the upstream nameservers and domain search suffixes used by the resolver library.

```ini
# /etc/resolv.conf
nameserver 1.1.1.1
nameserver 8.8.8.8
search production.internal aws.internal
options timeout:2 attempts:3 ndots:2
```

### Directive Breakdown:
- **`nameserver <IP>`**: Upstream DNS server to query. Up to 3 nameserver lines can be specified. Linux queries them in top-to-bottom order.
- **`search <domains...>`**: Search list for host-name lookup. If you type `ping database`, Linux automatically appends suffixes: `database.production.internal`, then `database.aws.internal`.
- **`options timeout:N`**: Number of seconds the resolver waits for a response from a remote nameserver before trying the next (default 5s).
- **`options attempts:N`**: Number of query attempts before failing.

### The Kubernetes `ndots:5` Production Trap:
In Kubernetes pods, `/etc/resolv.conf` is configured with `options ndots:5` by default.
- If a domain has fewer than 5 dots (e.g. `api.stripe.com` has 2 dots), the resolver treats it as an internal cluster domain first!
- It queries `api.stripe.com.default.svc.cluster.local`, then `api.stripe.com.svc.cluster.local`, generating **multiple wasted DNS queries for every single external HTTP request**!
- Hardening tip: Always use Fully Qualified Domain Names ending with a dot (`api.stripe.com.`) in latency-critical microservices.

---

## 4. Modern Linux DNS Caching: `systemd-resolved`

On modern Ubuntu (20.04+) and systemd-based distributions, `/etc/resolv.conf` is actually a symbolic link pointing to a local stub resolver maintained by **`systemd-resolved`**:

```bash
$ ls -l /etc/resolv.conf
lrwxrwxrwx 1 root root 39 /etc/resolv.conf -> ../run/systemd/resolve/stub-resolv.conf

$ cat /etc/resolv.conf
nameserver 127.0.0.53
options edns0 trust-ad
search company.internal
```

`systemd-resolved` runs a local DNS cache on IP **`127.0.0.53:53`**, dramatically reducing outbound network traffic by caching responses locally.

### Inspecting and Flushing DNS with `resolvectl`
```bash
# View active upstream DNS servers per network interface:
$ resolvectl status

# Flush local DNS cache (instant fix after updating public DNS records):
$ sudo resolvectl flush-caches

# Query hostname statistics:
$ resolvectl statistics
```

---

## 5. DNS Diagnostic Utilities: `dig`, `nslookup`, and `host`

```bash
# Fast DNS query with dig (displays answer section, TTL, and nameserver used):
dig +short api.github.com

# Query specific upstream nameserver directly:
dig @1.1.1.1 api.github.com

# Trace complete DNS resolution from root servers down:
dig +trace api.github.com

# Reverse DNS lookup (IP to domain):
dig -x 140.82.121.4 +short
```

---

## Summary

- Resolution precedence is defined in `/etc/nsswitch.conf` (defaults to `/etc/hosts` before external DNS).
- `/etc/resolv.conf` defines upstream `nameserver` IPs, search suffixes, and timeouts.
- Modern Linux uses `systemd-resolved` listening locally on `127.0.0.53` to provide local DNS caching.
- Flush DNS caches with `sudo resolvectl flush-caches`.
- Diagnose DNS resolution latency and records using `dig +short` and `dig +trace`.
