---
title: "Forms"
published: true
visible: true
taxonomy:
  audience: [webmaster, developer]
  difficulty: [intermediate]
  topic: [forms, content]
metadata:
  description: "Blueprint-driven contact forms with validation, file uploads, and email/webhook notifications"
---

# Forms

Dune includes a blueprint-driven form system. You define forms as YAML files and the engine handles validation, spam protection, file uploads, email notifications, and submission storage — no backend code required.

Forms are part of Dune itself. They work with the admin panel turned off (`admin.enabled: false`) or without an admin plugin at all; the admin panel only adds a screen for reading the submissions.

## Creating a form

Create a YAML file in the `forms/` directory at your site root:

```
forms/
  contact.yaml
  job-application.yaml
  newsletter.yaml
```

Each file defines one form. The filename (without extension) is the form's identifier.

### Basic example

```yaml
# forms/contact.yaml
title: "Contact Us"
success_url: "/contact/thanks"
honeypot: _email_confirm   # hidden field — bots fill it, humans don't

fields:
  name:
    type: text
    label: "Your name"
    required: true
    validate:
      min: 2

  email:
    type: email
    label: "Email address"
    required: true

  subject:
    type: select
    label: "Subject"
    required: true
    options:
      general: "General enquiry"
      support: "Technical support"
      billing: "Billing question"

  message:
    type: textarea
    label: "Message"
    required: true
    validate:
      min: 20
      max: 5000
```

### With notifications

```yaml
# forms/contact.yaml
title: "Contact Us"
success_url: "/thanks"

notifications:
  email: "hello@example.com"        # override To: for this form only
  webhook: "https://hooks.example.com/contact"   # POST JSON payload

fields:
  # ...
```

The `notifications.email` value overrides the `To:` address for this specific form. The webhook URL replaces the site-wide one but keeps its signing secret.

Site-wide notifications are set in `site.yaml`. The email is sent through the provider you configured under [`email`](/docs/extending/email), so it can use SMTP, Resend, Postmark or SendGrid:

```yaml
# config/site.yaml
email:
  provider: resend
  from: hello@example.com
  resend:
    apiKey: "$RESEND_API_KEY"

forms:
  honeypot: _hp                      # default; a form's own `honeypot` wins
  notifications:
    email:
      to: hello@example.com          # string or list
      subject: "New {form} submission"   # optional; {form} is the form name
    webhook:
      url: https://hooks.example.com/forms
      secret: "$FORM_WEBHOOK_SECRET" # signs the body: X-Dune-Signature: sha256=<hex>
```

Sending failures are logged and never shown to the person submitting the form.

> **Moved from `admin:`.** `admin.honeypot` and `admin.notifications` still work and log a deprecation warning. A `notifications.email` block with its own `smtp:` credentials also keeps working with a warning; move the credentials to `email:` instead.

## Field types

| Type | Description |
|------|-------------|
| `text` | Single-line text input |
| `email` | Email address (validated format) |
| `tel` | Phone number |
| `textarea` | Multi-line text |
| `number` | Numeric input |
| `select` | Dropdown — requires `options` map |
| `checkbox` | Boolean on/off |
| `file` | File upload |
| `hidden` | Hidden field with a fixed value |

## Field options

| Option | Types | Description |
|--------|-------|-------------|
| `label` | all | Human-readable label |
| `required` | all | Field must be present and non-empty |
| `placeholder` | text-like | Placeholder text, returned in the schema |
| `validate.min` | text, email, tel, textarea / number | Minimum length / minimum value |
| `validate.max` | text, email, tel, textarea / number | Maximum length / maximum value |
| `validate.pattern` | text, email, tel | Regex the value must match |
| `options` | select | Map of `value: label` pairs |

## The API endpoint

Each form in `forms/` is automatically served at two endpoints:

| Method | URL | Description |
|--------|-----|-------------|
| `GET` | `/api/forms/{name}` | Returns the public form schema as JSON |
| `POST` | `/api/forms/{name}` | Submits the form |

`POST /api/contact` is deprecated. Define `forms/contact.yaml` and post to `/api/forms/contact` instead.

To react to a stored submission from code, register an `onFormSubmit` hook; see [Hooks](/docs/extending/hooks).

### GET — fetch schema

```http
GET /api/forms/contact
```

```json
{
  "name": "contact",
  "title": "Contact Us",
  "success_url": "/thanks",
  "honeypot": "_email_confirm",
  "fields": {
    "name": { "type": "text", "label": "Your name", "required": true, "validate": { "min": 2 } },
    "email": { "type": "email", "label": "Email address", "required": true },
    "message": { "type": "textarea", "label": "Message", "required": true }
  }
}
```

Sensitive keys (`notifications.email`, `notifications.webhook`) are omitted from the public schema. An unknown form and a disabled form both return `404`.

### POST — submit

Send `multipart/form-data` (required for file uploads) or `application/x-www-form-urlencoded`:

