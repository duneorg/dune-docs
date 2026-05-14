---
title: "File Upload"
published: true
visible: true
taxonomy:
  audience: [developer, webmaster]
  difficulty: [intermediate]
  topic: [forms, api, uploads]
metadata:
  description: "Accept file uploads from site visitors via the public upload endpoint"
---

# File Upload

Dune provides a `POST /api/upload` endpoint for accepting file uploads from public site visitors. Uploaded files are stored under `public/uploads/` and served at `GET /uploads/*`.

This is separate from the admin media library — it is intended for visitor-submitted files (avatar uploads, document submissions, etc.).

## Configuration

Enable uploads in `site.yaml`:

```yaml
uploads:
  max_size_mb: 10                # Max file size in MB (default: 10)
  allowed_types:                 # MIME types derived from extension (never trust client-supplied type)
    - "image/jpeg"
    - "image/png"
    - "image/webp"
    - "image/gif"
    - "image/avif"
    - "application/pdf"
  require_auth: false            # Require a logged-in SiteUser (default: false)
```

When `uploads:` is absent from config, the endpoint is not registered.

The server derives the MIME type from the file extension — the `Content-Type` header sent by the client is not trusted for access control decisions.

## Uploading a file

```
POST /api/upload
Content-Type: multipart/form-data

file=<binary data>
```

The file field must be named `file`. Only one file per request is accepted.

Response on success (`200`):

```json
{
  "ok": true,
  "url": "/uploads/3f8a2c1d-e4b9-4f7a-a6b0-1234567890ab.jpg",
  "filename": "3f8a2c1d-e4b9-4f7a-a6b0-1234567890ab.jpg",
  "mimeType": "image/jpeg",
  "size": 48231
}
```

The stored filename is a UUID with the original extension — original filenames are discarded to prevent path traversal.

### Error responses

| Status | Reason |
|--------|--------|
| `401` | `require_auth: true` and no valid site user session |
| `413` | File exceeds `max_size_mb` |
| `415` | File extension maps to a MIME type not in `allowed_types` |
| `400` | No `file` field in the form data |

## Accessing uploaded files

Files are stored at `public/uploads/{uuid}.{ext}` in the site root and served at `/uploads/{uuid}.{ext}` with no authentication required.

The returned `url` field can be saved to a Flex Object record, a form submission, or any other data store. Use it directly in `<img>` tags or links.

## Example: avatar upload form

```html
<form action="/api/upload" method="POST" enctype="multipart/form-data">
  <input type="file" name="file" accept="image/jpeg,image/png,image/webp" />
  <button type="submit">Upload avatar</button>
</form>
```

From JavaScript:

```js
async function uploadFile(file) {
  const form = new FormData();
  form.append("file", file);

  const res = await fetch("/api/upload", { method: "POST", body: form });
  if (!res.ok) throw new Error(`Upload failed: ${res.status}`);
  const { url } = await res.json();
  return url;  // e.g. "/uploads/3f8a2c1d.jpg"
}
```

## Blueprint `file` field type

Forms defined with blueprints support a `file` field type that renders a file input and stores the uploaded URL:

```yaml
# schemas/profile.yaml
title: Profile
fields:
  name:
    type: text
    label: Name
    required: true
  avatar:
    type: file
    label: Avatar
    accept: "image/*"
    max_size_mb: 5
```

When a blueprint form with a `file` field is submitted, Dune uploads the file to `/api/upload` first, then stores the returned URL as the field value in the form submission record.
