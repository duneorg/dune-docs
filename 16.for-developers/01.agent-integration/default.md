---
title: "AI Agent Integration"
published: true
visible: true
taxonomy:
  audience: [developer]
  difficulty: [beginner]
  topic: [agents, mcp, ai, cli]
metadata:
  description: "How AI coding agents like Claude Code and Cursor work with Dune sites — llms.txt, MCP server, skill files, and the change API"
---

# AI Agent Integration

Dune is designed to work well with AI coding agents. Four complementary layers give an agent the context and capabilities it needs to work on a Dune project accurately.

## llms.txt and llms-full.txt

The docs site publishes two files for agent consumption:

- **`/llms.txt`** — a concise overview of Dune's conventions, config schema, common tasks, and known gotchas. Fits comfortably in a context window.
- **`/llms-full.txt`** — the complete docs in a single flat file, for deep research or one-shot ingestion.

These follow the [llms.txt standard](https://llmstxt.org) and are served directly from the Dune docs site:

```
https://dune.dev/llms.txt
https://dune.dev/llms-full.txt
```

In Claude Code, you can load `llms.txt` at the start of a session with:

```
/fetch https://dune.dev/llms.txt
```

## Skill files

`dune new` installs a set of skill files into `.claude/skills/` that brief an agent on Dune's conventions without requiring a fetch. They cover:

| Skill file | Topic |
|------------|-------|
| `dune-content.md` | Content conventions, frontmatter, file naming, ordered folders |
| `dune-themes.md` | Theme structure, TSX templates, Preact islands, layout components |
| `dune-schemas.md` | Data models — `store: local` and `store: db`, query interface |
| `dune-plugin-authoring.md` | Plugin shape, hooks, admin routes, security guards |
| `dune-auth.md` | Public user authentication, `ctx.state.user`, protecting routes |
| `dune-authz.md` | Authorization via polizy, `authz.check()`, content gating |
| `dune-email.md` | Transactional email, `email.send()`, template formats |
| `dune-jobs.md` | Background jobs, cron schedules, `JobContext` |
| `dune-mcp.md` | MCP server tools and resources reference |

To update skill files in an existing project after upgrading Dune:

```bash
dune update:skills
```

## MCP server

The MCP server gives an agent live access to the content index, config, plugins, and blueprints — without a running web server. See [MCP Server](/docs/for-developers/mcp-server) for setup and the full tools and resources reference.

Start it with:

```bash
dune mcp:serve
```

## Change API

The `POST /admin/api/dev/apply` endpoint lets an agent apply batched content and config mutations in a single validated request — without writing files directly. It supports `dry_run: true` for previewing changes before committing them. Available in dev mode only.

See [REST API — Apply batched mutations](/docs/reference/api#apply-batched-mutations-dev-mode-only) for the full request and response shape.

## How the layers fit together

| Layer | What it provides | When to use |
|-------|-----------------|-------------|
| `llms.txt` | Static conventions, gotchas, patterns | Start of session; one-time ingestion |
| Skill files | Domain-specific background in `.claude/skills/` | Loaded automatically by Claude Code |
| MCP server | Live site state — content tree, config, templates | Querying before generating or editing |
| Change API | Validated writes back to the site | Applying batched changes with feedback |
