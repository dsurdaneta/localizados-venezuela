# Ideas Backlog - localizados-venezuela

A Staff-Engineer review of the project, turned into an actionable, prioritized backlog.
Each idea lives in its own file using a consistent template so it can become a GitHub issue
or a small, reviewable PR.

> Context that shapes the priorities: this is a **disaster-response tool** handling the
> **personal data of victims** (names, cédulas, phones, addresses) and is expected to take
> **traffic spikes** right when it matters most. Therefore **privacy** and **reliability**
> are weighted more heavily than features.

## How to read the scores

Each idea is rated on four axes, plus a derived priority.

| Axis            | Meaning                                                                 |
| --------------- | ----------------------------------------------------------------------- |
| **Value**       | Benefit to users/operators (safety, privacy, correctness, UX).          |
| **Effort**      | Engineering cost to implement well (Low = hours, Med = 1-3 days, High = week+). |
| **Risk**        | Chance the change breaks something or has unintended consequences.      |
| **Performance** | Impact on latency / load resilience (n/a if not relevant).              |

**Priority** is derived, not just averaged. The rule of thumb:

- **P0** - Safety, privacy, or correctness defect that can cause real-world harm, data
  leakage, or full outage. Do first regardless of effort.
- **P1** - High value with reasonable effort; protects the service under load or against abuse.
- **P2** - Solid improvement to quality, security depth, or UX; not blocking.
- **P3** - Strategic / nice-to-have; do when capacity allows.

When two ideas tie, prefer the one with **lower effort** and **lower risk** (faster payback).

## Prioritized backlog

**Progress** tracks implementation status: `Not started` -> `In progress` -> `In review` -> `Done` (use `Blocked` when stuck).

| #   | Idea                                                          | Priority | Value | Effort | Risk | Perf | Progress    |
| --- | ------------------------------------------------------------ | -------- | ----- | ------ | ---- | ---- | ----------- |
| 01  | [PII exposure in public API](01-pii-exposure-public-api.md)  | **P0**   | High  | Med    | Med  | -    | Not started |
| 02  | [DB connection cache wedge](02-db-connection-cache-bug.md)   | **P0**   | High  | Low    | Low  | High | Not started |
| 03  | [Admin auth: shared static secret](03-admin-auth-shared-secret.md) | **P0** | High | Med  | Med  | -    | Not started |
| 04  | [Rate limiting & abuse protection](04-rate-limiting-abuse.md) | **P1**   | High  | Med    | Low  | Med  | Not started |
| 05  | [Public form creates Lugares](05-public-form-creates-lugares.md) | **P1** | Med | Low    | Low  | -    | Not started |
| 06  | [Cédula search can't use index](06-cedula-search-index.md)   | **P1**   | Med   | Low    | Low  | High | Not started |
| 07  | [No caching on public reads](07-public-read-caching.md)      | **P1**   | High  | Med    | Med  | High | Not started |
| 08  | [No automated tests](08-automated-tests.md)                  | **P1**   | High  | Med    | Low  | -    | Not started |
| 09  | [Upload hardening](09-upload-hardening.md)                   | **P2**   | Med   | Med    | Low  | -    | Not started |
| 10  | [Bulk-op error reporting](10-bulk-op-error-reporting.md)     | **P2**   | Med   | Low    | Low  | Med  | Not started |
| 11  | [Observability & error logging](11-observability.md)         | **P2**   | Med   | Med    | Low  | -    | Not started |
| 12  | [Search/listing pagination UX](12-search-pagination-ux.md)   | **P2**   | Med   | Low    | Low  | -    | Not started |
| 13  | [Dedup / fuzzy matching](13-dedup-fuzzy-matching.md)         | **P3**   | Med   | High   | Med  | -    | Not started |
| 14  | [Dependency & supply-chain hygiene](14-dependency-supply-chain.md) | **P3** | Med | Low | Low  | -    | Not started |
| 15  | [Generated OpenAPI docs](15-api-openapi-docs.md)             | **P3**   | Low   | Med    | Low  | -    | Not started |

## Suggested execution order

1. **Quick safety wins first:** `02` (one-line resilience fix), `06` (index fix), `05` (stop polluting data).
2. **Privacy hardening:** `01` then `03` - these are the highest-stakes items for a victim registry.
3. **Survive the spike:** `07` (caching) and `04` (rate limiting) before the next high-traffic event.
4. **Lock in quality:** `08` (tests) so the above changes don't regress.
5. **Then depth & polish:** `09`-`12`, followed by strategic `13`-`15`.

## Notes

- These documents are analysis/recommendations only; no application code is changed in this branch.
- File/line references point at `main` as reviewed; confirm before implementing.
