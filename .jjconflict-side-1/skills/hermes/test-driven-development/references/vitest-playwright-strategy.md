# Vitest + Playwright Dual Test Strategy

## Overview

For full-stack projects (Vite + React + TanStack Start), use a two-layer test strategy:

- **Vitest** (unit tests) — fast, isolated, runs in < 1s. Tests data models, validation logic, collections, route structure, and feature parity.
- **Playwright** (e2e tests) — browser-level, verifies routing, navigation, form interaction, responsive layout. Runs on a hot dev server or preview build.

Both tools live side by side. The rule: unit tests run on every change, e2e on commit/CI.

## Vitest Configuration

```ts
// vitest.config.ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    environment: "node",
    include: ["src/**/*.test.ts", "src/**/*.test.tsx"],
  },
});
```

## Playwright Configuration

```ts
// playwright.config.ts
import { defineConfig } from "@playwright/test";

export default defineConfig({
  testDir: "./e2e",
  fullyParallel: true,
  use: {
    baseURL: "http://localhost:4000",
    trace: "on-first-retry",
  },
  webServer: {
    command: "pnpm dev",
    url: "http://localhost:4000",
    reuseExistingServer: !process.env.CI,
  },
});
```

## TDD Workflow with Dual Layer

### RED (unit test first)

Write a vitest test for the data/logic layer first:

```ts
import { describe, expect, it } from "vitest";

describe("data model validation", () => {
  it("should validate schema fields", () => {
    const result = mySchema.safeParse({ name: "test" });
    expect(result.success).toBe(true);
  });
});
```

Run: `vitest run` — verify FAIL.

### GREEN

Implement the minimal model/logic.

### Write E2E test (second RED)

After the unit test passes, write a Playwright test for the user-facing behavior:

```ts
import { test, expect } from "@playwright/test";

test("can navigate to page", async ({ page }) => {
  await page.goto("/");
  await expect(page.getByText("Title")).toBeVisible();
});
```

### Route Structure Tests (Vitest)

An additional useful pattern: verify that all route files exist and the route tree is complete:

```ts
import { describe, expect, it } from "vitest";

describe("route structure", () => {
  const routes = [
    { path: "/", file: "index.tsx" },
    { path: "/dashboard", file: "dashboard.index.tsx" },
    { path: "/s/$slug", file: "s.$slug.index.tsx" },
  ];

  for (const route of routes) {
    it(`should have route file for ${route.path}`, async () => {
      const fs = await import("node:fs/promises");
      const filepath = new URL(`../routes/${route.file}`, import.meta.url);
      await expect(fs.stat(filepath)).resolves.toBeDefined();
    });
  }

  it("route tree imports all routes", async () => {
    const fs = await import("node:fs/promises");
    const content = await fs.readFile(
      new URL("../routeTree.gen.ts", import.meta.url),
      "utf-8",
    );
    expect(content).toContain("IndexRoute");
  });
});
```

## Package.json Scripts

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:e2e": "playwright test"
  }
}
```

## Pitfalls

- **Playwright needs browsers installed** — `pnpm exec playwright install` on fresh machines.
- **Vitest config import issue** — If `vitest` is not directly listed in the app's `package.json`, use `npx vitest run` or add it as an explicit devDependency (not just hoisted from another workspace package).
- **Route files missing from tree** — TanStack Router auto-generates `routeTree.gen.ts` via its Vite plugin, but if the plugin isn't running (dev server is off), you may need to maintain it manually. Write a route-structure test that catches orphaned files.
- **E2E tests depend on dev server** — Playwright's `webServer` config starts it automatically; use `reuseExistingServer: !process.env.CI` to avoid restarting during local dev.
- **Strictest tsconfig + vitest** — `@tsconfig/strictest` may cause `Cannot find module 'vitest'` LSP errors. These are harmless for the build (Vite resolves modules at runtime), but `vitest/config` may also fail at import time. Use `npx vitest run` to bypass.
