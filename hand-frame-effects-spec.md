# Hand Frame Live Effects — Complete Implementation Specification

---

## 1. Project Overview

A **single HTML file**, no server, no build tools, no npm install. Open it in Chrome/Firefox/Edge and it works.

The app does the following:
1. Accesses the user's webcam via `getUserMedia`
2. Tracks both hands in real time using MediaPipe Hand Landmarker (runs in-browser via WebAssembly)
3. Constructs a live rectangle whose four corners are defined by:
   - Left hand: **thumb tip** and **index tip**
   - Right hand: **thumb tip** and **index tip**
4. Applies a stylized visual effect **only inside that rectangle** on the live camera feed, every frame
5. When the user brings their hands together so the rectangle's width collapses to near-zero, the effect **cycles to the next one**
6. The video feed is **horizontally mirrored** (selfie view — natural for the user)

---

## 2. Deliverable Format

- **One file:** `index.html`
- All JavaScript inside `<script type="module">` (required for ES module imports)
- All CSS inside `<style>`
- Opens directly in browser from filesystem (`file://`) — no localhost required
- Target browsers: Chrome 100+, Firefox 100+, Edge 100+ (desktop). Mobile optional.

---

## 3. Technology Stack

| Concern | Library | Source |
|---|---|---|
| Hand tracking | MediaPipe Tasks Vision `0.10.14` | `https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.14/vision_bundle.mjs` |
| Hand landmarker model | Downloaded at runtime (~8 MB, once) | `https://storage.googleapis.com/mediapipe-models/hand_landmarker/hand_landmarker/float16/latest/hand_landmarker.task` |
| WASM runtime files | Served by jsdelivr | `https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.14/wasm` |
| Rendering | Native Canvas 2D API | Built into browser |
| Camera | Native `getUserMedia` | Built into browser |

**No other dependencies.** All visual effects are pure JavaScript using Canvas 2D pixel operations and drawing primitives.

---

## 4. HTML Structure

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Hand Frame Effects</title>
  <style>
    /* CSS defined in Section 4.1 */
  </style>
</head>
<body>
  <div id="container">
    <!-- video is hidden from the user but kept in the DOM — MediaPipe requires it -->
    <video id="video" autoplay muted playsinline></video>
    <!-- canvas shows everything: mirrored video + effect + overlays -->
    <canvas id="canvas"></canvas>
    <!-- top-center status message: loading, errors, hints -->
    <div id="status">Loading model…</div>
    <!-- bottom-center effect name label, briefly shown after each switch -->
    <div id="effect-label"></div>
  </div>
  <script type="module">
    /* All JS here — Sections 5 onward */
  </script>
</body>
</html>
```

### 4.1 CSS

```css
* { margin: 0; padding: 0; box-sizing: border-box; }

body {
  background: #111;
  display: flex;
  align-items: center;
  justify-content: center;
  height: 100vh;
  overflow: hidden;
  font-family: monospace;
}

#container {
  position: relative;
  width: 640px;
  height: 480px;
}

/* IMPORTANT: visibility:hidden, NOT display:none.
   MediaPipe needs the video element in the DOM and visible to the render engine. */
#video {
  position: absolute;
  width: 100%;
  height: 100%;
  object-fit: cover;
  visibility: hidden;
}

#canvas {
  position: absolute;
  width: 100%;
  height: 100%;
  cursor: none;
}

#status {
  position: absolute;
  top: 14px;
  left: 50%;
  transform: translateX(-50%);
  color: #fff;
  font-size: 13px;
  background: rgba(0, 0, 0, 0.55);
  padding: 5px 14px;
  border-radius: 12px;
  pointer-events: none;
  transition: opacity 0.3s;
  white-space: nowrap;
}

#effect-label {
  position: absolute;
  bottom: 22px;
  left: 50%;
  transform: translateX(-50%);
  color: #fff;
  font-size: 17px;
  font-weight: bold;
  letter-spacing: 3px;
  background: rgba(0, 0, 0, 0.6);
  padding: 7px 20px;
  border-radius: 16px;
  pointer-events: none;
  opacity: 0;
  transition: opacity 0.4s;
  white-space: nowrap;
}
```

---

## 5. Global State Variables

Declare all of these at the top of the `<script type="module">` block, before any functions.

```javascript
// ── MediaPipe ─────────────────────────────────────────────────────────────────
let handLandmarker = null;   // MediaPipe HandLandmarker instance, assigned in setupMediaPipe()
let isModelReady   = false;  // set to true after model finishes loading

// ── DOM references ────────────────────────────────────────────────────────────
const video         = document.getElementById('video');
const canvas        = document.getElementById('canvas');
const ctx           = canvas.getContext('2d');
const statusEl      = document.getElementById('status');
const effectLabelEl = document.getElementById('effect-label');

// ── Canvas dimensions ─────────────────────────────────────────────────────────
// Assigned in setupCamera() once the video stream is ready.
let W = 0;  // canvas.width  (matches video.videoWidth)
let H = 0;  // canvas.height (matches video.videoHeight)

// ── Per-frame hand data ───────────────────────────────────────────────────────
// Each is null when that hand is not detected.
// When set, shape: { thumbTip: {x, y}, indexTip: {x, y} }
// Coordinates are in PIXEL space, already mirrored for the flipped display.
let leftHandData  = null;  // hand appearing on the LEFT  side of the mirrored screen
let rightHandData = null;  // hand appearing on the RIGHT side of the mirrored screen

// ── Rectangle (computed every frame from the four tip points) ─────────────────
// null when fewer than 2 hands are detected.
// Shape: { x, y, width, height } — integer pixels, clamped to canvas bounds.
let rect = null;

// ── Gesture detection ─────────────────────────────────────────────────────────
const COLLAPSE_THRESHOLD_PX  = 50;  // rect.width below this → "closing"
const COLLAPSE_FRAMES_NEEDED = 18;  // must stay below threshold this many consecutive frames to trigger
const REOPEN_THRESHOLD_PX    = 80;  // rect.width above this → reset to open state

let collapseFrameCount = 0;     // counts consecutive "closing" frames
let gestureIsOpen      = true;  // false while in collapsed state, prevents repeated triggers

