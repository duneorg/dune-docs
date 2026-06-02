---
title: "Configuration Schema"
published: true
visible: true
taxonomy:
  audience: [webmaster, developer]
  difficulty: [intermediate]
  topic: [reference, configuration]
metadata:
  description: "Complete configuration schema reference for site.yaml and system.yaml"
---

# Configuration Schema

## site.yaml

```yaml
# REQUIRED
title: string                    # Site title

# OPTIONAL (with defaults)
description: ""                  # string — Site description
url: "http://localhost:8000"     # string — Canonical base URL

author:
  name: ""                       # string — Author name
  email: ""                      # string — Author email (optional)

metadata: {}                     # Record<string, string> — HTML meta tags

taxonomies:                      # string[] — Enabled taxonomy types
  - "category"
  - "tag"

routes: {}                       # Record<string, string> — Route aliases
redirects: {}                    # Record<string, string> — 301 redirects
home: null                       # string | null — Home page slug. Auto-detected if null.
cors_origins: []                 # string[] — Extra origins allowed for cross-origin API requests

feed:
  enabled: true                  # boolean — Generate /feed.xml and /atom.xml (default: true)
  items: 20                      # number  — Most-recent dated pages to include (default: 20)
  content: "summary"             # "summary" | "full" — Feed item body (default: "summary")

sitemap:
  exclude: []                    # string[] — Route prefixes to omit from /sitemap.xml
  changefreq: {}                 # Record<string, changefreq> — Per-route overrides (longest prefix wins)
  # Valid changefreq values: "always" | "hourly" | "daily" | "weekly" | "monthly" | "yearly" | "never"

http_cache:
  default_max_age: 0             # number — Browser max-age in seconds (0 = revalidate every time)
  default_swr: 60                # number — Default stale-while-revalidate in seconds
  rules:                         # Per-route overrides — longest matching prefix wins
    - pattern: "/blog"           # string — URL prefix or exact path
      max_age: 3600              # number — Browser max-age override
      stale_while_revalidate: 86400  # number — SWR override
    - pattern: "/search"
      no_store: true             # boolean — Emit Cache-Control: no-store (disables all caching)

# ── Public site user authentication ─────────────────────────────────────────
auth:
  mode: "dune"                   # "dune" | "external-jwt" (default: "dune")
  sessionLifetime: 2592000       # number — Session TTL in seconds (default: 30 days)
  providers:
    github:
      clientId: "$GITHUB_CLIENT_ID"
      clientSecret: "$GITHUB_CLIENT_SECRET"
    google:
      clientId: "$GOOGLE_CLIENT_ID"
      clientSecret: "$GOOGLE_CLIENT_SECRET"
    discord:
      clientId: "$DISCORD_CLIENT_ID"
      clientSecret: "$DISCORD_CLIENT_SECRET"
    magicLink:
      enabled: true              # boolean — Enable magic-link email login
  authzStore: "local"            # "local" — Permission tuple storage.
                               #   dune mode: defaults to "local".
                               #   external-jwt mode: no default — omit for JWT-claims-only authz,
                               #   set "local" to enable local polizy authz alongside the external IdP.
  # external-jwt mode options (used when mode: "external-jwt")
  jwt:
    secret: "$JWT_SECRET"        # string — HMAC-SHA256 shared secret (HS256)
    jwksUrl: "https://..."       # string — JWKS endpoint for RS256 (Clerk, Auth0, etc.)
    userIdClaim: "sub"           # string — Claim containing user ID (default: "sub")
    emailClaim: "email"          # string — Claim containing email (default: "email")
    rolesClaim: "roles"          # string — Claim containing role(s) (default: "roles")

# ── Transactional email ───────────────────────────────────────────────────────
email:
  provider: "console"            # "smtp" | "resend" | "postmark" | "sendgrid" | "console"
  from: "hello@example.com"     # string — Default From address
  # SMTP
  smtp:
    host: "smtp.example.com"
    port: 587
    secure: false                # true = implicit TLS (port 465); false = STARTTLS
    user: "$SMTP_USER"
    pass: "$SMTP_PASS"
  # Resend
  resend:
    apiKey: "$RESEND_API_KEY"
  # Postmark
  postmark:
    apiKey: "$POSTMARK_API_KEY"
  # SendGrid
  sendgrid:
    apiKey: "$SENDGRID_API_KEY"

# ── Public file uploads ───────────────────────────────────────────────────────
uploads:
  max_size_mb: 10               # number — Max upload size in MB (default: 10)
  allowed_types:                # string[] — Permitted MIME types (server-derived from extension)
    - "image/jpeg"
    - "image/png"
    - "image/webp"
    - "image/gif"
    - "image/avif"
    - "application/pdf"
  require_auth: false           # boolean — Require a logged-in SiteUser (default: false)

# ── Payments ──────────────────────────────────────────────────────────────────
payments:
  provider: "stripe"            # "stripe" — only provider in this release
  secret_key: "$STRIPE_SECRET_KEY"      # string — Stripe secret key
  webhook_secret: "$STRIPE_WEBHOOK_SECRET"  # string — Stripe webhook signing secret
  products:
    - id: "membership"          # string — Site-defined product ID (used in checkout URL)
      name: "Monthly Membership" # string — Human-readable name
      price_id: "price_xxx"     # string — Stripe Price ID
      role: "member"            # string — Role assigned to user on successful payment
      mode: "subscription"      # "subscription" | "payment" (default: "subscription")

# ── Feature flags ─────────────────────────────────────────────────────────────
flags:
  comments: true                # boolean — static value
  new_editor: false
  beta_search: "env:ENABLE_BETA_SEARCH"  # string — resolves from env var at startup
```

