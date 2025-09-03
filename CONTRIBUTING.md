# Contributing to Awesome Linux

Thank you for your interest in contributing to **Awesome Linux**! We welcome contributions from the community.

## How to Contribute

### Reporting Content Issues

If you find a bug in the shell scripts, command syntax, or have a suggestion for the content:

1. Check if the issue already exists
2. Open a new issue with a clear title and description
3. Include your Linux distribution, kernel version, and steps to reproduce

### Making Changes

1. **Fork** the repository
2. **Clone** your fork locally:
   ```bash
   git clone https://github.com/bhagirath00/awesome-linux.git
   ```
3. Create a **branch** for your changes:
   ```bash
   git checkout -b feature/your-feature-name
   ```
4. Make your changes following the style guide below
5. **Commit** your changes:
   ```bash
   git commit -m "feat: Add your feature description"
   ```
6. **Push** to your fork:
   ```bash
   git push origin feature/your-feature-name
   ```
7. Open a **Pull Request**

## Style Guide

### Commands & Shell Scripts

- Ensure commands are portable (POSIX-compliant or explicitly specified for Bash/Zsh).
- Use `shellcheck` best practices for all shell scripts.
- Document any required root/sudo privileges clearly.
- Provide clear, expected command output snippets alongside commands.
