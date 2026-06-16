---
title: "CLI Commands"
published: true
visible: true
taxonomy:
  audience: [webmaster, developer]
  difficulty: [beginner]
  topic: [reference, cli]
metadata:
  description: "Complete reference for Dune CLI commands"
---

# CLI Commands

All commands are run with `dune` (or `deno task dune`).

## Development

| Command | Description |
|---------|-------------|
| `dune dev` | Start dev server with hot-reload. Watches content and themes for changes. |
| `dune serve` | Start production server. Uses pre-built content index. |
| `dune serve --port 3000` | Serve on a specific port. |
| `dune serve --root my-site` | Serve a site in a subdirectory without `cd`-ing into it first. |

## Build & Cache

| Command | Description |
|---------|-------------|
| `dune build` | Build content index, validate config. Run before production serving. |
| `dune build --static` | Generate a fully static site into `dist/` (SSG). |
| `dune cache:clear` | Delete all cached data (rendered HTML, content index, images). |
| `dune cache:rebuild` | Rebuild content index from scratch. Use after bulk content changes. |
| `dune validate` | Whole-project lint: config, plugins, templates, schemas, and content. |

`dune validate` checks:

- Config structure and field types (`site.yaml`, `system.yaml`)
- Plugin specs are pinned to a version
- All `template:` values in frontmatter resolve to an existing theme template
- Schema files under `schemas/` have a `store:` field
- Content integrity: missing titles, duplicate routes, future dates

Add `--json` for machine-parseable output.

### Static build options (`dune build --static`)

| Option | Default | Description |
|--------|---------|-------------|
| `--out <dir>` | `dist` | Output directory |
| `--base-url <url>` | `config.site.url` | Canonical base URL for sitemap and feeds |
| `--no-incremental` | — | Rebuild all pages regardless of content changes |
| `--concurrency <n>` | `8` | Number of pages to render in parallel |
| `--hybrid` | — | Emit `_routes.json`, `_redirects`, `_headers` for edge deployments |
| `--include-drafts` | — | Include unpublished pages |
| `--verbose` | — | Print each rendered route |

See [Static Site Generation](../deployment/static) for full documentation.

## Lockfile

A site's `deno.lock` only gains entries for a plugin's dependencies (and, via the client-bundling step, its browser-side npm packages) the first time `serve` actually starts after that plugin or a version bump is installed — until then, the running process resolves them itself against an unfrozen lockfile, which is what silently dirties `deno.lock` on a server's working tree after a deploy.

| Command | Description |
|---------|-------------|
| `dune lockfile:check` | Read-only: exits non-zero if `deno.lock` is missing entries the current plugins/imports need, or if a sync would be unable to add them safely. Never writes. Suitable as a pre-restart gate (e.g. an `ExecStartPre=` step in a systemd unit) so a deploy never gets partway through restarting before discovering the lockfile is stale. |
| `dune lockfile:sync` | Resolves the current plugin/import graph ahead of time and writes the result, but **only ever adds genuinely missing entries** — an already-pinned entry that would resolve to a different value (e.g. the registry now serves a newer match for an already-locked semver range) is left exactly as committed. |

Both commands support `--json` for machine-readable output.

### Using `lockfile:check` as a production gate

When you run Dune via the pinned remote specifier (`deno run ... jsr:@dune/core@X/cli ...`, the default `serve` task), Deno resolves that entry script's own module graph against the site's `deno.lock` *before* the command's code runs — and absent a flag, it will write any updates to that lockfile as a side effect. For a read-only gate you don't want that. Pass **`--no-lock`** on the outer `deno run` to disable Deno's automatic lockfile discovery for the launcher: the gate can't touch `deno.lock`, and unlike `--frozen` it doesn't fail fast, so `lockfile:check`'s own diagnostics (the missing-entry list, the `--upgrade` hints) still print:

```ini
# systemd unit
ExecStartPre=/usr/bin/deno run -A --no-lock --config=deno.json jsr:@dune/core@0.21.6/cli lockfile:check --root .
ExecStart=/usr/bin/deno run -A --frozen --config=deno.json jsr:@dune/core@0.21.6/cli serve --root .
```

Use `--frozen` (not `--no-lock`) on the actual `serve` task: there you *want* to enforce a complete lockfile and fail loudly if anything's missing, rather than silently resolving it. The gate ensures that never happens in normal operation; `--frozen serve` is the safety net.

The most robust model is to treat `deno.lock` as a build artifact: run `lockfile:sync` in a controlled place (locally or in CI), commit the result alongside the version bump, and deploy it read-only. The gate above then only ever passes — it exists to catch the case where that discipline slips.

### `dune lockfile:sync` options

| Option | Description |
|--------|-------------|
| `--upgrade <specifier>` | Allow an already-pinned entry to change. Repeatable, or comma-separated. Get the exact key to pass from the "left unchanged" list printed by `check`/`sync`. |

