---
title: "Authorization (Polizy)"
published: true
visible: true
taxonomy:
  audience: [developer]
  difficulty: [advanced]
  topic: [authentication, authorization, security]
metadata:
  description: "Zanzibar-inspired relationship-based authorization with polizy — group membership, resource-level grants, and hierarchy"
---

# Authorization (Polizy)

Dune uses [polizy](https://github.com/bratsos/polizy) for authorization — a Zanzibar-inspired relationship-based model. One `authz.check()` call covers all authorization layers uniformly:

- **Content gating** (`roles:` frontmatter) — checked automatically by Dune's page middleware
- **Route middleware guards** — protect programmatic routes
- **Resource-level grants** — per-object permissions (e.g. a user owns a specific page)

## How it works

Polizy stores permission *tuples*: `(subject, relation, object)`. A check asks "can subject X perform action Y on object Z?" — the answer is derived by traversing the tuple graph, following group membership and hierarchy rules.

Dune's default schema defines:

| Relation | Type | Used for |
|----------|------|----------|
| `member` | group | Content gating — user is a member of a role group |
| `admin` | direct | Admin panel access |
| `editor` | direct | Editor-level access |
| `author` | direct | Author-level access |
| `owner` | direct | Per-resource ownership |

| Action | Satisfied by |
|--------|-------------|
| `access` | member, admin, editor, author |
| `edit` | owner, admin, editor |
| `pages.update` | admin, editor |
| `users.manage` | admin |
| `media.upload` | admin, editor, author |

## Config

```yaml
# site.yaml
auth:
  mode: dune           # "dune" | "external-jwt"
  authzStore: local    # default in dune mode — data/permissions/*.json (in-memory index)
```

`authzStore` is independent of `userStore` but its default depends on `mode`:

- **`dune` mode**: defaults to `local` — tuples are stored as flat JSON files in `data/permissions/` and indexed in-memory on startup.
- **`external-jwt` mode**: no default — omitting `authzStore` gives JWT-claims-only authorization (no local tuple store). Set `authzStore: local` explicitly to opt in to local polizy authz (see [External JWT + local authz](#external-jwt--local-authz) below).

## Common patterns

### Check group membership (content gating)

```ts
const ok = await authz.check({
  who: { type: "user", id: ctx.state.siteUser.id },
  canThey: "access",
  onWhat: { type: "group", id: "member" },
});
```

### Grant group membership (after login or payment)

```ts
await authz.addMember({
  member: { type: "user", id: userId },
  group: { type: "group", id: "member" },
});
```

Call this after a successful payment or when a user qualifies for a role. The group `id` matches the role name used in `roles:` frontmatter.

### Grant a direct resource permission

```ts
await authz.allow({
  who: { type: "user", id: userId },
  toBe: "owner",
  onWhat: { type: "resource", id: pageRoute },
});
```

### Revoke permissions

```ts
// Remove a specific tuple
await authz.disallowAllMatching({
  who: { type: "user", id: userId },
  was: "member",
  onWhat: { type: "group", id: "member" },
});

// Remove all permissions for a user
await authz.disallowAllMatching({ who: { type: "user", id: userId } });
```

## Route middleware

For programmatic route protection (not frontmatter-based):

```ts
// routes/dashboard/_middleware.ts
import { FreshContext } from "fresh";
import { createDuneAuthSystem } from "@dune/core/auth/authz";

// In a real app, get authz from your bootstrap context
const { authz } = createDuneAuthSystem({ dataDir: "data" }, storage);

export async function handler(req: Request, ctx: FreshContext) {
  const user = ctx.state.siteUser;
  if (!user) {
    return Response.redirect(new URL("/auth/login", req.url));
  }
  const allowed = await authz.check({
    who: { type: "user", id: user.id },
    canThey: "access",
    onWhat: { type: "group", id: "member" },
  });
  if (!allowed) return new Response(null, { status: 403 });
  return ctx.next();
}
```

**Multi-site:** `setGatingAuthz()` sets a process-wide singleton. In multi-site setups where each site has its own authz bundle, pass the site-specific instance via the `authzOverride` parameter on `checkRolesAsync`, `enforceRoles`, and `enforceRolesFromRequest` instead of relying on the global:

```ts
import { checkRolesAsync } from "@dune/core/auth/gating";

const allowed = await checkRolesAsync(req, ["member"], { authzOverride: siteAuthz });
```

## Bootstrap from existing users

On first startup after authz is introduced, Dune automatically derives permission tuples from the `roles[]` array on existing `SiteUser` records. This is idempotent — it does not create duplicates.

After bootstrap, **tuples are the authority** for `authz.check()`. The `roles[]` array on `SiteUser` remains in sync as a cache (it is still updated when roles change) but `authz.check()` is the correct enforcement path.

## Tuple storage

Two backends are available via `authzStore`:

### `authzStore: local` (default)

Tuples are stored as HMAC-signed JSON files:

```
data/permissions/
  {uuid}.json   →  { id, subject, relation, object, sig }
```

The in-memory index is rebuilt from files on restart. Each file is signed with a per-installation HMAC key (stored in `data/permissions/.key`) — tampered or externally-written files are rejected on load. Do not edit these files directly; use `authz.allow()`, `authz.addMember()`, `authz.disallowAllMatching()`.

### `authzStore: db`

Tuples are stored in the site's database (SQLite or PostgreSQL). Suitable for large tuple sets or multi-process deployments where flat-file I/O is a bottleneck.

```yaml
auth:
  authzStore: db
```

Requires `DUNE_DB_PATH` (SQLite) or `DUNE_DB_URL` (PostgreSQL). The `authz_tuples` table is created automatically on first startup. The `authz.allow()` / `authz.addMember()` / `authz.disallowAllMatching()` API is identical to `local`; only the storage layer changes.

## Using `createDuneAuthSystem` directly

```ts
import { createDuneAuthSystem } from "@dune/core/auth/authz";
import type { StorageAdapter } from "@dune/core";

// createDuneAuthSystem returns { authz, adapter }
const { authz, adapter } = createDuneAuthSystem(
  { authzStore: "local", dataDir: "data" },
  storage,  // your Dune StorageAdapter
);

// Wire content gating (done automatically by mountDuneAuth)
import { setGatingAuthz } from "@dune/core";
setGatingAuthz(authz);
```

## External JWT + local authz

`auth.mode: external-jwt` handles authentication only — the external provider (Clerk, Auth0, etc.) issues and validates tokens. Authorization (content gating, route guards) can be handled two ways:

**JWT-claims-only** (default in `external-jwt` mode — `authzStore` omitted): role membership is derived directly from the `roles` claim on every request. No local tuple store. Simple, but limited to what the JWT carries.

**External JWT + local polizy** (opt-in via `authzStore: local`): the external provider handles authn; polizy handles authz. Opt in by setting `authzStore: local` explicitly:

```yaml
auth:
  mode: external-jwt
  authzStore: local
  jwt:
    jwksUrl: "https://your-tenant.clerk.accounts.dev/.well-known/jwks.json"
```

In this mode:

- **First appearance**: when a user presents a valid JWT for the first time, Dune seeds polizy group-membership tuples from their `roles` JWT claim. Subsequent `authz.check()` calls use the local tuple store — not the JWT.
- **Role changes**: on every authenticated request Dune computes a fingerprint of the current JWT's roles. If it differs from the last-seen fingerprint, Dune wipes the user's existing tuples (`disallowAllMatching`) and re-seeds from the new claim before the request continues. Role grants and revocations from the IdP are reflected within one JWT TTL.
- **Programmatic grants**: you can call `authz.addMember()` or `authz.allow()` to grant permissions beyond what the JWT carries (e.g. after a payment). These survive JWT rotations — they are only wiped if the JWT role fingerprint changes.
- **Deleted users**: if a user is deleted in the IdP their tuples accumulate until explicitly revoked. In practice this is bounded: deleted users cannot obtain a new valid JWT, so stale tuples are unexploitable. Call `authz.disallowAllMatching({ who: { type: "user", id } })` from your own deletion webhook to clean up proactively.

## Payment integration

Pass `authz` from `mountDuneAuth()` to `mountPaymentRoutes()` so that role grants after a successful payment are reflected in the authz tuple store immediately:

```ts
const publicAuthCtx = await mountDuneAuth(app, ctx);

mountPaymentRoutes(app, {
  siteConfig: ctx.config.site,
  userStore,
  baseUrl,
  authz: publicAuthCtx.authz,
});
```

Without this, the granted role is present in `userStore` but `authz.check()` returns false until the next restart.

## Admin panel authz

Admin users (`AdminUser`) are bootstrapped into the authz tuple store at startup via `bootstrapAdminTuples()`. Each admin user gets a direct relation tuple on `{ type: "app", id: "admin" }` matching their role:

```
admin  → tuple: (user:alice, admin,  app:admin)
editor → tuple: (user:bob,   editor, app:admin)
author → tuple: (user:carol, author, app:admin)
```

The admin panel middleware enforces panel access via `authz.check()` when authz is wired. Role changes and user deletes via the admin API automatically sync the tuple store.

**Granular admin permission checks** (`pages.create`, `config.read`, etc.) still use the flat `ROLE_PERMISSIONS` model — full polizy migration for those is tracked in the later roadmap.

## Limitations

- `authzStore: local` is single-process only. Multi-process deployments should use a shared database (future `authzStore: db`).
- Granular admin permissions (`pages.create`, `config.read`, etc.) — not yet through polizy. `ROLE_PERMISSIONS` remains the authority for those.
