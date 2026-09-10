# CineView — Interactive 3D Cinema Seat Preview

**[▶ Live demo — cinema-seat-preview.pages.dev](https://cinema-seat-preview.pages.dev/)**

Pick any seat in a 3D auditorium and the camera flies into it, giving you a first-person view of the
screen from that exact chair — so you can judge distance, angle and sightline before buying. Price,
row/seat label and a computed *view score* update live.

It's a **concept demo**: the room, the seat map, availability and prices are all simulated. No real
tickets are sold.

> The port target of the football-stadium "experience every seat" demo, rebuilt for a movie theatre.
> See [`SPEC.md`](SPEC.md) for the full design brief and the technique-by-technique mapping.

---

## Run it

Easiest is the [live demo](https://cinema-seat-preview.pages.dev/) — no install, just a browser
with WebGL.

To run it locally: the whole app is a single self-contained `index.html` (Three.js + GSAP from CDN,
no build step). It still needs to be served over HTTP — the screen texture loads
`references/images/*.jpg`, which browsers block from `file://`:

```bash
python3 -m http.server 8000   # from the project root
# open http://localhost:8000
```

Any static server works (`npx serve`, `php -S`, VS Code Live Server…).

## Controls

| Action | Result |
|---|---|
| Drag | Orbit around the auditorium (auto-rotates gently until you first interact) |
| Scroll | Zoom (clamped 14–55 m) |
| Hover a seat | Highlight + tooltip with label, price and view score |
| Click an available seat | GSAP flight into the seat → first-person view of the screen |
| Drag in seat mode | Small free-look around (±0.7 rad yaw, ±0.5 rad pitch) |
| `Esc` / "Back to auditorium" | Fly back out to the orbit view |
| `Enter` / "Select seat" | Confirm the seat (mock checkout) |
| "Best seat" | Fly to the highest-scoring available seat |

Clicking a sold seat just toasts a message; sold seats aren't selectable.

---

## What's in the room

Generated procedurally from a small config block — no imported models, no image assets except the
screen still.

```js
const ROOM   = { rows: 14, seatsPerRow: 22, seatW: 0.95, rowDepth: 1.2,
                 firstRowZ: -13, floorY: 0.5, rake: 0.42, curve: 0.012 };
const SCREEN = { w: 22, h: 9.2, cx: 0, cy: 5.6, cz: -18, tilt: -0.05 };
```

**308 seats** (14 rows × 22), on a stepped floor that rises 0.42 m per row, with a gentle parabolic
row curve so the end seats pull back toward the screen. Each seat yaws to face the screen centre.

Set dressing, all drawn on `<canvas>` → `CanvasTexture`: carpeted step platforms, fluted acoustic
side-wall panels, ceiling + downlights, aisle strip lighting, `EXIT` signs, and golden wall
sconces (bright fixture plane + additive glow quad + a real `SpotLight` aimed up the wall). The
screen is the room's main light source; everything else is dim.

### Tiers & pricing

| Rows | Tier | Ticket price | Seat colour |
|---|---|---|---|
| A–D (front) | Standard | $12.50 | `#8a2b2b` |
| E–J (middle) | Premium | $18.50 | `#b8863b` |
| K–N (back) | Premium Plus | $24.50 | `#3b6db8` |

Sold seats (~40%, deterministic) render dark grey and are excluded from selection.

### View score

`seatScore(i)` starts at 99 and subtracts three penalties, clamped to 45–99:

- **Distance** — deviation from an ideal ~17 m eye-to-screen-centre distance (up to −24)
- **Off-centre** — horizontal distance from the centreline (up to −14)
- **Look-up angle** — how steeply you have to crane upward, past ~0.12 rad (up to −16)

Labels: *Excellent view / Great view / Good view / Fair view / Restricted view*.
The "recommended" seat shown on boot is the highest-scoring **available** seat.

---

## How it works

- **Instanced seats** — one `InstancedMesh` for all 308 chairs; the chair itself is five boxes
  (pedestal, cushion, reclined backrest, two armrests) merged into a single `BufferGeometry`.
  Per-instance matrices and colours; hover/select just repaint one instance colour.
- **GPU colour picking** — a parallel `InstancedMesh` of pick boxes in an offscreen scene, each
  painted a unique ID colour by a two-line shader. `pickAt()` uses `camera.setViewOffset` to render
  a **1×1 pixel** at the cursor and reads it back → seat index. Hover picking is throttled to 40 ms.
- **Deterministic RNG** — `mulberry32(20260727)` drives sold seats and per-seat colour jitter, so
  every reload gives the identical room.
- **Camera flight** — GSAP tweens a single scalar `t: 0→1` along a 4-point `CatmullRomCurve3`
  (arcing up over the rows, then down into the seat eye at +1.15 m), while `lookAt` lerps toward
  the screen centre and the FOV narrows 58° → 54°. Exiting reverses it with a 3-point curve.
- **Seat mode** — camera pinned at the seat eye with a subtle two-sine breathing sway, drag for
  free-look, screen-locked base orientation.
- **POV thumbnail** — `captureView(i)` teleports the camera to the seat eye, renders one frame,
  copies the canvas into a 480×300 offscreen canvas and stores it as a JPEG data URL in the panel,
  then restores the camera. Same-origin image only, so nothing taints the canvas.
- **Minimap** — top-down 2D seat map redrawn every frame in the legend, colour-coded by tier with
  the selection in green.
- **Adaptive quality** — FPS sampled every 2.5 s; `pixelRatio` steps down below 40 fps and back up
  above 75 fps, between 0.75 and 2.
- **Reduced motion** — `prefers-reduced-motion` kills auto-rotate, the sway, and shortens flights.

---

## Files

```
cinema/
├─ index.html          # the entire app (~1,040 lines: inline CSS + HTML + JS)
├─ SPEC.md             # design brief: technique port table, geometry model, build order
├─ README.md           # this file
└─ references/
   ├─ README.md        # what reference material to drop here to match a real cinema
   └─ images/
      └─ the-odyssey-trailer-still.jpg   # what plays on the screen
```

Dependencies are loaded from CDN at runtime:
[Three.js r128](https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js) and
[GSAP 3.12.5](https://cdnjs.cloudflare.com/ajax/libs/gsap/3.12.5/gsap.min.js).

## Deploying

Live on Cloudflare Pages at **<https://cinema-seat-preview.pages.dev/>**, deployed straight from
GitHub on every push.

Static hosting, no build command: connect the repo, leave **Build command** empty, and set **Build
output directory** to the folder holding `index.html`. For a client-facing deploy, consider
vendoring `three.min.js` and `gsap.min.js` into the repo so a CDN hiccup can't break the demo.

## Tweaking it

Everything worth changing lives in the two config objects near the top of the `<script>`:

- **Room size** — `ROOM.rows` / `ROOM.seatsPerRow` (seat count and step platforms follow).
- **Rake** — `ROOM.rake`, the vertical rise per row.
- **Row curvature** — `ROOM.curve`; set to `0` for straight rows.
- **Screen** — `SCREEN.w/h/cz` for size and how far back it sits.
- **Tier bands and prices** — `tierOf()` and the `TIER` array (`base` is USD; `fmtPrice()` formats it).
- **Ideal viewing distance** — `idealD` inside `seatScore()`.
- **Sold ratio** — the `rng() < 0.4` test in the seat fill loop.
- **What's on the screen** — `refImg.src`, or replace `drawScreen()` with a `<video>` texture.

Not yet built from the spec: aisle gaps in the seat block, per-tier seat geometry (recliner
silhouette), instanced patrons on sold seats, and the ambient Web Audio room tone.

---

Made with ♥ by [Rey Molina](https://x.com/reymlgz).
