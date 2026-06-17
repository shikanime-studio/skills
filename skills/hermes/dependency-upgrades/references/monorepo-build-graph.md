# Monorepo build graph notes

Observed in this repo during the Vite 8 migration:

- Root build using `pnpm -r run build` was too loose for packages that depend on
  sibling build artifacts.
- `pnpm -r --workspace-concurrency=1 run build` fixed the race, but the better
  long-term shape was `pnpm -r --sort run build` so pnpm topologically orders
  workspace builds.
- The client build succeeded with LightningCSS warnings from DSFR legacy media
  queries; the warnings were non-fatal.
- `apps/server` and `apps/server-nestjs` had `prebuild` scripts that re-ran
  `pnpm i --frozen-lockfile`, which caused concurrent install contention during
  recursive builds.
- Guarding that install with
  `test -d node_modules || pnpm i --frozen-lockfile --ignore-scripts` avoided
  the lock contention during workspace builds.

Practical guidance:

1. Prefer a graph-aware recursive build (`--sort`) before falling back to full
   serialization.
2. If a package’s build script triggers `pnpm i`, make it idempotent or move
   install/setup out of the recursive build path.
3. Keep package exports pointing at built artifacts (`dist/`) when runtime
   imports cross workspace boundaries.
