---
title: "Islands (Client-Side Interactivity)"
published: true
visible: true
taxonomy:
  audience: [developer]
  difficulty: [intermediate]
  topic: [themes, islands, interactivity]
metadata:
  description: "Adding interactive Preact island components to your Dune theme"
---

# Islands

Islands are Preact components that hydrate in the browser, adding client-side interactivity to otherwise static server-rendered pages. Everything else in a Dune page is inert HTML — only the components you opt into as islands run in the browser.

## What's an island?

The island architecture keeps JavaScript minimal. Your templates run only on the server; you explicitly opt specific components into client-side execution by placing them in the `islands/` directory.

```
themes/my-theme/
├── templates/      ← server-only (no JS sent to browser)
├── components/     ← server-only shared components
├── islands/        ← hydrated in the browser ✦
│   ├── Counter.tsx
│   └── SearchBox.tsx
└── static/
```

At startup, Dune bundles all files in `islands/` using esbuild and serves the compiled JS at `/_fresh/js/*`. Fresh's boot script — automatically injected into every page — handles hydration.

## Creating an island

Place a standard Preact component file inside `themes/{name}/islands/`:

`themes/my-theme/islands/Counter.tsx`:
```tsx
import { useState } from "preact/hooks";

export default function Counter({ initial = 0 }: { initial?: number }) {
  const [count, setCount] = useState(initial);
  return (
    <div class="counter">
      <button onClick={() => setCount(c => c - 1)}>−</button>
      <span>{count}</span>
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </div>
  );
}
```

Any Preact hooks (`useState`, `useEffect`, `useRef`, etc.) are available — import from `preact/hooks`.

### `deno.json` import map

esbuild requires explicit import map entries for each Preact subpath — a catch-all prefix entry is not sufficient. Add these to your site's `deno.json`:

```json
{
  "imports": {
    "preact": "npm:preact@^10",
    "preact/hooks": "npm:preact@^10/hooks",
    "preact/jsx-runtime": "npm:preact@^10/jsx-runtime",
    "preact/jsx-dev-runtime": "npm:preact@^10/jsx-dev-runtime"
  }
}
```

`preact/jsx-dev-runtime` is required in dev mode; `preact/jsx-runtime` in production. `preact/hooks` is required whenever you use any hook.

## Using an island in a template

Import the island component into a server-side template. Fresh detects the import path and wires up the hydration automatically:

`themes/my-theme/templates/default.tsx`:
```tsx
import type { TemplateProps } from "dune/types";
import Counter from "../islands/Counter.tsx";

export default function DefaultTemplate({ page, site }: TemplateProps) {
  return (
    <html lang="en">
      <head>
        <meta charset="UTF-8" />
        <title>{page.frontmatter.title} | {site.title}</title>
      </head>
      <body>
        <h1>{page.frontmatter.title}</h1>
        <div dangerouslySetInnerHTML={{ __html: await page.html() }} />

        {/* This component hydrates in the browser */}
        <Counter initial={0} />
      </body>
    </html>
  );
}
```

The `Counter` renders on the server first (the initial HTML is present immediately), then re-hydrates on the client to become interactive.

## Props and serialisation

Props passed to island components must be JSON-serialisable — strings, numbers, booleans, plain objects, and arrays. Functions and class instances cannot be passed as props because they cannot be serialised into the page's HTML.

```tsx
{/* ✅ Serialisable props */}
<SearchBox placeholder="Search…" minChars={2} />
<Gallery images={page.frontmatter.custom.gallery} />

{/* ❌ Not serialisable — functions can't cross the server/client boundary */}
<Counter onChange={(n) => console.log(n)} />
```

## Multiple islands per page

You can use as many island components as needed. Each island is independently hydrated:

```tsx
<SearchBox placeholder="Search…" />
<Counter initial={page.frontmatter.custom.count ?? 0} />
<Newsletter apiUrl="/api/subscribe" />
```

Islands do not share state by default. Use browser APIs (`localStorage`, `BroadcastChannel`, a shared URL store) if you need islands to communicate.

## `dune dev` and `dune serve`

Islands are bundled at startup in both modes. In dev mode (`dune dev`), Fresh watches the `islands/` directory and rebuilds the JS bundle automatically when any island file changes — no restart needed. The browser reloads via the `/_fresh_live_reload` SSE endpoint once the new bundle is ready.

Content changes (Markdown files, templates, components) trigger a separate rebuild cycle via Dune's own file watcher, which pushes a reload over `/__dune_reload`. Both live-reload channels operate independently and coexist without interference.

## Directory structure requirements

- Island files must be in `themes/{active-theme}/islands/`. Sub-themes inherit templates and components from parent themes but **not** islands — each theme has its own island bundle.
- File names become the component's bundled module name. Keep names unique across the islands directory.
- The `islands/` directory is optional. If it does not exist, no JS bundle is generated and no boot script is activated.
