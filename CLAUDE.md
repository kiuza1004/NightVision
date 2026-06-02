# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Shape

Single-file vanilla web app. **Everything lives in `index.html`** — HTML, inline CSS, and a single inline `<script>` block. No package manager, no build step, no transpilation, no framework, no external runtime dependencies (no CDN scripts either).

`README.md` is the user-facing description. UI strings and code comments are Korean — keep that style when editing.

## Deploy / Run

- **Production**: GitHub Pages at https://kiuza1004.github.io/NightVision/ — serves `main` branch root. Pushing to `main` auto-rebuilds (≈1–2 min). Verify with `curl -I https://kiuza1004.github.io/NightVision/` and check build state via `gh api repos/kiuza1004/NightVision/pages/builds/latest`.
- **Local**: `python -m http.server 8000` from repo root. `getUserMedia` only works on `https://` or `localhost`/`127.0.0.1` — plain LAN IP over HTTP will refuse camera access. For real mobile testing, push to Pages rather than fighting local HTTPS certs.
- No lint/test commands exist. Syntax-check the script block before pushing: `awk '/<script>/{f=1;next}/<\/script>/{f=0}f' index.html > /tmp/nv.js && node --check /tmp/nv.js`.

## Architecture

### Mode system
Three modes — `filter`, `longexposure`, `sensor` — switched by `setMode(name)`. Each switch:
1. Cancels the active `requestAnimationFrame` loop via `stopRenderLoop()`.
2. Toggles `.tab.active`, `controls.hidden`, `settingsBar.hidden`.
3. Starts the new render loop (or, for sensor mode, runs a one-shot detection).

A single shared `<video>` (`visibility: hidden`) feeds the back-camera stream; all visual output is drawn into one full-screen `<canvas>` overlaid on top. Always cover-fit video → canvas via `drawVideoCover()` — never rely on CSS `object-fit` for the canvas.

### Filter mode (`renderFilter`)
Per-frame pixel pass with **Reinhard soft-knee tone-mapping** `boost = 255*raw / (raw + knee)`. This is deliberate — earlier versions stacked `ctx.filter = brightness(2.0)` + per-pixel multiply + camera `exposureCompensation=max` and bleached normal scenes to pure white-green. **Do not re-introduce stacked brightness boosts.** The canvas draw uses `saturate(0)` only; all brightness control happens in the pixel loop driven by the `gain` slider (×1–×8). Green tone is fixed: `R·0.10`, `G·1.0`, `B·0.18`.

### Long-exposure mode (`startLongExposure`)
Accumulates 3 s of frames into a `Float32Array(w*h*3)` on an offscreen canvas, then dumps to a PNG dataURL shown in a modal.
- `lxMode === 'max'` (lighten): per-channel max — good for sparse point lights (gathers stars, headlights).
- `lxMode === 'avg'`: per-channel sum, divided by frame count — `√N` SNR improvement, needs a still device.
- Final `od[i] = min(255, value * 1.15 * gain)` — final clip happens here; over-cranking `gain` in `max` mode can still saturate.

### Sensor mode (`startSensorMode`)
**Detection-only stub.** Calls `navigator.xr.isSessionSupported('immersive-ar')` then attempts `requestSession` with `requiredFeatures: ['depth-sensing']` and **immediately ends the session**. Any throw (no WebXR, no AR, no depth-sensing) is caught and renders the Korean "미지원" message via `drawSensorMessage()`. There is no actual depth rendering. iOS Safari always falls through to the unsupported branch.

### Camera capability boost (`tryBoostCameraExposure`)
Runs once after `getUserMedia` succeeds. Applies `exposureMode/whiteBalanceMode: 'continuous'` if supported and surfaces a hint about EV range — but **deliberately does NOT set `exposureCompensation = max`** (that caused the bleached-scene bug). Software gain via the slider is the sole brightness control above the camera's auto-exposure point.

### Lifecycle invariants
- `visibilitychange` cancels the RAF when tab hidden and restarts the appropriate loop on resume. Don't add a new render loop without wiring it into this handler.
- `resize`/`orientationchange` is debounced and re-runs `startSensorMode()` so the centered message re-flows.
- `capturing` flag gates the shutter and suppresses the preview loop during accumulation — preserve this guard if changing long-exposure flow.

## Editing Rules Specific to This Repo

- One file. Use `Edit` against `index.html`; never split into modules unless explicitly asked.
- Korean UI strings — match the existing tone (e.g., "감도", "장노출 촬영 중", "하드웨어 깊이 센서를 지원하지 않습니다"). Don't translate to English.
- Browser-only `getImageData` work runs every frame; keep the inner loop allocation-free (reuse buffers, `| 0` truncation, no `Math.floor` in hot path).
- Any new visual stack must respect: **camera auto-exposure + software gain + soft tone-map**, in that order. Adding a fourth multiplicative stage will re-create the clipping regression.
