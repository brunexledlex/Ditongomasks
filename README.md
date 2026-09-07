# Ditongomasks

An AR experiment with illustration.

A symmetrical tribal mask is painted over your face in the browser and rebuilds
itself from what your face is actually doing — blink and the eyes become struck
discs, scowl and the brows invert, shout and the mouth opens into a toothed
hexagon. Everything is drawn procedurally on a canvas. There are no image files.

Open `index.html` over `http://localhost` or HTTPS (the camera needs a secure
context) and allow the camera. Nothing leaves the device.

## How it works

[MediaPipe Face Landmarker](https://ai.google.dev/edge/mediapipe) runs in
`VIDEO` mode with blendshapes on, one face, GPU delegate. Each frame gives 478
landmarks and ~52 blendshape scores.

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
| Eyes | `eyeBlinkLeft/Right` | `> 0.4` | `eye_circle` |
| | `eyeWideLeft/Right` | `> 0.4` | `eye_square` |
| | — | default | `eye_triangle` |
| Brows | `browDownLeft/Right` | `> 0.35` | `brow_angry` |
| | `browInnerUp` | `> 0.35` | `brow_arch` |
| | — | default | `brow_flat` |
| Mouth | `jawOpen` | `> 0.5` | `mouth_hexagon` |
| | `jawOpen` | `0.15 – 0.5` | `mouth_oval` |
| | `jawOpen` | `< 0.15` | `mouth_diamond` |

The thresholds live in one object (`const T`) — tune them there.

## Drawing

Features are drawn in a unit space where **1 unit = one measured face width**,
after the canvas has been translated to the nose tip, rotated by the roll and
scaled. So the mask scales with the face rather than with the screen, and the
layout constants in `LAYOUT` are readable as real proportions.

Each paired feature is drawn once and mirrored with `scale(-1, 1)` rather than
placed twice, so the two halves can't drift apart.

## Overriding the artwork with images

Every feature is procedural by default. `ASSETS` maps each one to a unit-space
box; point `src` at a local file to override just that one:

```js
const ASSETS = {
  base:         { src:'./art/base.png', w:1.00, h:1.18 },
  eye_triangle: { src:null,             w:0.20, h:0.17 },
  // …
};
```

The image is drawn into the same box the procedural version uses, centred on the
same anchor, so nothing else changes. A file that fails to load silently falls
back to the procedural shape.

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
