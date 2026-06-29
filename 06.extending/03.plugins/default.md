---
title: "Plugins"
published: true
visible: true
taxonomy:
  audience: [developer]
  difficulty: [intermediate]
  topic: [extending, plugins]
metadata:
  description: "Creating, installing, and distributing Dune CMS plugins"
---

# Plugins

A plugin is a TypeScript module that bundles hooks, a config schema, and optional setup logic into a distributable package. Plugins are loaded at startup from your `config/site.yaml` and can be managed from the admin panel at **Plugins** (🔌).

## Installing a plugin

Add a `plugins:` list to `config/site.yaml`. Each entry needs at minimum a `src` pointing to the plugin module:

```yaml
# config/site.yaml
plugins:
  - src: "./plugins/my-plugin/mod.ts"
  - src: "jsr:@dune/seo"
    config:
      sitemap: true
      robots: true
  - src: "npm:dune-analytics"
    config:
      provider: plausible
      domain: example.com
```

Or use the CLI:

```bash
dune plugin:install ./plugins/my-plugin/mod.ts
dune plugin:install jsr:@dune/seo
```

This adds the entry to `site.yaml` automatically. Run `dune dev` to activate it.

### Plugin source formats

| Format | Example | Description |
|--------|---------|-------------|
| Local path | `./plugins/my-plugin/mod.ts` | TypeScript file relative to site root |
| JSR | `jsr:@scope/plugin-name` | Package from the JSR registry |
| npm | `npm:dune-plugin-name` | npm package (resolved via Deno) |
| HTTPS | `https://example.com/plugin.ts` | Arbitrary URL import |

## Plugin module format

A plugin module must export a `DunePlugin` as its **default export**, either as a plain object or as a factory function.

### Object form

Use this when your plugin does not need configuration:

```typescript
// plugins/logger/mod.ts
import type { DunePlugin } from "@dune/core/plugins";

export default {
  name: "logger",
  version: "1.0.0",
  description: "Logs every request to the console",
  author: "Your Name",

  hooks: {
    onRequest: ({ data }) => {
      const url = new URL(data.req.url);
      console.log(`[${new Date().toISOString()}] ${data.req.method} ${url.pathname}`);
    },
  },
} satisfies DunePlugin;
```

### Factory form

Use this when your plugin needs to read its config at initialization time. The factory receives the merged plugin config (site.yaml static config merged with any admin-saved overrides):

```typescript
// plugins/analytics/mod.ts
import type { DunePlugin } from "@dune/core/plugins";

export interface AnalyticsConfig {
  provider?: "plausible" | "umami";
  domain?: string;
  enabled?: boolean;
}

export default function createAnalytics(config: AnalyticsConfig = {}): DunePlugin {
  const { provider = "plausible", domain = "", enabled = true } = config;

  return {
    name: "analytics",
    version: "1.0.0",
    description: "Privacy-focused page view analytics",

    hooks: {
      onAfterRender: ({ data }) => {
        if (!enabled || !domain) return;

        const snippet = provider === "plausible"
          ? `<script defer data-domain="${domain}" src="https://plausible.io/js/script.js"></script>`
          : `<script async src="https://analytics.umami.is/script.js" data-website-id="${domain}"></script>`;

        data.html = data.html.replace("</head>", `${snippet}\n</head>`);
      },
    },
  };
}
```

## Plugin fields

| Field | Required | Description |
|-------|----------|-------------|
| `name` | ✅ | Unique identifier — used as the key in `config.plugins` |
| `version` | ✅ | SemVer version string |
| `description` | — | Human-readable description shown in the admin panel |
| `author` | — | Author name or contact shown in the admin panel |
| `hooks` | ✅ | Map of hook event names to handler functions (can be empty `{}`) |
| `configSchema` | — | Blueprint-style field definitions for admin-driven config UI |
| `setup` | — | One-time initialization function called when the plugin is registered |
| `dependencies` | — | Names of other plugins this plugin requires (soft warning at startup) |
| `publicRoutes` | — | Fresh routes registered on the public site (see below) |
| `adminPages` | — | Admin panel pages contributed by this plugin (see below) |

## Request interception

The `onRequest` hook fires at the very start of every request, before Dune's routing pipeline. If a plugin replaces the request data with a `Response` and calls `stopPropagation()`, that response is returned immediately — Dune's page routing, admin panel, and API handlers are skipped entirely.

This gives plugins a first-class way to add custom API endpoints, authentication guards, or any other per-request middleware without touching the serve command.

