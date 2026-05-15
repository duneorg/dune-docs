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
| `access` | member, admin, editor, author, owner |
| `edit` | owner, admin, editor |
| `pages.update` | admin, editor |
| `users.manage` | admin |
| `media.upload` | admin, editor, author |

## Config

```yaml
# site.yaml
auth:
  mode: dune
  authzStore: local    # default — data/permissions/*.json (in-memory index)
```

`authzStore` is independent of `userStore`. Default is `local` — permission tuples are stored as flat JSON files in `data/permissions/` and indexed in-memory on startup.

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

## Bootstrap from existing users

On first startup after authz is introduced, Dune automatically derives permission tuples from the `roles[]` array on existing `SiteUser` records. This is idempotent — it does not create duplicates.

After bootstrap, **tuples are the authority** for `authz.check()`. The `roles[]` array on `SiteUser` remains in sync as a cache (it is still updated when roles change) but `authz.check()` is the correct enforcement path.

## Tuple storage

With `authzStore: local`, tuples are stored as JSON files:

```
data/permissions/
  {uuid}.json   →  { id, subject, relation, object }
```

The in-memory index is rebuilt from files on restart — no KV dependency. Do not edit these files directly; use `authz.allow()`, `authz.addMember()`, `authz.disallowAllMatching()`.

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

## external-jwt mode

In `auth.mode: external-jwt`, roles come from JWT claims — the authz tuple store is not consulted. Do not call `authz.addMember()` in this mode; it writes to the local store, which is never read.

## Limitations

- `authzStore: local` is single-process only. Multi-process deployments should use a shared database (future `authzStore: db`).
- No admin panel integration yet — admin permissions still use the flat `ROLE_PERMISSIONS` model. Polizy-backed admin authz is planned.
