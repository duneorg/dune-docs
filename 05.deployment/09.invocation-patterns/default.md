---
title: "Invocation Patterns"
published: true
visible: true
taxonomy:
  audience: [webmaster]
  difficulty: [intermediate]
  topic: [deployment, server, lockfile]
metadata:
  description: "How to invoke Dune in production — JSR-URL vs installed binary, and the lockfile-as-build-artifact model"
---

# Invocation Patterns

There are two ways to run Dune in production. Both are fully supported; which one to choose depends on how you prefer to manage upgrades and how your deployment pipeline is structured.

## The two patterns

### JSR-URL (version in deno.json)

```bash
deno run -A jsr:@dune/core@0.26.0/cli serve
```

The version is pinned in `deno.json` alongside your site's other imports. Upgrading core is a one-line edit to that file. This is what `dune new` generates and what `dune upgrade` manages.

```json
{
  "imports": {
    "@dune/core": "jsr:@dune/core@^0.22"
  }
}
```

**Tradeoffs:**

- Upgrading core = edit `deno.json`, run `dune lockfile sync`, commit, deploy
- The entry script resolves its full module graph on each launch — this is why `duno.lock` management matters here (see [Lockfile as build artifact](#lockfile-as-build-artifact) below)
- Downgrading is a `deno.json` edit; no reinstall step

### Installed binary (version at install time)

```bash
deno install -g -n dune -A jsr:@dune/core@0.26.0/cli
dune serve
```

The module graph is resolved once at install time and cached. Subsequent launches read from that cache — no module graph resolution on startup, and `deno.lock` is not consulted for the core CLI itself.

**Tradeoffs:**

- Simpler systemd unit — just `ExecStart=dune serve`
- Upgrading core = re-run `deno install` with the new version tag on the server
- The lockfile still covers your site's plugin dependencies, but core's own graph is pinned in the binary cache rather than `deno.lock`

## Systemd units

### JSR-URL pattern

```ini
[Unit]
Description=Dune CMS — my-site
After=network.target

[Service]
Type=simple
User=www-data
WorkingDirectory=/srv/my-site
Environment=DUNE_ENV=production
ExecStart=deno run -A --config=deno.json jsr:@dune/core@0.26.0/cli serve --frozen
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

The `--frozen` flag tells Dune to treat a stale lockfile as a hard error rather than a warning. If the lockfile is incomplete (i.e. `dune lockfile sync` was not run after a version bump), the server refuses to start and prints:

```
  ✗ deno.lock is incomplete for the current deno.json.
    Run `dune lockfile sync` and commit the result before deploying.
```

This prevents silent drift on the server. See [Lockfile as build artifact](#lockfile-as-build-artifact) for how to keep the lockfile correct.

#### Optional: `ExecStartPre` pre-flight gate

Add a pre-flight `lockfile:check` before the server starts so systemd refuses to launch with a stale lockfile, and you get a human-readable diagnosis instead of a raw Deno error:

```ini
ExecStartPre=/usr/bin/deno run -A --no-lock --config=deno.json jsr:@dune/core@0.26.0/cli lockfile:check --root .
ExecStart=deno run -A --config=deno.json jsr:@dune/core@0.26.0/cli serve --frozen
```

The `--no-lock` flag on `ExecStartPre` is important: without it, Deno would write any missing entries to `deno.lock` while loading the entry script — defeating the purpose of the check. `--no-lock` disables automatic lockfile discovery for that process only, so `lockfile:check`'s own diagnostic output (the missing-entry list and `--upgrade` hints) still prints correctly. Unlike `--frozen`, it does not fail fast before the command runs.

Use `--frozen` (not `--no-lock`) on `ExecStart`: there you want the server to refuse to start if the lockfile is incomplete, not just report it.

### Installed binary pattern

```ini
[Unit]
Description=Dune CMS — my-site
After=network.target

[Service]
Type=simple
User=www-data
WorkingDirectory=/srv/my-site
Environment=DUNE_ENV=production
ExecStart=/home/www-data/.deno/bin/dune serve
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

No lockfile concerns for the core CLI itself. Your site's plugin dependencies are still covered by `deno.lock`.

## Lockfile as build artifact

When using the JSR-URL pattern, treat `deno.lock` as a build artifact — something you produce locally and commit to the repository, not something the server manages at runtime.

The workflow:

```
1. Bump @dune/core in deno.json  (or add a plugin)
2. dune lockfile sync             — resolves missing entries locally
3. git add deno.json deno.lock
4. git commit
5. git pull on the server + restart
```

`dune upgrade` and `dune add` run step 2 automatically. You still need to commit the result.

On the server, `dune serve --frozen` enforces this: it checks the lockfile at startup and exits if it looks incomplete. If you see the error, the fix is always the same — run `dune lockfile sync` locally, commit, and redeploy.

### Why this matters

Without a committed lockfile, Deno resolves missing entries itself when the server starts. This works, but it silently modifies `deno.lock` on the server's working tree — which then diverges from what's in git. On the next deploy, `git pull` may fail or silently overwrite the server-written changes. With `--frozen`, the server never writes to `deno.lock`; it only reads from what you committed.

### Checking the lockfile before deploying

```bash
# Verify the lockfile is complete for the current deno.json
dune lockfile check
```

Returns exit code 0 if complete, 1 if anything is missing. Safe to run in CI as a pre-deploy gate.

## Choosing between the two

| | JSR-URL | Installed binary |
|---|---|---|
| Version tracked in | `deno.json` | server install |
| Upgrade step | edit + `lockfile sync` + commit | `deno install -g` on server |
| Lockfile needed for core | yes | no (for core itself) |
| `--frozen` enforcement | recommended | not applicable for core |
| Systemd unit complexity | slightly more | simpler |

For most sites, the **JSR-URL pattern** is the better default: version changes are a git commit, and the lockfile-as-build-artifact model gives you strict reproducibility. The **installed binary** pattern suits operators who prefer to manage tool versions separately from application code, or who want the simplest possible systemd unit.
