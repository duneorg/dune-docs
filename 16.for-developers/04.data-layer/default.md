---
title: "Data Layer"
published: true
visible: true
taxonomy:
  audience: [developer]
  difficulty: [advanced]
  topic: [developer, database, schemas]
metadata:
  description: "Schema-first database layer with typed repositories, migrations, and optional REST API generation"
---

# Data Layer

Dune's data layer lets you define typed data models in `schemas/*.yaml`, then work with them through a generated Repository API backed by SQLite, Deno KV, or PostgreSQL. Optionally, `dune codegen` generates REST API endpoints for your models.

This is distinct from [Flex Objects](../../09.flex-objects) — Flex Objects are CMS-managed records edited through the admin UI; the data layer is for application data that your code creates and queries directly.

## Defining a schema

Create a YAML file in `schemas/` at your site root. The `model` field becomes the TypeScript type name; `table` defaults to a snake_case plural of the model name.

```yaml
# schemas/comments.yaml
model: Comment
table: comments      # optional — defaults to "comments" (snake_case plural)

fields:
  pageRoute:
    type: string
    required: true
    index: true        # creates a secondary index for fast lookups

  authorEmail:
    type: string
    required: true

  body:
    type: text         # "text" = long string (TEXT in SQL, no maxLength limit)
    required: true

  approved:
    type: boolean
    default: false

  createdAt:
    type: datetime
    default: "now"     # auto-set on insert

  updatedAt:
    type: datetime
    onUpdate: "now"    # auto-set on every update
```

### Field types

| Type | TypeScript | SQL | Notes |
|------|-----------|-----|-------|
| `string` | `string` | `TEXT` | Short strings; use `maxLength` to enforce |
| `text` | `string` | `TEXT` | Long text; no length limit |
| `integer` | `number` | `INTEGER` | Whole numbers |
| `number` | `number` | `REAL` | Floating point |
| `boolean` | `boolean` | `INTEGER 0/1` | |
| `datetime` | `Date` | `TEXT (ISO 8601)` | Use `default: "now"` and/or `onUpdate: "now"` |
| `json` | `unknown` | `TEXT (JSON string)` | Arbitrary JSON, serialised on write |

### Field options

| Option | Type | Description |
|--------|------|-------------|
| `required` | boolean | Cannot be null. Required fields become non-optional in the TypeScript interface. |
| `default` | any | Default value on insert. Use `"now"` for datetime auto-timestamps. |
| `onUpdate` | `"now"` | Set field to current timestamp on every update. |
| `maxLength` | number | Max string length (validated before write). |
| `index` | boolean | Create a secondary index on this field (SQLite/Postgres only). |
| `enum` | string[] | Restrict to these string values. The TypeScript type becomes a union of string literals. |

## Code generation

Run `dune codegen` to generate TypeScript types and a repository index from your schemas:

```bash
dune codegen
# Generated 3 file(s):
#   src/db/types/comment.ts
#   src/db/types/post.ts
#   src/db/index.ts
```

Filenames are `camelCase(model)`, not the model name verbatim — a multi-word model like `BlogPost` generates `src/db/types/blogPost.ts`, not `BlogPost.ts`.

`src/db/types/comment.ts` (example output):

```ts
// GENERATED — do not edit. Run `dune codegen` to regenerate.

export interface Comment {
  id: string;
  pageRoute: string;
  authorEmail: string;
  body: string;
  approved: boolean;
  createdAt: Date;
  updatedAt: Date;
}

export interface CommentCreate {
  pageRoute: string;
  authorEmail: string;
  body: string;
  approved?: boolean;
  // createdAt and updatedAt excluded — auto-managed
}

export interface CommentUpdate {
  pageRoute?: string;
  authorEmail?: string;
  body?: string;
  approved?: boolean;
}
```

`src/db/index.ts` exports a **single `db` object** keyed by table name — not one named export per model:

```ts
// GENERATED — do not edit.
import { createDbAdapter, createRepository } from "@dune/core/db";
import type { Comment, CommentCreate, CommentUpdate } from "./types/comment.ts";
// ... one import per model

const adapter = await createDbAdapter();

export const db = {
  comments: createRepository<Comment, CommentCreate, CommentUpdate>("comments", adapter),
  // ... one entry per model, keyed by table name
};

export type { Comment, CommentCreate, CommentUpdate };
```

`dune codegen` never writes an import-map alias into `deno.json` — `@/db` only resolves if your project's `deno.json` already maps `@/` to `./src/`; otherwise import the relative path. Re-run `dune codegen` whenever you change a schema file.

## Repository API

Import the generated `db` object and query your data — everything hangs off `db.{table}`:

