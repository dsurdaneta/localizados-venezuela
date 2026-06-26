# 09 - Image upload hardening

| Priority | Value | Effort | Risk | Performance |
| -------- | ----- | ----- | ---- | ----------- |
| **P2**   | Med   | Med   | Low  | -           |

## Problem

The contribution image upload trusts the **client-declared MIME type**, does no content
(magic-byte) validation, doesn't strip metadata, and writes files into `public/uploads`, which
is served as static content from the web root. An attacker can set `file.type` to an allowed
value while uploading arbitrary bytes, and uploaded files are then publicly reachable by URL.
Uploaded photos of listings may also carry **EXIF GPS/metadata** that gets republished.

## Evidence

Validation relies on the client-provided `file.type`:

```31:39:src/app/api/v1/contribuciones/route.ts
function validateImage(file: File): string | null {
  if (!ALLOWED_IMAGE_TYPES.has(file.type)) {
    return "Formato de imagen no permitido (JPEG, PNG, GIF o WebP)";
  }
  if (file.size > MAX_IMAGE_BYTES) {
    return "Imagen demasiado grande (máximo 10 MB)";
  }
  return null;
}
```

Files are written under the public web root and served back by path:

```87:101:src/app/api/v1/contribuciones/route.ts
    await mkdir(UPLOAD_DIR, { recursive: true });
    const safeName = `${Date.now()}-${file.name.replace(/[^a-zA-Z0-9._-]/g, "_")}`;
    const buffer = Buffer.from(await file.arrayBuffer());
    const fullPath = path.join(UPLOAD_DIR, safeName);
    await writeFile(fullPath, buffer);
    ...
      imagenPath: `/uploads/${safeName}`,
```

(The filename is sanitized, which is good - path traversal is mitigated. The gaps are content
validation, metadata, and serving location.)

## Impact

- **Content spoofing:** non-image payloads can be stored and served from your domain.
- **Privacy:** EXIF GPS/device metadata in uploaded photos can leak location/identity.
- **Hosting abuse:** the upload dir can be used as arbitrary public file hosting.

## Proposed solution

1. **Validate by content, not header:** sniff magic bytes / decode the image server-side; reject
   anything that doesn't actually decode as an allowed raster format.
2. **Re-encode and strip metadata** (e.g. via `sharp`): normalize to JPEG/PNG/WebP, cap
   dimensions, and drop EXIF. This neutralizes embedded payloads and removes GPS data.
3. **Serve uploads from outside the web root** (or via an authenticated/admin-only route until
   moderated), so raw public submissions aren't instantly world-readable.
4. Keep the size cap; add an explicit max-pixel guard to prevent decompression-bomb images.

## Acceptance criteria

- [ ] A file with a spoofed MIME type that isn't a real image is rejected.
- [ ] Stored images are re-encoded with metadata stripped.
- [ ] Unmoderated uploads are not directly world-readable by guessable URL (or are clearly
      accepted as public by policy).
