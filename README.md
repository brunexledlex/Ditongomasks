# Monos

An AR experiment with illustration.

A symmetrical tribal mask is painted over every face in frame and rebuilds
itself from what each face is actually doing — blink and the eyes become struck
discs, scowl and the brows invert, shout and the mouth opens into a toothed
hexagon. The neutral eyes and the closed-eye blink are real artwork
(`Assets/Mascaras/`); every other expression falls back to a shape drawn
procedurally on the canvas until it has matching art of its own.

Open `index.html` over `http://localhost` or HTTPS (the camera needs a secure
context) and allow the camera. Nothing leaves the device.

## How it works

[MediaPipe Face Landmarker](https://ai.google.dev/edge/mediapipe) runs in
`VIDEO` mode with blendshapes on, up to `MAX_FACES` (4) faces, GPU delegate.
Each detected face gives 478 landmarks and ~52 blendshape scores per frame.

**Pose** comes from three measurements, so the mask tracks translation, scale
and roll:

| Quantity | Landmarks |
|---|---|
| Face centre | `1` — nose tip |
| Face width (the unit everything scales by) | `234` ↔ `454` |
| Head roll | `33` ↔ `263`, the outer eye corners |

The eye line is treated as an *undirected* axis folded into ±90°, so the mask
never flips upside down at a steep tilt and doesn't care which landmark ends up
on which side once the selfie mirror is applied. All three numbers run through
one EMA — raw landmarks jitter a pixel or two per frame, which on a 400px mask
reads as a shiver.

**Features** are chosen by quantising blendshape scores into discrete states.
The mask is symmetrical, so the stronger of a left/right pair drives both sides,
which also makes a one-eyed wink read as a blink:

| Feature | Blendshape | Threshold | Asset |
|---|---|---|---|
| Eyes | `eyeBlinkLeft/Right` | `> 0.4` | `eye_circle` (art) |
| | `eyeWideLeft/Right` | `> 0.4` | `eye_square` |
| | — | default | `eye_triangle` (art) |
| Brows | `browDownLeft/Right` | `> 0.35` | `brow_angry` |
| | `browInnerUp` | `> 0.35` | `brow_arch` |
| | — | default | `brow_flat` |
| Mouth | `jawOpen` | `> 0.5` | `mouth_hexagon` |
| | `jawOpen` | `0.15 – 0.5` | `mouth_oval` |
| | `jawOpen` | `< 0.15` | `mouth_diamond` |

The thresholds live in one object (`const T`) — tune them there.

## Drawing

Everything happens inside one transform: translate to the nose tip, rotate by
the roll, scale by `face.w * S.maskScale` (1 unit = one measured face width,
times the mask-size setting). So the mask scales and tracks with the face
rather than with the screen.

Two different kinds of drawing share that space:

- **Procedural fallback shapes** (`DRAW`) are drawn once per side and mirrored
  with `scale(-1, 1)` at a `LAYOUT` position given in face-width units, so the
  two halves can't drift apart.
- **Painted layers** (`ART`, plus the base `HEAD_ART`) are full-width PNGs that
  already contain *both* sides — the artist positions left and right together
  in one file — so they're drawn once, unmirrored, at a fixed offset.

## Painted layers (`Assets/Mascaras/<Set>/`)

Assets live under a named set folder (currently `Set 1`) rather than directly
in `Assets/Mascaras/`, so a future alternate mask design can sit alongside
this one — swapping skins is meant to be a one-line change to the `SET`
constant, nothing else. Filenames follow `<feature>_<variant>_<n>.png`
(`boca_A_1.png`, `boca_B_1.png`, …); the variant letter is just the artist's
own naming, not something the code parses — the mapping from a file to a
blendshape state is the explicit `ART` key below, chosen by hand.

Every layer in a set was exported from the **same 1916×1701 artboard**, so no
layer needs independent horizontal centring or scaling — only a vertical
offset in that artboard's own pixels:

```js
const HEAD_ANCHOR = { x:958, y:803 };   // the head art's own "nose tip"
const HEAD_UNIT = 875;                  // head-art px per 1 face-width unit
const SET = './Assets/Mascaras/Set 1/';
const HEAD_ART = { src:SET+'cabeca_A_1.png', w:1916, h:1701, x:0, y:0 };
const ART = {
  brow_flat:     { src:SET+'sobrancelha_A_1.png',    w:1916, h:296, x:0, y:48 },
  eye_triangle:  { src:SET+'olhos_A_1.png',          w:1910, h:473, x:3, y:309 },
  eye_circle:    { src:SET+'olhos_fechados_A_1.png', w:1910, h:286, x:3, y:466 },
  mouth_diamond: { src:SET+'boca_A_1.png',           w:1916, h:381, x:0, y:1060 },
  mouth_oval:    { src:SET+'boca_B_1.png',           w:1916, h:381, x:0, y:1069 },
};
```

`ART` is keyed by the **same state names** `shapes` uses (`eye_triangle`,
`brow_angry`, `mouth_hexagon`, …), so adding art for one more expression is a
one-line addition with nothing else to touch — `paint()` already checks
`ART[shapes.X]` before falling back to the procedural shape for every feature.

**`boca_B_1` → `mouth_oval` is a judgment call, not a derivation.** The art
itself is a thinner, flatter pill than the neutral `boca_A_1` — it doesn't
depict an open mouth the way the procedural `mouth_oval` it replaces did.
Absent a naming convention that states the intended expression (contrast
`olhos_fechados`, which names its own meaning), it was wired to the next
un-arted mouth threshold in sequence. Re-key it to a different `ART` entry —
or add a fourth mouth state if that's not the intended read — if that guess
is wrong.

**Finding the `y` offset for a new layer:** the layer's own content (its alpha
bounding box) has to land on the matching feature in the head art. The eye
sockets are the one feature baked into the head art with a hard edge, so they
can be found by thresholding for dark pixels within the eye band and used to
solve the offset exactly:

