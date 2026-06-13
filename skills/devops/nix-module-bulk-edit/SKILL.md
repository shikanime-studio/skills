---
name: nix-module-bulk-edit
description:
  "Bulk-edit Nix module files (e.g. adding integrations to language modules)
  using reliable patterns. Use when modifying many .nix files in a Nix project."
version: 1.0.0
author: Operator 21O
license: MIT
---

# Nix Module Bulk Edit

Reliable patterns for bulk-editing Nix module files in a project like devenv.

## When to use

- Adding a new config line/block to many language modules
- Updating template references across many files
- Any repetitive Nix file modification across a directory

## Pitfalls

### 1. Don't use `execute_code` Python scripts for bulk file writes

The sandbox can run out of disk space
(`OSError: [Errno 28] No space left on device`). **Use `terminal` with `perl` or
`sed` instead.**

### 2. Don't use `patch` tool for Nix files with complex indentation

The `patch` tool's fuzzy matching often fails on Nix due to subtle whitespace
differences. **Use `terminal` with `perl -i -0pe` for reliable multi-line
replacements.**

### 3. Always verify Nix option paths

In devenv, `gitnr` is `config.gitnr`, NOT `config.integrations.gitnr`. Using the
wrong path causes: `The option 'integrations' does not exist.` **Check the
module's `options` block to confirm the exact path.**

### 4. Verify template existence before committing

Many assumed templates don't exist. Always check:

- GitHub main: `https://api.github.com/repos/github/gitignore/contents/`
- GitHub community:
  `https://api.github.com/repos/github/gitignore/contents/community`
- GitHub global: `https://api.github.com/repos/github/gitignore/contents/Global`
- TopTal: `https://www.toptal.com/developers/gitignore/api/list`

### 5. Handle multi-line blocks carefully

When removing previously-added blocks, use `perl -0pe` with `/s` flag:

```bash
perl -i -0pe 's/^\s+integrations\.gitnr\."\.gitignore"\.templates = \[\n.*?\];\n//gms' file.nix
```

## Workflow

1. **Map first**: Build a complete mapping of file → template before editing
2. **Verify options first**: Use `search.nixos.org` or the module docs to
   confirm the exact NixOS option path before editing
3. **Prefer direct module options**: If a module exposes a native option, use it
   instead of raw `environment.etc` / tmpfiles plumbing
4. **Clean first**: Remove any previously-added incorrect lines
5. **Apply**: Use `terminal` + `perl` for bulk insertion
6. **Verify**: `grep` or a module-specific check to confirm correct lines exist
7. **Build**: Run the relevant rebuild/build command to validate

## Nix style

- Prefer short-hand attr syntax when it stays readable and accurate.
- Keep YAML-shaped Nix attrsets close to the final manifest shape; avoid
  unnecessary wrapper layers.
- For RKE2 manifests on NixOS, prefer `services.rke2.manifests` with `target` +
  `content` over manual `environment.etc` and `systemd.tmpfiles.rules`.

See `references/rke2-manifests.md` for the option shape and the RKE2 manifest
pattern.

## NixOS module editing notes

- Prefer `services.<name>.manifests`, `services.<name>.settings`, or similar
  native options when the module already supports generated configuration.
- Use raw filesystem staging only when the module has no native declarative
  hook.
- For RKE2, `services.rke2.manifests.<name>.content` can generate YAML directly,
  and `target` controls the manifest filename under
  `/var/lib/rancher/rke2/server/manifests`.
- Keep generated YAML minimal and explicit; avoid extra tmpfiles indirection
  when the module can write the file itself.
- For already-formatted external disks on hosts, prefer `fileSystems` with
  `by-label`/`by-uuid` and `nofail` + `x-systemd.automount`; see
  `references/external-disks-by-label.md`.
- See `references/rke2-manifests.md` for the concise option shape and the
  manifest-generation pattern.

## Example: Adding gitnr to language modules

```bash
# Step 1: Clean any previous attempts
for f in $(grep -rl 'integrations\.gitnr\|gitnr\.' src/modules/languages/); do
  perl -i -0pe 's/^\s+integrations\.gitnr\."\.gitignore"\.templates = \[.*?\];\n//gm' "$f"
  perl -i -0pe 's/^\s+integrations\.gitnr\."\.gitignore"\.templates = \[\n.*?\];\n//gms' "$f"
done

# Step 2: Add correct line (example for go.nix)
perl -i -pe 's/(config = lib\.mkIf cfg\.enable \{)/$1\n    gitnr.\".gitignore\".templates = [ \"gh:Go\" ];/' src/modules/languages/go.nix

# Step 3: Verify
grep -rn 'gitnr' src/modules/languages/
```
