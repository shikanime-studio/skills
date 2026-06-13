---
name: home-manager-pitfalls
description: Home Manager module gotchas — silent failures, hidden dependencies, and non-obvious requirements when configuring programs via home-manager. Use when debugging why a home-manager module isn't producing expected config, or before enabling any home-manager program module.
---

# Home Manager Pitfalls

## Core Pattern: Parent Program Must Be Enabled

Many home-manager modules configure a *sub-tool* of a larger program (e.g. mergiraf is a sub-tool of git). **The parent program must be explicitly enabled**, or the module silently produces no output.

### Mergiraf requires Git

Setting `programs.mergiraf.enable = true` without `programs.git.enable = true` results in:
- No `[merge "mergiraf"]` driver in `.gitconfig`
- No `conflictStyle = diff3`
- No `.gitattributes` with `* merge=mergiraf`

The module writes to `programs.git.extraConfig` and `programs.git.attributes`, which are dead-letter channels if git isn't enabled.

**Fix:**
```nix
programs.git.enable = true;
programs.mergiraf.enable = true;
```

### Jujutsu merge-tools need explicit config

jj ships with a built-in mergiraf preset, but it still needs explicit wiring:
```nix
programs.jj = {
  enable = true;
  settings."merge-tools.mergiraf" = {
    program = "mergiraf";
    merge-args = ["merge" "--git" "$base" "$left" "$right"];
  };
};
```

### General Rule

Before enabling any home-manager module that seems to do nothing:
1. Check what `programs.<parent>` it writes to
2. Ensure the parent program is also enabled (`programs.<parent>.enable = true`)
3. Verify the generated config in `/nix/store/*-home-manager-files/.config/<program>/`

## Nix Store Debugging on macOS

Broad searches across `/nix/store` consistently time out. Use targeted approaches:

- **Find a specific module:** `find /nix/store -name "mergiraf.nix" -maxdepth 6 2>/dev/null` (add `-maxdepth` to limit)
- **Read known paths directly** from `flake.lock` revisions or `which <binary>` output
- **Check generated config** at the home-manager files symlink target:
  `ls -la ~/.config/git/config` → follow symlink to `/nix/store/*-home-manager-files/.config/git/config`

## Verification Pattern

After changing home-manager config:
1. Rebuild: `home-manager switch --flake .#<host>`
2. Check the symlink target: `readlink -f ~/.config/git/config`
3. Read the actual generated file (not through the symlink, which may be cached)
4. Confirm expected config sections exist
