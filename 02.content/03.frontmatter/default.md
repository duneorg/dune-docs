---
title: "Frontmatter Reference"
published: true
visible: true
taxonomy:
  audience: [editor, webmaster, developer]
  difficulty: [beginner]
  topic: [content, reference]
metadata:
  description: "Complete reference for all frontmatter fields including image, date, taxonomy, and collection"
---

# Frontmatter

Frontmatter is the YAML block at the top of every content file. It controls how a page behaves, where it appears, and how it's rendered.

## All fields

### Identity

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `title` | string | `""` | Page title (required for meaningful pages) |
| `nav_title` | string | `title` | Navigation label — used in menus and breadcrumbs instead of `title` when set |
| `heading` | string | `title` | On-page `<h1>` — use when the SEO title differs from the desired on-page heading |
| `descriptor` | string | — | Short subtitle shown after title in browser tab |
| `slug` | string | folder name | Override the URL slug |
| `order` | number | from folder prefix | Explicit sort position among siblings. Overrides the numeric folder/filename prefix when set. Unprefixed pages with no `order` sort after all explicitly-ordered ones. |
| `template` | string | filename | Override template selection (`.md` only) |
| `layout` | string \| false | `"default"` | Layout wrapping (`.tsx` only) |

The `descriptor` field controls the page title shown in the browser tab. When present, the title format is **"Title - Descriptor | Site Name"**. Without it, the format is **"Title | Site Name"**.

```yaml
title: "Services"
descriptor: "Custom Web Solutions"
# → browser tab: "Services - Custom Web Solutions | My Site"
```

### Publishing

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `published` | boolean | `true` | If `false`, page doesn't exist for visitors |
| `status` | string | — | Workflow status: `draft`, `in_review`, `published`, `archived` |
| `date` | string | — | Publication date (ISO 8601) |
| `publish_date` | string | — | Scheduled publish (ISO 8601 datetime) |
| `unpublish_date` | string | — | Scheduled unpublish (ISO 8601 datetime) |
| `visible` | boolean | `true` | If `false`, accessible but hidden from navigation |
| `routable` | boolean | `true` | If `false`, page has no URL (modular sections) |
| `trusted_html` | boolean | `false` | If `true`, raw HTML embedded in this page's markdown is rendered as-is, bypassing the HTML sanitiser. Only use for pages whose content you fully trust. Site-wide opt-out via `trusted_html: true` in `site.yaml`. |

The `status` field is used by the content workflow system for editorial management. It works independently of `published` — a page with `status: draft` and `published: true` is still accessible. Use `published: false` to actually hide a page from visitors.

### Taxonomy

```yaml
taxonomy:
  category: [tutorials, guides]
  tag: [deno, fresh, cms]
  author: [jane]
```

Taxonomies are defined in `config/site.yaml` under `taxonomies`. Any taxonomy listed there can be used in frontmatter. Values are always arrays.

### Metadata

```yaml
metadata:
  description: "Page description for SEO"
  og:image: cover.jpg
  robots: "noindex"
```

Key-value pairs for HTML `<meta>` tags and SEO. Values are strings.

### Routes

```yaml
routes:
  aliases:
    - /old-path
    - /another-old-path
  canonical: /the-real-url
```

`aliases` create additional URLs that serve this page. `canonical` sets the preferred URL for search engines.

### Summary

```yaml
summary:
  size: 300          # characters
  format: short      # "short" or "long"
```

Controls automatic excerpt generation from the page body.

### Cache

```yaml
cache:
  enable: true
  lifetime: 3600     # seconds
```

Per-page cache overrides. Takes precedence over system-level cache settings.

### Collection

```yaml
collection:
  items:
    "@self.children": true
  order:
    by: date
    dir: desc
  filter:
    published: true
  limit: 10
  pagination:
    size: 10
```

Declarative content queries. See the [Collections](../04.collections) page for full details.

### Named collections

```yaml
collections:
  featured:
    items:
      "@self.children": true
    filter:
      custom.featured: true
    limit: 3
  recent:
    items:
      "@self.children": true
    order:
      by: date
      dir: desc
    limit: 5
```

`collections` (plural) declares several independent named collections on one page — useful for landing pages that show more than one list at once (e.g. featured items alongside recent posts). Each entry under the map is a full collection definition, using the same fields as the singular `collection:` field above. Templates receive the resolved results as `props.collections`, keyed by the same names:

```tsx
{props.collections?.featured?.items.map((item) => ...)}
{props.collections?.recent?.items.map((item) => ...)}
```

Unlike `collection:` (singular), `collections:` is per-page only — it isn't inherited by descendant pages.

### Theme config override

```yaml
theme_config:
  primary_color: "#c9622a"
  show_author: false
```

Overrides theme config (from `data/theme-config.json`) for this page and, when set on a section, for all of its descendants. Shallow-merged over the site-level config and any ancestor section's own `theme_config`, with the closest override winning key-by-key. See [Theme Config](../../04.themes/07.theme-config) for the full resolution model.

### Media

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `image` | string | — | Cover image filename (co-located with the content file) |

The `image` field names a file in the same directory as the content file. It is used in three places:

- **RSS & Atom feeds** — included as the feed item's cover image (`<enclosure>` / `<media:content>`)
- **XML Sitemap** — generates a `<image:image>` entry for the page
- **Templates** — accessible as `page.frontmatter.image`; use with the image processing API for responsive images

```yaml
---
title: "My Post"
date: 2026-03-09
image: cover.jpg
---
```

The image must be a bare filename, not a path or URL. The served URL mirrors the content path with numeric prefixes stripped — so `02.blog/01.post/cover.jpg` is served at `/blog/post/cover.jpg`.

### Custom data

```yaml
custom:
  featured: true
  rating: 5
  hero_color: "#1a1a2e"
```

Arbitrary structured data accessible in templates. Use this for any page-specific data that doesn't fit the standard fields.

## Minimal frontmatter

The only truly required field is `title`. Everything else has sensible defaults:

```yaml
---
title: "My Page"
---
```

This creates a published, visible, routable page with no taxonomies, default caching, and template selection by filename.
