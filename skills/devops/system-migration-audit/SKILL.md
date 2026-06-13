---
name: system-migration-audit
description: "Audit and plan system package manager migrations (e.g. Homebrew
  to MacPorts), including inventory, categorization, migration planning, and
  cleanup."
version: 1.0.0
author: Operator 11O
license: MIT
platforms: [macos]
metadata:
  hermes:
    tags: [macOS, Migration, Homebrew, MacPorts, mise, Package-Manager,
           Audit]
    related_skills: [darwin-host-provisioning]
---

# System Migration Audit

Audit, plan, and execute system-level package manager migrations on macOS.
Primary use case: migrating from Homebrew (x86_64 EOL) to MacPorts + mise.

## Trigger

Use this skill when the user mentions:

- "migrate brew to port"
- "homebrew deprecated"
- "macports install"
- "package migration"
- "audit brew"
- "reinstall packages"
- "intel mac eol"

## Phase 1: Inventory

```bash
# Top-level Homebrew packages (not dependencies)
brew leave 2>/dev/null

# Full formula + cask list
brew list --formula
brew list --casks

# Check what Homebrew itself needs
brew deps --tree $(brew leave 2>/dev/null) 2>/dev/null

# Optionally check MacPorts for available equivalents
port search --name --regex "^<package>$" 2>/dev/null
```

### Output Format

Produce a table:

| #  | Brew formula  | Category     | MacPorts equiv. | Status          |
|----|---------------|--------------|-----------------|-----------------|
| 1  | `openssl`     | crypto       | `openssl3`      | port available  |
| 2  | `postgresql`  | db           | `postgresql16`  | port available  |
| 3  | `cmatrix`     | fun          | --              | skip            |

Phases:

1. **Audit** -- map every top-level package
2. **Categorize** -- system | dev | lang-runtime | skip
3. **Plan** -- write the migration commands
4. **Execute** -- run them in dependency order
5. **Verify** -- check the replaced packages work
6. **Cleanup** -- optionally remove Homebrew

## Phase 2: Categorize

Sort every package into one of five buckets:

| Bucket     | Action                                  | Examples                |
|------------|-----------------------------------------|-------------------------|
| `port-has` | MacPorts provides it, install there     | cmake, openssl, curl    |
| `mise-has` | Better managed by mise (asdf-style)     | nodejs, python, ruby    |
| `brew-only`| Only available via Homebrew -- keep     | mas, some GUI casks     |
| `system`   | macOS or MacPorts built-in -- skip      | perl, python3 (system)  |
| `skip`     | Not needed on this host                 | fun/temporary packages  |

## Phase 3: Port Mapping

MacPorts naming conventions differ from Homebrew:

| Pattern                       | Example                                   |
|-------------------------------|-------------------------------------------|
| `openssl` -> `openssl3`       | Versioned port                            |
| `postgresql` -> `postgresql16`| Major version port                        |
| `node` -> `nodejs22`          | Major version (or `nodejs` for latest)    |
| `@version` suffixes           | `ruby3.3` becomes `ruby33`               |
| No equivalent                 | Accept the drop or keep via Homebrew      |

```bash
# Verify a port exists before declaring it
port search --exact --name <candidate> 2>/dev/null || echo "NOT FOUND"

# Check variants (macports equivalent of brew options)
port variants <portname>
```

## Phase 4: Migration Order

Install in this exact order:

1. **First**: `sudo port install curl openssl3` (build toolchain
   foundation)
2. **Then**: core developer tools (`cmake`, `ninja`, `pkg-config`,
   `autoconf`)
3. **Then**: languages via `mise` (so they shadow system/port versions)
4. **Then**: services (`redis`, `postgresql`, etc.)
5. **Last**: apps and casks

## Phase 5: Cleanup

```bash
# Uninstall a brew package (leaves formula in place)
brew uninstall <formula>

# Remove ALL Homebrew (only after confirming every dependency is migrated)
/bin/bash -c "$(curl -fsSL \
  https://raw.githubusercontent.com/Homebrew/install/HEAD/uninstall.sh)"
```

## Pitfalls

1. **MacPorts builds from source by default.** Large packages (LLVM,
   Chromium) can take 30+ minutes. Use `port install -b` to attempt
   binary archives first.

2. **MacPorts names might not exist yet.** A port can lag behind the
   latest upstream release. Always verify with `port search` before
   declaring a mapping.

3. **PATH ordering matters.** MacPorts lives in `/opt/local/bin`; system
   tools in `/usr/bin`. Ensure MacPorts is before system but after mise
   shims: `mise > macports > homebrew > system`.

4. **GPG signing on macOS.** `pinentry-mac` requires GUI. For headless
   environments, switch to SSH commit signing:
   `git config --global gpg.format ssh`.

5. **mise vs. MacPorts for language runtimes.** Prefer mise: it supports
   `.tool-versions` files, per-project overrides, and is
   cross-platform. Only use MacPorts for system libraries (openssl,
   readline, sqlite).

6. **Do not mix Homebrew and MacPorts for the same library.** Linker
   conflicts will silently break builds. Pick one, stick to it, and
   unlink the other.