// ── Effects ───────────────────────────────────────────────────────────────────
const EFFECT_NAMES = [
  'PENCIL SKETCH',   // 0
  'OIL PAINT',       // 1
  'CARTOON',         // 2
  'ANIME',           // 3
  'WATERCOLOR',      // 4
  'VAN GOGH',        // 5
  'CROSS-HATCH',     // 6
  'WOOL',            // 7
  'OLD MASTER',      // 8
];
let currentEffectIndex = 0;

// ── Effect label auto-hide timer ──────────────────────────────────────────────
let effectLabelTimer = null;

// ── Animation loop ────────────────────────────────────────────────────────────
let lastVideoTime = -1;  // guards against processing the same video frame twice
```

---

## 6. Startup Sequence

This is the entry point. Call `init()` at the bottom of the module script.

```javascript
async function init() {
  try {
    setStatus('Loading hand tracking model…');
    await setupMediaPipe();

    setStatus('Starting camera…');
    await setupCamera();

    setStatus('');
    isModelReady = true;
    startLoop();
  } catch (err) {
    setStatus('Error: ' + err.message);
    console.error(err);
  }
}

init();
```

---

## 7. Camera Setup

```javascript
async function setupCamera() {
  const stream = await navigator.mediaDevices.getUserMedia({
    video: {
      width:      { ideal: 640 },
      height:     { ideal: 480 },
      facingMode: 'user',         // front-facing camera
      frameRate:  { ideal: 30 },
    },
    audio: false,
  });

  video.srcObject = stream;

  // Wait for metadata (videoWidth / videoHeight become available)
  await new Promise((resolve) => {
    video.onloadedmetadata = () => {
      video.play();
      resolve();
    };
  });

  // Set canvas to exactly match the video dimensions
  W = canvas.width  = video.videoWidth;
  H = canvas.height = video.videoHeight;
}
```

**Error:** If `getUserMedia` throws (permission denied, no camera), the error propagates to `init()` and is shown in `#status`.

---

## 8. MediaPipe Setup

```javascript
import {
  HandLandmarker,
  FilesetResolver,
} from 'https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.14/vision_bundle.mjs';

async function setupMediaPipe() {
  const vision = await FilesetResolver.forVisionTasks(
    'https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.14/wasm'
  );

  handLandmarker = await HandLandmarker.createFromOptions(vision, {
    baseOptions: {
      modelAssetPath:
        'https://storage.googleapis.com/mediapipe-models/hand_landmarker/hand_landmarker/float16/latest/hand_landmarker.task',
      delegate: 'GPU',   // uses WebGL if available; silently falls back to CPU
    },
    runningMode:                 'VIDEO',  // VIDEO mode enables temporal tracking across frames
    numHands:                    2,
    minHandDetectionConfidence:  0.5,
    minHandPresenceConfidence:   0.5,
    minTrackingConfidence:       0.5,
  });
}
```

### 8.1 MediaPipe Result Structure (reference)

```javascript
// result = handLandmarker.detectForVideo(video, timestamp)
//
// result.landmarks  → Array, length 0–2 (one entry per detected hand)
//   result.landmarks[i]  → Array of 21 objects: { x, y, z }
//     x, y  → normalized float in range [0.0 … 1.0], origin = top-left
//     z     → depth estimate (ignore for this project)
//
// Landmark indices used in this project:
//   4  = THUMB_TIP
//   8  = INDEX_FINGER_TIP
//   0  = WRIST  (used only to assign left/right)
//
// result.handedness  → Array parallel to result.landmarks
//   result.handedness[i][0].categoryName  → "Left" or "Right"
//   ⚠️  DO NOT use handedness for left/right assignment.
//       MediaPipe's labels are from the perspective of the unmirrored image,
//       which is the opposite of what the user sees in the mirrored display.
//       Use wrist x-position instead (see Section 10).
```

---

## 9. Main Render Loop

```javascript
function startLoop() {
  function loop() {
    requestAnimationFrame(loop);

    // Skip if the video hasn't advanced to a new frame since last tick
    if (video.currentTime === lastVideoTime) return;
    lastVideoTime = video.currentTime;

    if (!isModelReady) return;

    // ── Step 1: Draw the video to the canvas, horizontally mirrored ───────────
    ctx.save();
    ctx.scale(-1, 1);
    ctx.translate(-W, 0);
    ctx.drawImage(video, 0, 0, W, H);
    ctx.restore();

    // ── Step 2: Run MediaPipe hand detection on the (unmirrored) video element ─
    const results = handLandmarker.detectForVideo(video, performance.now());

    // ── Step 3: Parse landmarks into leftHandData / rightHandData ─────────────
    processHandResults(results);

    // ── Step 4: Compute the rectangle ─────────────────────────────────────────
    rect = (leftHandData && rightHandData) ? computeRect() : null;

    // ── Step 5: Check collapse gesture ────────────────────────────────────────
    if (rect) {
      checkCollapseGesture();
    } else {
      // Reset gesture when hands disappear
      collapseFrameCount = 0;
      gestureIsOpen = true;
    }

    // ── Step 6: Apply the current effect inside the rectangle ─────────────────
    if (rect && rect.width > 20 && rect.height > 20) {
      applyCurrentEffect();
    }

    // ── Step 7: Draw overlays (rectangle border, corner dots, hint text) ──────
    drawOverlays();
  }

  loop();
}
```

---

## 10. Hand Data Processing

```javascript
function processHandResults(results) {
  leftHandData  = null;
  rightHandData = null;

  if (!results.landmarks || results.landmarks.length < 2) return;

  // Convert the first two detected hands to { thumbTip, indexTip, wrist } in PIXEL space
  const hands = results.landmarks.slice(0, 2).map((lm) => ({
    thumbTip:  mirroredPoint(lm[4]),   // landmark 4 = THUMB_TIP
    indexTip:  mirroredPoint(lm[8]),   // landmark 8 = INDEX_FINGER_TIP
    wrist:     mirroredPoint(lm[0]),   // landmark 0 = WRIST — used for left/right assignment only
  }));

  // Assign left / right by wrist x-position in MIRRORED pixel space.
  // Lower x → visual left of screen.  Higher x → visual right.
  if (hands[0].wrist.x <= hands[1].wrist.x) {
    leftHandData  = hands[0];
    rightHandData = hands[1];
  } else {
    leftHandData  = hands[1];
    rightHandData = hands[0];
  }
}

// Convert a normalized MediaPipe landmark {x, y} to pixel coordinates,
// applying horizontal mirror to match the flipped canvas.
// Mirror formula: pixelX = (1 - landmark.x) * W
function mirroredPoint(landmark) {
  return {
    x: (1 - landmark.x) * W,
    y:        landmark.y  * H,
  };
}
```

