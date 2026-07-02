---
title: "Theme Preview & Marketplace"
published: true
visible: true
taxonomy:
  audience: [webmaster]
  difficulty: [beginner]
  topic: [themes, admin]
metadata:
  description: "Preview themes before switching and install new themes from the admin marketplace"
---

# Theme Preview & Marketplace

The admin panel lets you preview what your site looks like with a different theme before committing the switch, and install new themes directly from a curated registry.

## Previewing a theme

The Theme tab of the [configuration editor](../administration) (`/admin/config` → Theme tab) has a **Preview** button next to the theme dropdown.

1. Select a theme from the dropdown
2. Click **Preview** — an iframe opens showing your site rendered in that theme (without switching)
3. Use the route picker to preview different pages, or type a custom path
4. Click **↻ Refresh** if the preview needs reloading
5. Click **Apply this theme** to commit the switch, or **× Close** to dismiss without changing anything

The preview renders a real page through the selected theme's templates and layout — what you see is what you get. The active theme is not changed until you click Apply.

> **TSX pages:** Pages that are self-rendering TSX components (`.tsx` content files) cannot be fully previewed this way. They render themselves and can't have their layout swapped from the server. The preview will show a notice for these pages — check a Markdown page instead.

## Theme marketplace

The marketplace is at `/admin/themes` (also reachable from the "Browse more themes →" link at the bottom of the Theme tab).

### Installed themes

The **Installed** section shows every theme in your `themes/` directory:

- The currently active theme is marked with an **Active** badge
- **Preview** opens the config editor with that theme pre-selected in the preview panel
- **Set Active** switches the active theme (same as the Theme tab dropdown)

### Available from registry

The **Available** section lists themes from the bundled registry:

- Each card shows the name, description, tags, author, and version
- **Demo** links open the theme's demo site in a new tab
- **Install** registers the theme on your site. Registry entries may use either delivery method:
  - **`jsr`** (preferred) — adds the theme to `themes:` in `site.yaml`, updates `deno.json`, and resolves the package from JSR (same as `dune theme:install`)
  - **`downloadUrl`** — downloads a ZIP and extracts it to `themes/{slug}/` on disk (legacy path)
- Once installed, the theme appears in the config editor dropdown. Package themes may need a server restart (or `dune lockfile:sync` if `deno.lock` changed)

After installation you can immediately preview the new theme using the Preview button, then switch to it when ready.

### The bundled registry

Dune ships with a curated `src/admin/registry/themes.json` that lists community themes. The registry is updated with each Dune release. Support for custom registry URLs is planned for a future release.

The registry API is available at `GET /admin/api/registry/themes` for programmatic access (admin auth required).

### Install security

The install endpoint (`POST /admin/api/themes/install`) enforces:

- Theme slugs must match `/^[a-z0-9][a-z0-9_-]*$/` — no path traversal via the slug
- Themes are resolved **only** from the bundled registry (`slug` lookup) — caller-supplied URLs are not accepted
- **JSR installs:** `jsr` in the registry entry must be pinned to an exact semver; writes `themes:` + `deno.json`
- **ZIP installs:** `downloadUrl` is fetched via SSRF-hardened `safeFetch`; optional `sha256` integrity check
- ZIP entries with path traversal components (`../`) are rejected; files are written to `themes/{slug}/` only
