---
title: "Inline Editing"
published: true
visible: true
taxonomy:
  audience: [editor, webmaster]
  difficulty: [beginner]
  topic: [admin, editing]
metadata:
  description: "Edit page content directly in the browser without opening the admin panel — hover an editable region and click ✎ Edit"
---

# Inline Editing

Inline editing lets logged-in admins edit page content directly in the public site view — no need to navigate to the admin panel, find the page, and use the full editor. Hover the title or body content and click the floating **✎ Edit** button to rename the title in place, or rewrite the body with a TipTap WYSIWYG editor that edits the Markdown source directly — with live presence and conflict-free merging when several admins edit at once.

## Installation

Inline editing is provided by the `@dune/plugin-inline-edit` plugin. Add it to your site to enable the admin bar and all editing features.

**1. Add the plugin to `site.yaml`:**

```yaml
plugins:
  - src: "jsr:@dune/plugin-inline-edit"
```

**2. Add the import to your site's `deno.json`:**

```json
{
  "imports": {
    "@dune/plugin-inline-edit": "jsr:@dune/plugin-inline-edit@^1"
  }
}
```

Without the plugin, admins see no admin bar and no edit controls on the public site.

## How it activates

Log in at `/admin`. From that point forward, every page you browse on your site gets an **admin bar** injected at the top — a thin fixed strip showing the page title, editing controls, and a link back to the full admin editor.

No template changes needed. The admin bar is injected server-side into the response whenever a valid admin session is detected.

Log out (or clear cookies) and the bar disappears immediately — anonymous visitors never see it.

## Admin bar

The admin bar appears at the top of every page while you are logged in:

```
✦ DUNE  [page title]  ✎ Editing  Save  Open in admin →  username
```

| Control | Action |
|---------|--------|
| **✎ Editing / 👁 Preview** | Toggle between edit mode (outlines appear on editable elements) and preview mode (page looks exactly as visitors see it) |
| **Save** | Commits any pending edits to the page file |
| **Open in admin →** | Opens the full admin editor for this page |

## Starting an edit

Editing always starts from a floating **✎ Edit** button — never by clicking the content itself. Links and other interactive elements on the page keep working normally while you browse in edit mode; hovering an editable region shows a dashed outline and the button.

## Editing the title

In edit mode, hover over the page's `<h1>` heading and click the **✎ Edit** button:

- Type to change the title
- **Enter** or clicking away saves and auto-patches the `title` frontmatter field
- **Escape** cancels and restores the original text

The save is debounced — it writes to disk after you stop typing, not on every keystroke.

## Editing the body

In edit mode, hover over the page body content and click the **✎ Edit** button:

- A floating **Save / Cancel** toolbar appears, sticky below the admin bar as you scroll
- Edit freely — type, delete, use the formatting toolbar or browser shortcuts (bold, italic, etc.)
- Other admins editing the same page are shown in the toolbar; concurrent edits sync live over a WebSocket and merge without conflicts (Y.js CRDT over the Markdown source)
- **Save** writes the changes back to the Markdown source on disk, then reloads the page
- **Cancel** or **Escape** discards unsaved changes

If the sync connection is unavailable, the editor falls back to standalone mode: editing and saving still work, presence and live merging don't (the toolbar shows "offline").

While a body edit is active, links inside it are temporarily inert (standard editor behaviour) — they work again as soon as you save or cancel.

## What is editable

| What | How identified | Edit behaviour |
|------|---------------|----------------|
| Page title | First `<h1>` on the page | Contenteditable in place, auto-saves on blur |
| Body content | The element your theme marks with `data-dune-body` | WYSIWYG with floating Save/Cancel toolbar |

The body is **never auto-detected** — the theme must mark the element wrapping the rendered markdown body with `data-dune-body` (the starter theme does). This is intentional: a wrong guess (for example treating a blog listing's post cards as body content) would write template-generated HTML back into the page's Markdown source on save. If your theme doesn't mark a body element, body editing is simply unavailable for that page and a hint is logged to the browser console.

To exclude an `<h1>` from title editing (e.g. a site logo rendered as `<h1>`), add `data-dune-no-edit` to it.

See [Inline editing in themes](themes/inline-editing) for the theme author's side of this contract.

## Explicit inline editing with marker components

For precise control — specific frontmatter fields, typed field markers — themes can use the marker components from `@dune/core/ui/editable` in their templates.

See [Inline editing in themes](themes/inline-editing) for how to use `EditableText`, `EditableMarkdown`, and the other marker components.

## Limitations

- **Body editing requires theme support.** The theme must mark the body element with `data-dune-body` (or render the `EditableMarkdown` marker component). Themes that don't are read-only for body content.
- **One page at a time.** The inline editor operates on the currently viewed page. There is no cross-page or bulk editing.
- **Admin session required.** The admin bar and all editing controls are invisible to anonymous visitors and require an active admin session.
- **Concurrent edits merge at the Markdown level.** Simultaneous edits to different parts of a page merge cleanly; two admins rewriting the exact same sentence at the same moment merge character-wise, which can need a quick proofread.