**Why mirror x:** The canvas is drawn with `ctx.scale(-1, 1)` which flips it horizontally. To align landmark positions with the flipped display, we also flip x: `1 - landmark.x`.

---

## 11. Rectangle Computation

```javascript
function computeRect() {
  const pts = [
    leftHandData.thumbTip,
    leftHandData.indexTip,
    rightHandData.thumbTip,
    rightHandData.indexTip,
  ];

  const xs = pts.map((p) => p.x);
  const ys = pts.map((p) => p.y);

  const rawX = Math.min(...xs);
  const rawY = Math.min(...ys);
  const rawW = Math.max(...xs) - rawX;
  const rawH = Math.max(...ys) - rawY;

  // Clamp to canvas bounds (prevents negative width/height, no out-of-bounds getImageData)
  const x = Math.max(0, Math.round(rawX));
  const y = Math.max(0, Math.round(rawY));
  const w = Math.min(W - x, Math.round(rawW));
  const h = Math.min(H - y, Math.round(rawH));

  return { x, y, width: w, height: h };
}
```

**`rect.width` approaching 0 means the hands are converging — that is the collapse gesture signal.**

---

## 12. Gesture Detection (Collapse / Reopen)

```javascript
function checkCollapseGesture() {
  if (rect.width < COLLAPSE_THRESHOLD_PX) {
    collapseFrameCount++;

    // Only trigger once per gesture (gestureIsOpen prevents re-triggering while held)
    if (collapseFrameCount >= COLLAPSE_FRAMES_NEEDED && gestureIsOpen) {
      gestureIsOpen = false;
      cycleEffect();
    }
  } else if (rect.width > REOPEN_THRESHOLD_PX) {
    // Hands have opened wide again — ready for the next collapse
    collapseFrameCount = 0;
    gestureIsOpen = true;
  }
  // If width is between 50 and 80: do nothing (hysteresis zone — prevents rapid re-firing)
}
```

**Constants explained:**
- `COLLAPSE_THRESHOLD_PX = 50` — entering: width below this starts the counter
- `COLLAPSE_FRAMES_NEEDED = 18` — ~0.6 seconds at 30fps; filters accidental quick hand movements
- `REOPEN_THRESHOLD_PX = 80` — exiting: width must exceed this to re-arm; gap of 30px between thresholds is intentional

---

## 13. Effect System

```javascript
function cycleEffect() {
  currentEffectIndex = (currentEffectIndex + 1) % EFFECT_NAMES.length;
  showEffectLabel(EFFECT_NAMES[currentEffectIndex]);
}

function showEffectLabel(name) {
  effectLabelEl.textContent = name;
  effectLabelEl.style.opacity = '1';
  clearTimeout(effectLabelTimer);
  effectLabelTimer = setTimeout(() => {
    effectLabelEl.style.opacity = '0';
  }, 1800);
}

function applyCurrentEffect() {
  switch (currentEffectIndex) {
    case 0: applyPencilSketch(); break;
    case 1: applyOilPaint();     break;
    case 2: applyCartoon();      break;
    case 3: applyAnime();        break;
    case 4: applyWatercolor();   break;
    case 5: applyVanGogh();      break;
    case 6: applyCrossHatch();   break;
    case 7: applyWool();         break;
    case 8: applyOldMaster();    break;
  }
}
```

---

## 14. Shared Helper Functions

Define all of these before the effect implementations.

### 14.1 Get / Put Crop ImageData

```javascript
function getCropImageData() {
  return ctx.getImageData(rect.x, rect.y, rect.width, rect.height);
}

function putCropImageData(imageData) {
  ctx.putImageData(imageData, rect.x, rect.y);
}
```

### 14.2 Grayscale Array (from ImageData)

Returns a `Float32Array` of luminance values, one per pixel, range 0–255.

```javascript
function toGrayscaleArray(imageData) {
  const { data, width, height } = imageData;
  const gray = new Float32Array(width * height);
  for (let i = 0; i < width * height; i++) {
    const b = i * 4;
    gray[i] = 0.299 * data[b] + 0.587 * data[b + 1] + 0.114 * data[b + 2];
  }
  return gray;
}
```

### 14.3 Apply Grayscale In-Place

Modifies the ImageData's `data` array directly.

```javascript
function applyGrayscaleInPlace(imageData) {
  const d = imageData.data;
  for (let i = 0; i < d.length; i += 4) {
    const g = 0.299 * d[i] + 0.587 * d[i + 1] + 0.114 * d[i + 2];
    d[i] = d[i + 1] = d[i + 2] = g;
  }
}
```

### 14.4 Blur ImageData (via Canvas filter — uses GPU if available)

Creates and discards temporary canvases. Returns a **new** ImageData; does not modify input.

```javascript
function blurImageData(imageData, radiusPx) {
  const { width, height } = imageData;

  const src = document.createElement('canvas');
  src.width = width; src.height = height;
  src.getContext('2d').putImageData(imageData, 0, 0);

  const dst = document.createElement('canvas');
  dst.width = width; dst.height = height;
  const dstCtx = dst.getContext('2d');
  dstCtx.filter = `blur(${radiusPx}px)`;
  dstCtx.drawImage(src, 0, 0);
  dstCtx.filter = 'none';

  return dstCtx.getImageData(0, 0, width, height);
}
```

### 14.5 Box Blur (CPU, returns new ImageData)

Used when multiple sequential blur passes are needed (Watercolor).

