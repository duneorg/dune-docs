---
title: "Configuration"
published: true
visible: true
taxonomy:
  audience: [webmaster, developer]
  difficulty: [beginner]
  topic: [configuration]
metadata:
  description: "Configuring Dune: YAML files, environment overrides, and programmatic config"
collection:
  items:
    "@self.children": true
  order:
    by: order
    dir: asc
---

# Configuration

Dune's configuration is layered: simple YAML files for most needs, TypeScript for advanced cases, and environment-specific overrides for deployment.

## Config files

```
config/
├── site.yaml               # Site identity: title, URL, taxonomies
├── system.yaml             # Engine behavior: cache, debug, languages
└── env/
    ├── development/
    │   └── system.yaml     # Dev overrides (debug: true, etc.)
    └── production/
        └── system.yaml     # Production overrides
```

## Merge order

Configuration merges from general to specific. Each layer overrides the previous:

```
1. System defaults (hardcoded in Dune)
    ↓ merged with
2. config/system.yaml + config/site.yaml
    ↓ merged with
3. config/env/{DUNE_ENV}/*.yaml
    ↓ merged with
4. dune.config.ts (programmatic)
    ↓ merged with
5. Page frontmatter (per-page)
```

You only need to set what you want to change. Everything else uses sensible defaults.

## Key naming

Every authored config key is `snake_case` (`session_lifetime`, `max_upload_mb`, `jwks_url`). Before 0.36 a number of keys were camelCase, sometimes next to snake_case siblings in the same block; those were renamed once, with no aliases.

A config file that still carries an old key **refuses to load**. The error names each old key, its new name and the file, so a renamed setting (a JWKS URL, an authorization store) can never be silently ignored.

### Upgrading to 0.36

Run the migrator from the site root, then check the diff:

```bash
dune migrate:config --dry-run   # list what would change
dune migrate:config             # rewrite the files in place
```

It rewrites `config/site.yaml`, `config/system.yaml` and every file under `config/env/*/`, keeps comments and key order, and is safe to run twice. At a multisite install root pass `--site <id>`.

The migrator only touches block-style YAML. A file that has an old key it cannot reach (flow style such as `metrics: { slowQueryThresholdMs: 50 }`, a quoted key, a key inside a list item, or a file with Windows `CRLF` line endings) is reported and left unchanged, so rename those by hand. Programmatic config in `dune.config.ts` gets the same check: an old key in its `site:` or `system:` block is refused at load, naming the new key. The migrator cannot rewrite a `.ts` file, so rename it yourself.

Some settings did not just change spelling, they moved, so the migrator reports them and you move them by hand (the loader refuses them too, naming the new place):

| Removed | Now |
|---------|-----|
| `admin.honeypot` | `forms.honeypot` in `site.yaml` |
| `admin.notifications` | `forms.notifications` in `site.yaml` |
| `admin.incoming_webhooks` | `webhooks.incoming` in `system.yaml` |
| `admin.data_dir` (and `admin.dataDir`) | `data_dir` at the top level of `system.yaml` |
| `admin.runtime_dir` (and `admin.runtimeDir`) | `runtime_dir` at the top level of `system.yaml` |
| `admin.max_revisions` (and `admin.maxRevisions`) | `max_revisions` at the top level of `system.yaml` |
| the whole `admin:` block of `system.yaml` | the admin plugin's `config:` (under its entry in `plugins:` in `site.yaml`); `dune migrate:config` moves it, and lifts `runtime_dir`, `max_revisions` and `data_dir` to the top level of `system.yaml`. A block in an environment overlay, or one holding `honeypot`, `notifications` or `incoming_webhooks`, you move by hand |
| `machine_translation` (site.yaml) | the admin plugin's `config:`, with `apiKey` → `api_key` and `baseUrl` → `base_url`; `dune migrate:config` moves it |
| `admin.enabled` | removed. Core loads the admin only when `plugins:` lists `jsr:@dune/plugin-admin@^4`; `dune migrate:config` deletes the key and adds the entry (or, for `enabled: false`, just deletes it) |
| `admin.authz_store` (and `admin.authzStore`) | removed. The admin's authz store is always local |
| `forms.notifications.email.smtp` | `site.email` |

`@dune/plugin-admin` reads the `admin:` block, so the rename ships with admin 4.0.0. With an older admin, core refuses to start if you have customised any renamed admin key, rather than falling back to defaults.

| Block | Old key → new key |
|-------|-------------------|
| `site.yaml` `auth:` | `sessionLifetime` → `session_lifetime`, `userStore` → `user_store`, `authzStore` → `authz_store` |
| `auth.providers` | `magicLink` → `magic_link`; per provider `clientId` → `client_id`, `clientSecret` → `client_secret` |
| `auth.jwt` | `jwksUrl` → `jwks_url`, `userIdClaim` → `user_id_claim`, `emailClaim` → `email_claim`, `rolesClaim` → `roles_claim` |
| `auth.webhook` | `signatureHeader` → `signature_header` |
| `system.yaml` `content:` | `allowTsxFormat` → `allow_tsx_format` |
| `system.yaml` `search:` | `customFields` → `custom_fields` |
| `system.yaml` `metrics:` | `slowQueryThresholdMs` → `slow_query_threshold_ms` |
| the admin plugin's `config:` | `sessionLifetime` → `session_lifetime`, `maxUploadMb` → `max_upload_mb`, `audit.logFile` → `audit.log_file` (the `dune migrate:config` rewrite happens before the move below) |

### Keys that were removed

These were declared and documented but nothing ever read them, so setting one never changed anything. A config that still sets one **keeps booting**: Dune logs a `config.key.ignored` warning (once per file and key) and ignores it. `dune migrate:config` deletes the keys and their nested blocks. The one exception is `admin.auth_provider` with `ldap` or `saml`, which is refused because ignoring it would leave you without the login you configured.

| Key | What to use |
|-----|-------------|
| `site.yaml` `uploads:` | Nothing: Dune never mounted a public upload endpoint. Accept visitor files with a `file` field in a [form](../forms) |
| `system.yaml` `cache:` (`enabled`, `driver`, `lifetime`, `check`) | `page_cache:` is the rendered-page cache; see [Caching](../deployment/caching) |
| `system.yaml` `timezone:` | Nothing |
| `system.yaml` `content.markdown:` (`extra`, `auto_links`, `auto_url_links`) | Nothing: the markdown renderer has fixed settings |
| `system.yaml` `search.include_flex:` | Nothing |
| `admin.auth_provider` with `type: ldap` or `saml` (and its `base_dn`, `bind_dn`, ... keys) | Only `local` exists; inject your own provider ([Authentication Providers](../administration/auth-providers)) |

Two keys the docs always described now work: `search.custom_fields`, `search.fields`, `search.highlight` and `search.excerpt_length` reach the search engine, and `flags:` (in **system.yaml**) is resolved at startup for `flag()`.

Options passed to functions in code (`createDuneAuthSystem({ authzStore, dataDir })`, `createSearchEngine({ customFields })`) are not config keys and keep their camelCase names.
