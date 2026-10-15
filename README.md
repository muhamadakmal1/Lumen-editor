# Lumen-editor

**A non-destructive, in-browser photo editor in a single HTML file.**

Lumen is a working prototype of a darkroom-style photo editor. It has tone and colour adjustments, curves, filters, crop and transform, text, shapes, freehand drawing, a layer stack, full undo history and multi-format export. It runs entirely on the client with the Canvas 2D API, with no framework, no build step and no server.

> **Status:** Prototype. The core editing pipeline is complete and usable. Persistence, metadata handling and some export paths are intentionally out of scope (see [Known limitations](#known-limitations)).

---

## Contents

- [Highlights](#highlights)
- [Quick start](#quick-start)
- [Feature reference](#feature-reference)
- [Controls and shortcuts](#controls-and-shortcuts)
- [Architecture](#architecture)
- [Performance model](#performance-model)
- [Browser support](#browser-support)
- [Known limitations](#known-limitations)
- [Roadmap](#roadmap)

---

## Highlights

- **Non-destructive.** The source image is never modified. Every edit is a parameter in a serialisable state object, and the output is re-rendered from the original each time.
- **Zero dependencies.** One `index.html`. The only external request is Google Fonts, used for text layers and the UI.
- **Photographic adjustment set.** 30+ parameters cover exposure, highlights/shadows, white balance, vibrance, split-toning, levels, clarity and more.
- **Spline-based curves.** Master, red, green and blue channels use monotone cubic interpolation, so curves never overshoot or invert.
- **Live histogram.** RGB and luminance, recomputed from the rendered result.
- **Layered annotations.** Text, shapes, arrows, pixelate boxes and brush strokes are independent, reorderable, hideable layers.
- **80-step history** with click-to-jump, plus hold-to-compare against the original.
- **Touch-ready.** Pinch zoom, pointer-event interaction, and a layout that stacks on phones.
- **Light and dark themes** that follow the system setting.

---

## Quick start

No installation is required.

```bash
# Option 1: open the file directly
open index.html

# Option 2: serve it locally (recommended, avoids any file:// quirks)
python3 -m http.server 8000
# then visit http://localhost:8000
```

On launch, Lumen loads a procedurally generated sample photo (a dusk landscape) so every control can be tried immediately. To edit your own image, use **Open**, **drag and drop** a file onto the window, or **paste** from the clipboard.

---

## Feature reference

### Adjust

| Group | Controls |
|---|---|
| **Light** | Exposure, Brightness, Contrast, Highlights, Shadows, Whites, Blacks |
| **Colour** | Temperature, Tint, Saturation, Vibrance, Hue shift |
| **Detail** | Sharpen, Clarity, Blur, Grain |
| **Effects** | Vignette (dark or light), Fade, Sepia, Black & white, Invert, Pixelate |
| **Colour grading** | Shadow hue and tint, Highlight hue and tint, Balance |
| **Levels** | Black point, White point, Midtones (gamma) |

Double-click any slider label to reset it. Changed sliders are highlighted in the accent colour.

### Curves

Four independent channels (RGB, Red, Green, Blue). Click the curve to add a point, drag to shape it, double-click a point to remove it.

### Filters

Fifteen presets, shown as live thumbnails of your current photo: *Original, Vivid, Mono, Noir, Silver, Sepia, Fade, Warm, Cool, Vintage, Dramatic, Matte, Cinema, Golden hour, Cyber.*

Filters are **additive**. A preset is a set of parameter offsets scaled by a **Filter strength** slider and summed with your manual Adjust values, so the two always compose predictably.

### Crop and transform

- Aspect presets: Free, Original, 1:1, 4:5, 3:2, 16:9, 9:16, 3:4
- Rule-of-thirds overlay, eight resize handles, drag-to-move
- Rotate 90° left or right, flip horizontal or vertical
- **Straighten** ±45° in 0.1° steps, with automatic scale-to-fill so no empty corners appear
- **Frame:** border width and colour, rounded corners

### Text

Multi-line text layers with font choice, bold and italic, alignment, size, colour, outline, drop shadow, optional background box, letter spacing, rotation and opacity. Fonts: Bricolage Grotesque, IBM Plex Sans, Playfair Display, Anton, Caveat, IBM Plex Mono, Georgia, Impact.

### Shapes

Rectangle, Ellipse, Line, Arrow and **Pixelate box**, each with stroke, fill, corner radius and opacity where applicable. Drag on the photo to draw, click to select, and use the handle to resize. Line and arrow shapes expose both endpoints.

### Draw

Brush, Marker (wide, translucent) and Eraser. Every stroke is its own layer, so undo works stroke by stroke. The eraser removes whole strokes rather than pixels.

### Layers and History

- **Layers:** select, show or hide, reorder and delete. The base photo with its adjustments always sits at the bottom.
- **History:** a labelled list of up to 80 states. Click any entry to return to it. Stepping back and then editing discards the redo branch.

### Export

PNG (lossless), JPEG or WebP, at 100%, 75%, 50% or 25% scale, with a quality slider (30 to 100) for lossy formats. The dialog shows a live preview with exact pixel dimensions and file size. Choose **Save file** or **Copy image** (PNG to the clipboard).

---

## Controls and shortcuts

| Action | Input |
|---|---|
| Undo / Redo | `Ctrl/Cmd + Z` / `Ctrl/Cmd + Shift + Z` (or `Ctrl/Cmd + Y`) |
| Compare with original | Hold **Before**, or hold `\` |
| Zoom | Mouse wheel, trackpad pinch, `+` / `-`, or the toolbar |
| Fit to screen | `0` or **Fit** |
| Pan | Hold `Space` and drag, use the Hand tool, middle-click drag, or drag empty canvas |
| Pinch zoom (touch) | Two-finger gesture |
| Delete selected layer | `Delete` / `Backspace` |
| Nudge selected layer | Arrow keys (fine), `Shift` + arrows (coarse) |
| Jump to a layer's editor | Double-click it on the canvas |
| Reset one slider | Double-click its label |
| Close export dialog | `Esc` |

---

## Architecture

Lumen is deliberately small. It is organised as a single script with clearly separated concerns.

### State model

All edits live in one plain object, which makes history, serialisation and comparison trivial:

```js
{
  params:  { exposure, contrast, temp, ... },   // ~30 numeric adjustments
  filter:  'noir', filterAmt: 100,              // preset + strength
  curves:  { m: [[0,0],[255,255]], r, g, b },   // control points per channel
  geom:    { rot, fh, fv, angle, crop: {x,y,w,h} },
  frame:   { w, color, radius },
  layers:  [ { id, type, ... } ],               // text, shapes, strokes
  uid:     1
}
```

Layer geometry is stored in **normalised coordinates (0 to 1)**. This makes overlays resolution-independent, so the same layer renders identically on the 1400 px preview and on a full-resolution export.

History snapshots are `JSON.stringify(state)`. `push()` skips duplicate states, caps the stack at 80 entries and truncates any redo branch.

### Render pipeline

`render(which, maxDim, options)` is the single entry point used by the canvas, filter thumbnails and export:

1. **Geometry:** rotate, flip and straighten the source (cached by parameters), then crop.
2. **Adjustment pass:** one per-pixel loop over the image data (order below).
3. **Overlays:** text, shapes, strokes and pixelate boxes, in layer order.
4. **Frame:** border, then rounded-corner mask via `destination-in`.

Order of operations inside the adjustment pass:

```
pixelate → blur → (sharpen / clarity masks) → levels → exposure → brightness
→ contrast → highlights/shadows/whites/blacks → temperature/tint
→ saturation/vibrance → hue rotation → split-toning → fade → sepia → mono
→ curves (R,G,B, then master) → invert → vignette → grain
```

A few implementation notes:

- **Curves** are baked into 256-entry lookup tables using Fritsch–Carlson monotone cubic Hermite interpolation. An identity curve returns `null` and is skipped entirely.
- **Sharpen** is an unsharp mask. **Clarity** is a large-radius local-contrast boost weighted towards midtones.
- **Blur** is a two-pass separable box blur using running sums, so it is O(n) regardless of radius.
- **Tonal sliders** (highlights, shadows, whites, blacks) use smooth luminance-weighted masks rather than hard thresholds.
- **Hue shift** is a 3×3 colour matrix rotation. **Vibrance** scales saturation inversely to existing chroma, protecting already-saturated colours.
- **Grain** uses a deterministic hash of pixel coordinates, so it is stable between redraws and between preview and export.

### Preview and caching

The adjusted base image is cached under a key derived from every input that affects it (source, params, filter, curves, geometry, crop mode, compare state). Overlay changes, selection, zoom and pan therefore never re-run the pixel pass. Redraws are coalesced with `requestAnimationFrame`.

---

## Performance model

| Stage | Resolution |
|---|---|
| Imported source (`srcFull`) | Downscaled to a maximum of **4096 px** on the long edge |
| Working preview source (`srcPrev`) | **1500 px** long edge |
| On-screen render | Up to **1400 px** long edge |
| Filter thumbnails | 132 px, rendered progressively, one per frame, and cached |
| Export | Re-rendered from `srcFull` at the chosen scale |

The pixel pass runs on the main thread. This is comfortable at preview size and acceptable at export size, but blocks the UI briefly for very large exports. Export is debounced (220 ms) and uses a token to discard stale renders.

---

## Browser support

Targets current evergreen browsers (Chrome, Edge, Safari, Firefox). It requires Canvas 2D, Pointer Events, `ResizeObserver`, `createImageBitmap` (with an `Image` fallback), CSS `color-mix()` and CSS custom properties. Letter spacing on text layers uses `CanvasRenderingContext2D.letterSpacing` where available and is silently ignored elsewhere.

EXIF orientation is honoured on import via `imageOrientation: 'from-image'`.

---

## Known limitations

These are known prototype boundaries, not bugs.

- **"Save file" depends on a host capability.** It uses a `claude.use('downloads')` runtime hook. In a plain browser tab that hook is absent, so the button shows a message instead of saving. Use **Copy image**, or press-and-hold or right-click the export preview to save. Replacing this with an `<a download>` fallback is the first roadmap item.
- **Pixelate box is not secure redaction.** It mosaics pixels at an adjustable block size. Small blocks can remain partially recoverable. For sensitive content, cover it with a filled, fully opaque rectangle instead.
- **No metadata preservation.** Re-encoding through canvas drops EXIF, GPS and colour profile data.
- **No colour management.** Processing is in the canvas's default sRGB space and ignores embedded ICC profiles.
- **No project persistence.** Reloading the page discards the session. State is serialisable but not saved.
- **JPEG export flattens transparency onto white**, including the corners created by *Rounded corners*. Use PNG or WebP to keep them transparent.
- **4096 px import cap.** Larger images are downscaled on load; the export dialog flags this.
- **Main-thread processing.** There is no Web Worker or GPU path yet.
- **Brush eraser works on strokes, not pixels**, and cannot erase the photo or non-brush layers.
- **Web fonts need a network connection** on first load. Offline, text falls back to system fonts.

---

## Roadmap

Suggested next steps, roughly in priority order:

1. `<a download>` fallback so **Save file** works outside the host environment
2. Project save and load (JSON state plus source reference) and autosave to IndexedDB
3. Move the adjustment pass to a Web Worker or WebGL for real-time full-resolution editing
4. Selective edits: radial and linear gradient masks, and a brush mask
5. Healing and clone tools
6. Optional EXIF passthrough and ICC-aware processing
7. Batch export and preset saving (user-defined filters)
8. Automated tests for `curveLUT`, `cropCalc` and the history stack

---

## Project structure

```
index.html    # entire application: markup, styles and script
README.md
```

Inside `index.html` the script is divided into labelled sections: utils, state, presets, curves, blur, adjustment pass, geometry, overlays, render, sample photo, loading, view and history, UI helpers, tabs (Adjust, Curves, Filters, Crop, Layers, Text, Shapes, Draw), selection overlay, interaction, toolbar, and export.

---

## Contributing

This is a prototype, so the most useful feedback is on the editing experience: slider ranges, preset looks, interaction feel on touch devices, and any case where the render differs between preview and export. Please include the browser, device and a sample image when reporting an issue.