```javascript
function boxBlur(imageData, radius) {
  const { data, width, height } = imageData;
  const out = new Uint8ClampedArray(data.length);

  for (let y = 0; y < height; y++) {
    for (let x = 0; x < width; x++) {
      let r = 0, g = 0, b = 0, count = 0;
      for (let dy = -radius; dy <= radius; dy++) {
        for (let dx = -radius; dx <= radius; dx++) {
          const nx = x + dx, ny = y + dy;
          if (nx >= 0 && nx < width && ny >= 0 && ny < height) {
            const i = (ny * width + nx) * 4;
            r += data[i]; g += data[i + 1]; b += data[i + 2];
            count++;
          }
        }
      }
      const i = (y * width + x) * 4;
      out[i]     = r / count;
      out[i + 1] = g / count;
      out[i + 2] = b / count;
      out[i + 3] = data[i + 3];
    }
  }
  return new ImageData(out, width, height);
}
```

### 14.6 Sobel Gradient

Returns `{ gx, gy, magnitude }` — all `Float32Array` of length `width * height`.
Pixels on the 1-pixel border are left at 0 (no off-by-one errors).

```javascript
function sobelGradient(grayArray, width, height) {
  const gx  = new Float32Array(width * height);
  const gy  = new Float32Array(width * height);
  const mag = new Float32Array(width * height);

  for (let y = 1; y < height - 1; y++) {
    for (let x = 1; x < width - 1; x++) {
      const tl = grayArray[(y - 1) * width + (x - 1)];
      const tc = grayArray[(y - 1) * width +  x     ];
      const tr = grayArray[(y - 1) * width + (x + 1)];
      const cl = grayArray[ y      * width + (x - 1)];
      const cr = grayArray[ y      * width + (x + 1)];
      const bl = grayArray[(y + 1) * width + (x - 1)];
      const bc = grayArray[(y + 1) * width +  x     ];
      const br = grayArray[(y + 1) * width + (x + 1)];

      const idx = y * width + x;
      gx[idx]  = (-tl - 2 * cl - bl) + (tr + 2 * cr + br);
      gy[idx]  = (-tl - 2 * tc - tr) + (bl + 2 * bc + br);
      mag[idx] = Math.sqrt(gx[idx] * gx[idx] + gy[idx] * gy[idx]);
    }
  }
  return { gx, gy, magnitude: mag };
}
```

### 14.7 RGB ↔ HSL Conversion (used by Anime effect)

```javascript
function rgbToHsl(r, g, b) {
  r /= 255; g /= 255; b /= 255;
  const max = Math.max(r, g, b), min = Math.min(r, g, b);
  const l = (max + min) / 2;
  if (max === min) return [0, 0, l];
  const d = max - min;
  const s = l > 0.5 ? d / (2 - max - min) : d / (max + min);
  let h;
  switch (max) {
    case r: h = ((g - b) / d + (g < b ? 6 : 0)) / 6; break;
    case g: h = ((b - r) / d + 2) / 6; break;
    default: h = ((r - g) / d + 4) / 6;
  }
  return [h, s, l];
}

function hslToRgb(h, s, l) {
  if (s === 0) {
    const v = Math.round(l * 255);
    return [v, v, v];
  }
  const hue2rgb = (p, q, t) => {
    if (t < 0) t += 1;
    if (t > 1) t -= 1;
    if (t < 1 / 6) return p + (q - p) * 6 * t;
    if (t < 1 / 2) return q;
    if (t < 2 / 3) return p + (q - p) * (2 / 3 - t) * 6;
    return p;
  };
  const q = l < 0.5 ? l * (1 + s) : l + s - l * s;
  const p = 2 * l - q;
  return [
    Math.round(hue2rgb(p, q, h + 1 / 3) * 255),
    Math.round(hue2rgb(p, q, h)         * 255),
    Math.round(hue2rgb(p, q, h - 1 / 3) * 255),
  ];
}
```

---

## 15. Effect Implementations

All effects read `rect` (global) and call `getCropImageData()` / `putCropImageData()`.
Effects that draw strokes directly on the canvas (Van Gogh, Cross-Hatch, Wool) always wrap all canvas calls in `ctx.save()` / `ctx.restore()`.

---

### 15.1 Pencil Sketch

**Visual:** grayscale line drawing that looks like pencil on white paper.

**Algorithm:** grayscale → invert → blur → color-dodge blend back.

```javascript
function applyPencilSketch() {
  const { width, height } = rect;
  const src = getCropImageData();

  // Step 1: Make a grayscale copy
  const grayID = new ImageData(new Uint8ClampedArray(src.data), width, height);
  applyGrayscaleInPlace(grayID);

  // Step 2: Invert the grayscale copy
  const invID = new ImageData(new Uint8ClampedArray(grayID.data), width, height);
  const inv   = invID.data;
  for (let i = 0; i < inv.length; i += 4) {
    inv[i] = inv[i + 1] = inv[i + 2] = 255 - inv[i];
  }

  // Step 3: Gaussian-blur the inverted copy
  const blurred = blurImageData(invID, 3);

  // Step 4: Color-dodge blend: result = clamp( gray * 255 / (255 - blurred), 0, 255 )
  const out  = new Uint8ClampedArray(src.data.length);
  const gray = grayID.data;
  const blur = blurred.data;
  for (let i = 0; i < out.length; i += 4) {
    const g  = gray[i];
    const bv = blur[i];
    const v  = bv === 255 ? 255 : Math.min(255, (g * 255) / (255 - bv));
    out[i] = out[i + 1] = out[i + 2] = v;
    out[i + 3] = 255;
  }

  putCropImageData(new ImageData(out, width, height));
}
```

---

### 15.2 Oil Paint

**Visual:** thick, chunky brush strokes; colors merge into blobs like impasto oil.

**Algorithm:** for each output pixel, sample a circular neighborhood, bin colors by brightness, output the average color of the most-populated bin.

