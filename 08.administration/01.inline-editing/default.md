---
title: "Inline Editing"
published: true
visible: true
taxonomy:
  audience: [editor, webmaster]
  difficulty: [beginner]
  topic: [admin, editing]
metadata:
  description: "Edit page content directly in the browser without opening the admin panel — click any title or body text to start editing"
---

# Inline Editing

Inline editing lets logged-in admins edit page content directly in the public site view — no need to navigate to the admin panel, find the page, and use the full editor. Click a title to rename it in place. Click body text to rewrite it with a TipTap WYSIWYG editor backed by Y.js for real-time collaboration.

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
| **Save** | Commits any pending Y.js-based edits to the page file |
| **Open in admin →** | Opens the full admin editor for this page |

## Editing the title

In edit mode, hover over any `<h1>` heading — a dashed blue outline appears. Click it to make it editable in place:

- Type to change the title
- **Enter** or clicking away saves and auto-patches the `title` frontmatter field
- **Escape** cancels and restores the original text

The save is debounced — it writes to disk after you stop typing, not on every keystroke.

## Editing the body

In edit mode, hover over the page body content — a dashed blue outline appears. Click anywhere in the body text to open a TipTap WYSIWYG editor:

- A floating **Save / Cancel** toolbar appears, sticky below the admin bar as you scroll
- Edit freely — type, delete, use the formatting toolbar or browser shortcuts (bold, italic, etc.)
- Changes are synced in real time via Y.js — multiple admins can edit the same page simultaneously without conflicts
- **Save** writes the changes back to the Markdown source on disk, then reloads the page
- **Cancel** or **Escape** discards unsaved changes

## What gets detected automatically

The auto-overlay scans each page for editable elements:

| What | How detected | Edit behaviour |
|------|-------------|----------------|
| Page title | First `<h1>` on the page | Contenteditable in place, auto-saves on blur |
| Body content | First `<article>` element, or first `<div>` with a standalone `content` CSS class | TipTap WYSIWYG with floating Save/Cancel toolbar |

If neither is found, the auto-overlay does nothing — no controls appear. This is intentional: Dune does not guess at content it cannot confidently identify.

## Opting elements out

If an element matches the auto-detection rules but should not be editable (navigation links, layout chrome, etc.), add `data-dune-no-edit` to it:

```html
<main class="main-content" data-dune-no-edit>
  <!-- layout wrapper — Dune skips this and continues scanning -->
</main>
```

The scanner skips any element with `data-dune-no-edit` and continues looking for the next candidate. This lets layout wrappers opt out while inner content elements remain detectable.

## Explicit inline editing with the component kit

The auto-overlay is a best-effort heuristic. For precise control — specific frontmatter fields, custom field types, real-time collaborative editing — use the component kit from `@dune/plugin-inline-edit/ui/editable` directly in your theme templates.

See [Inline editing in themes](themes/inline-editing) for how to use `EditableText`, `EditableMarkdown`, and the other component kit components.

## Limitations

- **Auto-overlay is heuristic.** It works well for standard templates following semantic HTML conventions (`<article>` for content, `<h1>` for the title). Templates with unconventional structure may need explicit component kit annotations.
- **One page at a time.** The inline editor operates on the currently viewed page. There is no cross-page or bulk editing.
- **Admin session required.** The admin bar and all editing controls are invisible to anonymous visitors and require an active admin session.
