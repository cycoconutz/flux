# FLUX — Kinetic Sound Arcade

> **Svelte 5 + SvelteKit (static) + Rapier2D-compat + Web Audio API + Canvas**

A flashy, highly-interactive physics arcade. Drag orbs to fling them, watch collisions become musical notes and paint splats.

Live: **https://cycoconutz.github.io/flux/** (after `git push`)
Repo: **https://github.com/cycoconutz/flux**

Catalog entry: appears as `06 — Arcade — 2026` on https://cycoconutz.github.io/portfolio-tabs

---

## Why it stands apart

The catalog was 5× React (Deadwax, TwilightVotes, Vault, Solace, Karmatic). FLUX is intentionally the odd one out:

- **Svelte 5 runes** (`$state`, `$derived`, `$effect`) — compiler reactivity, no VDOM, no React runtime
- **SvelteKit static** (`adapter-static`, `prerender = true`, `base: '/flux'` for GH Pages)
- **Rapier2D-compat WASM** (inlined base64, no separate `.wasm` fetch) — rigid bodies, restitution 0.88, pinball platforms
- **Web Audio API** — zero libraries, triangle oscillator → lowpass, quantized scales
- **Canvas 2D** — background foil shader via `globalCompositeOperation: lighter` + grain, paint splats + highlights

Static only — no backend, no DB, no tracking.

---

## Interactions (6+)

1. **Drag / fling orbs** — pointer velocity → `setLinvel`, damping 6 while dragging
2. **Click empty space / `N`** — spawn orb at pointer (max 24)
3. **Gravity** — slider `-1600…1600`, `G` invert, live `world.gravity`
4. **Bounce** — `0.2…0.99` restitution per collider
5. **Paint** — `P` toggle, collision mid-point splats, velocity trails with `trailFade` rect, `S` save PNG `toDataURL` → `localStorage`, `C` wipe
6. **Sound** — `M` mute, scale selector (pentatonic/major/minor/chromatic), collision → `pluck()` with `AudioContext`
7. **Palette** — `1–4` or dots: Foil / Nectar / Pacific / Void (iridescent CSS vars)
8. **Kinetic type** — `FLUX` glyphs chase pointer (`letterPos` + CSS transition), double-click scramble
9. **Shake** — `Space` → random impulses + torque
10. **Black hole** — `B` or Konami `↑↑↓↓←→←→BA` → central force `280000/(d+120)`, spawns shower
11. **Tilt** — `Tilt` button requests `DeviceOrientationEvent` permission, sets gravity from `gamma/beta`
12. **Help** — `?` modal, `Escape` close

Reduced-motion: grain and bloom still render, but `prefers-reduced-motion` disables spring transitions.

---

## Tech stack

- **Svelte 5.56**, **SvelteKit 2.63**, **Vite 6.4.3**, **TypeScript 6**
- **@dimforge/rapier2d-compat 0.20** (compat base64 WASM)
- **No Threlte/Three** — kept bundle small (~90 kB gzip)
- **Adapter:** `@sveltejs/adapter-static` → `build/` for GH Pages
- **Fonts:** Syne + Space Grotesk + JetBrains Mono

---

## Local dev

```bash
npm install
npm run dev      # http://localhost:5173
npm run build    # → build/ (prerendered)
npm run preview  # serve build/
```

Node 20+ required; Vite 6 pinned for Node 22.7 compat.

---

## Deploy to GitHub Pages

1. Create repo `cycoconutz/flux` (public).
2. Push this folder:

```bash
git init
git add .
git commit -m "feat: FLUX kinetic arcade — Svelte 5 + Rapier + Web Audio"
git branch -M main
git remote add origin https://github.com/cycoconutz/flux.git
git push -u origin main
```

3. GitHub → Settings → Pages → Source: **GitHub Actions** (workflow at `.github/workflows/deploy.yml` handles it).
4. Wait for Actions green, then open https://cycoconutz.github.io/flux/

> The build uses `paths.base = '/flux'` when `NODE_ENV=production` (via `svelte.config.js:13`). Local `npm run preview` after `npm run build` serves with base already baked; GH Pages serves `build/` correctly.

---

## Portfolio-tabs integration

A patch is ready in your local clone at `C:\Users\cyco\AppData\Local\Temp\opencode\portfolio-tabs-clone` (4 files + shot).

To apply to your real `portfolio-tabs` repo:

```bash
git clone https://github.com/cycoconutz/portfolio-tabs.git
cd portfolio-tabs
# copy src/data/projects.ts, src/theme/themes.css, src/transitions/wipes.ts, src/index.css patches
# and src/assets/shots/flux.png (14 kB placeholder — replace with a real screenshot from live FLUX if you want)
npm install
npx tsc -b && npx vite build   # verifies 6 tiles
git add .
git commit -m "feat: add FLUX arcade (06, Svelte/Rapier/Web Audio, shatter wipe)"
git push
```

Full diff saved at `portfolio-tabs-flux.patch` (ask me to emit it).

---

## Replacing the shot

Current `src/assets/shots/flux.png` is a generated placeholder. For a real shot:

1. Open live FLUX, throw a few orbs, set palette to `Foil`.
2. Screenshot 1280×720 (Chrome DevTools device frame off).
3. Save as `src/assets/shots/flux.png` and rebuild portfolio-tabs.

---

## License

MIT
