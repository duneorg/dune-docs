---
title: "Backup & Restore"
published: true
visible: true
taxonomy:
  audience: [webmaster]
  difficulty: [beginner]
  topic: [administration, backup]
metadata:
  description: "Create and restore point-in-time backups of your Dune site data"
---

# Backup & Restore

`dune backup` creates a single compressed archive of everything that matters for a Dune site. `dune restore` extracts it safely into a site directory, with version compatibility checks.

## What gets backed up

| Included | Excluded |
|----------|----------|
| `content/` | `.dune/cache/` |
| `data/` (users, submissions, sessions) | `node_modules/` |
| `public/uploads/` | Build artifacts (`dist/`) |
| `site.yaml` | Installed plugins from JSR/npm |
| Custom themes (`themes/` — local only) | |
| Local plugins (`plugins/`) | |
| A `manifest.json` with version and timestamp | |

Plugins and themes installed from JSR or npm are excluded — they are re-fetched on next startup from their specifiers in `deno.json`.

## Creating a backup

```bash
dune backup
# → backup-2026-05-14T10-30-00.tar.gz

dune backup --output /mnt/backups/site-$(date +%F).tar.gz
```

The archive is a standard gzip-compressed tar. You can inspect it with any `tar` tool:

```bash
tar -tzf backup-2026-05-14T10-30-00.tar.gz | head -20
```

The manifest inside the archive:

```json
{
  "version": "1.0.0",
  "createdAt": "2026-05-14T10:30:00.000Z",
  "files": ["content/", "data/", "public/uploads/", "site.yaml"]
}
```

## Restoring a backup

```bash
dune restore backup-2026-05-14T10-30-00.tar.gz
```

If the target directory is non-empty, Dune prompts for confirmation before extracting. Pass `--yes` to skip the prompt in scripts:

```bash
dune restore backup-2026-05-14T10-30-00.tar.gz --yes
```

Dune validates the manifest before extracting and warns if the backup was made with a different major version:

```
⚠  Backup was created with Dune 0.9.1, running 1.0.0 — proceed with care.
```

## Scheduling backups

On Linux/macOS with cron:

```cron
0 3 * * * cd /path/to/my-site && dune backup --output /backups/dune-$(date +\%F).tar.gz
```

Or using a systemd timer — create `/etc/systemd/system/dune-backup.timer` and a matching `.service` unit that runs `dune backup`.

## What to back up for full disaster recovery

For a complete restore you also need:

- Your `deno.json` (plugin and theme specifiers)
- Your `config/` directory (if separate from the site root)
- Any environment variables used by the site (`.env` file or secrets manager export)

These are outside the scope of `dune backup` because they are typically managed by your deployment tooling.
