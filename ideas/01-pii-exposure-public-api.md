# 01 - PII of victims exposed via open public API

| Priority | Value | Effort | Risk | Performance |
| -------- | ----- | ------ | ---- | ----------- |
| **P0**   | High  | Med    | Med  | -           |

## Problem

The public, unauthenticated API and the public person pages expose **sensitive personal data**
of disaster victims - full name, **cédula (national ID)**, **phone**, and **home address** -
to anyone, with `Access-Control-Allow-Origin: *`. This makes the dataset trivially scrapeable
by third parties (fraud, doxxing, targeted scams against vulnerable people).

This is the highest-stakes item in the backlog: the project's own README states it consolidates
"fuentes públicas y contribuciones ciudadanas", but consolidating and re-publishing structured,
queryable ID + phone + address per named individual is a materially different privacy exposure
than the scattered original sources.

## Evidence

The DTO returned to the public includes every sensitive field:

```28:47:src/lib/serializers.ts
  return {
    slug: localizado.slug,
    nombreCompleto: localizado.nombreCompleto,
    edad: localizado.edad ?? undefined,
    cedula: localizado.cedula ?? undefined,
    telefono: localizado.telefono ?? undefined,
    direccion: localizado.direccion ?? undefined,
    observaciones: localizado.observaciones ?? undefined,
    ...
```

It is served with wildcard CORS on every `GET /api/v1/*`:

```42:48:src/lib/api.ts
export function corsJson(data: unknown, init?: ResponseInit) {
  const headers = new Headers(init?.headers);
  headers.set("Access-Control-Allow-Origin", "*");
  ...
```

The search endpoint returns these fields in bulk and is reachable with arbitrary `q`, `page`,
and `limit` (up to 100/page), enabling enumeration:

- `[src/app/api/v1/localizados/route.ts](../src/app/api/v1/localizados/route.ts)`
- `LIST_PROJECTION` in `[src/lib/queries.ts](../src/lib/queries.ts)` selects `cedula`, `telefono`, `direccion`.

The per-person page at `/localizados/{slug}` also renders the full record and is indexable
(see `[src/app/sitemap.ts](../src/app/sitemap.ts)` and `[src/app/robots.ts](../src/app/robots.ts)`).

## Impact

- **Privacy / safety:** ID numbers + phone + address per named person is a high-value target
  for fraud and harassment, and the affected people did not opt in.
- **Legal / reputational:** likely conflicts with data-minimization expectations; could force
  an emergency takedown.
- **Scraping:** wildcard CORS + bulk pagination = the whole dataset is one script away.

## Proposed solution

Apply **data minimization** in layers (do the cheap ones first):

1. **Mask sensitive fields by default in the public DTO.** Return `cedula`/`telefono` only as
   masked values (e.g. `V-****1234`, `+58 ****-**89`); drop `direccion` from list responses
   entirely, keep only at most city/zone on the detail page.
2. **Reveal-on-exact-match only.** Allow the full cédula to be shown only when the requester
   already supplied a matching cédula (search returns "match found at Hospital X" rather than
   dumping the ID). This preserves the core "is my relative located?" use case without
   publishing a directory.
3. **Lock down the list endpoint:** lower the max `limit`, and consider requiring a non-empty
   query so the API can't be paginated as a full dump.
4. **Tighten CORS:** restrict `/api/v1/*` to known origins instead of `*` (the same-origin form
   already uses `isSameOriginRequest`); only relax if external integrations are a stated goal.
5. **De-index detail pages:** add `robots: noindex` (or `X-Robots-Tag`) to `/localizados/{slug}`
   so search engines don't cache victim records.

Coordinate the exact policy with the project maintainers, since it is partly a product/ethics
decision, not purely technical.

## Acceptance criteria

- [ ] Public list/detail responses no longer return raw `cedula`, `telefono`, full `direccion`.
- [ ] A relative can still confirm a person's location (match-based lookup) without the API
      acting as a bulk PII directory.
- [ ] CORS for `/api/v1/*` is scoped (or explicitly justified as public).
- [ ] `/localizados/{slug}` is excluded from indexing/sitemap, or only minimal data is indexed.
- [ ] Decision and rationale documented in the README.
