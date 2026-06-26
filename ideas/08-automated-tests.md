# 08 - No automated tests; CI only lints and builds

| Priority | Value | Effort | Risk | Performance |
| -------- | ----- | ----- | ---- | ----------- |
| **P1**   | High  | Med   | Low  | -           |

## Problem

The project has **zero automated tests**. CI runs only `lint`, `format:check`, and `build`, so a
change to core logic (search filters, serializers, normalization, moderation actions) can silently
break production with no signal. For a data-integrity-sensitive registry this is risky, and it
makes every other improvement in this backlog (caching, PII masking, search index changes) harder
to land safely.

This expands the existing `[.github/issues/06-tests.md](../.github/issues/06-tests.md)` with a
concrete, prioritized scope.

## Evidence

CI has no test step:

```21:31:.github/workflows/ci.yml
      - run: npm ci
      - run: npm run lint
      - run: npm run format:check
      - run: npm run build
        env:
          MONGODB_URI: mongodb://127.0.0.1:27017/localizados_venezuela
          NEXT_PUBLIC_SITE_URL: https://localizadosvenezuela.com
```

`package.json` has no `test` script:

```5:22:package.json
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    ...
    "admin:secret": "tsx scripts/create-admin-secret.ts"
  },
```

## Impact

- **Regressions ship silently** - no guardrail around search, serialization, or moderation.
- **Slows safe change** - reviewers can't trust that refactors preserve behavior.

## Proposed solution

1. Add **Vitest** + a `test` script and wire it into CI before `build`.
2. **Pure-unit tests first** (fast, no DB), covering the highest-leverage logic:
   - `normalizeNombre` in `[src/lib/models/Localizado.ts](../src/lib/models/Localizado.ts)`
     (accents, casing, whitespace).
   - `toLocalizadoDTO` / `toLugarDTO` in `[src/lib/serializers.ts](../src/lib/serializers.ts)` -
     critical to lock down once PII masking (idea `01`) is added.
   - `buildSearchFilter` logic in `[src/lib/queries.ts](../src/lib/queries.ts)` (text vs cédula
     branch, term trimming).
   - `escapeRegex` / `isSameOriginRequest` in `[src/lib/api.ts](../src/lib/api.ts)`.
   - slug helpers in `[src/lib/slug.ts](../src/lib/slug.ts)`.
3. **Integration tests** with `mongodb-memory-server` for `searchLocalizados`, pagination bounds,
   and the soft-delete/published filter.
4. Add the test step to `[.github/workflows/ci.yml](../.github/workflows/ci.yml)` and document
   `npm test` in the README/CONTRIBUTING.

## Acceptance criteria

- [ ] `npm test` runs and passes locally.
- [ ] At least the serializer, normalization, and search-filter logic are covered.
- [ ] CI runs tests on every PR and fails on regressions.
