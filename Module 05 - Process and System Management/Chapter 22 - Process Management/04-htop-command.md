# Chapter 22: Process Management - The htop Command

## 1. Introduction: What is htop?

`htop` is an advanced, cross-platform interactive process viewer and text-mode system monitor. It was created by Hisham Muhammad as an enhanced, modern alternative to the traditional `top` utility.

While `top` provides basic tabular output, `htop` delivers an intuitive, full-color ncurses terminal interface featuring:
- Visual color gauges for individual CPU cores, memory, and swap usage.
- Horizontal and vertical scrolling through the full process and argument lists.
- Tree view visualization (`F5`) showing process ancestry directly.
- Interactive mouse support (clicking to sort columns, select processes, or kill tasks).
- Simplified signal sending without memorizing numerical PIDs.

---

## 2. Installation

`htop` is not always pre-installed on base cloud OS images, but is available in official package repositories:

```bash
# Ubuntu / Debian
sudo apt update && sudo apt install -y htop

# RHEL / Rocky Linux / AlmaLinux
sudo dnf install -y epel-release && sudo dnf install -y htop

# Arch Linux
sudo pacman -S htop
```

---

## 3. The Visual Layout of htop

```
  1  [|||||||||||||||||||||||||||||||||||||||||||   94.2%]   Tasks: 78, 120 thr; 1 running
  2  [|||||||||                                     21.4%]   Load average: 1.12 0.85 0.54
  3  [||||||||||||||||||||                          45.0%]   Uptime: 12 days, 06:14:22
  4  [||||||                                        14.8%]
  Mem[|||||||||||||||||||||||||||||||||        3.24G/7.76G]
  Swp[|                                         12.0M/2.00G]

  PID USER      PRI  NI  VIRT   RES   SHR S CPU% MEM%   TIME+  Command
 1420 devops     20   0  984M  342M 42100 S 94.2  4.2  1:12.45 /usr/bin/node server.js
 1120 postgres   20   0  541M  128M 32100 S  1.0  1.6  0:45.10 /usr/lib/postgresql/bin/postgres
 1422 www-data   20   0 84320 24120  8200 S  0.5  0.3  0:02.14 nginx: worker process
  890 root       20   0 72456 18244  6100 S  0.0  0.2  0:00.89 /lib/systemd/systemd-journald

F1Help  F2Setup  F3Search F4Filter F5Tree  F6SortBy F7Nice - F8Nice + F9Kill  F10Quit
```

### Color Gauge Interpretations:
- **CPU Meter**:
  - **Blue**: Low-priority nice processes.
  - **Green**: Normal user processes.
  - **Red**: Kernel system processes.
  - **Cyan**: Virtualized guest steal time.
- **Memory Meter**:
  - **Green**: Used memory pages (allocated by running applications).
  - **Blue**: Buffer cache pages.
  - **Orange**: Disk page cache.

---

## 4. Key Function Keys and Shortcuts

The bottom bar in `htop` displays direct mappings to your keyboard's Function keys (`F1` through `F10`):

| Function Key | Alternative | Action |
| :--- | :--- | :--- |
| **`F1`** | `h`, `?` | **Help**: Displays all keybindings and color explanations. |
| **`F2`** | `S` | **Setup**: Customize meters, columns, display options, and colors. |
| **`F3`** | `/` | **Search**: Incrementally search for a process name. |
| **`F4`** | `\` | **Filter**: Filter the list to display *only* matching processes. |
| **`F5`** | `t` | **Tree View**: Toggle hierarchical process tree (parent/child view). |
| **`F6`** | `>` | **Sort By**: Open an interactive menu to choose sort column. |
| **`F7`** | `[` | **Nice Lower**: Decrease nice value (increases CPU priority). |
| **`F8`** | `]` | **Nice Raise**: Increase nice value (decreases CPU priority). |
| **`F9`** | `k` | **Kill**: Open a menu of all 64 POSIX signals (`SIGTERM`, `SIGKILL`, etc.). |
| **`F10`**| `q` | **Quit**: Exit `htop`. |

---

## 5. top vs. htop Comparison

| Feature | `top` | `htop` |
| :--- | :--- | :--- |
| **Availability** | Pre-installed everywhere by default | Must usually be installed manually via package manager |
| **Per-CPU Gauges** | Text summary line (or toggled with `1`) | Beautiful visual color bars per core out of the box |
| **Navigation** | Command-driven, limited scrolling | Smooth arrow-key scrolling (horizontal & vertical) |
| **Tree View** | Basic command text | Real-time interactive collapsible tree (`F5`) |
| **Mouse Interaction**| No | Yes (full click, scroll, and menu support) |
| **Process Termination** | Type PID and signal manually | Select process, press `F9`, select signal from menu |

---

## 6. Summary Takeaways

- `htop` provides an interactive, full-color visual monitoring dashboard.
- The function keys (`F1`-`F10`) make filtering, searching, tree viewing, and killing processes intuitive.
- Use `F4` to filter processes dynamically when troubleshooting microservices.
- Use `F5` tree view to quickly trace parent and child relationships.
