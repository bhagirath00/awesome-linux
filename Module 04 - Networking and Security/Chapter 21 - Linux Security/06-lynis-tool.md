# Chapter 21: Linux Security - Automated Auditing with Lynis

## 1. Introduction: What is Lynis?

**Lynis** is a battle-tested, open-source security auditing, compliance testing, and system hardening scanning tool for Linux, macOS, and BSD systems. 

Unlike heavy commercial endpoint agents, Lynis is lightweight, written completely in POSIX shell script, requires zero third-party dependencies, and can run natively directly from Git or via standard package managers.

DevOps engineers and Security teams use Lynis for:
- Continuous CI/CD infrastructure vulnerability scans.
- Pre-production baseline auditing.
- Compliance verification against CIS Benchmarks, HIPAA, and PCI DSS.
- Detecting misconfigured file permissions, obsolete kernel parameters, and exposed ports.

---

## 2. Installing Lynis

### Method 1: Via Official Repositories
```bash
# Ubuntu / Debian
sudo apt update && sudo apt install -y lynis

# RHEL / Rocky / AlmaLinux
sudo dnf install -y epel-release
sudo dnf install -y lynis
```

### Method 2: Running Directly via Git (Zero-Install / Ephemeral)
```bash
git clone https://github.com/CISOfoundation/lynis.git /opt/lynis
cd /opt/lynis
sudo ./lynis audit system
```

---

## 3. Running a System Audit

To perform a comprehensive security scan, execute `lynis audit system` with `sudo` permissions:

```bash
sudo lynis audit system --quick
```

Lynis executes over 300 automated tests covering:
- Bootloaders and GRUB password protection
- Kernel parameters (`sysctl`) and memory protections
- Authentication, PAM, password policies, and expired accounts
- Firewall configurations (`ufw`, `firewalld`, `iptables`)
- SSH server hardening (`sshd_config`)
- File integrity checkers (`AIDE`, `Tripwire`)
- Logging frameworks and audit daemons (`auditd`, `rsyslog`)

---

## 4. Understanding Lynis Output and Hardening Index

At the conclusion of the audit, Lynis generates a detailed summary containing the **Hardening Index**:

```
================================================================================

  -[ Lynis 3.0.9 Results ]-

  Great, no serious issues found.

  ===============================================================================

  [+] Security Scan Details:
  ---------------------------
  Hardening index : 64 [#############       ]
  Tests performed : 265
  Plugins enabled : 0

  [+] System report:
  ---------------------------
  Report file: /var/log/lynis-report.dat
  Log file   : /var/log/lynis.log

  [+] Warnings (2):
  ---------------------------
  ! Root password has no expiration date [AUTH-9288]
  ! Kernel parameter 'net.ipv4.conf.all.accept_redirects' is insecure [KRNL-5622]

  [+] Suggestions (28):
  ---------------------------
  * Configure password hashing rounds in /etc/login.defs [AUTH-9229]
  * Set a password on GRUB bootloader to prevent unauthorized console boots [BOOT-5122]
  * Enable process accounting with acct or psacct [ACCT-9622]
  * Consider installing fail2ban to automatically block brute-force attacks [SSH-7408]
  * Harden compilers by restricting permissions to root [HRDN-7222]

================================================================================
```

---

## 5. Investigating Warnings and Suggestions

Every suggestion and warning emitted by Lynis comes with a unique **Test ID** (e.g., `[KRNL-5622]`, `[SSH-7408]`).

You can query Lynis directly to obtain precise remediation instructions for any test code:

```bash
lynis show details KRNL-5622
```

```
2026-10-14 04:55:00 Description: Check IPv4 ICMP redirect acceptance
2026-10-14 04:55:00 Result: Insecure value (1) found for net.ipv4.conf.all.accept_redirects
2026-10-14 04:55:00 Recommendation: Set sysctl net.ipv4.conf.all.accept_redirects to 0 in /etc/sysctl.d/99-security.conf
2026-10-14 04:55:00 Reference: https://cisofy.com/lynis/controls/KRNL-5622/
```

---

## 6. Automating Lynis in CI/CD and Cron

To run Lynis unattended in automated pipelines or nightly cron audits without terminal colors or interactive prompts:

```bash
# Non-interactive cron scan
sudo lynis audit system --cronjob --quiet

# Check Hardening index directly from the report data
grep "^hardening_index=" /var/log/lynis-report.dat
```

```
hardening_index=78
```

You can set up threshold alerts in CI/CD (e.g., if `hardening_index` drops below 75, fail the pipeline).

---

## 7. Summary Takeaways

- Lynis is an industry-standard open-source auditing and compliance scanner.
- The **Hardening Index** provides a quantifiable benchmark score for OS security.
- Look up specific remediation advice using `lynis show details <TEST-ID>`.
- Use `--cronjob --quiet` for unattended compliance scans in automated environments.
