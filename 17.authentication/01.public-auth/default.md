---
title: "Public Authentication"
published: true
visible: true
taxonomy:
  audience: [developer, webmaster]
  difficulty: [intermediate]
  topic: [authentication, security]
metadata:
  description: "Set up OAuth, magic link, or external JWT authentication for public site visitors"
---

# Public Authentication

Dune's public auth system lets site visitors register and log in — completely separate from the admin panel. Login methods: OAuth (GitHub, Google, Discord), magic link (passwordless email), and external JWT (Clerk, Auth0, etc.).

**As of 0.31.7, writing `auth:` in `site.yaml` is enough on its own** — `createDuneApp()` (the code path both `dune serve` and `dune dev` go through) calls `mountDuneAuth(app, ctx)` automatically whenever `site.auth` is configured. A site that never writes an `auth:` block gets zero behavior change: no new directories, no `/auth/*` routes, no added per-request middleware. On earlier versions, `mountDuneAuth()` had no public export under any `@dune/core/*` subpath at all, and there was no supported way to call it, manually or otherwise.

**Headless mode is the one case that still needs a manual call** — headless sites don't go through `createDuneApp()`, so call `mountDuneAuth(app, ctx)` explicitly from `main.ts`, the same way they already call `mountDuneAdmin()`:

```ts
// main.ts — headless mode only; the default/full mode wires this automatically
import { App } from "fresh";
import { bootstrap } from "@dune/core/bootstrap";
import { mountDuneAuth } from "@dune/core/auth/mount";

const ctx = await bootstrap("./");
const app = new App();
await mountDuneAuth(app, ctx); // populates ctx.state.siteUser on every request from here on
```

`dune build --static` (the SSG builder) explicitly opts out of the auto-wiring (`mountAuth: false` on its own internal `createDuneApp()` call) — a static build has no live request flow for `/auth/*` to serve, and shouldn't create session/user-store directories as a side effect of generating HTML.

## Configuration

```yaml
# site.yaml
auth:
  mode: "dune"                   # "dune" | "external-jwt"
  sessionLifetime: 2592000       # Session TTL in seconds (default: 30 days)
  providers:
    github:
      clientId: "$GITHUB_CLIENT_ID"
      clientSecret: "$GITHUB_CLIENT_SECRET"
    google:
      clientId: "$GOOGLE_CLIENT_ID"
      clientSecret: "$GOOGLE_CLIENT_SECRET"
    discord:
      clientId: "$DISCORD_CLIENT_ID"
      clientSecret: "$DISCORD_CLIENT_SECRET"
    magicLink:
      enabled: true
```

Only configure the providers you need. Each OAuth provider requires a registered OAuth app pointing its callback to `{site.url}/auth/{provider}/callback`.

## Registered routes

