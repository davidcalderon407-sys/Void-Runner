# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

VOID RUNNER — an HTML5 Canvas arcade space-shooter (Spanish UI), installable as a PWA on
Android/iOS. The entire game (markup, styles, and logic) lives in one file: `index.html`.
`manifest.json`, `sw.js`, and `icons/` add PWA installability on top of it. There is no
build step, no package manager, no JS dependencies, and no test suite.

Files:
- `index.html` — the whole game (all HTML/CSS/JS). This is the only file you'll usually edit.
- `manifest.json` — PWA manifest (name, colors, icon references, `display: standalone`).
- `sw.js` — service worker; cache-first offline caching of the core assets. Bump
  `CACHE_NAME` here whenever a cached asset changes, or returning players stay stuck on a
  stale cached copy.
- `icons/` — app icons (`icon-192.png`, `icon-512.png`, `icon-maskable-512.png`,
  `apple-touch-icon.png`, `icon-32.png` favicon), all generated from a one-off Pillow
  script (not checked in) that vector-draws the in-game ship shape onto a gradient
  background at each size. If the ship's shape/palette in `drawPlayer()` changes
  meaningfully, regenerate these to match rather than hand-editing PNGs.

## Running it

**Base game**: open `index.html` directly in a browser (`file://` works fine — no server
needed) — or serve it locally:

```bash
python3 -m http.server 8934
# then open http://localhost:8934/index.html
```

**PWA features** (service worker registration, install prompt/"Add to Home Screen") only
work over `http://`/`https://`, never `file://` — the service-worker registration script at
the bottom of `index.html` already guards for this (checks `location.protocol` and no-ops
otherwise), so opening the file directly still works, it just won't be installable. To
actually test install behavior on a phone, the whole directory needs to be hosted
somewhere reachable from the device (GitHub Pages, Netlify, etc.) — a bare local server on
`localhost` won't be reachable from a separate physical phone.

There is no lint/build/test command — verify changes by loading the page and playing,
and by checking the browser console for errors.

## Architecture

Everything is inside one IIFE in the `<script>` tag at the bottom of `index.html`. Read
top to bottom, it's organized as: canvas/offscreen-layer setup → audio → utilities →
difficulty/level constants → game state → input handling (keyboard, touch joystick+fire
button) → menu/screen wiring → spawning → the `update(dt)` / `render()` game loop.

**Game state machine**: a single `state` string drives everything — `'start' | 'playing' |
'paused' | 'gameover'`. Screens are plain HTML `<div class="overlay">` elements toggled via
the `.hidden` class (`startScreen`, `pauseScreen`, `gameOverScreen`, `exitScreen`), not
canvas-drawn. The canvas keeps rendering behind them (background/stars always animate; the
gameplay entities and decorative start-screen ship are drawn conditionally on `state`).

**Performance-critical convention — pre-bake, then blit.** This game went through several
rounds of perf debugging (see comments throughout the code). The pattern that fixed it, and
that any new visual element must follow:

- Anything whose appearance doesn't change frame-to-frame (background sky+nebulae, the
  planet+ring, the galaxy, orb glows, asteroid rock textures) is painted **once** to a
  small offscreen `<canvas>` (see `renderBackgroundLayer`, `renderPlanetLayer`,
  `renderGalaxyLayer`, `renderOrbSprites`, `makeAsteroidSprite`), then every frame it's
  just `ctx.drawImage(...)`'d into place (with rotation/scale via `ctx.translate/rotate`
  around the blit, never by recomputing the gradient).
- Offscreen layers tied to screen size are rebuilt from `resize()`, which is **debounced**
  (150ms) because mobile browsers fire bursts of resize events.
- Asteroids draw from a **fixed pool** of pre-baked sprites (`buildAsteroidSpritePool`:
  5 shapes × 4 sizes = 20 canvases, built once at startup). Every spawned asteroid
  references one of these — it does not create its own canvas. (An earlier version created
  a new canvas per asteroid, and per fragment when asteroids split into pieces; that's why
  splitting-on-destroy was removed and asteroids now come in fixed size tiers instead.)
