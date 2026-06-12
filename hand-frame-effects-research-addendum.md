# Hand Frame Live Effects — Research Addendum

## Purpose of this file

This is **not** a rewrite of `hand-frame-effects-spec.md`. It's a second file to hand to the
build AI **alongside** the spec (and alongside the earlier review notes, if you're including
those too). Everything below is organized as upgrades/options tied to specific section numbers
in the spec, so it can be applied incrementally — exactly in the spirit of the "build one
effect at a time" advice from the earlier review.

Two kinds of entries appear below:

- **[Drop-in]** — same inputs/outputs, same architecture, low risk, can replace the
  corresponding spec code directly.
- **[Optional / bigger change]** — genuinely better or more authentic, but either changes
  the visual result noticeably, adds a new dependency, or is a larger lift. Treat these as
  "nice to have after the base version works," per the incremental-build advice.

Every external API claim below was checked against current (June 2026) documentation —
versions and browser-support notes are as of this date.

---

## Part A — Cross-cutting engine & performance upgrades

These apply to the shared plumbing (Sections 5–14), not to any single effect, so they're
worth doing once, early.

### A1. `requestVideoFrameCallback` instead of the `lastVideoTime` poll — relates to Section 9 [Drop-in]

`HTMLVideoElement.requestVideoFrameCallback()` (rVFC) is a purpose-built replacement for the
"`requestAnimationFrame` + check `video.currentTime`" pattern in Section 9. It fires exactly
once per *new* decoded video frame, with a high-res timestamp and frame metadata handed to you
— so the `lastVideoTime` guard becomes unnecessary.

**Browser status:** rVFC reached *Baseline: newly available* in October 2024 (Chrome, Edge,
Safari, and Firefox 131+). Given the ~20 months since then, current Chrome/Edge/Firefox/Safari
all support it. One caveat: a Firefox-specific issue caps rVFC's effective interval at ~40ms on
some platforms — at a 30fps target (33.3ms/frame) this can occasionally throttle Firefox to
~25fps. Worth a feature-detect + fallback (it's ~6 lines):

```javascript
function startLoop() {
  // Steps 1–7 from the current Section 9 loop, unchanged, just extracted into a function:
  function renderFrame(now) {
    if (!isModelReady) return;

    ctx.save();
    ctx.scale(-1, 1);
    ctx.translate(-W, 0);
    ctx.drawImage(video, 0, 0, W, H);
    ctx.restore();

    const results = handLandmarker.detectForVideo(video, now);
    processHandResults(results);

    rect = (leftHandData && rightHandData) ? computeRect() : null;

    if (rect) {
      checkCollapseGesture();
    } else {
      collapseFrameCount = 0;
      gestureIsOpen = true;
    }

    if (rect && rect.width > 20 && rect.height > 20) {
      applyCurrentEffect();
    }

    drawOverlays();
  }

  if ('requestVideoFrameCallback' in HTMLVideoElement.prototype) {
    const onVideoFrame = (now) => {
      renderFrame(now);
      video.requestVideoFrameCallback(onVideoFrame);
    };
    video.requestVideoFrameCallback(onVideoFrame);
  } else {
    // Fallback: original rAF + currentTime guard for older browsers
    function rafLoop() {
      requestAnimationFrame(rafLoop);
      if (video.currentTime === lastVideoTime) return;
      lastVideoTime = video.currentTime;
      renderFrame(performance.now());
    }
    rafLoop();
  }
}
```

Note `detectForVideo(video, now)` — rVFC's `now` is already a `DOMHighResTimeStamp` matching
what `performance.now()` would give, so it can be passed straight through.

---

### A2. `willReadFrequently` on the main canvas context — relates to Section 5 [Drop-in]

```javascript
const ctx = canvas.getContext('2d', { willReadFrequently: true });
```

This one-line change is a near-perfect match for what this app does: `getCropImageData()` /
`putCropImageData()` (Section 14.1) run on `ctx` every single frame, for every effect. That's
exactly the "frequent readback" pattern this flag exists for — it tells the browser to keep
the canvas's backing store in CPU memory so `getImageData`/`putImageData` skip a GPU↔CPU
round trip, instead of the browser guessing (and potentially guessing wrong mid-session,
which causes a visible hitch when it re-allocates the canvas onto a different backend).

**Trade-off to know about:** this also means `ctx`'s *own* draw calls (the mirrored
`drawImage(video,...)`, and the direct-to-`ctx` strokes in Van Gogh/Cross-Hatch/Wool) lose GPU
compositing too. At 640×480 with fairly simple draw calls, the CPU path is comfortable — the
`getImageData`/`putImageData` savings on the crop (called every frame, for every effect) are
the dominant factor. If profiling later shows the direct-stroke effects (15.6–15.8) are
unexpectedly slow specifically because of this flag, an alternative is a **second, offscreen
canvas** with `willReadFrequently: true` used only for the `getCropImageData`/`putCropImageData`
round trip, composited back onto the main `ctx` via `drawImage` — but start with the simple
one-line version.