The handler receives a `HookContext<Request>` with the following properties:

| Property | Type | Description |
|----------|------|-------------|
| `data` | `Request` | The incoming request |
| `config` | `DuneConfig` | Full merged site config (read-only) |
| `storage` | `StorageAdapter` | Read/write access to site storage |
| `setData(value)` | `fn` | Replace `data` — pass a `Response` to short-circuit routing |
| `stopPropagation()` | `fn` | Prevent subsequent `onRequest` handlers from running |

### Custom API endpoint

```typescript
import type { DunePlugin } from "@dune/core/plugins";

export default {
  name: "my-api",
  version: "1.0.0",
  hooks: {
    onRequest: ({ data: req, setData, stopPropagation }) => {
      const url = new URL(req.url);

      if (url.pathname === "/api/status") {
        setData(Response.json({ ok: true, timestamp: Date.now() }));
        stopPropagation();
      }
    },
  },
} satisfies DunePlugin;
```

### Auth guard

```typescript
onRequest: ({ data: req, setData, stopPropagation }) => {
  const url = new URL(req.url);

  if (url.pathname.startsWith("/members/")) {
    const token = req.headers.get("Authorization")?.replace("Bearer ", "");
    if (!isValidToken(token)) {
      setData(new Response("Unauthorized", { status: 401 }));
      stopPropagation();
    }
  }
},
```

### Reading config or storage

`config` and `storage` are available on the context object — useful when the interception logic depends on plugin settings or persisted data:

```typescript
onRequest: async ({ data: req, config, storage, setData, stopPropagation }) => {
  const url = new URL(req.url);
  const cfg = (config.plugins["my-plugin"] ?? {}) as MyPluginConfig;

  if (url.pathname.startsWith("/protected/") && cfg.enabled) {
    const allowList = JSON.parse(
      await storage.readText("data/plugins/my-plugin/allow.json").catch(() => "[]"),
    ) as string[];
    if (!allowList.includes(req.headers.get("x-api-key") ?? "")) {
      setData(new Response("Forbidden", { status: 403 }));
      stopPropagation();
    }
  }
},
```

If `setData()` is not called (or the new value is not a `Response`), the request continues through Dune's normal routing pipeline unchanged. Multiple plugins can register `onRequest` handlers — they run in registration order and the first one to call `stopPropagation()` wins.

## Public routes

`publicRoutes` is the preferred way to add custom endpoints to the public site. Each entry is a proper Fresh route registered before the content catch-all, so it takes priority over content pages and the `onRequest` hook.

```typescript
import type { DunePlugin } from "@dune/core/plugins";

export default {
  name: "my-api",
  version: "1.0.0",
  hooks: {},

  publicRoutes: [
    {
      path: "/api/status",
      method: "GET",           // optional — defaults to "GET"
      handler: (_fc) => Response.json({ ok: true, ts: Date.now() }),
    },
    {
      path: "/api/subscribe",
      method: "POST",
      handler: async (fc) => {
        const { email } = await fc.req.json();
        // … store email …
        return Response.json({ subscribed: true });
      },
    },
  ],
} satisfies DunePlugin;
```

### `PublicRouteRegistration` shape

| Field | Type | Description |
|-------|------|-------------|
| `path` | `string` | URL path pattern — supports Fresh param syntax (`:id`, `*`) |
| `method` | `"GET" \| "POST" \| "PUT" \| "DELETE" \| "all"` | HTTP method. Defaults to `"GET"`. Use `"all"` to match any method. |
| `handler` | `(fc: FreshContext) => Response \| Promise<Response>` | Request handler |

### `publicRoutes` vs `onRequest`

Use `publicRoutes` when:
- You need a stable, named endpoint at a specific path
- You want proper HTTP method matching without writing `if` checks
- The route should be visible in Fresh's route list

Use `onRequest` when:
- You need to intercept or modify requests that Dune itself handles
- You need to run logic before every request regardless of path

## Admin pages

Plugins can contribute pages to the admin panel via `adminPages`. Each entry adds a route under `/admin/` that is rendered inside the admin shell (sidebar, header, auth) automatically.

```typescript
import type { DunePlugin } from "@dune/core/plugins";

export default {
  name: "my-plugin",
  version: "1.0.0",
  hooks: {},
  adminPages: [
    {
      path: "/admin/my-plugin",
      label: "My Plugin",
      icon: "🔌",
      handler: async (ctx) => {
        return ctx.render(
          <div>
            <h1>My Plugin</h1>
            <p>Custom admin UI here.</p>
          </div>
        );
      },
    },
  ],
} satisfies DunePlugin;
```

