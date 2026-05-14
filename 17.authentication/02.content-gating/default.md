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
roles:               # array — user must have ALL listed roles
  - member
  - premium
---
```

```yaml
---
title: Account Required
roles: any           # special value — any logged-in user, regardless of role
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
| `any` | Any authenticated user (roles don't matter) |
| `"member"` | User must have `"member"` in their `roles[]` |
| `["member", "premium"]` | User must have **all** listed roles |

Roles are assigned at login (from OAuth profile or JWT claims) or after a successful payment (see [Payments](../../06.extending/05.payments)).

## Custom login redirect

The redirect target defaults to `/auth/login`. To send visitors to a custom login page:

```yaml
# site.yaml
auth:
  loginPath: "/join"   # custom login page route
```

The `?next=` query parameter carries the original URL and is restored after login.

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

`roles:` applies to individual pages — there is no directory-level config. To gate an entire section, add the field to each page's frontmatter. If you manage many gated pages, consider using a blueprint with `roles` as a default field.

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