---

### A3. `ctx.filter` compositing for color/tone steps — relates to Sections 14.4, 15.4, 15.9 [Drop-in]

The spec's own `blurImageData()` helper (14.4) already proves the pattern: draw to a temp
canvas with `ctx.filter` set, then `getImageData` the result. Canvas 2D's `filter` property
supports the full CSS filter set — `blur`, `brightness`, `contrast`, `grayscale`,
`hue-rotate`, `invert`, `opacity`, `saturate`, `sepia`, `drop-shadow` — all GPU-composited,
and combinable in one string (`"sepia(50%) contrast(105%)"`).

The same pattern generalizes to a reusable helper:

```javascript
// Generalizes blurImageData (14.4) to any ctx.filter string.
function applyCanvasFilter(imageData, filterString) {
  const { width, height } = imageData;

  const src = document.createElement('canvas');
  src.width = width; src.height = height;
  src.getContext('2d').putImageData(imageData, 0, 0);

  const dst = document.createElement('canvas');
  dst.width = width; dst.height = height;
  const dstCtx = dst.getContext('2d');
  dstCtx.filter = filterString;
  dstCtx.drawImage(src, 0, 0);
  dstCtx.filter = 'none';

  return dstCtx.getImageData(0, 0, width, height);
}
```

Two places this directly replaces a per-pixel loop:

- **Anime (15.4) Step 1** — the `rgbToHsl` → scale S by 1.6 → `hslToRgb` round trip becomes:
  ```javascript
  const boosted = applyCanvasFilter(src, 'saturate(160%)').data;
  ```
  No more per-pixel HSL conversion at all for this step.

- **Old Master (15.9) Step 1** — the warm-grade channel multiplies can become:
  ```javascript
  const graded = applyCanvasFilter(src, 'sepia(45%) contrast(108%) saturate(115%)').data;
  ```

**Caveat:** the browser's `saturate()`/`sepia()` use their own standard matrices, which are
*close to* but not numerically identical to HSL-saturation-scaling or the spec's ad-hoc
channel multipliers. The visual result will shift slightly — that's a styling knob to retune
(percentages above are starting points), not a correctness bug. If exact reproducibility of
the spec's current look matters more than the perf win, keep the per-pixel version for that
specific effect and use `ctx.filter` only where you're also intentionally tweaking the look
(e.g., alongside the Old Master upgrades in B9 below).

---

### A4. `globalCompositeOperation = 'color-dodge'` for Pencil Sketch — relates to Sections 14.4, 15.1 [Drop-in]

This one is a precise match, not just "close enough." MDN describes the canvas `color-dodge`
blend mode as *"divides the bottom layer by the inverted top layer"* — i.e.
`result = backdrop / (1 - source)`. The spec's Section 15.1 Step 4 formula is
`v = g * 255 / (255 - bv)`, which in normalized terms is exactly `g / (1 - bv)`. Same formula.

