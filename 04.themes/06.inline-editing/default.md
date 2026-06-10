---
title: "Inline Editing in Themes"
published: true
visible: true
taxonomy:
  audience: [developer]
  difficulty: [intermediate]
  topic: [themes, islands, editing]
metadata:
  description: "Support inline editing in your theme templates using the component kit, or rely on auto-overlay for zero-config editing"
---

# Inline Editing in Themes

Dune's inline editing works in two modes for theme developers:

1. **Auto-overlay** — zero template changes. Dune detects editable elements automatically based on semantic HTML.
2. **Component kit** — explicit, precise. Import `EditableText`, `EditableMarkdown`, etc. from `@dune/core/ui/editable` directly into your templates.

Both modes work simultaneously. The component kit takes precedence wherever it is used; auto-overlay handles everything else.

## Auto-overlay: what themes get for free

When an admin is logged in, Dune scans the rendered HTML of each page and annotates:

- The first `<h1>` → title field (click-to-edit, auto-saves)
- The first `<article>` element, or the first `<div>` with a standalone `content` CSS class → body content (click-to-edit, floating Save/Cancel)

**For a well-structured theme, this works without any changes.** An article template that renders:

```tsx
<article class="post">
  <h1>{fm.title}</h1>
  <div class="body" dangerouslySetInnerHTML={{ __html: await page.html() }} />
</article>
```

gets title and body editing automatically, with the edit controls positioned precisely at the `<h1>` and the body `<div>`.

### Helping the auto-overlay

Follow semantic HTML conventions and the overlay works without any explicit annotation:

- Use `<article>` for the main content container
- Use `<h1>` for the page title (one per page)
- Avoid naming layout wrappers with `content` in their CSS class (e.g. prefer `content-header` over a standalone `content` class on navigation)

### Opting elements out

If your layout contains elements that match the auto-detection rules but are not editable (e.g. a layout `<main>` wrapper that includes navigation), add `data-dune-no-edit`:

```tsx
<main class="site-layout" data-dune-no-edit>
  {/* Dune skips this and continues scanning inward */}
  <article class="page-content">
    <h1>{fm.title}</h1>
    ...
  </article>
</main>
```

The scanner skips any element with `data-dune-no-edit` and continues to the next candidate. This is preferable to relying on class names — it is explicit and does not break if you rename your CSS classes.

## Component kit

For precise control over which elements are editable and how, import the component kit islands into your templates. Components render their children identically for anonymous visitors — zero overhead, no JS shipped. Editing controls only activate when an admin session is present.

### Installation

The component kit is part of `@dune/core`. However, the edit handles only activate when the admin bar is present — which requires `@dune/plugin-inline-edit` to be installed. Without the plugin the components render their children identically for all visitors, including admins.

