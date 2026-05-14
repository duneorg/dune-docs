---
title: "Schema Reference"
published: true
visible: true
taxonomy:
  audience: [developer]
  difficulty: [advanced]
  topic: [developer, database, reference]
metadata:
  description: "Quick reference for schema YAML fields, WhereClause operators, and Repository methods"
---

# Schema Reference

## Schema YAML fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `model` | Yes | string | TypeScript interface name (e.g. `"Comment"`) |
| `table` | No | string | SQL table name. Default: snake_case plural of model |
| `fields` | Yes | map | Field definitions — see below |
| `api` | No | object | CRUD API generation config — see below |

### Field definition

| Option | Required | Type | Description |
|--------|----------|------|-------------|
| `type` | Yes | string | One of: `string`, `text`, `integer`, `number`, `boolean`, `datetime`, `json` |
| `required` | No | boolean | Non-null constraint. Default: `false` |
| `default` | No | any | Default value on insert. Use `"now"` for datetime auto-timestamps |
| `onUpdate` | No | `"now"` | Set to current timestamp on every update (datetime fields only) |
| `maxLength` | No | number | Maximum string length (enforced before write) |
| `index` | No | boolean | Create a secondary index on this field |
| `enum` | No | string[] | Restrict to these values; TypeScript type becomes a string union |

### `api:` block

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `enabled` | No | boolean | Default: `true` when `api:` block is present |
| `auth` | Yes | string | `"none"` \| `"required"` \| `"owner"` |
| `methods` | No | string[] | Subset of `["list","get","create","update","delete"]`. Default: all five |
| `ownerField` | Required when `auth: "owner"` | string | Field name that stores the owner's user ID |

---

## Repository methods

All methods return Promises and throw on database errors.

| Method | Signature | Description |
|--------|-----------|-------------|
| `find` | `(opts?: FindOptions) → T[]` | All rows matching options |
| `findOne` | `({ where }) → T \| null` | One row; throws if multiple match |
| `create` | `(data: TCreate) → T` | Insert row; returns inserted record with generated `id` |
| `update` | `(id, data: TUpdate) → { count }` | Update by id; returns rows affected |
| `delete` | `(id) → { count }` | Delete by id; returns rows affected |
| `count` | `({ where? }) → number` | Count matching rows |
| `upsert` | `(where, data: TCreate) → T` | Insert or replace by where clause |
| `getAdapter` | `() → DbAdapter` | Raw adapter for escape-hatch queries |

### FindOptions

```ts
interface FindOptions<T> {
  where?: WhereClause<T>;
  orderBy?: keyof T | [keyof T, "asc" | "desc"];
  limit?: number;
  offset?: number;
}
```

---

## WhereClause operators

A plain value means exact match (`field: value`). Use operator objects for range and membership queries.

| Operator | Type | Description |
|----------|------|-------------|
| `$gt` | V | Greater than |
| `$lt` | V | Less than |
| `$gte` | V | Greater than or equal |
| `$lte` | V | Less than or equal |
| `$in` | V[] | Value is in the array |
| `$notIn` | V[] | Value is not in the array |
| `$contains` | string | Substring match (string fields only) |
| `$startsWith` | string | Prefix match (string fields only) |
| `$isNull` | boolean | `true` = IS NULL, `false` = IS NOT NULL |
| `$or` | WhereClause[] | Top-level OR of multiple where clauses |

---

## CLI commands

| Command | Description |
|---------|-------------|
| `dune codegen` | Generate `src/db/types/*.ts` and `src/db/index.ts` from schemas |
| `dune migrate:generate` | Diff schemas against DB and write SQL migration files to `migrations/` |
| `dune migrate:run` | Apply all pending migration files |
| `dune migrate:status` | Show which migrations are applied and which are pending |

---

## Environment variables

| Variable | Description |
|----------|-------------|
| `DUNE_DB_URL` | PostgreSQL connection string (`postgres://...`). Triggers PostgreSQL adapter. |
| `DUNE_DB_PATH` | SQLite file path. Default: `data/dune.db`. |
| `DENO_DEPLOYMENT_ID` | Set automatically by Deno Deploy. Triggers KV adapter. |
