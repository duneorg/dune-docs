---
title: "Distributed Tracing"
published: true
visible: true
taxonomy:
  audience: [developer]
  difficulty: [advanced]
  topic: [administration, observability, tracing]
metadata:
  description: "Enable OpenTelemetry-compatible tracing to diagnose slow requests and pipeline bottlenecks"
---

# Distributed Tracing

Dune's tracing support adds OpenTelemetry-compatible spans to the major pipeline stages — content rebuilds, HTTP request handling, and storage reads. Traces are exported via OTLP/HTTP to any compatible backend: Jaeger, Grafana Tempo, Honeycomb, Datadog, or any OpenTelemetry Collector.

Tracing is off by default. No OTLP dependency is required unless you enable it.

## What gets traced

| Span | When created |
|------|-------------|
| `engine.rebuild` | Every content rebuild — includes `pageCount` attribute |
| `http.request` | Every incoming HTTP request — includes `pathname` and `status` attributes |

Each `http.request` span carries the request's `traceId`, which also appears in [structured log lines](structured-logging) when JSON logging is enabled — making it easy to jump from a slow log entry to its trace.

## Configuration

```yaml
# config/system.yaml
tracing:
  enabled: true
  endpoint: "http://localhost:4318/v1/traces"
  service_name: "my-site"   # optional, default: "dune"
  sample_rate: 1.0           # optional, default: 1.0 (trace everything)
```

| Field | Default | Description |
|-------|---------|-------------|
| `enabled` | `false` | Enable tracing. |
| `endpoint` | — | OTLP/HTTP collector URL. Required when `enabled: true`. |
| `service_name` | `"dune"` | `service.name` resource attribute attached to all spans. |
| `sample_rate` | `1.0` | Fraction of requests to trace (0.0–1.0). |

## Backends

Any OTLP/HTTP-compatible backend works. Common setups:

**Jaeger (local)**
```bash
docker run -d --name jaeger \
  -p 4318:4318 \
  -p 16686:16686 \
  jaegertracing/all-in-one:latest
```
Then open `http://localhost:16686` and search for service `dune`.

**Grafana Tempo**
Point `endpoint` at your Tempo OTLP/HTTP receiver, typically `http://tempo:4318/v1/traces`.

**Honeycomb**
```yaml
tracing:
  enabled: true
  endpoint: "https://api.honeycomb.io/v1/traces"
  # Add your API key as an Authorization header via a reverse proxy or
  # Otel Collector processor — Dune sends unauthenticated OTLP by default.
```

**OpenTelemetry Collector**
Route through a Collector for enrichment, sampling, and fan-out to multiple backends. Point `endpoint` at the Collector's OTLP/HTTP receiver.

## Trace IDs in logs

When both tracing and JSON logging are enabled, every log line for a traced request includes a `traceId` field:

```json
{"level":"info","traceId":"4bf92f3577b34da6a3ce929d0e0e4736","logger":"router","path":"/blog/post","status":200,"durationMs":78}
```

This lets you correlate a slow log entry directly with its trace in the backend UI. Set up a derived field in Grafana (or a similar link in your log tool) to turn `traceId` into a clickable link to the trace.

## Sampling

Set `sample_rate` to reduce trace volume in production:

```yaml
tracing:
  enabled: true
  endpoint: "http://collector:4318/v1/traces"
  sample_rate: 0.1   # trace 10% of requests
```

Head-based sampling: the decision is made at the start of each request. Rebuilds (`engine.rebuild` spans) are always traced regardless of `sample_rate`.

## Performance impact

The tracer is a lightweight no-op when `enabled: false`. When enabled, each span adds a small amount of allocation and a non-blocking OTLP export (fire-and-forget HTTP POST). The export does not block request handling — a failure to reach the collector is logged as a debug warning and ignored.
