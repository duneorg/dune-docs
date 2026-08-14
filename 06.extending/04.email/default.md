---
title: "Transactional Email"
published: true
visible: true
taxonomy:
  audience: [developer]
  difficulty: [intermediate]
  topic: [extending, email]
metadata:
  description: "Send transactional emails from plugins, route handlers, and background jobs using Dune's email abstraction"
---

# Transactional Email

Dune's email module provides an `EmailClient.send()` API for sending transactional emails, with pluggable delivery providers and a small template system (Markdown, MDX-as-Markdown, or TSX). There is no ambient, pre-configured client — you construct one from config, except inside background jobs, which receive one automatically.

This is separate from the admin form submission notifications configured under `admin.notifications.email`.

## Configuration

```yaml
# site.yaml — top level, not nested under a "site:" key
email:
  provider: "resend"             # console (default) | smtp | resend | postmark | sendgrid
  from: "hello@example.com"      # Default From address

  resend:
    apiKey: "$RESEND_API_KEY"

  # SMTP alternative:
  smtp:
    host: "smtp.example.com"
    port: 587
    secure: false                # true = implicit TLS on 465, false = STARTTLS on 587
    user: "$SMTP_USER"
    pass: "$SMTP_PASS"

  # Postmark:
  postmark:
    apiKey: "$POSTMARK_API_KEY"

  # SendGrid:
  sendgrid:
    apiKey: "$SENDGRID_API_KEY"
```

When `email:` is omitted, or the selected provider is missing its required credentials, `createEmailProvider()` falls back to the console provider (logs to stdout, does not send). This fallback is driven entirely by config, not by an environment variable — see "Dev-mode email preview" below for why that distinction matters.

## Sending an email

There is no `import { email } from "@dune/core/email"` — no such export exists. Build a client from a provider:

```ts
import { createEmailClient, createEmailProvider } from "@dune/core/email";

const provider = createEmailProvider({
  provider: "resend",
  from: "hello@example.com",
  resend: { apiKey: Deno.env.get("RESEND_API_KEY")! },
});

const email = createEmailClient({
  provider,
  from: "hello@example.com",
  // storage is required only if you use `template:` (see "Email templates")
});

await email.send({
  to: "user@example.com",
  subject: "Your order is confirmed",
  text: "Thanks for your purchase!",
  html: "<p>Thanks for your purchase!</p>",
});
```

`send()` requires either `template`, or both `subject` and `html` — it throws otherwise. When `from` is omitted, the client's configured default `from` address is used.

The one place a pre-built client is handed to you automatically is inside a background job — see "Using in background jobs" below.

### `send()` options

| Field | Type | Description |
|-------|------|-------------|
| `to` | `string \| string[]` | Recipient address(es). Required. |
| `subject` | `string` | Email subject. Required unless `template` supplies its own. When both are present, the explicit `subject:` passed to `send()` wins — the template's own subject is only used as a fallback. |
| `from` | `string` | Override the default From address. |
| `replyTo` | `string` | Reply-To address. |
| `text` | `string` | Plain-text body. Auto-generated from the rendered output only when using `template:` — a template renderer strips its own HTML into a text fallback. Sending raw `html` without a `template` does **not** auto-generate text if you omit it; no provider does this either. Pass `text:` explicitly for a non-template send if you want one. |
| `html` | `string` | HTML body. Ignored if `template` is also given. |
| `template` | `string` | Template name — loads from `emails/` directory (see below). |
| `data` | `Record<string, unknown>` | Data passed to the template renderer. |

There is no `attachments` field, at any layer — `EmailMessage`/`SendOptions` don't have one. If you need attachments, bypass Dune's abstraction and call your provider's SDK directly (see "Attachments" note at the end).

## Email templates

Templates live in `emails/` at your site root. Dune looks up `{template}.email.tsx`, then `{template}.email.md`, then `{template}.email.mdx` — in that order.

### TSX template

```tsx
// emails/order-confirmed.email.tsx
export type Data = { orderId: string };

export default function OrderConfirmed({ orderId }: Data) {
  return (
    <div>
      <h1>Order confirmed</h1>
      <p>Order #{orderId} is on its way.</p>
    </div>
  );
}

// Optional — falls back to the template's filename stem if omitted.
export const subject = "Your order is confirmed";
```

The component receives `data` directly as its props (not wrapped in `{ data, site }`). There's no automatic HTML-shell wrapping — whatever the component returns is the rendered output; include your own `<html><body>` structure if you need one. `export type Data` is compile-time TypeScript only; nothing validates it at runtime.

Rendering a `.tsx` template resolves its file path from the process's current working directory rather than the site's `StorageAdapter` root, so it only works reliably when the process cwd is the site root (true under `dune dev`/`dune serve`).

### Markdown template

```markdown
<!-- emails/welcome.email.md -->
# Welcome to {{siteName}}

Hi {{name}},

Thanks for signing up! Your account is ready.
```

`{{key}}` placeholders are substituted from `data:` and HTML-escaped. Unknown keys are left as the literal `{{key}}` text. **The subject is taken from the first `# Heading` line** in the body (then stripped from the rendered output) — there is no YAML frontmatter parsing anywhere in this path, so a `---\nsubject: ...\n---` block would not set the subject; it would just render as garbled Markdown.

