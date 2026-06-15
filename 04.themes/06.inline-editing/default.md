---
title: "Inline Editing in Themes"
published: true
visible: true
taxonomy:
  audience: [developer]
  difficulty: [intermediate]
  topic: [themes, islands, editing]
metadata:
  description: "Mark editable regions in your theme templates using typed components from @dune/core/ui/editable"
---

# Inline Editing in Themes

Dune's inline editing is driven by **marker components** — server-only components from `@dune/core/ui/editable` that annotate editable regions in your templates. They render `data-dune-*` attributes into the HTML, which editor plugins (like `@dune/plugin-inline-edit`) consume to mount their editors.

The components ship no JavaScript, imply no specific editor, and the page renders identically whether or not an editor plugin is installed. Markers are stripped from HTML served to anonymous visitors — only logged-in admins with a valid editing session see them.

## Marker components

Import from `@dune/core/ui/editable`. Your theme already depends on `@dune/core`, so no additional dependency is needed.

### EditableMarkdown

Marks the rendered Markdown body. The editor mounts a WYSIWYG over the page's Markdown source; changes are serialised back to Markdown losslessly — the rendered HTML is never reverse-engineered.

```tsx
import { EditableMarkdown } from "@dune/core/ui/editable";

<EditableMarkdown sourcePath={page.sourcePath}>
  <div dangerouslySetInnerHTML={{ __html: await page.html() }} />
</EditableMarkdown>
```

| Prop | Type | Description |
|------|------|-------------|
| `sourcePath` | `string` | Page source path (`page.sourcePath`) |
| `as` | `string` | Wrapper element tag (default `div`) |
| `children` | `ComponentChildren` | The rendered body — shown as-is to all visitors |

Renders `<div data-dune-body data-dune-source="…">`. The `<h1>` title on the page is detected automatically and linked to the `title` frontmatter field — no additional marker needed for it.

**Only mark one body region per template.** Listing and landing templates with no editable Markdown body should not carry the marker — leave it out and body editing is simply unavailable for those pages.

### EditableText

Marks an inline frontmatter field for in-place text editing.

```tsx
import { EditableText } from "@dune/core/ui/editable";

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
| `as` | `string` | Wrapper element tag (default `span`) |
| `children` | `ComponentChildren` | Current value — rendered as-is for all visitors |

Renders `<span data-dune-field="…" data-dune-source="…">`.

### EditableDate and EditableField

Typed field markers. They add a `data-dune-field-type` hint so editor plugins can mount a type-appropriate editor:

```tsx
import { EditableDate, EditableField } from "@dune/core/ui/editable";

<time>
  <EditableDate field="date" sourcePath={page.sourcePath}>{fm.date}</EditableDate>
</time>

<EditableField field="rating" type="star_rating" sourcePath={page.sourcePath}>
  {fm.rating}
</EditableField>
```

> The current inline-edit plugin edits all field markers as plain text. Type-specific editors (date picker, media picker, custom types) are read from the `data-dune-field-type` hint as the plugin grows — annotate now and richer editors arrive without template changes.

### Opting an element out of editing

Add `data-dune-no-edit` to any element that should not become an edit target — for example, a site logo that happens to be an `<h1>`:

```tsx
<h1 class="site-logo" data-dune-no-edit>My Site</h1>
```

## Full example: article template

```tsx
/** @jsxImportSource preact */
import { EditableText, EditableMarkdown } from "@dune/core/ui/editable";
import type { TemplateProps } from "@dune/core/types";
import Layout from "../components/layout.tsx";

export default function ArticleTemplate({ page, site, nav, Layout: DynamicLayout, children }: TemplateProps) {
  const LayoutComponent = DynamicLayout ?? Layout;
  const fm = page?.frontmatter ?? {};

  return (
    <LayoutComponent site={site} page={page} nav={nav}>
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

        <EditableMarkdown sourcePath={page.sourcePath}>
          <div dangerouslySetInnerHTML={{ __html: await page.html() }} />
        </EditableMarkdown>
      </article>
    </LayoutComponent>
  );
}
```

Anonymous visitors see the plain rendered article. Admins (with the inline-edit plugin installed) see ✎ Edit handles on the title, date, and body.

## Raw attributes

The marker components render `data-dune-*` HTML attributes — you can write those attributes directly when you need precise control over the element (e.g. you want `<article data-dune-body>` rather than a wrapping `<div>`):

```tsx
<article data-dune-body data-dune-source={page.sourcePath}>
  <div dangerouslySetInnerHTML={{ __html: await page.html() }} />
</article>
```

The full attribute vocabulary:

| Attribute | Set by component | Meaning |
|-----------|-----------------|---------|
| `data-dune-body` | `EditableMarkdown` | Marks the Markdown body region |
| `data-dune-field="key"` | `EditableText`, `EditableDate`, `EditableField` | Frontmatter key to patch |
| `data-dune-field-type="type"` | `EditableDate`, `EditableField` | Type hint for the editor |
| `data-dune-source="path"` | All components | Content file source path |
| `data-dune-no-edit` | — | Excludes an element from edit detection |

Prefer the components in new templates. Raw attributes are useful for wrapping elements with semantic meaning, or when integrating with a design system where the wrapper element matters.

## How it works

1. **Render time** — marker components emit `data-dune-*` attributes into the page HTML.
2. **Serve time** — Dune scrubs all `data-dune-*` attributes from responses that don't belong to a validated editing session. Anonymous visitors and crawlers never see them.
3. **Admin requests** — markers stay in the HTML, and the editor plugin injects its script into the response.
4. **In the browser, admin only** — the script finds the markers and mounts its editors. The TipTap stack loads only when an edit actually starts.

Because plugins consume markers rather than being imported by templates, themes stay portable across sites with or without an editor plugin — and editor plugins are swappable without touching any template.

> Never use `data-dune-*` attributes as CSS or JavaScript hooks for public site styling. They are stripped from public responses and their presence is not guaranteed. To inspect your markers, view the page logged in as an admin.

## Relative links in content

Folder pages serve at trailing-slash URLs (`/blog/my-post/`), so relative links written directly in Markdown resolve correctly without server-side rewriting:

```markdown
![Photo](./photo.jpg)          → /blog/my-post/photo.jpg  ✓
[Related post](./related/)     → /blog/my-post/related/   ✓
[Up to blog](../)              → /blog/                    ✓
```

This is the same convention browsers use for HTML documents in directories. No special syntax is required — standard Markdown relative links just work.

In multilingual sites, relative links are resolved at render time and then passed through the internal link rewriter, which adds the language prefix (`/de/blog/my-post/`). Cross-language links still need to be root-relative (`/de/other-page/`).
