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

In Dune, content is files. There is no database. Pages live on the filesystem — as named folders or as plain files in a directory.

```
content/
├── 01.home/
│   └── default.md          ← numeric page folder
├── 02.blog/
│   ├── blog.md             ← listing page for /blog
│   └── 01.hello-world/
│       ├── post.md         ← child page
│       └── cover.jpg       ← co-located media
├── 03.landing/
│   └── page.tsx            ← TSX content page
├── blog/
│   └── my-post/
│       ├── post.md         ← named page folder → /blog/my-post
│       └── cover.jpg       ← co-located media
└── articles/
    ├── default.md          ← /articles listing page
    ├── my-first-post.md    ← flat file → /articles/my-first-post
    └── deep-dive.md        ← flat file → /articles/deep-dive
```

## Content layouts

### Numeric page folders

A folder with a numeric prefix (`01.`, `02.`, …) is always a **page folder**. The file inside selects the theme template — `post.md` renders with `post.tsx`, `default.md` uses `default.tsx`. Co-located media lives in the same folder as the content file.

```
02.blog/01.hello-world/post.md   →  /blog/hello-world
03.landing/page.tsx              →  /landing
```

### Named page folders

A plain folder (no numeric prefix) whose content file stem matches a theme template name is also treated as a **page folder**. The folder's path becomes the route; the filename selects the template.

```
blog/my-post/post.md     →  /blog/my-post   ("post" matches templates/post.tsx)
news/launch/article.md   →  /news/launch    ("article" matches templates/article.tsx)
```

This mirrors the Grav convention: you can give any folder a human-readable name and put the template-named content file inside. Co-located media works the same way as with numeric folders.

### Flat content files

When a file's stem does not match any theme template name — and is not a reserved stem (`default`, `index`) — it becomes its own page, routed by filename:

```
articles/my-first-post.md    →  /articles/my-first-post
articles/deep-dive.md        →  /articles/deep-dive
articles/default.md          →  /articles  (reserved stem — listing page)
```

Flat files work well for articles, changelog entries, or any content that doesn't need co-located media or nested sub-pages. Add a `default.md` alongside them to create a listing page at the directory's own route.

### How Dune decides

| File path | Condition | Route |
|-----------|-----------|-------|
| `01.blog/01.post/post.md` | numeric parent folder | `/blog/post` |
| `blog/my-post/post.md` | stem matches template | `/blog/my-post` |
| `articles/first.md` | stem has no matching template | `/articles/first` |
| `articles/default.md` | reserved stem | `/articles` |

## Core ideas

**Frontmatter = metadata.** The YAML block at the top of each file controls title, date, taxonomies, collections, visibility, caching, and more.

**Filename = template** (page folders). `post.md` renders with the `post.tsx` theme template. `default.md` uses `default.tsx`. The filename both selects the template and — in plain folders — determines whether the folder is treated as a page folder or a flat archive.

**Formats are interchangeable.** Markdown for prose. TSX for interactive pages. They share the same folder conventions, frontmatter fields, collection system, and taxonomy system.

## Content formats at a glance

| Format | Frontmatter | Body | Rendering | Best for |
|--------|-------------|------|-----------|----------|
| `.md` | YAML between `---` | Markdown → HTML | Injected into theme template | Blog posts, docs, articles |
| `.tsx` | `export const frontmatter = {}` or sidecar YAML | JSX component | Self-rendering, optional layout | Landing pages, interactive content |
| `.mdx` | YAML between `---` | Markdown + JSX | Compiled, injected into template | Tutorials with live examples *(v0.2)* |

Read on for details on each format, frontmatter fields, media handling, collections, and taxonomies.
