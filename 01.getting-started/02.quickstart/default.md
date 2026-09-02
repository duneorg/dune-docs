---
title: "Quickstart: Your First Page"
published: true
visible: true
taxonomy:
  audience: [editor, webmaster, developer]
  difficulty: [beginner]
  topic: [content]
metadata:
  description: "Create your first content page in Dune"
---

# Your First Page

## Start the dev server

If you haven't already, start the development server so you can see changes in real time:

```bash
deno task dev
```

The server starts at `http://localhost:3000` with hot-reload — every content or theme change is picked up automatically.

## Create a blog section

Create a folder for your blog and add a listing page:

```
content/
├── 01.home/
│   └── default.md
└── 02.blog/              ← new folder
    └── blog.md           ← listing page
```

The `02.` prefix controls ordering in navigation. The folder name `blog` becomes the URL: `/blog`.

Write `content/02.blog/blog.md`:

```markdown
---
title: "Blog"
visible: true
collection:
  items:
    "@self.children": true
  order:
    by: date
    dir: desc
---

# Blog

Welcome to my blog. Below you'll find my latest posts.
```

The `collection` in the frontmatter tells Dune to gather all child pages, sorted by date (newest first). This is a declarative query — no code needed.

## Add a blog post

```
content/02.blog/
├── blog.md
└── 01.hello-world/       ← new folder
    ├── post.md           ← your post
    └── cover.jpg         ← co-located media
```

Write `content/02.blog/01.hello-world/post.md`:

```markdown
---
title: "Hello World"
date: 2025-06-15
published: true
taxonomy:
  tag: [dune, first-post]
  category: [announcements]
metadata:
  description: "My first post built with Dune CMS"
---

# Hello World

This is my first blog post built with **Dune CMS**.

![Cover image](cover.jpg)

Content is just files. This Markdown file sits next to its images
in the same folder. No media library, no upload forms — just files.
```

Visit `/blog/hello-world` and you'll see your post.

By filename convention, `post.md` looks for a `post.tsx` template. The starter theme doesn't ship one — it only has `default.tsx`, `blog.tsx`, and `error.tsx` — so this falls back to `default.tsx` instead of erroring. That's why the post above renders using the same layout as your homepage. If you want post-specific markup (a byline, a "back to blog" link, whatever), add `themes/starter/templates/post.tsx`; copy `themes/starter/templates/default.tsx` as a starting point, since a `post.md` frontmatter with no explicit `template:` field will pick it up automatically the next time you save.

## Key concepts to notice

1. **Folder = page**. The folder `01.hello-world/` IS the page. The `.md` file inside provides its content.
2. **Numeric prefix = order**. `01.hello-world` appears first in navigation. The number is stripped from the URL.
3. **Filename = template, with a fallback**. `post.md` looks for the `post.tsx` template; `blog.md` looks for `blog.tsx`; `default.md` looks for `default.tsx`. A page whose implied template doesn't exist in the theme falls back to `default.tsx` rather than failing — see the callout above. An explicit `template:` frontmatter field always overrides the filename convention.
4. **Co-located media**. `cover.jpg` lives next to `post.md`. Reference it with a plain relative path: `![alt](cover.jpg)`.
5. **Frontmatter = configuration**. YAML between `---` delimiters controls title, date, taxonomies, collections, and more.
6. **Collections are declarative**. `@self.children` gathers child pages. No code, no GraphQL, no database queries.
