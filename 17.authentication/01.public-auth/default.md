---
title: "Public Authentication"
published: true
visible: true
taxonomy:
  audience: [developer, webmaster]
  difficulty: [intermediate]
  topic: [authentication, security]
metadata:
  description: "Set up OAuth, magic link, password, or external JWT authentication for public site visitors"
---

# Public Authentication

Dune's public auth system lets site visitors register and log in — completely separate from the admin panel. Login methods: OAuth (GitHub, Google, Discord), magic link (passwordless email), password (for accounts you create), and external JWT (Clerk, Auth0, etc.).

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
  session_lifetime: 2592000       # Session TTL in seconds (default: 30 days)
  providers:
    github:
      client_id: "$GITHUB_CLIENT_ID"
      client_secret: "$GITHUB_CLIENT_SECRET"
    google:
      client_id: "$GOOGLE_CLIENT_ID"
      client_secret: "$GOOGLE_CLIENT_SECRET"
    discord:
      client_id: "$DISCORD_CLIENT_ID"
      client_secret: "$DISCORD_CLIENT_SECRET"
    magic_link:
      enabled: true
    password:
      enabled: true                # accounts with a password: see "Password login" below
```

Only configure the providers you need. Each OAuth provider requires a registered OAuth app pointing its callback to `{site.url}/auth/{provider}/callback`.

## Registered routes

| Method | Route | Description |
|--------|-------|-------------|
| `GET` | `/auth/login` | Login page: the theme's `templates/login.tsx`, or a built-in page (providers, magic link, and the password form when enabled) |
| `POST` | `/auth/login` | Password login (`404` unless `password.enabled`); see [Password login](#password-login) |
| `GET` | `/auth/logout` | Destroy session cookie and redirect to `/` (GET, not POST) |
| `GET` | `/auth/me` | Return current `User` as JSON, or `401` if not logged in |
| `GET` | `/auth/github` | Start GitHub OAuth flow |
| `GET` | `/auth/github/link` | Start GitHub OAuth to link it to the *current session's* account (`401` if not logged in) |
| `GET` | `/auth/github/callback` | GitHub OAuth callback — completes either an ordinary login/signup or a `/link` flow |
| `POST` | `/auth/github/unlink` | Remove a previously-linked GitHub identity (`401` if not logged in) |
| `GET` | `/auth/google` | Start Google OAuth flow |
| `GET` | `/auth/google/link` | Start Google OAuth to link it to the *current session's* account |
| `GET` | `/auth/google/callback` | Google OAuth callback |
| `POST` | `/auth/google/unlink` | Remove a previously-linked Google identity |
| `GET` | `/auth/discord` | Start Discord OAuth flow |
| `GET` | `/auth/discord/link` | Start Discord OAuth to link it to the *current session's* account |
| `GET` | `/auth/discord/callback` | Discord OAuth callback |
| `POST` | `/auth/discord/unlink` | Remove a previously-linked Discord identity |
| `POST` | `/auth/magic/send` | Send a magic link to `email` (form param) |
| `GET` | `/auth/magic` | Verify magic link token and create session |

Routes for unconfigured providers return `404`. The session cookie is `dune_auth` (HttpOnly, SameSite=Lax, Secure outside `DUNE_ENV=dev`).

## Password login

For sites whose logged-in users have passwords, including sites with `admin.enabled: false`. Accounts are `provider: "local"` users in the shared user store: create them with `dune users:create <email> --password-prompt` (an admin can also create them). There is no self-service sign-up and no password reset by email yet.

```yaml
# site.yaml
auth:
  providers:
    password:
      enabled: true
```

`POST /auth/login` takes `identifier` (an email or a username), `password` and an optional `next`, form-encoded or JSON, from the same origin:

| | Form (`application/x-www-form-urlencoded`) | JSON |
|---|---|---|
| Success | `303` to `next`, session cookie set | `200 { "ok": true, "next": "/…" }`, cookie set |
| Wrong credentials | `303` to `/auth/login?error=invalid` | `401 { "error": "Invalid credentials" }` |
| Too many attempts | `429`, the login page with `error: "rate_limited"` | `429 { "error": "Too many attempts" }` + `Retry-After` |

`next` is only followed when it is a path on your site (`/account`, `/members?tab=2`); anything else (`https://…`, `//host`, `javascript:`) falls back to `/`.

Every failure looks the same, whether the account does not exist, is disabled, signs in with GitHub rather than a password, or has the wrong password: same status, same body, and the password is verified against a dummy hash for the accounts that cannot sign in this way, so timing does not tell them apart. Attempts are limited to ten per IP address and ten failures per identifier (including identifiers that do not exist) per 15 minutes. Behind a reverse proxy, set `system.trusted_proxies: true` so the real client address is used. Old password hashes are upgraded on a successful login.

The session is the site session (`dune_auth`). It is separate from the admin session: logging in here does not log anyone into `/admin`, and the other way round. Password login needs persistent user records, so it is unavailable with `user_store: session`.

### A themed login page

Add `templates/login.tsx` to your theme and `GET /auth/login` renders it (also for the `429` case). Like `error.tsx`, it receives a synthetic page, and the details are in `page.frontmatter`:

