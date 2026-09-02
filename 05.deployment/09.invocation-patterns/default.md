---
title: "Invocation Patterns"
published: true
visible: true
taxonomy:
  audience: [webmaster]
  difficulty: [intermediate]
  topic: [deployment, server, lockfile]
metadata:
  description: "How to invoke Dune in production — the main.ts entrypoint pattern, the legacy JSR-URL/installed-binary patterns, and the lockfile-as-build-artifact model"
---

# Invocation Patterns

## The main.ts entrypoint (recommended)

Every site `dune new` scaffolds includes a one-line `main.ts`:

```ts
import { cli } from "@dune/core/cli";

await cli({ root: import.meta.dirname });
```

`deno.json`'s `dev`/`build`/`serve` tasks invoke it directly:

```json
{
  "tasks": {
    "dev": "deno run -A --watch=main.ts main.ts dev",
    "build": "deno run -A main.ts build",
    "serve": "deno run -A main.ts serve"
  }
}
```

`main.ts` is your site's own script — Deno resolves and runs it directly, with your site's own `deno.json`/`deno.lock` governing the process natively. There is no re-exec: no second process gets spawned, no config gets synthesized. This is deliberate. The two patterns below (JSR-URL and installed binary) both work by re-executing into a second `deno run` under the hood so the right import map is in scope; that indirection is where several subtle bugs used to live (a stale-config re-exec silently dirtying `deno.lock`, Deno-level flags getting dropped across the re-exec boundary). The entrypoint pattern has no re-exec to get wrong.

**Existing sites**: run `dune migrate:entrypoint` to move onto this pattern — it writes `main.ts`, adds any import-map entries dune-core needs that your site doesn't already declare, and rewrites your tasks. It's idempotent and refuses to touch a `main.ts` you've hand-edited. Run `dune validate` afterward and soak-test before rolling it out to more than one site.

```ini
# systemd unit — main.ts pattern
[Unit]
Description=Dune CMS — my-site
After=network.target

[Service]
Type=simple
User=www-data
WorkingDirectory=/srv/my-site
Environment=DUNE_ENV=production
ExecStart=deno run -A main.ts serve
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### Lockfile enforcement (`--frozen`) — status

`serve` accepts `--frozen` (or `DUNE_FROZEN=1`) to make Deno refuse to start if `deno.lock` is missing entries — the same enforcement described below for the legacy JSR-URL pattern. **`--frozen` is not the default for `serve`** (an earlier release briefly made it the default and that was reverted — see the CHANGELOG). A live regression surfaced: Deno's `--frozen` validation, at least as observed on Deno 2.7.14, can refuse to boot against a lockfile containing entries only reachable through the built-in admin plugin's dynamic import — even a lockfile `dune lockfile:sync` reports as complete. Whether this reproduces under the main.ts pattern specifically is still being verified; treat `--frozen` on `serve` as something to test against your own site's plugin set before relying on it, not as a default guarantee.

`dune lockfile:sync`/`dune lockfile:check` are unaffected by this — they use their own discovery-based subprocess resolution, not Deno's `--frozen` flag, and remain the reliable way to keep `deno.lock` complete and verify it before deploying (see [Lockfile as build artifact](#lockfile-as-build-artifact) below, which applies regardless of invocation pattern).

## Legacy patterns

These predate the `main.ts` entrypoint and still work — sites that haven't migrated yet use one of them. Both work by re-executing into a second process so the right import map is in scope for site-specific code (theme TSX, `config.ts`).

### JSR-URL (version in deno.json)

```bash
deno run -A --config=deno.json jsr:@dune/core@0.29.0/cli serve
```

The version is pinned in `deno.json` alongside your site's other imports. Upgrading core is a one-line edit to that file.

```json
{
  "imports": {
    "@dune/core": "jsr:@dune/core@^0.29"
  }
}
```

**Tradeoffs:**

- Upgrading core = edit `deno.json`, run `dune lockfile:sync`, commit, deploy
- The entry script resolves its full module graph on each launch — this is why `deno.lock` management matters here
- Downgrading is a `deno.json` edit; no reinstall step

### Installed binary (version at install time)

```bash
deno install -g -n dune -A jsr:@dune/core@0.29.0/cli
dune serve
```

The module graph is resolved once at install time and cached. Subsequent launches read from that cache — no module graph resolution on startup, and `deno.lock` is not consulted for the core CLI itself.

**Tradeoffs:**

- Simpler systemd unit — just `ExecStart=dune serve`
- Upgrading core = re-run `deno install` with the new version tag on the server
- The lockfile still covers your site's plugin dependencies, but core's own graph is pinned in the binary cache rather than `deno.lock`

### Systemd units — legacy patterns

#### JSR-URL pattern

```ini
[Unit]
Description=Dune CMS — my-site
After=network.target