So the entire Step 4 per-pixel loop can become two `drawImage` calls plus one `getImageData`:

```javascript
// Replaces Section 15.1, Step 4
function colorDodgeBlend(grayID, blurredInvertedID) {
  const { width, height } = grayID;

  const bottom = document.createElement('canvas');
  bottom.width = width; bottom.height = height;
  bottom.getContext('2d').putImageData(grayID, 0, 0);

  const top = document.createElement('canvas');
  top.width = width; top.height = height;
  top.getContext('2d').putImageData(blurredInvertedID, 0, 0);

  const out = document.createElement('canvas');
  out.width = width; out.height = height;
  const outCtx = out.getContext('2d');
  outCtx.drawImage(bottom, 0, 0);
  outCtx.globalCompositeOperation = 'color-dodge';
  outCtx.drawImage(top, 0, 0);
  outCtx.globalCompositeOperation = 'source-over';

  return outCtx.getImageData(0, 0, width, height);
}

// Section 15.1 then ends with:
putCropImageData(colorDodgeBlend(grayID, blurred));
```

`'color-dodge'` is one of the 26 standard `globalCompositeOperation` values and has been
supported across Chrome/Firefox/Edge/Safari for years — no compatibility concern.

---

### A5. MediaPipe version & an optional `GestureRecognizer` swap — relates to Section 8 [mostly Drop-in, one Optional part]