```python
# offset = head-space target Y  -  the layer's own content-center Y, in its own pixels
offset_y = head_socket_center_y - overlay_own_center_y
```

Brow and mouth have no equally sharp target in the head art, so their offsets
were set by compositing at a guess (`PIL.Image.alpha_composite`) and nudging
by eye against the reference art — close enough that the first derived value
for `HEAD_UNIT` (1750) independently reproduced the original procedural
layout's proportions almost exactly, a good sign the anchor and offsets were
right. The shipped value is smaller (larger mask) than that derivation: once
the proportions were confirmed correct, the whole mask was scaled up on top,
which — because `HEAD_UNIT` is *px per face-width unit* — means shrinking the
constant, not growing it.

A layer's image is loaded lazily and cached; if it fails to load (or hasn't
loaded yet) its feature silently falls back to the procedural shape for that
frame — same failure mode as the old per-icon override this replaced.

## Embers

While a face's mouth is open, it jets a small shower of glowing embers
downward off the chin — a plain particle system
(`spawnEmber`/`updateEmbers`/`drawEmbers`), one pool per face, living in that
face's own local unit-space so the particles ride along with its position,
rotation and scale for free. Spawn rate and count scale with `jawOpen`; a
closed mouth (`jaw ≤ 0.15`, the same cutoff the shape quantiser uses) emits
nothing and any embers already in flight simply finish their `EMBER_LIFE` and
stop being replaced. `vy` accelerates downward over each particle's life
(`p.vy += 0.5*dt`) rather than easing off, so the jet keeps falling rather
than drifting to a stop mid-air.

**Drawn on top of the base plate, not behind it**, despite the brief calling
for embers "behind the face" — the opaque plate is nearly as large as the
whole local unit-space (see `HEAD_UNIT` above), so a particle actually
drawn *behind* it would need an unbelievable lifetime or speed to ever clear
its silhouette, and would just render invisible the entire time it exists.
"Behind the face" reads better as *behind the sculpted features*: `paintOne`
draws the plate, then the embers (additive `globalCompositeOperation` —
`'lighter'`), then brow/eyes/mouth on top of those, so the glow shows on the
mask's own surface while the carved features stay crisp and unaffected.

