# Chapter 37: YUM and DNF Package Manager - DNF Architecture and Evolution

## 1. Introduction: From YUM to DNF

In enterprise Red Hat Linux ecosystems (RHEL, CentOS, Fedora, Rocky Linux, and AlmaLinux), software package management was historically driven by **YUM** (**Yellowdog Updater, Modified**). 

However, as enterprise applications grew complex, YUM began struggling with major architectural deficiencies:
- **Excessive Memory Footprint**: YUM was written in Python 2, loading large, unoptimized metadata structures into RAM.
- **Slow Dependency Resolution**: Its homegrown dependency resolution engine was slow and prone to unpredictable algorithm deadlocks.
- **Undocumented APIs**: Integrating YUM into modern DevOps automation and C/C++ libraries was difficult.

To resolve these bottlenecks, Red Hat engineered **DNF** (**Dandified YUM**). Introduced as default in Fedora and subsequently powering **RHEL 8, RHEL 9, Rocky Linux 8/9, and AlmaLinux 8/9**, DNF completely overhauled the package management engine while maintaining full backward command-line compatibility with YUM.

```
+--------------------------------------------------------------------------+
|                         DNF Internal Architecture                        |
+--------------------------------------------------------------------------+
|  User CLI Interface: dnf install <package> (or symlinked 'yum')          |
|                                |                                         |
|                                v                                         |
|  DNF Python/C API Layer (High-Level Controller)                          |
|                                |                                         |
|                                v                                         |
|  libsolv (High-Performance SAT Solver Dependency Engine)                 |
|  - Written in pure C (Created by SUSE)                                   |
|  - Solves dependencies using Boolean Satisfiability (SAT) in milliseconds|
|                                |                                         |
|                                v                                         |
|  librepo (High-Speed Parallel Downloader)                                |
|  - Multi-threaded, parallel mirror downloading via libcurl               |
|                                |                                         |
|                                v                                         |
|  librpm / rpm                                                            |
|  - Low-level database transaction execution                              |
|  - Writes to /var/lib/rpm/                                               |
+--------------------------------------------------------------------------+
```

---

## 2. Key Architectural Improvements in DNF

| Feature | Legacy YUM (RHEL 7 & Older) | Modern DNF (RHEL 8 & 9) |
| :--- | :--- | :--- |
| **Dependency Engine** | Custom Python heuristics (Slow, high RAM) | **`libsolv`** (High-speed C-based SAT solver) |
| **Download Engine** | Single-threaded sequential downloads | **`librepo`** (Parallel multi-connection downloads) |
| **Memory Footprint** | Extremely heavy (Can exhaust RAM on 1GB VMs) | Lightweight and optimized |
| **Transaction History** | Basic log files | Complete rollback and undo tracking (`dnf history`) |
| **Application Streams** | Not supported | **AppStreams & Modular Streams** (Multiple concurrent versions) |
| **CLI Compatibility** | `yum <command>` | `dnf <command>` (`/usr/bin/yum` is symlinked to `dnf`) |

> [!NOTE]
> On RHEL 8, RHEL 9, Rocky, and AlmaLinux, typing `yum` simply invokes `/usr/bin/dnf` via a symbolic link. You can use `yum` and `dnf` interchangeably, though using `dnf` is the modern standard.

---

## 3. What are Application Streams (AppStream Modules)?

One of the most powerful features introduced in DNF on RHEL 8 and 9 is **Modular Application Streams**.

Historically, enterprise Linux distributions locked you to a single version of a runtime for the entire 10-year lifecycle of the OS. If RHEL shipped with Node.js 16, getting Node.js 18 or 20 required messy third-party repositories.

With DNF **AppStreams**, a single repository can host multiple major versions of the same software, and administrators can enable and switch between streams cleanly:

```bash
# List available streams for Node.js
dnf module list nodejs
```

```
Name      Stream   Profiles                                Summary
nodejs    18       common [d], development, minimal, s2i   Javascript runtime
nodejs    20 [e]   common [d], development, minimal, s2i   Javascript runtime
nodejs    22       common [d], development, minimal, s2i   Javascript runtime

Hint: [d]efault, [e]nabled, [x]disabled, [i]nstalled
```

To switch to Node.js 20:
```bash
sudo dnf module enable -y nodejs:20
sudo dnf install -y nodejs
```

---

## 4. Summary Takeaways

- DNF replaced legacy YUM as the modern package manager in RHEL 8/9, Rocky, and AlmaLinux.
- `libsolv` provides lightning-fast SAT-based dependency calculations.
- `librepo` accelerates package installations via parallel downloads.
- Modular AppStreams (`dnf module`) allow choosing between multiple major versions of software runtimes.
- The `yum` binary is preserved as a transparent symlink to `dnf`.
