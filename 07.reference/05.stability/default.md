---
title: "API Stability"
published: true
visible: true
taxonomy:
  audience: [developer]
  difficulty: [intermediate]
  topic: [api, plugins, stability]
metadata:
  description: "Dune's versioning policy, stability guarantees, and what changes between major, minor, and patch releases"
---

# API Stability

Dune follows [Semantic Versioning](https://semver.org). This page describes what is stable, what can change, and how to write forward-compatible code.

## Version policy

Dune is currently pre-1.0. Per semver convention, minor versions may include breaking changes until v1.0.0.

| Version bump | Meaning |
|---|---|
| **Patch** (0.6.x) | Bug fixes. No API changes. Safe to apply immediately. |
| **Minor** (0.7.0+) | New features. May include breaking changes (pre-1.0 convention). Check the changelog before upgrading. |
| **1.0.0** | First stable release. From here: minor = backwards-compatible, major = breaking. |

## Stable APIs

Everything exported from `@dune/core` is the public API as of v0.6.0.

```ts
import {
  createDuneEngine,
  type DuneEngine,
  type Page,
  type PageFrontmatter,
  type DunePlugin,
  type HookEvent,
  sectionRegistry,
  renderSections,
} from "@dune/core";
```

The following sub-module exports are also available. (Theme helpers such as `paginate`, `formatDate` and `getSearchUrl` are in the separate `@dune/theme-api` package, not in core; `@dune/core/theme-helpers` was removed in 0.36.)

| Import path | Contents |
|---|---|
| `@dune/core/plugins` | `CORE_VERSION`, `CORE_INSTANCE`, `loadPlugins`, `loadPluginAdminConfigs`, `mountPlugins` (plugin types are in `@dune/plugin-api`) |
| `@dune/core/sections` | `SectionRegistry`, `sectionRegistry`, `renderSections`, section types |
| `@dune/core/content` | `ContentApi`, `createContentApi()` — content query API for headless Fresh routes, obtained via `BootstrapResult.contentApi` |
| `@dune/core/admin` | `mountDuneAdmin()`, `getDuneAdminIslands()` — headless admin mounting |

## Internal APIs

Anything **not** exported from `@dune/core` (or its sub-modules) is internal and may change at any time:

- Deep imports like `import { … } from "@dune/core/src/admin/server.ts"` are unsupported.
- The file and folder structure of `src/` is not part of the public API.
- Admin HTML templates and CSS class names are not stable.

## Plugin API

The plugin interface is frozen since v0.6.0. No field will be removed from `DunePlugin` before v1.0.

```ts
import type { DunePlugin } from "@dune/plugin-api";

const plugin: DunePlugin = {
  name: "my-plugin",
  version: "1.0.0",
  description: "Does something useful",
  hooks: {
    onBeforeRender: async ({ data, setData }) => {
      // data is { req, page, props }; setData() replaces what the template receives
      setData({ ...data, props: { ...data.props, extra: true } });
    },
  },
  setup: async (api) => {
    // api.hooks, api.config, api.storage, api.services (0.36)
  },
};

export default plugin;
```

### Plugin services (since 0.36)

`PluginApi.services` and `MountApi.services` (see [Plugin services](/docs/extending/plugins#plugin-services)) are new required fields, so code that builds those objects by hand, such as a test harness, has to supply them. The service interfaces and `isDuneError` in `@dune/core/plugin-api` are the plugin-facing contract. The implementations behind them are not: they change with core patch releases, which is the point.

### Plugin capabilities (since 0.36)

`PluginApi.capabilities` and `MountApi.capabilities` (see [Plugin capabilities](/docs/extending/plugins#plugin-capabilities)) are new required fields, so code that builds those objects by hand has to supply them. The slots core consumes (`admin.prefix`, `editor.session`, `scheduler.executor`, `editor.inline`) and their contracts are the plugin-facing surface. Plugins declare extension points of their own as typed keys (`capabilityKey<T>(name)`), named after the plugin that reads them (`admin.*` is `@dune/plugin-admin`'s, `bar.*` is `@dune/plugin-inline-edit`'s); each plugin versions its own. Core has no contribution points.

0.36 **removed** the members they replace, with no deprecation period: `DunePlugin.adminServices`, `adminPages`, `adminBarActions` and `mountEarly`; `MountApi.adminServices`; `BootstrapResult.adminServices` and `adminContext`; `ResponseTransformContext.plugins` (use `ctx.capabilities`); and the `AdminServices` / `AdminServicesContext` types. A plugin that used them fails type-checking until it moves to the registry; the [migration table](/docs/extending/plugins#moving-a-plugin-from-035) lists each replacement.

### Plugin API version

The plugin contract is the `@dune/plugin-api` package. Check its `PLUGIN_API_VERSION` (its major.minor, for example `"1.2"`) at runtime if your plugin uses something added in a specific minor:

```ts
import { PLUGIN_API_VERSION } from "@dune/plugin-api";

const [major, minor] = PLUGIN_API_VERSION.split(".").map(Number);
if (major !== 1 || minor < 2) {
  console.warn(`[my-plugin] needs plugin API 1.2 or later, got ${PLUGIN_API_VERSION}`);
}
```

### Plugins do not import `@dune/core`

A plugin imports types and helpers from `@dune/plugin-api` and nothing from `@dune/core`, so a core release does not force a new plugin release. Core 0.37 removed the subpaths whose content moved into the package: `@dune/core/hooks`, `/config`, `/inline-edit`, `/content/path-utils`, `/content/page-loader`, `/i18n`, `/auth/types`, `/auth/user-store-db`, `/auth/authz-adapter-local` and `/auth/authz-adapter-db`. A plugin that still imports one stops loading; swap the import for `@dune/plugin-api`.

The old `CORE_INSTANCE` handshake (a plugin re-exporting `resolvedCoreSentinel` so core could detect a second copy of itself) only applies to a plugin that imports core. Core still honours it; a plugin on `@dune/plugin-api` has nothing to detect.

### Hook events

All hook events as of v0.6.0:

| Category | Event | Data type |
|---|---|---|
| Startup | `onConfigLoaded` | `DuneConfig` |
| Startup | `onStorageReady` | `StorageAdapter` |
| Startup | `onContentIndexReady` | `PageIndex[]` |
| Request | `onRequest` | `Request` — replace with a `Response` + call `stopPropagation()` to short-circuit routing |
| Request | `onCollectionResolved` | `Collection` |
| Request | `onBeforeRender` | `TemplateProps` |
| Content | `onMarkdownProcess` | `string` (raw markdown) |
| Content | `onMarkdownProcessed` | `string` (HTML) |
| Content | `onMediaDiscovered` | `MediaFile` |
| Cache | `onCacheHit` | `string` (route) |
| Cache | `onCacheMiss` | `string` (route) |
| Cache | `onCacheInvalidate` | `string` (route) |
| API | `onApiRequest` | `Request` |
| API | `onApiResponse` | `Response` |
| Engine | `onRebuild` | `void` |
| Engine | `onThemeSwitch` | `string` (theme name) |
| Content mutation | `onPageCreate` | `{ sourcePath, title }` |
| Content mutation | `onPageUpdate` | `{ sourcePath }` |
| Content mutation | `onPageDelete` | `{ sourcePath }` |
| Content mutation | `onPageMove` | `{ moves: { from, to }[] }` (`PageMovePayload`) |
| Content mutation | `onWorkflowChange` | `{ sourcePath, from, to }` |

New events will be added in minor versions. Exhaustive `switch`/`if` chains over `HookEvent` values should have a fallthrough case.

## Section registry

The `SectionRegistry` and `sectionRegistry` singleton are stable. Register custom section types from your plugin's `setup()` function:

```ts
import { sectionRegistry } from "@dune/core/sections";
import type { DunePlugin } from "@dune/plugin-api";

const plugin: DunePlugin = {
  name: "my-sections",
  version: "1.0.0",
  hooks: {},
  setup: () => {
    sectionRegistry.register({
      type: "announcement",
      label: "Announcement",
      icon: "📢",
      description: "Full-width announcement strip",
      fields: [
        { id: "message", type: "text", label: "Message", required: true },
        {
          id: "color",
          type: "select",
          label: "Color",
          default: "blue",
          options: [
            { value: "blue", label: "Blue" },
            { value: "red", label: "Red" },
          ],
        },
      ],
    });
  },
};
```

Custom sections appear in the Visual Page Builder palette immediately.

## Type stability

Core types frozen since v0.6.0 — no fields will be removed before v1.0:

| Type | Stable since |
|---|---|
| `DunePlugin` | 0.1.0 |
| `HookEvent` | 0.1.0 |
| `HookContext` | 0.1.0 |
| `PluginApi` | 0.1.0 |
| `Page` | 0.1.0 |
| `PageIndex` | 0.1.0 |
| `PageFrontmatter` | 0.1.0 |
| `TemplateProps` | 0.1.0 |
| `DuneConfig` / `SiteConfig` | 0.1.0 |
| `TemplateConfig` | 0.36.0 |
| `StorageAdapter` | 0.1.0 |
| `SectionDef` | 0.6.0 |
| `SectionInstance` | 0.6.0 |

Since 0.36, `TemplateProps.config` is typed as `TemplateConfig` (`site`, `theme`, `system.languages`) instead of the whole `DuneConfig`: this is the part of the configuration a theme can rely on, and the rest is no longer typed for templates (the runtime value is unchanged). See [Templates](../../themes/templates#what-config-contains).

New **optional** fields may be added to these types in minor versions. Code that spreads or destructures these types should use rest parameters to remain forward-compatible:

```ts
// ✅ forward-compatible
const { title, date, ...rest } = page.frontmatter;

// ⚠️ may fail if a new required property is added
const fm: PageFrontmatter = { title: "…" };
```

## Config schema stability

Fields in `site.yaml` and `system.yaml` that are documented in [Config Schema](config-schema) are stable. New optional keys may be added in minor versions. Keys will not be removed before v1.0, with one exception: 0.36 renamed the camelCase keys to snake_case in a single breaking step (see [Key naming](/docs/configuration#key-naming)). After that the convention is fixed.

## What is NOT stable

- Admin HTML, CSS, and JavaScript — may change in any release.
- `deno.lock` — updated as dependencies are upgraded.
- Internal file structure under `src/` — import only from `@dune/core`.
- The admin URL structure (`/admin/*`) — may change in minor versions.
- CLI output format — do not parse `dune content:list` output programmatically; use the engine API instead.
