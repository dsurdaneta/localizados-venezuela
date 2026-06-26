# 06 - Cédula search uses an unanchored regex that can't use the index

| Priority | Value | Effort | Risk | Performance |
| -------- | ----- | ----- | ---- | ----------- |
| **P1**   | Med   | Low   | Low  | High        |

## Problem

When the search term looks like an ID (>= 4 digits), the query builds an **unanchored** regex
(`new RegExp(digits)`) on the `cedula` field. An unanchored regex (no `^` prefix, not a left
prefix) **cannot use the index** in MongoDB, so this becomes a **full collection scan** that
gets linearly slower as the dataset grows - the opposite of what you want during a high-traffic
disaster response.

There's also a subtle correctness quirk: stripping non-digits means searching `12.345.678`
matches against the stored `cedula` only if the stored value's digits contain that substring,
but stored cédulas may include prefixes/punctuation, so matching is inconsistent.

## Evidence

```40:47:src/lib/queries.ts
  const digits = term.replace(/\D/g, "");
  if (digits.length >= 4) {
    filter.cedula = new RegExp(digits);
    return filter;
  }
```

The model does index `cedula`, but the index is unusable by an unanchored regex:

```55:57:src/lib/models/Localizado.ts
localizadoSchema.index({ estado: 1, slug: 1 });
localizadoSchema.index({ estado: 1, lugarId: 1, nombreCompleto: 1 });
localizadoSchema.index({ estado: 1, cedula: 1 });
```

## Impact

- **Performance:** O(n) scan per ID search; degrades as records grow and under concurrent load.
- **Consistency:** substring matching on digits can produce surprising matches/misses.

## Proposed solution

1. **Store a normalized cédula** (digits-only) field, e.g. `cedulaNormalizada`, indexed. Populate
   it on write (seed, contribution, OCR import, admin CRUD).
2. **Query by exact equality** on the normalized field for ID lookups (`filter.cedulaNormalizada
   = digits`). Exact match uses the index and is the common "find this exact person" case.
3. If prefix search is genuinely needed, use an **anchored** regex (`new RegExp("^" + digits)`),
   which can use the index, instead of an unanchored one.
4. Backfill `cedulaNormalizada` for existing records via a small migration script in `scripts/`.

This pairs naturally with idea `01` (masking) - normalize for matching, mask for display.

## Acceptance criteria

- [ ] ID searches use an indexed exact (or anchored-prefix) match, not a full scan.
- [ ] `explain()` shows an `IXSCAN`, not a `COLLSCAN`, for a cédula query.
- [ ] Existing data backfilled with the normalized field.
