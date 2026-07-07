---
title: "Theme Config"
published: true
visible: true
taxonomy:
  audience: [developer, webmaster]
  difficulty: [intermediate]
  topic: [themes, config]
metadata:
  description: "data/theme-config.json, per-theme namespacing, and per-page/section theme_config frontmatter overrides"
---

# Theme Config

Themes can expose admin-editable settings — colours, feature flags, layout toggles — that live outside content and templates. These are declared as a schema on the theme and stored in `data/theme-config.json`.

## Declaring settings

A theme opts in by adding `config_schema` to its `theme.yaml`:

```yaml
# theme.yaml
config_schema:
  primary_color: { type: color, label: "Primary Colour", default: "#c9a96e" }
  show_author:   { type: toggle, label: "Show post author", default: true }
```

The admin UI renders a form from this schema (Theme tab) and persists submitted values to `data/theme-config.json`. Fields not declared in the schema are stripped on save.

## Namespacing by theme

`data/theme-config.json` is keyed by theme name:

```json
{
  "caravan": { "primary_color": "#1a1a2e", "show_author": true },
  "blox": { "accent": "#00ffaa" }
}
```

Only the block matching the currently active theme (`theme.name` in `config/site.yaml`) is loaded into `engine.themeConfig`. Switching themes and back doesn't discard the other theme's saved settings — each theme keeps its own namespace in the same file.

## Using theme config in templates

The resolved config for the current page is passed to templates as `themeConfig`:

```tsx
export default function PostTemplate({ page, themeConfig }: TemplateProps) {
  const primaryColor = themeConfig?.primary_color ?? "#c9a96e";
  // ...
}
```

## Per-page and per-section overrides

Pages and sections can override theme config for themselves and their descendants with a `theme_config` frontmatter field:

```yaml
---
title: "Autumn Campaign"
theme_config:
  primary_color: "#c9622a"
  show_author: false
---
```

Any page under this section — including the section's own index page — inherits `primary_color: "#c9622a"` unless it or a closer ancestor overrides it again.

### Resolution order

The effective `themeConfig` for a page is built by shallow-merging, in order:

1. The site-level config for the active theme, from `data/theme-config.json`
2. Each ancestor section's `theme_config` frontmatter, farthest ancestor first
3. The page's own `theme_config` frontmatter, applied last

Later layers win key-by-key — the merge is shallow, so overriding one key in a nested object replaces that whole object rather than merging into it.

```
data/theme-config.json (caravan)   { primary_color: "#c9a96e", show_author: true }
02.campaigns/ (theme_config)       { primary_color: "#c9622a" }
02.campaigns/01.autumn/ (page)     { show_author: false }
                                    ─────────────────────────────────────────────
effective themeConfig for the page: { primary_color: "#c9622a", show_author: false }
```

A page with no `theme_config` frontmatter of its own, and no ancestor section that sets one, just gets the site-level config unchanged.

### Where this applies

`theme_config` resolution runs for regular content pages — both Markdown (`.md`) and TSX (`.tsx`) — rendered from the content tree. Synthetic pages that don't correspond to a tree node (search results, 404/error pages) always render with the site-level config; there's no page in the tree for them to inherit an override from.

### Cache invalidation

Rendered HTML is cached per route. Saving theme config in the admin UI, or rebuilding content, invalidates that cache so pages pick up new values on the next request instead of serving a stale render.
