---
title: "CDN Cache Invalidation"
published: true
visible: true
taxonomy:
  audience: [webmaster, developer]
  difficulty: [intermediate]
  topic: [deployment, cdn, caching]
metadata:
  description: "Automatically purge CDN edge caches when Dune rebuilds your content"
---

# CDN Cache Invalidation

When Dune rebuilds your content — whether triggered by a content edit, an incoming webhook, or `dune cache:rebuild` — it can automatically send cache purge requests to your CDN. This keeps edge-cached pages in sync with the rebuilt content without requiring manual intervention or deploy hooks.

## How it works

After each successful rebuild, Dune collects the URLs of all changed pages and sends them to the configured CDN provider in batches. Unchanged pages are not purged. If the rebuild produces no changed pages, no purge request is sent.

Purge requests are fire-and-forget: they don't block the rebuild response and failures are logged as warnings, not errors.

## Configuration

Add a `cdn:` block to `config/system.yaml`:

```yaml
cdn:
  provider: cloudflare
  zone_id: "$CF_ZONE_ID"
  api_token: "$CF_API_TOKEN"
```

`"$ENV_VAR"` values are expanded from the process environment at startup — keep secrets out of committed config files.

### Supported providers

#### Cloudflare

```yaml
cdn:
  provider: cloudflare
  zone_id: "$CF_ZONE_ID"       # Settings → General → Zone ID in the Cloudflare dashboard
  api_token: "$CF_API_TOKEN"   # API token with Cache Purge permission
```

Calls the [Cloudflare Cache Purge API](https://developers.cloudflare.com/api/operations/zone-purge) (`POST /zones/{zone_id}/purge_cache`). Up to 30 URLs per request (Cloudflare's limit); Dune batches automatically.

#### Fastly

```yaml
cdn:
  provider: fastly
  service_id: "$FASTLY_SERVICE_ID"
  api_key: "$FASTLY_API_KEY"
```

Calls `POST /service/{service_id}/purge` for each changed URL via the Fastly Purge API.

#### BunnyCDN

```yaml
cdn:
  provider: bunny
  pull_zone_id: "$BUNNY_PULL_ZONE_ID"
  api_key: "$BUNNY_API_KEY"
```

Calls the BunnyCDN Pull Zone URL purge endpoint for each changed URL.

#### Custom webhook

```yaml
cdn:
  provider: custom
  url: "https://cdn.example.com/purge"   # POST endpoint
  token: "$CDN_PURGE_TOKEN"              # Sent as Authorization: Bearer {token}
```

Dune sends `POST {url}` with body `{ "urls": ["https://example.com/blog/post-1", ...] }` and header `Authorization: Bearer {token}`. The endpoint should return 2xx on success.

The custom provider validates that `url` is an HTTPS URL to prevent SSRF.

### Batch tuning

```yaml
cdn:
  provider: cloudflare
  zone_id: "$CF_ZONE_ID"
  api_token: "$CF_API_TOKEN"
  batch_size: 30        # Max URLs per request (default: 30)
  batch_delay_ms: 100   # Wait before flushing a partial batch (default: 100)
```

`batch_size` caps the number of URLs in a single purge request. `batch_delay_ms` controls how long to wait after the last URL is added before flushing — useful to coalesce rapid successive changes into fewer requests.

### Disabling temporarily

```yaml
cdn:
  provider: cloudflare
  zone_id: "$CF_ZONE_ID"
  api_token: "$CF_API_TOKEN"
  enabled: false   # Disable without removing the config
```

## Logs

Successful invalidations are logged at `info` level:

```
[cdn] purged 14 URLs via cloudflare (2 batches)
```

Failures are logged at `warn` level and include the provider response:

```
[cdn] purge failed: cloudflare returned 403 — check api_token permissions
```

Enable `DUNE_LOG_LEVEL=debug` to see each URL being purged.

## Verifying purges

You can trigger a manual rebuild and watch the logs:

```bash
DUNE_LOG_LEVEL=debug dune cache:rebuild
```

Or hit the incoming webhook endpoint to trigger a rebuild programmatically:

```bash
curl -X POST https://your-site.com/api/webhook/incoming \
  -H "Authorization: Bearer $DEPLOY_WEBHOOK_TOKEN" \
  -d '{"action":"rebuild"}'
```
