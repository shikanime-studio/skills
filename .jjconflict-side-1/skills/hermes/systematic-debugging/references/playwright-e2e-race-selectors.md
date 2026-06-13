# Playwright E2E Race Selectors

## Pattern

A Playwright test can pass or fail incorrectly when it selects a generic table row such as:

```ts
page.getByTestId('tableX').locator('tr').nth(1)
```

This is fragile when the table renders a placeholder/loading/empty row before real data arrives. The test may click the placeholder row, never navigate, and then fail on a later assertion such as a missing heading.

## Better selector contract

Prefer selecting the real data row by its stable `data-testid` prefix and waiting for it before clicking:

```ts
const firstRow = page
  .getByTestId('tableAdministrationServiceChains')
  .locator('[data-testid^="serviceChainTr-"]')
  .first()

await expect(firstRow).toBeVisible()
await firstRow.click()
```

If the heading includes dynamic data, avoid exact text unless the full value is part of the contract:

```ts
await expect(
  page.getByRole('heading', { name: /Chaîne de services/ }),
).toBeVisible()
```

## Diagnostic signal

If a test fails after a click with “element not found” for a detail-page heading, but the preceding list assertion uses a generic row index, suspect that the click targeted a transient row rather than the loaded data row.

## Scope

This is a test synchronization fix, not a product-code fix. Do not bundle separate flaky tests unless they share the same root cause.