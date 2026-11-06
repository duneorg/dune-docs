---
title: "Administration"
published: true
visible: true
taxonomy:
  audience: [webmaster, editor]
  difficulty: [beginner]
  topic: [admin]
metadata:
  description: "Admin panel overview: authentication, roles, content editing, and media management"
---

# Admin Panel

The Dune admin panel is a browser-based interface for managing content, media, users, and site configuration without editing files directly.

> **Stability note**: The admin panel UI and its internal API endpoints (`/admin/api/…`) are functional but should be considered **beta**. Breaking changes to panel internals may occur in minor releases. The panel is intended for human editors, not programmatic integrations — use the [REST API](reference/api) for machine-to-machine access.

## Accessing the panel

By default the panel is at `/admin`. Dune does not create a default admin account: create the first admin yourself (see below), then log in with it. A site that has no enabled admin logs a warning at startup with the command to run.

> Before `@dune/plugin-admin` 3.3.0, first startup created a default `admin` account and wrote its password to a file. That no longer happens. Sites that already have an admin are unaffected.

### Creating the first admin

Create the first admin from the command line:

```bash
dune users:create admin@example.com --role admin --username admin --password-prompt
```

or from the environment, for container and server deployments:

```bash
DUNE_ADMIN_EMAIL=admin@example.com
DUNE_ADMIN_PASSWORD=...            # at least the minimum strength; never logged
DUNE_ADMIN_USERNAME=admin          # optional, default "admin"
```

If both `DUNE_ADMIN_EMAIL` and `DUNE_ADMIN_PASSWORD` are set and the site has no enabled admin, Dune creates one when it boots. This lives in core, so it works with any admin implementation. Rules:

