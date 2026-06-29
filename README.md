# 🧊 Cube Solver — 3×3 Rubik's Cube

> **Upload photos of each face → Get a near-optimal solution → Watch the 3D cube solve itself.**

A fully browser-based Rubik's Cube solver that runs 100% offline after the first load. No server, no backend — pure HTML, CSS, and JavaScript.

---

## 📸 Preview

| 3D Viewer (Solved) | 3D Viewer (Flipped View) |
|---|---|
| White face on top, solved state | Yellow face on top, camera inverted |

> Drag the cube to rotate it, scroll to zoom, or use the **Flip** button to invert the camera vertically.

---

## ✨ Features at a Glance

| Feature | Description |
|---|---|
| 📷 **Photo Input** | Upload one photo per face; colors are auto-detected |
| 🎨 **Color Editor** | Manually click stickers to paint the correct colors |
| ⚡ **Kociemba Solver** | Near-optimal solution (Kociemba two-phase algorithm) |
| 🎬 **3D Animation** | Step-by-step animated playback on the 3D cube |
| 🔄 **Flip View** | Smoothly inverts the camera 180° to see the bottom face |
| 📱 **Touch Support** | Full drag + pinch-zoom on mobile |
| 🌙 **Dark UI** | Premium dark theme with purple accents |

---

## 🗂️ Project Structure

```
Cube Solver/
│
├── index.html            ← Single-page app shell
│
├── css/
│   └── style.css         ← All styling (dark theme, animations, layout)
│
├── js/
│   ├── app.js            ← Main controller (ties everything together)
│   ├── cube3d.js         ← Three.js 3D cube renderer + camera controls
│   ├── cubeState.js      ← Cube state logic (facelet string, validation)
│   ├── imageParser.js    ← Photo → color detection (HSV algorithm)
│   └── solver.js         ← Kociemba solver wrapper + scrambler
│
└── lib/
    ├── cube.js           ← cubejs: cube model + move engine
    ├── solve.js          ← cubejs: Kociemba two-phase solver tables
    └── three.min.js      ← Three.js r128 (3D rendering, bundled locally)
```

---

## 🧠 How It All Works — Step by Step

### Step 1 — Represent the Cube as a String

A Rubik's Cube has **54 stickers** (6 faces × 9 stickers). We store the entire cube state as a **54-character string** in this face order:

```
U R F D L B
```

Each character is one of: `U` (white), `R` (red), `F` (green), `D` (yellow), `L` (orange), `B` (blue).

**Solved state looks like:**
```
UUUUUUUUU  RRRRRRRRR  FFFFFFFFF  DDDDDDDDD  LLLLLLLLL  BBBBBBBBB
   Up          Right      Front      Down       Left       Back
```

**Why a string?** It's simple, fast to compare, and directly matches what the `cubejs` library expects as input/output.

---

### Step 2 — Photo Color Detection (`imageParser.js`)

When you upload a photo of a face, here is what happens:

```
Photo → Resize to 300×300px → Divide into 3×3 grid → Sample center of each cell → Classify color
```

#### HSV Color Classification

Plain RGB comparison fails badly (yellow looks almost white under bright light). We convert each sampled pixel to **HSV** (Hue, Saturation, Value) first:

```
White  → low Saturation  (s < 0.2) + high Value (v > 0.75)
Yellow → Hue 20°–45°,    high Saturation
Green  → Hue 45°–160°,   high Saturation
Blue   → Hue 160°–260°,  high Saturation
Red    → Hue 340°–20°,   high Saturation
Orange → Hue 260°–340°,  high Saturation
```

The **center sticker is always trusted** — the user aligned the cube to that face, so its color is forced to the expected face letter regardless of what the camera saw.

---

### Step 3 — Cube State Validation (`cubeState.js`)

Before solving, we check that the input is physically possible:

1. **Length check** — must be exactly 54 characters
2. **Count check** — each color must appear exactly 9 times
3. **Center check** — position 4 of each face group must be the face's own color
4. **Round-trip check** — convert string → internal cube model → back to string; if it changes, the state is physically impossible (twisted corner, flipped edge, etc.)

```javascript
const cube = Cube.fromString(facelets);
const roundTrip = cube.asString();
if (roundTrip !== facelets) {
  throw new Error('Physically impossible cube state');
}
```

---

### Step 4 — The Kociemba Two-Phase Algorithm (`lib/solve.js`)

