---
name: windows-hermes-setup
description:
  "Windows-specific Hermes Agent setup, pitfalls, and workarounds. Covers python
  path issues, GPG signing failures, gateway update locks, config encoding
  gotchas, kubectl context switches, and path conventions. Triggered when
  running Hermes on Windows or diagnosing Windows-specific errors."
author: Operator 17O
version: 1.0.0
---

# Windows — Hermes Agent Setup & Pitfalls

Field-discovered Windows issues and their fixes. All operators on Windows should
read this first.

## Path Convention

All source code repos go under `D:\Source\Repos\<domain>\<org>\<repo>`.

Example: `D:\Source\Repos\GitHub.com\shikanime-studio\skills`

**Do not clone repos to `C:\Users\<user>\`** — that location is not shared and
not backed by the canonical path convention.

## Python

`python` is the Microsoft Store stub — it opens the Store instead of running
Python.

**Always use:**

```bash
python3 -c "..."
# or
uv run python -c "..."
```

`uv run python` works without any system Python install.

## Hermes Update

`hermes update` fails while the gateway is running — Windows locks the
executable.

```bash
# Stop gateway first
hermes gateway stop
hermes update --yes

# Or force (may need reboot to fully replace)
hermes update --force --yes
```

## Config Encoding

`config.yaml` saved with UTF-8 BOM causes HTTP 400 "No models provided."

Re-save as UTF-8 without BOM. `hermes config edit` handles this correctly;
Notepad does not.

## GPG Signing

Windows Defender silently blocks `gpg` and `gpg-agent`. jj signs commits but
produces no signature when Defender intercepts.

If signing silently fails, check Defender's app control / SmartScreen settings.

## Shell

Hermes terminal on Windows runs through **bash** (git-bash / MSYS), NOT
PowerShell or cmd.exe.

Use POSIX syntax: `ls`, `$HOME`, `&&`, `|`, single-quoted strings. PowerShell
builtins (`Get-ChildItem`, `$env:FOO`, `Select-String`) will NOT work.

## kubectl Context

`tailscale configure kubeconfig <host>` switches the active context to the new
cluster.

Always verify before running cluster commands:

```bash
kubectl config current-context
kubectl config use-context <desired-context>
```

## execute_code / Sandbox

**WinError 10106** — sandbox child process can't create `AF_INET` socket.

Root cause: Hermes's env scrubber drops `SYSTEMROOT` / `WINDIR`. Python's
`socket` module needs `SYSTEMROOT` to locate `mswsock.dll`.

Diagnose inside `execute_code`:

```python
import os; print(os.environ.get("SYSTEMROOT"))
```

## Key Paths

| Purpose     | Path                                               |
| ----------- | -------------------------------------------------- |
| Config      | `C:\Users\<user>\AppData\Local\hermes\config.yaml` |
| Skills      | `C:\Users\<user>\AppData\Local\hermes\skills\`     |
| Sessions DB | `C:\Users\<user>\AppData\Local\hermes\state.db`    |
| Logs        | `C:\Users\<user>\AppData\Local\hermes\logs\`       |

## skill_manage cross_profile

When editing skills that belong to another Hermes profile, `skill_manage` and
`write_file` are blocked by default. Pass `cross_profile=true` only when
explicitly directed to edit another profile's assets.
