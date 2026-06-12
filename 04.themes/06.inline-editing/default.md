---
title: "Inline Editing in Themes"
published: true
visible: true
taxonomy:
  audience: [developer]
  difficulty: [intermediate]
  topic: [themes, islands, editing]
metadata:
  description: "Support inline editing in your theme templates with the data-dune-body marker or the typed marker components"
---

# Inline Editing in Themes

Dune's inline editing works in two modes for theme developers:

1. **Body marker** — one attribute. Put `data-dune-body` on the element that wraps the rendered markdown body; the overlay handles title and body editing from there.
2. **Marker components** — the same markers as typed components. Import `EditableText`, `EditableMarkdown`, etc. from `@dune/core/ui/editable` for per-field granularity and typed props.

These are two spellings of the same contract: the components render exactly the marker attributes, nothing more. Editor plugins consume the markers from the rendered HTML — templates never import from an editor plugin.

## Marking the body: `data-dune-body`

When an admin is logged in, the overlay annotates two things in the rendered HTML:

- The first `<h1>` → title field (detected automatically, auto-saves)
- The element carrying `data-dune-body` → body content (floating Save/Cancel)

Editing starts from a floating **✎ Edit** button shown on hover — never from clicking the content itself, so links inside editable regions stay followable while browsing in edit mode.

The body element is **never guessed** from page structure. A wrong guess — say, treating a blog listing's post cards as body content — would write template-generated HTML back into the Markdown source on save, so the theme must be explicit:

```tsx
<article class="post">
  <h1>{fm.title}</h1>
  <div data-dune-body dangerouslySetInnerHTML={{ __html: await page.html() }} />
</article>
```

Rules:

- Put `data-dune-body` on exactly the element that wraps the rendered markdown body (`page.html()` / `{children}`) — nothing more, nothing less.
- **Listing and landing templates must not carry the attribute.** A blog index that renders post cards has no editable markdown body — leave it unmarked and body editing is simply unavailable there.
- Only the first marked element on a page is used.

If you are converting a template from another system, the marker belongs on the converted equivalent of Grav's `{{ page.content }}`, Hugo's `.Content`, or WordPress's `the_content()`.

### Opting the title out

The `<h1>` title detection is automatic. If your layout renders an `<h1>` that is not the page title (e.g. a site logo), add `data-dune-no-edit` to it:

```tsx
<h1 class="site-logo" data-dune-no-edit>My Site</h1>
```

## Marker components

Hand-written attributes and the typed marker components from `@dune/core/ui/editable` are **interchangeable** — the components are server-only sugar that render exactly the marker attributes. They ship no JavaScript, imply no specific editor, and the page renders identically whether or not an editor plugin is installed. Use them when you want typed props and per-field granularity; use raw attributes when one line is enough.

No plugin import, no island setup, and no Preact import-map entries are required — the components come with `@dune/core`, which your theme already depends on.

### EditableText

Marks an inline frontmatter field. With the inline-edit plugin installed, admins get in-place text editing that auto-saves to the named frontmatter key.

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

### EditableMarkdown

Marks the rendered markdown body — identical to writing `data-dune-body` by hand, plus the source path.

```tsx
import { EditableMarkdown } from "@dune/core/ui/editable";

<EditableMarkdown sourcePath={page.sourcePath}>
  <div dangerouslySetInnerHTML={{ __html: await page.html() }} />
</EditableMarkdown>
```

With the inline-edit plugin installed, admins get a TipTap WYSIWYG editor over the page's markdown source. The editor loads lazily (admin sessions only) and serialises back to markdown losslessly — the rendered HTML is never reverse-engineered into markdown.

### EditableField, EditableDate, EditableImage

Typed field markers. They add a `data-dune-field-type` hint so editor plugins can mount a type-appropriate editor:

```tsx
import { EditableDate, EditableImage, EditableField } from "@dune/core/ui/editable";

<time>
  <EditableDate field="date" sourcePath={page.sourcePath}>{formatDate(fm.date)}</EditableDate>
</time>

<EditableField field="rating" type="star_rating" sourcePath={page.sourcePath}>
  {fm.rating}
</EditableField>
```

> The current inline-edit plugin edits all field markers as plain text; type-specific editors (date picker, media picker, custom types) are consumed from the `data-dune-field-type` hint as the plugin grows. The markers are forward-compatible — annotate now, richer editors arrive without template changes.

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

        <EditableMarkdown sourcePath={page.sourcePath}>
          <div dangerouslySetInnerHTML={{ __html: await page.html() }} />
        </EditableMarkdown>
      </article>
    </Layout>
  );
}
```

Anonymous visitors see the plain rendered article — Dune strips the marker attributes from their HTML. Admins (with the inline-edit plugin installed) see ✎ Edit handles on the title, date, and body.

## How it works

The markers are the entire contract between themes and editor plugins:

1. **Render time:** templates (or core's marker components) emit `data-dune-*` attributes.
2. **Serve time:** Dune scrubs all `data-dune-*` attributes from responses that don't belong to a logged-in editing session. Anonymous visitors and crawlers never see the markers — content source paths stay private and the HTML carries no editing fingerprint.
3. **Admin requests only:** the markers stay in the HTML, and the editor plugin injects its admin bar script into the response.
4. **In the browser, admin only:** the script finds the markers and mounts its editors on them. The TipTap stack loads only when an edit actually starts.

Because plugins consume markers rather than being imported by templates, themes stay portable across sites with or without an editor plugin — and editor plugins are swappable without touching any template.

> Because markers are stripped from public responses, never use `data-dune-*` attributes as CSS or JavaScript hooks for site styling or behaviour. To check your markers, view the page while logged in as an admin — an incognito window won't show them.