```ts
import { db } from "@/db"; // or "../../src/db/index.ts" if you have no @/ alias

// Create
const comment = await db.comments.create({
  pageRoute: "/blog/hello-world",
  authorEmail: "alice@example.com",
  body: "Great post!",
});

// Find all
const all = await db.comments.find();

// Find with conditions
const approved = await db.comments.find({
  where: { approved: true, pageRoute: "/blog/hello-world" },
  orderBy: ["createdAt", "desc"],
  limit: 10,
});

// Find one — returns null if zero rows match, THROWS if more than one matches
const one = await db.comments.findOne({ where: { id: "abc123" } });

// Update — positional id, not { where, data }
await db.comments.update(comment.id, { approved: true });

// Delete — positional id, single row only
await db.comments.delete(comment.id);

// Count
const total = await db.comments.count({ where: { approved: false } });

// Upsert — positional (where, data)
const upserted = await db.comments.upsert(
  { pageRoute: "/blog/hello", authorEmail: "alice@example.com" },
  { pageRoute: "/blog/hello", authorEmail: "alice@example.com", body: "Updated" },
);
```

### WhereClause operators

Use operator objects for range and membership queries:

```ts
// Greater/less than
await db.comments.find({ where: { createdAt: { $gt: new Date("2026-01-01") } } });

// In set
await db.comments.find({ where: { pageRoute: { $in: ["/blog/a", "/blog/b"] } } });

// Not in set
await db.comments.find({ where: { approved: { $notIn: [false] } } });

// Contains (substring match)
await db.comments.find({ where: { body: { $contains: "hello" } } });

// Starts with
await db.comments.find({ where: { authorEmail: { $startsWith: "alice" } } });

// Null check
await db.comments.find({ where: { updatedAt: { $isNull: true } } });

// OR clause
await db.comments.find({
  where: {
    $or: [{ approved: true }, { authorEmail: "alice@example.com" }],
  },
});
```

## Adapter selection

Dune auto-selects the database backend from environment variables:

| Condition | Adapter |
|-----------|---------|
| `DUNE_DB_URL` starts with `postgres://` or `postgresql://` | PostgreSQL |
| `DENO_DEPLOYMENT_ID` is set | Deno KV |
| Otherwise | SQLite (file at `DUNE_DB_PATH`, default `data/dune.db`) |

You can force a specific adapter by setting the environment variable before starting the server:

```bash
# PostgreSQL
DUNE_DB_URL=postgresql://user:pass@localhost/mydb dune serve

# SQLite at a custom path
DUNE_DB_PATH=/var/data/site.db dune serve
```

## Migrations

Generate and apply SQL migrations when schemas change:

```bash
dune migrate:generate    # diff current schemas against DB, emit SQL migration files
dune migrate:run         # apply pending migration files
dune migrate:status      # show which migrations are applied and which are pending
```

Migration files are written to `data/migrations/` (not bare `migrations/`) and should be committed to version control. The `_dune_migrations` table in the database tracks which have been applied.

Typical workflow after changing a schema:

```bash
# 1. Edit schemas/comments.yaml
# 2. Generate the migration
dune migrate:generate
# → data/migrations/001_create_comments.sql

# 3. Review the generated SQL
cat data/migrations/001_create_comments.sql

# 4. Apply it
dune migrate:run
```

Migrations are not auto-applied on startup — run `dune migrate:run` as part of your deploy process.

## CRUD API generation

Add an `api:` block to a schema to generate REST endpoints for it:

```yaml
# schemas/posts.yaml
model: Post
fields:
  title:
    type: string
    required: true
  body:
    type: text
  authorId:
    type: string
    required: true
    index: true

api:
  enabled: true
  auth: "owner"          # "none" | "required" | "owner"
  ownerField: "authorId" # required when auth: "owner"
  methods:               # default: all five
    - list
    - get
    - create
    - update
    - delete
```

`dune codegen` generates route handlers in `src/routes/api/{model}/`:

| Route | Method | Description |
|-------|--------|-------------|
| `/api/posts` | `GET` | List records (paginated via `?limit=&offset=`) |
| `/api/posts` | `POST` | Create record |
| `/api/posts/{id}` | `GET` | Get single record |
| `/api/posts/{id}` | `PUT` | Update record (not `PATCH`) |
| `/api/posts/{id}` | `DELETE` | Delete record |

### Authentication modes

| Mode | Behaviour |
|------|-----------|
| `none` | No authentication required |
| `required` | Any logged-in `SiteUser` can access all records |
| `owner` | Reads/writes are scoped to the current user's own records (matched via `ownerField`) |

`owner` mode automatically filters list results to the current user's records and rejects updates/deletes on records owned by others.

## Raw adapter access

For queries the Repository API doesn't cover, use `getAdapter()`:

```ts
const adapter = db.comments.getAdapter();
const rows = await adapter.query<{ count: number }>(
  "SELECT COUNT(*) as count FROM comments WHERE approved = ?",
  [1],
);
```

Use with care — raw queries bypass the Repository's type safety and datetime serialisation.
