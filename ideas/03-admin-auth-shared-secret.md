# 03 - Admin auth uses a shared static secret as the session token

| Priority | Value | Effort | Risk | Performance |
| -------- | ----- | ------ | ---- | ----------- |
| **P0**   | High  | Med    | Med  | -           |

## Problem

The moderation panel authenticates with a single shared `ADMIN_SECRET` (or a comma-separated
list of secrets). On login, the **raw secret itself is stored as the cookie value**. This means:

- The long-lived credential travels on every request and lives in the browser cookie jar; any
  cookie leak (XSS, shared device, misconfigured proxy log) discloses the actual master secret,
  not a revocable session token.
- There is **no per-moderator identity** - the audit fields `moderadoPor` / `deletedBy` can't be
  reliably attributed when everyone shares one secret.
- There is **no server-side revocation or expiry**. Rotating a leaked secret means changing the
  env var and re-deploying, which logs out everyone and invalidates nothing granularly.
- Multiple moderators are modeled as multiple plaintext secrets in one env var.

For a tool where moderators can publish, bulk-delete, and move victim records, this is a
P0-level weakness.

## Evidence

The raw secret is set directly as the cookie value:

```51:64:src/lib/admin-auth.ts
export function setAdminCookieResponse(
  secret: string,
  body: unknown = { ok: true }
): NextResponse {
  const res = NextResponse.json(body);
  res.cookies.set(ADMIN_COOKIE, secret, {
    httpOnly: true,
    secure: process.env.NODE_ENV === "production",
    sameSite: "lax",
    path: "/",
    maxAge: 60 * 60 * 24 * 7,
  });
  return res;
}
```

Validation just compares the cookie/header against the configured secret(s):

```22:24:src/lib/admin-auth-core.ts
export function isValidAdminSecret(secret: string): boolean {
  return getAdminSecrets().some((s) => safeEqual(s, secret));
}
```

Secrets are split from one env var:

```5:10:src/lib/admin-auth-core.ts
export function getAdminSecrets(): string[] {
  return (process.env.ADMIN_SECRET ?? "")
    .split(",")
    ...
```

(Credit where due: `safeEqual` already uses `timingSafeEqual` over SHA-256 digests, so the
comparison itself is constant-time. The problem is the credential model, not the compare.)

## Impact

- **Credential leakage blast radius:** leaking one cookie = leaking a master credential with no
  cheap rotation.
- **No accountability:** can't tell which moderator made a destructive change.
- **Operational pain:** rotating/revoking access is all-or-nothing and requires redeploy.

## Proposed solution

Move from "secret-as-cookie" to a proper session model. Incremental options, cheapest first:

1. **Minimum (low effort):** issue a **signed, short-lived session token** (e.g. HMAC-signed JWT
   or signed cookie) on login instead of storing the raw secret. The secret stays server-side;
   the cookie carries only a signed token with an expiry. Validate the signature in middleware.
2. **Better (medium effort):** introduce lightweight **per-moderator accounts** (id + hashed
   secret/password, stored in Mongo or a small config), so sessions carry a moderator id that
   populates `moderadoPor` / `deletedBy`.
3. **Add server-side controls:** session expiry, the ability to invalidate sessions (e.g. a
   signing-key bump or a token version), and rotate without leaking other moderators.
4. **Audit log:** record moderator id + action + target on publish/delete/move (ties into idea
   `11-observability`).

Keep the existing `Authorization: Bearer` machine path for scripts, but back it with a separate,
clearly-scoped token rather than the human session cookie.

## Acceptance criteria

- [ ] The session cookie no longer contains the raw `ADMIN_SECRET`.
- [ ] Sessions are signed and expire; a leaked cookie can be invalidated without rotating the
      master secret for everyone.
- [ ] Moderation actions record an attributable moderator identity.
- [ ] Login still rejects when the panel is unconfigured (current 503 behavior preserved).
