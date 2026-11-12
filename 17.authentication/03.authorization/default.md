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
| `pages.update` | admin, editor, author |
| `users.update` | admin |
| `media.upload` | admin, editor, author |

## Config

```yaml
# site.yaml
auth:
  mode: dune           # "dune" | "external-jwt"
  authz_store: local    # default in dune mode — data/permissions/*.json (in-memory index)
```

`authz_store` is independent of `user_store` but its default depends on `mode`:

- **`dune` mode**: defaults to `local` — tuples are stored as flat JSON files in `data/permissions/` and indexed in-memory on startup.
- **`external-jwt` mode**: no default — omitting `authz_store` gives JWT-claims-only authorization (no local tuple store). Set `authz_store: local` explicitly to opt in to local polizy authz (see [External JWT + local authz](#external-jwt--local-authz) below).

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

On first startup after authz is introduced, Dune automatically derives permission tuples from the `roles[]` array on existing `User` records. This is idempotent — it does not create duplicates.

After bootstrap, **tuples are the authority** for `authz.check()`. The `roles[]` array on `User` remains in sync as a cache (it is still updated when roles change) but `authz.check()` is the correct enforcement path.

## Tuple storage

Two backends are available via `authz_store`:

### `authz_store: local` (default)

Tuples are stored as JSON files, one per tuple:

```
data/permissions/
  {uuid}.json   →  { id, subject: {type, id}, relation, object: {type, id}, condition?, hmac? }
```

The in-memory index is built from the files on first use and re-synchronised with `data/permissions/` at most once a second, so tuples written or deleted by another process (for example `dune users:create`, `users:grant-role` or `users:revoke-role` run against a live server) take effect without a restart. **HMAC signing is opt-in, not automatic** — set `DUNE_AUTHZ_HMAC_SECRET` (loaded via `loadHmacKeyFromEnv()`) to enable it; without it, tuple file integrity checking is disabled (`dune serve` logs a warning: `authz.hmac.disabled`). Once a key is set, an unsigned tuple file (no `hmac` field, e.g. one written before the key was configured, or by hand) is rejected on load by default — run `dune authz:sign` after setting the secret to sign existing files. Set `DUNE_AUTHZ_HMAC_ALLOW_UNSIGNED=1` to temporarily accept unsigned tuples during migration instead. Do not edit these files directly; use `authz.allow()`, `authz.addMember()`, `authz.disallowAllMatching()`.

### `authz_store: db`

Tuples are stored in the site's database (SQLite or PostgreSQL). Suitable for large tuple sets or multi-process deployments where flat-file I/O is a bottleneck.

```yaml
auth:
  authz_store: db
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

**JWT-claims-only** (default in `external-jwt` mode — `authz_store` omitted): role membership is derived directly from the `roles` claim on every request. No local tuple store. Simple, but limited to what the JWT carries.

**External JWT + local polizy** (opt-in via `authz_store: local`): the external provider handles authn; polizy handles authz. Opt in by setting `authz_store: local` explicitly:

```yaml
auth:
  mode: external-jwt
  authz_store: local
  jwt:
    jwks_url: "https://your-tenant.clerk.accounts.dev/.well-known/jwks.json"
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

## Site-wide permissions

Everything an editor may do (`pages.update`, `pages.create`, `config.read`, ...) is checked with `authz.check()` against one object, `{ type: "app", id: "site" }`, mapped through the canonical `actionToRelations` schema. The object names what is protected (this site's content), not the admin panel, so it is the same with no admin, with `@dune/plugin-admin`, or with another admin plugin. There is no parallel role table.

Who holds a relation on it:

- **Site users** hold it through their role. Core grants `admin`, `editor` and `author` once, at group level, and site auth already keeps `member of group:<role>` in sync, so having the role is enough:

  ```
  (group:admin,  admin,  app:site)
  (group:editor, editor, app:site)
  (group:author, author, app:site)
  ```

  A role that is only a content tag (`member`, `subscriber`, ...) grants nothing here.

  This is a default, set by `role_grants` in `system.yaml`, a mapping of group to relation. Change which group holds which relation, keep a subset, or switch the group-level grants off:

  ```yaml
  # system.yaml
  role_grants:
    staff: editor      # your own group holds the editor relation
    editor: author     # "editor" is a content tier here: members only create and edit their own pages
  # role_grants: {}    # no group holds an admin-tier relation on the site
  ```

  The relations (`admin`, `editor`, `author`) are fixed by the authz schema and plugins cannot add new ones: the setting chooses which group gets which. A group's members are the users with that name in `roles[]`. The grants are reconciled at every start, so one you remove from `role_grants` is revoked (every group → admin/editor/author tuple on the site belongs to this setting). An `editor` relation may edit any page and an `author` only the pages it created, whatever the group is called. What an editor may do follows what authz says they hold: remapping `editor: author` makes those users authors even though their role is called `editor`. `role_grants: {}` governs site users; the admin plugin's own accounts (below) keep their rights. In an environment overlay (`config/env/<env>/system.yaml`) `role_grants` replaces the base mapping instead of merging into it, so one environment can switch the grants off with `role_grants: {}`.

Membership follows the user record: user stores that core creates (the site auth store, and `services.users.createLocalStore()` for plugins) keep `member of group:<role>` equal to `roles[]` on every create, role update and delete, so granting or revoking a role applies at once, without a restart. At startup the tuples are re-seeded from the records as well. A plugin that builds its own user store must call `syncUserRoles()` itself.
- **The admin plugin's own accounts**, the ones that can sign in to it (a local password), get a direct tuple matching their role, seeded at startup through `services.authz.seedAdminTuples()` (`bootstrapAdminTuples()` in core). A site user who holds `editor` through OAuth or magic link gets no direct tuple: `role_grants` governs them:

  ```
  (user:alice, admin,  app:site)
  (user:bob,   editor, app:site)
  ```

The authz system exists whenever something uses it: the site has a login (`site.auth`), a plugin provides `editor.session`, or a plugin declares `authzActions`. If creation fails at startup, checks fail closed (403). Tuples are derived data, re-seeded at every start; after changing this resource's name on an existing site, run `dune authz:sign` once if the store is HMAC-protected.

## Limitations

- `authz_store: local` is single-process only. Multi-process deployments should use a shared database (future `authz_store: db`).
- The published synchronous hook `ResponseTransformContext.auth.hasPermission()` is a role-only approximation of that schema (`roleHasPermission()`). Use `authz.check()` for access decisions.
