---
title: "Hook System"
published: true
visible: true
taxonomy:
  audience: [developer]
  difficulty: [advanced]
  topic: [extending, hooks]
metadata:
  description: "Intercepting Dune's lifecycle with hooks"
---

# Hook System

Hooks let you run code at specific points in Dune's lifecycle — when a page loads, before rendering, after cache events, and more.

## Available hooks

### Startup hooks

| Hook | When it fires | Use case |
|------|--------------|----------|
| `onConfigLoaded` | Config fully merged and validated | Modify config, set up external services |
| `onStorageReady` | Storage adapter initialized | Verify connectivity, warm caches |
| `onContentIndexReady` | Content index built/loaded | Build search index, generate sitemap |

### Request lifecycle hooks

| Hook | When it fires | Use case |
|------|--------------|----------|
| `onRequest` | Incoming request (before routing) | Analytics, auth, rate limiting |
| `onCollectionResolved` | Collection query executed | Modify collection results |
| `onBeforeRender` | Before JSX rendering | Inject data, modify props |

### Content processing hooks

| Hook | When it fires | Use case |
|------|--------------|----------|
| `onMarkdownProcess` | Before markdown → HTML | Custom syntax, shortcodes |
| `onMarkdownProcessed` | After markdown → HTML | HTML post-processing |
| `onMediaDiscovered` | Media files found for page | Image optimization triggers |

### Cache hooks

| Hook | When it fires | Use case |
|------|--------------|----------|
| `onCacheHit` | Serving from cache | Analytics, cache headers |
| `onCacheMiss` | Cache miss, will process | Performance monitoring |
| `onCacheInvalidate` | Cache entry invalidated | CDN purging |

### API hooks

| Hook | When it fires | Use case |
|------|--------------|----------|
| `onApiRequest` | Before API request is handled | Auth, rate limiting, request logging |
| `onApiResponse` | After API response is built | Response transformation, headers |

### Engine lifecycle hooks

| Hook | When it fires | Use case |
|------|--------------|----------|
| `onRebuild` | After a successful `engine.rebuild()` | Clear downstream caches, notify search index |
| `onThemeSwitch` | When the active theme changes | Purge theme-specific caches, notify CDN |

### Search hooks

Fired during bootstrap when the search engine is created. They let a plugin add documents to the index or replace the engine entirely. See [Search](../../reference/search) for the end-to-end picture.

