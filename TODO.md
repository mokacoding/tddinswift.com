# TODO

Running list of things that need follow-up.
Agents: read this at the start of a session and append new tasks here rather than burying them in chat.

## Recover the Albertos demo gifs from wp.com

Two blog posts reference animated gifs that the wayback crawler never captured:

- `albertos-loading-demo.gif` — embedded in `content/posts/how-to-model-the-loading-state-with-remotedata.md` and in the `content/posts/albertos-loading-demo.md` attachment page.
- `albertos-retry-demo.gif` — embedded in `content/posts/chapter-8-testing-code-based-on-indirect-inputs.md` and in the `content/posts/albertos-retry-demo.md` attachment page.

Both figures are currently commented out (search the repo for `TODO(TODO.md): restore`).

Original WordPress upload paths:

- `https://tddinswift.com/wp-content/uploads/2021/07/albertos-loading-demo.gif`
- `https://tddinswift.com/wp-content/uploads/2021/07/albertos-retry-demo.gif`

Action: log in to the old `tddinswift.com` WordPress admin on wordpress.com, open the Media Library, and download both gifs.
Once recovered, drop them under `static/images/` (or equivalent), rewrite the four `<figure>` blocks above as clean markdown image syntax pointing at the local paths, and delete the TODO comments.

If the gifs are no longer in the WordPress Media Library either, check Time Machine backups and the iOS simulator recordings folder before giving up and re-recording them from the `test-driven-development-in-swift/13-testing-view-presentation/1-end/Albertos` example project.

## Experiment: darker background

The ported palette uses `#1f2527` for `--global--color-background`, matching the old WordPress site exactly.
It's fine, but `#1a1f21` (a couple of steps darker) may give the cyan and cream accents a touch more pop on modern OLED-era screens.

Try it as a one-line swap in whatever the palette ends up being defined in, compare side by side, and keep whichever looks better in a real browser (not just a design tool).

## Experiment: Inter for body text

The site currently uses the old WordPress pairing: Playfair Display for headings, Fira Sans for body — Fira being the original from the WordPress "Blank Canvas" theme.

Inter (by Rasmus Andersson) is the "boring default" of modern product UIs: wider x-height, tighter default spacing, more neutral than Fira.
Pairing Playfair + Inter gives one voice (Playfair) and a quiet frame around it, versus Playfair + Fira which gives two mildly-opinionated voices.

Try swapping body font from Fira Sans to Inter in the Tailwind font config, reload the landing page and a long post, and decide whether the quieter pairing reads better.
Flipping back is one line if it doesn't.
