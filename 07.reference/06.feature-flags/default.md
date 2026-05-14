---
title: "Feature Flags"
published: true
visible: true
taxonomy:
  audience: [developer, webmaster]
  difficulty: [beginner]
  topic: [reference, configuration]
metadata:
  description: "Toggle features at runtime without code changes using Dune's built-in flag system"
---

# Feature Flags

Feature flags let you toggle features on or off without changing code or redeploying. Flags are declared in `site.yaml`, resolved once at startup, and read anywhere via the `flag()` helper.

## Configuration

```yaml
# site.yaml
flags:
  comments: true
  new_editor: false
  beta_search: "env:ENABLE_BETA_SEARCH"
```

Flag values can be:

| Value | Resolves to |
|-------|-------------|
| `true` / `false` | Boolean literal |
| `"env:VAR_NAME"` | `true` when the env var is `"1"`, `"true"`, or `"yes"`; `false` otherwise |

Flags are resolved at startup. Changing `site.yaml` while the server is running has no effect until restart. Environment variables are read at startup and cached — they are not re-read per request.

Unknown flags (not declared in `site.yaml`) return `false` from `flag()`. There is no error for reading an undeclared flag.

## Reading flags

Import `flag` from `@dune/core`:

```ts
import { flag } from "@dune/core";

if (flag("comments")) {
  // show comments UI
}
```

In TSX templates:

```tsx
import { flag } from "@dune/core";
import type { TemplateProps } from "@dune/core";

export default function Post({ page, Layout, ...props }: TemplateProps) {
  return (
    <Layout {...props} page={page}>
      <article dangerouslySetInnerHTML={{ __html: page.html }} />
      {flag("comments") && <CommentSection pageRoute={page.route} />}
    </Layout>
  );
}
```

In plugins:

```ts
import { flag } from "@dune/core";
import type { DunePlugin } from "@dune/core/plugins";

const plugin: DunePlugin = {
  name: "my-plugin",
  setup(hooks) {
    hooks.on("onRebuild", async () => {
      if (flag("beta_search")) {
        // run experimental search indexing
      }
    });
  },
};
```

## Listing all flags

```ts
import { allFlags } from "@dune/core";

const flags = allFlags(); // → { comments: true, new_editor: false, beta_search: false }
```

`allFlags()` returns a snapshot of all resolved boolean values. The MCP server's `get_config` tool also includes flag values in its response.

## Environment-based rollout

Use `"env:VAR_NAME"` to control flags via deployment environment:

```yaml
flags:
  new_checkout: "env:ENABLE_NEW_CHECKOUT"
```

```bash
# Enable on staging
ENABLE_NEW_CHECKOUT=true dune serve

# Disabled on production (env var absent → false)
dune serve
```

This pattern lets you deploy new code behind a flag and enable it per environment without touching committed config.
