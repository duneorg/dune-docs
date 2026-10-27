---
title: "Hang Watchdog"
published: true
visible: true
taxonomy:
  audience: [webmaster, developer]
  difficulty: [intermediate]
  topic: [deployment, operations]
metadata:
  description: "How Dune detects a blocked event loop, what it does about it, and how to keep it from killing legitimate work"
---

# Hang Watchdog

Dune serves every request on one thread. If a synchronous loop gets stuck on that thread (catastrophic regex backtracking, a conversion that never terminates), the server stops answering **every** URL at once, logs nothing, and stays that way until someone restarts it. A blocked thread can't rescue itself, so `dune dev` and `dune serve` run a small watchdog on a separate thread.

## What it does

The main thread sends the watchdog a heartbeat every second. If the heartbeats stop for longer than the limit, the watchdog:

1. writes a report to `<site>/.dune/hangs/hang-<time>-<pid>.txt`: the time, how long the thread was silent, and the **requests that were in flight**, which usually answers "what hung?" On macOS it also saves a native stack sample next to it;
2. prints the key lines to stderr;
3. kills the process with `SIGKILL`, so a supervisor sees a dead process instead of a zombie.

It kills rather than exits cleanly because a blocked thread can't run shutdown code. The dune dev wrapper exits with status 137 when this happens. **Nothing restarts the process for you**: use a supervisor for `dune serve` (see below). The newest 20 reports are kept. Path segments longer than 24 characters are cut in reports, so tokens in URLs don't end up in them.

> **Add `.dune/` to your `.gitignore`.** `dune new` doesn't write one, so in a site repo that doesn't already ignore `.dune/`, the reports in `.dune/hangs/` show up as untracked files and are easy to commit by accident.

## Limits

| Setting | Default | Notes |
|---|---|---|
| `DUNE_WATCHDOG_TIMEOUT_MS` | `15000` | Silence that counts as a stall once the server is up (minimum 1000). |
| `DUNE_WATCHDOG_STARTUP_TIMEOUT_MS` | `120000` | The limit until the server is listening. Long, so a large site's initial indexing can't turn a supervised process into a kill/restart loop. |
| `DUNE_WATCHDOG` | `exit` | `exit` reports and kills; `warn` reports only; `off` disables it. |

The event loop also logs `[dune] event loop was blocked for ~N.Ns` whenever it was held for over a second. That shows a creeping problem before it becomes a hang. On a 4,474-page site, startup indexing and the search build never triggered it.

## When it holds fire

- **Suspended processes.** A laptop going to sleep, Ctrl-Z, or `kill -STOP` freezes every thread. The watchdog notices a gap in its own ticks and starts over rather than treating the resume as a hang.
- **Debuggers.** Under `--inspect`, `--inspect-brk` or `--inspect-wait` the default becomes `warn`, because a breakpoint pauses only the main thread.

## When it can't tell

The watchdog can't tell an infinite loop from a long computation by looking at the thread, only whether the loop is making **progress**. If your own code (a job, a hook, a plugin) legitimately blocks for a long time, say so. Both functions come from `@dune/core/watchdog` and do nothing when no watchdog is running.

```ts
import { watchdogGrace, watchdogProgress } from "@dune/core/watchdog";

// A long synchronous loop: report progress per item.
for (const record of records) {
  convert(record);
  watchdogProgress(); // counts as a heartbeat; cheap, rate-limited
}

// One long native call that can't report progress: declare the window.
const release = watchdogGrace(60_000, "rebuild thumbnails");
try {
  rebuildThumbnails();
} finally {
  release();
}
```

- `watchdogProgress()` only protects loops that call it. A loop with no natural end that calls it on every iteration looks alive forever, so don't put it in one. Most indexing and compile loops are asynchronous and yield on their own, and need nothing.
- `watchdogGrace(ms, reason)` pauses the stall check for at most `ms` (never more than 10 minutes). It expires by itself, nests, and logs each use. Prefer `watchdogProgress()` where you can.

If a site's startup or a job is simply slow in a way you can't annotate, raise `DUNE_WATCHDOG_TIMEOUT_MS`, or use `DUNE_WATCHDOG=warn` and rely on the report.

## Restarting after a kill

- **`dune serve`:** run it under a supervisor that restarts on failure, for example systemd:

  ```ini
  [Service]
  ExecStart=/usr/bin/deno task serve
  Restart=on-failure
  RestartSec=2
  StartLimitIntervalSec=60
  StartLimitBurst=5
  ```

  `StartLimitBurst` stops a site that hangs on every start from restarting forever.
- **`dune dev`:** the wrapper exits 137 and does not restart anything. That is deliberate: a dev server that hangs is something to look at, not to loop on.

## What happens to child processes

Only the server process is killed. Its helper processes (the esbuild service) exit by themselves when the pipe to their parent closes, which was checked on a real `dune dev`. The wrapper that launched it exits non-zero.

## If it hangs anyway

Before restarting: run `dune ps` (a server that accepts connections but never answers shows as **NOT RESPONDING**), check the CPU of the process (`ps -o pcpu -p <pid>`), and take a stack sample (`sample <pid> 5` on macOS) if the watchdog is off.