```html
<form method="POST" action="/api/forms/contact" enctype="multipart/form-data">
  <input type="text" name="name" required>
  <input type="email" name="email" required>
  <textarea name="message" required></textarea>
  <!-- Honeypot: hide this with CSS, don't remove it -->
  <input type="text" name="_email_confirm" style="display:none" tabindex="-1" autocomplete="off">
  <button type="submit">Send</button>
</form>
```

A browser form post and a `fetch` call get different responses. Send `Accept: application/json` to get JSON.

**Success.** A form post gets `302 Found` to `success_url`. Only a path on your own site is followed; anything else falls back to `/`. With `Accept: application/json` you get `200`:

```json
{ "ok": true }
```

**Validation error.** With `Accept: application/json`, `422 Unprocessable Entity`:

```json
{
  "error": "Validation failed",
  "errors": [
    { "field": "email", "message": "Invalid email address" },
    { "field": "message", "message": "Must be at least 20 characters" }
  ]
}
```

A form post is redirected back to the page it came from with `?form_error=1` added.

**Rate limit** (`429 Too Many Requests`, with a `Retry-After` header):

```json
{ "error": "Too many requests" }
```

**Other errors.** A form that doesn't exist is `404`. A form with `enabled: false` is `403` on submit. A body over 55 MB is `413`.

## Accessing submissions

Submissions are stored as JSON in `data/submissions/{form-name}/`. The directory is `data_dir` in `system.yaml` (default `data`), and it is meant to be git-tracked so submissions survive deploys:

```yaml
# config/system.yaml
data_dir: data
```

> `admin.dataDir` still works and logs a deprecation warning.

Each submission records:

- Timestamp
- Submitter IP address, language and user agent
- All submitted field values
- Any uploaded file references

With the admin panel enabled, submissions are listed under **Form Submissions**. Without it, read the JSON files directly, or react to new ones in a plugin with the `onFormSubmit` hook (see [Hooks](/docs/extending/hooks)).

## Spam protection

Dune applies two spam prevention layers:

**Honeypot field**: Add a `honeypot` key to your form definition (or set `forms.honeypot` in `site.yaml` for every form; the default is `_hp`) and render a hidden input with that name. Automated bots typically fill every field — if the honeypot field is non-empty, the submission is silently dropped and the response looks like a success, so bots learn nothing.

**Rate limiting**: Each IP can submit 5 times per minute across all forms. Set `system.trusted_proxies: true` if Dune runs behind a reverse proxy, so the real client address is used rather than the proxy's.

## File uploads

Add a `file` field to accept attachments:

```yaml
fields:
  resume:
    type: file
    label: "Upload your CV"
    required: true
```

Uploaded files are stored in `{data_dir}/uploads/{form}/{submission-id}/` and referenced from the submission record. The admin panel displays uploaded files with download links.

### Allowed file types

Dune enforces a server-side extension allowlist. Only the following types are accepted:

**Images:** `.jpg`, `.jpeg`, `.png`, `.gif`, `.webp`, `.avif`  
**Documents:** `.pdf`, `.txt`, `.csv`  
**Office:** `.doc`, `.docx`, `.xls`, `.xlsx`, `.ppt`, `.pptx`  
**Archives:** `.zip`

Script, executable, and HTML/SVG files (`.php`, `.sh`, `.exe`, `.html`, `.svg`, `.js`, etc.) are skipped: the submission is still saved, without that file, and the submitter is not told. The same goes for files over 10 MB and for any file past the fifth. The stored content-type is derived from the file extension rather than the browser-supplied type.

Form submissions larger than 55 MB are rejected before the body is parsed (`413 Request Too Large`).

## Using forms with JavaScript

For single-page applications or fetch-based submissions, you can use the JSON API directly:

```javascript
// Fetch the form schema
const schema = await fetch("/api/forms/contact").then(r => r.json());

// Build FormData from user input
const body = new FormData();
body.append("name", "Jane Smith");
body.append("email", "jane@example.com");
body.append("message", "Hello from the JS client!");

// Submit
const res = await fetch("/api/forms/contact", {
  method: "POST",
  body,
  headers: { Accept: "application/json" },
});

if (res.ok) {
  // Success — redirect to success_url or show inline message
  window.location.href = schema.success_url ?? "/";
} else if (res.status === 422) {
  const { errors } = await res.json();
  // Display field errors
}
```

## Disabling a form

Remove the YAML file to delete a form: its endpoint returns `404`. To pause one without deleting it, set `enabled: false`:

```yaml
# forms/contact.yaml
enabled: false
```

A paused form's schema returns `404` (the same as a form that doesn't exist) and submissions are refused with `403`. Existing submissions stay where they are.

## The legacy contact endpoint

`POST /api/contact` is deprecated and logs a warning the first time it is used. It stores whatever it receives under the form name in `form_name` (default `contact`) and only checks that there is a `name` or an `email`. Define `forms/contact.yaml` and post to `/api/forms/contact` instead, so the submission is validated.
