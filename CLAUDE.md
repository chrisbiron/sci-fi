# Science Fiction Website

## Project Overview
A marketing/company website for Science Fiction. Scroll-snap layout with word-by-word text animation, preceded by a full-screen intro animation. Self-contained single HTML file, no build step.

**Live site:** https://sci-fi.com/
**Repo:** https://github.com/chrisbiron/sci-fi (this directory)

**Code archive:** the local `archive/dev-controls-and-effects` git branch holds the previous version with dev sidebar (effect/timing controls), dither panel UI, pause/replay buttons, butterfly source, and 5 alternative word effects (`blur`, `blur-up`, `blob`, `stretch-y`, `hue-stretch-y` Hue Warp 1). `git checkout archive/dev-controls-and-effects` to restore.

## File Structure
- `index.html` — HTML markup, head meta tags + JSON-LD, references `./style.css` and `./script.js` (defer)
- `style.css` — All styles (fonts, theme vars, slides, intro, dither, backers ticker)
- `script.js` — All behavior (intro animation, dither engine, IO observer, loop, theme toggle)
- `robots.txt`, `sitemap.xml` — SEO scaffolding
- `apple-touch-icon.png` — 180×180 white-on-black logo for iOS home-screen install
- `fonts/Rhymes Text Medium.woff2` / `.woff` — Rhymes Text Medium (local copies required; `local()` font loading doesn't work for this font)
- `fonts/untitled-sans-regular.woff2` / `untitled-sans-medium.woff2` — Untitled Sans, embedded so visitors without it installed locally still see the correct typeface
- `eye.mp4` — eye video, source for the dither background on slide 0
- `intro-frames-webp/` — 180 individual WebP frames (`frame_000.webp` … `frame_179.webp`) played as an image sequence on a `<canvas>` for the intro logo animation. Replaced an animated WebP (which decoded slowly on Safari).
- `logo.svg` — vector logo that swaps in for the canvas after the intro frame sequence completes (sharp at any scale). In dark mode it's CSS-inverted to white.
- `og.png` — 1200×630 social preview image (referenced by `og:image` and `twitter:image`)

## Local-only files (gitignored)
- `intro-frames/` — source PNG frames (used to re-encode the WebPs in `intro-frames-webp/` if you ever need to)
- `eye2.mp4` — working/alternate eye video
- `fonts/*.otf` — Untitled Sans source files (NOT licensed for web distribution; only the woff2s ship)

## Deployment Workflow
This directory IS the `chrisbiron/sci-fi` GitHub repo. Static site on GitHub Pages from `main` branch — push to `main` triggers deploy automatically.

## Architecture
- `.slide` sections inside `.slides` (CSS scroll-snap, `y mandatory`, `position: fixed; inset: 0`). Order: dither bg + intro words (idx 0), intro copy (idx 1), team/roles (idx 2), press list (idx 3), then a `.loop-sentinel` that snaps scrollTop back to 0 for infinite loop.
- Each slide uses `height: 100dvh` so iOS Safari URL bar doesn't clip content.
- Each text slide has a `.slide-text` paragraph; `buildWords()` splits it into `<span class="word">` elements.
- Press slide (idx 3) uses `.press-list` of `<a class="press-item">` elements; items animate in with the Hue Warp 2 effect via `animatePressList()`. After settling, hover transitions the link color to `--hue-color` while siblings dim to 20% opacity.
- `IntersectionObserver` triggers `animateIn()` when a slide enters the viewport (≥60% visible), resets when it leaves (<10%); also drives `animatePressList()`, `expandIntroWords()`, and the dither bg lifecycle.
- Navigation: scroll, arrow keys (↑↓←→), PageUp/PageDown, and Space.
- Word animations are CSS keyframe animations; delay is set per-word via inline `animation-delay` to create stagger. Comma pause: extra stagger delay added after words ending in `,`.
- After each word's `animationend`, JS sets `animationName: none` + `filter: none` + `transform: none` to drop the GPU compositor layer Safari otherwise keeps active (which causes blurry text rasterization). Same treatment on the `.slide-btn`.

## Word Effect (single)
The only word effect is **Hue Warp 2** (`@keyframes word-hue-stretch-y-2`): scaleY stretch + blur + hue-rotate in, hue-color → text color, all in pure CSS. Driven by CSS custom properties set in `applyConfig()`: `--duration`, `--easing`, `--stretch-amount`, `--stretch-blur`, `--hue-color`, `--hue-angle`. Defaults in `config`:
```js
duration: 360, stagger: 85, pauseAfterComma: 320,
easing: 'cubic-bezier(0.87, 0, 0.13, 1)',
stretchAmount: 2, stretchBlur: 5,
hueColor: 'color(display-p3 0.992 0.263 1)', hueAngle: 180
```

## Dither Background (slide 0)
- Single `<video id="dither-video-eye">` source (kept off-screen + 1×1 + opacity 0; iOS Safari refuses to play `display:none` videos).
- `ditherConfig` defaults (eye preset baked in): `cell: 7, contrast: 1.85, gamma: 1.2, t1: 0.98, t2: 0.71, t3: 1.0, mouseBlur: 5.0, mouseBlurAmount: 1.0, zoom: 0.7, invert: true`. `invert` flips luminance (`l = 1 - l`) inside `adjustLum()` before brightness/contrast/gamma — needed because the eye source is dark by default.
- **Reveal-in animation:** when slide 0 enters, `start()` snaps brightness to `BRIGHTNESS_FROM = -1`, sets canvas opacity 1, plays the video, then tweens brightness to `BRIGHTNESS_TARGET = 0` over 2000ms (ease-out cubic). On the *first* run the tween is delayed by `START_DELAY_FIRST = 800ms` to sync with intro logo exit + word settle; on re-entry (loop / scroll-back) it starts immediately (`START_DELAY_RETURN = 0ms`).
- **Pink → black + hue-rotate sweep:** delayed by `SHAPE_REVEAL_DELAY = 400ms` so shapes are visible before color sweeps. `tweenShapeColor()` animates `ditherConfig.shapeColor` from `SHAPE_COLOR_FROM = '#ff2bd6'` → `SHAPE_COLOR_TO = '#000000'` over `SHAPE_REVEAL_DURATION = 1000ms`. Canvas CSS `filter: hue-rotate()` transitions from `HUE_ROTATE_FROM = -100deg` → `0deg` simultaneously. Both use `cubic-bezier(0.87, 0, 0.13, 1)` (matching Hue Warp 2).
- `fadeOut(duration)` reverses brightness to `-1` over 1000ms, fades `mouseBlurAmount` to 0 over `MOUSE_BLUR_OUT_DURATION = 300ms`, then fades canvas opacity to 0 and pauses the video.
- **Transparent canvas:** `render()` uses `bgCtx.clearRect()` so site bg shows through where there are no shapes.
- `render()` clips dithered shapes to the video's drawn bounds (with a 1-cell inset to avoid edge artifacts).

## Intro Animation
Three permanent fixed elements (z-index 201, above the black `#intro` overlay at z-index 200):
- `#intro-word-sci` — shows "Sci**ence**" in p1, collapses to "Sci" + moves left in p2
- `#intro-word-fi`  — shows "Fi**ction**" in p1, collapses to "Fi" + moves right in p2
- `#intro-logo-large` — wrapper containing a `<canvas id="intro-canvas">` (frame-by-frame WebP playback) and a `<img id="intro-logo-svg" src="./logo.svg">` (hidden until the sequence completes). Exits upward + scales down at phase 2.

**Sequence:** 180 frames × ~15KB each, displayed at 179×89 CSS via canvas (oversampled by DPR for sharp edges). All frames preload in parallel, then a single RAF loop swaps which one is `drawImage`'d at 60fps. At the final frame, the canvas is hidden and `#intro-logo-svg` (logo.svg) takes over so the persisting logo is vector-sharp at any scale. Constants in JS: `INTRO_FRAME_COUNT = 180`, `INTRO_FRAME_MS = 1000/60`, `introExitMs = 2400`, `wordsLeadMs = 200`.

**Flow:**
1. Black screen (`#intro` overlay, `body.intro-active`)
2. t=300ms: WebP `src` set on the visible `<img>`, animation starts
3. t=2.5s (introExitMs - wordsLeadMs): `startWordEntry()` — "Science"/"Fiction" blur-fade in close together, then chained settle (pull together 30px, then spread to `expandedEdgePx()` final position)
4. t=2.7s (introExitMs): `triggerPhase2()` — logo exits up + scales down (0.4s), `#intro.p2` overlay fades out (1s), `ditherBg.start()` triggers, real `.logo` fades in
5. After lockin: `intro-active` removed, `startObserving()` fires, main text animation begins
6. iOS Safari toolbar: no `theme-color` meta is set — Safari's translucent toolbar samples page edge color, automatically tinting dark during the intro and light after (same approach as born.com / Framer sites).

**Key details:**
- `setProperty('transition', ..., 'important')` is required on `wordSci`/`wordFi` because `.intro-word-anim` sets `transition: none !important`.
- `expandedEdgePx()` returns 24 on mobile, 200 desktop. CSS `.expanded` rule has a mobile override.
- `#intro-word-fi` is switched from `right` to `left` anchoring before final collapse to prevent "Fi" jumping as "ction" collapses.
- Both word elements: Untitled Sans Medium (500).
- `.intro-word-anim` adds `will-change: transform, filter, opacity, color` for animation duration; class is removed on completion so the hint is transient.

## Typography
- **"Science Fiction™"** (first two words of slide 0): Rhymes Text Medium, 500 — applied via `.slide[data-index="0"] .slide-text .word:nth-child(-n+2)`
- **All other text:** Untitled Sans Regular (400)
- **Intro words:** Untitled Sans Medium (500)
- Font size: 32px desktop / 28px mobile (≤640px)
- Line height: 1.18, letter spacing: -0.01em
- `™` is wrapped in `<sup>` by `buildWords()`; font preload `<link>` is required because opacity:0 words suppress browser font loading

## SEO / Social
- `<meta name="description">`, `<link rel="canonical">`, full Open Graph tags, and Twitter `summary_large_image` in `<head>` (no `theme-color` — see Intro Animation section). They reference `https://sci-fi.com/` and `og.png` (1200×630).
- `<link rel="preload" as="font">` for the woff2s. Intro frames preload via JS `new Image()` calls.

## Mobile
- Side padding: 24px (desktop: 120px)
- `viewport-fit=cover` for edge-to-edge layout on iOS
- Slides use `100dvh` (dynamic viewport height) to track Safari URL bar show/hide

## Key Gotchas
- `buildWords()` must `.trim()` raw innerHTML before splitting or leading whitespace creates blank word spans, breaking the `:nth-child` Rhymes Text selector
- `.slide-text` defaults to `opacity: 0`; revealed by `data-built` attribute after `buildWords()` to prevent flash on first scroll
- `animation-fill-mode: both` conflict: keyframe end-state values win against inline styles in the cascade. Must set `w.style.animationName = 'none'` before clearing/overriding `filter`/`transform`/etc.
- Safari blur-after-animation: `filter: blur(0)` and `transform: scaleY(1)` keep the word on a GPU compositor layer that rasterizes text through a sub-pixel filter buffer (looks blurry). After `animationend`, set `animationName/filter/transform` to `'none'` so the layer is destroyed.
- Intro word "Fi" uses left-anchoring (not right) to prevent jump during "ction" collapse
- `.otf` font files in `fonts/` are gitignored — only woff2 is licensed for web distribution
- Test changes locally on phone: `python3 -m http.server 8000` from project root, then `http://<mac-ip>:8000` on iPhone Safari (same Wi-Fi). For DevTools: USB cable + iOS Settings → Apps → Safari → Advanced → Web Inspector, then Mac Safari → Develop menu → iPhone.