```javascript
function applyOilPaint() {
  const { width, height } = rect;
  const src     = getCropImageData();
  const srcData = src.data;

  const RADIUS = 4;   // neighborhood radius; increase for larger/chunkier strokes
  const LEVELS = 20;  // number of intensity quantization levels

  const out = new Uint8ClampedArray(srcData.length);

  for (let y = 0; y < height; y++) {
    for (let x = 0; x < width; x++) {
      const counts = new Int32Array(LEVELS);
      const rSums  = new Float32Array(LEVELS);
      const gSums  = new Float32Array(LEVELS);
      const bSums  = new Float32Array(LEVELS);

      for (let dy = -RADIUS; dy <= RADIUS; dy++) {
        for (let dx = -RADIUS; dx <= RADIUS; dx++) {
          if (dx * dx + dy * dy > RADIUS * RADIUS) continue; // circular mask
          const nx = x + dx, ny = y + dy;
          if (nx < 0 || nx >= width || ny < 0 || ny >= height) continue;

          const i = (ny * width + nx) * 4;
          const intensity = Math.floor(
            ((srcData[i] + srcData[i + 1] + srcData[i + 2]) / 3) *
            (LEVELS - 1) / 255
          );
          counts[intensity]++;
          rSums[intensity] += srcData[i];
          gSums[intensity] += srcData[i + 1];
          bSums[intensity] += srcData[i + 2];
        }
      }

      // Find the most-populated intensity bin
      let maxCount = 0, maxIdx = 0;
      for (let l = 0; l < LEVELS; l++) {
        if (counts[l] > maxCount) { maxCount = counts[l]; maxIdx = l; }
      }

      const oi = (y * width + x) * 4;
      out[oi]     = rSums[maxIdx] / maxCount;
      out[oi + 1] = gSums[maxIdx] / maxCount;
      out[oi + 2] = bSums[maxIdx] / maxCount;
      out[oi + 3] = 255;
    }
  }

  putCropImageData(new ImageData(out, width, height));
}
```

---

### 15.3 Cartoon

**Visual:** flat, posterized color regions with dark outlines, like a comic book panel.

```javascript
function applyCartoon() {
  const { width, height } = rect;
  const src = getCropImageData();

  // Step 1: Blur source to reduce noise before quantization
  const blurred = boxBlur(src, 2);

  // Step 2: Quantize each channel to 6 levels (0, 42, 84, 126, 168, 210)
  const quantized = new Uint8ClampedArray(blurred.data.length);
  for (let i = 0; i < blurred.data.length; i += 4) {
    quantized[i]     = Math.floor(blurred.data[i]     / 42) * 42;
    quantized[i + 1] = Math.floor(blurred.data[i + 1] / 42) * 42;
    quantized[i + 2] = Math.floor(blurred.data[i + 2] / 42) * 42;
    quantized[i + 3] = 255;
  }

  // Step 3: Edge detection on the ORIGINAL (not blurred) grayscale
  const gray = toGrayscaleArray(src);
  const { magnitude } = sobelGradient(gray, width, height);
  const EDGE_THRESHOLD = 60;

  // Step 4: Where edge magnitude > threshold, override with black
  const out = new Uint8ClampedArray(quantized);
  for (let i = 0; i < width * height; i++) {
    if (magnitude[i] > EDGE_THRESHOLD) {
      out[i * 4]     = 0;
      out[i * 4 + 1] = 0;
      out[i * 4 + 2] = 0;
    }
  }

  putCropImageData(new ImageData(out, width, height));
}
```

---

### 15.4 Anime

**Visual:** vibrant flat colors, bold outlines, and manga-style halftone dot shadow areas.

```javascript
function applyAnime() {
  const { width, height } = rect;
  const src = getCropImageData();

  // Step 1: Boost saturation (convert to HSL, scale S by 1.6, convert back)
  const boosted = new Uint8ClampedArray(src.data.length);
  for (let i = 0; i < src.data.length; i += 4) {
    const [h, s, l] = rgbToHsl(src.data[i], src.data[i + 1], src.data[i + 2]);
    const [nr, ng, nb] = hslToRgb(h, Math.min(1, s * 1.6), l);
    boosted[i] = nr; boosted[i + 1] = ng; boosted[i + 2] = nb; boosted[i + 3] = 255;
  }

  // Step 2: Quantize to 8 levels per channel (crisper than cartoon)
  const quantized = new Uint8ClampedArray(boosted.length);
  for (let i = 0; i < boosted.length; i += 4) {
    quantized[i]     = Math.floor(boosted[i]     / 32) * 32;
    quantized[i + 1] = Math.floor(boosted[i + 1] / 32) * 32;
    quantized[i + 2] = Math.floor(boosted[i + 2] / 32) * 32;
    quantized[i + 3] = 255;
  }

  // Step 3: Edge detection (lower threshold than cartoon = thicker lines)
  const gray = toGrayscaleArray(src);
  const { magnitude } = sobelGradient(gray, width, height);
  const EDGE_THRESHOLD = 40;

  // Step 4: Screen-tone dot pattern in dark areas + edge overlay
  const out = new Uint8ClampedArray(quantized);
  for (let y = 0; y < height; y++) {
    for (let x = 0; x < width; x++) {
      const i  = y * width + x;
      const pi = i * 4;
      const lum = 0.299 * out[pi] + 0.587 * out[pi + 1] + 0.114 * out[pi + 2];

      // Screen-tone: 4×4 dot pattern in shadow areas (luminance < 80)
      if (lum < 80 && (x % 4 < 2) && (y % 4 < 2)) {
        out[pi] = out[pi + 1] = out[pi + 2] = 20;
      }

      // Edge override: black outlines on top of everything
      if (magnitude[i] > EDGE_THRESHOLD) {
        out[pi] = out[pi + 1] = out[pi + 2] = 0;
      }
    }
  }

  putCropImageData(new ImageData(out, width, height));
}
```

---

### 15.5 Watercolor

**Visual:** soft bleeding edges, off-white paper texture showing through, washed-out darks.

