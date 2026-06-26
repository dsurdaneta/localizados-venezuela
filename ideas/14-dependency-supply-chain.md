# 14 - Dependency & supply-chain hygiene

| Priority | Value | Effort | Risk | Performance |
| -------- | ----- | ----- | ---- | ----------- |
| **P3**   | Med   | Low   | Low  | -           |

## Problem

There's no automated dependency-update or vulnerability-scanning setup, and at least one
dependency warrants review. Without this, the project drifts onto vulnerable transitive deps and
relies on manual vigilance.

## Evidence

- No Dependabot/Renovate config in the repo, and CI does not run `npm audit` or any SCA step
  (`[.github/workflows/ci.yml](../.github/workflows/ci.yml)`).
- `xlsx@0.18.5` (SheetJS) is pinned in `[package.json](../package.json)`:

```40:41:package.json
    "slugify": "^1.6.6",
    "xlsx": "^0.18.5"
```

  The npm-published `xlsx` has known prototype-pollution / ReDoS advisories historically; SheetJS
  now ships security fixes via their own CDN rather than the npm registry version. Since `xlsx` is
  only used by maintainer-run seed scripts (`scripts/seed-from-excel.ts`,
  `scripts/export-seed-json.ts`), the runtime exposure is low, but it should still be reviewed.

## Impact

- **Latent vulnerabilities** accumulate in transitive dependencies.
- **`xlsx`** specifically: parsing untrusted spreadsheets with an outdated build is risky (low
  here because it's maintainer-only, but worth confirming inputs are trusted).

## Proposed solution

1. **Enable Dependabot** (or Renovate) for npm + GitHub Actions, grouped weekly to limit noise.
2. **Add an audit step** to CI (`npm audit --audit-level=high`, non-blocking at first) or use
   GitHub's dependency review on PRs.
3. **Review `xlsx`:** confirm it's only fed trusted maintainer files; consider installing from the
   SheetJS-recommended source or replacing with a maintained alternative if untrusted input is
   ever possible.
4. Optionally enable CodeQL for basic static analysis.

## Acceptance criteria

- [ ] Automated dependency updates are configured.
- [ ] CI reports high/critical advisories on PRs.
- [ ] The `xlsx` usage/source is reviewed and documented as trusted-input-only (or replaced).
