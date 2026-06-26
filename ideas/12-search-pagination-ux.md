# 12 - Missing pagination UI on search and place pages

| Priority | Value | Effort | Risk | Performance |
| -------- | ----- | ----- | ---- | ----------- |
| **P2**   | Med   | Low   | Low  | -           |

## Problem

The backend already supports pagination (`page`/`limit`, returning `meta.totalPages`), but the
`/buscar` and `/lugares/{slug}` pages don't render pagination controls. When a common surname or a
large hospital has more than one page of results, users **can't reach the rest** - a real problem
for the core "find my relative" task. This consolidates existing issues
`[.github/issues/03-buscar-paginacion.md](../.github/issues/03-buscar-paginacion.md)` and
`[.github/issues/04-lugar-paginacion.md](../.github/issues/04-lugar-paginacion.md)`.

## Evidence

The search page computes a single page and never reads `meta.totalPages` for navigation:

```22:27:src/app/buscar/page.tsx
  const params = await searchParams;
  const q = params.q ?? "";
  const page = Number(params.page ?? "1");
  const result = q
    ? await searchLocalizados({ q, page, limit: 20 })
    : { data: [], meta: { page: 1, limit: 20, total: 0, totalPages: 0 } };
```

`searchLocalizados` / `getLugarBySlug` already return full `meta` with `totalPages`
(`[src/lib/queries.ts](../src/lib/queries.ts)`), so only the UI is missing.

## Impact

- **Findability:** results beyond page 1 are unreachable - people may wrongly conclude their
  relative isn't listed.
- **Wasted backend capability:** pagination is implemented but not surfaced.

## Proposed solution

1. Build a reusable, accessible `Pagination` component (Prev/Next + page indicator, `aria-label`s),
   usable by both `/buscar` and `/lugares/{slug}`.
2. Preserve query state in the URL (`/buscar?q=gonzalez&page=2`) so pages are shareable and
   back/forward works.
3. Show "Página X de Y" and total count; ensure it works on mobile.

## Acceptance criteria

- [ ] Users can navigate all result pages on `/buscar` and `/lugares/{slug}`.
- [ ] `q` (and any filters) persist across page changes via the URL.
- [ ] Controls are keyboard- and screen-reader-accessible; works on mobile.
