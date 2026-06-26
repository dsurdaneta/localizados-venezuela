# 04 - No rate limiting on public submission and admin login

| Priority | Value | Effort | Risk | Performance |
| -------- | ----- | ----- | ---- | ----------- |
| **P1**   | High  | Med   | Low  | Med         |

## Problem

Two state-changing endpoints have no rate limiting:

1. **Public contribution** `POST /api/v1/contribuciones` is protected only by reCAPTCHA v3 + a
   honeypot. Neither stops a determined script (reCAPTCHA scores can be farmed; the honeypot is a
   one-liner to bypass). Each accepted "persona" submission **writes a `Localizado` and may create
   a `Lugar`** (see idea `05`), and each "lista_imagen" **writes a file to disk**. So abuse can
   flood the DB and fill the upload volume.
2. **Admin login** `POST /api/admin/auth/login` has no attempt throttling or lockout, allowing
   unlimited brute-force against the admin secret.

The code already computes an `ipHash` for contributions but never uses it for throttling.

## Evidence

Contribution endpoint - only reCAPTCHA + honeypot, then it writes/uploads:

```55:64:src/app/api/v1/contribuciones/route.ts
  const honeypot = String(form.get("website") ?? "").trim();
  if (honeypot) {
    return jsonResponse({ error: "Solicitud rechazada" }, { status: 400 });
  }

  const recaptchaToken = String(form.get("recaptchaToken") ?? "");
  const captcha = await verifyRecaptcha(recaptchaToken);
```

`ipHash` is stored but not used to limit anything:

```23:29:src/app/api/v1/contribuciones/route.ts
function hashIp(req: Request): string {
  const ip =
    req.headers.get("x-forwarded-for")?.split(",")[0]?.trim() ??
    ...
```

Login - no attempt counter / lockout:

```8:20:src/app/api/admin/auth/login/route.ts
export async function POST(req: Request) {
  if (!isAdminConfigured()) {
    return jsonResponse({ error: "ADMIN_SECRET no configurado" }, { status: 503 });
  }
  const body = (await req.json().catch(() => ({}))) as { secret?: string };
  const secret = body.secret?.trim();
  if (!secret || !isValidAdminSecret(secret)) {
    return jsonResponse({ error: "Clave incorrecta" }, { status: 401 });
  }
  ...
```

## Impact

- **DB / disk flooding** via automated contributions (storage exhaustion, moderation queue spam,
  and - combined with idea `05` - junk `Lugar` records).
- **Brute force** against the admin secret with no slowdown.
- **Cost:** each contribution that reaches OCR later also costs OpenAI calls.

## Proposed solution

1. Add an **IP-based rate limiter** (sliding window) to both endpoints. Reuse the existing
   `ipHash` as the key for contributions. For a single-instance deploy an in-memory limiter is
   fine; for multi-instance use a small Mongo TTL collection or Redis.
2. **Login throttling / lockout:** exponential backoff or temporary block after N failed attempts
   per IP; log failures (ties into idea `11`).
3. **Per-IP submission cap** (e.g. N contributions / hour) returning `429` with `Retry-After`.
4. Consider a max upload count/size per IP per window to protect the disk.

## Acceptance criteria

- [ ] Exceeding the contribution limit returns `429` and writes nothing.
- [ ] Repeated bad logins are throttled/locked out and logged.
- [ ] Limits are configurable via env vars and documented in `.env.example`.
