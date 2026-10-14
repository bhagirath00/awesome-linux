<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://cdn.simpleicons.org/linux/white" />
    <img alt="Linux Logo" src="https://cdn.simpleicons.org/linux/black" width="60" />
  </picture>
</p>

<h1 align="center">Awesome Linux</h1>

<p align="center">
  <a href="https://awesome.re"><picture><source media="(prefers-color-scheme: dark)" srcset="https://shieldcn.dev/badge/awesome.svg?variant=outline&amp;mode=dark&amp;font=geist&amp;logo=awesomelists" /><img alt="awesome" src="https://shieldcn.dev/badge/awesome.svg?variant=outline&amp;mode=light&amp;font=geist&amp;logo=awesomelists" /></picture></a>
  <a href="https://kernel.org/"><picture><source media="(prefers-color-scheme: dark)" srcset="https://shieldcn.dev/badge/Linux-Kernel.svg?variant=outline&amp;mode=dark&amp;font=geist&amp;logo=linux" /><img alt="Linux" src="https://shieldcn.dev/badge/Linux-Kernel.svg?variant=outline&amp;mode=light&amp;font=geist&amp;logo=linux" /></picture></a>
  <a href="LICENSE"><picture><source media="(prefers-color-scheme: dark)" srcset="https://shieldcn.dev/badge/license-MIT.svg?variant=outline&amp;mode=dark&amp;font=geist" /><img alt="license" src="https://shieldcn.dev/badge/license-MIT.svg?variant=outline&amp;mode=light&amp;font=geist" /></picture></a>
</p>

<p align="center">
  <em>A curated collection of Linux fundamentals, system administration, shell scripting, and DevOps engineering workflows.</em>
</p>

## Table of Contents

- [Introduction](#introduction)
- [Why Linux for DevOps?](#why-linux-for-devops)
- [Quick Command Reference](#quick-command-reference)
- [Learning Modules](#learning-modules)
  - [Core System Foundations](#core-system-foundations)
  - [Administration & Automation](#administration--automation)
  - [Production Operations & Performance](#production-operations--performance)
- [Resources](#resources)
- [Contributing](#contributing)
- [License](#license)

## Introduction

Welcome to **Awesome Linux**! This repository is an extensive, hands-on learning roadmap and reference manual designed for developers, systems administrators, and DevOps engineers mastering the Linux operating system.

> **"In open source, we feel strongly that to really do something well, you have to get a lot of people involved."** — Linus Torvalds.

## Why Linux for DevOps?

Linux powers more than 90% of the world's cloud infrastructure, container runtimes, and enterprise clusters:

| Capability | Importance in DevOps |
|---|---|
| **Cloud & Container Native** | Docker, Kubernetes, and major cloud compute instances run natively on Linux kernels |
| **System Visibility** | Deep diagnostic tooling via `/proc`, `sysfs`, `systemd`, `cgroups`, and `ebpf` |
| **Automation First** | Powerful CLI utilities and Bash shell scripting enable automated CI/CD pipelines |
| **Networking & Security** | Granular control over iptables, nftables, namespaces, SSH, and POSIX permissions |

## Quick Command Reference

Explore curated, instant command cheatsheets for daily engineering:

- [**Linux Command Guides & Cheatsheets**](./commands/) — File manipulation, permissions, disk usage, processes, networking, and compression.

## Learning Modules

### Core System Foundations

| Module | Description |
|---|---|
| [**Module 01 - Linux Fundamentals**](./Module%2001%20-%20Linux%20Fundamentals/) | Architecture, Shell Basics, FHS (Filesystem Hierarchy Standard), and Navigation |
| [**Module 02 - File Management and Processing**](./Module%2002%20-%20File%20Management%20and%20Processing/) | Text Processing (`grep`, `sed`, `awk`), Find, Redirection, and Pipelines |
| [**Module 03 - User and Permission Management**](./Module%2003%20-%20User%20and%20Permission%20Management/) | UIDs/GIDs, `chmod`, `chown`, `umask`, SUID/SGID/Sticky bit, ACLs, and Sudoers |

### Administration & Automation

| Module | Description |
|---|---|
| [**Module 04 - Networking and Security**](./Module%2004%20-%20Networking%20and%20Security/) | IP Addressing, DNS, Ports, `curl`, `ss`, Firewalls (UFW/Firewalld), and SSH Hardening |
| [**Module 05 - Process and System Management**](./Module%2005%20-%20Process%20and%20System%20Management/) | Process Lifecycle, Signals, `top`/`htop`, `systemd` Services, and Timers |
| [**Module 06 - Shell Scripting**](./Module%2006%20-%20Shell%20Scripting/) | Bash Variables, Control Flow, Functions, Error Handling, and Automation Scripts |
| [**Module 07 - Package Management**](./Module%2007%20-%20Package%20Management/) | APT (Debian/Ubuntu), YUM/DNF (RHEL/CentOS/Fedora), Repositories, and Compilation |

### Production Operations & Performance

| Module | Description |
|---|---|
| [**Module 08 - Linux Performance Tuning**](./Module%2008%20-%20Linux%20Performance%20Tuning/) | CPU, Memory, Disk I/O, Network Tuning, Sysctl, and Kernel Parameters |
| [**Module 09 - Linux for Containers and DevOps**](./Module%2009%20-%20Linux%20for%20Containers%20and%20DevOps/) | Cgroups, Namespaces, Storage Drivers, Ephemeral VMs, and Cloud-Init |
| [**Module 10 - Linux Troubleshooting**](./Module%2010%20-%20Linux%20Troubleshooting/) | System Crash Analysis, OOM Killer, Log Auditing (`journalctl`), and Recovery |

## Resources

### Official Documentation

| Resource | Link |
|---|---|
| Linux Kernel Documentation | [kernel.org/doc](https://www.kernel.org/doc/) |
| Debian Documentation | [debian.org/doc](https://www.debian.org/doc/) |
| Red Hat Enterprise Linux Docs | [access.redhat.com/documentation](https://access.redhat.com/documentation/) |
| ArchWiki (Comprehensive Reference) | [wiki.archlinux.org](https://wiki.archlinux.org/) |

## Contributing

Contributions are welcome! Please read our [Contributing Guidelines](CONTRIBUTING.md) before submitting a Pull Request.

## License

This educational content is open source and available under the **MIT License**. See [LICENSE](LICENSE) for details.

<p align="center">
  <strong>Happy Scripting!</strong>
</p>
