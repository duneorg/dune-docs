---
title: "Content Workflow"
published: true
visible: true
taxonomy:
  audience: [editor, webmaster]
  difficulty: [intermediate]
  topic: [content, workflow]
metadata:
  description: "Managing content status with built-in or fully customizable workflow stages, role-based transitions, and scheduling"
---

# Content Workflow

Dune includes a built-in editorial workflow with four content states. You can also define fully custom stages and transitions with per-role permissions — no code required, just YAML config.

## States

| Status | Description |
|--------|-------------|
| `draft` | Work in progress — not visible to visitors (unless `published: true` is also set) |
| `in_review` | Submitted for editorial review — not yet public |
| `published` | Approved and live — visible to visitors |
| `archived` | Removed from circulation — not visible, kept for historical reference |

The `status` frontmatter field sets a page's workflow state:

```yaml
---
title: "My New Article"
status: draft
---
```

## Allowed transitions

Not all state changes are valid. The engine enforces the following transition graph:

```
draft ──────────→ in_review ──→ published ──→ archived
  ↑                   │                          │
  └───────────────────┘←─────────────────────────┘
  ↑                                              │
  └──────────────────────────────────────────────┘
```

Specifically:

| From | To | Notes |
|------|----|-------|
| `draft` | `in_review` | Submit for review |
| `draft` | `published` | Publish directly (bypass review) |
| `in_review` | `published` | Approve and publish |
| `in_review` | `draft` | Return to author for changes |
| `published` | `archived` | Archive published content |
| `published` | `draft` | Unpublish and return to draft |
| `archived` | `draft` | Restore from archive |

Any other transition (e.g. `archived → published`) is invalid and will be rejected.

---

## Custom workflows

Override the built-in 4-stage workflow by adding a `workflow:` block to `site.yaml`. Define your own stages and transitions with labels, colors, and per-role permissions.

### Defining stages

```yaml
# site.yaml
workflow:
  stages:
    - id: "draft"
      label: "Draft"
      color: "amber"
    - id: "in_review"
      label: "In Review"
      color: "blue"
    - id: "needs_changes"
      label: "Needs Changes"
      color: "orange"
    - id: "approved"
      label: "Approved"
      color: "teal"
    - id: "published"
      label: "Published"
      color: "green"
      publish: true       # sets published: true on transition in
    - id: "archived"
      label: "Archived"
      color: "gray"
      terminal: true

  transitions:
    - from: "draft"
      to: "in_review"
      label: "Submit for Review"
      roles: ["admin", "editor", "author"]

    - from: "in_review"
      to: "approved"
      label: "Approve"
      roles: ["admin"]

    - from: "in_review"
      to: "needs_changes"
      label: "Request Changes"
      roles: ["admin"]

    - from: "needs_changes"
      to: "in_review"
      label: "Resubmit"
      roles: ["admin", "editor", "author"]

    - from: "approved"
      to: "published"
      label: "Publish"
      roles: ["admin"]

    - from: "published"
      to: "archived"
      label: "Archive"
      roles: ["admin"]

    - from: "published"
      to: "draft"
      label: "Unpublish"
      roles: ["admin"]

    - from: "archived"
      to: "draft"
      label: "Restore"
      roles: ["admin"]
```

### Stage fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | `string` | Unique identifier — stored as the `status` value in frontmatter |
| `label` | `string` | Display name in admin UI |
| `color` | `string` | Badge color: `amber`, `blue`, `green`, `gray`, `orange`, `teal`, `red`, `purple` |
| `publish` | `boolean` | If `true`, transitioning to this stage sets `published: true` on the page |
| `terminal` | `boolean` | Informational marker — no effect on transitions |

### Transition fields

| Field | Type | Description |
|-------|------|-------------|
| `from` | `string` | Source stage ID |
| `to` | `string` | Target stage ID |
| `label` | `string` | Button label in admin UI (e.g. "Approve", "Request Changes") |
| `roles` | `string[]` | Roles allowed to perform this transition. If omitted, any role can perform it. Valid: `admin`, `editor`, `author` |

### Role-based permissions

When `roles` is set on a transition, only users with one of those roles see the transition button and can perform the action. The server enforces this — a role mismatch returns `403 Forbidden`.

Omitting `roles` means any authenticated admin user can perform the transition.

### REST API

```
GET /admin/api/workflow/stages
```

Returns all configured stages. No authentication required (used by the admin UI to render status badges on page lists).

```
GET /admin/api/workflow/status/{path}
```

