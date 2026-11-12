---
title: "Content Gating"
published: true
visible: true
taxonomy:
  audience: [developer, webmaster]
  difficulty: [beginner]
  topic: [authentication, content, security]
metadata:
  description: "Restrict page access by login status or role using the roles: frontmatter field"
---

# Content Gating

Add a `roles:` field to any page's frontmatter to restrict who can view it. Dune checks the current visitor's roles before rendering — unauthenticated visitors are redirected to login; visitors with the wrong role get a 403.

## Basic usage

```yaml
---
title: Members Only
roles: member        # single role — string shorthand
---
```

```yaml
---
title: Premium Content
roles: [member, premium]   # array — user needs AT LEAST ONE of the listed roles
---
```

```yaml
---
title: Verified Members
roles:               # all: — user must have EVERY listed role
  all: [member, verified]
---
```

```yaml
---
title: Account Required
roles: any           # special value — any logged-in user, regardless of role
---
```

```yaml
---
title: Teaser
roles: public        # open to everyone, even inside a gated section or a gated site
---
```

## How it works

Before rendering a page with a `roles:` field, Dune checks the request:

1. **No session / no Bearer token** → `302` redirect to `/auth/login?next={current-path}`
2. **Session valid but role check fails** → `403 Forbidden`
3. **Role check passes** → page renders normally

The check happens server-side, before any template is executed. There is no client-side bypass.

## Role values

| Value | Meaning |
|-------|---------|
| `any` | Any authenticated user (roles don't matter). Same as `[]`. |
| `public` | Open to everyone. Overrides an inherited or site-default gate. |
| `"member"` | User must have `"member"` in their `roles[]` |
| `["member", "premium"]` | User must have **at least one** listed role |
| `all: ["member", "verified"]` | User must have **every** listed role |

A role literally named `any` or `public` can't be used: both words are reserved. (Before 0.37, `any` was read as a role named `any` and did not mean "any logged-in user".)

Roles are assigned at login (from OAuth profile or JWT claims) or after a successful payment (see [Payments](../../06.extending/05.payments)).

## Login redirect

Logged-out visitors are sent to `/auth/login?next={current-path}`, and the `next` parameter is restored after login.

## 403 page

When a logged-in user fails the role check, Dune renders a `403` response. Customise the error page by adding `templates/errors/403.tsx` to your theme:

```tsx
export default function Forbidden({ site, Layout, ...props }) {
  return (
    <Layout {...props} site={site} pageTitle="Access denied">
      <h1>Members only</h1>
      <p>You need a membership to view this page.</p>
      <a href="/pricing">See pricing</a>
    </Layout>
  );
}
```

## Gating a section

A page's `roles:` applies to everything beneath it, in every language. Put it on the section's top page and every page below inherits it:

```
content/
  05.members/
    default.md        # roles: member
    01.handbook/      # gated: inherits `member`
    02.downloads/     # gated
      default.md
      guide.pdf       # gated too (see Files below)
```

Dune resolves the rule for each page in this order, and the first match wins:

1. the page's own `roles:`
2. the nearest ancestor's `roles:`
3. the site default, `auth.default_roles` in `site.yaml`
4. otherwise the page is public

`roles: public` on a page or section opts it out of an inherited or default gate. A malformed `roles:` value counts as "not declared", so it inherits rather than opening the page.

A translated section takes its rule from the same language first, then from another language. If the languages disagree, the gate wins over `public`.

## Gating the whole site

For a site that is entirely members-only (a private preview, an event site), set a default instead of repeating `roles:` everywhere:

```yaml
# site.yaml
auth:
  default_roles: any    # same values as `roles:`
```

Every page that neither declares nor inherits a rule now requires login. Mark the exceptions with `roles: public`, such as a teaser page. `/auth/*` (login, logout, callbacks) and the theme's static assets are never gated.

## Files in a gated page's folder

Audio, images, PDFs and other files next to a gated page follow that page's rule. A file in a folder with no page of its own follows the nearest ancestor page. A visitor who may not see the page gets a `404` for its files, which doesn't confirm the file exists. The one exception is a logged-out visitor who opens a file directly in the browser: they are sent to the login page. Gated files are sent with `Cache-Control: private, no-store`.

A file with no owning page stays public, unless the site sets `auth.default_roles`, which then gates it too.

## Checking roles in templates

For conditional content within a page (show some content to all, premium content to members only), read the current user in the template:

```tsx
import type { TemplateProps } from "@dune/core";

export default function Post({ page, siteUser, Layout, ...props }: TemplateProps) {
  const isMember = siteUser?.roles.includes("member") ?? false;

  return (
    <Layout {...props} page={page}>
      <article dangerouslySetInnerHTML={{ __html: page.html }} />
      {isMember && (
        <section>
          <h2>Members-only resources</h2>
          {/* ... */}
        </section>
      )}
    </Layout>
  );
}
```

`siteUser` is `null` when the visitor is not logged in.
