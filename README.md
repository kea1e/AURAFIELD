# AURAFIELD

Hand-tracked particle field in the browser. One hand, three gestures, 40,000 particles.

Inspired by [nicholaspjm's TouchDesigner hand-tracked particle sim](https://www.youtube.com/watch?v=biGb10vV3ek) — rebuilt from scratch for the web.

## Run

```bash
cd ~/projects/AURAFIELD
python3 -m http.server 8777
```

Open <http://localhost:8777> in Chrome and allow camera access.

**Must be served over HTTP.** `getUserMedia()` is blocked on `file://` URLs — double-clicking the HTML file will not work.

## Gestures

| Gesture | Behavior |
|---|---|
| ✋ **Open hand** | Particles flow along your hand's direction of travel. Move faster, push harder. |
| ✊ **Closed fist** | True freeze frame — the field stops on the exact frame you closed your hand. |
| 🤏 **Pinch** | Particles gather into a real rotating sphere between your fingertips. |

Going **pinch → open** throws the sphere apart in a radial shockwave. Fist → open gives a
softer push.

Drop your hand out of frame and the field drifts back to rest.

## Keys

| Key | Action |
|---|---|
| `H` | toggle HUD |
| `V` | toggle camera preview |
| `S` | toggle hand skeleton |
| `R` | reseed particles |
| `Space` | pause |

## How it works

- **MediaPipe Tasks-Vision** `HandLandmarker` — 21 landmarks per hand, GPU delegate, video mode.
- **Gesture classification** is pure landmark geometry. Distances are normalized by
  `dist(wrist, middle_MCP)` so thresholds hold whether your hand is near or far.
  Pinch is tested *first* — a pinching hand still reads as "fingers extended."
- **Physics** runs CPU-side over flat `Float32Array`s. Each gesture is a set of force
  parameters (damping, hand force, orb force, noise, home pull) that get **lerped**
  toward, never hard-switched — that's what makes transitions feel liquid instead of snappy.
- **Rendering** is a Three.js `Points` cloud with a custom additive shader. Particle color
  maps to speed: cold indigo → cyan → white-hot.
- **Trails** come from disabling `autoClear` and drawing a translucent black quad each
  frame instead of clearing.

## Tuning

Constants at the top of the `<script type="module">` block:

| Constant | Default | Effect |
|---|---|---|
| `COUNT` | `40000` | Particle count. CPU-bound — 100k is roughly the ceiling before FPS drops. |
| `FADE` | `0.16` | Trail persistence. Lower = longer smears. |
| `ORB_RADIUS` | `0.45` | Pinch orb size. |
| `HAND_RADIUS` | `1.5` | Open-hand influence falloff. |
| `uSize` | `78.0` | Particle size. Also clamped to 6px max in the vertex shader. |
| `PINCH_IN` / `PINCH_OUT` | `0.34` / `0.52` | Pinch enter/exit thresholds. Gap = hysteresis. |
| `EXT_IN` / `EXT_OUT` | `1.55` / `1.35` | Finger-extended enter/exit thresholds. |
| `DEBOUNCE` | per-gesture | Stable frames required before a state switch. |
| `PARAMS` | — | Per-gesture force table. This is the fun one. |

## Gesture recognition

Classification is rotation-invariant and hysteretic:

- **Scale normalizer** is 3D `dist(wrist, middle_MCP)` — a rigid bone, so it works as a
  ruler regardless of distance from camera.
- **Finger extension** is measured as `dist(tip, wrist) / scale`, *not* a `tip.y < pip.y`
  compare. The y-compare breaks the instant you tilt or invert your hand; distance-from-wrist
  doesn't care about orientation.
- **Hysteresis** on every threshold: a finger must pass `EXT_IN` to count as extended but
  only falls below `EXT_OUT` to stop counting. The gap between the two is what kills
  boundary flicker. Same pattern for pinch.
- **Per-gesture debounce** — pinch commits in 2 frames (it's intentional and distinct),
  open/fist take 3, dropping to idle takes 4.

## Known limits

- Physics is CPU-side; the ceiling is ~100k particles. Moving to GPGPU ping-pong render
  targets would get to 1M+.
- A tight pinch at certain hand angles can occasionally read as a fist. Adjust `PINCH_T`.
- Single hand only (`numHands: 1`).