Occasionally an addition introduces a second, different version range for an already-shared dependency, and another existing entry referencing it ambiguously needs disambiguating too — `sync` can't safely apply that on its own (it would be indistinguishable from unwanted drift), so it refuses to write and reports exactly which entries are blocked and why. Rerunning with `--upgrade` for one of the reported keys applies it.

## Configuration

| Command | Description |
|---------|-------------|
| `dune config:show` | Display the final merged config with source annotations showing where each value comes from. |
| `dune config:validate` | Validate all config files against schemas. Reports errors with suggestions. |
| `dune schema:export` | Print the JSON Schema for `site.yaml` to stdout. Useful for editor autocompletion or agent tooling. |

## Content

| Command | Description |
|---------|-------------|
| `dune content:list` | List all pages with their routes, templates, and publish status. |
| `dune content:check` | Validate all content: broken links, missing templates, orphaned media. |
| `dune content:i18n-status` | Report translation coverage across all configured languages. |
| `dune content:create <route>` | Scaffold a new content page at the given route. |
| `dune content:delete <route>` | Delete a content page by route. Requires `--confirm` or `--dry-run`. |

## Blueprints

Blueprints are per-template frontmatter schemas defined in `blueprints/`. These commands let you inspect available schemas from the CLI.

| Command | Description |
|---------|-------------|
| `dune blueprint:list` | List all available blueprints (one per template). |
| `dune blueprint:show <template>` | Show the full field schema for a template's blueprint. |
| `dune blueprint:validate <file>` | Validate a content file's frontmatter against its blueprint. |

All blueprint commands accept `--json` for machine-readable output.

### `dune content:create` options

| Option | Default | Description |
|--------|---------|-------------|
| `--title <text>` | Derived from slug | Page title written into frontmatter. |
| `--template <name>` | `default` | Template to use. |
| `--flat` | — | Create a flat file (`slug.md`) instead of `slug/default.md`. |
| `--publish` | — | Mark the page as `published: true` (default is draft). |
| `--json` | — | Output result as JSON. |

### `dune content:delete` options

| Option | Description |
|--------|-------------|
| `--confirm` | Confirm deletion without an interactive prompt. |
| `--dry-run` | Preview what would be deleted without writing any changes. |
| `--json` | Output result as JSON. |

## Packages

| Command | Description |
|---------|-------------|
| `dune add <package>` | Add an npm or JSR package to the site's `deno.json` import map. Accepts bare names (`polizy`), versioned names (`polizy@^2`), explicit specifiers (`npm:polizy@^2.0.0`), and JSR packages (`jsr:@scope/pkg`). |

## Plugins

| Command | Description |
|---------|-------------|
| `dune plugin:list` | List all installed plugins with their registered hooks and config fields. |
| `dune plugin:install <src>` | Add a plugin to `config/site.yaml`. |
| `dune plugin:remove <src\|name>` | Remove a plugin from `config/site.yaml`. |
| `dune plugin:create [name]` | Scaffold a new plugin project at `plugins/{name}/`. |
| `dune plugin:publish [name]` | Publish a local plugin to JSR (runs `deno publish`). |
| `dune plugin:search <query>` | Search JSR for Dune-compatible plugins. |
| `dune plugin:update [name]` | Update a JSR or npm plugin to its latest version. Omit name to update all. |

## Migration

| Command | Description |
|---------|-------------|
| `dune migrate:from-grav <src>` | Import a Grav site from its `user/pages/` folder. |
| `dune migrate:from-wordpress <src>` | Import a WordPress WXR export (`.xml` file). |
| `dune migrate:from-markdown <src>` | Import a flat folder of markdown files. |
| `dune migrate:from-hugo <src>` | Import a Hugo site from its `content/` folder. |
| `dune migrate:flex [type]` | Apply pending schema migrations to Flex Object records. Omit `type` to migrate all types. |
| `dune migrate:generate` | Diff `schemas/*.yaml` against the database and emit SQL migration files to `migrations/`. |
| `dune migrate:run` | Apply all pending SQL migration files. |
| `dune migrate:status` | Show which migrations have been applied and which are pending. |

### Migration options

| Option | Default | Description |
|--------|---------|-------------|
| `--out <dir>` | `<root>/content` | Content directory to import into (import commands only) |
| `--dry-run` | — | Report what would be imported/migrated without writing any files |
| `--verbose` | — | Print each imported page |
| `--trust-source` | — | Skip HTML sanitization — only use for sources you fully trust |

