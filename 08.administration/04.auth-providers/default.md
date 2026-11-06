---
title: "Authentication Providers"
published: true
visible: true
taxonomy:
  audience: [webmaster, developer]
  difficulty: [advanced]
  topic: [administration, security, authentication]
metadata:
  description: "Use the built-in local accounts, or plug in your own identity provider for admin panel authentication"
---

# Authentication Providers

By default, Dune manages admin users with local password accounts (PBKDF2-SHA256). For organizations that already have a central directory service, you can delegate authentication to an identity provider you write.

The `AuthProvider` interface abstracts the identity verification step. Once a provider confirms who the user is, Dune's session management and permission system take over — all existing roles, revision history, and audit logging work unchanged.

> **LDAP and SAML are not built in.** Earlier versions listed them in the configuration but never implemented them (the providers threw), so they were removed in 0.36 and `admin.auth_provider.type` accepts only `local`. To use LDAP, SAML, OpenID Connect or an internal SSO, write a provider and inject it as shown below.

---

## Default: local accounts

No configuration needed. Users are managed in Admin → Users.

```yaml
# site.yaml — this is the implicit default
admin:
  # auth_provider: not set → local passwords
```

---

## Building a custom provider

Implement the `AuthProvider` interface from `src/admin/auth/provider.ts`:

```typescript
import type { AuthProvider, AuthCredentials, AuthProviderUser } from "./provider.ts";

export class MyAuthProvider implements AuthProvider {
  readonly type = "my-sso";  // any string; "local" is the built-in password provider

  async authenticate(creds: AuthCredentials): Promise<AuthProviderUser | null> {
    // Verify credentials against your system
    // Return null on failure, AuthProviderUser on success
    const user = await myIdentityService.verify(creds.username, creds.password);
    if (!user) return null;
    return {
      externalId: user.id,
      username: user.username,
      email: user.email,
      name: user.displayName,
      roles: ["editor"],
    };
  }
}
```

Build the admin plugin yourself, give it the provider, and pass it to `bootstrap()` instead of listing the admin in `site.yaml`:

```typescript
// main.ts (headless mode) or your own entry point
import { bootstrap } from "@dune/core";
import { createAdminPlugin } from "@dune/plugin-admin";
import { MyAuthProvider } from "./auth/my-auth-provider.ts";

const root = Deno.cwd();
const ctx = await bootstrap(root, {
  plugins: [createAdminPlugin({ root, dev: false, authProvider: new MyAuthProvider() })],
});
```

The injected provider takes precedence over anything set in `system.yaml`. This is the way to use any identity source other than the built-in local accounts (LDAP, SAML, OpenID Connect, internal SSO, ...).

### `AuthProviderUser` fields

| Field | Type | Description |
|-------|------|-------------|
| `externalId` | `string` | Stable external ID (an LDAP DN, SAML NameID, OIDC `sub`, ...) |
| `username` | `string` | Login name — used to find/create the local user |
| `email` | `string?` | Used in the local user record |
| `name` | `string?` | Display name |
| `roles` | `string[]?` | Roles to assign/sync on each login — `findOrProvisionUser()` picks the highest-ranked valid admin `Role` out of this list |

---

## Configuration reference

```yaml
# config/system.yaml
admin:
  auth_provider:
    type: local   # the only value; omit the section entirely for the same result
```

Selection priority:
1. `authProvider` passed to `createAdminPlugin()` — highest priority, ignores config
2. Default — local password authentication

Any other `admin.auth_provider.type` is a configuration error (`dune validate` reports it and the site refuses to start).
