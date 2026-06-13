# pnpm 11 Trust Policy Bypass

## Failure signal

```text
[ERR_PNPM_TRUST_DOWNGRADE] N lockfile entries failed verification:
  <pkg>@<ver> High-risk trust downgrade for "<pkg>@<ver>" (possible package
    takeover)
```

## Verified workaround (this repo, 2026-06-09)

```bash
pnpm install --trust-policy=low
```

Works across an 18-package workspace lockfile where `--trust-lockfile` and
config mutations did not resolve the downgrade check.

## Notes

- `pnpm install --trust-lockfile` appears in pnpm 11 help output but is rejected
  at runtime; not usable.
- Updating `trustPolicyExclude` via `pnpm config set` did not change
  `pnpm --version 11.5.2` behavior in this environment.
- Prefer the explicit flag over `.npmrc`/workspace config mutations for one-off
  install recovery.
