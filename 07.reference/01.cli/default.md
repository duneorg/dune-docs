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
| `dune dev --env-file` | Load `.env` from the site root into the process environment before plugins/config are read. |
| `dune dev --env-file=path/to/file` | Same, from an explicit path (relative to the site root, or absolute). |
| `dune ps` | List your dune dev/serve instances currently running on this machine — PID, mode, version, site title, root, the port(s) actually listening, and the command line. Mostly read-only: it scans the process table, so it works retroactively on anything already running, and it sends a `GET /health/live` to each listening instance (this appears in the instance's access log). Finds servers started through the generated `main.ts`, the `dune` CLI itself (`cli.ts dev`, the global shim, `jsr:@dune/core/cli`) and `deno task dev|serve`; a server's wrapper, re-exec child and esbuild workers are reported as one instance. Instances also record themselves in `~/.dune/instances/` (`DUNE_REGISTRY_DIR` overrides), so one started in an unusual way is still listed. A process that is running but holds no port is shown as **not listening**. A listening instance that fails three health probes in a row (about 10 seconds) is shown as **NOT RESPONDING**: a blocked event loop accepts the connection but never answers. It probes the port the instance registered, else every listening port, counting only dune's own answer. A process that matches dune only by its command line, with no `config/site.yaml` at the root that command line points to, is listed but marked unconfirmed. Only your own processes are shown. It needs permission to run `ps` and `lsof` (`--allow-run`, or `-A`) and says so rather than reporting no instances. macOS/Linux only. |
| `dune stop <pid>` | Stop one instance, with its **whole process tree** (including any other tool a `deno task` wrapper started). The pid can be any process `dune ps` shows for it, including an unconfirmed one (with a warning). The prompt shows each instance's full command line. Sends SIGTERM and waits up to 10 seconds for a server to drain; a process that ignores it is reported, not killed. Each pid's start time is re-checked immediately before it is signalled, so a pid that exited and was reused while the prompt was open is never hit. Asks first; `--yes` skips the prompt, and without a terminal it refuses unless `--yes` is given. It never signals itself or its parent processes. Exit code 1 if the pid matches no instance or something would not stop. |
| `dune stop --port <n>` | Stop the instance listening on that port. |
| `dune stop --idle` | Stop only **confirmed** instances that aren't serving: running but **not listening**, or listening but **not responding**, for at least 2 minutes (younger ones may still be starting, for example indexing a large site). Leaves live servers alone. An instance whose port state is unknown (no `lsof`) is never treated as idle. |
| `dune stop --all` | Stop every **confirmed** instance `dune ps` lists. An instance is confirmed when its own command line points at a dune site (the directory of the `main.ts` it runs, its `--root`, or the dune CLI's working directory, each holding a `config/site.yaml`) or when it registered itself in the instance registry. Merely running from inside a site directory confirms nothing, so another Deno project's `main.ts serve` is left out and reported; name it by pid if you mean it. A target is always required — a bare `dune stop` only prints usage (exit 2) — so stopping everything has to be asked for by name. |
| `dune stop … --force` | Also SIGKILL processes that ignore SIGTERM after the grace period, naming them. Opt-in: killing a wrapper process can leave its child running. A server stuck in a loop can't handle SIGTERM at all, so this is how to end a hung one. |

`--env-file` is off by default — nothing auto-loads secrets from disk on a plain `dune dev`/`serve`. It parses simple `KEY=VALUE` lines (blank lines and `#` comments skipped, surrounding quotes stripped); a key already set in the environment always wins over the file. An explicitly-requested file that doesn't exist is a hard error, not a silent no-op. Also accepted by `dune serve`.

## Build & Cache

| Command | Description |
|---------|-------------|
| `dune build` | Build content index, validate config. Run before production serving. |
| `dune build --static` | Generate a fully static site into `dist/` (SSG). |
| `dune cache:clear` | Delete all cached data (rendered HTML, content index, images). |
| `dune cache:rebuild` | Rebuild content index from scratch. Use after bulk content changes. |
| `dune validate` | Whole-project lint: config, plugins, templates, schemas, and content. |
| `dune doctor` | Environment/runtime health checks — distinct from `validate`'s project-correctness checks: can this site's dependency graph actually resolve and boot, on this machine, right now? Checks Deno version, dependency resolution (`deno cache main.ts`, reformatting an npm-cache-mismatch error into actionable steps instead of a raw stack trace), and lockfile staleness. Fast — no live boot. |
| `dune doctor --boot` | Also spawns the site for real (on a free ephemeral port) and makes a request against it. Slower and more failure-prone than the default checks, hence opt-in. |
| `dune doctor --json` | Machine-parseable output for either form above. |

`dune validate` checks:

- Config structure and field types (`site.yaml`, `system.yaml`)
- Plugin specs are pinned to a version
- All `template:` values in frontmatter resolve to an existing theme template
- Schema files under `schemas/` have a `store:` field
- Content integrity: missing titles, duplicate routes, future dates
- Content files that failed to parse (e.g. malformed YAML frontmatter) — these are otherwise silently dropped from routing, search, and the sitemap with no other signal

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
| `dune content:check` | Validate all content: broken links, missing templates, orphaned media, content files that failed to parse. Frontmatter-only by default — pass `--render` to also compile every `.md`/`.mdx` page body and catch render failures (e.g. an MDX syntax error) that indexing alone can't see. |
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

Fires `onPageCreate` — the same hook the admin panel fires when creating a page through the UI.

### `dune content:delete` options

| Option | Description |
|--------|-------------|
| `--confirm` | Confirm deletion without an interactive prompt. |
| `--dry-run` | Preview what would be deleted without writing any changes. |
| `--json` | Output result as JSON. |

Fires `onPageDelete` — the same hook the admin panel fires when deleting a page through the UI.

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

`plugin:install`, `plugin:remove` and `plugin:update` edit `config/site.yaml` in place: only the affected lines change, so comments, blank lines and quote style survive (removing a plugin also removes the comment lines directly above it). Each edit is re-parsed and checked against the intended result before it is written. If the file uses a layout the edit can't handle safely (for example a flow-style `plugins: [...]` list), the command exits with an error and leaves the file untouched. All three accept `--dry-run` to show what would change without writing.

## Themes

| Command | Description |
|---------|-------------|
| `dune theme:list` | List local themes in `themes/` and registered package themes from `site.yaml`. |
| `dune theme:install <src>` | Register a theme package (`jsr:`, `npm:`, or local path). Adds to `themes:` and `deno.json`, syncs the lockfile. |
| `dune theme:install <src> --name <slug>` | Override the derived theme name. |
| `dune theme:install <src> --activate` | Also set `theme.name` / `theme.src` as the active theme. |
| `dune theme:remove <name>` | Remove a theme from the `themes:` registry (does not delete local `themes/{name}/`). |
| `dune theme:publish [dir]` | Publish a theme package to JSR (`deno publish` in the package directory). |

Remote specifiers must pin an exact semver (`jsr:@scope/theme-name@1.0.0`). After install, run `dune dev` or restart the server to load the theme.

If the installed theme declares a `parent:` in its `theme.yaml` (see [Theme Inheritance](../../04.themes/02.inheritance)), `theme:install` walks the parent chain and installs any ancestor not already registered or present under `themes/`, recursing through multi-level chains. There's no recorded "install source" for a parent slug, so it's derived by convention: for a local path install, a sibling package directory (`theme-{parent}` next to the child's own package dir); for a `jsr:` install, a same-scope `theme-{parent}` package, resolved to its latest published version. Installing fails with an actionable error if the sibling directory doesn't exist or the JSR parent package isn't published yet — install it manually first in either case.

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
| `dune migrate:users` | Reshape existing `data/users/` records (added by earlier admin-account/public-auth accounts before they shared one store) to the current unified `User` shape — adds a `by-email/` index entry, backfills missing fields. Idempotent. |

### Migration options

| Option | Default | Description |
|--------|---------|-------------|
| `--out <dir>` | `<root>/content` | Content directory to import into (import commands only) |
| `--dry-run` | — | Report what would be imported/migrated without writing any files |
| `--verbose` | — | Print each imported page |
| `--trust-source` | — | Skip HTML sanitization — only use for sources you fully trust |

See [Flex Object Schema Migrations](../flex-objects#schema-migrations) for full documentation on versioning schemas and writing migration files.

## Users

| Command | Description |
|---------|-------------|
| `dune users:create <email> [--role x[,y]] [--name "..."]` | Create a `User` record before the person has ever logged in, marked `provider: "invited"`. An admin-tier role (`admin`/`editor`/`author`) syncs the `app:admin` authz tuple immediately, same as `users:grant-role`. |
| `dune users:grant-role <email> <role>` | Grant an admin-tier role (`admin`, `editor`, or `author`) to an existing user, identified by email. Updates `roles[]` and syncs the corresponding authz tuple — the same two things the admin panel's user-edit UI does over HTTP. |
| `dune users:revoke-role <email> <role>` | Revoke an admin-tier role from an existing user. |

For operators without (or before) web access: first-admin bootstrap on a headless install, scripted/CI-driven site provisioning, or granting admin access to an existing OAuth/magic-link account that has never had one. Idempotent — granting a role already held, or revoking one not held, is a no-op. Accepts `--dry-run`.

An invited person's first magic-link login matches `users:create`'s record by email and just works. First login via OAuth does not — the account-takeover guard rejects an email-only match from a different provider, so closed/invite-only OAuth signup needs a separate feature, not this command.

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
| `dune new [name]` | Create a new Dune site with starter content and default theme. Automatically runs `dune doctor`'s fast checks afterward (Deno version, dependency resolution, lockfile staleness) and prints any findings — informational only, doesn't fail the scaffold. |
| `dune new [name] --verify` | Same as above, but also runs `dune doctor --boot` (spawns the site for real and makes a request against it) and exits non-zero if that fails. |
| `dune new [name] --headless` | Create a headless Fresh+Dune site. No theme — you own all routes. See [Headless Mode](/docs/for-developers/headless-mode). |
| `dune generate --list` | List all available generators. |
| `dune generate:plugin <name>` | Scaffold a plugin at `plugins/{name}/index.ts`. |
| `dune generate:route <name>` | Create a content page at `content/{name}.md`. Name may include path separators (`blog/archive`). |
| `dune generate:form <name>` | Create a form definition at `forms/{name}.yaml` with example fields. |
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
| `dune upgrade [version]` | Move every `@dune/core` reference in `deno.json` to the latest version on JSR, or to `version`. |

`dune upgrade` reads the site's `deno.json`, checks JSR for the latest `@dune/core` release, and moves **every** `jsr:@dune/core@…` reference to it: the `@dune/core` import, its subpath entries (such as `@dune/core/ui/editable`) and the CLI specifier in legacy `tasks`. Each reference keeps its own style: an exact pin stays exact, and `^0.35` stays a caret on the minor. Then it syncs the lockfile. Deno fetches the new version automatically on next startup.

Pass a full version to set it explicitly instead of the latest, for example `dune upgrade 0.35.1`. An explicit version is applied to every reference, so it also works for rollbacks, and for upgrading while the newest release is still inside Deno's minimum-dependency-age window.

After upgrading, a site that still launches through a versioned CLI specifier in its `tasks` is pointed at `dune migrate:entrypoint`, and plugin versions are pointed at `dune plugin:update`; neither is changed by `dune upgrade` itself.

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
| `--help`, `-h` | Print help, then exit. After a command (`dune plugin:update --help`) it prints that command's usage instead of running it. Only `plugin:update` has its own usage text so far; other commands show the global help. |
| `--root <dir>` | Site root directory (default: `.`). |
| `--port <n>` | Server port for `dev` and `serve` (default: `3000`). |
| `--debug` | Enable verbose debug output. |
| `--json` | Output result as machine-readable JSON (supported by `build`, `validate`, `content:list`, `content:check`, `content:create`, `content:delete`, `config:show`, `config:validate`, `blueprint:*`). |
| `--render` | (`content:check` only) Compile every `.md`/`.mdx` page body and report pages that fail to render, not just frontmatter issues. Off by default — slower since it renders the whole site. |

### Diagnosing local vs JSR installs

`dune --version` shows where the CLI is running from:

```
dune 0.6.9 (jsr:@dune/core)           ← installed from JSR
dune 0.6.9 (source: /path/to/dune)    ← running from a local clone
```

This is useful when debugging version mismatches or confirming which code is active.

If you're developing Dune itself (not a site built with it), the local-dev install documented in the [repo README](https://github.com/duneorg/dune#readme) creates a global `dune` shim pointed at your checkout. That shim freezes a snapshot of `deno.json` at install time, so it silently goes stale the moment `deno.json`'s `imports` change afterward — failing with a confusing `Import "X" not in import map` error unrelated to whatever was actually just changed.

| Command | Description |
|---------|-------------|
| `dune dev:link` | Reinstall the global `dune` shim against this checkout, refreshing its frozen import-map snapshot. Run it after pulling changes that touch `deno.json`'s `imports`. |

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
