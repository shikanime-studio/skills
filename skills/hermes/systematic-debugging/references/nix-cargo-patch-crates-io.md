# Cargo `[patch.crates-io]` in Nix Builds

## Problem

When upstream `Cargo.toml` has a `[patch.crates-io]` section pointing to a git source, `fetchCargoVendor` does not vendor those git dependencies (they are not in `[[package]]` in the lock file — they appear in `[[patch.unused]]`). During the Nix build, Cargo tries to clone the git repo, which fails because the Nix sandbox has a read-only filesystem.

Error looks like:
```
Updating git repository `https://github.com/...`
error: failed to load source for dependency `...`
Caused by:
  failed to create directory `/homeless-shelter/.cargo/git/db/...`
Caused by:
  Read-only file system (os error 30)
```

## Diagnosis

1. Check if `Cargo.toml` has `[patch.crates-io]`:
   ```bash
   grep -A5 '\[patch.crates-io\]' eden/scm/Cargo.toml
   ```
2. Check if the patched crate is actually used by any package (search `[[package]]` dependencies, not `[[patch.unused]]`):
   ```bash
   grep '"<crate>"' Cargo.lock | grep -v "patch"  # check if any package depends on it
   ```

## Fix

If the patch is unused (no package in the resolved graph depends on the crates-io version), strip it during `postPatch`:

```nix
postPatch = ''
  sed -i '/^\[patch\.crates-io\]$/,/^$/d' Cargo.toml
  ...
'';
```

If the patch IS used, you need to either:
- Vendor the git dependency manually and add it to the cargo config
- Or use `importCargoLock` with `outputHashes` instead of `fetchCargoVendor`

Do NOT add `chmod +w Cargo.lock` — it is unnecessary. The `fetchCargoVendor` derivation handles the lock file separately, and the main build's Cargo only reads it.

## Also: `openssl-src` needs perl

If `openssl-sys` pulls in `openssl-src`, the OpenSSL `Configure` script requires `perl`. Add to `nativeBuildInputs`:

```nix
nativeBuildInputs = [ perl ... ];
```

And set `OPENSSL_NO_VENDOR = "1"` in `env` to prefer the Nix-provided OpenSSL.