The page appears in the admin sidebar under the label and icon you provide. The `handler` receives a Fresh `FreshContext` and must return a response via `ctx.render()`.

## Search plugins

Two hooks let a plugin extend or replace site search. Both fire once during startup when the search engine is created. See the [hook reference](../hooks#search-hooks) for the exact payload shapes and [Search](../../reference/search) for the end-to-end picture.

### Injecting records

`onSearchRecordsCollect` lets you add documents to the index alongside content pages — useful for indexing text that doesn't live in a Markdown file (extracted PDF text, an external API, a database table). Each record carries its own result `route` and is indexed from memory, so there's no file read.

```typescript
import type { DunePlugin } from "@dune/core/plugins";

export default {
  name: "changelog-index",
  version: "1.0.0",
  hooks: {
    onSearchRecordsCollect: async ({ data, storage }) => {
      const raw = await storage.readText("data/changelog.json").catch(() => "[]");
      for (const entry of JSON.parse(raw) as Array<{ slug: string; title: string; notes: string }>) {
        data.records.push({
          route: `/changelog#${entry.slug}`,
          title: entry.title,
          body: entry.notes,
        });
      }
    },
  },
} satisfies DunePlugin;
```

An `InjectedSearchRecord` is `{ route, title, body, fields?, template? }`. Anything in `fields` is indexed as searchable text but not displayed. Records survive index rebuilds.

### Replacing the engine

`onSearchEngineCreate` lets you swap the built-in in-memory engine for an alternative backend. Assign `ctx.data.engine` to any object implementing the `SearchEngine` interface (`build`, `rebuild`, `search`, `suggest`). Leave it unset to keep the built-in engine. The context also provides the page list, any records collected by `onSearchRecordsCollect`, and a `loadText(page)` helper that returns a page's plain-text body — so an alternative engine can index the same text the built-in engine would.

```typescript
import type { DunePlugin } from "@dune/core/plugins";
import { createMyEngine } from "./engine.ts";

export default function mylSearch(config: { url?: string } = {}): DunePlugin {
  return {
    name: "my-search",
    version: "1.0.0",
    hooks: {
      onSearchEngineCreate: ({ data }) => {
        data.engine = createMyEngine(
          { url: config.url ?? Deno.env.get("MY_SEARCH_URL") },
          data.pages,
          { loadText: data.loadText, injectedRecords: data.injectedRecords },
        );
      },
    },
  };
}
mylSearch.pluginName = "my-search";
```

The official [`@dune/plugin-pdf`](https://github.com/duneorg/plugin-pdf) (record injection) and [`@dune/plugin-meilisearch`](https://github.com/duneorg/plugin-meilisearch) (engine replacement) are working references for both patterns.

## Static assets

If your plugin needs to serve CSS, JavaScript, images, or other static files, place them in an `assets/` subdirectory next to your plugin's `mod.ts`:

```
plugins/my-plugin/
  mod.ts
  assets/
    widget.css
    widget.js
    logo.svg
```

Dune detects the `assets/` directory automatically at startup and serves its contents at `/__plugins/{name}/`. No config required.

```html
<!-- Reference plugin assets from your theme templates -->
<link rel="stylesheet" href="/__plugins/my-plugin/widget.css">
<script src="/__plugins/my-plugin/widget.js" defer></script>
```

Assets are served directly — no bundling or transformation.

## Plugin templates

Plugins can contribute Preact/JSX template components by placing them in a `templates/` subdirectory:

```
plugins/my-plugin/
  mod.ts
  templates/
    landing.tsx
    newsletter-confirm.tsx
```

Plugin templates are appended to the end of the theme resolution chain, after the active theme and all its parents. This means themes can override plugin templates — if the active theme has its own `landing.tsx`, it takes precedence over the plugin's version.

## Plugin dependencies

Declare soft dependencies with the `dependencies` field. If a named plugin is not installed, Dune logs a warning at startup but continues loading:

```typescript
export default {
  name: "dune-comments",
  version: "1.0.0",
  dependencies: ["dune-auth"],   // warns if dune-auth is not installed
  hooks: { ... },
} satisfies DunePlugin;
```

For hard requirements (your plugin cannot function without them), check in `setup()` and throw with a clear message:

```typescript
setup: ({ hooks }) => {
  const installed = hooks.plugins().map((p) => p.name);
  if (!installed.includes("dune-auth")) {
    throw new Error("[dune-comments] Requires the dune-auth plugin — install it first.");
  }
},
```

## Config schema

When `configSchema` is defined, the admin panel renders a typed form for the plugin's settings. Users can edit config values without touching `site.yaml`. Changes are saved to `data/plugins/{name}.json` and loaded at the next startup.

```typescript
import type { DunePlugin } from "@dune/core/plugins";

