# 11 - No observability: error logging, monitoring, or audit trail

| Priority | Value | Effort | Risk | Performance |
| -------- | ----- | ----- | ---- | ----------- |
| **P2**   | Med   | Med   | Low  | -           |

## Problem

There is no structured logging, error monitoring, or audit trail. When something fails in
production (DB connection issues per idea `02`, OCR/OpenAI failures, moderation errors), there's no
signal beyond whatever lands in stdout, and no way to know it happened. For a service that must be
trustworthy and available during emergencies, "we find out when a user complains" is too late.

## Evidence

- API errors are returned to the client but not logged/aggregated, e.g. the OCR route just maps
  the error to a 500 message:

```75:78:src/app/api/admin/contribuciones/[id]/ocr/route.ts
  } catch (err) {
    const msg = err instanceof Error ? err.message : "Error OCR";
    return jsonResponse({ error: msg }, { status: 500 });
  }
```

- No monitoring dependency or config exists (`[package.json](../package.json)`), and there's no
  audit record of who published/deleted what (related to idea `03`).

## Impact

- **Blind operations:** outages and recurring failures go unnoticed.
- **No forensics:** can't investigate bad/destructive moderation actions after the fact.
- **Hard debugging:** OpenAI/OCR and DB failures lack context (request id, inputs, timing).

## Proposed solution

1. **Structured logging** with levels and request context (a small logger util; log warn/error on
   caught exceptions across API routes instead of swallowing them).
2. **Error monitoring** (e.g. Sentry) for server and client, gated behind an env var so it's
   optional for local dev. Capture OCR/OpenAI and DB-connect failures explicitly.
3. **Moderation audit log:** persist `{ moderatorId, action, targetId, timestamp }` for
   publish/reject/delete/move (pairs with idea `03`'s per-moderator identity).
4. Optional: a lightweight `/api/health` (or `/api/admin/health`) endpoint reporting DB
   connectivity for uptime checks.

## Acceptance criteria

- [ ] Server-side exceptions are logged with context, not silently dropped.
- [ ] An optional monitoring integration captures errors in production.
- [ ] Destructive moderation actions leave an audit record.