See [Flex Object Schema Migrations](../flex-objects#schema-migrations) for full documentation on versioning schemas and writing migration files.

## Backup & Restore

| Command | Description |
|---------|-------------|
| `dune backup` | Create a compressed archive of all site data. |
| `dune restore <file>` | Restore a site from a backup archive. |

### `dune backup` options

| Option | Default | Description |
|--------|---------|-------------|
| `--output <file>` | `backup-<timestamp>.tar.gz` | Output file path. |
| `--root <dir>` | `.` | Site root to back up. |

Archives include `content/`, `data/`, `public/uploads/`, `site.yaml`, custom themes, and local plugins. Excluded: `.dune/cache/`, `node_modules/`, build artifacts. A `manifest.json` inside the archive records the Dune version and timestamp.

### `dune restore` options

| Option | Default | Description |
|--------|---------|-------------|
| `--yes` | — | Skip the confirmation prompt when restoring into a non-empty directory. |
| `--root <dir>` | `.` | Target site root to restore into. |

`dune restore` validates the manifest before extracting and warns when the backup was created by a different major version of Dune.

See [Backup & Restore](../../08.administration/06.backup-restore) for a full guide.

## Code generation

| Command | Description |
|---------|-------------|
| `dune codegen` | Generate TypeScript types (`src/db/types/`) and a repository index (`src/db/index.ts`) from `schemas/*.yaml`. Also generates REST API route handlers when schemas have an `api:` block. |

See [Data Layer](../../16.for-developers/04.data-layer) for full documentation.

## Scaffolding

| Command | Description |
|---------|-------------|
| `dune new [name]` | Create a new Dune site with starter content and default theme. |
| `dune new [name] --headless` | Create a headless Fresh+Dune site. No theme — you own all routes. See [Headless Mode](/docs/for-developers/headless-mode). |
| `dune generate --list` | List all available generators. |
| `dune generate:plugin <name>` | Scaffold a plugin at `plugins/{name}/index.ts`. |
| `dune generate:route <name>` | Create a content page at `content/{name}.md`. Name may include path separators (`blog/archive`). |
| `dune generate:form <name>` | Create a blueprint YAML at `schemas/{name}.yaml` with example fields. |
| `dune generate:theme <name>` | Scaffold a theme at `themes/{name}/` with `theme.yaml`, a default template, and a CSS file. |
| `dune generate:schema <name>` | Create a Flex Object schema at `flex-objects/{name}.yaml`. |
| `dune generate:admin-route <name>` | Scaffold a custom admin panel route at `src/admin/routes/{name}/index.tsx` with a matching handler and basic auth guard. |
| `dune deploy:init <target>` | Scaffold deployment config for the given target. |
| `dune update:skills` | Reinstall AI agent skill files from the current package into `.claude/skills/`. |

### Generator options

| Option | Description |
|--------|-------------|
| `--force` | Overwrite existing files. Without this flag, the command exits with an error if the target file already exists. |

All generators slugify the given name (lowercase, spaces and underscores become hyphens) and derive a title from the slug for use in frontmatter and YAML fields. The output path is printed on success.

### `dune deploy:init` targets and options

Supported targets: `fly`, `docker`, `deno-deploy`.

| Option | Default | Description |
|--------|---------|-------------|
| `--app <name>` | Derived from site title | App or service name. |
| `--region <code>` | `iad` | Fly.io primary region code. |
| `--port <n>` | `3000` | Internal port. |
| `--out <dir>` | Site root | Output directory for generated files. |

## Upgrade

| Command | Description |
|---------|-------------|
| `dune upgrade` | Update the `@dune/core` specifier in `deno.json` to the latest version on JSR. |

`dune upgrade` reads the site's `deno.json`, checks JSR for the latest `@dune/core` release, and writes the updated specifier. Deno fetches the new version automatically on next startup.

When running from a local source clone, the command prints the current version and the appropriate `git pull` command instead.

## Agent integration

| Command | Description |
|---------|-------------|
| `dune mcp:serve` | Start the Dune MCP server over stdio for AI agent integration. |

See [MCP Server](/docs/for-developers/mcp-server) for full documentation and configuration.

## Global flags

| Flag | Description |
|------|-------------|
| `--version`, `-V` | Print version and install source, then exit. |
| `--help`, `-h` | Print help, then exit. |
| `--root <dir>` | Site root directory (default: `.`). |
| `--port <n>` | Server port for `dev` and `serve` (default: `3000`). |
| `--debug` | Enable verbose debug output. |
| `--json` | Output result as machine-readable JSON (supported by `build`, `validate`, `content:list`, `content:check`, `content:create`, `content:delete`, `config:show`, `config:validate`, `blueprint:*`). |

### Diagnosing local vs JSR installs

`dune --version` shows where the CLI is running from:

```
dune 0.6.9 (jsr:@dune/core)           ← installed from JSR
dune 0.6.9 (source: /path/to/dune)    ← running from a local clone
```

This is useful when debugging version mismatches or confirming which code is active.

## Config show example

```bash
$ dune config:show

site.title: "My Site"                    ← config/site.yaml:1
system.cache.enabled: false              ← config/env/development/system.yaml:3
system.cache.driver: "memory"            ← default
system.debug: true                       ← config/env/development/system.yaml:5
theme.name: "default"                    ← default
```

Each value shows exactly where it came from in the merge hierarchy.