**Version check (informational):** `@mediapipe/tasks-vision` is currently at **0.10.35** on
npm/jsdelivr (the spec pins **0.10.14**). The `HandLandmarker` surface this spec uses —
`createFromOptions`, `detectForVideo`, `baseOptions.{modelAssetPath, delegate}`, `numHands`,
`min*Confidence` — is unchanged across that range, so bumping the pin is low-risk if you want
current bugfixes. That said, 0.10.14 was already confirmed live on jsdelivr with its `/wasm`
folder intact, so there's **no requirement** to change it — the spec's own rationale ("avoid
API drift from `@latest`") still holds either way.

**[Optional / bigger change] Swap `HandLandmarker` → `GestureRecognizer`:** MediaPipe's
Gesture Recognizer task returns the *same* `result.landmarks` / `result.handedness` shape as
HandLandmarker, **plus** `result.gestures[i][0].categoryName` per hand — one of
`Open_Palm`, `Closed_Fist`, `Pointing_Up`, `Thumb_Up`, `Thumb_Down`, `Victory`, `ILoveYou`, or
`None`.

Why this is interesting *for this specific project*: the status overlay (Section 16) already
tells the user **"Show both hands with palm open"** — but Section 11's `computeRect()` will
happily build a rectangle from a closed fist's thumb/index tips too, since it only looks at
two landmark positions. With the Gesture Recognizer, you could gate rect computation on both
hands reporting `Open_Palm`, making that on-screen instruction *actually load-bearing* instead
of just a hint.

```javascript
import {
  GestureRecognizer,
  FilesetResolver,
} from 'https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.35/vision_bundle.mjs';

gestureRecognizer = await GestureRecognizer.createFromOptions(vision, {
  baseOptions: {
    modelAssetPath:
      'https://storage.googleapis.com/mediapipe-models/gesture_recognizer/gesture_recognizer/float16/1/gesture_recognizer.task',
    delegate: 'GPU',
  },
  runningMode: 'VIDEO',
  numHands: 2,
  minHandDetectionConfidence: 0.5,
  minHandPresenceConfidence: 0.5,
  minTrackingConfidence: 0.5,
});

// result.gestures[i][0].categoryName → 'Open_Palm' | 'Closed_Fist' | ... | 'None'
```

**Caveat:** gesture *classification* is its own model with its own confidence/jitter
characteristics, separate from landmark tracking. If you gate `rect` strictly on
`Open_Palm` for both hands and the classifier is even slightly jumpy, the rectangle could
flicker in and out even while the user holds a steady pose. Safer framing: keep the current
geometry-only `rect` as the baseline, and only use the gesture signal as a *negative* filter
— e.g., suppress the rect (and show a more specific hint, "open your hand more") only when a
hand is confidently `Closed_Fist` or `Pointing_Up`, rather than requiring `Open_Palm`
positively. Treat this whole item as a post-MVP enhancement.

---

### A6. GPU shader path (WebGL2 / WebGPU) — relates to Section 18, all of Section 15 [Optional / bigger change]

If, after the per-effect upgrades in Part B, the Section 18 budget still doesn't hold up on
real hardware — Oil Paint is the most likely culprit, with Watercolor and Cartoon next —
the per-pixel effects are exactly the workload that benefits from a fragment-shader rewrite.
Kuwahara-family filters (B2 below) in particular show up repeatedly in "painterly shader"
writeups (Maxime Heckel's WebGL painterly-shaders piece is a good deep dive) running at full
resolution in real time — something a JS double-loop on a 200×200 crop can approach but
which a shader does with headroom to spare.

**Practical recommendation: WebGL2, not WebGPU, as the first step.** WebGL2 has ~99% browser
support going back to ~2017, and the setup for this use case is small: one offscreen
`<canvas>`, one fullscreen-quad fragment shader per effect, the crop's `ImageData` uploaded
via `texImage2D`, result read back (or, better, `drawImage`'d straight from the WebGL canvas
onto the main canvas at `(rect.x, rect.y)` — avoiding a `getImageData` round trip entirely for
that effect).

**WebGPU status as of mid-2026** (for awareness, not as the first move): Chrome/Edge have had
it since version 113 (2023); Safari since 26.0; Firefox shipped it on Windows at 141 and on
Apple-Silicon macOS at 145. Linux and Android support is still rolling out through 2026.
Feature-detect via `'gpu' in navigator` and fall back to WebGL2 or the CPU path. Given the
setup overhead (WGSL shaders, explicit pipeline/bind-group setup) is meaningfully higher than
WebGL2 for a single-file project, this is best treated as a "future" option rather than part
of the initial build.

**Suggested approach if you go here at all:** after all 9 CPU effects work (per the
incremental-build plan), pick the single most expensive one — Oil Paint — and reimplement
*just that one* as a WebGL2 Kuwahara shader (B2 has the exact algorithm) as a proof of
concept, with the existing CPU version kept as a fallback if `canvas.getContext('webgl2')`
returns null. Don't attempt to convert all 9 effects in one pass.

---

### A7. OffscreenCanvas + Worker for MediaPipe — relates to Sections 8–9 [Optional / bigger change, lower priority]

MediaPipe's web runtime detects `OffscreenCanvas` automatically and can run its internal
WebGL graph inside a Worker, which would move the ~5–15ms `detectForVideo` cost off the main
thread entirely.

**Caveat that matters for a single-file, no-build-step project:** there's a known issue where
`@mediapipe/tasks-vision` running inside a Worker on Safari/iOS calls `document.createElement`
in a code path that assumes `OffscreenCanvas` implies "no DOM," throwing `Can't find variable:
document`. The target browsers here are desktop Chrome/Firefox/Edge, where this is less likely
to bite — but it's the kind of thing that's easy to lose an afternoon to.

Given that detection is already only 5–15ms of a 30–42ms budget (Section 18), and rVFC (A1)
plus `willReadFrequently` (A2) likely recover more headroom for less risk, this is the lowest
-priority item in Part A. If pursued: the main thread keeps the visible `<canvas>` and does the
cheap mirrored `drawImage` every frame; a Worker owns an `OffscreenCanvas`, runs
`HandLandmarker`/`GestureRecognizer` on `VideoFrame`s (via `requestVideoFrameCallback`'s
`metadata` or a `MediaStreamTrackProcessor`), and `postMessage`s back just
`{ landmarks, handedness }` — small JSON, not pixel data.