## system.yaml

```yaml
content:
  dir: "content"                 # string — Content directory path
  markdown:
    extra: true                  # boolean — Extended markdown features
    auto_links: true             # boolean — Auto-link URLs
    auto_url_links: true         # boolean — Auto-link bare URLs

cache:
  enabled: true                  # boolean
  driver: "filesystem"           # "memory" | "filesystem" | "kv"
  lifetime: 3600                 # number — Seconds
  check: "file"                  # "file" | "hash" | "none"

images:
  default_quality: 80            # number — 1-100
  cache_dir: ".dune/cache/images"  # string
  allowed_sizes:                 # number[] — widths/heights allowed for on-the-fly processing
    - 320
    - 640
    - 768
    - 1024
    - 1280
    - 1536
    - 1920

languages:
  supported: ["en"]              # string[] — Language codes
  default: "en"                  # string — Must be in supported list
  include_default_in_url: false  # boolean — /en/page vs /page

typography:
  orphan_protection: true        # boolean — Insert &nbsp; before last word in paragraphs

search:
  customFields: []               # string[] — Extra frontmatter field names to include in search index
                                 #   e.g. ["summary", "author", "series"]
  highlight: true               # boolean — Return highlighted excerpts with <mark> tags (default: true)
  excerpt_length: 160           # number — Character length of returned excerpts (default: 160)
  include_flex: []              # string[] — Flex Object type names to include in search index
                                 #   e.g. ["products", "events"]
  facets: []                    # Facet definitions for filtering search results
    # - field: "taxonomy.category"   # dot-path into frontmatter
    # - field: "template"
  fields:                       # Per-field relevance weight multipliers (default weight: 1)
    title:
      weight: 3                 # number — Boost title matches (default: 3)
    summary:
      weight: 2                 # number — Boost summary/description matches
    body:
      weight: 1                 # number — Body text weight (default: 1)

page_cache:
  enabled: false                 # boolean — Enable in-process rendered HTML cache (default: false)
  max_entries: 500               # number — Max pages held in memory; oldest evicted when full (default: 500)
  ttl: 30                        # number — Seconds before an entry is re-rendered (default: 30)
  warm: false                    # boolean — Pre-resolve all pages at startup (default: false)

logging:
  format: "text"                 # "text" | "json" — Log output format (default: "text")
                                 # "json" emits NDJSON lines for log aggregators (Loki, Datadog, etc.)
                                 # Override with DUNE_LOG_FORMAT env var
  level: "info"                  # "debug" | "info" | "warn" | "error" — Minimum level (default: "info")
                                 # Override with DUNE_LOG_LEVEL env var

cdn:
  provider: "cloudflare"         # "cloudflare" | "fastly" | "bunny" | "custom"
  enabled: true                  # boolean — Enable invalidation on rebuild (default: true)
  batch_size: 30                 # number — Max URLs per invalidation request (default: 30)
  batch_delay_ms: 100            # number — ms to wait before flushing a partial batch (default: 100)

  # Cloudflare — set zone_id and api_token
  zone_id: "$CF_ZONE_ID"
  api_token: "$CF_API_TOKEN"

  # Fastly — set service_id and api_key
  service_id: "$FASTLY_SERVICE_ID"
  api_key: "$FASTLY_API_KEY"

  # BunnyCDN — set pull_zone_id and api_key
  pull_zone_id: "$BUNNY_PULL_ZONE_ID"
  api_key: "$BUNNY_API_KEY"

  # Custom webhook — POST { urls: string[] } to url with Bearer token
  url: "https://cdn.example.com/purge"
  token: "$CDN_PURGE_TOKEN"

tracing:
  enabled: false                 # boolean — Enable distributed tracing (default: false)
  endpoint: "http://localhost:4318/v1/traces"  # string — OTLP/HTTP collector URL
  service_name: "dune"           # string — Service name in all spans (default: "dune")
  sample_rate: 1.0               # number — 0.0–1.0 fraction of requests sampled (default: 1.0)

debug: false                     # boolean
timezone: "UTC"                  # string — IANA timezone
```