```javascript
function applyWatercolor() {
  const { width, height } = rect;
  const src = getCropImageData();

  // Step 1: 3 passes of box blur — simulates color spreading on wet paper
  let blurred = boxBlur(src, 3);
  blurred     = boxBlur(blurred, 2);
  blurred     = boxBlur(blurred, 1);

  // Step 2: Blend 35% of original edges back in (re-sharpen contours slightly)
  const gray = toGrayscaleArray(src);
  const { magnitude } = sobelGradient(gray, width, height);

  const out = new Uint8ClampedArray(blurred.data.length);
  for (let i = 0; i < width * height; i++) {
    const pi = i * 4;
    const edgeFactor = Math.min(1, magnitude[i] / 120) * 0.35;
    out[pi]     = blurred.data[pi]     * (1 - edgeFactor) + src.data[pi]     * edgeFactor;
    out[pi + 1] = blurred.data[pi + 1] * (1 - edgeFactor) + src.data[pi + 1] * edgeFactor;
    out[pi + 2] = blurred.data[pi + 2] * (1 - edgeFactor) + src.data[pi + 2] * edgeFactor;
    out[pi + 3] = 255;
  }

  // Step 3: Paper texture via sine-based noise, lightens toward white
  for (let y = 0; y < height; y++) {
    for (let x = 0; x < width; x++) {
      const pi = (y * width + x) * 4;
      // noise in range 0.85–1.0; higher = more paper-white bleed
      const noise = 0.85 + 0.15 * (0.5 + 0.5 * Math.sin(x * 17.3) * Math.sin(y * 13.7));
      out[pi]     = Math.min(255, out[pi]     * noise + (1 - noise) * 255);
      out[pi + 1] = Math.min(255, out[pi + 1] * noise + (1 - noise) * 255);
      out[pi + 2] = Math.min(255, out[pi + 2] * noise + (1 - noise) * 255);
    }
  }

  // Step 4: Desaturate dark areas (watercolor darks look muted, not saturated)
  for (let i = 0; i < out.length; i += 4) {
    const lum = 0.299 * out[i] + 0.587 * out[i + 1] + 0.114 * out[i + 2];
    if (lum < 90) {
      const t = 0.4; // 40% pull toward gray
      out[i]     = out[i]     * (1 - t) + lum * t;
      out[i + 1] = out[i + 1] * (1 - t) + lum * t;
      out[i + 2] = out[i + 2] * (1 - t) + lum * t;
    }
  }

  putCropImageData(new ImageData(out, width, height));
}
```

---

### 15.6 Van Gogh

**Visual:** swirling, direction-following paintbrush strokes that echo the image's own edge flow.

**Important:** This effect draws directly onto `ctx` rather than using `putImageData`.

```javascript
function applyVanGogh() {
  const { x: rx, y: ry, width, height } = rect;
  const src = getCropImageData();   // save source pixels before painting over the region

  // Step 1: Draw a blurred version of the source as the background
  //         (fills gaps between strokes with approximate color)
  const bgBlurred = blurImageData(src, 2);
  const tmpCanvas = document.createElement('canvas');
  tmpCanvas.width = width; tmpCanvas.height = height;
  tmpCanvas.getContext('2d').putImageData(bgBlurred, 0, 0);

  ctx.save();
  ctx.globalAlpha = 0.5;
  ctx.drawImage(tmpCanvas, rx, ry);   // drawImage respects globalAlpha; putImageData does not
  ctx.globalAlpha = 1.0;

  // Step 2: Compute gradient flow field from source
  const gray = toGrayscaleArray(src);
  const { gx, gy } = sobelGradient(gray, width, height);

  // Step 3: Paint oriented strokes on a regular grid
  const GRID  = 6;    // pixels between stroke centers
  const S_LEN = 13;   // half-length of each stroke in pixels
  const S_W   = 3;    // stroke line width

  ctx.lineWidth = S_W;
  ctx.lineCap   = 'round';

  for (let y = Math.floor(GRID / 2); y < height; y += GRID) {
    for (let x = Math.floor(GRID / 2); x < width; x += GRID) {
      const idx = y * width + x;
      const pi  = idx * 4;

      // Stroke direction: perpendicular to gradient → strokes run ALONG edges
      const angle = Math.atan2(gy[idx], gx[idx]) + Math.PI / 2;
      const cos   = Math.cos(angle);
      const sin   = Math.sin(angle);

      // Source color with slight random variation for natural organic look
      const v = () => Math.floor((Math.random() - 0.5) * 22);
      const r = Math.min(255, Math.max(0, src.data[pi]     + v()));
      const g = Math.min(255, Math.max(0, src.data[pi + 1] + v()));
      const b = Math.min(255, Math.max(0, src.data[pi + 2] + v()));

      ctx.beginPath();
      ctx.strokeStyle = `rgb(${r},${g},${b})`;
      ctx.moveTo(rx + x - cos * S_LEN, ry + y - sin * S_LEN);
      ctx.lineTo(rx + x + cos * S_LEN, ry + y + sin * S_LEN);
      ctx.stroke();
    }
  }

  ctx.restore();
}
```

---

### 15.7 Cross-Hatch

**Visual:** steel engraving look — white paper background with layered diagonal hatching, denser in dark areas.

**Important:** Draws directly onto `ctx`, not pixel manipulation.

```javascript
function applyCrossHatch() {
  const { x: rx, y: ry, width, height } = rect;
  const src  = getCropImageData();
  const gray = toGrayscaleArray(src);

  ctx.save();

  // Clip all drawing to the rectangle
  ctx.beginPath();
  ctx.rect(rx, ry, width, height);
  ctx.clip();

  // Off-white paper background
  ctx.fillStyle = '#f5f0e8';
  ctx.fillRect(rx, ry, width, height);

  ctx.strokeStyle = '#1a1208';
  ctx.lineWidth   = 0.9;

  const SPACING = 8;   // pixels between parallel lines in each layer
  const LAYERS  = [
    { angle:  Math.PI / 4,  threshold: 195 },  //  45° — drawn in light AND dark areas
    { angle: -Math.PI / 4,  threshold: 145 },  // -45° — drawn in medium and dark areas
    { angle:  0,            threshold: 95  },  //   0° — drawn only in dark areas
    { angle:  Math.PI / 2,  threshold: 50  },  //  90° — drawn only in the darkest areas
  ];

  const diag = Math.ceil(Math.sqrt(width * width + height * height));

  for (const { angle, threshold } of LAYERS) {
    const cos = Math.cos(angle);
    const sin = Math.sin(angle);
    // Perpendicular direction (used to offset parallel lines)
    const px = -sin;
    const py =  cos;

    for (let offset = -diag; offset < diag; offset += SPACING) {
      // Center of the line's offset position
      const ocx = width  / 2 + px * offset;
      const ocy = height / 2 + py * offset;

      // Check luminance at the midpoint of this line
      const mx = Math.round(ocx);
      const my = Math.round(ocy);
      if (mx < 0 || mx >= width || my < 0 || my >= height) continue;
      if (gray[my * width + mx] >= threshold) continue; // too bright — skip

      // Line endpoints extended across the full rectangle (clipping handles the rest)
      const x1 = rx + ocx - cos * diag;
      const y1 = ry + ocy - sin * diag;
      const x2 = rx + ocx + cos * diag;
      const y2 = ry + ocy + sin * diag;

      ctx.beginPath();
      ctx.moveTo(x1, y1);
      ctx.lineTo(x2, y2);
      ctx.stroke();
    }
  }

  ctx.restore();
}
```