Each ember's gradient goes hot-white centre → its own colour → transparent,
not a flat colour fading to nothing — a single-colour disc blended
additively onto this saturated blue/purple mask reads as pink where the two
mix, not orange; a bright core is what actually sells "glowing ember" rather
than "tinted circle".

## Multiple faces

Every face MediaPipe detects (up to `MAX_FACES`) gets its own independent
mask, position, expression and shape-commit timer — `faces` is an array of
otherwise-complete per-face state objects, everywhere the app used to hold
one `face`/`shapes`/`scores` singleton.

MediaPipe hands back a fresh array of detections every frame with **no
identity across frames** — the face at index 0 this frame is not necessarily
the same person as index 0 last frame, especially once anyone moves. So
`updateFaces()` matches by position instead: each existing track claims
whichever undetected-so-far face is nearest to where it last was (greedy
nearest-neighbour, capped at `MATCH_MAX_DIST` face-widths so two different
people can't be confused for one). A track nobody claims is dropped outright
— same instant-loss behaviour the single-face version had — and leftover
detections become new tracks. The HUD detail rows show whichever tracked face
is currently widest (closest to camera); the state line just says how many
there are.

**Draw order is smallest-first.** The canvas has no other depth cue, so size
stands in for distance from camera: `paint()` draws off a copy of `faces`
sorted by `fs.w` rather than `faces` itself (whose own order is what
`updateFaces` matches against next frame — sorting it in place would fight
that), so a bigger, nearer-reading mask overlaps and hides smaller ones
instead of tracking order deciding who's on top.

## Tracking style

Settings has a 4-way **Tracking style** control (`S.motion`) for how a mask
chases its face's raw landmark target — all four expressed against wall-clock
`dt`, not per-frame, so the feel doesn't change with frame rate:

| Style | Feel | How |
|---|---|---|
| `none` | Tight, instant | Snaps straight to the target every frame — the original behaviour. |
| `lag` (default) | Sluggish, trailing | Exponential approach, `k = 1 - exp(-dt/MOTION_TAU)`. No overshoot. |
| `spring` | Bouncy, overshoots | Underdamped spring-damper (`SPRING_FREQ`/`SPRING_ZETA`), semi-implicit Euler integration — stable even across a janky frame, unlike plain Euler on a spring this stiff. |
| `stepped` | Jerky, stop-motion | Holds position for `STEP_HOLD` seconds, then snaps with no interpolation at all. |

Blendshape-driven expression changes (blink, angry brows, …) get their own,
separate clumsiness: `SHAPE_DELAY` (0.2s) is how long a new candidate shape
has to hold *before* it commits, so a reaction reads as catching up to your
face rather than snapping instantly — independent of `S.motion`.

## Parallax

A small cosmetic depth cue: brows, eyes and mouth each sit at a notional
depth (`DEPTH`) relative to the base plate and shift against a yaw/pitch
proxy, capped by `PARALLAX` at a few percent of a face-width even at the
proxy's extremes — "a bit," not a real 3D head. The proxy itself comes from
landmark asymmetry, not MediaPipe's facial transformation matrix: turning the
head foreshortens the near cheek landmark toward the nose and leaves the far
one relatively put (and the same trick works vertically against the
forehead/chin), which is cheap and doesn't need a second model output.

## Mask size

Settings has a **Mask size** slider (50–400%, `S.maskScale`) that multiplies
the automatic face-width fit — 100% is the calibrated default. Persisted in
`localStorage`.

## Gallery

Snapshots composite the video frame and the mask overlay into one PNG
(`toDataURL('image/png')`), capped at 1080px on the long edge, and are kept in
`localStorage`. That's a ~5MB budget for the whole origin, so the store is
capped on **bytes as well as count** and evicts oldest-first, warning once when
it starts dropping. Download anything worth keeping.

## Chrome

Black ground, one mint accent, one blue selected fill, pill geometry, dark only
on purpose — the identity is inherited from
[SPLIIIT](https://github.com/brunexledlex/slitscan) so the two read as one
family. Icons are Material Symbols Sharp on the `0 -960 960 960` grid, inlined
as path data. No webfont, no CSS framework, no build step. One file.

## Credits

Design and direction by [Bruno Silva](https://ditongo.com) / Ditongo.
