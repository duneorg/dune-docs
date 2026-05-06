---
title: "Headless Mode"
published: true
visible: true
taxonomy:
  audience: [developer]
  difficulty: [intermediate]
  topic: [headless, fresh, api, content]
metadata:
  description: "Run Dune as a headless CMS — own all your Fresh routes while Dune manages content, admin, and search"
---

# Headless Mode

In the default Dune setup, the framework owns the entire request pipeline: it resolves URLs to content files, picks a template, and renders the page. Themes control the look.

**Headless mode flips this.** You write the Fresh routes; Dune manages content, search, the admin panel, and the content API. Your site is a standard Fresh app that happens to use Dune as its data layer.

Use headless mode when:
- You need a design system or component library that doesn't fit the theme model
- You're building a hybrid app (CMS-driven pages alongside custom interactive routes)
- You want full control over how content is fetched, transformed, and rendered

---

## Quick start

```bash
dune new my-site --headless
cd my-site
deno task dev
```

`--headless` scaffolds a Fresh project with Dune wired as a content provider. No theme is created.

---

## Project layout

```
my-site/
├── config/
│   ├── site.yaml       ← site metadata, plugins, taxonomies
│   └── system.yaml     ← content dir, cache, debug
├── content/            ← your Markdown / MDX / TSX pages
├── routes/             ← Fresh routes — you own these
│   ├── _layout.tsx
│   ├── index.tsx
│   └── blog/
│       ├── index.tsx
│       └── [slug].tsx
├── islands/            ← your Fresh islands
├── static/             ← static assets
├── main.ts             ← entry point
└── deno.json
```

Dune manages `content/` and `data/`. Everything else is yours.

---

## Entry point (`main.ts`)

```ts
import { App, staticFiles } from "fresh";
import { Builder } from "jsr:@fresh/core@^2/dev";
import { bootstrap } from "@dune/core";
import { mountDuneAdmin, getDuneAdminIslands } from "@dune/core/admin";

// 1. Bootstrap Dune — content index, search, admin, plugins
const ctx = await bootstrap("./");
const app = new App();

// 2. Static file serving
app.use(staticFiles());

// 3. Admin panel + public API (contact forms, webhooks)
await mountDuneAdmin(app, ctx);

// 4. Your routes — Fresh discovers them from routes/ automatically
app.fsRoutes("./routes");

// 5. Bundle islands — yours + admin islands
const builder = new Builder({
  root: "./",
  islandDir: "./islands",
  islandSpecifiers: getDuneAdminIslands(),
});
const applySnapshot = await builder.build({ mode: "production", snapshot: "memory" });
applySnapshot(app);

// 6. Serve
Deno.serve({ port: 3000, handler: app.handler() });
```

### `mountDuneAdmin(app, ctx)`

Registers on the Fresh app:
- Admin panel routes under `/admin` (or your configured `admin.path`)
- Per-site admin context middleware
- Plugin public routes
- Public API endpoints (`/api/contact`, `/api/forms/:name`, `/api/webhook/incoming`)

### `getDuneAdminIslands()`

Returns absolute paths to all island `.tsx` files bundled with Dune's admin panel. Pass these to `Builder({ islandSpecifiers })` so admin islands are included in the production JS bundle alongside your own islands.

---

## Reading content in routes

Use `getContent()` from `@dune/core/content` to query the content index from any Fresh route handler.

```ts
import { getContent } from "@dune/core/content";
```

`getContent()` returns a `ContentApi` object initialized by `bootstrap()`. It is synchronous — no await needed.

### `pages(options?)`

List pages with optional filtering and ordering:

```ts
// routes/blog/index.tsx
import type { FreshContext, PageProps } from "fresh";
import { getContent } from "@dune/core/content";
import type { PageIndex } from "@dune/core";

export function handler(_req: Request, ctx: FreshContext) {
  const posts = getContent().pages({
    orderBy: "date",
    orderDir: "desc",
    limit: 20,
  });
  return ctx.render(posts);
}

export default function BlogIndex({ data }: PageProps<PageIndex[]>) {
  return (
    <ul>
      {data.map((p) => (
        <li key={p.route}>
          <a href={p.route}>{p.title}</a>
          {p.date && <time> — {p.date}</time>}
        </li>
      ))}
    </ul>
  );
}
```

