---
name: system-migration-audit
description: "Audit and plan system package manager migrations (e.g. Homebrew to MacPorts). Use when migrating CLI tools, dev tools, or libraries between package managers on macOS or Linux."
version: 1.0.0
author: Operator 11O
license: MIT
platforms: [macos, linux]
metadata:
  hermes:
    tags: [migration, package-manager, brew, macports, nix, audit]
    related_skills: [darwin-host-provisioning]
---

# System Migration Audit

Audit installed packages and plan a migration between package managers.

## When to Use

- Homebrew dropping support for your architecture (e.g. x86_64 macOS)
- Switching from one package manager to another
- Auditing what's actually needed vs. orphaned deps
- Preparing a clean reinstall

## Phase 1: Inventory

### Get top-level packages

```bash
# Homebrew — only user-installed (not deps)
brew leaves

# MacPorts — explicitly installed
port installed requested

# Nix — profiles
nix profile list
```

### Categorize each package

For each leaf/required package, determine:

| Category | Examples | Strategy |
|----------|----------|----------|
| Dev tool | node, python, go, rust | Prefer mise or nix |
| CLI tool | gh, rg, jq, less, tree | MacPorts or nix |
| Library | openssl, libffi, zlib | Keep as dep, don't reinstall |
| GUI app | firefox, vscode, slack | brew cask or MacPorts |
| System-level | pinentry, pkgconf | Keep, managed by new PM |
| Hermes dependency | — | Do not touch |

**Rule:** Libraries and deps should never be explicitly installed in the
new manager — they'll be pulled in automatically.

### Check for drops

Some packages may have different names or not exist in the target manager:

```bash
# MacPorts
port search --name "^<pkg>$"

# Check if there's a nixpkgs equivalent
nix search nixpkgs#<pkg>
```

## Phase 2: Migration Plan

Create a migration document:

```
## Migration: Homebrew → MacPorts + mise

### Stay on Homebrew (no MacPorts equivalent)
- mas (Mac App Store CLI — macOS-only)

### Move to MacPorts
- gh → gh
- ripgrep → ripgrep
- ffmpeg → ffmpeg
- pinentry → pinentry

### Move to mise
- node@22 → mise use node@22
- python@3.12 → mise use python@3.12
- go → mise use go@latest

### Drop (unused / replaced)
- transmission-cli (use web UI instead)
```

## Phase 3: Execute

### Clean Homebrew

```brew remove --force $(brew list --formula)
brew cleanup -s
brew autoremove
rm -rf /Library/Caches/Homebrew/*
```

Then uninstall Homebrew itself:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/uninstall.sh)"
```

### Install MacPorts packages

```bash
sudo port install gh ripgrep ffmpeg pinentry
```

### Install mise tools

```bash
mise use -g node@22
mise use -g python@3.12
mise use -g go@latest
```

## Phase 4: Verify

```bash
# Verify all critical tools work
gh --version
rg --version
ffmpeg -version
node --version
python3 --version

# Verify no broken symlinks from old package manager
find -L ~ -maxdepth 4 -type l -not -path "*/Library/Containers*" 2>/dev/null
```

## Pitfalls

- **Homebrew Intel EOL:** Homebrew is dropping x86_64 macOS support.
  `brew leaves` then `brew remove` is the migration path. Do not wait
  until it breaks.
- **MacPorts source builds:** Expect long compile times for large ports
  (ffmpeg, cmake). Run `sudo port install` in background.
- **PATH ordering:** MacPorts (`/opt/local/bin`) must come before
  Homebrew (`/opt/homebrew/bin` or `/usr/local/bin`) in PATH, or
  binaries will shadow each other. Remove Homebrew from PATH after
  migration.
- **mise vs brew:** mise manages dev tool versions, not system packages.
  They are complementary. Use mise for languages, MacPorts for everything
  else.
- **Hermes Node deps:** Hermes has its own node_modules
  (`~/.hermes/...`). Do not alter or remove these.
