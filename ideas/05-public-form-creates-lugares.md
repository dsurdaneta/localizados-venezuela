# 05 - Public contribution form creates Lugar records from unmoderated input

| Priority | Value | Effort | Risk | Performance |
| -------- | ----- | ----- | ---- | ----------- |
| **P1**   | Med   | Low   | Low  | -           |

## Problem

When a citizen submits a "persona" contribution, the endpoint immediately **creates a `Lugar`
(place) document** if no case-insensitive name match exists - using raw, unmoderated form input.
The whole point of the pending/moderation flow is that public input is reviewed before it lands in
the canonical data, but `Lugar` creation bypasses that. Result: the places collection accumulates
typos, duplicates, joke entries, and spam ("Hospital aaaa", "asdf", etc.), which then pollute the
`/lugares` listing, the place filter, and dedup logic.

A related smell: the persona branch writes both a `Contribucion` **and** a `Localizado` (pending)
plus the `Lugar`, duplicating the moderation pathway and making the data model harder to reason about.

## Evidence

```148:157:src/app/api/v1/contribuciones/route.ts
    let lugar = await Lugar.findOne({
      nombre: new RegExp(`^${escapeRegex(lugarNombre)}$`, "i"),
    });
    if (!lugar) {
      lugar = await Lugar.create({
        slug: makeSlug(lugarNombre),
        nombre: lugarNombre,
        tipo: "otro",
      });
    }
```

This runs inside the public `POST` handler, before any moderator sees the submission.

## Impact

- **Data pollution:** unmoderated, free-text place names become first-class `Lugar` records that
  show up publicly in `[src/app/lugares/page.tsx](../src/app/lugares/page.tsx)`.
- **Dedup drift:** near-duplicate places fragment counts and make merging harder (idea `13`).
- **Abuse surface:** combined with no rate limiting (idea `04`), this is an easy spam vector.

## Proposed solution

1. **Defer place creation to moderation.** Store the submitted `lugarNombre` only on the
   `Contribucion.persona.lugarNombre` field (already modeled) and let a moderator pick an existing
   `Lugar` or create one at approval time - which the admin OCR/import flow already supports.
2. Reconsider creating a pending `Localizado` at submission time; keeping the raw data on the
   `Contribucion` until approval is simpler and avoids half-published records tied to junk places.
3. If immediate matching to existing places is desired, **only link to an existing `Lugar`** and
   never create a new one from the public path.

## Acceptance criteria

- [ ] Submitting a "persona" with a brand-new place name does **not** create a `Lugar`.
- [ ] Moderators can still resolve the place (existing or new) at approval time.
- [ ] `/lugares` no longer shows places that were never moderated.
