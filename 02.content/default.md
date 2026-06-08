---
title: "Content"
published: true
visible: true
taxonomy:
  audience: [editor, webmaster, developer]
  difficulty: [beginner]
  topic: [content]
metadata:
  description: "Understanding Dune's content model: folders, formats, and frontmatter"
collection:
  items:
    "@self.children": true
  order:
    by: order
    dir: asc
---

# Content Model

In Dune, content is files. There is no database. Pages live on the filesystem — either as named folders or as plain files in a directory.

```
content/
├── 01.home/
│   └── default.md          ← a folder-based page
├── 02.blog/
│   ├── blog.md             ← a listing page
│   └── 01.hello-world/
│       ├── post.md         ← a child page
│       └── cover.jpg       ← co-located media
├── 03.landing/
│   └── page.tsx            ← a TSX content page
└── articles/
    ├── default.md          ← the /articles listing page
    ├── my-first-post.md    ← flat file → /articles/my-first-post
    └── deep-dive.md        ← flat file → /articles/deep-dive
```

## Two content layouts

### Folder-based pages

A folder in `content/` with a numeric prefix (`01.`, `02.`, …) is a **page folder**. The content file inside selects the theme template — `post.md` renders with `post.tsx`, `default.md` uses `default.tsx`. Co-located media lives in the same folder.

### Flat content files

A plain folder (no numeric prefix) is a **content directory**. `.md` or `.tsx` files placed directly inside it each become their own page, routed by filename stem:

```
articles/my-first-post.md    →  /articles/my-first-post
articles/deep-dive.md        →  /articles/deep-dive
articles/default.md          →  /articles  (reserved stem — the listing page)
```

Reserved stems (`default`, `index`) still belong to the containing folder's page. Everything else becomes an independent page.

Flat files work well for articles, changelog entries, or any content that doesn't need co-located media or nested sub-pages.

## Core ideas

**Frontmatter = metadata.** The YAML block at the top of each file controls title, date, taxonomies, collections, visibility, caching, and more.

**Filename = template** (folder-based pages). `post.md` renders with the `post.tsx` theme template. `default.md` uses `default.tsx`. This is convention over configuration.

**Formats are interchangeable.** Markdown for prose. TSX for interactive pages. They share the same folder conventions, frontmatter fields, collection system, and taxonomy system.

## Content formats at a glance

| Format | Frontmatter | Body | Rendering | Best for |
|--------|-------------|------|-----------|----------|
| `.md` | YAML between `---` | Markdown → HTML | Injected into theme template | Blog posts, docs, articles |
| `.tsx` | `export const frontmatter = {}` or sidecar YAML | JSX component | Self-rendering, optional layout | Landing pages, interactive content |
| `.mdx` | YAML between `---` | Markdown + JSX | Compiled, injected into template | Tutorials with live examples *(v0.2)* |

Read on for details on each format, frontmatter fields, media handling, collections, and taxonomies.
