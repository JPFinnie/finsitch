# Cerebro Gesture + UI Hotfix Plan (for `JPFinnie/CerebroFinnie`)

## Why this plan exists

I could not directly patch `JPFinnie/CerebroFinnie` from this environment because outbound GitHub clone access is blocked (`CONNECT tunnel failed, response 403`) and `gh` CLI is not installed.

This file captures a concrete implementation plan for the exact issues you raised so you can apply quickly in the Cerebro repo.

## 0) Immediate Vercel unblock

From your screenshot, Vercel fails with:

- `src/components/BrainScene.tsx(4,10): error TS6133: 'BufferAttribute' is declared but its value is never read`

### Fix

In `src/components/BrainScene.tsx`, remove `BufferAttribute` from imports if unused, or use it explicitly if intended.

Example:

```ts
// before
import { BufferAttribute, ... } from 'three'

// after
import { ... } from 'three'
```

Then run:

```bash
npm run build
```

---

## 1) Gesture controls: make pan usable + reduce sensitivity

### A) Add state machine for gestures

Implement a small finite state machine in your gesture controller:

- `IDLE`
- `ORBIT`
- `PAN`
- `ZOOM`
- `PAUSED`
- `SELECTING`

Use confidence thresholds + debounce windows:

- Enter threshold: `>= 0.75`
- Exit threshold: `< 0.55`
- Hold to confirm new mode: `120-180ms`

This hysteresis prevents mode flicker and over-sensitivity.

### B) Smooth hand deltas

Apply EMA smoothing to pointer/pose deltas:

- `smoothed = alpha * current + (1 - alpha) * previous`
- start with `alpha = 0.18` for cursor motion and `alpha = 0.12` for camera rotation

Add velocity clamp per frame to prevent jump spikes:

- `maxDeltaPx = 26` (desktop)
- `maxDeltaPx = 18` (tablet)

### C) Pan mapping

Instead of direct world transform, pan in screen-space and project to camera target plane:

1. Convert hand delta to normalized device coords
2. Scale by camera distance (`distance * panGain`)
3. Apply to camera target X/Y
4. Keep Z unchanged (unless explicitly in zoom mode)

Recommended defaults:

- `panGain = 0.0016`
- `orbitGain = 0.0010`
- `zoomGain = 0.0022`

---

## 2) Add pause/resume so note interaction is possible

### UX behavior

- Add persistent top-layer control bar:
  - `Gesture: ON/OFF`
  - `Pause/Resume`
  - `Recenter`
  - live confidence indicator
- Keep camera feed visible as a floating overlay (always mounted).
- Closing settings must **not** stop camera stream.

### Gesture-level pause

Pick a dedicated pause gesture (e.g., `Thumb_Up`) and require dwell `>700ms`.

When paused:

- Disable orbit/pan/zoom handlers.
- Enable pointer emulation + dwell click (for opening notes):
  - Crosshair follows hand
  - Dwell on interactive target for `1.2-1.5s` to trigger click

---

## 3) Camera should be top layer, not buried in settings

### Refactor outline

- Move camera lifecycle from settings component into app-level provider:
  - `GestureCameraProvider`
- Initialize stream once at app shell mount.
- Settings only toggles visibility/preferences; never owns `getUserMedia` lifecycle.

This avoids stream teardown when panels close.

---

## 4) Brain graph visual simplification + performance

You mentioned the current "balls" are too heavy and slow.

### Replace heavy spheres

- Render nodes as tiny billboards/sprites or `Points` instead of lit sphere meshes.
- Use one shared material.
- Turn off shadows and expensive lighting for nodes.

### Use instancing

- Use `InstancedMesh` (or `THREE.Points`) for nodes.
- Keep edges in a single `LineSegments` buffer.

### Limit expensive updates

- Freeze layout after settle.
- Update only changed node attributes.
- Throttle hover/selection raycasting to `20-30Hz`.

### Visual style target (brain-like small connected nodes)

- Node radius: `1.5-2.5px` equivalent
- Low-alpha thin edges (`0.15-0.28`)
- Slight color variance by cluster
- Optional glow post-process disabled by default on mobile

---

## 5) Suggested control map

- `Victory`: orbit
- `Pointing_Up`: pan
- `Closed_Fist`: zoom in
- `Open_Palm`: zoom out
- `Thumb_Up (hold)`: pause/resume toggle
- `Open_Palm (hold 1.3s while paused)`: click/select note

---

## 6) Concrete implementation checklist in Cerebro

1. Fix TS6133 import issue in `BrainScene.tsx`.
2. Introduce gesture FSM + hysteresis.
3. Add smoothing + velocity clamping.
4. Create top-layer gesture HUD and persistent camera widget.
5. Move camera ownership to app root provider.
6. Add paused-mode dwell click for note selection.
7. Replace sphere graph rendering with instanced points + batched edges.
8. Validate with `npm run build` and Vercel preview deployment.

---

## 7) Validation tests to add

- Unit:
  - gesture mode transitions with confidence hysteresis
  - pause toggle hold timing
  - dwell click trigger timing
- Integration:
  - camera stream survives settings close/open
  - selecting note while paused works
- Perf:
  - FPS before/after for graph scene at 500, 2k, 10k nodes

---

## 8) If you want me to implement directly

If you can provide a local checkout of `CerebroFinnie` in this environment (or enable GitHub clone), I can patch it end-to-end in one pass and submit a PR-ready commit.
