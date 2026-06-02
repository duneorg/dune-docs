---
title: "Background Jobs"
published: true
visible: true
taxonomy:
  audience: [developer]
  difficulty: [intermediate]
  topic: [jobs, background, cron]
metadata:
  description: "Schedule recurring background tasks with Dune's cron-based job system"
---

# Background Jobs

Dune includes a cron-based background job system for recurring server-side tasks — digest emails, sitemap rebuilds, data synchronisation, cache warm-ups, and similar work.

## Defining a job

Create a TypeScript file in your site's `jobs/` directory:

```
jobs/
  weekly-digest.ts
  sitemap-rebuild.ts
```

Each file must export a `JobDefinition` as its default export:

```typescript
// jobs/weekly-digest.ts
import type { JobDefinition } from "@dune/core/jobs";

export default {
  name: "weekly-digest",
  schedule: "0 8 * * 1",   // every Monday at 08:00
  handler: async (ctx) => {
    const pages = await ctx.content.pages;
    const recent = pages
      .filter((p) => p.published && p.date)
      .sort((a, b) => (b.date ?? 0) - (a.date ?? 0))
      .slice(0, 5);

    await ctx.email.send({
      to: "team@example.com",
      subject: "Weekly digest",
      template: "digest",
      data: { pages: recent },
    });

    ctx.logger.info("digest_sent", { count: recent.length });
  },
} satisfies JobDefinition;
```

### `JobDefinition` fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | `string` | Unique identifier. Used in logs, state files, and the admin trigger endpoint. |
| `schedule` | `string` | Standard 5-field cron expression (`min hour dom month dow`). |
| `handler` | `(ctx: JobContext) => Promise<void> \| void` | The job function. |

### `JobContext`

| Property | Type | Description |
|----------|------|-------------|
| `content` | `DuneEngine` | Full access to the content index and site engine. |
| `config` | `DuneConfig` | Merged site and system configuration. |
| `storage` | `StorageAdapter` | Raw storage adapter for plugin-specific reads/writes. |
| `logger` | `JobLogger` | Structured logger. Entries include the job name automatically. |
| `email` | `EmailClient` | Transactional email client. Present when an email provider is configured. Guard with `ctx.config.site.email?.provider` if email may not be set up. |

## Enabling jobs in site.yaml

Jobs must be explicitly declared in `site.yaml` before they are loaded:

```yaml
jobs:
  - ./jobs/weekly-digest.ts
  - ./jobs/sitemap-rebuild.ts
```

Only the listed files are loaded. Files present in `jobs/` that are not declared are ignored — this prevents arbitrary code execution if unexpected files appear in the directory. Set `jobs: []` to disable all jobs with no warning.

Omitting the `jobs:` key entirely falls back to auto-discovery (scanning all `.ts` files in `jobs/`) with a startup warning. The security fix in 0.13 introduced the explicit allowlist; migrate by adding the `jobs:` list to your config.

## Scheduling

Dune uses standard 5-field cron syntax:

```
┌───── minute (0–59)
│ ┌───── hour (0–23)
│ │ ┌───── day of month (1–31)
│ │ │ ┌───── month (1–12)
│ │ │ │ ┌───── day of week (0–6, Sunday=0)
│ │ │ │ │
* * * * *
```

Common examples:

| Expression | Meaning |
|-----------|---------|
| `0 * * * *` | Every hour on the hour |
| `0 8 * * 1` | Every Monday at 08:00 |
| `0 0 * * *` | Daily at midnight |
| `*/15 * * * *` | Every 15 minutes |
| `0 3 1 * *` | First of each month at 03:00 |

On Deno Deploy, Dune uses `Deno.cron()` for native scheduling. On all other environments, a 1-minute tick interval is used.

## Job state

Each job's execution state is persisted to `{runtimeDir}/jobs/{name}.json`:

| Field | Description |
|-------|-------------|
| `lastRun` | Timestamp (ms) of the most recent execution start, or `null` if never run. |
| `nextRun` | Best-estimate timestamp (ms) of the next scheduled run. |
| `status` | `"idle"`, `"running"`, or `"errored"`. |
| `lastError` | Error message from the most recent failed run, or `null`. |

State files are in `runtimeDir` (default `.dune/admin/`) — gitignored, not committed to version control.

## Manual trigger

Trigger a job immediately from the admin panel or via the API:

```
POST /admin/api/jobs/{name}/run
```

Requires admin authentication and a valid CSRF token. Returns `200` with `{ "ok": true }` once the handler completes, or `500` with `{ "error": "…" }` if it throws.

## Error handling

If a job handler throws, the error is logged (including the message) and the job's `status` is set to `"errored"`. Scheduling continues — the next tick will fire regardless of the previous run's outcome. There is no automatic retry.

## Multi-process deployments

If your site runs multiple server processes (e.g. `--workers 4`), each process runs its own scheduler. All processes will attempt to fire the same job at the same scheduled time. Dune emits a startup warning when `workers > 1` and jobs are configured.

To avoid duplicate execution in multi-process deployments, either:
- Run jobs on a single dedicated worker using a process-level guard (e.g. `DUNE_JOB_WORKER=1` env var checked in the handler), or
- Use an external job queue (Redis, Postgres `LISTEN/NOTIFY`) with your handler as the consumer.
