# CineView — Interactive 3D Cinema Seat Preview

**Goal:** Recreate the football-stadium "experience every seat" concept, but for a **movie theater**. A client browses a 3D auditorium, clicks any seat, and the camera flies into a **first-person view from that seat looking at the screen** — so they can judge sightline, distance, and angle before buying. The seat's price, row/number, and a "view quality" score update live.

This is a **concept/demo** (like StadiView): fixtures, prices, availability, and checkout are simulated. No real tickets sold.

---

## 1. What we're porting from the football-stadium project

The football project (`../football-stadium/index.html`, one self-contained ~4,800-line file) proved out a set of techniques. Almost all of them transfer directly. The table below maps each technique to its cinema equivalent.

| Technique (football) | What it did | Cinema adaptation |
|---|---|---|
| **`InstancedMesh` for all seats** | One draw call for ~35k seats sharing geometry; per-seat matrix + color buffers | Same, but hundreds of seats (~100–400). Still use instancing — it keeps picking/coloring uniform and trivially scales to multiplex-sized rooms. |
| **Parametric seat layout** | Ellipse tiers + arc-length tables for even spacing on a curve | Cinema rows are simpler: a **grid** (rows × seats) on a **sloped/stepped floor**, optionally **curved** rows facing the screen. No arc-length table needed for straight rows; a gentle arc is a single `curve(x)` offset. |
| **GPU color-picking** | Render a 1×1 px at the cursor into an offscreen target; each seat painted a unique ID-color; read back pixel → seat ID | **Port verbatim.** This is the cleanest way to pick one seat out of many. Works identically regardless of seat count. |
| **Deterministic RNG (`mulberry32`)** | Seeded PRNG → same layout, same "sold" seats every reload; reproducible featured seat | **Port verbatim.** Use it for which seats are sold, tiny per-seat color jitter, and a reproducible "recommended seat" (e.g. center, ~⅔ back — the acoustic/visual sweet spot). |
| **GSAP camera flight on a Catmull-Rom spline** | Tween `{t:0→1}` along a 4-point spline; simultaneously lerp `lookAt` + narrow FOV; respects `prefers-reduced-motion` | **Port the pattern.** Cinema flight arcs from orbit position → over the rows → down into the seat eye, ending aimed at the screen center. |
| **Canvas-drawn textures (asset-free)** | Pitch stripes, concrete, LED boards, sky all drawn on `<canvas>` → `CanvasTexture` | Cinema needs: carpet, acoustic wall panels, screen content, ceiling, aisle strip lighting, EXIT signs. All drawable on canvas. |
| **POV thumbnail capture** | Render scene from seat eye → offscreen canvas → `toDataURL` JPEG for the info panel | **Port verbatim.** Show a thumbnail of "your view of the screen" in the seat panel. |
| **Synthesised audio (Web Audio)** | Crowd noise generated, no audio files | Optional: low room ambience / muffled trailer audio, or skip. Lower priority than the stadium's crowd. |
| **Geometric "view score" + pricing** | Score from distance-to-pitch, halfway-line alignment, elevation | Cinema score from **distance to screen** (too close = neck strain, too far = small), **horizontal offset from centerline**, and **viewing angle**. Premium rows priced higher. |
| **Adaptive quality** | Measure FPS every 2.5s, nudge `pixelRatio` | **Port verbatim.** |
| **Single self-contained `index.html`** | All CSS/HTML/JS inline; Three.js + GSAP from CDN | **Keep this.** One portable file. (Optionally start under Vite for DX, then it's still deployable as static HTML.) |

### Techniques we DON'T need
- Arc-length ellipse tables (`arcTable`/`thetaAt`) — cinema rows are grids, not a 360° bowl.
- Full-bowl crowd instancing, players/ball match simulation, scoreboards, floodlight cones — no gameplay. (A subtle "other patrons" instanced crowd on sold seats is a nice-to-have, not core.)

---

## 2. Tech stack

| Layer | Choice | Notes |
|---|---|---|
| 3D engine | **Three.js** (r128 like the reference, or latest via ES modules) | Reference uses global `THREE` from CDN r128. Recommend **latest Three.js as ES modules** for a fresh build + `OrbitControls` instead of hand-rolling orbit. |
| Animation | **GSAP 3.x** | Camera flights: tween one scalar along a spline. |
| Build/dev | **Vite** | `npm run dev`. App still ships as static HTML. |
| Picking | **WebGL GPU picking** | 1×1 readback, ID-encoded colors. |
| Textures | **Canvas 2D → `CanvasTexture`** | Screen image, carpet, panels, signage — asset-free, or swap in real images later. |
| Optional audio | **Web Audio API** | Ambient room tone. |

Modern-build alternative to r128 globals:
```bash
npm create vite@latest cineview -- --template vanilla
npm i three gsap
# import * as THREE from 'three'
# import { OrbitControls } from 'three/addons/controls/OrbitControls.js'
```

---

## 3. Cinema geometry — the core model

A cinema auditorium is much simpler than a stadium bowl. The scene is essentially:

```
        ┌─────────────────────────────┐
        │          SCREEN             │   ← large plane, slightly tilted, at the front
        └─────────────────────────────┘
   ~~~~~~~~~~~~~ (stage / masking) ~~~~~~~~~
      row A  ● ● ● ● ● ● ● ● ● ● ● ●         ← front rows (close, cheap, steep look-up)
      row B  ● ● ● ● ● ● ● ● ● ● ● ●
      row C  ● ● ● ● ● ● ● ● ● ● ● ●   floor slopes UP toward the back
       ...
      row K  ● ● ● ● ● ● ● ● ● ● ● ●         ← back rows (premium/recliner, best angle)
              \_____ aisle _____/
```

### Parametric layout (the cinema equivalent of the tier/ellipse config)

Define the room as data, then generate seats in code:

```js
const ROOM = {
  screen:   { width: 20, height: 8.5, y: 5.5, z: -18, tilt: -0.06 }, // metres
  rows:     14,            // A..N
  seatsPerRow: 22,         // may vary per row (aisles create gaps)
  seatW: 0.95,             // seat spacing across a row (x)
  rowDepth: 1.15,          // spacing between rows (z)
  firstRowZ: -10,          // z of the front row (closest to screen)
  rake: 0.42,              // vertical rise per row (stadium seating slope)
  curve: 0.015,            // row curvature: how much ends pull back toward screen
  aisles: [7, 15],         // seat indices where a walking aisle splits the block
};
```

### Seat placement loop (replaces the arc-length math)

```js
// For each row r (0 = front), each seat s (0 = house-left):
const z = ROOM.firstRowZ - r * ROOM.rowDepth;          // deeper = further back
const y = ROOM.floorY + r * ROOM.rake;                 // stadium rake: back rows higher
const xCenter = (s - (seatsPerRow - 1) / 2) * ROOM.seatW;
// optional gentle curve so end seats angle toward the screen:
const curveZ = ROOM.curve * xCenter * xCenter;         // parabola, ends pull toward screen (+z)
const pos = new THREE.Vector3(xCenter, y, z + curveZ);
// seats face the screen; yaw toward screen center, plus a slight inward toe from the curve:
const yaw = Math.atan2(screenCenter.x - pos.x, screenCenter.z - pos.z);
```

Then it's the **same instancing pipeline as football**: count seats → allocate `InstancedMesh(seatGeo, seatMat, SEAT_COUNT)` → set per-instance matrix (position+yaw), per-instance color, and a parallel `pickMesh` with ID-encoded colors.

### Seat geometry (cinema chair, merged like the football seat)
Build from boxes and merge into one buffer geometry (matches football's `pan + backrest + pedestal` approach):
- seat base/cushion, backrest (reclined a few degrees), two armrests, optional cupholder nub.
- Premium tier: bigger recliner silhouette. Encode tier per seat so geometry or scale differs.

---

## 4. The "view from the seat" — the whole point

This is where cinema differs most and matters most.

**Eye position** for seat `i`: seat position + seated eye height (~1.1 m above the seat base).

**Look target:** the **center of the screen** (`ROOM.screen` center). Unlike the stadium (look at pitch center + free look-around), the natural cinema gaze is locked toward the screen, with optional small free-look.

**FOV:** narrow to ~50–55° on arrival so the screen frames like a real viewing experience. Front-row seats will have the screen filling/overflowing the frame (conveys "too close"); back rows frame it comfortably (conveys "great seat").

**POV thumbnail:** render from eye → `toDataURL` → show in the panel (football's `captureView`, verbatim).

**Play something on the screen:** the screen plane's texture should show *content*, not black — a looping gradient/bars/movie-still drawn on canvas, or a real `<video>` texture. This sells the preview.

---

## 5. View score & pricing (cinema logic)

Replace the football `seatScore` (distance-to-pitch / halfway-line / elevation) with cinema optics:

```js
function seatScore(pos) {
  const toScreen = screenCenter.clone().sub(pos);
  const dist = toScreen.length();
  // 1. Distance sweet spot: THX-ish ideal viewing distance ~ where screen subtends a good angle.
  const distPenalty = ...;   // penalize too close (< ~8m) AND too far (> ~24m)
  // 2. Horizontal centering: seats near the centerline score higher.
  const offCenter = Math.abs(pos.x) ...;
  // 3. Vertical/pitch angle: looking too steeply up (front rows) is bad.
  const lookUp = ...;
  return clamp(99 - distPenalty - offCenter - lookUp, 45, 99);
}
```

- **Sweet spot:** center block, ~⅔ of the way back — highest score, this is the reproducible "recommended seat" (football reserved Section 125/Row 12/Seat 18; cinema reserves e.g. Row J, Seat 11).
- **Tiers → pricing:** Standard / Premium / Recliner. `price = base[tier] + score * k`. Front two rows cheapest.
- **Score labels:** `Excellent / Great / Good / Fair / Restricted` (screen partly blocked / extreme angle).

---

## 6. Auditorium set-dressing (canvas textures / simple geometry)

To make it read as a real cinema, add (in rough priority order):
1. **Screen** — bright plane with animated content; subtle masking/curtains border, slight tilt.
2. **Sloped floor / steps** — the rake made visible as tiered platforms under each row.
3. **Side wall acoustic panels** — vertical fluted/quilted panels, dim lit (canvas texture).
4. **Aisle floor strip lighting** — emissive line lights along the steps (guides the eye, very "cinema").
5. **EXIT signs** — small emissive green planes flanking the screen and at the back.
6. **Ceiling** — dark, with a few downlights (PointLights, low intensity) and maybe a projector beam cone toward the screen.
7. **Ambient dark lighting** — the room is dim; the **screen is the main light source** (a large area/rect light or an emissive plane + a fill light). This is the signature cinema mood.
8. Optional: instanced "other patrons" seated on sold seats (football's crowd-on-sold-seats trick), a back-wall projection booth window.

---

## 7. UI / interaction (mirror the football UX)

- **Orbit mode:** drag to rotate around the auditorium, wheel to zoom, gentle auto-rotate until first interaction. Use `OrbitControls`.
- **Hover:** GPU-pick on mousemove (throttled) → highlight seat + tooltip with price/score.
- **Click seat:** `flyToSeat(i)` → GSAP spline flight → enter seat mode aimed at screen.
- **Seat mode:** first-person, small free-look (drag), subtle idle "breathing" sway, POV thumbnail, "Back to auditorium" pill, "Grab seat" confirm.
- **Info panel:** row/seat label (e.g. "Row J · Seat 11"), tier, price, view score + label, availability, POV thumbnail.
- **Minimap / overview:** top-down 2D canvas of the seat map with the selected seat marked and sold seats greyed (football drew a minimap + overview canvas — port it; a top-down seat map is even more natural for a cinema).
- **Keyboard:** `Esc` exit seat mode, `Enter` confirm. Respect `prefers-reduced-motion`.

---

## 8. Suggested file structure

```
cinema/
├─ SPEC.md            ← this file
├─ references/        ← DROP CINEMA REFERENCE IMAGES / LAYOUTS HERE (see references/README.md)
├─ index.html         ← (to build) self-contained app, football-stadium style
└─ package.json       ← (to build) vite + three + gsap, if using a build
```

Start either:
- **A) Single-file (fastest, matches reference):** one `index.html`, Three.js + GSAP from CDN, inline everything. Best for a portable demo.
- **B) Vite + ES modules (better DX):** modern Three.js, `OrbitControls`, hot reload; still deploys as static files.

Recommendation: **B for development**, keeping the whole app effectively in one module so it stays as portable as the reference.

---

## 8b. Deployment (GitHub + Cloudflare Pages)

The single-file approach is ideal here — **no build step means the static HTML deploys as-is.**

- **GitHub:** push the `cinema/` project to a repo (or its own repo). The demo is just `index.html` + assets.
- **Cloudflare Pages:** connect the GitHub repo. Since it's static, config is trivial:
  - **Build command:** *(none)* — leave empty.
  - **Build output directory:** the folder containing `index.html` (e.g. `/` or `cinema`).
  - Every push auto-deploys; you get a `*.pages.dev` URL to share with clients.
- **CDN caveat:** the app loads Three.js + GSAP from public CDNs. That's fine for a demo, but for a reliable client-facing deploy, **vendor the two libraries into the repo** (download `three.min.js` and `gsap.min.js`, reference them locally) so the demo never breaks if a CDN hiccups.
- **Cloudflare Workers:** **not needed for the static POC.** Only add a Worker if the project later needs a backend — real-time seat availability, saving selections, or a checkout/payment flow. Pages Functions (a Worker under the hood) can be added incrementally without changing hosting.

---

## 9. Build order (milestones)

1. **Scene skeleton** — renderer, dark scene, camera, `OrbitControls`, ground/floor, a lit screen plane. Confirm it renders.
2. **Parametric seats** — `ROOM` config + placement loop → single `InstancedMesh`, sloped/curved rows facing screen. Confirm the block looks like an auditorium.
3. **GPU picking + hover/click** — pick mesh with ID colors, 1×1 readback, hover highlight, tooltip.
4. **Seat data** — deterministic sold/available, `seatScore`, pricing, tiers, section/row/seat labels.
5. **Camera flight** — GSAP spline `flyToSeat` → seat mode aimed at screen, FOV narrow, free-look, breathing sway.
6. **POV thumbnail + info panel** — `captureView`, live panel, "recommended seat" on boot.
7. **Set dressing** — acoustic panels, aisle lights, EXIT signs, screen content, ceiling downlights.
8. **UI polish** — minimap/overview, dock controls, reduced-motion, adaptive quality, checkout mock.

---

## 10. Open questions / decisions to confirm before building

- **Auditorium style:** single standard screen, or a specific real cinema layout you want matched? (Drop a layout/photo in `references/`.)
- **Row curvature:** straight rows (simplest) or curved rows (more premium look)?
- **Seat count / room size:** small boutique (~80 seats) vs. large (~300)? Affects rake and screen size.
- **Screen content:** abstract animated placeholder, a movie still, or a real looping video texture?
- **Build mode:** single-file `index.html` (A) or Vite/ES-modules (B)?
- **Branding:** name (CineView is a placeholder), colors, whether to mirror the football project's paper/ink UI style or go dark-cinema.

> If you have a specific cinema you want to recreate, put photos, seat-map screenshots, or floor plans in `cinema/references/` and I'll match dimensions, row/seat counts, curvature, and screen proportions to it.
