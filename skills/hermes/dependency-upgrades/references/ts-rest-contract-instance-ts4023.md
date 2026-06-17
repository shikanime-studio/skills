# ts-rest contract instance extraction pitfall

Context: this repo split shared ts-rest wiring out of
`packages/shared/src/api-client.ts` into a dedicated instance module while
keeping contracts in sibling files.

Observed failure:

- `tsc` failed with TS4023 on an exported `contractInstance`:
  - `Exported variable 'contractInstance' has or is using name 'tag' from`
    `external module '@ts-rest/core' but cannot be named.`

Takeaways:

- A raw exported generic instance can break declaration emit even when runtime
  code is fine.
- If the new module exists only to reduce coupling, keep the public surface tiny
  and prefer a wrapper that hides the generic return type rather than exporting
  the raw instance.
- In this repo, a safe fallback was to export `apiPrefix` plus a `router()`
  helper that calls `initContract()` internally, while `api-client.ts` imports
  that helper and keeps the contract assembly local.
- If declaration emit still breaks, revert the split before continuing the
  upgrade work; do not keep layering workarounds onto the shared package.

Verification:

- Re-run `pnpm build` after the change.
- Re-run `pnpm test:cov` to ensure the workspace still resolves the shared
  package correctly.
