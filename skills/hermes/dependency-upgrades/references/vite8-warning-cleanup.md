# Vite 8 warning cleanup in this repo

Goal: keep the Vite 8 migration simple and remove non-fatal build noise when the
warnings are not worth a permanent workaround.

## What we changed

1. Switched the client CSS minifier from Lightning CSS to esbuild:
   - `apps/client/vite.config.ts`
   - `build.cssMinify: 'esbuild'`
   - Result: DSFR legacy media-query warnings stopped.

2. Added the missing asset referenced by client CSS:
   - `apps/client/public/icons/government-fill-blue.svg`
   - Result: the CSS `mask-image: url("/icons/government-fill-blue.svg")`
     stopped producing a runtime-resolution warning during build.

3. Removed the dynamic import wrapper in `packages/shared/src/api-client.ts`:
   - Replaced repeated `await import('./contracts/index.js')` with static
     imports from the same barrel.
   - Result: `[INEFFECTIVE_DYNAMIC_IMPORT]` warning disappeared.

## Why this is the preferred shape here

- The build becomes quieter without carrying a Lightning CSS compatibility shim
  forever.
- The contract helper stays easy to read.
- The asset problem is solved at the source instead of with a build hack.

## Verification

Run:

```sh
pnpm build
```

Expected outcome:

- no Lightning CSS invalid media query warnings
- no ineffective-dynamic-import warning for
  `packages/shared/dist/contracts/index.js`
- build succeeds end to end

## Notes

- If a future dependency genuinely requires Lightning CSS features, reintroduce
  `css.lightningcss.errorRecovery` only for that case.
- If a warning references a CSS asset path, first check whether the asset exists
  under `apps/client/public/` and matches the path exactly.
- If a warning comes from a repeated dynamic import of a barrel, prefer a single
  static import when the module graph is already needed at startup.