| Method | Route | Description |
|--------|-------|-------------|
| `GET` | `/auth/login` | Default login page (renders `LoginForm` or theme's `auth/login.tsx`) |
| `POST` | `/auth/logout` | Destroy session cookie and redirect to `/` |
| `GET` | `/auth/me` | Return current `User` as JSON, or `401` if not logged in |
| `GET` | `/auth/github` | Start GitHub OAuth flow |
| `GET` | `/auth/github/callback` | GitHub OAuth callback |
| `GET` | `/auth/google` | Start Google OAuth flow |
| `GET` | `/auth/google/callback` | Google OAuth callback |
| `GET` | `/auth/discord` | Start Discord OAuth flow |
| `GET` | `/auth/discord/callback` | Discord OAuth callback |
| `POST` | `/auth/magic/send` | Send a magic link to `email` (form param) |
| `GET` | `/auth/magic` | Verify magic link token and create session |

Routes for unconfigured providers return `404`.

## OAuth login

Each OAuth provider follows the standard authorization code flow:

1. User visits `/auth/github` → redirected to GitHub with `state` parameter
2. GitHub redirects back to `/auth/github/callback?code=...&state=...`
3. Dune exchanges the code for an access token, fetches the user profile
4. Dune upserts a `User` record (creates on first login, updates on subsequent)
5. Session cookie `dune-site-session` is set; user is redirected to `?next=` or `/`

### OAuth app setup

| Provider | Callback URL |
|----------|-------------|
| GitHub | `{site.url}/auth/github/callback` |
| Google | `{site.url}/auth/google/callback` |
| Discord | `{site.url}/auth/discord/callback` |

## Magic link

The magic link flow is passwordless:

1. User enters their email at `/auth/login` (or any form POSTing to `/auth/magic/send`)
2. Dune generates a token (HMAC-SHA256 signed, 15-minute TTL), sends an email with a link to `/auth/magic?token=...`
3. User clicks the link → Dune verifies the token, upserts the user, sets the session cookie

Magic link requires the [email module](../../06.extending/04.email) to be configured — without it, the link is logged to stdout (development only).

## External JWT mode

Use `mode: "external-jwt"` to delegate authentication entirely to an external provider (Clerk, Auth0, Supabase, etc.). Dune validates the Bearer token on each request and maps JWT claims to a `User`.

```yaml
auth:
  mode: "external-jwt"
  jwt:
    jwksUrl: "https://your-tenant.clerk.accounts.dev/.well-known/jwks.json"
    userIdClaim: "sub"         # default
    emailClaim: "email"        # default
    rolesClaim: "roles"        # default — string or string[]
```

For HS256 shared-secret tokens:

```yaml
auth:
  mode: "external-jwt"
  jwt:
    secret: "$JWT_SECRET"
```

In external-JWT mode, there are no session cookies and no `/auth/*` login routes — your external provider handles the login UI. Clients pass tokens as `Authorization: Bearer {token}` headers. The auth middleware injects a synthetic `User` from the validated claims.

## User store backends

The `userStore` setting controls where `User` records are persisted:

| Value | Description |
|-------|-------------|
| `"local"` | Default. Flat YAML files in `data/users/`. Committed to version control. Suitable for most sites. |
| `"session"` | No server-side records. Identity is embedded in the session cookie from OAuth/magic-link claims. Roles assigned after login (e.g. via payment webhook) are not visible until the user logs out and back in. |
| `"db"` | Records stored in the site's database (SQLite or PostgreSQL). Requires `DUNE_DB_PATH` or `DUNE_DB_URL`. Suitable for large user bases or multi-process deployments where flat-file contention is a concern. |

```yaml
auth:
  userStore: "db"   # default: "local"
```

Sessions themselves (not user records) use `system.session_store` — `"local"` (file-backed, default), `"kv"` (Deno Deploy/multi-isolate), or `"redis"` (multi-process behind a load balancer). Public-auth sessions and admin sessions share this same setting; there's no separate config for either.

## IdP webhook (user deletion)

When using `mode: external-jwt` with `authzStore: local`, configure a webhook so Dune can clean up authorization tuples when a user is deleted in the external provider:

```yaml
auth:
  mode: "external-jwt"
  authzStore: local
  webhook:
    provider: "clerk"               # "clerk" | "auth0" | "generic"
    secret: "$DUNE_CLERK_WEBHOOK_SECRET"
```

This activates `POST /auth/webhook`. On receiving a `user.deleted` event (provider-specific payload format), Dune revokes all authz tuples for that user. Role-change events are handled automatically by per-request fingerprint reconciliation and do not require a webhook.

For `provider: "generic"`, the signature header defaults to `x-dune-signature`. Override with `signatureHeader: "x-my-sig"`.

The webhook endpoint verifies the provider's HMAC signature before processing. Configure the webhook URL in your IdP's dashboard as `{site.url}/auth/webhook`.

## User

Every logged-in visitor is represented as a `User` — the same account record type shared with admin panel accounts (`@dune/plugin-admin`); a public site visitor and an admin panel user are both just rows in the same store, distinguished only by what's in `roles`:

```ts
interface User {
  id: string;
  email: string;         // Primary identifier
  name?: string;          // Display name (from OAuth profile, or set by an admin)
  avatarUrl?: string;     // Avatar URL (from OAuth profile)
  username?: string;      // Set for password-login admin accounts; absent for OAuth/magic-link-only visitors
  passwordHash?: string;  // Set only for accounts with a local password
  roles: string[];        // e.g. ["member"], or ["admin"] for an admin-panel account
  provider: string;       // "github" | "google" | "discord" | "magic" | "jwt" | "local"
  providerId?: string;    // Provider's user ID (for OAuth)
  createdAt: number;      // Unix timestamp (ms)
  updatedAt: number;
  lastSeenAt: number;
  stripeCustomerId?: string;
}
```

Users are stored as flat JSON files in `data/users/` (controlled by `admin.dataDir`). The directory should be committed to version control — user records are site data, not ephemeral runtime state.

An email-based index in `data/users/by-email/` allows O(1) lookups by email address for login flows.

## Accessing the current user

**In route handlers / Fresh middleware:**

```ts
const siteUser = ctx.state.siteUser as User | null;
```

**In generated CRUD route handlers** (from `dune codegen`) and other code that only has a raw `Request`, not a Fresh context — the resolved user is serialized into an internal `x-dune-user` request header (JSON). `requireAuth(req, mode)` from `@dune/core/auth/api-guard` reads it. This header is for internal use only: Dune strips any externally-supplied copy of it from every incoming request before any route or plugin sees it, so it can't be set from outside.

**In hand-written TSX content pages** — the default component receives a `siteUser` prop (`ContentPageProps.siteUser`), the same resolved `User | null`, populated automatically for every request. No manual header parsing needed.

**Via the API:**

```
GET /auth/me
→ 200 { id, email, name, avatarUrl, roles, provider, createdAt }
   401 if not logged in
```

## Login page template

To customise the login page, add `templates/auth/login.tsx` to your theme:

```tsx
import type { TemplateProps } from "@dune/core";
import { LoginForm } from "@dune/core/ui";

export default function AuthLogin({ site, Layout, ...props }: TemplateProps) {
  return (
    <Layout {...props} site={site} pageTitle="Log in">
      <h1>Log in to {site.title}</h1>
      <LoginForm providers={["github", "google", "magic"]} />
    </Layout>
  );
}
```

If no `auth/login.tsx` exists in the theme, Dune renders a minimal built-in login page.
