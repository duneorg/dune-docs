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

### Migration options

| Option | Default | Description |
|--------|---------|-------------|
| `--out <dir>` | `<root>/content` | Content directory to import into |
| `--dry-run` | — | Report what would be imported without writing any files |
| `--verbose` | — | Print each imported page |
| `--trust-source` | — | Skip HTML sanitization — only use for sources you fully trust |

## Scaffolding

| Command | Description |
|---------|-------------|
| `dune new [name]` | Create a new Dune site with starter content and default theme. |
| `dune new [name] --headless` | Create a headless Fresh+Dune site. No theme — you own all routes. See [Headless Mode](/docs/for-developers/headless-mode). |
| `dune deploy:init <target>` | Scaffold deployment config for the given target. |
| `dune update:skills` | Reinstall AI agent skill files from the current package into `.claude/skills/`. |

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