| Hook | When it fires | Use case |
|------|--------------|----------|
| `onSearchRecordsCollect` | Before the search index is built | Inject extra records (e.g. extracted PDF text) into the index |
| `onSearchEngineCreate` | When the search engine is created | Replace the built-in engine with an alternative backend (e.g. Meilisearch). If it fails to build, Dune [falls back and retries](../../reference/search#when-a-search-backend-is-down) instead of stopping |

### Content mutation hooks

Fired by whoever writes a page: the admin panel, the MCP server, the `dune content:*` commands, inline editing, or your own plugin. Core never fires them on a writer's behalf, so a plugin that writes pages must fire the matching event itself (see [writing your own admin](../plugins)). Useful for triggering external systems (CDN purges, search re-indexing, notifications) without using outbound [webhooks](../webhooks).

Paths in the payloads are relative to the content directory. Edits made outside Dune (an editor, `git pull`) fire no per-page event; they show up as `onRebuild`.

| Hook | When it fires | Use case |
|------|--------------|----------|
| `onPageCreate` | New page created and index rebuilt | Notify external search, invalidate CDN |
| `onPageUpdate` | Page saved and index rebuilt | Notify external search, invalidate CDN |
| `onPageDelete` | Page deleted and index rebuilt | Remove from external search, purge CDN |
| `onPageMove` | Pages renamed or reordered | Re-index moved pages, redirect old URLs |
| `onWorkflowChange` | Page workflow status changed | Trigger review notifications, update CMS |

### Public feature hooks

Fired by Dune itself, so they work without an admin plugin.

| Hook | When it fires | Use case |
|------|--------------|----------|
| `onFormSubmit` | A public form submission was accepted and stored | Sync to a CRM, post to Slack |
| `onSystemAction` | An [incoming webhook](../../webhooks) ran `rebuild` or `purge-cache` | Audit log, notify a deploy channel |

## Registering hooks

```typescript
// plugins/my-hooks.ts
import type { DunePlugin } from "@dune/core/plugins";

export default {
  name: "my-hooks",
  version: "1.0.0",
  hooks: {
    onRequest: async ({ data, config }) => {
      // data is the Request itself — not wrapped in { req: Request }
      console.log(`[${new Date().toISOString()}] ${data.method} ${data.url}`);
    },

    onMarkdownProcess: async ({ data, setData }) => {
      // Replace custom shortcodes before markdown processing
      const modified = data.raw.replace(
        /\{\{youtube\s+(\w+)\}\}/g,
        '<iframe src="https://youtube.com/embed/$1"></iframe>',
      );
      setData({ ...data, raw: modified });
    },

  },

  // Post-process the rendered page: transformResponse sees the finished
  // Response (there is no hook that hands you the HTML string).
  async transformResponse({ response }) {
    if (!(response.headers.get("Content-Type") ?? "").includes("text/html")) return response;
    const html = await response.text();
    const minutes = Math.ceil(html.split(/\s+/).length / 200);
    return new Response(
      html.replace("</article>", `<p class="reading-time">${minutes} min read</p></article>`),
      { status: response.status, headers: response.headers },
    );
  },
} satisfies DunePlugin;
```

## Hook context

Every hook handler receives a `HookContext` object:

```typescript
interface HookContext<T> {
  event: HookEvent;           // which hook is firing
  data: T;                    // event-specific data
  config: DuneConfig;         // full merged config
  storage: StorageAdapter;    // storage access
  stopPropagation(): void;    // stop further hooks for this event
  setData(data: T): void;     // replace event data
  jobs?: { run(name: string): Promise<void> }; // only set while the job scheduler is running
  content?: ContentApi;       // only set once bootstrap() has built it — see below
}
```

`stopPropagation()` prevents subsequent hooks from running for this event. Use it when a hook fully handles something (like a custom 404 page or an auth redirect).

`setData()` replaces the data flowing through the hook chain. The next hook receives the modified data.

`content` (added in 0.31.7) is the same content query API (`.pages()`, `.page()`, `.search()`, `.taxonomy()`) that `bootstrap()` returns as `contentApi`. It's `undefined` for the handful of hooks that fire before that API is built — `onConfigLoaded`, `onStorageReady`, `onContentIndexReady`, `onSearchRecordsCollect`, `onSearchEngineCreate` — and for the lightweight, standalone hook registries `content:create` and `migrate:*` (with `--fire-hooks`) build outside a full `bootstrap()`. It's present for every other live hook. Guard with `ctx.content?.` unless you've confirmed your handler only ever fires post-bootstrap.

## Event data shapes

The `data` field in `HookContext` is typed per event. Here is what each hook receives:

### Startup hooks (fired automatically by the engine)

| Hook | `data` type | Description |
|------|-------------|-------------|
| `onConfigLoaded` | `DuneConfig` | The fully merged config object |
| `onStorageReady` | `StorageAdapter` | The initialized storage adapter |
| `onContentIndexReady` | `PageIndex[]` | All indexed pages |

### Request lifecycle hooks

| Hook | `data` shape | Description |
|------|--------------|-------------|
| `onRequest` | `Request` | The incoming request itself — **not** wrapped in `{ req: Request }` |
| `onCollectionResolved` | `{ req: Request, collection: Collection }` | Collection query result. `setData()` a modified `collection` to change what the template receives. |
| `onBeforeRender` | `{ req: Request, page: Page, props: Record<string, unknown> }` | The fully-assembled template props, right before rendering. `setData()` a modified `props` to inject or change what the template receives. |

### Content processing hooks

| Hook | `data` shape | Description |
|------|--------------|-------------|
| `onMarkdownProcess` | `{ raw: string, page: Page }` | Raw Markdown before processing. `page` is the full `Page` object, not a lightweight index — it's already fully loaded by this point. `setData()` a modified `raw` to rewrite the source before compilation. |
| `onMarkdownProcessed` | `{ html: string, page: Page }` | Rendered HTML after processing (after sanitization). `setData()` a modified `html` to post-process the output. |
| `onMediaDiscovered` | `{ media: MediaFile[], page: PageIndex }` | Media files found for a page, fired once per page load (not per request — page loads are cached). |

### Cache hooks

| Hook | `data` shape | Description |
|------|--------------|-------------|
| `onCacheHit` | `{ key: string, value: unknown }` | Cache entry found |
| `onCacheMiss` | `{ key: string }` | Cache entry not found |
| `onCacheInvalidate` | `{ key: string }` | Cache entry removed |

### API hooks

| Hook | `data` shape | Description |
|------|--------------|-------------|
| `onApiRequest` | `{ req: Request }` | Before any `/api/*` request is routed |
| `onApiResponse` | `{ req: Request, response: Response }` | After the API response is built. `setData()` a modified `response` to replace it entirely (headers, body, or status). |

### Engine lifecycle hooks

| Hook | `data` shape | Description |
|------|--------------|-------------|
| `onRebuild` | `{}` | Fired at the end of a successful `engine.rebuild()` |
| `onThemeSwitch` | `{ from: string, to: string }` | Fired when the active theme changes (old and new theme names) |

### Search hooks

The payload objects are mutable — handlers populate them in place (push records, or assign `engine`). The bootstrap reads the result back.

| Hook | `data` shape | Description |
|------|--------------|-------------|
| `onSearchRecordsCollect` | `{ records: InjectedSearchRecord[] }` | Push records to index them alongside content pages. Each record carries its own result `route` and is indexed from memory (no file read). |
| `onSearchEngineCreate` | `{ engine, pages, injectedRecords, storage, contentDir, config, formats, loadText }` | Assign `engine` to replace the built-in engine. `loadText(page)` returns a page's plain-text body so an alternative engine can index the same text. Leaving `engine` unset keeps the built-in engine. |

`InjectedSearchRecord` is `{ route: string, title: string, body: string, fields?: Record<string, string>, template?: string }`.

### Content mutation hooks

| Hook | `data` shape | Description |
|------|--------------|-------------|
| `onPageCreate` | `{ sourcePath: string, title: string }` | Fired after a new page is created |
| `onPageUpdate` | `{ sourcePath: string }` | Fired after a page is saved |
| `onPageDelete` | `{ sourcePath: string }` | Fired after a page is deleted |
| `onPageMove` | `{ moves: { from: string, to: string }[] }` | Fired after pages are renamed or reordered. Each entry is a folder (or file) that was renamed; every page below `from` now lives below `to`. One event per operation |
| `onWorkflowChange` | `{ sourcePath: string, from: WorkflowStatus, to: WorkflowStatus }` | Fired after a page's workflow status changes |

`WorkflowStatus` is `"draft" | "in_review" | "published" | "archived"`.

Two more hooks come from the public form and webhook endpoints, which run in Dune itself:

| Hook | Payload | Notes |
|------|---------|-------|
| `onFormSubmit` | `{ form: string, submission: Submission }` | After the submission is stored. Not fired for honeypot hits or validation failures. `submission` has `id`, `fields`, `meta` and any `files`. |
| `onSystemAction` | `{ action: "rebuild" \| "purge-cache", ip: string \| null, userAgent: string \| null }` | Once per action an incoming webhook runs. `@dune/plugin-admin` uses it to write its audit log. |

These same events also trigger [outbound webhooks](../../webhooks) when `admin.webhooks` is configured — hooks and webhooks fire in parallel.

> **Note:** The startup hooks (`onConfigLoaded`, `onStorageReady`, `onContentIndexReady`) and engine lifecycle hooks (`onRebuild`, `onThemeSwitch`) are fired automatically by Dune. The request and API hooks can also be fired by custom server code using `hooks.fire(event, data)` when integrating Dune into a custom server.

## Events that no longer exist

`onRouteResolved`, `onPageLoaded`, `onAfterRender` and `onResponse` were declared in older versions but nothing ever fired them, so a handler registered for one silently never ran. They were removed in 0.36; registering one is now a type error, and `dune validate` flags them in skills code. What to use instead:

- **Props before rendering:** `onBeforeRender` gives you the assembled template props.
- **The rendered page:** a plugin's `transformResponse` sees the finished `Response` (see the example above). It has to read the body, so only do it for the responses you need to change.
- **Everything else about the page:** do it in your theme's template or layout component.

## Hook execution order

Hooks fire in the order they're registered. If multiple plugins register the same hook, they run sequentially. Each hook sees the data as modified by previous hooks.
