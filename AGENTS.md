# CLAUDE.md

## What this is

Marketing/landing site for the book *Test-Driven Development in Swift*, built with [Hugo](https://gohugo.io/).
Single Hugo site, no multi-language setup, deployed as a static site from `public/`.

## Commands

Hugo extended is required (the Tailwind pipeline below depends on it).
Refer to [README.md](/Users/gio/Developer/mokacoding-pty-ltd/tddinswift.com/README.md) for the list of commands used to install dependencies, serve the site, build it, and prepare it for deployment.

There is no test suite.
The `npm test` script is the npm-init placeholder and intentionally fails.

## Architecture

### Hugo + theme layout

The site uses a **local theme** at `themes/blank-canvas/`, selected via `theme = 'blank-canvas'` in `hugo.toml`.
The repo's top-level `layouts/` directory is empty — all templates live inside the theme.
When changing layouts, edit files under `themes/blank-canvas/layouts/`, not the root `layouts/`.

Key template entry points:

- `themes/blank-canvas/layouts/_default/baseof.html` — base shell (header, footer, `main` block).
- `themes/blank-canvas/layouts/index.html` — landing page (rendered for `content/_index.md`, which sets `type: landing`).
- `themes/blank-canvas/layouts/_default/{single,list,home}.html` — defaults for other page kinds.
- `themes/blank-canvas/layouts/partials/` — `head.html`, `header.html`, `footer.html`, `menu.html`, `terms.html`.
- `themes/blank-canvas/layouts/_partials/css.html` — Tailwind pipeline (see below).

Content lives at the repo root under `content/` (`_index.md` for the landing page, `errata.md`, etc.).
Front matter is YAML.

### Tailwind via Hugo's built-in pipeline

CSS is processed by Hugo's `css.TailwindCSS` function, **not** by running `tailwindcss` or `postcss` from npm scripts.
The flow:

1. `themes/blank-canvas/layouts/_partials/css.html` calls `resources.Get "css/main.css" | css.TailwindCSS`.
2. Hugo invokes the Tailwind CLI under the hood, minifies in production, and fingerprints the asset.
3. The source stylesheet is `themes/blank-canvas/assets/css/main.css`.

The root-level `tailwind.config.js` and `postcss.config.js` exist but the active pipeline is Hugo-driven.
Tailwind config currently points `content` at `./layouts/**/*.html` and `./content/**/*.{md,html}` —
note that the actual templates live under `themes/blank-canvas/layouts/`, so this config does not see them.
The recent "errata" commit flagged this as a known rough edge to clean up; keep it in mind when touching styles or class names.

`package.json` pulls in both `tailwindcss@^3` and `@tailwindcss/cli@^4`, plus `@tailwindcss/typography`.
Don't assume which major version is in effect without checking what Hugo's pipeline actually resolves.

### Generated / ignored paths

- `public/` — Hugo build output, gitignored.
- `resources/` — Hugo asset cache, currently untracked.
- `hugo_stats.json` — written by Hugo when stats are enabled, currently untracked.
- `.hugo_build.lock` — Hugo's build lock.

## Conventions

- Front matter is **YAML** (`---` fenced), per the recent migration commit.
- Prefer editing the theme under `themes/blank-canvas/` for layout/style changes; only add files at the repo root `layouts/` if intentionally overriding the theme.

## Follow-ups

Check [`TODO.md`](TODO.md) at the start of a session for outstanding follow-ups, and append new tasks there rather than burying them in chat.