| Field | Meaning |
|-------|---------|
| `providers` | Configured OAuth providers, e.g. `["github"]`; each signs in at `/auth/<name>` |
| `magicLink` | `true` when magic-link login is on (`POST /auth/magic/send` with JSON `{ "email" }`) |
| `passwordLogin` | `true` when password login is on |
| `error` | `"invalid"` or `"rate_limited"` after a failed attempt, else absent |
| `next` | Where to go after signing in; already a safe path |

```tsx
/** @jsxImportSource preact */
export default function Login({ page, site, Layout }: any) {
  const { passwordLogin, error, next } = page.frontmatter;
  return (
    <Layout site={site}>
      {error === "invalid" && <p role="alert">Wrong email or password.</p>}
      {passwordLogin && (
        <form method="POST" action="/auth/login">
          <input type="hidden" name="next" value={next} />
          <input name="identifier" autocomplete="username" required />
          <input name="password" type="password" autocomplete="current-password" required />
          <button>Log in</button>
        </form>
      )}
    </Layout>
  );
}
```

A theme without `login.tsx` keeps working: the built-in page is served.

## Linking an additional provider

Any logged-in user can connect another OAuth provider to their account:

```html
<a href="/auth/github/link">Connect GitHub</a>
```

`getByProvider()` checks both the account's original signup `provider`/`providerId` and everything in `linkedProviders` (added via `/link`). The original signup provider is never removable through `/unlink` — only entries added later via `/link` can be removed. `GET /auth/me` includes linked provider names as `linkedProviders: string[]`.

If the provider identity being linked already belongs to a *different* account, `/callback` doesn't error — it logs into that account instead (redirects to `/?dune_link=linked_elsewhere`), since completing that provider's OAuth consent screen is real proof of controlling it. Check `?dune_link=` (`linked` | `already_linked` | `linked_elsewhere`) to show your own message — there's no built-in UI for any of this.

`POST /auth/{provider}/unlink` refuses to remove your last *additional* identity while magic link is disabled site-wide (your original signup provider is unaffected either way). Only works with `user_store: local` or `db` — `400`s under `user_store: session` (no persistent record to attach a link to).

## OAuth login

Each OAuth provider follows the standard authorization code flow:

1. User visits `/auth/github` → redirected to GitHub with `state` parameter
2. GitHub redirects back to `/auth/github/callback?code=...&state=...`
3. Dune exchanges the code for an access token, fetches the user profile
4. Dune upserts a `User` record (creates on first login, updates on subsequent)
5. Session cookie `dune_auth` is set; user is redirected to `?next=` or `/`

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
    jwks_url: "https://your-tenant.clerk.accounts.dev/.well-known/jwks.json"
    user_id_claim: "sub"         # default
    email_claim: "email"        # default
    roles_claim: "roles"        # default — string or string[]
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

The `user_store` setting controls where `User` records are persisted:

| Value | Description |
|-------|-------------|
| `"local"` | Default. Flat YAML files in `data/users/`. Committed to version control. Suitable for most sites. |
| `"session"` | No server-side records. Identity is embedded in the session cookie from OAuth/magic-link claims. Roles assigned after login (e.g. via payment webhook) are not visible until the user logs out and back in. |
| `"db"` | Records stored in the site's database (SQLite or PostgreSQL). Requires `DUNE_DB_PATH` or `DUNE_DB_URL`. Suitable for large user bases or multi-process deployments where flat-file contention is a concern. |

```yaml
auth:
  user_store: "db"   # default: "local"
```

Sessions themselves (not user records) use `system.session_store` — see [System Configuration](/docs/configuration/system-config#session-store) for the full backend reference (`local`, `kv`, `redis`). Public-auth sessions and admin sessions share this same setting; there's no separate config for either.

## IdP webhook (user deletion)

When using `mode: external-jwt` with `authz_store: local`, configure a webhook so Dune can clean up authorization tuples when a user is deleted in the external provider:

```yaml
auth:
  mode: "external-jwt"
  authz_store: local
  webhook:
    provider: "clerk"               # "clerk" | "auth0" | "generic"
    secret: "$DUNE_CLERK_WEBHOOK_SECRET"
```

This activates `POST /auth/webhook`. On receiving a `user.deleted` event (provider-specific payload format), Dune revokes all authz tuples for that user. Role-change events are handled automatically by per-request fingerprint reconciliation and do not require a webhook.

For `provider: "generic"`, the signature header defaults to `x-dune-signature`. Override with `signature_header: "x-my-sig"`.

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

Users are stored as flat JSON files in `data/users/` (under `data_dir` in `system.yaml`). The directory should be committed to version control — user records are site data, not ephemeral runtime state.

An email-based index in `data/users/by-email/` allows O(1) lookups by email address for login flows.

## Accessing the current user

**In route handlers / Fresh middleware:**

```ts
const siteUser = ctx.state.siteUser as User | null;
```

**In generated CRUD route handlers** (from `dune codegen`) and other code that only has a raw `Request`, not a Fresh context — the resolved user is serialized into an internal `x-dune-user` request header (JSON). `requireAuth(req, mode)` from `@dune/core/auth/api-guard` reads it. This header is for internal use only: Dune strips any externally-supplied copy of it from every incoming request before any route or plugin sees it, so it can't be set from outside.

**In hand-written TSX content pages** — the default component receives a `user` prop (`ContentPageProps.user`), the same resolved `User | null`, populated automatically for every request. No manual header parsing needed.

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
