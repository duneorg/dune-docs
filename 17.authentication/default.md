---
title: "Authentication"
published: true
visible: true
taxonomy:
  audience: [developer, webmaster]
  difficulty: [intermediate]
  topic: [authentication, security]
metadata:
  description: "Dune's two authentication systems — admin panel auth and public site user auth"
---

# Authentication

Dune runs two completely separate authentication systems on the same server:

| System | Who it's for | Sessions stored at | Cookie name |
|--------|--------------|--------------------|-------------|
| **Admin auth** | Content editors, admins | `admin.runtimeDir/sessions/` | `dune-admin-session` |
| **Public auth** | Site visitors | `admin.runtimeDir/site-sessions/` | `dune-site-session` |

The two systems share no state, no user records, and no session cookies. An admin user is not a site user, and vice versa.

## Admin authentication

Covered in [Administration](../08.administration). Login at `{admin.path}/login` (default `/admin/login`). Users managed via the admin panel UI or `data/users/` YAML files.

## Public site authentication

Allows visitors to register and log in to your public site — for gated content, comments, subscriptions, or any user-specific feature. Three login methods are supported:

- **OAuth** — GitHub, Google, Discord
- **Magic link** — passwordless email link
- **External JWT** — validate tokens issued by Clerk, Auth0, or any HS256/RS256 issuer

See [Public Authentication](01.public-auth) for setup and usage.

## Content gating

Once a visitor is authenticated, their roles (assigned at login or after payment) control access to individual pages via the `roles:` frontmatter field.

See [Content Gating](02.content-gating) for configuration.

## Authorization (polizy)

Dune uses [polizy](https://github.com/bratsos/polizy) for relationship-based authorization — group membership, resource-level ownership grants, and hierarchy. The same `authz.check()` call backs both `roles:` frontmatter enforcement and custom route middleware.

See [Authorization](03.authorization) for configuration and usage, including the `external-jwt + authzStore: local` combination for using an external IdP for authentication while keeping polizy as the authorization authority.