This is the math brain of the project. It finds solutions in **≤ 20 moves** (God's Number) for any valid cube state.

#### Phase 1 — Get into a Subgroup
Move the cube into a reduced state where only `U`, `D`, and double moves (`R2`, `F2`, etc.) are needed. This is the "coarse" phase.

**Coordinates tracked in Phase 1:**
- `twist` — orientation of all 8 corners (2,187 states)
- `flip` — orientation of all 12 edges (2,048 states)
- `FRtoBR` — which 4 edges are in the FR→BR slice (495 states)

#### Phase 2 — Solve within the Subgroup
From the reduced state, find the final sequence of moves. Only `U`, `D`, `R2`, `L2`, `F2`, `B2` are allowed.

**Coordinates tracked in Phase 2:**
- `URFtoDLF` — permutation of 6 specific corners
- `URtoDF` — permutation of 6 specific edges
- `Parity` — even/odd parity of the permutation

#### Pruning Tables
To search efficiently, the algorithm precomputes **pruning tables** — lookup tables that say "from this state, you need at least N more moves". If N exceeds the current depth limit, that branch is abandoned immediately (IDA* search).

```
Pruning table size ≈ millions of entries, but built once and reused
Building time: ~3–5 seconds on first use
```

**In practice:** our wrapper tries depths 1–4 first (instant), then falls back to full 22-move depth search:

```javascript
for (let depth = 1; depth <= 4; depth++) {
  const s = cube.solve(depth);
  if (s && s.trim()) { solution = s; break; }
}
if (!solution) solution = cube.solve(22);
```

---

### Step 5 — 3D Rendering (`cube3d.js` + Three.js)

The 3D cube is built from **26 cubie objects** (3×3×3 minus the invisible center core).

#### How each cubie is built

```
Each cubie = 1 black box (core) + up to 6 colored sticker planes
```

- **Core**: `MeshStandardMaterial` with `color: 0x0a0a0a` — near-black with slight metalness
- **Stickers**: thin `PlaneGeometry` quads placed 0.001 units above each face, `MeshStandardMaterial` with `roughness: 0.25`

#### Coordinate System
Cubies sit on a 3×3×3 grid. Each position is tracked as `(gx, gy, gz)` where each value is -1, 0, or +1:

```
  y=+1  →  Top layer (U face)
  y=-1  →  Bottom layer (D face)
  z=+1  →  Front layer (F face)
  x=+1  →  Right layer (R face)
```

#### Applying Colors
The `FACELET_MAP` (built in `cubeState.js`) maps each of the 54 string positions to a `(x, y, z, direction)`. When we want to color the cube, we:

1. Find the cubie at grid position `(x, y, z)`
2. Determine which local face of that cubie faces the given world direction (accounting for any rotation the cubie may have accumulated)
3. Set that sticker's `material.color`

---

### Step 6 — Layer Turn Animation (`cube3d.js`)

When animating a move like `R` (right face clockwise):

1. **Find affected cubies** — all cubies where `gx === 1`
2. **Reparent them** into a temporary `layerGroup`
3. **Rotate the `layerGroup`** by 90° around the X axis over the animation duration
4. On completion, **reparent cubies back** to the main `cubeGroup`, update their `(gx, gy, gz)` grid positions, and bake their accumulated quaternion

```javascript
// Easing: smooth start + smooth end (ease in-out quad)
const eased = t < 0.5 ? 2*t*t : 1 - Math.pow(-2*t+2, 2)/2;
layerGroup.rotateOnWorldAxis(axisVec, totalAngle * eased);
```

---

### Step 7 — Camera Controls (Spherical Coordinates)

The camera is positioned using **spherical coordinates**:

```
x = radius × sin(φ) × sin(θ)
y = radius × cos(φ)
z = radius × sin(φ) × cos(θ)
```

- `θ` (theta) = horizontal angle — changed by left/right drag
- `φ` (phi) = vertical angle — changed by up/down drag
- `radius` = zoom distance — changed by scroll wheel

#### The Flip View Feature
"Flip" mirrors the camera vertically by animating `φ → (π - φ)` and rotating `θ → (θ + π)` simultaneously over 600ms using a **cubic ease-in-out** curve:

```javascript
// Cubic ease in-out
const t = raw < 0.5
  ? 4 * raw³
  : 1 - (-2*raw + 2)³ / 2;
this.spherical.phi   = startPhi   + (π - startPhi - startPhi) * t;
this.spherical.theta = startTheta + Math.PI * t;
```

The result: the cube smoothly rolls upside-down so you can see the bottom (yellow) face.

---

## 🖥️ UI & Styling (`css/style.css`)

Built with **vanilla CSS** + **Bootstrap 5** for layout.

### Design System

| Token | Value | Used for |
|---|---|---|
| `--bg` | `#0f1117` | Page background |
| `--bg-card` | `#1a1d27` | Card backgrounds |
| `--accent` | `#6c63ff` | Buttons, active states |
| `--accent2` | `#a78bfa` | Move chips, tab labels |
| `--border` | `rgba(255,255,255,0.08)` | Card borders |

### Key Visual Features
- **Viewer card** — purple-glowing border that intensifies on hover
- **Solution move chips** — purple-tinted pills; active move glows; completed moves fade to 35% opacity
- **Flip button icon** — bouncy spring animation (`cubic-bezier(0.34, 1.56, 0.64, 1)`) on hover
- **Viewer hint** — "Drag to rotate · Scroll to zoom" pill that fades when you interact
- **Floor plane** — subtle dark plane beneath the cube with a soft shadow blob

---

## 📦 Libraries Used

| Library | Version | Why |
|---|---|---|
| **Three.js** | r128 | 3D scene, WebGL renderer, geometry, materials, lighting |
| **cubejs** (cube.js) | custom | Cube model, move engine, string conversion |
| **cubejs** (solve.js) | custom | Kociemba two-phase solver + pruning tables |
| **Bootstrap** | 5.3.3 | Responsive grid, tabs, buttons, badges |
| **Bootstrap Icons** | 1.11.3 | All icons throughout the UI |
| **Inter** (Google Fonts) | — | Primary typeface |

> **Why is Three.js bundled locally?**
> Three.js r150+ removed the old global (`window.THREE`) build from their CDN. The CDN URL `three@0.164.1/build/three.min.js` returns **HTTP 404**. We ship r128 locally in `lib/three.min.js` which still provides the global build.

---

## 🚀 How to Run

No build step required. Just open with any local web server:

**Option A — VS Code Live Server**
1. Open the `Cube Solver` folder in VS Code
2. Right-click `index.html` → **Open with Live Server**
3. Go to `http://127.0.0.1:5500/index.html`

**Option B — Python**
```bash
cd "Cube Solver"
python -m http.server 8000
# Visit http://localhost:8000
```

**Option C — Node.js**
```bash
npx serve .
```

> ⚠️ **Do not open `index.html` directly as a `file://` URL** — browsers block canvas pixel reading (`getImageData`) on local file origins, which breaks the color detection.

---

## 📖 How to Use

```
1. Take 6 clear photos, one per face
   └── Hold the cube so the CENTER sticker matches the face label shown

2. Upload each photo into the matching card (U/R/F/D/L/B)
   └── Click anywhere on the card to open the file picker

3. Click "Detect Colors from Images"
   └── HSV algorithm reads 9 color samples per face (3×3 grid)

4. Review in the Color Editor tab
   └── Click any sticker to cycle its color if detection was wrong

5. Click "Solve Cube"
   └── Validates the state → builds solver tables (~3s first time) → finds solution

6. Click "Play" to animate
   └── Or use "Step" to go one move at a time
   └── Adjust the Speed slider for faster/slower animation
```

---

## 🔧 Controls

| Control | Action |
|---|---|
| **Drag** on the 3D viewer | Rotate camera |
| **Scroll** on the 3D viewer | Zoom in/out |
| **↺ Reset** button | Return camera to default angle |
| **⇄ Flip** button | Invert camera vertically (see bottom face) |
| **⇀ Scramble** button | Apply a random valid scramble |
| **Play / Step / Stop** | Control solution animation |
| **Speed slider** | 1 = slow (~500ms/move), 10 = fast (~50ms/move) |

---

## 🐛 Known Limitations

- **Color detection accuracy** depends on good, even lighting. Bright reflections or shadows on stickers can cause misclassification.
- The solver's **first initialization takes 3–5 seconds** while it builds the pruning tables in memory. Subsequent solves are instant.
- Very short solutions (1–4 moves) are found immediately. Solutions for scrambled cubes typically run through the full Kociemba search and complete in under 1 second.
- The cube model does **not currently support** slice moves (`M`, `E`, `S`) or whole-cube rotations in the manual editor.

---

## 🏗️ Architecture Diagram

```
┌─────────────────────────────────────────────────────┐
│                    index.html                        │
│  (Bootstrap layout, tab panels, button wiring)       │
└───────────┬───────────────────┬─────────────────────┘
            │                   │
    ┌───────▼──────┐   ┌────────▼────────┐
    │   app.js     │   │   cube3d.js     │
    │  Controller  │──▶│  Three.js 3D   │
    │  (events,    │   │  Viewer +       │
    │   state mgmt)│   │  Camera/Flip   │
    └──────┬───────┘   └────────────────┘
           │
    ┌──────▼──────────────────────────────┐
    │           cubeState.js              │
    │  - 54-char facelet string           │
    │  - FACELET_MAP (index → x,y,z)      │
    │  - validate(), applyMove()          │
    └──────┬───────────────────┬──────────┘
           │                   │
   ┌───────▼──────┐   ┌────────▼────────┐
   │imageParser.js│   │   solver.js     │
   │ Photo → HSV  │   │ Kociemba        │
   │ → color char │   │ two-phase wrap  │
   └──────────────┘   └────────┬────────┘
                               │
                    ┌──────────▼────────┐
                    │  lib/cube.js      │
                    │  lib/solve.js     │
                    │  (cubejs library) │
                    └───────────────────┘
```

---

## 📄 License

This project uses the **cubejs** library (MIT License) and **Three.js** (MIT License). All custom application code is free to use and modify.

---

*Built with ❤️ using vanilla JavaScript, Three.js, and the Kociemba algorithm.*