[Service]
Type=simple
User=www-data
WorkingDirectory=/srv/my-site
Environment=DUNE_ENV=production
ExecStart=deno run -A --config=deno.json jsr:@dune/core@0.29.0/cli serve --frozen
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

**Flag position matters here in a way it doesn't for `main.ts`.** `--frozen` after the script specifier (as shown in the `dune serve --frozen`-style examples elsewhere in these docs) is parsed by Dune's own argument parser and threaded into the re-exec'd child process — it does *not* apply to the outer `deno run` process's own resolution of the entry script. If you want Deno-level enforcement on the outer process too, it must come before the script specifier as a genuine Deno CLI flag: `deno run -A --frozen --config=deno.json jsr:@dune/core@0.29.0/cli serve`. This is exactly the kind of re-exec subtlety the `main.ts` pattern above avoids by construction — there's only one process, so there's no "which half of the invocation did the flag apply to" question.

#### Optional: `ExecStartPre` pre-flight gate

```ini
ExecStartPre=/usr/bin/deno run -A --no-lock --config=deno.json jsr:@dune/core@0.29.0/cli lockfile:check --root .
ExecStart=deno run -A --config=deno.json jsr:@dune/core@0.29.0/cli serve --frozen
```

The `--no-lock` flag on `ExecStartPre` is important: without it, Deno would write any missing entries to `deno.lock` while loading the entry script — defeating the purpose of the check. `--no-lock` disables automatic lockfile discovery for that process only, so `lockfile:check`'s own diagnostic output (the missing-entry list and `--upgrade` hints) still prints correctly.

#### Installed binary pattern

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

Regardless of invocation pattern, treat `deno.lock` as a build artifact — something you produce locally and commit to the repository, not something the server manages at runtime.

The workflow:

```
1. Bump @dune/core in deno.json  (or add a plugin)
2. dune lockfile:sync             — resolves missing entries locally
3. git add deno.json deno.lock
4. git commit
5. git pull on the server + restart
```

`dune upgrade` and `dune add` run step 2 automatically. You still need to commit the result. Never run `dune lockfile:sync` on the server as part of a deploy — resolution decisions belong in dev/CI, reviewed and committed; the server should only ever verify (`dune lockfile:check`), never resolve fresh dependency versions at deploy time.

### Why this matters

Without a committed lockfile, Deno resolves missing entries itself when a process starts — silently modifying `deno.lock` on the server's working tree, which then diverges from what's in git. On the next deploy, `git pull` may fail or silently overwrite the server-written changes.

### Checking the lockfile before deploying

```bash
dune lockfile:check
```

Returns exit code 0 if complete, 1 if anything is missing. Safe to run in CI as a pre-deploy gate.

## Choosing an invocation pattern

For a new site, use the `main.ts` entrypoint pattern — it's what `dune new` scaffolds, and it removes a whole class of re-exec-related footguns by construction. For an existing site, `dune migrate:entrypoint` gets you there; migrate one low-stakes site first and soak-test before doing the rest of a fleet.

| | main.ts entrypoint | JSR-URL (legacy) | Installed binary (legacy) |
|---|---|---|---|
| Re-exec involved | no | yes | yes |
| Version tracked in | `deno.json` | `deno.json` | server install |
| Upgrade step | edit + `lockfile:sync` + commit | edit + `lockfile:sync` + commit | `deno install -g` on server |
| Flag-position footguns | none | yes (see above) | n/a |
| Systemd unit complexity | simplest | more | simple |
