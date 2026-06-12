# Hand Frame Effects

A zero-dependency, single-file webcam art app. Hold up both hands and frame a region with your thumbs and index fingers — live art effects render **only inside that window**, every frame, in real time.

Pinch your hands together to cycle through effects. No install, no server — just open `index.html` in Chrome, Edge, or Firefox.

---

## Demo

| Effect | Look |
|---|---|
| **Pencil Sketch** | Grayscale line drawing on white paper |
| **Oil Paint** | Thick impasto brush strokes |
| **Cartoon** | Flat posterized color + bold outlines |
| **Anime** | Vibrant saturation + manga screen-tone shadows |
| **Watercolor** | Soft bleeding edges + paper texture |
| **Van Gogh** | Swirling direction-following strokes |
| **Cross-Hatch** | Steel-engraving hatching by luminance |
| **Wool / Knit** | Contour-following fiber weave |
| **Old Master** | Warm sepia sfumato + craquelure varnish cracks |

---

## How to Use

1. Open `index.html` in **Chrome 100+**, Edge 100+, or Firefox 131+
2. Allow camera access when prompted (~8 MB model downloads once)
3. Hold both hands up with palms facing the camera
4. Touch thumb tip to index tip on each hand to form a frame — the effect renders inside it
5. **Bring hands together** (hold ~0.6 s) to cycle to the next effect

No localhost required. Works directly from `file://`.

---

## How It Works

```
Webcam → MediaPipe Hand Landmarker (WebAssembly) → 4 fingertip coordinates
→ bounding rectangle → Canvas 2D pixel effects inside that region → display
```

- **Hand tracking:** MediaPipe Tasks Vision `0.10.14` running entirely in-browser via WebAssembly + WebGL
- **Effects:** Pure Canvas 2D — pixel manipulation, `getImageData`/`putImageData`, `globalCompositeOperation`, direct stroke drawing
- **Gesture:** Collapse detection uses frame counting with hysteresis to prevent accidental triggers
- **Performance:** Effects operate only on the crop region (~200×200 px typical), not the full 640×480 frame

---

## Tech Stack

| Concern | Solution |
|---|---|
| Hand tracking | MediaPipe Tasks Vision (WASM) |
| Rendering | Native Canvas 2D API |
| Camera | `getUserMedia` |
| Build tools | None |
| Dependencies | None |

---

## Browser Requirements

- **Webcam** access required
- **Internet** required on first load (model ~8 MB cached after that)
- Chrome 100+ / Edge 100+ / Firefox 131+ (desktop)

---

## Project Structure

```
index.html                          ← entire app (HTML + CSS + JS, one file)
hand-frame-effects-spec.md          ← implementation specification
hand-frame-effects-research-addendum.md  ← performance & quality upgrades
```

---

## License

MIT