- It only acts when **no enabled admin exists**. It never overrides or resets an existing account; to change a password use [`dune users:passwd`](reference/cli#users).
- Both variables must be set, or neither. A weak password, an invalid email, a username that is taken, or an email that belongs to a non-admin account stops boot with a message saying which.
- The password stays in the process environment. Remove `DUNE_ADMIN_PASSWORD` from your deployment config after the first boot.

Change the URL prefix with `admin.path` in your config:

```yaml
# dune.config.ts
export default {
  admin: {
    path: "/cms",   # panel available at /cms
  },
};
```

The admin panel is a plugin, `@dune/plugin-admin`, and core runs it only when `config/site.yaml` lists it under `plugins:` (`dune new` writes the entry, and `dune migrate:config` adds it to an existing site):

```yaml
# config/site.yaml
plugins:
  - src: jsr:@dune/plugin-admin@^4
```

> **Headless / API-only deployments:** do not list it. With no admin plugin there are no admin routes, no admin user pool and no related overhead, and core says once at boot that nothing provides the admin. (`admin.enabled` no longer exists; a config that still sets it is refused.) Advanced users can replace the admin with their own plugin that provides `auth.session`; see [Writing your own admin](/docs/extending/plugins#writing-your-own-admin-the-authsession-contract).

## Roles and permissions

Three roles control what each user can do:

| Role | Can do |
|------|--------|
| `admin` | Everything: content, media, users, config, form submissions |
| `editor` | Content CRUD, media management, read config and submissions. Cannot manage users or change config. |
| `author` | Create and edit pages, upload media, read submissions. Cannot delete pages, media, or access config. |

Users are stored in `data/users/` (under `data_dir` in `system.yaml`) and should be committed to version control. Passwords are stored as PBKDF2 hashes — never in plaintext.

There is no default account; see [Creating the first admin](#creating-the-first-admin). To change a password later, use the admin panel or `dune users:passwd`.

## What the panel provides

| Feature | Description |
|---------|-------------|
| **Inline editing** | Edit page titles and body content directly in the public site view — hover an editable region and click ✎ Edit while logged in. Requires the `@dune/plugin-inline-edit` plugin. See [Inline editing](administration/inline-editing). |
| **Content editor** | Create, edit, and delete pages. Supports Markdown, MDX, and frontmatter editing. Blueprint-driven custom fields are auto-generated from `blueprints/{template}.yaml`. |
| **Draft preview** | Save a draft and get a shareable preview URL that renders through the active theme. Reviewers don't need admin access. See [Draft preview](content/workflow#draft-preview). |
| **Workflow** | Move pages through `draft → in_review → published → archived` states. |
| **Scheduled actions** | Set a date/time for automatic publish or unpublish. |
| **Revision history** | Browse and restore previous versions of any page. Up to `system.max_revisions` (default 50) saved per page. The current revision count is shown in the editor toolbar. |
| **Comments & annotations** | Leave threaded comments on pages with @mention support and a resolve workflow for editorial sign-off. See [Comments](comments). |
| **Webhooks** | Fire outbound HTTP POST requests when pages are created, updated, deleted, or workflow status changes. See [Webhooks](webhooks). |
| **Flex Objects** | Create, edit, and delete records for schema-driven custom data types (products, team members, events, etc.). Schemas defined in `flex-objects/{type}.yaml`. See [Flex Objects](flex-objects). |
| **Configuration editor** | Edit `config/site.yaml` and `config/system.yaml` through a form UI without touching files directly (admin role only). |
| **Themes** | Browse installed and registry themes, preview any theme before switching, and install new themes from the marketplace. See `/admin/themes`. |
| **Media library** | Upload, browse, and delete media files co-located with content pages. |
| **Form submissions** | View submissions collected from blueprint-driven forms. See [Forms](forms). |
| **User management** | Create, edit, enable/disable admin users (admin role only). |

## Configuration editor

The configuration editor (admin role only) lets you edit `config/site.yaml` and `config/system.yaml` through a form UI at `/admin/config`. Fields are rendered based on the schema for each config file — no need to know the YAML syntax.

Changes are written back to the config files and take effect on next request (for system config) or after a content rebuild (for site config). The editor shows all recognised fields with their current values and inline help text.

## Themes

The **Themes page** (`/admin/themes`) lets you switch the active theme, preview alternatives before committing, and edit that theme's own settings. Select a theme from the dropdown and click **Preview** to see an iframe rendering of your site in that theme; click **Apply this theme** to make the switch. Below the switcher, **Theme settings** shows a schema-driven form for the active theme's own configuration (colors, layout options, etc., as defined by the theme).

From the same page, **Browse Marketplace** takes you to the full theme browser — your installed themes and a curated registry of community themes. You can install a theme directly from the registry — no restart required. See [Theme Preview & Marketplace](themes/preview-and-marketplace) for details.

## Revision history

Every save to a page creates a revision snapshot. The page editor toolbar shows a **History** button with the current revision count. Clicking it opens a timeline of all saved versions, with timestamps and author information.

From the history view you can restore any previous version — this creates a new revision rather than overwriting the current one, so you can always undo a restore by reverting again.

Revisions are stored in `system.runtime_dir` (default `.dune/admin/`) and are **not** committed to version control. The maximum number of revisions kept per page is controlled by `system.max_revisions` (default 50); older revisions are pruned automatically.

## Draft staging

Staging lets editors save work-in-progress content and share a live preview with reviewers — before the page goes public.

Saving a draft from the page editor stores a staging file in `runtime_dir/staging/`. Each draft gets a random opaque token. The preview URL is:

```
https://example.com/__preview?path={encoded-source-path}&token={token}
```

The token is stable across saves, so sharing the link once is enough — it stays valid as the editor continues working. When the draft is published, the staging file is removed.

Staging files are ephemeral (live in `runtime_dir`, not committed to git). They are lost if you wipe `runtime_dir` or deploy to a fresh server. To keep drafts across deploys, back up `runtime_dir` or mount it on persistent storage.

## Git auto-commit

When `admin.git_commit: true` is enabled, every page save and staging publish triggers an automatic `git add` + `git commit`:

```yaml
# dune.config.ts (or config/system.yaml)
export default {
  admin: {
    git_commit: true,
  },
};
```

The commit message includes the file path and editor username:

```
Admin: update content/blog/my-post/default.md (by jane)
```

Commit failures are logged as warnings and never block the save. Git must be available in the server's PATH.

## Sessions

Sessions are cookie-based and stored in `system.runtime_dir` (default `.dune/admin/sessions/`). They expire after `admin.session_lifetime` seconds (default 24 hours). Sessions are ephemeral — they are lost on restart or deploy.

The session cookie is `HttpOnly`, `SameSite=Strict`, and `Secure` (in production). In development (`DUNE_ENV=dev`), the `Secure` flag is omitted to allow HTTP.

## Security considerations

- Admin routes are isolated under `admin.path` and require authentication on every request
- Sessions use PBKDF2-hashed passwords with per-user salts
- The `admin` role is required to create other `admin`-role accounts — editors and authors cannot escalate their own privileges
- Form submission data lives in `data_dir` (git-tracked); session and revision data lives in `system.runtime_dir` (gitignored) — keep these locations separate
- If you expose the panel publicly, put it behind HTTPS and consider restricting access by IP at the infrastructure level for higher-security sites
