# 07 - No caching on public reads; every request hits MongoDB

| Priority | Value | Effort | Risk | Performance |
| -------- | ----- | ----- | ---- | ----------- |
| **P1**   | High  | Med   | Med  | High        |

## Problem

The public read paths do no caching. The search page forces dynamic rendering, and the v1 API
sets no `Cache-Control`. Every visitor - including bursts of people checking on the same
hospitals during a disaster - triggers fresh MongoDB queries (and aggregations). This is the
classic "thundering herd at the worst moment" risk for a humanitarian site that gets sudden,
correlated traffic.

## Evidence

`/buscar` is forced fully dynamic:

```1:1:src/app/buscar/page.tsx
export const dynamic = "force-dynamic";
```

The v1 list API returns data with CORS headers but no cache directives:

```42:48:src/lib/api.ts
export function corsJson(data: unknown, init?: ResponseInit) {
  const headers = new Headers(init?.headers);
  headers.set("Access-Control-Allow-Origin", "*");
  headers.set("Access-Control-Allow-Methods", "GET, OPTIONS");
  headers.set("Access-Control-Allow-Headers", "Content-Type");
  return NextResponse.json(data, { ...init, headers });
}
```

`listLugares()` and `getLugarBySlug()` run aggregations on every call
(`[src/lib/queries.ts](../src/lib/queries.ts)`), with no memoization.

## Impact

- **Load resilience:** read load scales linearly with traffic; a spike can saturate Mongo.
- **Latency:** repeated identical queries pay full DB cost each time.
- **Cost:** more DB CPU/IO than necessary on what is mostly read-only, slowly-changing data.

## Proposed solution

Layer caching appropriate to how fresh data must be (moderation cadence is minutes, not seconds):

1. **API responses:** add `Cache-Control: public, s-maxage=<n>, stale-while-revalidate=<m>` to
   `/api/v1/*` GETs (e.g. `s-maxage=60`), so a CDN/proxy can absorb repeats. Vary by query string.
2. **Pages:** replace `force-dynamic` on listing pages with **ISR** (`export const revalidate =
   60`) where possible, or segment-level caching. Person/place pages change rarely - good ISR
   candidates with on-demand revalidation when a moderator publishes.
3. **`/lugares` index:** cache the aggregated counts (revalidate on a timer or on publish).
4. Optionally add **on-demand revalidation** (`revalidatePath`) from admin publish/move actions
   so updates appear promptly without abandoning caching.

Be mindful: if idea `01` (masking) lands, ensure no PII is cached at a shared CDN beyond what's
intended to be public.

## Acceptance criteria

- [ ] `/api/v1/*` GETs send sensible `Cache-Control` headers.
- [ ] Listing/detail pages are served from cache/ISR instead of always hitting Mongo.
- [ ] Publishing in admin makes updated data appear within the chosen revalidation window.
- [ ] Load test shows reduced DB query volume under repeated identical requests.