See [Inline editing](administration/inline-editing#installation) for how to install the plugin.

Add the required import map entries to your site's `deno.json` if not already present (see [Islands](themes/islands#deno-json-import-map)):

```json
{
  "imports": {
    "preact": "npm:preact@^10",
    "preact/hooks": "npm:preact@^10/hooks",
    "preact/jsx-runtime": "npm:preact@^10/jsx-runtime",
    "preact/jsx-dev-runtime": "npm:preact@^10/jsx-dev-runtime"
  }
}
```

### EditableText

Makes a frontmatter field editable in place. Click to edit, auto-saves on blur.

```tsx
import { EditableText } from "@dune/core/ui/editable";

// In your template:
<h1>
  <EditableText field="title" sourcePath={page.sourcePath}>
    {fm.title}
  </EditableText>
</h1>

<p class="byline">
  <EditableText field="author" sourcePath={page.sourcePath}>
    {fm.author ?? "Unknown"}
  </EditableText>
</p>
```

| Prop | Type | Description |
|------|------|-------------|
| `field` | `string` | Frontmatter key to patch on save |
| `sourcePath` | `string` | Page source path (`page.sourcePath`) |
| `children` | `ComponentChildren` | Current value — rendered as-is for all visitors |

### EditableMarkdown

Full WYSIWYG body editing using TipTap, backed by Y.js for real-time collaboration. Multiple admins can edit the same page simultaneously — changes are merged without conflicts.

```tsx
import { EditableMarkdown } from "@dune/core/ui/editable";

<article class="post-body">
  <EditableMarkdown sourcePath={page.sourcePath} />
</article>
```

The component renders the page's HTML body for anonymous visitors. For admins in edit mode it activates a TipTap editor with a formatting toolbar.

| Prop | Type | Description |
|------|------|-------------|
| `sourcePath` | `string` | Page source path |

> `EditableMarkdown` requires a WebSocket connection to `/admin/collab/edit-ws` for Y.js sync. Make sure the admin panel is enabled (`admin.enabled: true`).

### EditableImage

A media picker for image frontmatter fields. Displays the current image; clicking opens the media library to select a replacement.

```tsx
import { EditableImage } from "@dune/core/ui/editable";

<EditableImage
  field="cover"
  sourcePath={page.sourcePath}
  src={fm.cover}
  alt={fm.title}
  class="post-cover"
/>
```

### EditableDate

A date picker for date frontmatter fields.

```tsx
import { EditableDate } from "@dune/core/ui/editable";

<time>
  <EditableDate field="date" sourcePath={page.sourcePath}>
    {formatDate(fm.date)}
  </EditableDate>
</time>
```

### EditableField (generic)

Looks up the right editor component by field type from the registry. Useful for custom field types registered by plugins.

```tsx
import { EditableField } from "@dune/core/ui/editable";

<EditableField field="rating" sourcePath={page.sourcePath} value={fm.rating} />
```

### Custom field editors

Register a Preact island component as the editor for a custom field type:

```ts
// themes/my-theme/islands/StarRating.tsx
import { registerFieldEditor } from "@dune/core/ui/editable";
registerFieldEditor("star_rating", StarRatingIsland);
```

Any `EditableField` with a matching field type will use your component.

## Full example: article template

```tsx
/** @jsxImportSource preact */
import { EditableText, EditableMarkdown } from "@dune/core/ui/editable";

export default function ArticleTemplate({ page, site, nav, Layout }: any) {
  if (!Layout) return null;
  const fm = page?.frontmatter ?? {};

  return (
    <Layout site={site} page={page} nav={nav}>
      <article class="article">
        <header>
          <h1>
            <EditableText field="title" sourcePath={page.sourcePath}>
              {fm.title}
            </EditableText>
          </h1>
          <time>
            <EditableText field="date" sourcePath={page.sourcePath}>
              {fm.date}
            </EditableText>
          </time>
        </header>

        {/* Body: full WYSIWYG with Y.js real-time collaboration */}
        <EditableMarkdown sourcePath={page.sourcePath} />
      </article>
    </Layout>
  );
}
```

Anonymous visitors see the rendered article. Admins see click-to-edit controls on the title and date, and a TipTap editor on the body.

## Auto-overlay vs component kit

| | Auto-overlay | Component kit |
|---|---|---|
| Template changes required | None | Yes |
| Precision | Heuristic | Exact |
| Body editing | HTML → Markdown round-trip | TipTap WYSIWYG (no round-trip) |
| Real-time collaboration | No | Yes (Y.js) |
| Custom fields | No | Yes (registry) |
| Works on any template | Yes | Requires explicit wiring |

Use the auto-overlay as the default. Add component kit components where you need precision, richer editing, or real-time collaboration.

## How islands are discovered

Component kit components are Preact islands. Dune discovers them automatically — you do not need to register them explicitly:

- Files in `themes/{name}/islands/` are bundled at startup by `collectThemeIslands()`
- Imports from `@dune/core/ui/editable` resolve to islands inside the `@dune/core` package, which are collected the same way
- See [Islands](themes/islands) for the full discovery and bundling details
