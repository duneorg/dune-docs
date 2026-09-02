---
title: "Installation"
published: true
visible: true
taxonomy:
  audience: [webmaster, developer]
  difficulty: [beginner]
  topic: [setup]
metadata:
  description: "Install Deno and create your first Dune project"
---

# Installation

## Install Deno

Dune runs on Deno 2.x. If you don't have it yet:

```bash
# macOS / Linux
curl -fsSL https://deno.land/install.sh | sh

# Windows (PowerShell)
irm https://deno.land/install.ps1 | iex

# Homebrew
brew install deno
```

Verify your installation:

```bash
deno --version
# deno 2.x.x
```

## Create a new site

```bash
dune new my-site
cd my-site
```

This scaffolds a minimal Dune site:

```
my-site/
├── main.ts               # Site entrypoint — generated, don't customize
├── deno.json             # Import map and dev/build/serve tasks
├── .vscode/
│   └── settings.json     # YAML schema autocompletion for config/*.yaml
├── .mcp.json             # Claude Code MCP server config for this site
├── .claude/
│   └── skills/           # AI agent skill files (10 files) — how an agent
│                          #   session should work with this site
├── content/              # Your content lives here
│   ├── 01.home/
│   │   └── default.md    # Your homepage
│   └── 02.blog/
│       ├── blog.md       # Blog listing page
│       └── 01.hello-world/
│           └── post.md   # Your first post
├── config/
│   ├── site.yaml         # Site identity (title, URL, taxonomies)
│   └── system.yaml       # Engine behavior (cache, debug, languages)
└── themes/
    └── starter/          # Starter theme
        ├── templates/
        │   ├── default.tsx   # Page template
        │   ├── blog.tsx      # Blog listing template
        │   └── error.tsx     # 404/error pages
        ├── components/
        │   └── layout.tsx
        ├── islands/          # Preact islands (hydrated in the browser)
        │   └── NavToggle.tsx
        └── theme.yaml
```

This is everything `dune new` writes — nothing here is left for you to fill in before the site runs. [Project Structure](getting-started/project-structure) covers additional files (like `dune.config.ts` or `config/env/`) that are optional and not part of this scaffold.

## Scaffolding inside an existing Deno workspace

If `my-site` lands inside a directory tree that's already a Deno workspace (a parent `deno.json` with a `"workspace"` array — common if you keep several projects under one folder), `deno task dev` will fail with `Config file must be a member of the workspace` the moment you try to start it. `dune new` detects this and prints the exact fix — a `⚠️` warning naming the enclosing workspace's `deno.json` and the line to add to its `"workspace"` array. Add that line, then continue below.

In that same setup you may also see several `Warning "minimumDependencyAge" field can only be specified in the workspace root deno.json file` lines ahead of every `dune`/`deno task` command's real output. That's Deno itself, not Dune — it fires whenever a sibling project's own `deno.json` (valid on its own, when that project is used standalone) also happens to set `minimumDependencyAge` and gets linked in as a member of your enclosing workspace. Harmless; nothing to fix on the site side.

## Start the dev server

```bash
deno task dev
```

(A globally-installed `dune` binary also works — `dune dev` re-execs itself under the site's own `deno.json` — but `deno task dev` is what `dune new` itself recommends and needs no global install.)

Open `http://localhost:3000`. You should see your homepage.

The dev server watches for changes — edit `content/01.home/default.md` and your browser will refresh automatically.

**Using a different port** — pass `--port`: `deno task dev --port 4000`. Useful if `3000` is already taken by another project; the server prints the URL it actually bound once it starts, so check that if you're not sure what port ended up in use.

## What just happened?

1. Dune scanned the `content/` directory and built a **content index** — a lightweight map of every page, its route, and its frontmatter
2. It loaded `config/site.yaml` and `config/system.yaml`, merging them with defaults
3. It started a Fresh 2 server. GET requests for content pages are routed through Fresh's `ctx.render()`, which handles the response and injects Fresh's client bootstrap script into every HTML page — enabling island hydration once you add interactive components
4. When you visited `/`, Dune found `01.home/default.md`, loaded its Markdown, rendered it to HTML, passed it to your theme's `default.tsx` template, and returned the result via Fresh

## Troubleshooting

**Template changes not appearing after server restart**

Deno caches compiled `.tsx` files on disk. In rare cases the cached version survives a server restart and serves stale output. Force a clean compile with:

```bash
DENO_DIR=$(mktemp -d) dune dev
```

This uses a fresh Deno compile cache for the session. You can safely delete the stale cache manually from `~/Library/Caches/deno/gen/` (macOS) or `~/.cache/deno/gen/` (Linux) if you prefer a permanent fix.

**`dune dev`/`dune serve` fails with an npm module resolution error**

If the build fails with `[ERR_MODULE_NOT_FOUND]` or `Could not find referrer npm package`, naming two different resolved versions of the same package (e.g. `preact/10.29.8_1/...` in one line, `npm:preact@^10.29.1` in another), this is a stale or inconsistent local npm cache — not a Dune bug, and not something a fresh `dune new` scaffold can cause on its own. Try, in order:

```bash
deno cache --reload
```

If that doesn't clear it, remove Deno's npm cache directory entirely and retry:

```bash
# macOS
rm -rf ~/Library/Caches/deno/npm

# Linux
rm -rf ~/.cache/deno/npm

# Windows (PowerShell)
Remove-Item -Recurse -Force "$env:LOCALAPPDATA\deno\npm"
```

This is most likely to happen if the same Deno installation was previously used for another Fresh/Preact project with a different dependency version.
