# Vite 7 to Vite 8 Migration

## Core Changes

- **Rolldown** replaces Rollup for bundling
- **Oxc** replaces esbuild for JS transformation
- **Lightning CSS** replaces esbuild for CSS minification

## Required Config Changes

### build.target

`'ESNext'` is NOT valid for Lightning CSS. Change to:

```typescript
build: {
  target: 'baseline-widely-available',  // or 'esnext' for JS-only, but CSS
    needs browser targets
}
```

### CSS Minification

If CSS contains legacy hacks (DSFR uses `@media (min-width: 0\0)`):

```typescript
css: {
  lightningcss: {
    errorRecovery: true,
  },
}
```

## Deprecated Options (auto-converted)

### Dependency Optimizer

| esbuild (deprecated)                      | Rolldown (new)                                  |
| ----------------------------------------- | ----------------------------------------------- |
| `optimizeDeps.esbuildOptions.minify`      | `optimizeDeps.rolldownOptions.output.minify`    |
| `optimizeDeps.esbuildOptions.treeShaking` | `optimizeDeps.rolldownOptions.treeshake`        |
| `optimizeDeps.esbuildOptions.define`      | `optimizeDeps.rolldownOptions.transform.define` |
| `optimizeDeps.esbuildOptions.loader`      | `optimizeDeps.rolldownOptions.moduleTypes`      |
| `optimizeDeps.esbuildOptions.platform`    | `optimizeDeps.rolldownOptions.platform`         |

### JS Transform

| esbuild (deprecated)       | Oxc (new)                                   |
| -------------------------- | ------------------------------------------- |
| `esbuild.jsx: 'automatic'` | `oxc.jsx: { runtime: 'automatic' }`         |
| `esbuild.jsxImportSource`  | `oxc.jsx.importSource`                      |
| `esbuild.jsxFactory`       | `oxc.jsx.pragma`                            |
| `esbuild.define`           | `oxc.define`                                |
| `esbuild.supported`        | Not supported — track oxc-project/oxc#15373 |

## Not Supported

- **NativeDecorator lowering**: Oxc does not support it yet. Use
  `@rolldown/plugin-babel` with `@babel/plugin-proposal-decorators` (version
  `2023-11`) or `@rollup/plugin-swc`.
- `esbuild.supported` option
- `build.rollupOptions.output.format: 'system'` and `'amd'`
- `shouldTransformCachedModule` hook

## Migration Path

Vite 7 -> `rolldown-vite@7.2.2` (intermediate) -> `vite@^8.0.0` (final)

For a direct upgrade just update the version in package.json and fix config.
