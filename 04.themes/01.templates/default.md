---
title: "Templates"
published: true
visible: true
taxonomy:
  audience: [developer]
  difficulty: [intermediate]
  topic: [themes, templates]
metadata:
  description: "Writing JSX/TSX templates for Dune themes"
---

# Templates

Templates are JSX/TSX components that receive a page object and render it. Each content file maps to a template by filename convention.

## Template props

Every template receives `TemplateProps`:

```tsx
import type { TemplateProps } from "@dune/core";
import StaticLayout from "../components/layout.tsx";

export default function PostTemplate({ page, pageTitle, site, config, nav, Layout, children }: TemplateProps) {
  const LayoutComponent = Layout ?? StaticLayout;
  return (
    <LayoutComponent site={site} config={config} nav={nav} page={page} pageTitle={pageTitle}>
      <article>
        <h1>{page.frontmatter.title}</h1>

        <time datetime={page.frontmatter.date}>
          {new Date(page.frontmatter.date).toLocaleDateString()}
        </time>

        <div>{children}</div>

        {page.frontmatter.taxonomy?.tag?.map((tag) => (
          <a key={tag} href={`/tag/${tag}`}>{tag}</a>
        ))}
      </article>
    </LayoutComponent>
  );
}
```

### What's in `TemplateProps`

| Prop | Type | Description |
|------|------|-------------|
| `page` | `Page` | The full page object (frontmatter, content, media, relations) |
| `pageTitle` | `string` | Pre-formatted title: "Title - Descriptor \| Site Name" |
| `site` | `SiteConfig` | Site configuration (title, URL, metadata) |
| `config` | `DuneConfig` | Full merged configuration |
| `nav` | `PageIndex[]` | Top-level navigation pages |
| `Layout` | `Component?` | Dynamically loaded layout component (see below) |
| `collection` | `Collection?` | Collection results if page defines one |
| `children` | `Element?` | Pre-rendered content (HTML wrapped in a div) |
| `searchQuery` | `string?` | Set when rendering the `/search` page. The raw query string from `?q=`. |
| `searchResults` | `SearchResult[]?` | Set when rendering the `/search` page. Ranked results from the search engine. |

### Using the Layout prop

The router passes a `Layout` prop that is loaded dynamically on each request. This ensures layout changes take effect during development without restarting the server. Templates should prefer the `Layout` prop over a static import:

```tsx
import StaticLayout from "../components/layout.tsx";

export default function MyTemplate({ Layout, ...props }: TemplateProps) {
  // Layout prop is fresh on every request; StaticLayout is the build-time fallback
  const LayoutComponent = Layout ?? StaticLayout;
  return (
    <LayoutComponent {...props}>
      {/* content */}
    </LayoutComponent>
  );
}
```

If a template only uses `import Layout from "../components/layout.tsx"` directly, layout file changes won't appear until the server restarts. Dune logs a warning when it detects this pattern during development.

### What's in `Page`

| Property | Type | Description |
|----------|------|-------------|
| `page.frontmatter` | `PageFrontmatter` | All frontmatter fields |
| `page.route` | `string` | URL path: `/blog/hello-world` |
| `page.format` | `ContentFormat` | `"md"`, `"tsx"`, or `"mdx"` |
| `page.template` | `string` | Template name: `"post"` |
| `page.media` | `MediaFile[]` | Co-located media files |
| `page.html()` | `Promise<string>` | Rendered HTML (Markdown pages) |
| `page.summary()` | `Promise<string>` | Auto-generated excerpt |
| `page.children()` | `Promise<Page[]>` | Child pages |
| `page.parent()` | `Promise<Page\|null>` | Parent page |
| `page.siblings()` | `Promise<Page[]>` | Sibling pages |

Note: `html()`, `children()`, `parent()`, and `siblings()` are lazy — they only load data when called.

### Async templates

These lazy accessors return Promises, so a template that calls them must be declared `async function`. Dune awaits the top-level template component before rendering; everything the template returns renders synchronously. Two constraints follow: nested components (including `Layout`) must be synchronous, and an async template cannot use hooks. If you only need the rendered body, prefer the `children` prop — it's the pre-rendered HTML and keeps the template synchronous.

## Template naming convention

| Content file | Template used |
|-------------|---------------|
| `default.md` | `templates/default.tsx` |
| `post.md` | `templates/post.tsx` |
| `blog.md` | `templates/blog.tsx` |
| `item.md` | `templates/item.tsx` |

Override with the `template` frontmatter field:

```yaml
template: landing   # uses templates/landing.tsx instead
```

### Template names affect routing

Template names do more than select a component — they also determine how Dune routes content in plain (non-numeric) folders. When Dune builds the page index, if a content file's stem matches a template defined in `templates/`, the parent folder is treated as a **page folder** and the folder's path becomes the route:

```
blog/my-post/post.md   →  /blog/my-post   (because templates/post.tsx exists)
news/launch/article.md →  /news/launch    (because templates/article.tsx exists)
```

Files whose stems don't match any template — and aren't reserved stems (`default`, `index`) — are treated as flat content files at their own routes:

```
articles/first.md      →  /articles/first  (no templates/first.tsx)
```

This means adding a new template to your theme can change how existing content files are routed. Keep template names purposeful: verbs and nouns that describe the content type (`post`, `article`, `project`, `event`), not structural words that might collide with content filenames.

### Reserved template names

The following template names are used by Dune's built-in routes:

| Template | Route | When rendered |
|----------|-------|---------------|
| `search` | `/search` | When a visitor submits a search query. Receives `searchQuery` and `searchResults` in props. If not present, Dune falls back to a built-in standalone page. |

See [Search](../reference/search#search-ui) for a full example.

## Blog listing template example

```tsx
import StaticLayout from "../components/layout.tsx";

export default function BlogTemplate({ page, pageTitle, site, config, nav, Layout, collection, children }: TemplateProps) {
  const LayoutComponent = Layout ?? StaticLayout;
  return (
    <LayoutComponent site={site} config={config} nav={nav} page={page} pageTitle={pageTitle}>
      <h1>{page.frontmatter.title}</h1>

      <div>{children}</div>

      {collection && (
        <ul>
          {collection.items.map((post) => (
            <li key={post.route}>
              <a href={post.route}>
                <h2>{post.frontmatter.title}</h2>
                <time>{post.frontmatter.date}</time>
              </a>
            </li>
          ))}
        </ul>
      )}

      {collection?.hasNext && (
        <a href={`${page.route}/page:${collection.page + 1}`}>
          Older posts →
        </a>
      )}
    </LayoutComponent>
  );
}
```