- Ship hull/cockpit gradients (`playerHullGrad`, `playerCockpitGrad`) are built once and
  reused; only the thruster-flame gradient is rebuilt per frame (its length is dynamic).
- Bullets are drawn with two batched `beginPath()`/`stroke()` calls (normal vs. spread-shot
  color), never one `save()`/`restore()` per bullet.
- **`backdrop-filter` (CSS blur) is deliberately not used anywhere.** It was the actual
  root cause of an earlier "gets slower over time" bug: blurring live canvas content behind
  an element forces the browser to recompute the blur every frame the canvas changes, and
  the canvas is basically always changing (star twinkle, motion) during `'playing'`. Panels
  use solid semi-transparent backgrounds instead. Don't reintroduce `backdrop-filter` on
  anything that sits above the canvas during `'playing'`.
- Star twinkle/position updates (and `galaxyRotation`) are skipped entirely while
  `state === 'paused'` so a paused screen is truly static, not just slowed down.
- Layered glow effects (the galaxy's `globalCompositeOperation = 'lighter'` arms/halo/core)
  are baked at deliberately low alpha — additive blending stacks fast, and an earlier pass
  had it bright enough to wash out everything drawn on top of it.

**Gotcha — the planet's ring is drawn in two passes for occlusion**, one arc before the
sphere fill (the "far" half, behind the planet) and one arc after (the "near" half, in
front). The two `ellipse(...)` calls in `renderPlanetLayer` must use angle ranges that
meet exactly at `0` and `Math.PI` (i.e. `(0, Math.PI)` and `(Math.PI, Math.PI * 2)`) — an
earlier version used inset ranges like `(0.15π, 0.85π)` for a "nicer" arc and it left
visible gaps in the ring where neither pass drew anything.

**Difficulty/level curve is level-based, not time-based.** `levelProgress()` returns 0→1
across `MAX_LEVEL` (20) levels and everything (`spawnIntervalFor`, `asteroidSpeedMultFor`)
derives from that, scaled by the chosen `DIFFICULTIES` preset. Do not reintroduce a
`elapsed`-based multiplier for spawn rate/speed — an earlier version divided by session time
directly and the difficulty spiraled out of control the longer a run lasted.
`MAX_CONCURRENT_ASTEROIDS` is a hard ceiling on simultaneous asteroids regardless of what
the spawn timer computes, as a safety net. Leveling up (`checkLevelUp`/`triggerLevelUp`)
grants +1 life (capped at `MAX_LIVES`), or bonus score once at the cap.

**Entities** (`asteroids`, `orbs`, `bullets`, `particles`) are plain arrays of objects,
updated and pruned in `update(dt)` (iterate backwards, `splice` when off-screen/expired/
consumed), then drawn in `render()`. Orbs carry a `type` (`'energy' | 'spread' | 'life'`)
that picks both their sprite and their pickup effect. `spreadShotTime` is a countdown that
switches `fireBullet()` from a single straight shot to a 5-way angled spread
(`SPREAD_ANGLES`).

**Persistence**: high score only, via `localStorage`, wrapped in `loadBest`/`saveBest`
try/catch — file:// origins (e.g. Safari) can throw on `localStorage` access, and an
earlier version let that exception abort the whole script before event listeners were
attached, silently breaking every button.

**Audio**: procedural beeps via Web Audio (`beep()` + the `sfx` map), no audio assets.
`AudioContext` is created lazily on first user interaction (`ensureAudio()`), since
browsers block autoplay before a gesture.

**Input**: keyboard (`WASD`/arrows to move, `Space` to fire, `P`/`Esc` to pause) and touch
(a floating joystick built by the generic `createJoystick()` for movement, plus a fixed
`#shootZone` button) both feed the same `axisX/axisY`/`wantShoot` values consumed in
`update(dt)` — there's no separate touch-only code path in the simulation itself.

There is no version control (`git`) initialized in this directory as of this writing.
