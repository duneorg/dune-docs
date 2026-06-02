---
title: "Transactional Email"
published: true
visible: true
taxonomy:
  audience: [developer]
  difficulty: [intermediate]
  topic: [extending, email]
metadata:
  description: "Send transactional emails from plugins, route handlers, and templates using Dune's email abstraction"
---

# Transactional Email

Dune's email module provides a `send()` API for sending transactional emails from plugins, route handlers, and server-side code. It supports multiple delivery providers and a template system backed by the same content formats Dune uses for pages (Markdown, MDX, TSX).

This is separate from the admin form submission notifications configured under `admin.notifications.email`.

## Configuration

```yaml
# site.yaml
email:
  provider: "resend"             # smtp | resend | postmark | sendgrid | console
  from: "hello@example.com"     # Default From address

  resend:
    apiKey: "$RESEND_API_KEY"

  # SMTP alternative:
  smtp:
    host: "smtp.example.com"
    port: 587
    secure: false                # true = TLS on 465, false = STARTTLS on 587
    user: "$SMTP_USER"
    pass: "$SMTP_PASS"

  # Postmark:
  postmark:
    apiKey: "$POSTMARK_API_KEY"

  # SendGrid:
  sendgrid:
    apiKey: "$SENDGRID_API_KEY"
```

When `email:` is omitted from config, the `console` provider is used — emails are logged to stdout. This is the default in development.

## Sending an email

Import `email` from `@dune/core/email`:

```ts
import { email } from "@dune/core/email";

await email.send({
  to: "user@example.com",
  subject: "Your order is confirmed",
  text: "Thanks for your purchase!",
  html: "<p>Thanks for your purchase!</p>",
});
```

All fields except `to` and `subject` are optional. When `from` is omitted, the configured default `from` address is used.

### `send()` options

| Field | Type | Description |
|-------|------|-------------|
| `to` | `string \| string[]` | Recipient address(es). Required. |
| `subject` | `string` | Email subject. Required. |
| `from` | `string` | Override the default From address. |
| `replyTo` | `string` | Reply-To address. |
| `text` | `string` | Plain-text body. |
| `html` | `string` | HTML body. |
| `template` | `string` | Template name — loads from `emails/` directory (see below). |
| `data` | `Record<string, unknown>` | Data passed to the template. |
| `attachments` | `Array<{filename, content, mimeType}>` | File attachments. |

## Email templates

Templates live in `emails/` at your site root. Dune looks for files named `{template}.email.md`, `{template}.email.mdx`, or `{template}.email.tsx`.

### Markdown template

```markdown
<!-- emails/welcome.email.md -->
---
subject: "Welcome to {{site.title}}"
---

Hi {{data.name}},

Thanks for signing up! Your account is ready.

[Log in]({{site.url}}/auth/login)
```

### TSX template

```tsx
// emails/order-confirmed.email.tsx
export default function OrderConfirmed({ data, site }: { data: { orderId: string }; site: { title: string } }) {
  return (
    <div>
      <h1>Order confirmed</h1>
      <p>Order #{data.orderId} is on its way.</p>
    </div>
  );
}
```

### Sending with a template

```ts
await email.send({
  to: "user@example.com",
  template: "welcome",
  data: { name: "Alice" },
});
```

The subject line is taken from the template's frontmatter `subject` field. If the template defines a `subject`, the `send()` call does not need one.

## Using in plugins

```ts
import { email } from "@dune/core/email";
import type { DunePlugin } from "@dune/core/plugins";

const plugin: DunePlugin = {
  name: "welcome-email",
  setup(hooks) {
    hooks.on("onSiteUserCreated", async (user) => {
      await email.send({
        to: user.email,
        template: "welcome",
        data: { name: user.name ?? user.email },
      });
    });
  },
};
```

## Provider notes

**Console (default)** — Logs the email to stdout. Subject, to, and a text preview are printed. No email is actually sent. Useful in development.

**Resend** — Uses the Resend HTTP API. Requires a verified sending domain in the Resend dashboard.

**Postmark** — Uses the Postmark API. Requires a verified sender signature.

**SendGrid** — Uses the SendGrid Mail Send API. Requires API key with Mail Send permission.

**SMTP** — Uses a raw SMTP connection. `secure: true` uses implicit TLS (port 465); `secure: false` upgrades with STARTTLS (port 587). Set `pass: "$SMTP_PASS"` to avoid committing credentials.

## Dev-mode email preview

In development (`DUNE_ENV=dev`), Dune intercepts all outgoing emails and writes them to `.dune/admin/dev-email/` as JSON files instead of sending them. This applies to all providers, including production ones — no emails escape to real recipients during local development.

Browse intercepted emails in the admin panel under **Dev → Email Preview** (`/admin/email-preview`), or query the API directly:

```
GET /admin/api/email-preview
→ { "emails": [ { "to", "subject", "template", "timestamp", "id" }, … ] }

GET /admin/api/email-preview/{id}
→ { "to", "subject", "html", "text", "template", "data", "timestamp" }
```

Both endpoints require admin authentication and return `404` outside of dev mode.