export default function createSeo(config = {}): DunePlugin {
  return {
    name: "dune-seo",
    version: "1.0.0",
    description: "SEO automation: sitemap, robots meta, canonical URLs",

    configSchema: {
      sitemap: {
        type: "toggle",
        label: "Generate sitemap.xml",
        default: true,
      },
      robots: {
        type: "toggle",
        label: "Add robots meta tag",
        default: true,
      },
      default_robots: {
        type: "text",
        label: "Default robots value",
        default: "index, follow",
      },
      changefreq: {
        type: "select",
        label: "Sitemap change frequency",
        options: {
          always: "Always",
          hourly: "Hourly",
          daily: "Daily",
          weekly: "Weekly",
          monthly: "Monthly",
          yearly: "Yearly",
          never: "Never",
        },
        default: "weekly",
      },
    },

    hooks: {
      onContentIndexReady: async ({ data, config: cfg, storage }) => {
        const pluginCfg = cfg.plugins["dune-seo"] ?? {};
        if (pluginCfg.sitemap === false) return;
        // … generate sitemap
      },
    },
  };
}
```

Config schema fields use the same types as [Flex Object fields](../flex-objects#field-types): `text`, `textarea`, `number`, `toggle`, `date`, `select`, `color`.

## The `setup()` function

`setup()` is called once when the plugin is registered — before any hook events fire. It receives a `PluginApi` object:

```typescript
interface PluginApi {
  /** Hook registry — register additional hooks dynamically */
  hooks: HookRegistry;
  /** Merged configuration (read-only) */
  config: DuneConfig;
  /** Storage adapter — read and write plugin-specific data */
  storage: StorageAdapter;
}
```

Use `setup()` for one-time initialization that needs access to the storage layer or config, or when you need to register hooks conditionally:

```typescript
export default function createCache(config: CacheConfig = {}): DunePlugin {
  return {
    name: "edge-cache",
    version: "1.0.0",

    setup: async ({ hooks, config: cfg, storage }) => {
      // Only register cache-clearing hooks if a purge URL is configured
      const purgeUrl = (cfg.plugins["edge-cache"] as CacheConfig)?.purge_url;
      if (!purgeUrl) return;

      hooks.on("onCacheInvalidate", async ({ data }) => {
        await fetch(`${purgeUrl}?key=${data.key}`, { method: "POST" });
      });

      // Verify the purge endpoint is reachable at startup
      try {
        await fetch(purgeUrl, { method: "HEAD" });
        console.log("[edge-cache] Purge endpoint reachable ✓");
      } catch {
        console.warn("[edge-cache] Purge endpoint unreachable — cache clearing disabled");
      }
    },

    hooks: {},
  };
}
```

## Accessing plugin config in hooks

Plugin config is always available in hook handlers via `config.plugins["plugin-name"]`. Config comes from three merged sources (last wins):

1. Default values from your plugin code
2. Static config in `site.yaml` under the `config:` key
3. Admin-saved overrides from `data/plugins/{name}.json`

```typescript
hooks: {
  onAfterRender: ({ data, config }) => {
    // config.plugins["my-plugin"] is the fully merged config object
    const myConfig = (config.plugins["my-plugin"] ?? {}) as MyPluginConfig;
    if (!myConfig.enabled) return;
    // …
  },
},
```

## CLI commands

| Command | Description |
|---------|-------------|
| `dune plugin:list` | List all installed plugins with their hooks and config fields |
| `dune plugin:install <src>` | Add a plugin to `config/site.yaml` |
| `dune plugin:remove <src\|name>` | Remove a plugin from `config/site.yaml` |
| `dune plugin:create [name]` | Scaffold a new plugin project in `plugins/{name}/` |
| `dune plugin:publish [name]` | Publish a local plugin to JSR |
| `dune plugin:search <query>` | Search JSR for Dune-compatible plugins |
| `dune plugin:update [name]` | Update a JSR or npm plugin to the latest version |

### `dune plugin:create`

Scaffolds a fully-typed plugin template in `plugins/{name}/`:

```bash
dune plugin:create dune-social
```

Creates:

```
plugins/dune-social/
  mod.ts        ← plugin with example hooks + typed config interface
  deno.json     ← JSR package metadata
  README.md     ← installation and configuration docs
