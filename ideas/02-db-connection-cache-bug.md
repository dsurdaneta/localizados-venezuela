# 02 - MongoDB connection cache wedges the app after one failed connect

| Priority | Value | Effort | Risk | Performance |
| -------- | ----- | ------ | ---- | ----------- |
| **P0**   | High  | Low    | Low  | High        |

## Problem

The Mongoose connection helper caches the **connect promise** so it's reused across requests
(the standard Next.js pattern). But it never clears that promise if the connection **fails**.
A single transient failure (DB restart, network blip, cold start before Mongo is up) caches a
**rejected promise forever**, so every subsequent request re-awaits the same rejection until the
process is restarted. For a disaster-response site that must stay up exactly when load and
infra stress are highest, this turns a momentary blip into a hard outage.

## Evidence

```22:33:src/lib/db.ts
export async function connectDB(): Promise<typeof mongoose> {
  if (cached.conn) return cached.conn;

  if (!cached.promise) {
    cached.promise = mongoose.connect(MONGODB_URI, {
      bufferCommands: false,
    });
  }

  cached.conn = await cached.promise;
  return cached.conn;
}
```

If `mongoose.connect(...)` rejects, `cached.promise` keeps holding the rejected promise. The next
call sees `cached.promise` is truthy, skips reconnecting, and awaits the same rejection again.

## Impact

- **Reliability:** one transient DB error escalates to a full, persistent outage requiring a
  manual restart.
- **Worst timing:** most likely to trigger during traffic spikes / infra pressure - exactly when
  the site is most needed.

## Proposed solution

Reset the cached promise on failure so the next request can retry, and optionally add timeouts:

```ts
export async function connectDB(): Promise<typeof mongoose> {
  if (cached.conn) return cached.conn;

  if (!cached.promise) {
    cached.promise = mongoose
      .connect(MONGODB_URI, {
        bufferCommands: false,
        serverSelectionTimeoutMS: 5000,
      })
      .catch((err) => {
        cached.promise = null; // allow retry on next request
        throw err;
      });
  }

  try {
    cached.conn = await cached.promise;
  } catch (err) {
    cached.promise = null;
    throw err;
  }
  return cached.conn;
}
```

Optionally surface a clean 503 in API routes when `connectDB()` throws, instead of an unhandled
error, so clients get a retryable response.

## Acceptance criteria

- [ ] After a simulated connect failure, a later request succeeds once the DB is reachable again
      (no restart needed).
- [ ] `cached.promise` is nulled on rejection.
- [ ] A connection timeout is set so requests fail fast instead of hanging.