---

### 15.8 Wool / Knit

**Visual:** the image appears woven from threads — fibers run along contours of the scene like yarn.

**Important:** Draws directly onto `ctx`.

```javascript
function applyWool() {
  const { x: rx, y: ry, width, height } = rect;
  const src = getCropImageData();

  ctx.save();

  // Step 1: Muted background — draw source at low opacity then lighten with cream overlay
  const tmpCanvas = document.createElement('canvas');
  tmpCanvas.width = width; tmpCanvas.height = height;
  tmpCanvas.getContext('2d').putImageData(src, 0, 0);

  ctx.globalAlpha = 0.35;
  ctx.drawImage(tmpCanvas, rx, ry);
  ctx.globalAlpha = 1.0;

  ctx.fillStyle = 'rgba(240, 234, 220, 0.55)';
  ctx.fillRect(rx, ry, width, height);

  // Step 2: Compute gradient field
  const gray = toGrayscaleArray(src);
  const { gx, gy } = sobelGradient(gray, width, height);

  // Step 3: Draw fibers on a grid
  const STEP   = 5;    // sample every N pixels
  const F_LEN  = 10;   // fiber half-length in pixels
  const WOBBLE = 2.5;  // amplitude of sinusoidal wobble (simulates thread looseness)

  ctx.lineWidth = 1.8;
  ctx.lineCap   = 'round';

  for (let y = 0; y < height; y += STEP) {
    for (let x = 0; x < width; x += STEP) {
      const idx = y * width + x;
      const pi  = idx * 4;

      // Fiber direction: perpendicular to gradient → fiber runs ALONG edges
      const angle    = Math.atan2(gy[idx], gx[idx]) + Math.PI / 2;
      const cos      = Math.cos(angle);
      const sin      = Math.sin(angle);
      const perpCos  = -sin;    // perpendicular for wobble displacement
      const perpSin  =  cos;

      // Wobble varies by position to create organic weave pattern
      const wobble = WOBBLE * Math.sin(x * 0.42 + y * 0.31);

      const sx = rx + x;
      const sy = ry + y;

      // Three-point quadratic curve: start → control → end
      const x0 = sx - cos * F_LEN + perpCos * wobble;
      const y0 = sy - sin * F_LEN + perpSin * wobble;
      const xc = sx + perpCos * (wobble * 0.5);   // control point (slight bow)
      const yc = sy + perpSin * (wobble * 0.5);
      const x2 = sx + cos * F_LEN - perpCos * wobble;
      const y2 = sy + sin * F_LEN - perpSin * wobble;

      const r = src.data[pi];
      const g = src.data[pi + 1];
      const b = src.data[pi + 2];

      ctx.strokeStyle = `rgb(${r},${g},${b})`;
      ctx.beginPath();
      ctx.moveTo(x0, y0);
      ctx.quadraticCurveTo(xc, yc, x2, y2);
      ctx.stroke();
    }
  }

  ctx.restore();
}
```

---

### 15.9 Old Master ("Jesus Style")

**Visual:** warm golden-brown tones, soft sfumato edges, craquelure (cracked varnish lines), aged grain — resembles a Renaissance oil painting.

```javascript
function applyOldMaster() {
  const { width, height } = rect;
  const src = getCropImageData();
  const d   = src.data;  // modifying this in-place; src is the output

  // Step 1: Warm color grade — push toward golden-sepia palette
  for (let i = 0; i < d.length; i += 4) {
    d[i]     = Math.min(255, d[i]     * 1.12 + 18);  // red   ↑
    d[i + 1] = Math.min(255, d[i + 1] * 0.94 +  5);  // green slightly ↑
    d[i + 2] = Math.min(255, d[i + 2] * 0.76);        // blue  ↓
  }

  // Step 2: Sfumato — blur then blend 25% sharp back in
  // (src.data already has warm grade at this point — that is intentional)
  const sharpCopy = new Uint8ClampedArray(d);   // save warm-graded sharp version
  const blurredData = blurImageData(src, 1.5).data;
  for (let i = 0; i < d.length; i += 4) {
    d[i]     = blurredData[i]     * 0.75 + sharpCopy[i]     * 0.25;
    d[i + 1] = blurredData[i + 1] * 0.75 + sharpCopy[i + 1] * 0.25;
    d[i + 2] = blurredData[i + 2] * 0.75 + sharpCopy[i + 2] * 0.25;
  }

  // Step 3: Craquelure — domain-warped sine pattern simulates cracked varnish
  for (let y = 0; y < height; y++) {
    for (let x = 0; x < width; x++) {
      const pi     = (y * width + x) * 4;
      const crack1 = Math.abs(Math.sin(x * 0.15 + Math.sin(y * 0.05) * 10));
      const crack2 = Math.abs(Math.sin(y * 0.12 + Math.sin(x * 0.06) *  8));
      if (crack1 < 0.06 || crack2 < 0.06) {
        d[pi]     = Math.max(0, d[pi]     - 28);
        d[pi + 1] = Math.max(0, d[pi + 1] - 20);
        d[pi + 2] = Math.max(0, d[pi + 2] - 14);
      }
    }
  }

  // Step 4: Aged grain — subtle random per-pixel noise
  for (let i = 0; i < d.length; i += 4) {
    const noise  = (Math.random() - 0.5) * 14;
    d[i]     = Math.min(255, Math.max(0, d[i]     + noise));
    d[i + 1] = Math.min(255, Math.max(0, d[i + 1] + noise * 0.8));
    d[i + 2] = Math.min(255, Math.max(0, d[i + 2] + noise * 0.55));
  }

  // src.data was modified in-place above — put it directly
  putCropImageData(src);
}
```

