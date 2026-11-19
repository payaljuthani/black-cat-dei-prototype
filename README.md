# DEI Dirty Dozen — scroll prototype

Single static HTML file, no build step. Open `scroll_flow_prototype.html` directly or serve the folder with any static file server — paths are relative, so the folder structure here must stay as-is (`assets/brand`, `assets/logos/target.svg` + `assets/logos/cropped/*.png`, `assets/storyboard`).

## External dependencies (CDN, not vendored)

- GSAP 3.12.5 core + ScrollTrigger — `cdn.jsdelivr.net/npm/gsap@3.12.5/...`
- Google Fonts: Inclusive Sans (400/500/700), Passion One (400/700)

Versions are pinned in the `<script>`/`<link>` tags. Consider self-hosting both for production rather than relying on the CDNs long-term.

## Layout

One universal, scroll-scrubbed GSAP timeline per pinned sequence — not separate desktop/mobile code paths. Breakpoints only change CSS layout (grid vs stacked columns, card sizing, the company ticker) via media queries at 900px, 699px and 479px; the JS driving the animations is identical at every width.

## Things that look fragile but aren't — don't "simplify" them

- **`document.fonts.ready` gate before the sequence setup runs** (near the bottom of the script): several measurements (caption box height, card travel distance) depend on real text metrics. Google Fonts loads with `font-display:swap`, so on a slow/cold load the fallback font is still showing when the script would otherwise measure — wrong numbers get baked into the timeline for the rest of the session. This wait fixes that; removing it reintroduces a layout bug that's invisible in local dev (fonts are cached) but real on a first production load.
- **Company ticker's active-name position is computed analytically** (`cardsLayerRect.left + (cardsLayerRect.width - cardWidth)/2`), not read directly off the card element. The card 3D-rotates during its flip; reading its live bounding box while mid-rotation gives a distorted position. `#cardsLayer` never transforms, so it's used as the stable reference instead.

## Testing scroll-driven behavior with scripted scroll (automated tests, not real user scrolling)

`window.scrollTo()` alone doesn't reliably fire the scroll event ScrollTrigger listens for. Call `ScrollTrigger.update()` right after, and wait ~1–1.5s before reading computed positions — `scrub:1` has its own inertia, so values read immediately after a big programmatic jump can be stale. This is a test-methodology detail, not app behavior to fix.

## Known limitation: brief stale frame on a very fast scroll

A fast scroll burst (flick, or several quick wheel ticks) through a pinned section can leave a stale painted frame on screen for a moment — the underlying DOM/layout is correct throughout (verified directly), but the paint doesn't catch up until the next scroll event, at which point it self-corrects completely. It's never a permanent break.

Root cause per GSAP's own maintainers: native scrolling runs on its own compositor thread, decoupled from the JS thread where ScrollTrigger applies the pin, so under load the two can render a frame apart — a browser rendering-engine limitation, not a ScrollTrigger defect. See the [GSAP forum thread](https://gsap.com/community/forums/topic/40298-strange-behaviour-of-a-pin-element-using-scrolltriggers/) on this exact symptom.

`fastScrollEnd`, `ScrollTrigger.normalizeScroll(true)`, `pinType:'transform'`, and `will-change` hints on the heavily-animated elements (logo items, flip cards, the pinned sections themselves) are all applied as mitigations — these are the standard documented responses to this issue. In testing, none of them provably eliminated it under an aggressive enough automated scroll burst; real device/trackpad testing is the more reliable way to gauge how often this actually surfaces for real users, since typical human scrolling may not trigger it as readily as a scripted burst does.
