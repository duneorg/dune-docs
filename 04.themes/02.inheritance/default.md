---
title: "Theme Inheritance"
published: true
visible: true
taxonomy:
  audience: [developer]
  difficulty: [advanced]
  topic: [themes]
metadata:
  description: "Extending themes with inheritance"
---

# Theme Inheritance

Dune supports theme inheritance — a child theme can extend a parent theme, overriding only the templates and assets it needs to change.

## How it works

Declare a parent in your theme manifest:

```yaml
# themes/my-theme/theme.yaml
name: my-theme
version: 1.0.0
parent: default     # inherits from themes/default/
```

You can also set `parent` on the active theme in `config/site.yaml` — it overrides `theme.yaml` for the active theme only:

```yaml
theme:
  name: my-brand
  parent: paper     # local themes/my-brand/ extends package "paper"
```

Parents can be a **local theme name**, an **import-map alias**, or a **pinned package specifier**:

```yaml
themes:
  - name: paper
    src: jsr:@dune/theme-paper@1.0.0

theme:
  name: my-brand
  parent: jsr:@dune/theme-paper@1.0.0
```

## Resolution order

When Dune looks for a template, it checks:

1. **Child theme** `templates/` directory
2. **Parent theme** `templates/` directory
3. Error if not found in either

This means:
- Override `post.tsx` in your child theme, and all posts use your version
- Leave `default.tsx` alone, and the parent's version is used
- You only customize what you need

## Example

Parent theme (`themes/default/`):
```
templates/
├── default.tsx     ← used for default.md pages
├── post.tsx        ← used for post.md pages
├── blog.tsx        ← used for blog listing
└── error.tsx       ← 404 page
```

Child theme (`themes/my-theme/`):
```
templates/
├── post.tsx        ← overrides parent's post template
└── landing.tsx     ← adds a new template type
```

Result:
- `default.md` → parent's `default.tsx` (not overridden)
- `post.md` → child's `post.tsx` (overridden)
- `blog.md` → parent's `blog.tsx` (not overridden)
- Pages with `template: landing` → child's `landing.tsx` (new)

## Static assets

Static assets also follow the inheritance chain. If your child theme provides `static/styles.css`, it overrides the parent's. If it doesn't, the parent's is served.

For package-backed themes, static files live in the package's `static/` directory and are served at `/themes/{name}/static/...` without copying them into your site tree.

## Locale strings

Theme UI strings in `locales/{lang}.json` are merged **per key** across the inheritance chain: parent keys are applied first, child keys last, so a child theme can override or add individual strings without copying the parent's whole locale file. For languages other than `en`, the chain-merged `en` locale is layered underneath as a fallback for missing keys.

For example, to relabel a language in a language switcher, a stub child theme only needs:

```json
// themes/my-theme/locales/en.json
{ "de": "German" }
```

All other keys from the parent's `en.json` remain available via `t(key)`.

## Practical use cases

**Branding customization.** Start from a well-tested base theme, override just the layout and styles.

**Template additions.** Add new content types (portfolio, gallery, documentation) without duplicating the base templates.

**Seasonal variants.** Create holiday themes that override just the header and color scheme.