---

## 16. Visual Overlays

```javascript
function drawOverlays() {
  // ── Status hints ───────────────────────────────────────────────────────────
  if (!rect || rect.width <= 20 || rect.height <= 20) {
    if (leftHandData || rightHandData) {
      setStatus('Show both hands with palm open');
    } else {
      setStatus('Hold both hands up, palms facing camera');
    }
    return;
  }
  setStatus('');

  // ── Dashed rectangle border ────────────────────────────────────────────────
  ctx.save();
  ctx.strokeStyle = 'rgba(255, 255, 255, 0.80)';
  ctx.lineWidth   = 2;
  ctx.setLineDash([6, 4]);
  ctx.strokeRect(rect.x, rect.y, rect.width, rect.height);
  ctx.setLineDash([]);

  // ── Corner tip dots ────────────────────────────────────────────────────────
  const tipPts = [
    leftHandData.thumbTip,  leftHandData.indexTip,
    rightHandData.thumbTip, rightHandData.indexTip,
  ];
  for (const pt of tipPts) {
    ctx.beginPath();
    ctx.arc(pt.x, pt.y, 5, 0, Math.PI * 2);
    ctx.fillStyle = 'rgba(255, 215, 0, 0.92)';
    ctx.fill();
  }

  // ── Collapse progress: white fill bleeds in as hands approach ──────────────
  if (collapseFrameCount > 0) {
    const progress = Math.min(1, collapseFrameCount / COLLAPSE_FRAMES_NEEDED);
    ctx.globalAlpha = progress * 0.30;
    ctx.fillStyle   = '#ffffff';
    ctx.fillRect(rect.x, rect.y, rect.width, rect.height);
    ctx.globalAlpha = 1.0;
  }

  ctx.restore();
}

function setStatus(text) {
  statusEl.textContent = text;
  statusEl.style.opacity = text.length > 0 ? '1' : '0';
}
```

---

## 17. Edge Cases

| Situation | Required Behavior |
|---|---|
| Camera permission denied | `setupCamera` throws; `init` catches and calls `setStatus('Camera permission denied.')` |
| Model download fails | `setupMediaPipe` throws; `init` catches and calls `setStatus('Failed to load model. Check internet connection.')` |
| Only one hand visible | `processHandResults` sets both vars to null; `rect = null`; no effect applied; hint shown in status |
| Zero hands detected | Same as above |
| Hands partially out of frame | `computeRect()` clamps to canvas bounds — result may be a narrow sliver, effect still applies |
| `rect.width` or `rect.height` < 20px | `applyCurrentEffect()` is skipped entirely — prevents `getImageData` errors on near-zero areas |
| `rect` extends partially off-screen | Clamping in `computeRect()` ensures `x >= 0`, `y >= 0`, `x + width <= W`, `y + height <= H` |
| Video frame repeated (no advance) | `lastVideoTime` guard skips the frame — no wasted MediaPipe calls |
| Van Gogh / Wool / Cross-Hatch on tiny rect | Grid loops simply produce fewer strokes — no crash, graceful degradation |
| `blurImageData` called rapidly | Creates and discards temp canvases each call — garbage-collected; no persistent leak |
| Both hands on the same side (x nearly equal) | One becomes left, other becomes right by the `<=` comparison — rect may be narrow but no crash |

---

## 18. Performance Budget (640 × 480, crop ~200 × 200px)

| Operation | Typical cost |
|---|---|
| `ctx.drawImage` (video → canvas, mirrored) | ~1 ms |
| MediaPipe `detectForVideo` | ~5–15 ms |
| `getImageData` on crop | ~0.5 ms |
| Pencil Sketch | ~3–5 ms |
| Oil Paint (radius 4) | ~8–12 ms |
| Cartoon | ~4–6 ms |
| Anime | ~5–7 ms |
| Watercolor | ~6–9 ms |
| Van Gogh (stroke drawing) | ~4–8 ms |
| Cross-Hatch | ~3–5 ms |
| Wool | ~5–8 ms |
| Old Master | ~3–5 ms |
| `putImageData` + overlays | ~1 ms |
| **Worst-case total** | **~30–42 ms → ~24–30 fps** |

All effects operate on the **crop region only**, not the full frame. This is the key optimization. A very small crop runs much faster. A very large crop (hands far apart) may dip on older hardware; consider capping `rect.width` and `rect.height` at 350px in `computeRect()` if needed.

**If performance is too low:** reduce `RADIUS` in Oil Paint, increase `GRID` / `STEP` in Van Gogh / Wool, or reduce box-blur passes in Watercolor.

---

## 19. Implementation Checklist

- [ ] `<script type="module">` — required for `import` to work
- [ ] MediaPipe pinned to version `0.10.14` — avoid API drift from `@latest`
- [ ] `#video` uses `visibility: hidden`, NOT `display: none`
- [ ] Canvas dimensions set in `onloadedmetadata`, not hardcoded
- [ ] All landmark x-coords mirrored: `(1 - landmark.x) * W`
- [ ] Left/right hand assigned by wrist x-position, NOT by MediaPipe's handedness label
- [ ] `checkCollapseGesture` uses both `COLLAPSE_THRESHOLD_PX` and `REOPEN_THRESHOLD_PX` (hysteresis gap of 30px)
- [ ] `applyCurrentEffect()` guarded: skipped when `rect.width < 20` or `rect.height < 20`
- [ ] `computeRect()` produces non-negative width/height and stays within canvas bounds
- [ ] `rgbToHsl` / `hslToRgb` defined before `applyAnime()` in source order
- [ ] Van Gogh, Wool, and Cross-Hatch all use `ctx.drawImage(tmpCanvas, ...)` (not `putImageData`) when globalAlpha is needed
- [ ] Van Gogh, Wool, Cross-Hatch all wrapped in `ctx.save()` / `ctx.restore()`
- [ ] Cross-Hatch uses `ctx.clip()` to prevent strokes bleeding outside the rectangle
- [ ] Effect label fades out after 1800ms via CSS `transition: opacity 0.4s` + `setTimeout`
- [ ] `lastVideoTime` guard in render loop prevents duplicate MediaPipe calls on static frames
- [ ] `init()` is called at the bottom of the module script
