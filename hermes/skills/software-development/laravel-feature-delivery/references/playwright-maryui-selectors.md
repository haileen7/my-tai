# Playwright selectors against MaryUI / DaisyUI renderings

Observed DOM facts, not guesses — probe first, then encode. `error-context.md`
under `test-results/<test>/` contains a YAML snapshot of the page at failure,
which is usually faster than re-running to inspect the DOM.

## MaryUI inputs gain a trailing space in `placeholder`

```html
<!-- DOM actually looks like this -->
<input placeholder="جستجو... ">
```

So an exact match fails while a substring match succeeds:

```ts
// WRONG — never matches
page.locator('input[placeholder="جستجو..."]')
// RIGHT
page.locator('input[placeholder*="جستجو"]')
```

Always prefer `*=` for placeholders, labels and any Persian text in a
`Mary\Traits` input. The same trailing-space behavior applies to `label` text, so
prefer `getByRole('button', { name: ... })` over attribute selectors for buttons.

## Waiting for Livewire

Reuse the project's helpers (`waitForLivewire`, `waitForToast`,
`waitForSearchResults`) rather than `waitForTimeout`. For a debounced search,
`waitForLivewire` returns as soon as the request starts resolving — follow it
with an explicit expectation on the result row, which retries until the DOM
settles.

## Asserting absence

Assert a non-existent row is absent from the table, not from the whole body — the
form that submitted it still holds the typed text:

```ts
await expect(page.locator('tbody td', { hasText: 'نام تست' })).toHaveCount(0);
```

`exact: true` is not a supported option on `toContain` — if you need exactness,
use a locator-scoped assertion instead.

## 403 from a Livewire route

A permission-denied SPA navigation returns HTTP 403 with a body mentioning
`access rights`. Assert the status from the navigation response and that no
table body rendered:

```ts
const resp = await page.goto('/the/route');
expect(resp!.status()).toBe(403);
await expect(page.locator('body')).toContainText('access rights');
await expect(page.locator('table tbody tr')).toHaveCount(0);
```

## Sidebar / drawer

Menu links live under `.drawer-side a[href]`. Collect them with `evaluateAll` to
assert per-role visibility:

```ts
const hrefs = await page.locator('.drawer-side a[href]').evaluateAll((as) =>
  as.map((a) => a.getAttribute('href') || ''),
);
expect(hrefs).toContain('/it/zabbix-devices');
```

When a new admin-only page is added, add its href to the shared admin-only list in
the RBAC spec so every non-admin role is asserted to lack it — a one-line change
that keeps the role matrix honest.
