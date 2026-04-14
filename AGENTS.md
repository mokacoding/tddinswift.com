# CLAUDE.md

## What this is

Marketing/landing site for the book *Test-Driven Development in Swift*, built with [Hugo](https://gohugo.io/).
Single Hugo site, no multi-language setup, deployed as a static site from `public/`.

## Commands

Refer to [README.md](README.md) for the commands to serve the site locally and build it for deployment.

There is no test suite, and no local build step beyond Hugo itself —
no npm tooling, no Tailwind build, no PostCSS.

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

Content lives at the repo root under `content/` (`_index.md` for the landing page, `errata.md`, etc.).
Front matter is YAML.

### Styling: Tailwind via the play CDN

Tailwind CSS is loaded at runtime via the play CDN
(`https://cdn.tailwindcss.com?plugins=typography`),
included as a `<script>` tag in `themes/blank-canvas/layouts/partials/head.html`.
The theme palette (colors, fonts) is configured in an inline `tailwind.config = {...}` in the same partial.
A small `<style media="print">` block lives there too, to revert the dark theme to ink-on-paper.

There is no local Tailwind build step.
No `tailwind.config.js`, no `postcss.config.js`, no `package.json`, no npm dependencies.

The CDN approach is intentional but has known production downsides
(FOUC on first paint, runtime JS dependency on a third-party CDN, larger payload, Tailwind v3 only).
See `TODO.md` for the eventual move to a proper build pipeline.

### Generated / ignored paths

- `public/` — Hugo build output, gitignored.
- `resources/` — Hugo asset cache, gitignored.
- `hugo_stats.json` — written by Hugo when stats are enabled, gitignored.
- `.hugo_build.lock` — Hugo's build lock.

## Conventions

- Front matter is **YAML** (`---` fenced), per the recent migration commit.
- Prefer editing the theme under `themes/blank-canvas/` for layout/style changes; only add files at the repo root `layouts/` if intentionally overriding the theme.

## Follow-ups

Check [`TODO.md`](TODO.md) at the start of a session for outstanding follow-ups, and append new tasks there rather than burying them in chat.
