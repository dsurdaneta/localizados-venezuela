# 13 - Deduplication is exact-only; near-duplicates slip through

| Priority | Value | Effort | Risk | Performance |
| -------- | ----- | ----- | ---- | ----------- |
| **P3**   | Med   | High  | Med  | -           |

## Problem

Deduplication relies on a unique index over `(lugarId, nombreNormalizado)` that applies **only to
published** records. This catches exact normalized-name collisions for the same place, but misses:

- **Pending** duplicates (the partial filter excludes them).
- **Near**-duplicates: spelling variants, OCR errors, name order swaps ("José Pérez" vs "Perez
  Jose"), accents/initials, or the **same person at a different place**.

Since data arrives from multiple sources (Excel, OCR markdown, OCR-from-image, citizen
contributions), duplicate and fragmented records are likely, which undermines trust ("is this two
people or one listed twice?").

## Evidence

The unique index only covers published records by exact normalized name:

```59:67:src/lib/models/Localizado.ts
// Deduplicación en seed y moderación (solo publicados)
localizadoSchema.index(
  { lugarId: 1, nombreNormalizado: 1 },
  {
    unique: true,
    partialFilterExpression: { estado: "published" },
    name: "dedupe_publicado",
  }
);
```

Normalization is exact (accent/case/space only), so fuzzy variants don't collapse:

```76:83:src/lib/models/Localizado.ts
export function normalizeNombre(nombre: string): string {
  return nombre
    .normalize("NFD")
    .replace(/[\u0300-\u036f]/g, "")
    .toUpperCase()
    .replace(/\s+/g, " ")
    .trim();
}
```

A `merge-duplicates` script exists (`[scripts/merge-duplicates.ts](../scripts/merge-duplicates.ts)`,
`npm run merge`) but operates on exact matches.

## Impact

- **Data quality / trust:** fragmented or duplicated victim records.
- **Moderator load:** manual cleanup of near-duplicates.

## Proposed solution

1. **Moderation-time duplicate suggestions:** when approving/creating, surface likely matches using
   token-sorted name comparison + similarity (e.g. trigram / Levenshtein) and cédula equality, so
   the moderator can merge instead of creating a new record.
2. **Improve `merge-duplicates`** to propose fuzzy candidates (sorted name tokens + edit distance,
   cédula match across places) in dry-run, with `--apply` to merge.
3. Consider a **token-sorted normalized name** field to catch order swaps cheaply.
4. Treat this as iterative - start with cédula-based cross-place matching (highest precision), then
   layer fuzzy name matching with a human in the loop (avoid auto-merging different people).

## Acceptance criteria

- [ ] Moderators see probable duplicates when approving/creating a record.
- [ ] `npm run merge` can propose (and, with `--apply`, perform) fuzzy merges, not just exact.
- [ ] Cédula matches across different places are detectable.
