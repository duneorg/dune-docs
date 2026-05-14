---
title: "Structured Logging"
published: true
visible: true
taxonomy:
  audience: [webmaster, developer]
  difficulty: [intermediate]
  topic: [administration, logging, observability]
metadata:
  description: "Configure Dune's log format and level for human terminals or log aggregation pipelines"
---

# Structured Logging

Dune emits logs for server startup, content rebuilds, request handling, and background operations. The format and verbosity are configurable for both local development and production log aggregation.

## Formats

**Text (default)** — human-readable, ANSI-coloured output for terminals:

```
INFO  2026-05-14T10:30:00.123Z  server  Listening on http://0.0.0.0:3000
INFO  2026-05-14T10:30:01.456Z  engine  Rebuild complete — 42 pages in 312ms
WARN  2026-05-14T10:30:02.789Z  cdn     Purge failed: cloudflare returned 403
```

**JSON** — NDJSON (one JSON object per line) for log aggregators like Loki, Datadog, or Splunk:

```json
{"level":"info","time":"2026-05-14T10:30:00.123Z","service":"dune","logger":"server","msg":"Listening on http://0.0.0.0:3000"}
{"level":"info","time":"2026-05-14T10:30:01.456Z","service":"dune","logger":"engine","msg":"Rebuild complete","pages":42,"durationMs":312}
{"level":"warn","time":"2026-05-14T10:30:02.789Z","service":"dune","logger":"cdn","msg":"Purge failed","provider":"cloudflare","status":403}
```

JSON logs always include `level`, `time`, `service`, `logger`, and `msg`. Additional structured fields (like `pages`, `durationMs`, `status`) are appended for relevant events.

## Configuration

Set format and level in `config/system.yaml`:

```yaml
logging:
  format: "json"    # "text" | "json"
  level: "info"     # "debug" | "info" | "warn" | "error"
```

Or use environment variables (take precedence over config):

```bash
DUNE_LOG_FORMAT=json DUNE_LOG_LEVEL=debug dune serve
```

Environment variables are useful for toggling log format in production without modifying committed config files.

## Log levels

| Level | When used |
|-------|-----------|
| `debug` | Detailed internals: individual URL purges, cache hits/misses, trace spans, middleware decisions |
| `info` | Normal operations: server start, rebuild complete, CDN purge summary, request IDs |
| `warn` | Non-fatal issues: CDN purge failure, missing optional config, deprecated usage |
| `error` | Fatal or request-failing conditions: unhandled exceptions, config parse errors |

The default level is `info`. Set `debug` locally when diagnosing issues; use `warn` or `error` in production to reduce volume.

## Request IDs

Every incoming request gets a unique `requestId` (16 hex characters) that appears in log lines for that request lifecycle. If the client sends an `X-Request-Id` header, that value is used instead — useful for correlating logs across services or with your load balancer's access log.

With text format:

```
INFO  …  router [req=a1b2c3d4e5f6a7b8] GET /blog/my-post → 200 (34ms)
```

With JSON format:

```json
{"level":"info","logger":"router","requestId":"a1b2c3d4e5f6a7b8","method":"GET","path":"/blog/my-post","status":200,"durationMs":34}
```

## Shipping logs to aggregators

When running under systemd or Docker, redirect stdout to your aggregator of choice:

**Loki + Promtail** — configure Promtail to tail the process's stdout/stderr with `pipeline_stages.json`:

```yaml
# promtail config snippet
pipeline_stages:
  - json:
      expressions:
        level: level
        logger: logger
  - labels:
      level:
      logger:
```

**Datadog** — set `DUNE_LOG_FORMAT=json` and configure the Datadog Agent's `logs_config` to collect from the process or container stdout.

**Docker** — use the `json-file` log driver (default) and forward with `docker logs --follow` or a sidecar log shipper.

## Plugin logging

Plugins receive a pre-configured child logger via `hooks.logger`:

```ts
const plugin: DunePlugin = {
  name: "my-plugin",
  setup(hooks) {
    const log = hooks.logger;   // child logger scoped to "my-plugin"

    hooks.on("onRebuild", async () => {
      log.info("rebuild started");
      // → {"logger":"my-plugin","msg":"rebuild started",...}
    });
  },
};
```

Child logger entries inherit the parent's format and level and include `logger: "my-plugin"` automatically.

## Trace ID correlation

When [Distributed Tracing](distributed-tracing) is enabled, each log line for a traced request includes a `traceId` field:

```json
{"level":"info","logger":"router","traceId":"4bf92f3577b34da6a3ce929d0e0e4736","requestId":"a1b2c3d4e5f6a7b8","msg":"GET /blog/post","status":200}
```

This lets you jump from a log line directly to the corresponding trace in Jaeger, Grafana Tempo, or Honeycomb.
