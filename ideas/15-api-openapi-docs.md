# 15 - Replace hand-written API docs with a generated OpenAPI spec

| Priority | Value | Effort | Risk | Performance |
| -------- | ----- | ----- | ---- | ----------- |
| **P3**   | Low   | Med   | Low  | -           |

## Problem

The public API is documented manually in two places - the README table and a hand-built `/api`
page - which drift out of sync with the actual route handlers. There's no machine-readable schema
for integrators to generate clients or validate responses against. The README explicitly lists
"Documentación de la API" as a wanted contribution.

## Evidence

- API docs are maintained by hand in `[src/app/api/page.tsx](../src/app/api/page.tsx)` and in the
  README "API pública (v1)" section.
- Route handlers (`[src/app/api/v1/localizados/route.ts](../src/app/api/v1/localizados/route.ts)`,
  `[src/app/api/v1/lugares/route.ts](../src/app/api/v1/lugares/route.ts)`, etc.) and DTO types
  (`[src/lib/types.ts](../src/lib/types.ts)`) are the real source of truth but aren't linked to the
  docs.

## Impact

- **Drift:** docs can silently diverge from behavior (especially relevant after PII masking in
  idea `01` changes response shapes).
- **Integrator friction:** no spec to generate clients or validate against.

## Proposed solution

1. Author an **OpenAPI 3 spec** for `/api/v1/*` (paths, params, and response schemas derived from
   `LocalizadoDTO` / `LugarDTO` / `ApiListResponse`).
2. Serve it (e.g. `/api/openapi.json`) and render the `/api` page from it (Swagger UI / Scalar /
   Redoc) so the page can't drift.
3. Optionally derive the schema from Zod (if validation is added) so types, validation, and docs
   share one source.
4. Keep this **after idea `01`**, so the documented shapes reflect the privacy-corrected responses.

## Acceptance criteria

- [ ] A machine-readable OpenAPI spec describes all public v1 endpoints.
- [ ] The `/api` page is generated from the spec (no separate hand-maintained copy).
- [ ] Response schemas match the actual DTOs.
