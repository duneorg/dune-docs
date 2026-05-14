---
title: "Graceful Shutdown"
published: true
visible: true
taxonomy:
  audience: [webmaster, developer]
  difficulty: [intermediate]
  topic: [deployment, operations]
metadata:
  description: "How Dune handles SIGTERM for zero-downtime deploys and process managers"
---

# Graceful Shutdown

When Dune receives `SIGTERM` or `SIGINT`, it enters a graceful drain phase before exiting. In-flight HTTP requests are allowed to finish; new connections are held at the load balancer; the process exits cleanly once all work is done.

This is the standard behaviour expected by process managers (systemd, Docker, Fly.io, Kubernetes) for rolling deploys without dropped requests.

## Shutdown sequence

1. **Signal received** — `SIGTERM` or `SIGINT` arrives.
2. **Health check flips** — `GET /health/ready` immediately returns `503 Service Unavailable`. Load balancers and orchestrators that poll the readiness endpoint stop routing new requests to this instance.
3. **In-flight drain** — Dune waits for all requests currently being processed to finish.
4. **Timeout** — If in-flight requests don't finish within the drain timeout (default 30 seconds), Dune exits anyway. Long-running requests are interrupted.
5. **Clean exit** — The process exits with code `0`.

## Drain timeout

Control the drain window with the `DUNE_SHUTDOWN_TIMEOUT_MS` environment variable:

```bash
DUNE_SHUTDOWN_TIMEOUT_MS=60000 dune serve   # 60-second drain
```

Default is `30000` (30 seconds). Set it to match the longest expected request duration in your application. For most content sites 30 seconds is generous; if you have long-running API requests (exports, bulk operations), increase it accordingly.

## Health check endpoint

```
GET /health/ready
```

Returns `200 OK` with body `{"ok":true}` during normal operation.  
Returns `503 Service Unavailable` with body `{"ok":false,"reason":"shutting down"}` during the drain phase.

Load balancers should be configured to remove a backend from rotation when `/health/ready` returns non-2xx. Most platforms (Kubernetes, Fly.io, Traefik) do this automatically.

The liveness endpoint is separate:

```
GET /health/live
```

Always returns `200 OK`. Use this for liveness probes — it never returns 503 even during shutdown.

## systemd

```ini
[Service]
ExecStart=/usr/local/bin/dune serve --root /var/www/my-site
KillSignal=SIGTERM
TimeoutStopSec=45
```

Set `TimeoutStopSec` to a value slightly larger than `DUNE_SHUTDOWN_TIMEOUT_MS` so systemd doesn't force-kill the process before the drain completes.

## Docker

```dockerfile
STOPSIGNAL SIGTERM
```

Docker sends `SIGTERM` by default on `docker stop`. Set `--time` to give the container enough time to drain:

```bash
docker stop --time=45 my-dune-container
```

## Kubernetes

```yaml
spec:
  containers:
    - name: dune
      lifecycle:
        preStop:
          exec:
            command: ["/bin/sleep", "5"]  # give the LB time to drain
  terminationGracePeriodSeconds: 60
```

The `preStop` sleep gives the load balancer time to finish updating its routing table before Dune starts draining. `terminationGracePeriodSeconds` should be larger than the preStop sleep plus `DUNE_SHUTDOWN_TIMEOUT_MS / 1000`.

## Fly.io

Fly sends `SIGTERM` when your machine is stopped or replaced during a deploy. No additional configuration is needed — Dune's default 30-second drain fits within Fly's stop timeout. If you have long requests, increase `DUNE_SHUTDOWN_TIMEOUT_MS` and set `[processes] stop_timeout` in `fly.toml` accordingly.
