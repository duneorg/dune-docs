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

The migrator only touches block-style YAML. A file that has an old key it cannot reach (flow style such as `uploads: { maxSizeMb: 5 }`, a quoted key, a key inside a list item, or a file with Windows `CRLF` line endings) is reported and left unchanged, so rename those by hand. Programmatic config in `dune.config.ts` is not checked: an old key there is ignored, so rename it yourself.

Some settings did not just change spelling, they moved, so the migrator reports them and you move them by hand (the loader refuses them too, naming the new place):

| Removed | Now |
|---------|-----|
| `admin.honeypot` | `forms.honeypot` in `site.yaml` |
| `admin.notifications` | `forms.notifications` in `site.yaml` |
| `admin.incoming_webhooks` | `webhooks.incoming` in `system.yaml` |
| `admin.data_dir` (and `admin.dataDir`) | `data_dir` at the top level of `system.yaml` |
| `forms.notifications.email.smtp` | `site.email` |

`@dune/plugin-admin` reads the `admin:` block, so the rename ships with admin 4.0.0. With an older admin, core refuses to start if you have customised any renamed admin key, rather than falling back to defaults.

| Block | Old key → new key |
|-------|-------------------|
| `site.yaml` `auth:` | `sessionLifetime` → `session_lifetime`, `userStore` → `user_store`, `authzStore` → `authz_store` |
| `auth.providers` | `magicLink` → `magic_link`; per provider `clientId` → `client_id`, `clientSecret` → `client_secret` |
| `auth.jwt` | `jwksUrl` → `jwks_url`, `userIdClaim` → `user_id_claim`, `emailClaim` → `email_claim`, `rolesClaim` → `roles_claim` |
| `auth.webhook` | `signatureHeader` → `signature_header` |
| `site.yaml` `uploads:` | `maxSizeMb` → `max_size_mb`, `allowedTypes` → `allowed_types`, `requireAuth` → `require_auth` |
| `system.yaml` `content:` | `allowTsxFormat` → `allow_tsx_format` |
| `system.yaml` `search:` | `customFields` → `custom_fields` |
| `system.yaml` `metrics:` | `slowQueryThresholdMs` → `slow_query_threshold_ms` |
| `system.yaml` `admin:` | `sessionLifetime` → `session_lifetime`, `dataDir` → `data_dir`, `runtimeDir` → `runtime_dir`, `maxRevisions` → `max_revisions`, `maxUploadMb` → `max_upload_mb`, `authzStore` → `authz_store` |
| `admin.audit` | `logFile` → `log_file` |
| `admin.auth_provider` | `baseDn` → `base_dn`, `bindDn` → `bind_dn`, `bindPassword` → `bind_password`, `usernameAttr` → `username_attr`, `emailAttr` → `email_attr`, `nameAttr` → `name_attr`, `roleMap` → `role_map`, `defaultRole` → `default_role`, `entityId` → `entity_id`, `acsUrl` → `acs_url`, `idpMetadata` → `idp_metadata`, `roleAttr` → `role_attr` |

Options passed to functions in code (`createDuneAuthSystem({ authzStore, dataDir })`, `createSearchEngine({ customFields })`) are not config keys and keep their camelCase names.
