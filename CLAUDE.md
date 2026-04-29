# Science Fiction Website

## Project Overview
A marketing/company website for Science Fiction. Two-slide scroll-snap layout with word-by-word text animation effects, preceded by a full-screen intro animation. Self-contained single HTML file with no build step.

**Live site:** https://chrisbiron.github.io/sci-fi/
**Repo:** https://github.com/chrisbiron/sci-fi (this directory)

## File Structure
- `index.html` — Entire site: HTML, CSS, and JS in one self-contained file
- `fonts/Rhymes Text Medium.woff2` / `.woff` — Rhymes Text Medium (local copies required; `local()` font loading doesn't work for this font)
- `fonts/untitled-sans-regular.woff2` / `untitled-sans-medium.woff2` — Untitled Sans, embedded so visitors without it installed locally still see the correct typeface (`local()` is preferred when available, woff2 is the fallback)
- `eye.mp4` — eye video, default source for the dither background on slide 0
- `butterfly.mp4` — alternate butterfly video for the dither background; switchable via the dither panel "Source" dropdown
- `intro.webp` — animated WebP for intro logo sequence (510×254 native @2x, 180 frames @ 60fps, 3.0s, ~2.9 MB). Replaced the original 49 MB PNG sequence in `intro-frames/`. The PNGs are kept locally in `intro-frames/` (gitignored) as source for re-encoding but are NOT deployed.
- `og.png` — 1200×630 social preview image (referenced by `og:image` and `twitter:image`)

## Local-only files (gitignored)
- `intro-frames/` — source PNG frames for re-encoding `intro.webp`
- `eye2.mp4` — working/alternate eye video
- `fonts/*.otf` — Untitled Sans source files (NOT licensed for web distribution; only the woff2s ship)

## Deployment Workflow
This directory IS the `chrisbiron/sci-fi` GitHub repo. Static site on GitHub Pages from `main` branch — push to `main` triggers deploy automatically:
```bash
git add index.html        # or whatever changed
git commit -m "..."
git push
```

## Architecture
- `.slide` sections inside `.slides` (CSS scroll-snap, `y mandatory`, `position: fixed; inset: 0`). Order: butterfly bg (idx 0), intro copy (idx 1), team/roles (idx 2), press list (idx 3), then a `.loop-sentinel` that snaps scrollTop back to 0 for infinite loop.
- Each text slide has a `.slide-text` paragraph; `buildWords()` splits it into `<span class="word">` elements at page load
- Press slide (idx 3) uses `.press-list` of `<a class="press-item">` elements (whole publication+headline is one anchor); items animate in with the Hue Warp 2 effect via `animatePressList()`. After settling, hover transitions the link color to `--hue-color` with `hue-rotate(--hue-angle)` while siblings dim to 20% opacity (`:has(.press-item:hover) .press-item:not(:hover)`).
- `IntersectionObserver` triggers `animateIn()` when a slide enters the viewport (≥60% visible), resets when it leaves (<10%); also drives `animatePressList()`, `expandIntroWords()`, and the dither bg lifecycle.
- Navigation: scroll, arrow keys (↑↓←→), PageUp/PageDown, and Space. Click-to-advance is intentionally NOT wired up.
- Word animations are CSS keyframe animations assigned via `animation-name`; delay is set per-word via inline `animation-delay` to create stagger
- Comma pause: extra stagger delay added after words ending in `,`
- Blob effect is two-phase: CSS opacity fade (phase 1) → JS RAF resolves stroke+blur to zero after each word's `animationend` (phase 2)
- Dev controls (sidebar + dither panel + pause/replay) are currently hidden via `display: none !important` rules at the top of the CSS — markup and JS still present.

## Dither Background (slide 0)
- Two `<video>` sources: `dither-video-eye` (default) and `dither-video-butterfly`. Each has a preset of dither defaults applied via `applyDitherPreset()` when the source dropdown changes.
- **Eye preset:** `t3: 1.0`, `mouseBlur: 5.0`, `invert: true`. **Butterfly preset:** `t3: 0.55`, `mouseBlur: 3`, `invert: false`. Both share `zoom: 0.7`. Other params (cell, brightness, contrast, gamma, t1, t2, mouseBlurAmount) are shared.
- `invert` flips luminance (`l = 1 - l`) inside `adjustLum()` before brightness/contrast/gamma — needed for the eye source which is darker by default.
- **Reveal-in animation:** when slide 0 enters, `start()` snaps `ditherConfig.brightness` to `-1`, sets canvas opacity 1 immediately, plays the video, then tweens brightness from `-1 → target` over 2000ms (ease-out cubic). On the *first* run the tween is delayed by `START_DELAY_FIRST = 800ms` to sync with the intro logo exit + word settle; on every re-entry (loop / scroll-back) it starts immediately (`START_DELAY_RETURN = 0ms`). Source-switching via the dropdown does NOT trigger this animation.
- **Pink → black + hue-rotate sweep:** simultaneously with the brightness tween (delayed by `SHAPE_REVEAL_DELAY = 400ms` so shapes are visible before color sweeps), `tweenShapeColor()` animates `ditherConfig.shapeColor` from `SHAPE_COLOR_FROM = '#ff2bd6'` (pink) to `SHAPE_COLOR_TO = '#000000'` over `SHAPE_REVEAL_DURATION = 1000ms`. At the same time, the canvas's CSS `filter: hue-rotate()` transitions from `HUE_ROTATE_FROM = 180deg` → `0deg`. Both use cubic-bezier(0.87, 0, 0.13, 1) easing (matching Hue Warp 2 text). `cubicBezierEasing()` helper implements Newton's method for the JS shape color tween. `shapeColorRAF` and `shapeRevealTimer` are cancelled in `stop()`/`start()` (NOT in `fadeOut()`).
- `fadeOut(duration)` is called when leaving slide 0 (via the slide-0 scroll trigger or `doCollapse`); reverses the brightness tween to `-1` over 1000ms, fades `mouseBlurAmount` to 0 over 300ms, then fades canvas opacity to 0 and pauses the video.
- **Transparent canvas:** `render()` uses `bgCtx.clearRect()` (not a white `fillRect`) so the marketing site bg shows through where there are no shapes.
- **Out-of-video clipping:** `render()` clips dithered shapes to the video's drawn bounds (with a 1-cell inset to avoid edge artifacts). Off-cells render nothing (proximity bars previously drawn around the cursor were removed).
- `proximityRadius: 330` — kept as a config but cursor halo no longer renders grey bars.

## Intro Animation
Three permanent fixed elements (z-index 201, above the black overlay at z-index 200):
- `#intro-word-sci` — shows "Sci**ence**" in p1, collapses to "Sci" + moves left in p2
- `#intro-word-fi`  — shows "Fi**ction**" in p1, collapses to "Fi" + moves right in p2
- `#intro-logo-large` — `<img src="./intro.webp">` playing the animated WebP; exits upward + scales down at 3.6s

**Sequence:** `intro.webp` is 510×254 native @2x (displayed at 179×89 via CSS), 180 frames @ 60fps, 3.0s total, loop=1 (plays once). Re-encode from `intro-frames/*.png` with: `cd intro-frames && img2webp -loop 1 -d 17 -lossy -q 80 -m 6 $(ls frame_*.png | sort) -o ../intro.webp`. Constants in JS: `introSrc`, `introExitMs = 2400`, `wordsLeadMs = 200` (words enter this many ms before phase 2).

**Flow:**
1. Black screen (`#intro` overlay, `body.intro-active`)
2. t=300ms: WebP `src` is set on the visible `<img>` (preloaded earlier off-DOM), animation starts
3. t=2.5s (introExitMs - wordsLeadMs): `startWordEntry()` — "Science"/"Fiction" blur-fade in close together (60px tight gap, JS-measured: `tightEdgePx = (window.innerWidth - sciW - fiW - 60) / 2`), then chained settle:
   - settleDelay = wordsAppearDelay + animDuration + wordStagger + 350ms after entry
   - Step A: pull together by 15px each side (30px total) over 300ms ease-in-out-quart `cubic-bezier(0.76, 0, 0.24, 1)`
   - Step B: spread to final `expandedEdgePx()` (200px desktop / 24px mobile) over 500ms ease-in-out-quart
4. t=2.7s (introExitMs): `triggerPhase2()` — all in one RAF:
   - `#intro-logo-large.exit` → transform slides logo to top + scales to 0.23× (0.4s ease-in-out)
   - `#intro.p2` → black overlay fades out (1s ease)
   - `ditherBg.start()` triggers (eye reveal with `START_DELAY_FIRST = 800ms` then brightness ramp + pink→black sweep)
   - Real `.logo` fades in
   - `theme-color` meta tag is swapped from `#000000` → `#ffffff` so iOS Safari toolbars match the page background after the intro fade
5. After lockin (`lockinDelay = wordsAppearDelay + animDuration + wordStagger - wordsLeadMs`): `intro-active` removed, `startObserving()` fires, main text animation begins

**Key details:**
- Word entry is split from logo phase 2 via `wordsLeadMs` so word animation/settle can be tuned independently of the logo exit timing.
- `setProperty('transition', ..., 'important')` is required on `wordSci`/`wordFi` because `.intro-word-anim` sets `transition: none !important`.
- Mobile shows the expanded "Science"/"Fiction" words too (no longer desktop-only); `expandedEdgePx()` returns 24 on mobile, 200 desktop. CSS `.expanded` rule has a mobile override for `left`/`right`.
- `#intro-word-fi` is switched from `right` to `left` anchoring via JS measurement before animation starts, to prevent "Fi" jumping as "ction" collapses (used during the final collapse path).
- Both word elements: Untitled Sans Medium (500 weight).

## Typography
- **"Science Fiction®"** (first two words of slide 0): Rhymes Text Medium, 500 weight — applied via `.slide[data-index="0"] .slide-text .word:nth-child(-n+2)`
- **All other text:** Untitled Sans Regular (400) — system local font
- **Intro words:** Untitled Sans Medium (500) — system local font
- Font size: 32px desktop / 28px mobile (≤640px)
- Line height: 1.18, letter spacing: -0.01em
- `-webkit-font-smoothing: antialiased` for sharp rendering matching Figma
- `®` is wrapped in `<sup>` by `buildWords()`; font preload `<link>` is required because opacity:0 words suppress browser font loading

## SEO / Social
- `<meta name="description">`, `theme-color`, `<link rel="canonical">`, full Open Graph tags, and Twitter `summary_large_image` are in `<head>`. They reference `https://chrisbiron.github.io/sci-fi/` and `og.png` (1200×630).
- `<link rel="preload" as="image">` for `intro.webp` and `<link rel="preload" as="font">` for the woff2s.

## Intro Word Animation Perf
- `#intro-word-sci`/`#intro-word-fi` use `transform: translate3d(0, -50%, 0)` (not `translateY`) and `backface-visibility: hidden` to force a GPU compositor layer.
- `.intro-word-anim` adds `will-change: transform, filter, opacity, color` for the duration of the keyframe animation; the class is removed on completion so the will-change hint is transient.

## Animation Config Defaults
```js
effect: 'hue-stretch-y', duration: 500, stagger: 85, pauseAfterComma: 320,
easing: 'cubic-bezier(0.87, 0, 0.13, 1)', blur: 12, translate: 12,
strokeWidth: 10,
stretchAmount: 2.4, stretchBlur: 6,
stretchColor1: '#00ccff', stretchColor2: '#ff75e1', stretchFadeDuration: 1600,
hueColor: '#d52fc0', hueAngle: 250,
perLetter: false
```
Per-effect blur defaults: blob→3px, blur-up→12px, blur→12px

## Available Effects
`hue-stretch-y` (default), `stretch-y`, `blur-up`, `blur`, `blob`

## Mobile
- Side padding: 24px (desktop: 120px)
- `viewport-fit=cover` for edge-to-edge layout on iOS

## Key Gotchas
- `-webkit-text-stroke` shorthand is NOT animatable in CSS — blob effect drives `webkitTextStrokeWidth` via JS RAF instead
- `buildWords()` must `.trim()` raw innerHTML before splitting or leading whitespace creates blank word spans, breaking the `:nth-child` Rhymes Text selector
- `.slide-text` defaults to `opacity: 0`; revealed by `data-built` attribute after `buildWords()` to prevent flash on first scroll
- Blob pause/resume: RAF controller tracks `resolveStarts` timestamps and offsets them by the paused duration on resume
- `animation-fill-mode: both` conflict: after `animationend`, set `w.style.animationName = 'none'` before clearing inline color to prevent fill-mode re-locking
- `replace_all: true` is dangerous near CSS selectors — targeted edits only
- Intro word "Fi" uses left-anchoring (not right) to prevent jump during "ction" collapse
- `.otf` font files in `fonts/` are gitignored — only woff2 is licensed for web distribution
