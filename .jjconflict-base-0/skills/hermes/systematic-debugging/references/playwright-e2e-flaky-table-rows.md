# Playwright E2E Flaky Table Rows

## Symptom

A test waits for or clicks `locator('tr').nth(1)` after navigating to a table page, then fails later because the expected detail page content never appears.

Typical failure shape:

```ts
await page.getByTestId('tableAdministrationServiceChains').locator('tr').nth(1).click()
await expect(page.getByRole('heading', { name: 'Chaîne de services' })).toBeVisible()
```

The click can target a placeholder row such as `Chargement...`, an empty-state row, or a row that exists before API data has hydrated.

## Debugging steps

1. Read the component template for the table.
2. Identify whether rows exist before data loads (`v-if="isLoading || !items.length"`, skeleton rows, empty state rows).
3. Find the durable data-row selector used by real records, usually a `data-testid` built from resource data.
4. Change the test to wait for the data-row selector, not table row position.
5. Click the data-row only after `await expect(row).toBeVisible()`.

## Preferred pattern

```ts
const firstResourceRow = page
  .getByTestId('tableResource')
  .locator('[data-testid^="resourceTr-"]')
  .first()

await expect(firstResourceRow).toBeVisible()
await firstResourceRow.click()
```

If asserting a dynamic heading, use a regex or exact full expected name intentionally:

```ts
await expect(page.getByRole('heading', { name: /Chaîne de services/ })).toBeVisible()
```

## Pitfall

Do not rely on `await expect(page.getByTestId('loader')).toHaveCount(0)` when the loader may not have mounted yet. That assertion can pass too early. Waiting for the data row expresses the real readiness condition.