```

The generated `mod.ts` exports a factory function and is ready to register in `site.yaml`.

### `dune plugin:list`

```bash
$ dune plugin:list

Installed plugins (2):

  dune-seo@1.0.0 by Jane Doe  — SEO automation: sitemap, robots meta, canonical URLs
    hooks: onContentIndexReady, onAfterRender
    config fields: sitemap, robots, default_robots, changefreq

  analytics@0.2.0
    hooks: onAfterRender
```

### `dune plugin:publish`

Publish a local plugin to JSR. Run this from your plugin directory — it validates that `jsr.json` (or `deno.json`) is present, then delegates to `deno publish`:

```bash
cd plugins/my-plugin
dune plugin:publish

# Or by name from the site root:
dune plugin:publish my-plugin
```

Before publishing, make sure your `jsr.json` has a `name` in the `@scope/package` format, a `version`, and an `exports` field pointing to your `mod.ts`.

### `dune plugin:search`

Search JSR for Dune-compatible plugins:

```bash
dune plugin:search analytics

# Results:
#   @dune/analytics-plausible@1.2.0
#     Privacy-focused Plausible analytics integration
#     dune plugin:install jsr:@dune/analytics-plausible
#
#   @janedev/dune-umami@0.8.1
#     Umami analytics for Dune CMS
#     dune plugin:install jsr:@janedev/dune-umami
```

Results are fetched from the JSR search API filtered to the `@dune` scope and packages tagged with `dune-plugin`.

### `dune plugin:update`

Update a JSR or npm plugin to its latest published version and patch `site.yaml`:

```bash
# Update a specific plugin
dune plugin:update dune-seo

# Update all JSR/npm plugins
dune plugin:update
```

The command fetches the latest version from the registry, updates the `src:` value in `config/site.yaml`, and prints what changed. Local path plugins (`./plugins/…`) are skipped.

## Admin panel

Open **Plugins** (🔌) in the admin sidebar to see all installed plugins. For each plugin that defines a `configSchema`, a **Config** form is rendered automatically. Changes are saved immediately and take effect on the next server restart.

Config saved via the admin panel is stored in `data/plugins/{name}.json`. This file is git-tracked so config changes are part of your deployment.

## Scaffolding a local plugin

The fastest way to start a plugin:

```bash
# Create the scaffold
dune plugin:create my-shortcodes

# Register it
# (plugin:create prints the site.yaml snippet — copy it, or:)
dune plugin:install ./plugins/my-shortcodes/mod.ts

# Start developing
dune dev
```

Edit `plugins/my-shortcodes/mod.ts`, add hooks, and the dev server will reload on save.

## Distribution

Plugins are standard Deno/TypeScript modules. Publish them so others can install them with a single `src:` line.

**JSR (recommended)** — free, TypeScript-native, scoped packages:

```bash
# In your plugin directory
deno publish
```

Users install with:

```yaml
plugins:
  - src: "jsr:@yourscope/dune-my-plugin"
```

**npm** — for broader ecosystem reach:

```yaml
plugins:
  - src: "npm:dune-my-plugin"
```

**URL** — for quick sharing without a registry:

```yaml
plugins:
  - src: "https://raw.githubusercontent.com/you/plugin/main/mod.ts"
```

## Best practices

**Give your plugin a unique name.** The `name` field is also the config key in `config.plugins`. Use a scoped name (e.g. `@yourscope/plugin-name`) or a prefixed name (e.g. `dune-seo`) to avoid collisions.

**Provide sensible defaults.** Don't require users to configure every option. Every `configSchema` field should have a `default` that makes the plugin useful out of the box.

**Fail gracefully.** If your plugin can't do its job — missing config, network error, optional dependency not installed — log a warning and return early. Never crash the site.

**Document your hooks.** Tell users which lifecycle events your plugin listens to. The `dune plugin:list` command shows this automatically, but your `README.md` should explain *why* and the performance implications.

**Respect `stopPropagation`.** If a previous hook has already handled something, don't re-process it.

**Keep setup() fast.** `setup()` runs during server startup. Avoid slow I/O or network calls that would delay boot time. For expensive initialization, fire-and-forget with a logged warning if it fails.
