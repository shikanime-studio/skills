# Nix Flake + Devlib Regeneration

When working with repos that use Nix flakes with devenv + devlib
(shikanime-studio/devlib), the devenv shell manages generated files like GitHub
Actions workflows. This creates a regeneration workflow that differs from plain
git repos.

## Regeneration Pattern

```bash
# 1. Enter the devenv shell (triggers file generation)
direnv allow
# OR
nix develop --accept-flake-config --no-pure-eval

# 2. Run a specific devenv task
nix develop --accept-flake-config --no-pure-eval --command bash -c "
  devenv tasks run devlib:github:workflows:install 2>&1
"

# 3. Format all files (treefmt runs actionlint, zizmor, nixfmt, etc.)
nix fmt

# 4. Run the full flake check
nix flake check --no-pure-eval
```

## Key Points

- `direnv allow` triggers `devenv:enterShell` which runs all
  `before: ["devenv:enterShell"]` tasks including
  `devlib:github:workflows:install`, `devlib:ghstack:hooks:install`,
  `devlib:gitignore:install`, `devlib:renovate:install`
- Generated files (`.github/workflows/*.yaml`, `.gitignore`, `.renovate.json`)
  are written to the repo root, NOT to the devenv root
- If generated files are git-tracked with stale content, `git rm --cached` them
  first so devenv can regenerate without git restoring old versions
- `nix fmt` runs treefmt which includes actionlint (YAML lint) and zizmor
  (GitHub Actions security audit) -- these can fail on devlib template bugs

## Common Issues

### actionlint "could not read reusable workflow file"

Devlib's `release.yaml` uses `uses: ./.github/workflows/nix.yaml` to call the
nix workflow as a reusable workflow. actionlint can't resolve local files in the
sandbox. Fix: inline the nix job steps directly in release.yaml instead of using
`uses:`.

### zizmor security audit failures

Devlib templates may generate workflows that fail zizmor audits (e.g., missing
`permission-*` inputs on `create-github-app-token`, overly broad workflow-level
permissions). Fix: update the devlib template, or add `.github/actionlint.yaml`
/ `.zizmor.yml` suppressions.

### Cachix token revoked

The `nix` check job uses Cachix for binary caching. If the Cachix auth token is
revoked (repo-level secret), the check fails. This is an infrastructure issue,
not a code issue.

## Repos Using This Pattern

- `x-shikanime/algorithm` -- flake.nix with devenv + devlib + shikanime shell
- `x-shikanime/curriculum-vitae` -- flake.nix with devenv + devlib + shikanime
  shell
- `x-shikanime/identities` -- flake.nix with devenv + devlib + shikanime-studio
  shell
- `x-shikanime/dotfiles` -- flake.nix with devenv + devlib + shikanime shell
- `x-shikanime/niximgs` -- flake.nix with devenv + devlib + shikanime-studio
  shell
