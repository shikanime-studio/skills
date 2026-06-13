---
name: darwin-host-provisioning
description: "Provision and restore macOS Nix-Darwin hosts, including host configuration, home-manager, SOPS secrets, and flake registration. Covers both new host creation and restoring retired hosts from git history."
version: 1.0.0
author: Operator 11O
license: MIT
platforms: [macos]
metadata:
  hermes:
    tags: [macOS, Nix, Nix-Darwin, Home-Manager, SOPS, Provisioning, Host-Management]
    related_skills: [github-auth, github-repo-management]
---

# Darwin Host Provision

Provision or restore a macOS (Darwin) Nix-Darwin managed host within the
Shikanime Studio monorepo.

## Scope

This skill covers two scenarios:

1. **New host** — a brand-new macOS machine needs Nix-Darwin flakes, a user
   profile, secrets, and HCP vault access.
2. **Restore from git history** — the machine's config was previously removed
   (e.g. NixOS-to-Darwin migration) and needs reconstruction.

## Detection

Use this skill when the user mentions:

- "provision darwin host"
- "add darwin host"
- "restore kaltashar" (or any hostname)
- "migrate nix-darwin"
- "host-configuration.nix"
- "home-configuration.nix"

## Workflow

Every host requires these items:

| Layer   | What                                           | Where                                       |
|---------|------------------------------------------------|---------------------------------------------|
| Host    | `hosts/<host>/darwin-configuration.nix`       | `modules/flake/darwin.nix`                  |
| User    | `hosts/<host>/users/<user>/home-configuration.nix` | `modules/flake/home-manager.nix`       |
| Secret  | SOPS rule for host `<host>.age`                | `.sops.yaml`                                |
| devenv  | `.envrc` registering the host                  | `modules/flake/devenv.nix`                  |
| Flake   | Output registered in `flake.nix`               | Monorepo root                               |

### Step 1: Inventory

```bash
# Check what the old host had
jj log -r 'ancestors(main)' -- 'hosts/<host>/*'

# Find the last known good state before deletion
jj log --limit 20 -- 'hosts/<host>/*' --template 'change_id ++ " " ++ description'

# Identify secrets
cat ~/.config/sops/age/keys.txt
cat .sops.yaml
```

### Step 2: Harden Secrets First

```bash
# Generate an age key for a new host
ssh-keygen -t ed25519 -f /tmp/age -N ""
age1pub=$(ssh-keygen -y -f /tmp/age | age-keygen -y - 2>/dev/null)
# Or convert SSH key to age directly
age-keygen -y ~/.ssh/id_ed25519
```

- Add the host age public key to `.sops.yaml` under `creation_rules`.
- Re-encrypt affected secrets: `sops updatekeys <file>`.
- Push the new `.sops.yaml` **before** the host config (separate PR if
  stacked).

### Step 3: Confirm Host Facts

Collect these before writing any file:

| Fact               | Source / Command                              |
|--------------------|-----------------------------------------------|
| Hostname           | User input                                    |
| Architecture       | `uname -m` → `aarch64` or `x86_64`          |
| Nix profile        | `nix profile info` or `/nix/var/nix/profiles`|
| Tailscale status   | `tailscale status`                            |
| hardware uuid      | `system_profiler SPHardwareDataType 2>/dev/null \| grep UUID` |
| SSH host key       | `/etc/ssh_host_ed25519_key.pub`              |

### Step 4: Write Host Config

Create `hosts/<host>/darwin-configuration.nix`:

```nix
{ pkgs, lib, config, ... }:
{
  imports = [ ../../modules/darwin ];

  networking = {
    hostName = "<host>";
    computerName = "<host>";
  };

  # Enable the services you need
  services = {
    tailscale.enable = true;
    # Add other services as needed
  };

  # If a NixOS host previously used declarative networking
  # this is handled by Nix-Darwin services instead

  nixpkgs.hostPlatform = lib.mkDefault "<arch>-darwin";
  system.stateVersion = 5;
}
```

### Step 5: Write User Config

```bash
mkdir -p hosts/<host>/users/<user>
```

Create `hosts/<host>/users/<user>/home-configuration.nix`
### Step 6: Register Host

1. Add the host import in `modules/flake/darwin.nix`
2. Add the user import in `modules/flake/home-manager.nix`
3. Add the `.envrc` line in `modules/flake/devenv.nix`

### Step 7: Validate

```bash
# Check that the flake evaluates
nix flake check --no-build 2>&1 | tail -20

# Check file formatting
nix fmt -- --check 2>&1 | head -10

# Attempt a build (may fail on non-darwin)
nix build .#darwinConfigurations.<host>.system --no-link 2>&1 | tail -20
```

## Pitfalls

1. **Never commit a plaintext secret.** Use SOPS + age for `.env` files
   containing tokens or passwords.

2. **`SSH_AUTH_SOCK` in Nix-Darwin.** macOS provides this natively via
   `launchd`. Do **not** add a hardcoded `SSH_AUTH_SOCK` value — it changes
   between sessions and breaks non-nix-darwin builds.

3. **`hardware.uuid`.** Required for services that bind to hardware identity.
   Without it, HCP vault and tailgate may refuse to connect. Use `system_profiler`
   to retrieve it; never fabricate one.

4. **Nixpkgs Darwin support lifecycle.** Verify the target nixpkgs version
   still ships `darwin` lib. Nixpkgs `26.11`+ dropped `x86_64-darwin`.

5. **Files that require host identity.** `gpg-agent.conf`, `ssh_config`,
   and `1password` socket paths may differ across hosts. Use conditional Nix
   expressions rather than hardcoded paths.

6. **The `stdenv.hostPlatform` vs `nixpkgs.hostPlatform` mismatch.** Always set
   `nixpkgs.hostPlatform` as the default. Setting `stdenv` directly will silently
   break cross-compilation.