Returns the current status, all allowed transitions (filtered to the requesting user's role), and the full stages list.

```json
{
  "sourcePath": "content/blog/my-post/default.md",
  "status": "in_review",
  "allowedTransitions": ["approved", "needs_changes"],
  "transitions": [
    { "to": "approved", "label": "Approve" },
    { "to": "needs_changes", "label": "Request Changes" }
  ],
  "stages": [...]
}
```

```
POST /admin/api/workflow/transition
```

```json
{ "sourcePath": "content/blog/my-post/default.md", "status": "approved" }
```

Returns `403` if the requesting user's role is not permitted to make the transition.

---

## Status vs. `published`

`status` and `published` are independent fields:

- `published: false` — page has no URL and does not exist for visitors, regardless of status
- `published: true` with `status: draft` — technically accessible via URL, but the workflow treats it as a draft

In practice, use `status` for editorial tracking and use `published: false` for pages that should be completely hidden.

## Scheduling

A page can have a publish window in its frontmatter:

```yaml
publish_date: "2025-12-01T09:00:00Z"    # not live before this time
unpublish_date: "2026-01-01T00:00:00Z"  # not live from this time on
```

The page is live when `publish_date <= now < unpublish_date`; either date may be left out. Outside the window it behaves like `published: false`: no route, and it is left out of feeds, the sitemap, search and static builds. `published: false` always wins; the dates only narrow it to a window.

Dates are ISO 8601. A value without a time zone, including a bare `2026-12-01`, is read as UTC. A date that can't be parsed is ignored and logged. Nothing is written back to the file, and no admin panel is needed: a running `dune serve` rebuilds the index at the next boundary, so a page appears and disappears on time without a restart. A static build (`dune build --static`) reflects the moment it ran.

The admin panel can also schedule a publish or unpublish for a page. Those are stored separately as scheduled actions, applied by the scheduler in a running `dune serve`, and don't execute during one-shot commands like `dune build`.

## Draft preview

Before publishing, editors can preview how a draft will look on the live site — with the active theme, layout, and styles applied — without making the page public.

### Creating a preview

From the page editor, save a draft and click **Preview Draft**. This generates a secure, shareable preview URL:

```
https://example.com/__preview?path=content%2Fblog%2Fmy-post%2Fdefault.md&token=abc123…
```

The URL can be shared with reviewers who don't have admin access — no login required.

### How it works

- Draft content (frontmatter + body) is saved to the staging area in `system.runtime_dir` (default `.dune/admin/staging/`).
- A random opaque token is generated and tied to the draft. The token is preserved across subsequent saves so shared links remain valid.
- `GET /__preview` verifies the token and renders the draft through the active theme, with an orange "Draft preview" banner injected so viewers know the page is not live.
- Staging files are ephemeral — they live in `runtime_dir` and are not committed to version control.

### Publishing a staged draft

Clicking **Publish** in the panel writes the staged draft to the content files, records a revision, and optionally creates a git commit (see [Git auto-commit](#git-auto-commit) below). The staging file is removed after a successful publish.

### Git auto-commit

When `git_commit: true` is set in the admin plugin's config, every page save and staged publish triggers a `git add` + `git commit` automatically:

```yaml
# config/site.yaml
plugins:
  - src: jsr:@dune/plugin-admin@^4
    config:
      git_commit: true
```

The commit message includes the page path and the editor's username:

```
Admin: update content/blog/my-post/default.md (by jane)
```

Git must be available in the server's PATH. Commit failures are logged as warnings and do not block the save.

## Revision history

Every time a page is saved through the admin panel, a revision is recorded. Revisions store:
- Complete page content at that point in time
- Frontmatter snapshot
- Author who made the change
- Timestamp
- Optional change message

### Storage

Revisions are written to the runtime data directory (configured as `system.runtime_dir`, default `.dune/admin`). Each page gets its own folder:

```
.dune/admin/history/{url-encoded-source-path}/{revision-number}.json
```

This directory is ephemeral — it should be in `.gitignore` and is not committed to version control.

> **⚠️ Deploy warning**: Because revisions live in `runtime_dir`, **they are not preserved across deployments to a fresh server**. Every cold deploy (new container, new VM, fresh clone) starts with an empty revision history. If you need persistent revision history across deploys, back up the `runtime_dir` before deployment and restore it after, or mount it on persistent storage.

### Capacity

Up to 50 revisions per page are kept by default. Older revisions are automatically pruned when a new save would exceed the limit. Configure the limit via `system.max_revisions`:

```yaml
# config/system.yaml
max_revisions: 100   # keep more history (uses more disk); 0 keeps all
```

### Accessing revisions

Revisions are accessible from the admin panel's page editor: open any page, then click **History** to browse the revision timeline, compare versions, and restore a previous version.

The admin panel also exposes revisions via its internal API (used by the panel UI):

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/admin/api/pages/{route}/history` | List all revisions for a page |
| `GET` | `/admin/api/pages/{route}/history/{n}` | Get a specific revision |
| `POST` | `/admin/api/pages/{route}/history/{n}/restore` | Restore revision `n` as the current content |

These endpoints require admin authentication (editor role or above).
