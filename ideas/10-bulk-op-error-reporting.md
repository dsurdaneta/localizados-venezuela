# 10 - Bulk moderation operations swallow errors and do N+1 writes

| Priority | Value | Effort | Risk | Performance |
| -------- | ----- | ----- | ---- | ----------- |
| **P2**   | Med   | Low   | Low  | Med         |

## Problem

The bulk admin endpoint's `publish` and `reject` actions loop over ids one at a time, wrap each in
a `try/catch` that **silently discards failures**, and only report a count of successes. A
moderator who selects 50 records and sees "affected: 30" has no idea which 20 failed or why
(commonly the unique published-dedup index rejecting a duplicate). It's also N round-trips to the
DB instead of one operation.

## Evidence

```40:63:src/app/api/admin/localizados/bulk/route.ts
      case "publish": {
        let n = 0;
        for (const id of ids) {
          try {
            await updateLocalizado(id, { estado: "published" });
            n++;
          } catch {
            // skip dupes
          }
        }
        return jsonResponse({ ok: true, affected: n });
      }
      case "reject": {
        let n = 0;
        for (const id of ids) {
          try {
            await updateLocalizado(id, { estado: "rejected" });
            n++;
          } catch {
            // skip
          }
        }
        return jsonResponse({ ok: true, affected: n });
      }
```

## Impact

- **Moderator confusion:** failures are invisible; records appear "stuck" pending with no reason.
- **Hidden data issues:** dedup-index collisions (a real signal worth surfacing) are swallowed.
- **Inefficiency:** N sequential DB calls per bulk action.

## Proposed solution

1. **Return a per-id result**: `{ ok, succeeded: [...], failed: [{ id, reason }] }` so the UI can
   show exactly what failed and why (e.g. "duplicate of an already-published record").
2. Surface those failures in the `AdminPanel` UI instead of just a count.
3. Where safe (reject, soft-delete), use a single `updateMany`; keep per-id handling only where
   business rules require it (publish, due to the unique dedup index), but still collect reasons.

## Acceptance criteria

- [ ] Bulk publish/reject returns which ids failed and a reason.
- [ ] The admin UI displays failures, not just a success count.
- [ ] Operations that can be batched use a single DB call.
