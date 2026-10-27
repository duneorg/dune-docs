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
| `t` | `(key: string, fallback?: string) => string` | Theme UI-string translator, built from the theme chain's merged `locales/{lang}.json` (see [Theme Inheritance](../02.inheritance#locale-strings)). `t(key)` returns the key itself if missing — a visible, debuggable signal outside production, an empty string when `DUNE_ENV=production` — and warns once per distinct missing key. `t(key, fallback)` returns `fallback` instead when the key is missing, with no warning; this is the normal path while a locale file is incrementally filled in, not an error. |
| `navAll` | `PageIndex[]?` | Full site navigation for the page's language — every page in the index with `depth`, `order`, `parentPath` (same data as `/api/nav`) |
| `translations` | `PageTranslation[]?` | Languages the current page exists in, with the URL for each — for language switchers and `hreflang` links (see below) |
| `Layout` | `Component?` | Dynamically loaded layout component (see below) |
| `collection` | `Collection?` | Collection results if page defines one |
| `children` | `Element?` | Pre-rendered content (HTML wrapped in a div) |
| `searchQuery` | `string?` | Set when rendering the `/search` page. The raw query string from `?q=`. |
| `searchResults` | `SearchResult[]?` | Set when rendering the `/search` page. Ranked results from the search engine. |
| `content` | `ContentApi?` | The content query API (`.pages()`, `.page()`, `.search()`, `.taxonomy()`) — the same instance `bootstrap.contentApi` exposes. Optional on the type, but populated on every normal render; only the bare-HTML fallback used when no theme resolves at all omits it. Added in 0.31.7. |

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

### Language switchers and hreflang links

On multilingual sites (more than one entry in `config.system.languages.supported`), templates receive a `translations` prop: one entry per language the *current page* actually exists in. Each entry has:

| Field | Description |
|-------|-------------|
| `lang` | Language code, e.g. `"de"` |
| `route` | Canonical route shared across languages, e.g. `"/about/"` |
| `url` | Root-relative URL for that language, e.g. `"/de/about/"` |

URLs respect `config.system.languages.include_default_in_url` — the default language is unprefixed unless that option is set. The list includes the page's own language; mark it active or filter it out. On single-language sites `translations` is an empty array, so switchers naturally render nothing.

A language switcher:

```tsx
{translations && translations.length > 1 && (
  <nav class="lang-switcher">
    {translations.map((t) => (
      <a href={t.url} class={t.lang === page.language ? "active" : ""}>
        {t.lang.toUpperCase()}
      </a>
    ))}
  </nav>
)}
```

Hreflang alternate links in the document `<head>` (prefix with `site.url` for absolute URLs, as search engines expect):

```tsx
{translations?.map((t) => (
  <link rel="alternate" hreflang={t.lang} href={site.url + t.url} />
))}
```

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

## Output modes: fragments, JSON, XML, text

By default a template renders into a full HTML document (your layout, the head, the feed links, and the admin bar for logged-in editors). A template can instead produce the **entire response body itself** by exporting `output`:

```tsx
// themes/mytheme/templates/pricing-table.tsx
export const output = "fragment";

export default function PricingTable() {
  return <table id="pricing">…</table>;
}
```

A page then uses it like any other template (`template: pricing-table` in its frontmatter).

| `output` | Media type | The template returns |
|---|---|---|
| `"fragment"` | `text/html` | JSX, rendered as HTML with no `<html>`/`<head>`/`<body>` wrapper |
| `"json"` | `application/json` | a value (`JSON.stringify`'d) or an already-serialized string |
| `"xml"` | `application/xml` | a string |
| `"text"` | `text/plain` | a string |
| `"text/csv"` (any `type/subtype`) | that media type | a string |

Text-like types are sent with `; charset=utf-8`. A string is always sent unchanged, so Dune never HTML-escapes your JSON or XML. Returning the wrong type (for example an object from an `"xml"` template) is a 500 with a message naming the template, not `[object Object]`. Async templates work in every mode.

A JSON endpoint is just a flat content file with a dotted `slug`:

```md
---
title: Pricing
template: pricing-json
slug: pricing.json
---
```

This is served at `/pricing.json` (a flat file has no trailing slash, and `/pricing.json/` redirects to it). `/robots.txt`, `/sitemap.xml` and the favicon are reserved and served from static files first.

What a bare response skips, and what it keeps:

- **Skipped:** feed-link injection, the dev live-reload script, RTL `dir`, and plugin response transforms (the admin bar). A fragment swapped into a page must not carry its own admin bar or reload script.
- **Kept:** security headers, ETag and caching, compression, base-path rewriting, and editing-marker scrubbing. Markers (`data-dune-*`) are removed from fragments even for a logged-in editor, since a fragment has no editing overlay.

A few things to know:

- **Set in the template, never in frontmatter.** If a page author could choose the media type, an editor could serve attacker-controlled `text/javascript` or `text/html` from your site's origin. Choosing which template a page uses is the existing trust boundary.
- **Output templates run with theme-code trust, so escape what you embed.** `output` accepts any `type/subtype`, including `image/svg+xml`, `text/javascript` and `application/javascript`. Unlike uploaded SVG and HTML media, which are served under a sandboxing `Content-Security-Policy`, template output is **not** sandboxed: it is your code speaking from your site's origin. A `"fragment"` template is escaped by Preact like any component, but a `"text"`, `"xml"` or custom-type template returns a string that Dune sends **unchanged**. If you interpolate page content (a title, a frontmatter value, body text) into SVG, JavaScript or XML, escape it for that format yourself; unescaped content is a cross-site scripting hole. `JSON.stringify` is the safe way to put values into a JSON or JavaScript string.
- **Islands in a fragment render on the server but do not hydrate** when the fragment is inserted into a page after load (Fresh hydrates at page load).
- **A template that returns a string or object is called as a plain function**, not rendered as a component, so a `"json"`/`"xml"`/`"text"` template cannot use hooks (the same constraint as async templates). `"fragment"` templates render as normal components.
- An unrecognised `output` value logs a warning once and the template renders as a normal HTML page.
- Typography's orphan protection inserts `&nbsp;` before the last word of rendered Markdown paragraphs; that also appears inside fragments (it is configurable).

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
