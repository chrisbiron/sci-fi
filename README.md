# Science Fiction

Marketing site at https://sci-fi.com — static, no build step.

## Stack

Pure HTML/CSS/JS. Three files:

- **`index.html`** — markup, meta tags, JSON-LD
- **`style.css`** — all styles
- **`script.js`** — intro animation, dither bg, scroll-snap loop, theme toggle (loaded with `defer`)

Plus assets (`fonts/`, `intro-frames-webp/`, `eye.mp4`, `logo.svg`, `favicon.svg`, `apple-touch-icon.png`, `og.png`) and SEO files (`robots.txt`, `sitemap.xml`, `CNAME`).

## Local development

Any static server works. Two easy options:

```bash
# Quick — built into macOS / most Linux
python3 -m http.server 8000

# With live-reload
npx browser-sync start --server --files "*.html,*.css,*.js,*.mp4,*.webp" --no-notify --no-open
```

Then http://localhost:8000 (or http://localhost:3000 with browser-sync).

To test on your phone over the same Wi-Fi:

```bash
ipconfig getifaddr en0   # find Mac IP
# Then on phone Safari: http://<that-ip>:3000
```

For iOS Safari Web Inspector: phone connected via USB → Settings → Apps → Safari → Advanced → Web Inspector. Then Mac Safari → Develop menu → your iPhone.

## Deployment

There's no build step — just upload the repo contents to any static host and point `sci-fi.com` at it.

A few options that all work out of the box:

- **Vercel** — `vercel deploy` (or connect the repo via the dashboard). No config needed.
- **Netlify** — `netlify deploy --prod --dir .` (or drag-and-drop the folder, or connect the repo). Add `_redirects` later if you ever need them.
- **Cloudflare Pages** — connect the repo via the dashboard; build command empty, output dir `/`.
- **S3 + CloudFront / Fastly / any CDN** — sync the directory, set CloudFront default-root to `index.html`.
- **GitHub Pages** — would also work; needs a `CNAME` file containing `sci-fi.com` at the repo root.

### Recommended caching

Static hosts default to fairly long cache TTLs already, but for best performance the following headers are worth setting (Vercel/Netlify/Cloudflare all let you set them per-path):

- `intro-frames-webp/*` — `Cache-Control: public, max-age=31536000, immutable` (the frames never change individually)
- `fonts/*` — same
- `*.html` — `Cache-Control: public, max-age=600, must-revalidate` (so content updates land within ~10 min)

### DNS for sci-fi.com

Point the apex at the host's recommended target — usually a hostname (`cname.vercel-dns.com`, `<site>.netlify.app`, etc.) via an ALIAS/ANAME record, or A records the host provides.

## Things to know / tweak

### Where copy lives
- **Slide 1 text** (`Science Fiction™ is a software company...`) — `index.html` line ~56
- **Slide 2 text + Open roles button** — `index.html` lines ~62–66. The Open roles URL currently points at a Lapse Workable page (`https://jobs.workable.com/.../jobs-at-lapse`) — **swap this for the real Sci-Fi roles URL when ready.**
- **Press list** — `index.html` lines ~70–86. Each `<a class="press-item">` is one card.
- **Backers ticker** — `index.html` lines ~91–102. Note: backers are duplicated for the seamless infinite-scroll loop; keep the `aria-hidden` copies in sync.
- **Page title / description / OG** — `index.html` `<head>`.

### Where assets live
- **`og.png`** — 1200×630 social card. Update it when you change branding.
- **`apple-touch-icon.png`** — 180×180 white-on-black logo. Generate from `favicon.svg` with ImageMagick: `magick -background black -density 300 favicon.svg -resize 110x55 -gravity center -extent 180x180 apple-touch-icon.png` (after temporarily forcing white fill).
- **`intro-frames-webp/`** — 180 individual WebP frames for the intro logo animation. Source PNGs are in `intro-frames/` (gitignored) — re-encode with `cd intro-frames && for f in *.png; do magick "$f" -quality 80 "../intro-frames-webp/${f%.png}.webp"; done`.

### Things to know
- **Loop sentinel** — there's an empty `<section class="slide loop-sentinel">` at the end of `.slides`. When it scrolls into view ≥60%, JS resets `scrollTop = 0` and toggles `html.dark` for an alternating theme on each loop. It's *not* a regular slide and isn't part of the slide indexer.
- **Theme system** — light is default. `html.dark` flips CSS variables (`--color-bg`, `--color-text`, etc.). The whole thing is just a class toggle.
- **No build step / no dependencies** — everything is hand-written. If you introduce a build tool, watch out for the relative paths (`./style.css`, `./logo.svg`) and font preload `<link>`s in the `<head>`.
- **iOS quirks** — `100dvh` for slides (URL bar handling), `viewport-fit=cover`, `playsinline` + `muted` on the dither video, off-screen 1×1 video element (iOS won't play `display:none` video).
- **Safari blurry-text fix** — after each word's keyframe ends, JS sets `animation-name: none; filter: none; transform: none` to drop the GPU compositor layer. Don't remove this — text goes blurry on Safari without it.

### Code archive
A local-only git branch `archive/dev-controls-and-effects` holds an older version with a dev sidebar (effect/timing controls), dither panel UI, pause/replay buttons, and 5 alternative word effects. `git checkout archive/dev-controls-and-effects` to restore. Won't be needed for normal work.

## SEO

- `robots.txt` + `sitemap.xml` at the root.
- JSON-LD `Organization` schema in `<head>` for rich results.
- Canonical URL, Open Graph, Twitter card all set.
- No `theme-color` meta — Safari's translucent toolbar samples the page edge color, which gives a smoother dark-intro → light-page transition than a hard-coded color (same approach as Framer/born.com).

## Lighthouse

Last measured (Apr 2026):
- **Desktop**: 100 / 100 / 100 / 100 (Performance / Accessibility / Best Practices / SEO)
- **Mobile**: 96 / 100 / 100 / 100

The mobile -4 is LCP from the intro animation paint.

## Detailed architecture

See `CLAUDE.md` for a deeper architecture write-up — animation timing, IntersectionObserver flow, dither shader internals, all the gotchas. Written for AI assistants but useful for humans too.
