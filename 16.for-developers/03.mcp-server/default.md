---
title: "MCP Server"
published: true
visible: true
taxonomy:
  audience: [developer]
  difficulty: [intermediate]
  topic: [mcp, agents, api]
metadata:
  description: "Using Dune's MCP server to give AI coding agents direct access to your site's content engine"
---

# MCP Server

Dune ships a built-in [Model Context Protocol](https://modelcontextprotocol.io) (MCP) server that lets AI coding agents — Claude Code, Cursor, and others — read your site's content, schema, and configuration directly, without a running web server.

## Starting the server

```bash
dune mcp:serve
```

The server reads from stdin and writes to stdout using newline-delimited JSON-RPC 2.0. It loads your site's content index at startup and serves requests from there — no HTTP port is opened.

## Security model

The MCP server is a **local-development tool** with no authentication of its own: any process that can write to its stdin can call every tool, including the write tools. The suggested configs below run it with `deno -A` (all permissions), so tools execute with your full user privileges — they can read and write files and reach the network.

Keep these rules in mind:

- Only connect the server to agents you trust, on machines you trust.
- Never bridge its stdio to a network socket or any other untrusted transport — there is no session, token, or origin check.
- `install_plugin` and `update_config` modify `site.yaml`, which controls what code your site executes later. Treat MCP tool access as equivalent to code execution.
- To reduce blast radius, replace `-A` with the narrowest permission flags that work for your site, for example:

```json
"args": [
  "run",
  "--allow-read=.", "--allow-write=.", "--allow-env", "--allow-net=jsr.io",
  "jsr:@dune/core/cli", "mcp:serve"
]
```

## Configuring Claude Code

Add an entry to `.mcp.json` in your site root (or `~/.claude.json` for global access):

```json
{
  "mcpServers": {
    "dune": {
      "command": "deno",
      "args": ["run", "-A", "jsr:@dune/core/cli", "mcp:serve"],
      "cwd": "/path/to/your/site"
    }
  }
}
```

After restarting Claude Code, the `dune` server appears in the MCP tools panel and all tools are available in your conversations.

## Configuring Cursor

Add the same entry under `mcpServers` in Cursor's MCP config file (`~/.cursor/mcp.json`):

```json
{
  "mcpServers": {
    "dune": {
      "command": "deno",
      "args": ["run", "-A", "jsr:@dune/core/cli", "mcp:serve"],
      "cwd": "/path/to/your/site"
    }
  }
}
```

## Tools

Tools are callable functions that agents can invoke with parameters.

### Read tools

| Tool | Description |
|------|-------------|
| `list_pages` | List pages in the content index. Filter by template, published status, language, taxonomy, or date range. |
| `get_page` | Get a single page by route — full frontmatter, rendered HTML, media files, and metadata. |
| `get_page_source` | Read the raw source (YAML frontmatter + markdown body) of a page by route. |
| `search_content` | Full-text search across all pages. Returns matches with relevance scores and excerpts. |
| `get_taxonomy` | Get all values for a taxonomy (e.g. tags, categories) with per-value page counts. Omit the name argument to list all taxonomies. |
| `get_config` | Get the site configuration summary — title, URL, theme, taxonomies, languages, feature flags. Secrets are never included. |
| `get_runtime_info` | Live snapshot of the engine state — page counts, content formats, sections, plugins, theme details. |
| `list_templates` | List all templates and layouts available in the active theme. |
| `list_blueprints` | List all blueprint (frontmatter schema) definitions. Returns field names, types, and validation rules per template. |

### Write tools

Write tools modify site content and configuration directly on disk. They're registered by `mcp:serve` unconditionally — there's no read-only or remote-mount mode that disables them; if the server is running, the write tools are available.

| Tool | Description |
|------|-------------|
| `write_page` | Write or overwrite a content file. Accepts a path relative to the content dir and the full file content (frontmatter + body). Writes the content as-is — does not parse or validate YAML frontmatter before writing. |
| `delete_page` | Delete a content file by route (e.g. `/blog/hello`) or by path relative to the content dir. |
| `update_frontmatter` | Patch frontmatter fields on an existing page. Pass `null` as a value to remove a field. Leaves the body unchanged. |
| `update_config` | Merge fields into `site.yaml`. Accepts a partial config object; existing keys not in the patch are preserved. |
| `install_plugin` | Add a plugin specifier to the `plugins:` list in `site.yaml`. No-op if already present. |
| `scaffold_plugin` | Generate a plugin skeleton at `plugins/{name}/index.ts`. |
| `scaffold_route` | Generate a Markdown content page stub at `content/{path}.md` (not TSX, despite the name). |
| `scaffold_form` | Generate a form/blueprint schema at `schemas/{name}.yaml` with example fields. |
| `scaffold_theme` | Generate a minimal theme skeleton at `themes/{name}/`. |

> **Note**: Scaffold tools (`scaffold_*`) invoke the same generators as `dune generate:*` CLI commands. Each generator accepts a `log` callback (falling back to `console.log` when none is given) — the MCP server passes a scoped collector instead of touching global `console.log`, so concurrent scaffold calls don't interleave output. The collected lines are returned as the tool result.

## Resources

Resources are static URIs that agents can read directly.

| URI | Description |
|-----|-------------|
| `dune://site/config` | Full site configuration as JSON. |
| `dune://site/schema` | JSON Schema for `site.yaml` — the same as `GET /_dune/schema/config`. |
| `dune://content/pages` | Complete page index as JSON. |
| `dune://content/taxonomy` | All taxonomy data with page counts. |
| `dune://content/blueprints` | All blueprint definitions. |
| `dune://content/forms` | All schema/form definitions under `schemas/` — read before `scaffold_form` to avoid name collisions or confusing this with the DB data-layer's own `schemas/*.yaml` format (see [Data Layer](../04.data-layer)). |
| `dune://site/audit` | Last 50 admin audit-log entries. Empty array when audit logging isn't enabled. |

## Skill files

`dune new` installs a set of agent skill files into `.claude/skills/` that brief an AI session on Dune's conventions, file layout, plugin shape, config schema, and common tasks. These complement the MCP server — the skills provide static background knowledge while the MCP tools provide live site data.

To reinstall or update skill files in an existing project:

```bash
dune update:skills
```
