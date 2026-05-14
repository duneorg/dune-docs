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
| `GET` | `/auth/me` | Return current `SiteUser` as JSON, or `401` if not logged in |
| `GET` | `/auth/github` | Start GitHub OAuth flow |
| `GET` | `/auth/github/callback` | GitHub OAuth callback |
| `GET` | `/auth/google` | Start Google OAuth flow |
| `GET` | `/auth/google/callback` | Google OAuth callback |
| `GET` | `/auth/discord` | Start Discord OAuth flow |
| `GET` | `/auth/discord/callback` | Discord OAuth callback |
| `POST` | `/auth/magic-link/send` | Send a magic link to `email` (form param) |
| `GET` | `/auth/magic-link/verify` | Verify magic link token and create session |

Routes for unconfigured providers return `404`.

## OAuth login

Each OAuth provider follows the standard authorization code flow:

1. User visits `/auth/github` → redirected to GitHub with `state` parameter
2. GitHub redirects back to `/auth/github/callback?code=...&state=...`
3. Dune exchanges the code for an access token, fetches the user profile
4. Dune upserts a `SiteUser` record (creates on first login, updates on subsequent)
5. Session cookie `dune-site-session` is set; user is redirected to `?next=` or `/`

### OAuth app setup

| Provider | Callback URL |
|----------|-------------|
| GitHub | `{site.url}/auth/github/callback` |
| Google | `{site.url}/auth/google/callback` |
| Discord | `{site.url}/auth/discord/callback` |

## Magic link

The magic link flow is passwordless:

1. User enters their email at `/auth/login` (or any form POSTing to `/auth/magic-link/send`)
2. Dune generates a token (HMAC-SHA256 signed, 15-minute TTL), sends an email with a link to `/auth/magic-link/verify?token=...`
3. User clicks the link → Dune verifies the token, upserts the user, sets the session cookie

Magic link requires the [email module](../../06.extending/04.email) to be configured — without it, the link is logged to stdout (development only).

## External JWT mode

Use `mode: "external-jwt"` to delegate authentication entirely to an external provider (Clerk, Auth0, Supabase, etc.). Dune validates the Bearer token on each request and maps JWT claims to a `SiteUser`.

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

In external-JWT mode, there are no session cookies and no `/auth/*` login routes — your external provider handles the login UI. Clients pass tokens as `Authorization: Bearer {token}` headers. The auth middleware injects a synthetic `SiteUser` from the validated claims.

## SiteUser

Every logged-in visitor is represented as a `SiteUser`:

```ts
interface SiteUser {
  id: string;           // Internal UUID
  email: string;        // Primary identifier
  name?: string;        // Display name (from OAuth profile)
  avatarUrl?: string;   // Avatar URL (from OAuth profile)
  roles: string[];      // Assigned roles (e.g. ["member"])
  provider: string;     // "github" | "google" | "discord" | "magic" | "jwt"
  providerId?: string;  // Provider's user ID (for OAuth)
  createdAt: number;    // Unix timestamp (ms)
}
```

Users are stored as flat YAML files in `data/site-users/` (controlled by `admin.dataDir`). The directory should be committed to version control — site user records are site data, not ephemeral runtime state.

An email-based index in `data/site-users/by-email/` allows O(1) lookups by email address for login flows.

## Accessing the current user

**In route handlers / Fresh middleware:**

```ts
const siteUser = ctx.state.siteUser as SiteUser | null;
```

**In TSX page handlers** — the middleware injects a JSON-encoded `SiteUser` in the `x-dune-site-user` request header, which Dune automatically parses and makes available as `page.siteUser` in `TemplateProps`.

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