`pages()` options:

| Option | Type | Description |
|--------|------|-------------|
| `orderBy` | `"date" \| "title" \| "route"` | Sort field |
| `orderDir` | `"asc" \| "desc"` | Sort direction |
| `limit` | `number` | Maximum results |
| `offset` | `number` | Pagination offset |

### `page(route)`

Resolve a single page by route path. Returns a `ResolvedPage` with the full HTML, frontmatter, and summary, or `null` if not found.

```ts
// routes/blog/[slug].tsx
import type { FreshContext, PageProps } from "fresh";
import { getContent, type ResolvedPage } from "@dune/core/content";

export async function handler(req: Request, ctx: FreshContext) {
  const page = await getContent().page(`/blog/${ctx.params.slug}`);
  if (!page) return ctx.next();
  return ctx.render(page);
}

export default function Post({ data }: PageProps<ResolvedPage>) {
  return (
    <article>
      <h1>{data.title}</h1>
      {data.date && <time>{data.date}</time>}
      <div dangerouslySetInnerHTML={{ __html: data.html }} />
    </article>
  );
}
```

`ResolvedPage` shape:

| Field | Type | Description |
|-------|------|-------------|
| `route` | `string` | URL path (e.g. `/blog/hello-world`) |
| `title` | `string` | Page title |
| `date` | `string \| null` | Publication date |
| `html` | `string` | Rendered HTML body |
| `summary` | `string` | First paragraph or `description` frontmatter |
| `frontmatter` | `FM` | Full typed frontmatter (generic — pass your type as `page<MyFM>()`) |

### `search(query, limit?)`

Full-text search across all indexed pages. Returns synchronously.

```ts
const results = getContent().search("deno deploy", 10);
// results: Array<{ route, title, score, excerpt }>
```

### `taxonomy(name)`

Get all values for a taxonomy (e.g. tags, categories) with their page counts.

```ts
const tags = getContent().taxonomy("tag");
// tags: Array<{ name, slug, count }>
```

---

## Typed frontmatter

Pass a frontmatter type to `page<FM>()` and `pages<FM>()` for end-to-end type safety:

```ts
interface PostFM {
  title: string;
  date: string;
  tags?: string[];
  hero?: string;
}

const post = await getContent().page<PostFM>(`/blog/${slug}`);
// post.frontmatter.tags is string[] | undefined
```

---

## Dev mode

The scaffold's `deno task dev` uses `--watch` on `main.ts`. Fresh rebuilds island bundles on change; Dune rebuilds the content index when Markdown files change.

For a more responsive dev loop, `bootstrap()` accepts a `dev: true` option that disables the page cache and enables live-reload SSE — but in headless mode you manage the server yourself, so you can wire this up as needed.

---

## Admin panel

The admin panel is available at `/admin` (or your configured `admin.path`) automatically via `mountDuneAdmin()`. On first run, a default admin account is created and the password written to `.dune/admin/admin-password.txt`. Read the file, then delete it and change the password in the admin UI.

The admin panel is fully functional in headless mode: create and edit pages, manage users, view metrics, configure plugins.

---

## Limitations

- **`getContent()` is a singleton** — it is initialized once by `bootstrap()`. In a single-process multi-site setup, only the last `bootstrap()` call's content index is accessible via `getContent()`. Use the `engine` object from `BootstrapResult` directly if you need per-site isolation.
- **No `/*` catch-all** — unlike full Dune mode, there is no automatic content routing. Every public URL must be handled by one of your Fresh routes or it will 404.
- **Static file serving** — theme static files and `/static/*` are not automatically mounted. Add `app.use(staticFiles())` and serve your own `static/` directory.
