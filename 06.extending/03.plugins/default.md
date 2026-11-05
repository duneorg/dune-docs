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

There's also an opt-in auto-discovery mode: set `auto_discover_plugins: true` at the top level of `site.yaml` and every file directly under `plugins/` loads without needing its own `plugins:` entry. It's off by default on purpose — loading an arbitrary `.ts` file out of that directory executes its module code at startup, so enabling it means anyone who can write to `plugins/` can get code executed. Explicit `plugins:` entries always take priority over an auto-discovered plugin with the same effect.

### Plugin source formats

| Format | Example | Description |
|--------|---------|-------------|
| Local path | `./plugins/my-plugin/mod.ts` | TypeScript file relative to site root |
| JSR | `jsr:@scope/plugin-name` | Package from the JSR registry |
| npm | `npm:dune-plugin-name` | npm package (resolved via Deno) |
| HTTPS | `https://example.com/plugin.ts` | Arbitrary URL import |

JSR and npm specifiers must name a version — exact (`jsr:@dune/plugin-seo@1.0.0`) or a `^`/`~` range (`jsr:@dune/plugin-seo@^1.0.0`) — not a bare name. Plugins run with full process privileges at load time, so an unpinned specifier is rejected before it's ever imported; drift after that point is caught by `--lock --frozen` (the default), which freezes whatever version a range resolves to on first sync. A range is what lets `minimumDependencyAge` degrade gracefully instead of failing outright when the newest matching release is too fresh. Local paths aren't subject to this check. An HTTPS specifier needs its own `integrity: sha256:<64 hex chars>` (or SRI `sha256-<base64>`) alongside `src` in `site.yaml`'s `plugins:` entry — without it, a compromised or swapped origin could serve different code on the next load with no way to detect it.

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
      // data is the Request itself — not wrapped in { req: Request }
      const url = new URL(data.url);
      console.log(`[${new Date().toISOString()}] ${data.method} ${url.pathname}`);
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

    // NOT CURRENTLY POSSIBLE VIA A HOOK: this example needs onAfterRender to
    // rewrite the final rendered HTML string, but onAfterRender is declared
    // and documented above as intentionally unimplemented — Dune never sees
    // the rendered HTML as a string to hand to a hook (Fresh's render()
    // returns a Response directly). Until that changes, script injection
    // like this has to happen inside your own theme layout instead — e.g.
    // read `config.plugins["analytics"]` there and conditionally render the
    // <script> tag. Left here as an illustration of the shape this hook
    // would have if it existed, not a working example.
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
| `capabilities` (via `setup(api)`) | — | What this plugin provides to core and other plugins: admin pages, guarded admin routes, inline editing, bar buttons, ... (see [Plugin capabilities](#plugin-capabilities)) |

## Request interception

The `onRequest` hook fires at the very start of every request, before Dune's routing pipeline. If a plugin replaces the request data with a `Response` and calls `stopPropagation()`, that response is returned immediately — Dune's page routing, admin panel, and API handlers are skipped entirely.

This gives plugins a first-class way to add custom API endpoints, authentication guards, or any other per-request middleware without touching the serve command.

The handler receives a `HookContext<Request>` with the following properties:

| Property | Type | Description |
|----------|------|-------------|
| `data` | `Request` | The incoming request |
| `config` | `DuneConfig` | Full merged site config (read-only) |
| `storage` | `StorageAdapter` | Read/write access to site storage |
| `content` | `ContentApi` | Content query API (`.pages()`, `.page()`, `.search()`, `.taxonomy()`). `onRequest` fires well after bootstrap completes, so this is always populated here — see [Hooks](../01.hooks) for the events where it can be `undefined`. |
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
| `method` | `"GET" \| "POST" \| "PUT" \| "DELETE" \| "ALL"` | HTTP method. Defaults to `"GET"`. Use `"ALL"` to match any method. |
| `handler` | `(fc: FreshContext) => Response \| Promise<Response>` | Request handler |

A route path can't shadow a core-owned prefix — the admin panel, `/api/forms`, `/api/webhook`, `/api/inline-edit`, `/api/pages`, `/api/search`, `/api/nav`, `/api/flex`, `/auth`, `/_fresh`, `/health`. A shadowed route is skipped with a `[dune] plugin route ... rejected: shadows reserved prefix ...` console warning at startup, not a thrown error.

### `publicRoutes` vs `onRequest`

Use `publicRoutes` when:
- You need a stable, named endpoint at a specific path
- You want proper HTTP method matching without writing `if` checks
- The route should be visible in Fresh's route list

Use `onRequest` when:
- You need to intercept or modify requests that Dune itself handles
- You need to run logic before every request regardless of path

## Admin pages and routes

A plugin adds to the admin panel by **contributing** to it from `setup()`; the admin reads the contributions and mounts them behind its own login, CSRF check and permission check. Your handler does not check anything itself, and runs only for a request that passed all of that. (If no admin plugin is installed, nothing is mounted.) Two contribution points cover it:

### `admin.pages` — GET pages with a sidebar entry

Each entry adds a route under `/admin/` that is rendered inside the admin shell (sidebar, header, auth) automatically.

```typescript
import type { DunePlugin } from "@dune/core/plugins";

export default {
  name: "my-plugin",
  version: "1.0.0",
  hooks: {},
  setup(api) {
    api.capabilities.contribute("admin.pages", {
      path: "/my-plugin", // relative to the admin prefix — registers at /admin/my-plugin
      label: "My Plugin",
      icon: "🔌",
      permission: "pages.read", // required — who may view this page
      handler: async (ctx) => {
        return ctx.render(
          <div>
            <h1>My Plugin</h1>
            <p>Custom admin UI here.</p>
          </div>
        );
      },
    });
  },
} satisfies DunePlugin;
```

The page appears in the admin sidebar under the label and icon you provide. The `handler` receives a Fresh `FreshContext` and must return a response via `ctx.render()`.

### `admin.routes` — guarded routes of any method

For a `POST`, `PUT`, `PATCH` or `DELETE` endpoint, or a `GET` API route that is not a sidebar page:

```typescript
setup(api) {
  api.capabilities.contribute("admin.routes", {
    method: "POST",
    path: "/my-plugin/rebuild-index", // relative to the admin prefix → /admin/my-plugin/rebuild-index
    permission: "pages.publish",      // required — who may call this route
    handler: async (_fc) => {
      // The requester is an authenticated admin, passed the CSRF check
      // (every non-GET method) and holds pages.publish.
      await rebuildMyIndex();
      return Response.json({ ok: true });
    },
  });
},
```

`permission` is required on both `admin.pages` and `admin.routes`: the request must be an authenticated admin who holds it (any string works and maps to an action in the authz schema, including ones the plugin declares with `authzActions`). There is no "any authenticated admin" default, so a page or a mutation endpoint cannot be exposed to every admin by leaving a field out. An anonymous request is sent to the admin login like any other admin route. The CSRF check cannot be turned off.

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

`onSearchEngineCreate` lets you swap the built-in in-memory engine for an alternative backend. Pass any object implementing the `SearchEngine` interface (`build`, `rebuild`, `search`, `suggest`) to `ctx.data.register(name, engine)`, then call `ctx.data.setActiveEngine(name)` to make it the one that serves queries. Register nothing to keep the built-in engine. (Assigning `ctx.data.engine` was removed in 0.36.) The context also provides the page list, any records collected by `onSearchRecordsCollect`, and a `loadText(page)` helper that returns a page's plain-text body — so an alternative engine can index the same text the built-in engine would.

```typescript
import type { DunePlugin } from "@dune/core/plugins";
import { createMyEngine } from "./engine.ts";

export default function mylSearch(config: { url?: string } = {}): DunePlugin {
  return {
    name: "my-search",
    version: "1.0.0",
    hooks: {
      onSearchEngineCreate: ({ data }) => {
        const engine = createMyEngine(
          { url: config.url ?? Deno.env.get("MY_SEARCH_URL") },
          data.pages,
          { loadText: data.loadText, injectedRecords: data.injectedRecords },
        );
        data.register("my-search", engine);
        data.setActiveEngine("my-search");
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

Dune detects the `assets/` directory automatically at startup and serves its contents at `/plugins/{name}/`. No config required.

```html
<!-- Reference plugin assets from your theme templates -->
<link rel="stylesheet" href="/plugins/my-plugin/widget.css">
<script src="/plugins/my-plugin/widget.js" defer></script>
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

`setup()` is called once when the plugin is registered. **If it's `async`, the registry does not wait for it to resolve before hooks can fire** — `registerPlugin()` calls `setup()` and only attaches a `.catch()` for error logging, it never blocks registration or dispatch on the returned Promise. Register hooks (via `hooks.on()`, synchronously) before any `await` in `setup()`, and don't rely on async initialization work having finished by the time the first hook event arrives. It receives a `PluginApi` object:

```typescript
interface PluginApi {
  /** Hook registry — register additional hooks dynamically */
  hooks: HookRegistry;
  /** Merged configuration (read-only) */
  config: DuneConfig;
  /** Storage adapter — read and write plugin-specific data */
  storage: StorageAdapter;
  /** Services built by core: security policy, sessions, users, passwords, rate limiting, logger (0.36) */
  services: PluginServices;
  /** Declare what this plugin provides: `provide(slot, factory)` and `contribute(point, item)` (0.36) */
  capabilities: CapabilityRegistrar;
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

      // onCacheInvalidate's payload is always {} — a "cache is stale"
      // signal, not scoped to a key — so there's nothing to read off `data`.
      hooks.on("onCacheInvalidate", async () => {
        await fetch(purgeUrl, { method: "POST" });
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

## The `mount()` function

`mount()` is called after `setup()`, once the Fresh `App` instance is ready. It is the correct place to register Fresh routes and middleware that require an `App` reference. (Admin pages and guarded admin routes are not registered here: contribute them from `setup()`, see [Admin pages and routes](#admin-pages-and-routes).)

```typescript
interface MountApi {
  /** The Fresh App instance — register routes, middleware, layouts here */
  app: App<any>;
  /** Bootstrap result — content engine, config, storage, etc. */
  bootstrap: BootstrapResult;
  /** The same `PluginServices` object `setup()` received (0.36) */
  services: PluginServices;
  /** What plugins provide: `get(slot)`, `list(point)`, `providers()`. Read it lazily, at request time (0.36) */
  capabilities: CapabilityReader;
}
```

Most plugins do not need `mount()` — hooks and `setup()` cover the vast majority of use cases. Use `mount()` only when your plugin genuinely needs to mount Fresh routes:

```typescript
import type { App } from "jsr:@fresh/core";
import type { DunePlugin, MountApi } from "jsr:@dune/core/hooks";

export default function createMyPlugin(): DunePlugin {
  return {
    name: "my-api-plugin",
    version: "1.0.0",

    async mount({ app, bootstrap }: MountApi) {
      // Register a custom API route — unauthenticated, public data only.
      app.get("/api/my-plugin/status", (_req) => {
        return Response.json({ ok: true, pages: bootstrap.engine.pages.length });
      });
    },

    hooks: {},
  };
}
```

That example is intentionally public — it doesn't touch anything a visitor shouldn't see. **If your route mutates data or reads anything private, gate it.** The next section covers that.

### Routes in `mount()` are not behind the admin

A route you register with `app.get()`/`app.post()` in `mount()` runs outside the admin's middleware: plugins mount in registration order and the admin mounts last, so it has no admin login, no CSRF check and no permission check. **Do not put an admin-only operation there; contribute it as an [`admin.routes`](#admin-pages-and-routes) route.**

A public route that behaves differently for a signed-in admin asks the `auth.session` capability, at request time:

```typescript
import type { DunePlugin, MountApi } from "jsr:@dune/core/hooks";

export default function createMyPlugin(): DunePlugin {
  return {
    name: "my-export-plugin",
    version: "1.0.0",

    async mount({ app, capabilities }: MountApi) {
      app.get("/export/report", async (fc) => {
        // `auth.session` is provided by the admin plugin; with none installed
        // it is undefined and every visitor is anonymous.
        const who = await capabilities.get("auth.session")?.authenticate(fc.req);
        if (!who?.authenticated) return new Response("Unauthorized", { status: 401 });
        return Response.json({ report: await buildReport(), requestedBy: who.user?.id });
      });
    },

    hooks: {},
  };
}
```

`authenticate()` resolves to `{ authenticated, user? }`, where `user` always has an `id` and may have `username`, `email`, `name` and `roles`. It tells you *who*; whether they may do this is a permission decision, which the admin's `authz` makes for `admin.routes`.

### Lifecycle

| Phase | When | What's available |
|-------|------|-----------------|
| `setup()` | During `bootstrap()` | `hooks`, `config`, `storage`, `services`, `capabilities` (declare only) |
| Capability resolution | After every plugin's `setup()` has settled, before any `mount()` | factories receive `storage`, `config`, `history`, `hooks`, `services`, `dataDir`, `contentDir` |
| `mount()` | After `createDuneApp()` | `app`, `bootstrap`, `services`, `capabilities` |
| Hook events | Per-request / per-rebuild | Full runtime context |

## Plugin services

Since 0.36 core hands a plugin its security-sensitive building blocks as `services` on both `setup(api)` and `mount(api)`, instead of the plugin importing them from `@dune/core`. Policy that has to stay fixable by a core patch release (HTML sanitising, SSRF guarding, body limits, password hashing, session backends) is reached only through this object, so a plugin always runs against the policy of the core that is actually running it, not whichever copy its own dependency resolved.

```typescript
interface PluginServices {
  security: {
    sanitizeHtml(input: string, opts?: SanitizeOptions): string;
    safeFetch(url: string, init?: RequestInit, opts?: SsrfCheckOptions): Promise<Response>;
    checkBodySize(req: Request, maxBytes: number): Response | null;   // 413 response, or null if within limit
    limitedBody(stream, maxBytes): ReadableStream<Uint8Array>;        // fails with a body_too_large error past the limit
    checkPasswordStrength(password: unknown): PasswordCheckResult;
    clientIp(req: Request, opts?: ClientIpOptions): string;
    parseUserYaml(text: string): unknown;                             // safe-schema YAML for user-written input
  };
  rateLimit: {
    createLimiter(maxRequests: number, windowMs: number, store?: RateLimitStore): RateLimiter;
    createLocalStore(): RateLimitStore;
  };
  sessions: {
    createManager(opts: { sessionsDir: string; lifetimeMs: number }): Promise<SessionManager>;
  };
  users: {
    createLocalStore(opts: { usersDir: string }): UserStore;
  };
  passwords: {
    hash(password: string): Promise<string>;
    verify(password: string, stored: string): Promise<boolean>;
    needsRehash(stored: string): boolean;
    readonly dummyHash: string;      // verify against this for unknown users, so a login costs the same either way
  };
  logger: Logger;
  engines: {
    workflow: WorkflowEngine;        // content status transitions, from site.workflow
    scheduler: Scheduler;            // scheduled publish/unpublish store (CRUD; the serving process runs the polling loop)
    staging: StagingEngine;          // staged drafts of published pages
    mt: MachineTranslator | null;    // null unless site.machine_translation is configured
    audit: AuditLogger | null;       // null when admin is disabled or admin.audit.enabled is false
  };
  authz: {
    seedAdminTuples(users: Array<{ id: string; role: string }>): Promise<void>;   // idempotent; no-op without an authz system
  };
  blueprints: {
    has(template: string): boolean;
    resolve(template: string): ResolvedBlueprint | null;                          // `extends` chain flattened
    validate(frontmatter: PageFrontmatter, template: string): BlueprintValidationError[];   // [] = valid, or no blueprint
  };
  search: {
    analytics: {
      readonly enabled: boolean;                                                  // system.search.analytics
      summarize(limit?: number): Promise<SearchAnalyticsSummary>;                 // empty summary when off
    };
  };
  paths: {
    readonly dataDir: string;        // system.data_dir, default "data"
    readonly runtimeDir: string;     // admin.runtime_dir, default ".dune/admin"
  };
}
```

- **Sessions** use the backend configured in `system.session_store` (local files under `sessionsDir`, Deno KV or Redis). In a multisite install they are scoped to the site by the same key prefix core uses, so a plugin's sessions never share keys with another site's.
- **Users** `createLocalStore` returns the flat-file store under `usersDir` in the site's storage. It is the same store shape core uses; `user_store: db` is not reflected here yet.
- **Engines** are built once per site by core, under `admin.runtime_dir`, so a plugin uses them rather than calling the factories. The scheduler only stores scheduled actions; whatever runs `dune serve` starts its polling loop, so a plugin that wants scheduled actions to fire exposes an `executeScheduledAction` on the admin context for it to call. The `audit` trail is created for the admin panel and is `null` for a site with admin disabled.
- **Authz** `seedAdminTuples` registers the access tuple for each given user's admin-tier role. Call it after creating or loading your users, as the admin panel does at mount.
- **Blueprints** read the content engine's currently loaded blueprints on every call, so a reload is picked up. Use `resolve` for the form definition and `validate` before saving frontmatter.
- **Search analytics** are recorded by core when `system.search.analytics` is on (see [Query analytics](../../reference/search#query-analytics)); a plugin only reads the summary.
- **Paths** are site-relative storage paths. Use them instead of re-deriving `data_dir` and `runtime_dir` from the config.
- **Types** only: the interfaces are what a plugin may rely on. Import them with `import type { PluginServices } from "@dune/core/plugin-api"`.

```typescript
setup: ({ services }) => {
  const limiter = services.rateLimit.createLimiter(30, 60_000);
  // ...
},
mount: ({ app, services }) => {
  app.post("/api/my-plugin/note", async (ctx) => {
    const tooBig = services.security.checkBodySize(ctx.req, 64 * 1024);
    if (tooBig) return tooBig;
    // ...
  });
},
```

### Telling errors apart

`instanceof` fails when a plugin and the site hold two copies of a module, so the errors that cross this boundary carry a stable `code` and are tested with `isDuneError`:

| Code | Raised by |
|------|-----------|
| `body_too_large` | a body read through `limitedBody` past its limit |
| `ssrf_blocked` | `safeFetch` refusing a private, loopback or otherwise unsafe destination |
| `duplicate_email` | `users.createLocalStore().create()` on an existing email |
| `weak_password` | a password rejected by the strength policy |

```typescript
import { isDuneError } from "@dune/core/plugin-api";

try {
  await services.security.safeFetch(url);
} catch (err) {
  if (isDuneError(err, "ssrf_blocked")) return new Response("Blocked destination", { status: 400 });
  throw err;
}
```

A plugin written before 0.36 keeps working: the old imports from `@dune/core` still resolve. They are the ones that can drift from the running core's policy, so move to `services` when you next touch the plugin.

## Plugin capabilities

Since 0.36 a plugin *provides* things to core and to other plugins through a typed registry; core no longer reaches into the admin plugin's internals. [Plugin services](#plugin-services) go the other way: core implements them and your plugin uses them. A **capability** is implemented by a plugin and used by core or another plugin. It is optional by nature: when nobody provides it, the consumer carries on without that feature.

There are two shapes. A **slot** has at most one active provider; a **contribution point** collects one item from each contributing plugin.

| Slot | Contract | Typically provided by | Used by | If nobody provides it |
|------|----------|----------------------|---------|-----------------------|
| `auth.session` | `authenticate(req) → { authenticated, user? }`, optional `sessionCookies` | the admin plugin | core: the admin bar, the inline-edit WebSocket | every visitor is anonymous for admin purposes (no bar) |
| `scheduler.executor` | `execute(action)` | the admin plugin | `dune serve` runs core's scheduler loop with it | the loop is not started; scheduled actions stay queued |
| `inline-edit` | `InlineEditManager` | `@dune/plugin-inline-edit` | core's `/api/inline-edit/ws`, the admin | the WebSocket route answers 501 |
| `editor.content` | `ContentEditorPlugin` | an editor plugin | the admin | the admin uses its built-in block editor |

| Contribution point | Item | Used by |
|--------------------|------|---------|
| `admin.pages` | `AdminPageRegistration` | the admin (sidebar pages) |
| `admin.routes` | `AdminRouteRegistration` | the admin (guarded routes) |
| `bar.actions` | `(ctx) => AdminBarAction[]` | the admin bar, via `ResponseTransformContext.barActions` |

The key set is closed in 1.0: these are the only names. Third-party keys are not supported.

### Providing and contributing

Declare in `setup()`; this only records the declaration. Slots are given as a factory:

```typescript
import type { DunePlugin } from "jsr:@dune/core/hooks";

export default {
  name: "my-editor",
  version: "1.0.0",
  hooks: {},
  setup(api) {
    // A slot: one provider. The factory runs once, after every setup() has settled.
    api.capabilities.provide("editor.content", ({ storage, history, hooks }) =>
      createMyEditor({ storage, history, hooks }));

    // A contribution: appended to the point, in plugin registration order.
    api.capabilities.contribute("bar.actions", ({ page }) =>
      page?.template === "worksheet"
        ? [{ id: "my-editor:open", label: "Open in editor", href: `/editor?path=${encodeURIComponent(page.sourcePath)}` }]
        : []);
  },
} satisfies DunePlugin;
```

The factory receives a `CapabilityContext`:

```typescript
interface CapabilityContext {
  storage: StorageAdapter;   // the site's storage (content and data files)
  config: DuneConfig;        // merged site configuration
  history: HistoryEngine;    // records content revisions
  hooks: HookRegistry;       // the engine's hook registry
  services: PluginServices;  // the services described above
  dataDir: string;           // plugin runtime data directory (admin.runtime_dir)
  contentDir: string;        // content directory relative to the site root
}
```

Resolution rules:

- Factories run **once each, in plugin registration order**, after core infrastructure exists and before any `mount()`; they may be `async`.
- A second provider for the same slot **replaces** the first and logs a warning naming both, so a site plugin can replace a built-in provider.
- A factory that **throws** is logged and leaves the slot empty; the site keeps serving. A factory that returns `undefined` declines the slot.
- `capabilities.providers()` lists who provides what and each slot's status (`unresolved`, `active`, `declined`, `failed`).

### Reading capabilities

`MountApi.capabilities` (and `bootstrap.capabilities`) is the read side. Read it lazily, at request time, and treat a missing provider as normal:

```typescript
mount({ app, capabilities }) {
  app.get("/whoami", async (fc) => {
    const who = await capabilities.get("auth.session")?.authenticate(fc.req);
    return Response.json({ signedIn: who?.authenticated ?? false });
  });
  const extraPages = capabilities.list("admin.pages"); // readonly array
}
```

### Writing your own admin: the `auth.session` contract

The built-in admin is just a provider. Any plugin can be *the* admin by providing `auth.session`, and core treats it exactly the same: it calls `authenticate(req)` on public pages to decide whether to inject the admin bar and whether editing markers survive, and in the inline-edit WebSocket. What it needs from you:

```typescript
interface RequestAuthenticator {
  authenticate(req: Request): Promise<{
    authenticated: boolean;
    // Only `id` is guaranteed; OAuth and magic-link accounts have no username.
    user?: { id: string; username?: string; email?: string; name?: string; roles?: string[] };
  }>;
  /** Cookies that carry your session. Declare them. */
  readonly sessionCookies?: readonly string[];
}
```

- **Declare `sessionCookies`.** Core uses the names for two cheap decisions that must not depend on a particular admin: whether to call `authenticate()` at all on a public page, and whether to keep the response out of the shared page cache (a response rendered for an editor must never be cached for anonymous visitors). If you leave it out, core treats *any* request with a `Cookie` header as possibly signed in: always correct, but every visitor with an analytics cookie costs a lookup and bypasses the cache.
- **Be cheap without a session.** `authenticate()` runs on public pages. Return `{ authenticated: false }` straight away, without touching a session store, when the request carries no session of yours.
- **Authorisation is core's.** Core checks `pages.update` for your user through its authz system, which exists whenever a plugin provides `auth.session`, even with `admin.enabled: false`. Give your accounts an admin-tier role and seed their tuples with `services.authz.seedAdminTuples([{ id, role }])`; a session without the tuple is authenticated but never gets the bar.
- **You are the writer.** If your admin writes content itself, fire `onPageUpdate` (and `onPageCreate` / `onPageDelete` / `onWorkflowChange`) after each write: core does not fire them for you.
- Set `admin.enabled: false` in `system.yaml` if you replace the built-in admin entirely.

[`@dune/testing`](https://jsr.io/@dune/testing) ships `createStubAdmin()`, a minimal admin built only on these services and capabilities, and `createAppHarness()`, which serves the full app in process. Use them to test a plugin against an admin that is not the built-in one, or as a reference for your own.

### Firing hooks from a provider

A provider that writes content outside the standard admin CRUD routes (`/admin/api/pages/...`), which are what normally fire `onPageCreate`/`onPageUpdate`/`onPageDelete` after a successful write, must fire the matching event itself, because nothing else will. Otherwise search indexes, webhooks and cache invalidation miss the write. The factory's `hooks` is the registry to fire on:

```typescript
import type { CapabilityContext } from "jsr:@dune/core/plugin-api";
import type { DunePlugin } from "jsr:@dune/core/hooks";
import type { InlineEditManager } from "jsr:@dune/core/inline-edit";

function createAutosaveManager(ctx: CapabilityContext): InlineEditManager {
  const { storage, hooks } = ctx;

  return {
    handleUpgrade: (req, user) => openAutosaveSocket(req, user), // your transport
    getPresence: () => [],
    async commit(sourcePath, _author) {
      await storage.write(sourcePath, await currentDraft(sourcePath));
      // The admin CRUD route fires this inline on a normal save; this manager
      // writes directly through storage, so it must fire it too.
      await hooks.fire("onPageUpdate", { sourcePath });
    },
    async patchFields(sourcePath, fields, _author) {
      await applyFields(storage, sourcePath, fields);
      await hooks.fire("onPageUpdate", { sourcePath });
    },
  };
}

export default function createAutosavePlugin(): DunePlugin {
  return {
    name: "autosave",
    version: "1.0.0",
    hooks: {},
    setup(api) {
      api.capabilities.provide("inline-edit", createAutosaveManager);
    },
  };
}
```

### Moving a plugin from 0.35

| Before (0.35) | Now (0.36) |
|---------------|------------|
| `adminServices() → { inlineEdit }` | `capabilities.provide("inline-edit", factory)` in `setup()` |
| `adminServices() → { contentEditor }` | `capabilities.provide("editor.content", factory)` |
| `adminPages: [...]` | `capabilities.contribute("admin.pages", page)` |
| `adminBarActions` and `ctx.plugins[].adminBarActions` in `transformResponse` | `capabilities.contribute("bar.actions", fn)` and `ctx.barActions` |
| `app.post("/admin/…", withGuards(…))` in `mount()` | `capabilities.contribute("admin.routes", { method, path, permission, handler })` |
| `fc.state.adminContext.auth.authenticate(req)` in a plugin route | `capabilities.get("auth.session")?.authenticate(req)` |
| `mountEarly()` | removed: nothing needs admin middleware to mount first |
| `MountApi.adminServices`, `BootstrapResult.adminServices`, `BootstrapResult.adminContext` | removed |

## Accessing plugin config in hooks

Plugin config is always available in hook handlers via `config.plugins["plugin-name"]`. Config comes from three merged sources (last wins):

1. Default values from your plugin code
2. Static config in `site.yaml` under the `config:` key
3. Admin-saved overrides from `data/plugins/{name}.json`

```typescript
hooks: {
  onBeforeRender: ({ data, config }) => {
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
    hooks: onContentIndexReady, onBeforeRender
    config fields: sitemap, robots, default_robots, changefreq

  analytics@0.2.0
    hooks: onRequest
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
