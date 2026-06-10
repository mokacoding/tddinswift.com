# Open Graph share card

How the site's social-share preview image (the "OG image") is wired, and how to
swap in a different one.

## What this is

`og:image` is the picture that appears when a `tddinswift.com` link is shared on
X, LinkedIn, Slack, Facebook, or iMessage.
It is set once, in `themes/blank-canvas/layouts/partials/head.html`:

```html
<meta property="og:image" content="https://tddinswift.com/images/og-card.png" />
```

Two non-obvious requirements:

- The URL **must be absolute** (`https://tddinswift.com/...`).
  A relative `/images/...` path silently fails to preview on most scrapers.
  This is why `og:image` differs from the on-page cover, which is relative.
- The canonical card size is **1200 × 630 px**, landscape (the
  `summary_large_image` / 1.91:1 standard).

## Current state

`og:image` points at `static/images/og-card.png` — the official Apress/Springer
promo card (1200 × 628, landscape): book cover, title, author, and publisher
branding. No letterboxing.

The Canva guidance below is kept in case you ever want a custom card matching the
site's own dark/cyan/gold identity instead of the publisher template.

## Swapping in a different card

Design a landscape card featuring the cover thumbnail + the title + a short
tagline, then:

1. Export it as **PNG** at 1200 × 630.
2. Drop it in `static/images/` (e.g. `static/images/og-card.png`).
3. Repoint the `og:image` line in `head.html` at
   `https://tddinswift.com/images/og-card.png`.
4. Re-share a link (or use a scraper debugger) to confirm the new card renders.

## Making the card in Canva (free tier)

**Best free path: start a "Blog Banner" design** (≈2240 × 1260, the same 1.91:1
ratio as OG), pick a dark/minimal template, customize, export, and downscale to
1200 × 630.
Category page: <https://www.canva.com/blog-banners/templates/>.

**Equally good: a custom-size design from scratch** — Canva Home → Custom size →
`1200 × 630 px` → Create new design.
Pixel-perfect, zero crop risk, and the layout here is simple enough (flat dark
fill + uploaded cover + two text blocks) that a blank canvas is often faster than
fighting a template's locked groups.

### Free-tier gotchas

- **Canva's "Resize" button is Pro-only.**
  A free account cannot re-dimension an existing design, so you must create it at
  the right size from the start — pick an already-landscape category, or use
  Custom size.
- **"Free template" does not mean every element is free.**
  Individual photos, fonts, and graphics can carry a Pro crown even inside a free
  template.
  Easy to dodge here: the card is text + your own uploaded cover PNG on a flat
  fill, so it depends on no Pro stock.
  Check for the small crown badge before exporting.

### Other free categories (fallbacks)

- **X/Twitter Post** (1600 × 900, 16:9) — built for this sharing context, slightly
  taller crop than OG: <https://www.canva.com/twitter/templates/posts/>
- **Web Banner** — clean product-launch layouts:
  <https://www.canva.com/web-banners/templates/>
- **Author / book-promo** templates — right vibe, but mostly portrait/square; mine
  them for styling ideas only, do not ship one directly (recreates the
  letterboxing): <https://www.canva.com/templates/s/author/>

Note: Canva gates per-template URLs (they 403/404 to fetchers and often sit behind
login), so the links above are category pages, not deep template links.
Confirm free/Pro per template via the crown badge in the editor.

## Design guidance (match the site identity)

Palette: dark background `#1f2527`, cyan accent `#9fd3e8`, cream/gold accent
`#fbe6aa`.
Fonts: serif display (Lora) + clean sans body (Fira Sans).

- **Solid `#1f2527` background; use cyan/gold as sparse accents, not fills.**
  Title in the serif, with the key phrase in cyan and a small "2nd Edition ·
  Apress" tag in gold.
  A thin cyan rule echoes the tech feel without clutter.
  **Lora is free in Canva**; if Playfair Display shows a crown, use Lora.
  Fira Sans may not be free in Canva — substitute Inter / Work Sans / Source Sans
  for body if so.
- **Composition: cover-right, text-left, generous margins.**
  Upload `cover.png` to the right third with a soft shadow or faint cyan glow;
  reserve the left two-thirds for title + tagline.
  Keep ~60–80 px safe padding — iMessage and Slack round corners and crop edges;
  keep text out of the outer 5%.
- **Optional subtle dev texture.**
  A faint `>` prompt glyph, a thin dotted grid, or a low-opacity monospace snippet
  at ~5–8% cyan reinforces the Swift/TDD identity.
  Keep the dark calm — do not fill the negative space.

Export PNG (not JPG) to keep the cover and cyan edges crisp; the file stays well
under social platforms' ~5 MB limit.
