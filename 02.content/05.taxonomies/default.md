---
title: "Taxonomies"
published: true
visible: true
taxonomy:
  audience: [editor, webmaster]
  difficulty: [beginner]
  topic: [content, taxonomies]
metadata:
  description: "Organizing content with taxonomies (tags, categories, and custom types)"
---

# Taxonomies

Taxonomies let you classify and cross-reference content. Tags, categories, authors — any grouping you need.

## Defining taxonomies

Taxonomies are declared in `config/site.yaml`:

```yaml
taxonomies:
  - category
  - tag
  - author
```

Once declared, you can use them in any page's frontmatter:

```yaml
---
title: "My Post"
taxonomy:
  category: [tutorials]
  tag: [deno, fresh, cms]
  author: [jane]
---
```

Values are always arrays, even for single values. This keeps the data model consistent.

## How taxonomies work

When Dune builds its content index, it creates a **reverse taxonomy map**:

```
tag:
  deno     → ["/blog/post-1", "/blog/post-3", "/tutorials/intro"]
  fresh    → ["/blog/post-1", "/blog/post-2"]
  cms      → ["/blog/post-1"]

category:
  tutorials → ["/blog/post-1", "/tutorials/intro"]
  news      → ["/blog/post-2"]

author:
  jane     → ["/blog/post-1", "/blog/post-3"]
```

This map is built once (during indexing) and kept in memory. Looking up "all pages tagged deno" is an instant map lookup — not a filesystem scan.

## Using taxonomies in collections

The most common use: pull pages by taxonomy in a collection definition.

```yaml
---
title: "Deno Tutorials"
collection:
  items:
    "@taxonomy.tag": "deno"
  order:
    by: date
    dir: desc
---
```

Multiple values use OR logic:

```yaml
collection:
  items:
    "@taxonomy.tag": ["deno", "fresh"]
```

Multiple taxonomy types use AND logic:

```yaml
collection:
  items:
    "@taxonomy":
      tag: "deno"
      category: "tutorials"
```

This finds pages that are tagged "deno" AND in the "tutorials" category.

## Taxonomy listing pages

Create a page that lists all values for a taxonomy. This is useful for "Browse by tag" or "All categories" pages:

```yaml
---
title: "Tags"
template: taxonomy-listing
custom:
  taxonomy_type: tag
---
```

The `taxonomy-listing.tsx` template in your theme receives the full taxonomy map and renders links to each value.

## Term pages

Any published page can declare itself the editorial home for a taxonomy term — a dedicated page with its own content (description, image, featured posts) that is associated with that term in the index.

```yaml
---
title: "Deno"
template: tag-page
termPageFor: deno          # shorthand → implies the "tag" vocabulary
---

Deno is a modern JavaScript runtime...
```

For non-tag vocabularies, use an explicit map:

```yaml
termPageFor:
  category: tutorials      # this page is the home for the "tutorials" category
```

Only one page per term is recognised — if multiple pages declare the same term, the first one encountered during indexing wins.

### Looking up term pages

**In a template**, use `page.termPage(vocab, value)` to get the editorial page for a term:

```tsx
// In a taxonomy-listing template
const denoPage = await page.termPage("tag", "deno");
// → ResolvedPage | null
```

**Via the content API**:

```ts
import { content } from "@dune/core/content";
const termPage = await content.termPage("category", "tutorials");
```

**In a collection**, `TaxonomyTerm.pageRoute` is populated automatically — useful when rendering a list of terms with links to their editorial pages:

```tsx
{tags.map((term) => (
  <li>
    <a href={term.pageRoute ?? `/tag/${term.value}`}>
      {term.value}
    </a>
    {term.pageRoute && <span> — has editorial page</span>}
  </li>
))}
```

## Custom taxonomies

You're not limited to tags and categories. Any classification you need:

```yaml
# config/site.yaml
taxonomies:
  - category
  - tag
  - author
  - difficulty      # beginner, intermediate, advanced
  - audience        # editor, webmaster, developer
  - language        # en, fr, de
  - product         # for product documentation
```

Then in your content:

```yaml
taxonomy:
  difficulty: [intermediate]
  audience: [developer]
  product: [dune-cms]
```

This documentation site itself uses `audience` and `difficulty` taxonomies to let you filter content by who it's for and how advanced it is.