### MDX template

**`.email.mdx` files are rendered as plain Markdown, not compiled MDX/JSX** — same `{{key}}` substitution and heading-derived subject as `.email.md`, no JSX support, no component imports. This is a known, explicit limitation of the current template engine, distinct from how `.mdx` content pages render. Use `.email.tsx` if you need real JSX in an email.

### Sending with a template

```ts
await email.send({
  to: "user@example.com",
  template: "welcome",
  data: { name: "Alice" },
});
```

## Using in background jobs

`JobContext.email` is a real, already-configured `EmailClient` — the only context in Dune that gets one for free, with `template:` support already wired up:

```ts
// jobs/weekly-digest.ts
import type { JobContext } from "@dune/core/jobs";

export const schedule = "0 9 * * MON";

export default async function handler(ctx: JobContext) {
  // ctx.content is a full DuneEngine, not a query API — there's no
  // ctx.content.find(). Filter/sort ctx.content.pages directly.
  const posts = ctx.content.pages
    .filter((p) => p.sourcePath.startsWith("02.blog/") && p.published)
    .sort((a, b) => (b.date ?? "").localeCompare(a.date ?? ""))
    .slice(0, 5);

  await ctx.email.send({
    to: "subscribers@example.com",
    template: "digest",
    data: { posts, siteUrl: ctx.config.site.url },
  });
}
```

## Using in plugins

Plugin hook handlers (`HookContext`) have no ambient `email`, `content`, or `db` — construct your own client in `setup()` and close over it in your hooks:

```ts
import type { DunePlugin, PluginApi } from "@dune/core/plugins";
import { createEmailClient, createEmailProvider } from "@dune/core/email";

let email: ReturnType<typeof createEmailClient>;

export default {
  name: "my-plugin",
  version: "1.0.0",
  setup(api: PluginApi) {
    const provider = createEmailProvider({
      provider: "resend",
      from: "hello@example.com",
      resend: { apiKey: Deno.env.get("RESEND_API_KEY")! },
    });
    email = createEmailClient({ provider, from: "hello@example.com" });
  },
  hooks: {
    onPageCreate: async (ctx) => {
      await email.send({
        to: "list@example.com",
        subject: `New page: ${ctx.data.title}`,
        html: `<p>${ctx.data.sourcePath} was just created.</p>`,
      });
    },
  },
} satisfies DunePlugin;
```

**There is no hook that fires on new-user creation** — no `onUserCreate`, no `onSiteUserCreated`, nothing in the real `HookEvent` union relates to signup. "Send a welcome email when someone signs up" is not currently wireable through Dune's hook system; you'd need to build it into your own auth callback route directly.

## Provider notes

**Console (default)** — Logs the email to stdout. No email is actually sent. Useful in development, but see "Dev-mode email preview" below — this is not the same thing as "development is always safe."

**Resend** — Uses the Resend HTTP API. Requires a verified sending domain in the Resend dashboard.

**Postmark** — Uses the Postmark API. Requires a verified sender signature.

**SendGrid** — Uses the SendGrid Mail Send API. Requires API key with Mail Send permission.

**SMTP** — Uses a raw SMTP connection. `secure: true` uses implicit TLS (port 465); `secure: false` upgrades with STARTTLS (port 587). Set `pass: "$SMTP_PASS"` (field is `pass`, not `password`) to avoid committing credentials.

## Dev-mode email preview

**Provider selection does not check any environment variable.** If `email:` in `site.yaml` names a real provider with valid credentials, that provider is used and `send()` delivers for real — in development exactly as in production.

The only dev-aware behavior lives inside the console provider itself (the one you get when no provider is configured, or the configured one is missing credentials): when `DUNE_ENV=dev`, it additionally writes each message to `{runtimeDir}/dev-email/{id}.json` (default `.dune/admin/dev-email/`) so the admin panel can show it. It always logs to stdout regardless of `DUNE_ENV`; the file-write is the dev-only part.

**This means dev mode does not intercept a properly-configured real provider.** If real credentials are present in your local `site.yaml`, running locally with `DUNE_ENV=dev` still sends real email to real recipients. To be safe locally, don't put real provider credentials in your local config, or explicitly set `email.provider: console`.

Browse intercepted emails (console-provider-with-`DUNE_ENV=dev` case only) in the admin panel under **Dev → Email Preview** (`/admin/email-preview`), or query the API directly:

```
GET /admin/api/email-preview
→ { "emails": [ { "id", "to", "from", "subject", "timestamp" }, … ] }   (html/text omitted from the list)

GET /admin/api/email-preview/{id}
→ { "id", "to", "from", "subject", "timestamp", "html", "text" }
```

Both endpoints require the `config.read` admin permission and return `404` outside `DUNE_ENV=dev`.

## Attachments

Not supported — no `attachments` field exists anywhere in the send/message types. Bypass Dune's abstraction and call your provider's SDK directly:

```ts
import { Resend } from "npm:resend";

const resend = new Resend(Deno.env.get("RESEND_API_KEY"));
await resend.emails.send({
  from: "hello@example.com",
  to: "user@example.com",
  subject: "Invoice",
  html: "<p>See attached.</p>",
  attachments: [{ filename: "invoice.pdf", content: pdfBuffer }],
});
```
