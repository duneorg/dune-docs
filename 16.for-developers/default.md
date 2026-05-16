---
title: "Dune for Developers"
published: true
visible: false
taxonomy:
  audience: [developer]
  difficulty: [intermediate]
metadata:
  description: "Build themes, extend Dune, and integrate with its APIs"
---

# Dune for Developers

Build custom themes, extend Dune with plugins and format handlers, and integrate via the REST API or CLI.

## Your learning path

### 1. Understand the system

[Installation](/docs/getting-started/installation) — install Deno and Dune  
[Project Structure](/docs/getting-started/project-structure) — content, themes, config layout  
[Programmatic Config](/docs/configuration/programmatic-config) — configure Dune in TypeScript

### 2. Build themes

[Templates](/docs/themes/templates) — Preact TSX templates and template props  
[Theme Inheritance](/docs/themes/inheritance) — extend and override themes  
[Creating a Theme](/docs/themes/creating-a-theme) — from scratch  
[Islands](/docs/themes/islands) — client-side interactivity in SSR pages

### 3. Extend Dune

[Hooks](/docs/extending/hooks) — lifecycle hooks for content and routing  
[Format Handlers](/docs/extending/format-handlers) — add new content formats  
[MDX Content](/docs/extending/mdx-content) — JSX inside Markdown  
[Plugins](/docs/extending/plugins) — packaging and distributing extensions  
[Email](/docs/extending/email) — transactional email via `email.send()` with SMTP, Resend, Postmark, or SendGrid  
[Payments](/docs/extending/payments) — Stripe checkout, webhooks, and billing portal

### 3a. Build data-driven features

[Data Layer](/docs/for-developers/data-layer) — schema-first data modelling, Repository API, KV/SQLite/PostgreSQL adapters, migrations  
[UI Components](/docs/for-developers/ui-components) — `@dune/core/ui` Preact components for search, auth, profiles, and payments

### 3b. Add user authentication

[Authentication](/docs/authentication) — public site user auth overview  
[Public Auth](/docs/authentication/public-auth) — OAuth (GitHub, Google, Discord), magic links, external JWT  
[Content Gating](/docs/authentication/content-gating) — role-based page access, redirect and 403 behaviour

### 3c. Control access (authorization)

[Authorization](/docs/authentication/authorization) — polizy-backed authz, `createDuneAuthSystem`, `authz.check()`, route middleware, external-JWT + local authz mode

### 4. Work with AI agents

[AI Agent Integration](/docs/for-developers/agent-integration) — llms.txt, skill files, MCP server, and the change API  
[MCP Server](/docs/for-developers/mcp-server) — live content and config context over the Model Context Protocol

### 5. Go headless (optional)

[Headless Mode](/docs/for-developers/headless-mode) — Fresh developer owns all routes; Dune manages content and admin  
`dune new my-site --headless` to scaffold a headless project

### 6. Deploy and integrate

[Deno Deploy](/docs/deployment/deno-deploy) — zero-config cloud deployment  
[Static Site Generation](/docs/deployment/static) — build to static HTML  
[REST API](/docs/reference/api) — content and search API endpoints  
[CLI Commands](/docs/reference/cli) — `dune serve`, `dune build`, `dune new`

### 7. Reference

[Config Schema](/docs/reference/config-schema) — full configuration reference  
[Search](/docs/reference/search) — search engine API and index format  
[API Stability](/docs/reference/stability) — what's stable, what may change

---

*Use the sidebar filter to keep only developer-relevant pages visible as you navigate.*
