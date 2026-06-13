---
name: darwin-host-provisioning
description: "Provision and restore macOS (Darwin) Nix-Darwin hosts: host configuration, home-manager, SOPS secrets, flake registration. Use when adding, restoring, or cloning a Darwin host in a Nix flake."
version: 1.0.0
author: Operator 11O
license: MIT
platforms: [macos]
metadata:
  hermes:
    tags: [nix-darwin, macos, host-provisioning, sops, home-manager, flake]
    related_skills: [home-manager-pitfalls, nix-module-bulk-edit, github-auth]
---

# Darwin Host Provisioning

Provision or restore a macOS host managed by Nix-Darwin and home-manager.

## When to Use

- Adding a new Darwin host to a Nix flake
- Restoring a retired host from git history
- Cloning an existing host config for a new machine
- Debugging host configuration failures

## Anatomy of a Darwin Host

Each host lives under `hosts/<name>/` with two files:

```
hosts/<name>/
├── darwin-configuration.nix    # System-level config
└── users/<user>/
    └── home-configuration.nix  # Home-manager config
```

And is registered in two flake modules:

- `modules/flake/darwin.nix` — `darwinConfigurations.<name>`
- `modules/flake/devenv.nix` — SOPS creation rules + age keys

## Step-by-Step: Add a New Host

### 1. Create `hosts/<name>/darwin-configuration.nix`

```nix
{ config, ... }:

{
  imports = [
    ../../modules/darwin/base.nix
    ../../modules/darwin/workstation.nix
  ];

  home-manager.users.<user>.imports = [
    ./users/<user>/home-configuration.nix
  ];

  networking.hostName = "<name>";

  nix.extraOptions = ''
    !include ${config.sops.templates.nix-config.path}
  '';

  sops = {
    age = {
      generateKey = true;
      keyFile = "/var/lib/sops-nix/key.txt";
      sshKeyPaths = [ "/etc/ssh/ssh_host_ed25519_key" ];
    };
    defaultSopsFile = ../../secrets/<name>.enc.yaml;
    defaultSopsFormat = "yaml";
    secrets.nix-access-token = { };
    templates.nix-config.content = ''
      extra-access-tokens = "github.com=${config.sops.placeholder.nix-access-token}";
    '';
  };

  system.primaryUser = "<user>";

  users.users.<user> = {
    home = "/Users/<user>";
    name = "<user>";
    openssh.authorizedKeys.keys = [
      "ssh-ed25519 AAAA..." # user's key
    ];
  };
}
```

### 2. Create `hosts/<user>/<user>/home-configuration.nix`

Import all required home modules. Reference secrets via
`config.sops.templates.<name>.path` (not `config.sops.secrets`).

Key imports (adjust per host):

```nix
imports = [
  ../../../../modules/home/base.nix
  ../../../../modules/home/cloud.nix
  ../../../../modules/home/fontconfig.nix
  ../../../../modules/home/ghostty.nix
  ../../../../modules/home/helix.nix
  ../../../../modules/home/starship.nix
  ../../../../modules/home/vcs.nix
  ../../../../modules/home/workstation.nix
  ../../../../modules/home/zed-editor.nix
];
```

### 3. Register in `modules/flake/darwin.nix`

```nix
darwinConfigurations.<name> = inputs.nix-darwin.lib.darwinSystem {
  pkgs = import inputs.nixpkgs {
    system = "<aarch64|x86_64>-darwin";
    config.allowUnfree = true;
  };
  modules = [
    ../../hosts/<name>/darwin-configuration.nix
    inputs.home-manager.darwinModules.home-manager  # or .default
    inputs.sops-nix.darwinModules.sops              # or .default
    {
      home-manager.sharedModules = [
        inputs.catppuccin.homeModules.default
        inputs.colemak.homeModules.default
        inputs.devlib.homeModules.default
        inputs.sops-nix.homeModules.default
      ];
    }
  ];
};
packages.<system>.<name> = self.darwinConfigurations.<name>.system;
```

**Note:** Older hosts use `home-manager.darwinModules.home-manager` (attribute
path) while newer ones use `.default`. Match the existing pattern in the file.

### 4. Register in `modules/flake/devenv.nix`

Add the host's age public key to the `age` list and a creation rule:

```nix
age = [
  "age1xxxxx # <name>"
  # ... existing keys
];
```

```nix
{
  path_regex = "secrets/<name>.enc.yaml";
  key_groups = [
    {
      age = age ++ [
        "age1yyyyy # host"
      ];
    }
  ];
}
```

### 5. Create SOPS secrets file

```bash
SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt \
  sops --age <pubkey1>,<pubkey2> \
  secrets/<name>.enc.yaml
```

## Restoring a Retired Host

1. Find the retirement commit: `git log --all --oneline --diff-filter=D -- '*<name>*'`
2. Get the parent commit (last state before deletion): `git show <commit>^ -- hosts/<name>/`
3. Extract both files from the parent commit
4. Adapt to current conventions (see Pitfalls below)
5. Re-register in flake modules

## Pitfalls

- **SSH_AUTH_SOCK corruption:** Some hosts have a truncated
  `SSH_AUTH_SOCK="***"` value. This is a pre-existing bug — do not
  propagate it. macOS handles SSH agent natively via launchd; omit the
  override entirely.
- **home-manager module path:** `darwinModules.home-manager` vs
  `darwinModules.default` — check existing hosts in the file and match.
- **SOPS secrets vs templates:** Current convention uses `sops.templates`
  (rendered files) not `sops.secrets` (raw secret references). The old
  `secrets = { foo = { }; }` pattern is deprecated.
- **Architecture:** Verify with `uname -m`. Intel Macs = `x86_64-darwin`,
  Apple Silicon = `aarch64-darwin`. Do not assume.
- **Authorized keys:** Always include the user's SSH public key in
  `users.users.<user>.openssh.authorizedKeys.keys`.
- **No PII in commits:** Never commit real secret values, private keys,
  or personal data. Only commit the Nix configuration files referencing
  secrets.

## Verification

```bash
# Build the configuration (dry run)
nix build .#darwinConfigurations.<name>.system

# Apply (requires sudo)
sudo nix run nix-darwin/master#darwin-rebuild -- switch --flake .#<name>
```