## admin.yaml (or admin: block in dune.config.ts)

```yaml
admin:
  enabled: true                  # boolean — Enable admin panel (default: true)
  path: "/admin"                 # string — URL prefix for the admin panel
  sessionLifetime: 86400         # number — Session lifetime in seconds (default: 86400 = 24 h)
  dataDir: "data"                # string — Persistent data directory (users, submissions, comments). Git-tracked.
  runtimeDir: ".dune/admin"      # string — Runtime data directory (sessions, revisions, staging, analytics). Not git-tracked.
  maxRevisions: 50               # number — Maximum revisions retained per page (default: 50)
  git_commit: false              # boolean — Auto-commit to git after every page save/publish (default: false)

  # Outbound webhooks — fired on content mutation events
  webhooks:
    - url: "https://hooks.example.com/content"
      secret: "$WEBHOOK_SECRET"   # string — HMAC-SHA256 signing secret ("$ENV_VAR" expansion supported)
      label: "My integration"     # string — Human-readable label for delivery logs (optional)
      enabled: true               # boolean — Disable without removing (default: true)
      events:                     # WebhookContentEvent[] — which events trigger this endpoint
        - onPageCreate
        - onPageUpdate
        - onPageDelete
        - onWorkflowChange

  # Incoming webhooks — let external systems trigger actions via POST /api/webhook/incoming
  incoming_webhooks:
    - token: "$DEPLOY_WEBHOOK_TOKEN"   # string — pre-shared token ("$ENV_VAR" expansion supported)
      actions: [rebuild]               # Array<"rebuild" | "purge-cache">
    - token: "$CACHE_WEBHOOK_TOKEN"
      actions: [purge-cache]
```

Valid `events` values: `onPageCreate`, `onPageUpdate`, `onPageDelete`, `onWorkflowChange`. See [Webhooks](../webhooks) for full documentation.

Valid incoming webhook `actions`: `rebuild` (re-indexes content), `purge-cache` (clears the processed image cache).

`dataDir` contains user accounts, form submissions, and editorial comments — should be committed to version control. `runtimeDir` contains ephemeral session, revision, staging, and analytics data — should be in `.gitignore`.

## theme config

Set in `dune.config.ts` or via config:

```yaml
theme:
  name: "default"                # string — Active theme name
  parent: null                   # string | null — Parent theme for inheritance
  custom: {}                     # Record<string, unknown> — Theme-specific settings
```

## plugins config

```yaml
plugins:
  plugin-name:                   # Plugin-specific config (arbitrary keys)
    key: value
```

## Validation

Run `dune config:validate` to check your config files. The validator produces actionable error messages:

```
✗ Config error in config/site.yaml:
  → site.taxonomies must be an array of strings
  → Got: "category, tag" (string)
  → Did you mean: ["category", "tag"]?
```
