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

Inline editing lets logged-in admins edit page content directly in the public site view — no need to navigate to the admin panel, find the page, and use the full editor. Hover the title or body content and click the floating **✎ Edit** button to rename the title in place, or rewrite the body with a TipTap WYSIWYG editor that edits the Markdown source directly — with a bubble formatting toolbar, live presence, and conflict-free merging when several admins edit at once.

## Installation

Inline editing is provided by the `@dune/plugin-inline-edit` plugin. Add it to your site to enable the admin bar and all editing features.

**1. Add the plugin to `site.yaml`:**

```yaml
plugins:
  - src: "jsr:@dune/plugin-inline-edit@^2"
```

**2. Add the import to your site's `deno.json`:**

```json
{
  "imports": {
    "@dune/plugin-inline-edit": "jsr:@dune/plugin-inline-edit@^2"
  }
}
```

Without the plugin, admins see no admin bar and no edit controls on the public site.

## How it activates

Log in at `/admin`. From that point forward, every page you browse on your site gets an **admin bar** injected at the top — a thin fixed strip showing the page title, editing controls, and a link back to the full admin editor.

No template changes needed. The admin bar is injected server-side into the response whenever a valid admin session is detected. It appears on the default-language pages and on all language-prefixed variants (`/de/`, `/fr/`, etc.) alike.

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
- A WYSIWYG editor replaces the page body — select text to see the **bubble formatting toolbar** appear above the selection
- **Save** writes the changes back to the Markdown source on disk, then reloads the page
- **Cancel** or **Escape** discards unsaved changes

Other admins editing the same page at the same time are shown in the toolbar. Their changes sync live over a WebSocket and merge without conflicts via a CRDT operating on the Markdown source.

If the sync connection is unavailable, the editor falls back to standalone mode: editing and saving still work, presence and live merging don't (the toolbar shows "offline").

### Bubble formatting toolbar

Selecting any text while the body editor is open shows a small floating toolbar just above the selection with one-click access to all common formats:

| Group | Buttons |
|-------|---------|
| Inline | **B** Bold · *I* Italic · ~~S~~ Strikethrough · `` ` `` Inline code |
| Headings | H₁ · H₂ · H₃ |
| Blocks | • Bullet list · 1. Ordered list · ☐ Task list · ❝ Blockquote · {} Code block |
| Insert | 🔗 Link · 🖼 Image · ⊞ Table |

Active marks highlight in the toolbar so you can see at a glance what formatting applies at the cursor. Clicking an active button toggles it off.

**Link and Image** switch the toolbar to a URL input field. Type or paste the URL and press Enter (or click ✓) to apply; press ← to cancel and return to the formatting buttons. For links, if the cursor is already inside a link the current href is pre-filled. The ✕ button removes the link without leaving the input view.

**Table** inserts a 3×3 table with a header row at the cursor position.

All content round-trips through the Markdown source losslessly — `[text](url)`, `![alt](url)`, GFM table syntax, and `- [ ] task` checkboxes are preserved exactly as written.

## What is editable

| What | How identified | Edit behaviour |
|------|---------------|----------------|
| Page title | First `<h1>` on the page | Contenteditable in place, auto-saves on blur |
| Body content | The element your theme marks with `data-dune-body` | WYSIWYG with bubble toolbar, floating Save/Cancel, live collab |

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
- **Image insertion requires a URL.** The toolbar does not include a media uploader — paste the URL of an already-hosted image. Upload via the admin media panel first if needed.
